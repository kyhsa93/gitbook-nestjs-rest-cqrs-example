# 인터페이스 계층의 컨트롤러와 DTO

도메인 주도 설계와 간단한 CQRS 모델을 이용하여 어플리케이션을 구성해보았습니다. CQRS 에서 고도화된 모델은 Event sourcing 까지 구현하여 사용하겠지만, 이 프로젝트에서는 어플리케이션 내부에서 Command 와 Query 를 분리 구현하는정도로 사용했습니다. 어플리케이션은 하나의 데이터베이스를 사용하며 Command 를 사용하여 도메인 모델에 접근하고, Query 를 사용하여 영속화되어 있는 데이터를 질의합니다. 

작성된 어플리케이션은 은행 계좌의 도메인에서 계좌 개설, 해지, 입금, 출금, 송금을 구현하였으며 이 중 몇가지 기능이 처리되는 시나리오를 통하여 어플리케이션의 전체적인 구조와 어떻게 동작하는지를 살펴보고자 합니다.

먼저 어플리케이션이 클라이언트의 계좌 생성요청을 처리하는 시나리오를 토대로 살펴보도록 하겠습니다. 이 시나리오에서 등장하는 클라이언트는 HTTP 를 사용하며 이 요청이 가장 먼저 마주하게 되는 것은 인터페이스 계층입니다. 인터페이스 계층에서는 클라이언트로부터 받은 요청을 검증하고 각 요청에 대해 어떻게 응답할지를 결정합니다. 또한, 요청에 맞는 Command 또는 Query 를 생성하여 각각의 맞는 Bus 로 전송하여 응용 계층에서 실행될 수 있도록 합니다. 아래는 요청을 처리하는 인터페이스 계층의 컨트롤러와 계정 생성 요청 바디 데이터를 받아서 검증하고 전달하는 dto 입니다.

```typescript
// open.account.body.ts
import { ApiProperty } from '@nestjs/swagger';
import { IsString } from 'class-validator';

export default class OpenAccountBody {
  @ApiProperty({ example: 'name' })
  @IsString()
  public readonly name: string = '';

  @ApiProperty({ example: 'testpassword' })
  @IsString()
  public readonly password: string = '';
}
```

```typescript
// account.controller.ts
import { ApiTags } from '@nestjs/swagger';
import {
  Controller,
  Post,
  Body,
  Get,
  Param,
  Put,
  Delete,
  NotFoundException,
  Query,
} from '@nestjs/common';
import { CommandBus, QueryBus } from '@nestjs/cqrs';

import OpenAccountBody from '@src/account/interface/dto/open.account.body';
import UpdateAccountPathParam from '@src/account/interface/dto/update.account.param';
import UpdateAccountBody from '@src/account/interface/dto/update.account.body';
import CloseAccountPathParam from '@src/account/interface/dto/close.account.param';
import CloseAccountBody from '@src/account/interface/dto/close.account.body';
import ReadAccountPathParam from '@src/account/interface/dto/get.account.by.id.param';
import GetAccountQuery from '@src/account/interface/dto/get.account.query';

import OpenAccountCommand from '@src/account/application/command/implements/open.account';
import ReadAccountQuery from '@src/account/application/query/implements/find.by.id';
import UpdateAccountCommand from '@src/account/application/command/implements/update.account';
import CloseAccountCommand from '@src/account/application/command/implements/close.account';
import { Account, AccountsAndCount } from '@src/account/application/query/query';
import FindAccountQuery from '@src/account/application/query/implements/find';
import RemittanceBody from '@src/account/interface/dto/remittance.body';
import RemittanceCommand from '@src/account/application/command/implements/remittance';

@ApiTags('Accounts')
@Controller('/accounts')
export default class AccountController {
  constructor(private readonly commandBus: CommandBus, private readonly queryBus: QueryBus) {}

  @Post()
  public async handleOpenAccountRequest(@Body() body: OpenAccountBody): Promise<void> {
    const { name, password } = body;
    await this.commandBus.execute(new OpenAccountCommand(name, password));
  }

  @Post('/remittance')
  public async handleRemittanceRequest(@Body() body: RemittanceBody): Promise<void> {
    const { senderId, receiverId, password, amount } = body;
    const command = new RemittanceCommand(senderId, receiverId, password, amount);
    await this.commandBus.execute(command);
  }

  @Put(':id')
  public async handleUpdateAccountRequest(
    @Param() param: UpdateAccountPathParam,
    @Body() body: UpdateAccountBody,
  ): Promise<void> {
    const { oldPassword, newPassword } = body;
    await this.commandBus.execute(new UpdateAccountCommand(param.id, oldPassword, newPassword));
  }

  @Delete(':id')
  public async handleCloseAccountRequest(
    @Param() param: CloseAccountPathParam,
    @Body() body: CloseAccountBody,
  ): Promise<void> {
    const { password } = body;
    await this.commandBus.execute(new CloseAccountCommand(param.id, password));
  }

  @Get()
  public async handleGetAccountRequest(
    @Query() { take = 10, page = 1, names = [] }: GetAccountQuery,
  ): Promise<AccountsAndCount> {
    const conditions = { names: this.toArray(names) };
    const query = new FindAccountQuery(take, page, conditions);
    return this.queryBus.execute(query);
  }

  @Get(':id')
  public async handlerGetAccountByIdRequest(
    @Param() param: ReadAccountPathParam,
  ): Promise<Account> {
    const account: Account = await this.queryBus.execute(new ReadAccountQuery(param.id));
    if (!account) throw new NotFoundException();
    return account;
  }

  private toArray(value: string | string[]): string[] {
    return Array.isArray(value) ? value : [value].filter(item => item !== undefined);
  }
}

```



