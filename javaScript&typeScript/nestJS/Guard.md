- **핸들러 전 실행**: Guard는 라우트 핸들러가 실행되기 전에 실행된다.
- **요청 승인 또는 거부**: Guard는 요청을 계속 진행할지 중단할지를 결정한다.
- **구현 인터페이스**: Guard는 `CanActivate` 인터페이스를 구현하여 사용된다.

### 가드의 구현
```typescript
import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';
import { Observable } from 'rxjs';

@Injectable()
export class CustomGuard implements CanActivate {
  canActivate(
    context: ExecutionContext,
  ): boolean | Promise<boolean> | Observable<boolean> {
    const request = context.switchToHttp().getRequest();
    return this.validateRequest(request);
  }

  // 검증하는 로직 구현
  private validateRequest(request: any): boolean {
    // 예를 들어, 토큰을 확인하거나 사용자 세션을 확인할 수 있다.
	if(!검증조건){
	  return false;
	}
    return true;
  }
}

```


### 가드의 적용
```typescript
import { Controller, Get, UseGuards } from '@nestjs/common';
import { CustomGuard } from './custom.guard';

@UseGuards(CustomGuard) // 컨트롤러 범위에 가드를 적용할 경우
@Controller()
export class AppController {
  constructor(private readonly appService: AppService) { }

  @UseGuards(CustomGuard) // 메서드 범위에 가드를 적용할 경우
  @Get()
  getHello() {
    return this.appService.getHello();
  }
}

```

##### 가드를 전역으로 적용할 경우
```typescript
async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.useGlobalGuards(new CustomGuard());
  await app.listen(3000);
}
bootstrap();
```


### 가드에 다른 프로바이더 주입하기
가드에 종속성 주입을 사용해서 다른 프로바이더를 주입하기 위해서는, 해당 가드를 커스텀 프로바이더로 선언해야 한다.
```typescript
// app.module.ts
import { Module } from '@nestjs/common';
import { APP_GUARD } from '@nestjs/core';

@Module({
  providers: [
    {
      provide: APP_GUARD,
      useClass: CustomGuard,
    },
  ],
})
export class AppModule {}
```