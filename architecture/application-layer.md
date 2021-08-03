# 응용 계층의 CQRS

응용 계층은 어플리케이션의 흐름을 제어하며 외부 컨텍스트의 서비스로 이벤트 전파하거나 호출하여 통신하는 역할을 담당합니다. 응용 계층은 도메인 계층을 의존하고 도메인 모델의 도메인 로직을 실행합니다. 이 계층은 도메인 계층에 대한 클라이언트로 동작하며 도메인에 대한 어떠한 것도 포함하지 않습니다. 도메인을 호출하고 작업에 대한 조율만을 담당하기 때문에 이 계층은 작게 유지되어야 합니다.

CQRS 의 Command 와 Command handler 가 여기에 위치하며 이 부분을 서비스라고 할 수 있습니다. Command handler 는 Command bus 를 통해 수신한 Command 를 사용하여 도메인의 변경을 담당합니다. 또한 각각의 Command handler 는 하나의 트랜젝션으로 동작합니다. 이것을 이용하여 하나의 기능인 Command 와 서비스에서 호출된 도메인 로직이 일관성 있게 동작하도록 합니다. 

```typescript
// commands/open-account.command.ts
import { ICommand } from '@nestjs/cqrs';

export class OpenAccountCommand implements ICommand {
  constructor(readonly name: string, readonly password: string) {}
}

```

```typescript
// commands/open-account.handler.ts
import { Inject } from '@nestjs/common';
import { CommandHandler, ICommandHandler } from '@nestjs/cqrs';

import { OpenAccountCommand } from 'src/accounts/application/commands/open-account.command';
import { InjectionToken } from 'src/accounts/application/injection.token';

import { AccountFactory } from 'src/accounts/domain/factory';
import { AccountRepository } from 'src/accounts/domain/repository';

@CommandHandler(OpenAccountCommand)
export class OpenAccountHandler
  implements ICommandHandler<OpenAccountCommand, void>
{
  constructor(
    @Inject(InjectionToken.ACCOUNT_REPOSITORY)
    private readonly accountRepository: AccountRepository,
    private readonly accountFactory: AccountFactory,
  ) {}

  async execute(command: OpenAccountCommand): Promise<void> {
    const account = this.accountFactory.create(
      await this.accountRepository.newId(),
      command.name,
    );

    account.open(command.password);

    await this.accountRepository.save(account);

    account.commit();
  }
}

```

```typescript
// commands/open-account.handler.spec.ts
import { ModuleMetadata, Provider } from '@nestjs/common';
import { Test } from '@nestjs/testing';

import { OpenAccountCommand } from 'src/accounts/application/commands/open-account.command';
import { OpenAccountHandler } from 'src/accounts/application/commands/open-account.handler';
import { InjectionToken } from 'src/accounts/application/injection.token';
import { AccountFactory } from 'src/accounts/domain/factory';

import { AccountRepository } from 'src/accounts/domain/repository';

describe('OpenAccountHandler', () => {
  let handler: OpenAccountHandler;
  let repository: AccountRepository;
  let factory: AccountFactory;

  beforeEach(async () => {
    const repoProvider: Provider = {
      provide: InjectionToken.ACCOUNT_REPOSITORY,
      useValue: {},
    };
    const factoryProvider: Provider = {
      provide: AccountFactory,
      useValue: {},
    };
    const providers: Provider[] = [
      OpenAccountHandler,
      repoProvider,
      factoryProvider,
    ];
    const moduleMetadata: ModuleMetadata = { providers };
    const testModule = await Test.createTestingModule(moduleMetadata).compile();

    handler = testModule.get(OpenAccountHandler);
    repository = testModule.get(InjectionToken.ACCOUNT_REPOSITORY);
    factory = testModule.get(AccountFactory);
  });

  describe('execute', () => {
    it('should execute OpenAccountCommand', async () => {
      const account = { open: jest.fn(), commit: jest.fn() };

      factory.create = jest.fn().mockReturnValue(account);
      repository.newId = jest.fn().mockResolvedValue('accountId');
      repository.save = jest.fn().mockResolvedValue(undefined);

      const command = new OpenAccountCommand('accountId', 'password');

      await expect(handler.execute(command)).resolves.toEqual(undefined);
      expect(repository.newId).toBeCalledTimes(1);
      expect(account.open).toBeCalledTimes(1);
      expect(account.open).toBeCalledWith(command.password);
      expect(repository.save).toBeCalledTimes(1);
      expect(repository.save).toBeCalledWith(account);
      expect(account.commit).toBeCalledTimes(1);
    });
  });
});

```

CQRS 의 Query 와 Query handler 역시 이곳에 구현됩니다. Query handler 도 Command handler 와 마찬가지로 Query bus 를 통해 받은 Query 를 사용하며, 각각의 Query 에 맞는 데이터 모델을 데이터 저장소에서 조회합니다. 다만 Query 로 조회된 데이터 모델은 도메인 모델과 일치하지 않을 수 있으며 다양한 형태를 가질 수 있습니다.

```typescript
// queries/find-accounts.query.ts
import { IQuery } from '@nestjs/cqrs';

export class FindAccountsQuery implements IQuery {
  constructor(readonly offset: number, readonly limit: number) {}
}

```

```typescript
// queries/find-accounts.result.ts
import { IQueryResult } from '@nestjs/cqrs';

export class ItemInFindAccountsResult {
  readonly id: string = '';
  readonly name: string = '';
  readonly balance: number = 0;
}

export class FindAccountsResult
  extends Array<ItemInFindAccountsResult>
  implements IQueryResult {}

```

```typescript
// queries/find-accounts.query.handler.ts
import { Inject, InternalServerErrorException } from '@nestjs/common';
import { IQueryHandler, QueryHandler } from '@nestjs/cqrs';

import { InjectionToken } from 'src/accounts/application/injection.token';
import {
  AccountQuery,
  ItemInAccounts,
} from 'src/accounts/application/queries/account.query';
import { FindAccountsQuery } from 'src/accounts/application/queries/find-accounts.query';
import {
  FindAccountsResult,
  ItemInFindAccountsResult,
} from 'src/accounts/application/queries/find-accounts.result';

@QueryHandler(FindAccountsQuery)
export class FindAccountsHandler
  implements IQueryHandler<FindAccountsQuery, FindAccountsResult>
{
  constructor(
    @Inject(InjectionToken.ACCOUNT_QUERY) readonly accountQuery: AccountQuery,
  ) {}

  async execute(query: FindAccountsQuery): Promise<FindAccountsResult> {
    return (await this.accountQuery.find(query.offset, query.limit)).map(
      this.filterResultProperties,
    );
  }

  private filterResultProperties(
    data: ItemInAccounts,
  ): ItemInFindAccountsResult {
    const dataKeys = Object.keys(data);
    const resultKeys = Object.keys(new ItemInFindAccountsResult());

    if (dataKeys.length < resultKeys.length)
      throw new InternalServerErrorException();

    if (resultKeys.find((resultKey) => !dataKeys.includes(resultKey)))
      throw new InternalServerErrorException();

    dataKeys
      .filter((dataKey) => !resultKeys.includes(dataKey))
      .forEach((dataKey) => delete data[dataKey]);

    return data;
  }
}

```

```typescript
// queries/find-accounts.query.handler.spec.ts
import { ModuleMetadata, Provider } from '@nestjs/common';
import { Test } from '@nestjs/testing';

import { InjectionToken } from 'src/accounts/application/injection.token';
import {
  AccountQuery,
  Accounts,
} from 'src/accounts/application/queries/account.query';
import { FindAccountsHandler } from 'src/accounts/application/queries/find-accounts.handler';
import { FindAccountsQuery } from 'src/accounts/application/queries/find-accounts.query';
import { FindAccountsResult } from 'src/accounts/application/queries/find-accounts.result';

describe('FindAccountsHandler', () => {
  let handler: FindAccountsHandler;
  let accountQuery: AccountQuery;

  beforeEach(async () => {
    const queryProvider: Provider = {
      provide: InjectionToken.ACCOUNT_QUERY,
      useValue: {},
    };
    const providers: Provider[] = [queryProvider, FindAccountsHandler];
    const moduleMetadata: ModuleMetadata = { providers };
    const testModule = await Test.createTestingModule(moduleMetadata).compile();
    handler = testModule.get(FindAccountsHandler);
    accountQuery = testModule.get(InjectionToken.ACCOUNT_QUERY);
  });

  describe('execute', () => {
    it('should return FindAccountsResult when execute FindAccountsQuery', async () => {
      const accounts: Accounts = [
        {
          id: 'accountId',
          name: 'test',
          password: 'password',
          balance: 0,
          openedAt: expect.anything(),
          updatedAt: expect.anything(),
          closedAt: null,
        },
      ];
      accountQuery.find = jest.fn().mockResolvedValue(accounts);

      const query = new FindAccountsQuery(0, 1);

      const result: FindAccountsResult = [
        {
          id: 'accountId',
          name: 'test',
          balance: 0,
        },
      ];

      await expect(handler.execute(query)).resolves.toEqual(result);
      expect(accountQuery.find).toBeCalledTimes(1);
      expect(accountQuery.find).toBeCalledWith(query.offset, query.limit);
    });
  });
});

```

```typescript
// queries/account.query.ts
export class Account {
  readonly id: string;
  readonly name: string;
  readonly password: string;
  readonly balance: number;
  readonly openedAt: Date;
  readonly updatedAt: Date;
  readonly closedAt: Date | null;
}

export class ItemInAccounts {
  readonly id: string;
  readonly name: string;
  readonly password: string;
  readonly balance: number;
  readonly openedAt: Date;
  readonly updatedAt: Date;
  readonly closedAt: Date | null;
}

export class Accounts extends Array<ItemInAccounts> {}

export interface AccountQuery {
  findById: (id: string) => Promise<Account>;
  find: (offset: number, limit: number) => Promise<Accounts>;
}

```

서비스에서 기능을 처리하기 위해 외부 컨텍스트의 서비스를 호출해야하는 상황이 있을 수 있습니다. 이를 위한 어댑터가 이곳에 인터페이스로 위치하며 서비스에서 필요시 주입받아 사용하게 됩니다. 도메인의 변경으로 발생한 도메인 이벤트를 도메인 이벤트 핸들러를 통하여 처리합니다. 도메인 이벤트 핸들러는 응용 계층에서 주어진 이벤트를 처리하며, 통합 이벤트의 형태로 외부에 전파하는 역할도 수행합니다.

```typescript
// events/integration.ts
import { AccountProperties } from 'src/accounts/domain/account';

export class Event {
  readonly subject: string;
  readonly data: AccountProperties;
}

export class IntegrationEvent {
  readonly subject: string;
  readonly data: Record<string, string>;
}

export interface IntegrationEventPublisher {
  publish: (event: IntegrationEvent) => Promise<void>;
}

export interface EventStore {
  save: (event: Event) => Promise<void>;
}

export enum IntegrationEventSubject {
  OPENED = 'account.opened',
  CLOSED = 'account.closed',
  DEPOSITED = 'account.deposited',
  WITHDRAWN = 'account.withdrawn',
  PASSWORD_UPDATED = 'account.password.updated',
}


```

```typescript
// events/account-opened.handler.ts
import { Inject, Logger } from '@nestjs/common';
import { EventsHandler, IEventHandler } from '@nestjs/cqrs';

import {
  EventStore,
  IntegrationEventPublisher,
  IntegrationEventSubject,
} from 'src/accounts/application/events/integration';
import { InjectionToken } from 'src/accounts/application/injection.token';

import { AccountOpenedEvent } from 'src/accounts/domain/events/account-opened.event';

@EventsHandler(AccountOpenedEvent)
export class AccountOpenedHandler implements IEventHandler<AccountOpenedEvent> {
  constructor(
    private readonly logger: Logger,
    @Inject(InjectionToken.INTEGRATION_EVENT_PUBLISHER)
    private readonly publisher: IntegrationEventPublisher,
    @Inject(InjectionToken.EVENT_STORE) private readonly eventStore: EventStore,
  ) {}

  async handle(event: AccountOpenedEvent): Promise<void> {
    this.logger.log(
      `${IntegrationEventSubject.OPENED}: ${JSON.stringify(event)}`,
    );
    await this.publisher.publish({
      subject: IntegrationEventSubject.OPENED,
      data: { id: event.id },
    });
    await this.eventStore.save({
      subject: IntegrationEventSubject.OPENED,
      data: event,
    });
  }
}

```

