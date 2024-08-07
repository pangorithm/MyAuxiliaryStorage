### Controller
MVC 패턴의 컨트롤러를 의미한다.

##### 클래스 데커레이터
* @Controller: 해당 클래스가 컨트롤러 역할을 하게 된다. 인수로 ControllerPotions 객체를 받는다. 이때 host 속성을 통해 해당 컨트롤러가 요청을 받을 하위 도메인을 설정 할 수 있다.
* @Module: 인자로 객체를 받으며 controllers 속성으로 Controller 객체 배열을 받아서 어떤 컨트롤러가 먼저 처리되게 할지 지정할 수 있다.

##### 메서드 데커레이터
* @Get, @Post 등: 데커레이터 인수에 경로를 넣어주면 라우팅 패스를 설정할 수 있다(기본 값은 / ). 또한 라우팅 패스는 와일드카드(* , _ , ? , + 등)를 사용할 수 있다.
* @HttpCode: 요청이 정상적으로 처리되었을 때의 상태 코드 오버라이드
* @Header: 헤더 이름과 값을 인수로 받아 커스텀 헤더를 추가할 수 있다.
* @Redirect: 인수로 경로를 입력하면, 요청 처리 후 해당 경로로 클라이언트를 리디렉션한다. (두번째 인자로 300번대의 Redirect 응답 코드를 입력한다)

##### 매개변수 데커레이터
* @Req: 원본 HTTP 요청 객체를 바인딩해준다.
* @Res: 원본 HTTP 응답 객체를 바인딩 해준다.
* @Query: 쿼리 스트링을 객체로 바인딩해준다. 인자를 넣으면 쿼리의 해당 인자의 값을 바인딩해준다.
* @Body: request의 body를 바인딩해준다.
* @Param: 경로 매개변수(경로에 :를 사용해서 정의)를 객체로 바인딩해준다. 인자를 넣으면 해당 매개변수의 값을 바인딩해준다.
* @HostParm: @Controller에 설정된 서브_도메인을 변수로 받을 수 있다.

##### 올바르지 않은 요청일 경우의 예외처리
```TypeScript
throw new HttpException({
	# 클라이언트에 응답할 데이터
	status: HttpStatus.해당하는_오류_코드,
	error:'에러 메세지'
}, HttpStatus.해당하는_오류_코드 # HTTP 응답 자체의 상태 코드);
```

##### DTO (Data Transfer Object)
@Query, @Body 등의 타입으로 설정하여 데이터 전달을 위해 사용하고 다음과 같은 특징이 있다,
- **단순한 데이터 구조**: DTO는 보통 단순한 데이터 구조를 가지며, 주로 getter와 setter 메서드를 통해 데이터를 접근한다. 일반적으로 비즈니스 로직을 포함하지 않는다.
- **계층 간 데이터 전달**: 주로 서비스 계층에서 데이터 액세스 계층 또는 프레젠테이션 계층으로 데이터를 전달하는 데 사용된다.
- **데이터의 직렬화와 역직렬화**: DTO는 데이터의 직렬화와 역직렬화에 용이하다. 이는 네트워크를 통해 데이터를 전송하거나, API 응답으로 데이터를 반환할 때 유용하다.
- **안전성과 일관성**: DTO를 사용하면, 특정 계층에서 필요한 데이터만을 노출하여 데이터의 안전성과 일관성을 유지할 수 있다.
추가적으로 DTO 클래스를 정의할 때 속성 파라미터를 사용하여 유효성 검사 등의 작업을 자동화 할 수 있다.

### 커스텀 매개변수 데커레이터
nestJS의 함수를 활용해 손쉽게 커스텀 파라미터 데커레이터를 만들 수 있다.
###### 정의
```typescript
import { createParamDecorator, ExcutionContext } from '@nestjs/common';

export const MyObj = createParamDecorator(
  (data: unknown, ctx: ExcutionContext) => { // data: 데커레이터에 인수로 넘기는 값
    const request = ctx.switchToHttp().getRequest();
    return request.myObj;
  }
)

```

###### 사용
```typescript
interface MyObj {
  // 속성 생략
}

@Controller()
export class AppController {
  
  @Get()
  getHello(@MyObj() myObj: MyObj) {
    console.log(myObj);
  }
}
```