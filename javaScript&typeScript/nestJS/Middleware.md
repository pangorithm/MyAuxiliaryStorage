### 미들웨어
##### 미들웨어의 특징
* 어떤 형태의 코드라도 수행할 수 있다.
* 요청과 응답에 변형을 가할 수 있다.
* 요청/응답 주기를 끝낼 수 있다.
* 여러 개의 미들웨어를 사용한다면 next()로 호출 스택상 다음 미들웨어에 제어권을 전달한다.
##### 미들웨어를 활용한 작업
* 쿠키 파싱: 헤더의 쿠키를 파싱하여, 사용하기 쉬운 데이터 구조로 변경한다.
* 세션 관리: 다른 핸들러가 세션 객체를 이용할 수 있게 한다.
* 인증/인가: 사용자가 서비스에 접근 가능한 권한이 있는지 확인한다.(nest는 인가 구현시 Guard를 이용하도록 권장한다.)
* 본문 파싱: 요청으로 들어오는 JSON, file stream 등을 읽고 해석한 다음 매개변수에 넣는 작업을 한다.

### NestJS에 미들웨어 적용하기
1. 미들웨어 구현하기
   ```typescript
   import { Injectable, NestMiddleware } from '@nestjs/common';
   import { Request, Response, NextFunctoin } from 'express';

   @Injectable()
   export class CustomMiddleware implements NestMiddleware {
     use(req: Request, res: Response, next: NextFunctoin) {
	   // 미들웨어에서 수행할 로직
	   next(); // next를 실행하지 않을 경우 해당 요청은 더이상 진행되지 않는다.
     }
   }
	```

2. 모듈에 미들웨어 설정하기
   ```typescript
   import { MiddlewareConsumer, Module, NestModule } from '@nestjs/common';

   @Module({
     imports: [사용할 모듈...],
   })
   export class AppModule implements NestModule {
     configure(consumer: MiddlewareConsumer): any {
       consumer
         .apply(미들웨어1, 미들웨어2, ...) // 적용할 미들웨어
         .exclude('/요청경로', RouteInfo인스턴스 ) // 미들웨어를 사용하지 않을 요청
         .forRoutes('/요청경로', 컨트롤러클래스, RouteInfo인스턴스); // 미들웨어를 사용할 요청
     }
   }
	```

3. 미들웨어를 전역으로 적용하기(main.ts 파일)
   ```typescript
   async function bootstrap(){
     const app = await NestFactory.create(AppModule);
     app.use(전역_적용할_미들웨어);
     await app.listen(3000);
   }
   bootstrap();
	```