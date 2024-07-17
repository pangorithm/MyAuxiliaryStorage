nest CLI: nest 관련 cli 명령어 제공
npm i -g @nestjs/cli
```bash
nest -h # nest cli에서 사용 가능한 단축어 확인
```

### 데커레이터의 합성
여러개의 데커레이터를 하나의 데커레이터로 합성하는 법
```typescript
import { applyDecorators } from '@nestjs/common';

export function CompositionDecorator(...roles: Rolle[]) {
  return applyDecorators(
    // 합성할 데커레이터를 인수들로 넣어준다.
    // 데커레이터 실행 순서는 인자로 넣은 순서와 같다. 
    Decorator1('roles', roles),
    Decorator2(),
    Decorator3({ description: 'Unauthorized'}),
  );
}
```


### 메타데이터와 Reflection 클래스
빌드 타임에 선언한 메타데이터를 활용하여 런타임에 동작을 제어할 수 있다.

##### @SetMetadata
[SetMetadata 정의](https://github.com/nestjs/nest/blob/master/packages/common/decorators/core/set-metadata.decorator.ts)
```typescript
import { SetMetadata } from '@nestjs/common';

@Post()
@SetMetadata('roles', ['admin']) // 인자로 메타데이터의 key와 value 배열을 전달한다.
create(@Body() createUserDto) {
  return this.userService.create(createUserDto);
}
```

```typescript
import { SetMetadata } from '@nestjs/common'; 

// SetMetadata를 직접 사용하지 않고 커스텀 데커레이터로 래핑해서 사용하는 것이 의미 전달에 더 효과적이다.
export const Roles = (...roles: string[]) => SetMetadata('roles', roles);
```

##### Reflector
```typescript
import { Reflector } from '@nestjs/core';
import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';
import { Observable } from 'rxjs';

@Injectable()
export class HandlerRolesGuard implements CanActivate {
  constructor(private reflector: Reflector) { }

  canActivate(
    context: ExecutionContext,
  ): boolean | Promise<boolean> | Observable<boolean> {
    // 기타 코드 생략
    
    // 핸들러(method)에 메타데이터를 정의했을 경우
    const roles = this.reflector.get<string[]>('roles', context.getHandler()); 
    // 컨트롤러(class)에 메타데이터를 정의했을 경우
    const roles = this.reflector.get<string[]>('roles', context.getClass());
    
    // 기타 코드 생략
  }
}
```

### 요청 생명주기
![nestjs-request-life-cycle.png](https://pangorithm.github.io/MyAuxiliaryStorage/image/nestjs-request-life-cycle.png)  
요청을 받을 때는 전역->컨트롤러->라우터 순서로 컴포넌트가 적용되지만 응답을 내보낼때는 반대로 라우터->컨트롤러->전역 순서로 컴포넌트가 적용된다