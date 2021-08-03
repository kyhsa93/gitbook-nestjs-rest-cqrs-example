# 인프라 계층의 기술적 구현

인프라 계층에서는 다른 각 계층에서의 세부적인 기술 구현을 담당합니다. 이는 도메인 계층의 영속화를 위한 Repository 의 데이터베이스 시스템, 응용 계층의 Message protocol 을 위한 Publisher, Subscriber 와 통신을 위한 Adapter, 인터페이스 계층 의 Controller 를 위한 프로토콜 등의 기술적인 내용들 입니다. 이러한 세부 구현을 인프라 계층에서 모두 담당하게 함으로써 기술적인 부분이 변경되어도 변경사항이 다른 부분으로 퍼지지 않도록 격리하고 의존성 역전을 구현할 수 있습니다.

```typescript
// repositories/account.repository.ts
import { getRepository, In } from 'typeorm';
import { Inject } from '@nestjs/common';

import { AccountEntity } from 'src/accounts/infrastructure/entities/account.entity';

import { AccountRepository } from 'src/accounts/domain/repository';
import { Account } from 'src/accounts/domain/account';
import { AccountFactory } from 'src/accounts/domain/factory';

export class AccountRepositoryImplement implements AccountRepository {
  constructor(
    @Inject(AccountFactory) private readonly accountFactory: AccountFactory,
  ) {}

  async newId(): Promise<string> {
    const emptyEntity = new AccountEntity();
    const entity = await getRepository(AccountEntity).save(emptyEntity);
    return entity.id;
  }

  async save(data: Account | Account[]): Promise<void> {
    const models = Array.isArray(data) ? data : [data];
    const entities = models.map((model) => this.modelToEntity(model));
    await getRepository(AccountEntity).save(entities);
  }

  async findById(id: string): Promise<Account | null> {
    const entity = await getRepository(AccountEntity).findOne({ id });
    return entity ? this.entityToModel(entity) : null;
  }

  async findByIds(ids: string[]): Promise<Account[]> {
    const entities = await getRepository(AccountEntity).find({ id: In(ids) });
    return entities.map((entity) => this.entityToModel(entity));
  }

  async findByName(name: string): Promise<Account[]> {
    const entities = await getRepository(AccountEntity).find({ name });
    return entities.map((entity) => this.entityToModel(entity));
  }

  private modelToEntity(model: Account): AccountEntity {
    const properties = model.properties();
    return {
      ...properties,
      createdAt: properties.openedAt,
      deletedAt: properties.closedAt,
    };
  }

  private entityToModel(entity: AccountEntity): Account {
    return this.accountFactory.reconstitute({
      ...entity,
      openedAt: entity.createdAt,
      closedAt: entity.deletedAt,
    });
  }
}

```

```typescript
// queries/account.query.ts
import { getRepository } from 'typeorm';
import { Injectable } from '@nestjs/common';

import { AccountEntity } from 'src/accounts/infrastructure/entities/account.entity';

import {
  Account,
  AccountQuery,
  Accounts,
} from 'src/accounts/application/queries/account.query';

@Injectable()
export class AccountQueryImplement implements AccountQuery {
  async findById(id: string): Promise<undefined | Account> {
    return this.convertAccountFromEntity(
      await getRepository(AccountEntity).findOne(id),
    );
  }

  async find(offset: number, limit: number): Promise<Accounts> {
    return this.convertAccountsFromEntities(
      await getRepository(AccountEntity).find({ skip: offset, take: limit }),
    );
  }

  private convertAccountFromEntity(
    entity?: AccountEntity,
  ): undefined | Account {
    return entity
      ? { ...entity, openedAt: entity.createdAt, closedAt: entity.deletedAt }
      : undefined;
  }

  private convertAccountsFromEntities(entities: AccountEntity[]): Accounts {
    return entities.map((entity) => ({
      ...entity,
      openedAt: entity.createdAt,
      closedAt: entity.deletedAt,
    }));
  }
}

```

```typescript
// entities/base.entity.ts
import {
  CreateDateColumn,
  DeleteDateColumn,
  Entity,
  UpdateDateColumn,
  VersionColumn,
} from 'typeorm';

@Entity()
export class BaseEntity {
  @CreateDateColumn()
  createdAt: Date = new Date();

  @UpdateDateColumn()
  updatedAt: Date = new Date();

  @DeleteDateColumn()
  deletedAt: Date | null = null;

  @VersionColumn()
  version = 0;
}

```

```typescript
// entities/account.entity.ts
import { Entity, PrimaryGeneratedColumn, Column } from 'typeorm';

import { BaseEntity } from 'src/accounts/infrastructure/entities/base.entity';

@Entity()
export class AccountEntity extends BaseEntity {
  @PrimaryGeneratedColumn('uuid')
  id!: string;

  @Column({ type: 'varchar' })
  name = '';

  @Column({ type: 'varchar' })
  password = '';

  @Column({ type: 'int' })
  balance = 0;
}

```

```typescript
// message/integration-event.publisher.ts
import { Injectable } from '@nestjs/common';
import { Channel, connect, Connection } from 'amqplib';

import { AppService, RabbitMQConfig } from 'src/app.service';

import {
  IntegrationEvent,
  IntegrationEventPublisher,
} from 'src/accounts/application/events/integration';

@Injectable()
export class IntegrationEventPublisherImplement
  implements IntegrationEventPublisher
{
  private static exchange: string;

  private readonly promisedChannel: Promise<Channel>;

  constructor() {
    const config = AppService.rabbitMQConfig();
    IntegrationEventPublisherImplement.exchange = config.exchange;
    this.promisedChannel = IntegrationEventPublisherImplement.connect(config);
  }

  async publish(message: IntegrationEvent): Promise<void> {
    this.promisedChannel.then((channel) =>
      channel.publish(
        IntegrationEventPublisherImplement.exchange,
        message.subject,
        Buffer.from(JSON.stringify(message.data)),
      ),
    );
  }

  private static async connect(config: RabbitMQConfig): Promise<Channel> {
    return connect(config)
      .then(IntegrationEventPublisherImplement.createChannel)
      .then(IntegrationEventPublisherImplement.assertExchange)
      .catch(() => IntegrationEventPublisherImplement.connect(config));
  }

  private static async createChannel(connection: Connection): Promise<Channel> {
    return connection.createChannel();
  }

  private static async assertExchange(channel: Channel): Promise<Channel> {
    await channel.assertExchange(
      IntegrationEventPublisherImplement.exchange,
      'topic',
      {
        durable: true,
      },
    );
    return channel;
  }
}

```

```typescript
// cache/event-store.ts
import Redis from 'ioredis';

import { AppService } from 'src/app.service';

import { Event, EventStore } from 'src/accounts/application/events/integration';

export class EventStoreImplement implements EventStore {
  private readonly master: Redis.Redis;

  private readonly slave: Redis.Redis;

  constructor() {
    const { master, slave } = AppService.redisClusterConfig();
    this.master = new Redis(master.port, master.host).on(
      'error',
      this.failToConnectRedis,
    );
    this.slave = new Redis(slave.port, slave.host).on(
      'error',
      this.failToConnectRedis,
    );
  }
  async save(event: Event): Promise<void> {
    await this.master.set(event.data.id, JSON.stringify(event.data));
  }

  async set(key: string, value: string): Promise<void> {
    await this.master.set(key, value, 'EX', 1);
  }

  async get(key: string): Promise<string | null> {
    return this.slave
      .get(key)
      .then((result) => result)
      .catch(() => null);
  }

  private failToConnectRedis(error: Error): Promise<void> {
    console.error(error);
    process.exit(1);
  }
}

```

