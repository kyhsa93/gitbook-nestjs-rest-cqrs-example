# 인프라스트럭처

인프라 계층에서는 다른 각 계층에서의 세부적인 기술 구현을 담당합니다. 이는 도메인 계층의 영속화를 위한 Repository 의 데이터베이스 시스템, 응용 계층의 Message protocol 을 위한 Publisher, Subscriber 와 통신을 위한 Adapter, 인터페이스 계층 의 Controller 를 위한 프로토콜 등의 기술적인 내용들 입니다. 이러한 세부 구현을 인프라 계층에서 모두 담당하게 함으로써 기술적인 부분이 변경되어도 변경사항이 다른 부분으로 퍼지지 않도록 격리하고 의존성 역전을 구현할 수 있습니다.

```typescript
// repository.ts
import { EntityRepository, getRepository } from 'typeorm';
import uuid from 'uuid';

import AccountEntity from '@src/account/infrastructure/entity/account';

import Account from '@src/account/domain/model/account';
import AccountRepository from '@src/account/domain/repository';
import AccountFactory from '@src/account/domain/factory';

@EntityRepository(AccountEntity)
export default class AccountRepositoryImplement implements AccountRepository {
  constructor(private readonly accountFactory: AccountFactory) {}

  public async newId (): Promise<string> {
    const emptyEntity = new AccountEntity();
    emptyEntity.name = uuid.v1();
    emptyEntity.openedAt = new Date();
    emptyEntity.updatedAt = new Date();
    const entity = await getRepository(AccountEntity).save(emptyEntity);
    return entity.id;
  };

  public async save(data: Account | Account[]): Promise<void> {
    const models = Array.isArray(data) ? data : [data];
    const entities = models.map((model) => this.modelToEntity(model));
    await getRepository(AccountEntity).save(entities);
  }

  public async findById(id: string): Promise<Account | undefined> {
    const entity = await getRepository(AccountEntity).findOne({ id });
    return entity ? this.entityToModel(entity) : undefined;
  }

  public async findByName(name: string): Promise<Account[]> {
    const entities = await getRepository(AccountEntity).find({ name });
    return entities.map((entity) => this.entityToModel(entity));
  }

  private modelToEntity(model: Account): AccountEntity {
    const anemic = model.toAnemic();
    return { ...anemic, password: { ...anemic.password } };
  };

  private entityToModel(entity: AccountEntity): Account {
    return this.accountFactory.reconstitute({ ...entity });
  }
}

```

```typescript
// query.ts
import { FindConditions, FindManyOptions, getRepository, In } from 'typeorm';
import { Injectable } from '@nestjs/common';

import AccountEntity from '@src/account/infrastructure/entity/account';

import { 
  Account, AccountFindConditions, Accounts, AccountsAndCount, AccountWhereConditions, Query 
} from '@src/account/application/query/query';

@Injectable()
export default class AccountQuery implements Query {
  private convertAccountFromEntity(entity?: AccountEntity): undefined | Account {
    return entity ? { ...entity } : undefined;
  }

  public async findById(id: string): Promise<undefined | Account> {
    return this.convertAccountFromEntity(await getRepository(AccountEntity).findOne(id));
  };

  private convertAccountsFromEntities(entities: AccountEntity[]): Accounts {
    return entities.map(entity => ({ ...entity }));
  }

  private convertAccountsAndCount([entities, count]: [AccountEntity[], number]): AccountsAndCount {
    return { data: this.convertAccountsFromEntities(entities), count };
  }

  private convertWhereConditions(conditions: AccountWhereConditions): undefined | FindConditions<AccountEntity> {
    let result = {};
    conditions.names.length === 0 ? undefined : Object.assign(result, { name: In(conditions.names) });
    return Object.keys(result).length === 0 ? undefined : result;
  }

  private convertFindConditions({ take, page, where }: AccountFindConditions): FindManyOptions<AccountEntity> {
    return { skip: take * (page - 1), take, where: where ? this.convertWhereConditions(where) : undefined };
  }

  public async findAndCount(conditions: AccountFindConditions): Promise<AccountsAndCount> {
    return getRepository(AccountEntity)
      .findAndCount(this.convertFindConditions(conditions))
      .then(entitiesAndCount => this.convertAccountsAndCount(entitiesAndCount));
  }
}

```

```typescript
// message/publisher.ts
import { Injectable, InternalServerErrorException } from '@nestjs/common';
import { connect, Options } from 'amqplib';

import AppConfiguration from '@src/app.config';

import { Event, Publisher } from '@src/account/application/event/publisher';

@Injectable()
export default class IntegrationEventPublisher implements Publisher {
  private readonly exchange: string;

  private readonly connectionOptions: Options.Connect;

  constructor() {
    this.exchange = AppConfiguration.rabbitMQ.exchange;
    this.connectionOptions = AppConfiguration.rabbitMQ;
  }

  public async publish(message: Event): Promise<void> {
    const channel = await (await connect(this.connectionOptions)).createChannel();
    await channel.assertExchange(this.exchange, 'topic', { durable: true });
    if (!channel) throw new InternalServerErrorException('cannot get publisher channel');

    channel.publish(this.exchange, message.key, Buffer.from(JSON.stringify(message.data)));
  }
}

```

