### 예외 필터
NestJS에 기본으로 내장된 전역 예외 필터는 내장 예외 필터가 인식할 수 없는 에러(HttpException 가 아니고 HttpException를 상속 받지도 않는 에러)를 InternalServerErrorException으로 변환한다. InternalServerError는 HttpException를 상속 받고 HttpException는 Error를 상속 받는다. 그 외 Nest에서 제공하는 모든 예외는 HttpException를 상속 받는다.

### HttpException
HttpException의 constructor는 `response: string | Record<string, any>` 와 `status: number` 를 인자로 받는다. 이는 각각 JSON 응답의 본문과 HTTP 상태 코드이다.

### ExceptionFilter 정의
```typescript
import { ArgumentsHost, Catch, ExceptionFilter, HttpException, InternalServerErrorException } from '@nestjs/common';
import { Request, Response } from 'express';

@Catch() // 처리되지 않은 모든 예외를 잡을 때 사용된다.
export class HttpExceptionFilter implements ExceptionFilter {
  catch(exception: Error, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const res = ctx.getResponse<Response>();
    const req = ctx.getRequest<Request>();
    
    // Nest에서 제공하는 에러는 모두 HttpException를 상속받는다.
    // 그렇지 않을 경우 알수 없는 에러이므로 500 에러 처리
	if (!(exception instanceof HttpException)) { 
	  exception = new InternalServerErrorException();
	}

	const response = (exception as HttpException).getResponse();

	const log = {
	  // 로그로 출력할 내용
	  timestamp: new Date(),
	  url: req.url,
	  response,
	}

	console.log(log);

	res.status((exception as HttpException).getStatus()).json(response);
  }
}
```

### ExceptionFilter 적용
@UseFilters: 예외필터를 인자로 받는다. 컨트롤러(class)에 사용할 경우 컨트롤러 전체에 예외 필터를 적용하고 엔드포인트(method)에 적용할 경우 해당 엔드포인트에 예외 필터를 적용한다.
##### 전역에 ExceptionFilter 적용
```typescript
async function bootstrap() {
  const app = await NestFactory.creaete(AppModule);
  app.useGlobalFilters(new HttpExceptionFilter()); // 전역 필터 적용
  await app.listen(3000);
}
```
전역 필터는 기본적으로 의존석을 주입할 수 없지만, 전역 필터를 커스텀 프로바이더로 등록하게 되면 다른 프로바이더를 주입 받아서 사용할 수 있다.
