# Command Query Responsibility Separation
명령과 조회를 분리하여 성능과 확장성 및 보안성을 높이도록 도와주는 아키텍처 패턴
* CQRS를 사용할 경우 복잡성이 추가되어 생산성이 감소하기 때문에, 모델을 공유하는 것이 도메인을 다루기 더 쉬운지 판단해야 한다. 따라서 시스템 전체가 아니라 bounded context(독립된 도메인 영역 내)에서만 사용해야 한다.
* 읽기 및 쓰기 작업에서 로드를 분리하여 각각 독립적으로 확장 및 최적화 할 수 있다.
* CRUD를 사용하는 단일 표현에서 작업 기반 UI로 쉽게 이동 가능하다.
* 이벤트 기반 프로그래밍 모델과 어울린다.
* 최종 일관성을 사용할 가능성이 높아진다
* EagerReadDerivation을 사용하여 쿼리 측 모델을 단순화 하는 것이 합리적일 수 있다.
* 쓰기 모델이 모든 업데이트에 대한 이벤트를 생성하는 경우, 읽기 모델을 별도로 구성하여 과도한 데이터베이스 상호작용을 피할 수 있다.
* DDD(Domain-Driven Design)를 적용하는데 적합하다.

### NestJS에서 제공하는 CQRS 모듈 사용하기
`npm install @nestjs/cqrs`

```typescript
import { CqrsModule } from '@nestjs/cqrs';

@Module({
  imports: [CqrsModule],
  // 생략
})
export class UsersModule { }
```

##### @nestjs/cqrs에서 제공하는 버스 종류
1. **Command Bus**: 커멘드를 처리하고 전달하는 역할
2. **Event Bus**: 이벤트를 퍼블리시하고 구독자들에게 전달하는 역할
3. **Query Bus**: 쿼리를 처리하고 결과를 반환하는 역할

### Command
```typescript
import { ICommand } from '@nestjs/cqrs';

export class CreateUserCommand implements ICommand {
  constructor(
    readonly name: string,
    readonly email: string,
    readonly password: string,
  ){ }
}
```

```typescript
import { CommandHandler, ICommandHandler } from '@nestjs/cqrs';
import { CreateUserCommand } from './create-user.command';

@CommandHandler(CreateUserCommand)
export class CreateUserHandler implements ICommandHandler<CreateUserCommand> {
  async execute(command: CreateUserCommand) {
    const { name, email, password } = command

    // 기존에 Service 프로바이더에서 수행하던 로직 작성
  }

}
```

```typescript
import { CqrsModule } from '@nestjs/cqrs';

@Module({
  // 생략
  providers: [CreateUserHandler],
})
export class UsersModule { }
```

```typescript
import { CommnadBus } from '@nestjs/cqrs';

@Controller('users')
export class UsersController {
  constructor(
    private commandBus: CommandBus,
  )
}

@Post()
async createUser(@Body() dto: CreasteUserDto): Promise<void> {
  const { name, email, password } = dto;

  const command = new CreateUserCommand(name, email, password);

  return this.command.excute(command);
}
```

### Event
```typescript
import { CommandHandler, ICommandHandler } from '@nestjs/cqrs';
import { CreateUserCommand } from './create-user.command';

@Injectable
@CommandHandler(CreateUserCommand)
export class CreateUserHandler implements ICommandHandler<CreateUserCommand> {
  async execute(command: CreateUserCommand) {
    const { name, email, password } = command

    // 기존에 Service 프로바이더에서 수행하던 로직 작성

	// command 실행시 이벤트 추가
	this.eventBus.publish(new UserCreatedEvent(email, signupVerifiyToken));
	this.eventBus.publish(new TestEvent());
  }

}

```

```typescript
export abstract class UsersEvent implements IEvent{
  constructor(readonly name: string) { }
}
```

```typescript
import { IEvent } from '@nestjs/cqrs';
import { UsersEvent } from './usersEvent';

export class UserCreatedEvent extends UsersEvent {
  constructor(
    readonly email: string,
    readonly signupVerifiyToken: string,
  ){
    super(UserCreatedEvent.name);
  }
}
```

```typescript
import { IEvent } from '@nestjs/cqrs';
import { UsersEvent } from './usersEvent';

export class TestEvent extends UsersEvent {
  constructor(
  ){
    super(TestEvent.name);
  }
}
```

```typescript
import { EventsHandler, IEventHandler } from '@nestjs/cqrs';
import { EmailService } from 'src/email/email.service';
import { UserCreatedEvent } from './user-created.event';
import { TestEvent } from './test.event';

// 이벤트 핸들러는 커멘드 핸들러와는 달리 여러 이벤트를 같은 핸들러가 받을 수 있다.
@EventsHandler(UserCreatedEvent, TestEvent)
export class UserEventsHandler implements IEventHandler<UsersEvent> {
  constructor(
    private emailService: EmailService;
  ){ }

  async handle(event: UsersEvent) {
    switch (event.name) {
      case UserCreatedEvent.name: {
        console.log('UserCreatedEvent!');
        const { email, signupVerifiyToken } = event;
        await this.emailService.sendMemberJoinverification(email, signupVerifiyToken);
        break;
      }
      case TestEvent.name: {
        console.log('TestEvent!');
        break;      
      }
      default:
        break;
    }
  }
}
```

```typescript
import { CqrsModule } from '@nestjs/cqrs';

@Module({
  // 생략
  providers: [UserEventsHandler],
})
export class UsersModule { }
```

### Query
```typescript
import { IQuery } from '@nestjs/cqrs';

export class GetUserInfoQuery implements IQuery {
  constructor(
    readonly userId: string,
  ) { }
}
```

```typescript
import { NotFoundException } from '@nestjs/common';
import { IQueryHandler, QueryHandler } from '@nestjs/cqrs';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { UserEntitiy } from '../entity/user.entity';
import { UserInfo } from '../UserInfo';
import { GetUserInfoQuery} from './get-user-info.query';

@QueryHandler(GetUserInfoQuery)
export class GetUserInfoQueryHandler implements IQueryHandler<GetUserInfoQuery> {
  constructor(
    @InjectRepository(UserEntity) private usersRepository: Reporitory<UserEntity>,
  ) { }

  async execute(query: GetUserInfoQuery): Promise<UserInfo> {
    const { userId } = query;
    const user = await this.usersReporitory.findOne({
      where: { id: userId }
    });
     
    if (!user) {
      throw new NotFoundException('User does not exist');
    }

    return {
      id: user.id,
      name: user.name,
      email: user.email,
    }
  }
}
```

```typescript
import { QueryBus } from '@nestjs/cqrs';
import { GetUserInfoQuery} from './get-user-info.query';

@Controller('users')
export class UsersController {
  constructor(
    private queryBus: QueryBus,
  ) { }
  
  // 생략
  
  @UseGuards(AuthGuard)
  @Get('id')
  async getUserInfo(@Param('id') userId: string): Promise<UserInfo> {
    const getUserInfoQuery = new GetUserInfoQuery(userId);
    
    return this.queryBus.execute(getUserInfoQuery);
  }
}
```