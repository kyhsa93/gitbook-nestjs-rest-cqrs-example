# 응용 계층의 CQRS

응용 계층은 어플리케이션의 흐름을 제어하며 외부 컨텍스트의 서비스로 이벤트 전파하거나 호출하여 통신하는 역할을 담당합니다. 응용 계층은 도메인 계층을 의존하고 도메인 모델의 도메인 로직을 실행합니다. 이 계층은 도메인 계층에 대한 클라이언트로 동작하며 도메인에 대한 어떠한 것도 포함하지 않습니다. 도메인을 호출하고 작업에 대한 조율만을 담당하기 때문에 이 계층은 작게 유지되어야 합니다.

CQRS 의 Command 와 Command handler 가 여기에 위치하며 이 부분을 서비스라고 할 수 있습니다. Command handler 는 Command bus 를 통해 수신한 Command 를 사용하여 도메인의 변경을 담당합니다. 또한 각각의 Command handler 는 하나의 트랜젝션으로 동작합니다. 이것을 이용하여 하나의 기능인 Command 와 서비스에서 호출된 도메인 로직이 일관성 있게 동작하도록 합니다. 

```typescript
// command/implements/open.account.ts
import { ICommand } from '@nestjs/cqrs';

export default class OpenAccountCommand implements ICommand {
  constructor(public readonly name: string, public readonly password: string) {}
}

```

```typescript
// command/handlers/open.account.ts
import { Transaction } from 'typeorm';
import { CommandHandler, EventPublisher, ICommandHandler } from '@nestjs/cqrs';
import { Inject } from '@nestjs/common';

import OpenAccountCommand from '@src/account/application/command/implements/open.account';

import AccountFactory from '@src/account/domain/factory';
import AccountRepository from '@src/account/domain/repository';

@CommandHandler(OpenAccountCommand)
export default class OpenAccountCommandHandler implements ICommandHandler<OpenAccountCommand> {
  constructor(
    private readonly accountFactory: AccountFactory,
    private readonly eventPublisher: EventPublisher,
    @Inject('AccountRepositoryImplement') private readonly accountRepository: AccountRepository,
  ) {}

  @Transaction()
  public async execute(command: OpenAccountCommand): Promise<void> {
    const { name, password } = command;

    const id = await this.accountRepository.newId();

    const account = this.eventPublisher.mergeObjectContext(
      this.accountFactory.create(id, name, password),
    );

    await this.accountRepository.save(account);
    
    account.commit();
  }
}

```

CQRS 의 Query 와 Query handler 역시 이곳에 구현됩니다. Query handler 도 Command handler 와 마찬가지로 Query bus 를 통해 받은 Query 를 사용하며, 각각의 Query 에 맞는 데이터 모델을 데이터 저장소에서 조회합니다. 다만 Query 로 조회된 데이터 모델은 도메인 모델과 일치하지 않을 수 있으며 다양한 형태를 가질 수 있습니다.

```typescript
// query/implements/find.ts
import { IQuery } from "@nestjs/cqrs";

export default class FindAccountsQuery implements IQuery {
  constructor(
    public readonly take: number,
    public readonly page: number,
    public readonly where?: { names: string[] },
  ){}
}

```

```typescript
// query/handlers/find.ts
import { Inject } from "@nestjs/common";
import { IQueryHandler, QueryHandler } from "@nestjs/cqrs";

import FindAccountsQuery from "@src/account/application/query/implements/find";
import { AccountsAndCount, Query } from "@src/account/application/query/query";

@QueryHandler(FindAccountsQuery)
export default class FindAccountsQueryHandler implements IQueryHandler<FindAccountsQuery> {
  constructor(@Inject('AccountQuery') private readonly accountQuery: Query ){}

  public async execute(query: FindAccountsQuery): Promise<AccountsAndCount> {
    return this.accountQuery.findAndCount(query);
  }
}

```

```typescript
// query/query.ts
import { IQueryResult } from "@nestjs/cqrs";

export interface AccountFindConditions {
  readonly take: number;
  readonly page: number;
  readonly where?: AccountWhereConditions;
}

export interface AccountWhereConditions {
  readonly names: string[];
}

// ...

export interface Accounts
  extends Array<{
    readonly id: string;
    readonly name: string;
    readonly openedAt: Date;
  }>, IQueryResult {}

export interface AccountsAndCount extends IQueryResult {
  readonly count: number;
  readonly data: { 
    readonly id: string;
    readonly name: string;
    readonly openedAt: Date;
   }[]
}

export interface Query {
  findById(id: string): Promise<undefined | Account>;
  findAndCount(conditions: AccountFindConditions): Promise<AccountsAndCount>;
}

```

서비스에서 기능을 처리하기 위해 외부 컨텍스트의 서비스를 호출해야하는 상황이 있을 수 있습니다. 이를 위한 어댑터가 이곳에 인터페이스로 위치하며 서비스에서 필요시 주입받아 사용하게 됩니다. 도메인의 변경으로 발생한 도메인 이벤트를 도메인 이벤트 핸들러를 통하여 처리합니다. 도메인 이벤트 핸들러는 응용 계층에서 주어진 이벤트를 처리하며, 통합 이벤트의 형태로 외부에 전파하는 역할도 수행합니다.

```typescript
// event/implements/account.opened.ts
import { Event } from '@src/account/application/event/publisher';

export default class AccountCreatedIntegrationEvent implements Event {
  constructor(
    public readonly key: string,
    public readonly data: { readonly id: string; },
  ) {}
}

```

```typescript
// event/handlers/account.opened.ts
import { Inject } from '@nestjs/common';
import { EventsHandler, IEventHandler } from '@nestjs/cqrs';

import IntegrationEvent from '@src/account/application/event/implements/account.opened';
import { Publisher } from '@src/account/application/event/publisher';

import AccountOpenedDomainEvent from '@src/account/domain/event/account.opened';

const MESSAGE_KEY = 'account.opened';

@EventsHandler(AccountOpenedDomainEvent)
export default class AccountOpenedDomainEventHandler
  implements IEventHandler<AccountOpenedDomainEvent> {
  constructor(@Inject('IntegrationEventPublisher') private readonly publisher: Publisher) {}

  public async handle(event: AccountOpenedDomainEvent): Promise<void> {
    const integrationEvent = new IntegrationEvent(MESSAGE_KEY, event);
    await this.publisher.publish(integrationEvent);
  }
}

```

