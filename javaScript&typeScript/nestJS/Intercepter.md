인터셉터는 요청과 응답을 가로채서 변형을 가할 수 있는 컴포넌트이며 AOP(관점 지향 프로그래밍)의 영향을 많이 받았다.

### 인터셉터의 역할
* 메서드 실행 전/후 추가 로직 바인딩
* 함수에서 반환된 결과를 변환
* 함수에서 던져진 예외를 변환
* 기본 기능의 동작을 확장
* 특정 조건에 따라 기능을 완전히 재정의
인터셉터의 역할은 미들웨어와 비슷하지만, 미들웨어는 요청이 라우트 핸들러로 전달되기 전에 동작하는 반면, 인터셉터는 요청에 대한 라우트 핸들러의 처리 전/후에 처리되어 요청과 응답을 모두 다룰 수 있다.
### NestInterceptor의 정의
```typescript
export interface NestInterceptor<T = any, R = any> {
  intercept(context: ExcutionContext, next: CallHandler<T>): Observable<R> | Promise<Observable<R>>;
}

export interfaceCallHandler<T = any> {
  handle(): Observable<T>;
}
```

### CustomInterceptor
```typescript
import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';
import { Observable } from 'rxjs';
import { tap }  from 'rxjs/operators';

@Injectable
export class LoggingInterceptor implements NestInterceptor {
  intercept(context: ExcutionContext, next: CallHandler<T>): Observable<R> {
    // 요청이 전달되기전 수행할 로직 작성
	console.log('before');

	return next
			.handle()
			.pipe(
			  tap(() => console.log('After...')), // 요청을 처리한 후 수행할 로직 작성
			);

  }
}

```

### 인터셉터의 적용
* @UseInterceptors(): 컨트롤러나 엔드포인트에 인터셉터를 적용한다.
* useGlobalInterceptors(): bootstrap과정에서 등록하여 전역으로 인터셉터를 적용한다.
