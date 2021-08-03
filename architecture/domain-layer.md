# 도메인 계층

도메인 계층은 도메인 주도 설계에서 가장 핵심이 되는 계층이자 어떠한 계층에도 의존하지 않는 가장 고수준의 계층입니다. 여기서는 Aggregate 단위로 구성된 도메인 모델이 구현되어 있습니다. Aggregate 이란 여러 Entity 와 Value Object 로 이루어져 있는 집합이자 도메인 모델을 정의하는 가장 기본이 되는 단위입니다. Aggregate root 가 되는 Root entity 를 반드시 하나 갖게 됩니다. 여기서 Entity 란 유일한 식별자로 구분이 필요한 객체를 이야기합니다. Entity 는 수정될 수 있으며, Entity 마다 가지고 있는 중복되지 않은 id 로 각각의 Entity 를 식별합니다. Value object 는 Entity 와 반대로 id 값을 가지지 않으며 불변한 객체로 수정할 수 없습니다. Value object 는 이름 그대로 값 그 자체를 표현하며 수정이 필요하다면 새로운 Value object 를 만들어서 사용하는 것을 원칙으로 합니다.

도메인 모델은 내부에는 도메인의 비지니스 로직이 위치합니다. 도메인 모델의 핵심적인 역할중에 하나인 중요한 비지니스 로직을 모델 내부로 감추고 도메인 모델이 어떠한 행위들을 하는지 명확히 표현할 수 있어야합니다. 도메인 모델 내부에서는 각각의 비지니스 로직이 실행됨에 따라 도메인 이벤트를 만들어 수 있어야합니다. 여기서 만들어진 도메인 이벤트들은 각각의 도메인 객체가 Repository 를 통해 영속화되는 시점에 발행될 수 있어야 하며 응용 계층에 위치한 도메인 이벤트 핸들러가 발행된 이벤트를 수신하여 처리할 수 있어야 합니다.

```typescript
// domain/account.ts
import {
  UnprocessableEntityException,
  UnauthorizedException,
  InternalServerErrorException,
} from '@nestjs/common';
import { AggregateRoot } from '@nestjs/cqrs';
import * as bcrypt from 'bcrypt';

import { ErrorMessage } from 'src/accounts/domain/error';
import { AccountClosedEvent } from 'src/accounts/domain/events/account-closed.event';
import { AccountOpenedEvent } from 'src/accounts/domain/events/account-opened.event';
import { DepositedEvent } from 'src/accounts/domain/events/deposited.event';
import { PasswordUpdatedEvent } from 'src/accounts/domain/events/password-updated.event';
import { WithdrawnEvent } from 'src/accounts/domain/events/withdrawn.event';

export type AccountEssentialProperties = Required<{
  readonly id: string;
  readonly name: string;
}>;

export type AccountOptionalProperties = Partial<{
  readonly password: string;
  readonly balance: number;
  readonly openedAt: Date;
  readonly updatedAt: Date;
  readonly closedAt: Date | null;
  readonly version: number;
}>;

export type AccountProperties = AccountEssentialProperties &
  Required<AccountOptionalProperties>;

export interface Account {
  properties: () => AccountProperties;
  compareId: (id: string) => boolean;
  open: (password: string) => void;
  updatePassword: (password: string, data: string) => void;
  withdraw: (amount: number, password: string) => void;
  deposit: (amount: number) => void;
  close: (password: string) => void;
  commit: () => void;
}

export class AccountImplement extends AggregateRoot implements Account {
  private readonly id: string;
  private readonly name: string;
  private password = '';
  private balance = 0;
  private readonly openedAt: Date = new Date();
  private updatedAt: Date = new Date();
  private closedAt: Date | null = null;
  private version = 0;

  constructor(
    properties: AccountEssentialProperties & AccountOptionalProperties,
  ) {
    super();
    Object.assign(this, properties);
  }

  properties(): AccountProperties {
    return {
      id: this.id,
      name: this.name,
      password: this.password,
      balance: this.balance,
      openedAt: this.openedAt,
      updatedAt: this.updatedAt,
      closedAt: this.closedAt,
      version: this.version,
    };
  }

  compareId(id: string): boolean {
    return id === this.id;
  }

  open(password: string): void {
    this.setPassword(password);
    this.apply(Object.assign(new AccountOpenedEvent(), this));
  }

  private setPassword(password: string): void {
    if (this.password !== '' || password === '')
      throw new InternalServerErrorException(ErrorMessage.CAN_NOT_SET_PASSWORD);
    const salt = bcrypt.genSaltSync();
    this.password = bcrypt.hashSync(password, salt);
    this.updatedAt = new Date();
  }

  updatePassword(password: string, data: string): void {
    if (!this.comparePassword(password)) throw new UnauthorizedException();

    this.updatedAt = new Date();
    const salt = bcrypt.genSaltSync();
    this.password = bcrypt.hashSync(data, salt);
    this.apply(Object.assign(new PasswordUpdatedEvent(), this));
  }

  withdraw(amount: number, password: string): void {
    if (!this.comparePassword(password)) throw new UnauthorizedException();
    if (amount < 1)
      throw new InternalServerErrorException(
        ErrorMessage.CAN_NOT_WITHDRAW_UNDER_1,
      );
    if (this.balance < amount)
      throw new UnprocessableEntityException(
        ErrorMessage.REQUESTED_AMOUNT_EXCEEDS_YOUR_WITHDRAWAL_LIMIT,
      );
    this.balance -= amount;
    this.updatedAt = new Date();
    this.apply(Object.assign(new WithdrawnEvent(), this));
  }

  deposit(amount: number): void {
    if (amount < 1)
      throw new InternalServerErrorException(
        ErrorMessage.CAN_NOT_DEPOSIT_UNDER_1,
      );
    this.balance += amount;
    this.updatedAt = new Date();
    this.apply(Object.assign(new DepositedEvent(), this));
  }

  close(password: string): void {
    if (!this.comparePassword(password)) throw new UnauthorizedException();
    if (this.balance > 0)
      throw new UnprocessableEntityException(
        ErrorMessage.ACCOUNT_BALANCE_IS_REMAINED,
      );
    this.closedAt = new Date();
    this.updatedAt = new Date();
    this.apply(Object.assign(new AccountClosedEvent(), this));
  }

  private comparePassword(password: string): boolean {
    return bcrypt.compareSync(password, this.password);
  }
}

```

```typescript
// domain/account.spec.ts
import {
  InternalServerErrorException,
  UnauthorizedException,
  UnprocessableEntityException,
} from '@nestjs/common';

import { AccountImplement } from 'src/accounts/domain/account';
import { AccountClosedEvent } from 'src/accounts/domain/events/account-closed.event';
import { AccountOpenedEvent } from 'src/accounts/domain/events/account-opened.event';
import { DepositedEvent } from 'src/accounts/domain/events/deposited.event';
import { PasswordUpdatedEvent } from 'src/accounts/domain/events/password-updated.event';
import { WithdrawnEvent } from 'src/accounts/domain/events/withdrawn.event';

describe('Account', () => {
  describe('properties', () => {
    it('should return AccountProperties', () => {
      const properties = {
        id: 'id',
        name: 'name',
        password: '',
        balance: 0,
        openedAt: expect.anything(),
        updatedAt: expect.anything(),
        closedAt: null,
        version: 0,
      };

      const account = new AccountImplement({ id: 'id', name: 'name' });

      const result = account.properties();

      expect(result).toEqual(properties);
    });
  });

  describe('open', () => {
    it('should apply AccountOpenedEvent', () => {
      const account = new AccountImplement({ id: 'id', name: 'name' });

      account.open('password');

      const result = account.getUncommittedEvents();

      expect(result).toEqual([
        Object.assign(new AccountOpenedEvent(), account),
      ]);
    });
  });

  describe('updatePassword', () => {
    it('should throw UnauthorizedException when password is not matched', () => {
      const account = new AccountImplement({ id: 'id', name: 'name' });
      account.open('password');
      account.uncommit();

      expect(() =>
        account.updatePassword('wrongPassword', 'newPassword'),
      ).toThrowError(UnauthorizedException);
    });

    it('should update password', () => {
      const account = new AccountImplement({ id: 'id', name: 'name' });
      account.open('password');
      account.uncommit();

      account.updatePassword('password', 'newPassword');

      account.getUncommittedEvents();

      expect(account.properties().password).not.toEqual('');
      expect(account.properties().password).not.toEqual('password');
      expect(() => account.open('data')).toThrowError(
        InternalServerErrorException,
      );
      expect(account.getUncommittedEvents().length).toEqual(1);
      expect(account.getUncommittedEvents()).toEqual([
        Object.assign(new PasswordUpdatedEvent(), account),
      ]);
      expect(account.updatePassword('newPassword', 'data')).toEqual(undefined);
    });
  });

  describe('withdraw', () => {
    it('should throw UnauthorizedException when password is not matched', () => {
      const account = new AccountImplement({ id: 'id', name: 'name' });
      account.open('password');
      account.uncommit();

      expect(() => account.withdraw(0, 'wrongPassword')).toThrowError(
        UnauthorizedException,
      );
    });

    it('should throw InternalServerErrorException when given amount is under 1', () => {
      const account = new AccountImplement({ id: 'id', name: 'name' });
      account.open('password');
      account.uncommit();

      expect(() => account.withdraw(0, 'password')).toThrowError(
        InternalServerErrorException,
      );
    });

    it('should throw UnprocessableEntityException when given amount is over account balance', () => {
      const account = new AccountImplement({
        id: 'id',
        name: 'name',
        balance: 0,
      });
      account.open('password');
      account.uncommit();

      expect(() => account.withdraw(1, 'password')).toThrowError(
        UnprocessableEntityException,
      );
    });

    it('should withdraw from account', () => {
      const account = new AccountImplement({
        id: 'id',
        name: 'name',
        balance: 1,
      });
      account.open('password');
      account.uncommit();

      expect(account.withdraw(1, 'password')).toEqual(undefined);

      expect(account.getUncommittedEvents()).toEqual([
        Object.assign(new WithdrawnEvent(), account),
      ]);
    });
  });

  describe('deposit', () => {
    it('should throw InternalServerErrorException when given amount is under 1', () => {
      const account = new AccountImplement({ id: 'id', name: 'name' });
      account.open('password');
      account.uncommit();

      expect(() => account.deposit(0)).toThrowError(
        InternalServerErrorException,
      );
    });

    it('should deposit to account', () => {
      const account = new AccountImplement({ id: 'id', name: 'name' });
      account.open('password');
      account.uncommit();

      account.deposit(1);

      expect(account.getUncommittedEvents()).toEqual([
        Object.assign(new DepositedEvent(), account),
      ]);
      expect(account.withdraw(1, 'password')).toEqual(undefined);
    });
  });

  describe('close', () => {
    it('should throw UnauthorizedException when password is not matched', () => {
      const account = new AccountImplement({ id: 'id', name: 'name' });
      account.open('password');
      account.uncommit();

      expect(() => account.close('wrongPassword')).toThrowError(
        UnauthorizedException,
      );
    });

    it('should throw UnprocessableEntityException when account balance is over 0', () => {
      const account = new AccountImplement({
        id: 'id',
        name: 'name',
        balance: 1,
      });
      account.open('password');
      account.uncommit();

      expect(() => account.close('password')).toThrowError(
        UnprocessableEntityException,
      );
    });

    it('should close account', () => {
      const account = new AccountImplement({ id: 'id', name: 'name' });

      account.open('password');
      account.uncommit();

      account.close('password');
      expect(account.getUncommittedEvents()).toEqual([
        Object.assign(new AccountClosedEvent(), account),
      ]);
    });
  });
});

```

```typescript
// event/account-opened.event.ts
import { IEvent } from '@nestjs/cqrs';

import { AccountProperties } from 'src/accounts/domain/account';

export class AccountOpenedEvent implements IEvent, AccountProperties {
  readonly id: string;
  readonly name: string;
  readonly password: string;
  readonly balance: number;
  readonly openedAt: Date;
  readonly updatedAt: Date;
  readonly closedAt: Date | null;
  readonly version: number;
}

```

도메인 계층에는 도메인 모델을 영속화하고 도메인 계층에서 도메인의 컬렉션 인터페이스를 제공하기 위한 Repository 도 위치합니다. 다만 여기서는 Repository 의 인터페이스만을 정의하고 실제 구현체는 인프라 계층에서 하게 되는데, 이는 도메인에 대한 컬렉션 인터페이스를 제공하지만 도메인 객체의 영속화를 위한 구현은 도메인이 아닌 기술적 구현에 해당하기 때문에 분리된 인터페이스\(Separated interface\)를 적용하여 인터페이스와 구현체를 서로 다른 계층으로 분리합니다.

```typescript
// domain/repository.ts
import { Account } from 'src/accounts/domain/account';

export interface AccountRepository {
  newId: () => Promise<string>;
  save: (account: Account | Account[]) => Promise<void>;
  findById: (id: string) => Promise<Account | null>;
  findByIds: (ids: string[]) => Promise<Account[]>;
  findByName: (name: string) => Promise<Account[]>;
}

```

하나 이상의 도메인 모델의 연산이나 도메인에 중요한 로직이지만 도메인 모델로 표현할 수 없는 경우가 있습니다. 이러한 경우에 사용하는것을 도메인 서비스라하고 이것 역시 도메인 계층에 속하게 됩니다. 도메인 모델 내부의 로직에서 표현하기 어색하거나 어려운 또는 도메인 모델에 온전히 책임을 맡길 수 없는 로직이나 연산의 경우 이를 억지로 도메인 모델 내부에서 표현하려고 하기보다 도메인 서비스로 명시하고 별도의 서비스로 구현하는 것이 좋습니다. 도메인 서비스는 그 자체로 상태를 갖지 않고 이는 온전히 도메인 모델과 응용 계층 서비스의 클라이언트가 될 수 있도록 구현해야 합니다. 또한 서비스를 구현할때에는 도메인 로직으로 충분히 해결할 수 없는 문제인지 반드시 서비스가 필요한지 확인하여 불필요하거나 꼭 필요하지 않는 서비스들로 도메인 모델이 빈약해지거나 서비스가 비대해지는 현상을 방지할 수 있어야합니다.

```typescript
// domain/service.ts
import { Account } from 'src/accounts/domain/account';

export class RemittanceOptions {
  readonly password: string;
  readonly account: Account;
  readonly receiver: Account;
  readonly amount: number;
}

export class AccountService {
  remit({ account, receiver, password, amount }: RemittanceOptions): void {
    account.withdraw(amount, password);
    receiver.deposit(amount);
  }
}

```

```typescript
// domain/service.spec.ts
import { Account } from 'src/accounts/domain/account';
import { AccountService, RemittanceOptions } from 'src/accounts/domain/service';

describe('AccountService', () => {
  describe('remit', () => {
    it('should run remit', () => {
      const service = new AccountService();

      const account = { withdraw: jest.fn() } as unknown as Account;
      const receiver = { deposit: jest.fn() } as unknown as Account;

      const options: RemittanceOptions = {
        account,
        receiver,
        password: 'password',
        amount: 1,
      };

      expect(service.remit(options)).toEqual(undefined);
      expect(account.withdraw).toBeCalledTimes(1);
      expect(account.withdraw).toBeCalledWith(options.amount, options.password);
      expect(receiver.deposit).toBeCalledTimes(1);
      expect(receiver.deposit).toBeCalledWith(options.amount);
    });
  });
});

```

