# 인터페이스 계층의 컨트롤러와 DTO

도메인 주도 설계와 간단한 CQRS 모델을 이용하여 어플리케이션을 구성해보았습니다. CQRS 에서 고도화된 모델은 Event sourcing 까지 구현하여 사용하겠지만, 이 프로젝트에서는 어플리케이션 내부에서 Command 와 Query 를 분리 구현하는정도로 사용했습니다. 어플리케이션은 하나의 데이터베이스를 사용하며 Command 를 사용하여 도메인 모델에 접근하고, Query 를 사용하여 영속화되어 있는 데이터를 질의합니다. 

작성된 어플리케이션은 은행 계좌의 도메인에서 계좌 개설, 해지, 입금, 출금, 송금을 구현하였으며 이 중 몇가지 기능이 처리되는 시나리오를 통하여 어플리케이션의 전체적인 구조와 어떻게 동작하는지를 살펴보고자 합니다.

먼저 어플리케이션이 클라이언트의 계좌 생성요청을 처리하는 시나리오를 토대로 살펴보도록 하겠습니다. 이 시나리오에서 등장하는 클라이언트는 HTTP 를 사용하며 이 요청이 가장 먼저 마주하게 되는 것은 인터페이스 계층입니다. 인터페이스 계층에서는 클라이언트로부터 받은 요청을 검증하고 각 요청에 대해 어떻게 응답할지를 결정합니다. 또한, 요청에 맞는 Command 또는 Query 를 생성하여 각각의 맞는 Bus 로 전송하여 응용 계층에서 실행될 수 있도록 합니다. 

해당 어플리케이션을 이용하여 클라이언트가 계좌를 생성하려 합니다. 이 기능을 사용하기 위해서 클라이언트는 HTTP POST 요청을 `/accounts` 라는 경로로 요청하게 되며 해당 경로를 담당하는 AccountController 라는 컨트롤러가 요청을 처리하게 됩니다. 해당 컨트롤러는 `@Controller('/accounts')` 데코레이터로 어떤 경로의 요청을 받을것인지를 지정하고, 요청된 경로 앞에 데코레이터로 입력받은 경로와 일치하는 경로가 있다면 해당 요청을 컨트롤러가 가지고 있는 메소드를 통하여 처리합니다. 

컨트롤러는 요청된 경로가 뒤에 추가적인 경로, 요청된 메소드와 일치하는 데코레이터를 가진 메소드를 통하여 요청을 처리합니다. 지금 이 요청에서는 전체 요청 경로가 `'/accounts'` 로 추가적인 경로가 없고, HTTP POST 를 사용하고 있기 때문에 `@Post()` 데코레이터가 있는 `handleOpenAccountRequest` 라는 메소드를 사용하여 요청을 처리하게 되며 클라이언트가 전송한 바디 데이터는 `@Body()` 데코레이터를 이용하여 해당 컨트롤러 메소드의 파라미터로 전달됩니다. `OpenAccountBody` 에서는 요청된 값이 정상적인지 각 속성에 정의된 데코레이터를 사용하여 검증합니다. 

컨트롤러의 메소드는 요청으로 받은 값을 이용하여 각각의 요청에 맞는 Command, Query 객체를 생성하여 주입된 CommandBus 혹은 QueryBus 를 통하여 실행하고, 결과 값을 받아 클라이언트에게 응답합니다.

```typescript
// dto/open-account.body.dto.ts
import { ApiProperty } from '@nestjs/swagger';
import { IsString, MaxLength, MinLength } from 'class-validator';

export class OpenAccountBodyDTO {
  @IsString()
  @MinLength(2)
  @MaxLength(8)
  @ApiProperty({ minLength: 2, maxLength: 8, example: 'young' })
  readonly name: string;

  @IsString()
  @MinLength(8)
  @MaxLength(20)
  @ApiProperty({ minLength: 8, maxLength: 20, example: 'password' })
  readonly password: string;
}

```

```typescript
// interface/accounts.controller.ts
import {
  Body,
  Controller,
  Delete,
  Get,
  Param,
  Post,
  Put,
  Query,
} from '@nestjs/common';
import { CommandBus, QueryBus } from '@nestjs/cqrs';
import {
  ApiBadRequestResponse,
  ApiInternalServerErrorResponse,
  ApiNotFoundResponse,
  ApiResponse,
  ApiTags,
  ApiUnauthorizedResponse,
  ApiUnprocessableEntityResponse,
} from '@nestjs/swagger';

import { DepositBodyDTO } from 'src/accounts/interface/dto/deposit.body.dto';
import { FindAccountsQueryDTO } from 'src/accounts/interface/dto/find-accounts.query.dto';
import { OpenAccountBodyDTO } from 'src/accounts/interface/dto/open-account.body.dto';
import { UpdatePasswordBodyDTO } from 'src/accounts/interface/dto/update-password.body.dto';
import { WithdrawBodyDTO } from 'src/accounts/interface/dto/withdraw.body.dto';
import { RemitBodyDTO } from 'src/accounts/interface/dto/remit.body.dto';
import { WithdrawParamDTO } from 'src/accounts/interface/dto/withdraw.param.dto';
import { DepositParamDTO } from 'src/accounts/interface/dto/deposit.param.dto';
import { RemitParamDTO } from 'src/accounts/interface/dto/remit.param.dto';
import { UpdatePasswordParamDTO } from 'src/accounts/interface/dto/update-password.param.dto';
import { DeleteAccountParamDTO } from 'src/accounts/interface/dto/delete-account.param.dto';
import { DeleteAccountQueryDTO } from 'src/accounts/interface/dto/delete-account.query.dto';
import { FindAccountByIdParamDTO } from 'src/accounts/interface/dto/find-account-by-id.param.dto';
import { FindAccountByIdResponseDTO } from 'src/accounts/interface/dto/find-account-by-id.response.dto';
import { FindAccountsResponseDTO } from 'src/accounts/interface/dto/find-accounts.response.dto';
import { ResponseDescription } from 'src/accounts/interface/response-description';

import { CloseAccountCommand } from 'src/accounts/application/commands/close-account.command';
import { DepositCommand } from 'src/accounts/application/commands/deposit.command';
import { OpenAccountCommand } from 'src/accounts/application/commands/open-account.command';
import { UpdatePasswordCommand } from 'src/accounts/application/commands/update-password.command';
import { WithdrawCommand } from 'src/accounts/application/commands/withdraw.command';
import { FindAccountByIdQuery } from 'src/accounts/application/queries/find-account-by-id.query';
import { FindAccountsQuery } from 'src/accounts/application/queries/find-accounts.query';
import { RemitCommand } from 'src/accounts/application/commands/remit.command';

@ApiTags('Accounts')
@Controller('accounts')
export class AccountsController {
  constructor(readonly commandBus: CommandBus, readonly queryBus: QueryBus) {}

  @Post()
  @ApiResponse({ status: 201, description: ResponseDescription.CREATED })
  @ApiBadRequestResponse({ description: ResponseDescription.BAD_REQUEST })
  @ApiInternalServerErrorResponse({
    description: ResponseDescription.INTERNAL_SERVER_ERROR,
  })
  async openAccount(@Body() body: OpenAccountBodyDTO): Promise<void> {
    const command = new OpenAccountCommand(body.name, body.password);
    await this.commandBus.execute(command);
  }

  @Post('/:id/withdraw')
  @ApiResponse({ status: 201, description: ResponseDescription.CREATED })
  @ApiBadRequestResponse({ description: ResponseDescription.BAD_REQUEST })
  @ApiNotFoundResponse({ description: ResponseDescription.NOT_FOUND })
  @ApiUnauthorizedResponse({ description: ResponseDescription.UNAUTHORIZED })
  @ApiUnprocessableEntityResponse({
    description: ResponseDescription.UNPROCESSABLE_ENTITY,
  })
  @ApiInternalServerErrorResponse({
    description: ResponseDescription.INTERNAL_SERVER_ERROR,
  })
  async withdraw(
    @Param() param: WithdrawParamDTO,
    @Body() body: WithdrawBodyDTO,
  ): Promise<void> {
    const command = new WithdrawCommand({ ...param, ...body });
    await this.commandBus.execute(command);
  }

  @Post('/:id/deposit')
  @ApiResponse({ status: 201, description: ResponseDescription.CREATED })
  @ApiBadRequestResponse({ description: ResponseDescription.BAD_REQUEST })
  @ApiNotFoundResponse({ description: ResponseDescription.NOT_FOUND })
  @ApiInternalServerErrorResponse({
    description: ResponseDescription.INTERNAL_SERVER_ERROR,
  })
  async deposit(
    @Param() param: DepositParamDTO,
    @Body() body: DepositBodyDTO,
  ): Promise<void> {
    const command = new DepositCommand({ ...param, ...body });
    await this.commandBus.execute(command);
  }

  @Post('/:id/remit')
  @ApiResponse({ status: 201, description: ResponseDescription.CREATED })
  @ApiBadRequestResponse({ description: ResponseDescription.BAD_REQUEST })
  @ApiNotFoundResponse({ description: ResponseDescription.NOT_FOUND })
  @ApiUnauthorizedResponse({ description: ResponseDescription.UNAUTHORIZED })
  @ApiUnprocessableEntityResponse({
    description: ResponseDescription.UNPROCESSABLE_ENTITY,
  })
  @ApiInternalServerErrorResponse({
    description: ResponseDescription.INTERNAL_SERVER_ERROR,
  })
  async remit(
    @Param() param: RemitParamDTO,
    @Body() body: RemitBodyDTO,
  ): Promise<void> {
    const command = new RemitCommand({ ...param, ...body });
    await this.commandBus.execute(command);
  }

  @Put('/:id/password')
  @ApiResponse({ status: 200, description: ResponseDescription.OK })
  @ApiBadRequestResponse({ description: ResponseDescription.BAD_REQUEST })
  @ApiNotFoundResponse({ description: ResponseDescription.NOT_FOUND })
  @ApiUnauthorizedResponse({ description: ResponseDescription.UNAUTHORIZED })
  @ApiInternalServerErrorResponse({
    description: ResponseDescription.INTERNAL_SERVER_ERROR,
  })
  async updatePassword(
    @Param() param: UpdatePasswordParamDTO,
    @Body() body: UpdatePasswordBodyDTO,
  ): Promise<void> {
    const command = new UpdatePasswordCommand({ ...param, ...body });
    await this.commandBus.execute(command);
  }

  @Delete('/:id')
  @ApiResponse({ status: 200, description: ResponseDescription.OK })
  @ApiBadRequestResponse({ description: ResponseDescription.BAD_REQUEST })
  @ApiNotFoundResponse({ description: ResponseDescription.NOT_FOUND })
  @ApiUnauthorizedResponse({ description: ResponseDescription.UNAUTHORIZED })
  @ApiUnprocessableEntityResponse({
    description: ResponseDescription.UNPROCESSABLE_ENTITY,
  })
  @ApiInternalServerErrorResponse({
    description: ResponseDescription.INTERNAL_SERVER_ERROR,
  })
  async closeAccount(
    @Param() param: DeleteAccountParamDTO,
    @Query() query: DeleteAccountQueryDTO,
  ): Promise<void> {
    const command = new CloseAccountCommand(param.id, query.password);
    await this.commandBus.execute(command);
  }

  @Get()
  @ApiResponse({
    status: 200,
    description: ResponseDescription.OK,
    type: FindAccountsResponseDTO,
  })
  @ApiBadRequestResponse({ description: ResponseDescription.BAD_REQUEST })
  @ApiInternalServerErrorResponse({
    description: ResponseDescription.INTERNAL_SERVER_ERROR,
  })
  async findAccounts(
    @Query() queryDto: FindAccountsQueryDTO,
  ): Promise<FindAccountsResponseDTO> {
    const query = new FindAccountsQuery(queryDto.offset, queryDto.limit);
    return { accounts: await this.queryBus.execute(query) };
  }

  @Get('/:id')
  @ApiResponse({
    status: 200,
    description: ResponseDescription.OK,
    type: FindAccountByIdResponseDTO,
  })
  @ApiBadRequestResponse({ description: ResponseDescription.BAD_REQUEST })
  @ApiNotFoundResponse({ description: ResponseDescription.NOT_FOUND })
  @ApiInternalServerErrorResponse({
    description: ResponseDescription.INTERNAL_SERVER_ERROR,
  })
  async findAccountById(
    @Param() param: FindAccountByIdParamDTO,
  ): Promise<FindAccountByIdResponseDTO> {
    const query = new FindAccountByIdQuery(param.id);
    return this.queryBus.execute(query);
  }
}


```



