들어오는 요청 데이터를 변환하거나 검증하는 데 사용되는 클래스이다. 파라미터 레벨에서 동작하며 데커레이터의 인자로 입력해서 파라미터 단위, 메서드 단위, 컨트롤러 단위, 전역 단위로 적용할 수 있다.

### 파이프의 내부 구현 이해
##### 커스텀 파이브
```typescript
import { PripeTransform, injectable, ArgumentMetadata } from '@nestjs/common';

@Injectable
export class customPipe implements PipeTransform {
  transform(value: any, metadata: ArgunmetMetadata) {
    // 검증, 변환 등을 수행할 로직을 수행한다.
    if(!isValid(value)){
      throw new Exception('Validation failed');
    }
    return value; // 변환할 경우 변환한 값을 return 해준다. 
  }
}
```

##### 사용한 인터페이스에 대한 설명
```typescript
export interface PipeTransform<T = any, R = any> {
  // value: 현재 파이프에 전달된 인수, metadata: 현재 파이프에 전달된 인수의 메타데이터
  transform(value: T, metadata: ArgunmetMetadata): R;
}

export declare type Paramtype = 'body' | 'guery' | 'param' | 'custom';

export interface ArgumentMetadata {
  readonly type: Paramtype; // 파이프에 전달된 인수의 유형
  readonly metatype?: Type<any> | undefined; // 라우트 핸들러에 정의된 인수의 타입
  readonly data?: string | undefined; // 매개변수의 이름
}
```

##### 유용한 라이브러리
* class-transformer: 인스턴스 객체와 순수 객체 상호간으로 변환시킨다.
	* plainToClass(클래스, 순수_객체) : 순수 객체를 인스턴스 객체로 변환한다.
	* classToPlain(인스턴스_객체) : 인스턴스 객체를 순수 객체로 변환한다.
	* @Transform: TransformFnParams을 인자로 하는 콜백함수를 인자로 받는 **속성 데커레이터**이다. 
		* TransformFnParams의 인자
			* value: any; // 변환할 값 (입력값) 
			* key: string | undefined; // 변환 대상 속성의 이름 
			* obj: any; // 해당 속성의 클래스의 인스턴스 
			* type: TransformOperationType; // 변환 유형 (serialization 또는 deserialization) 
			* options: TransformOptions; // 변환 옵션
* class-validator: class의 속성에 검증 데코레이터를 사용하고 validate(인스턴스_객체) 메서드를 사용해서 해당 인스턴스 객체가 class에 선언한 데코레이터를 준수하는지 검증하여 검증에 실패한 속성의 ValidationError 배열을 반환한다.
[nestJS Validation 튜토리얼 문서](https://docs.nestjs.com/techniques/validation)
[class-validator 깃허브 페이지](https://github.com/typestack/class-validator)

### 인증 vs 인가
* 인증: 유저나 디바이스의 신원을 증명하는 행위
* 인가: 유저나 디바이스에게 접근 권한을 부여하거나 거부하는 행위
* 인증은 인가 의사결정의 한 요소가 될 수 있다.
* 인가 가공물(토큰)로 유저나 디바이스의 신원을 파악하는 방법은 유용하지 않다.