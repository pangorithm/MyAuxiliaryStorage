[데코레이터 설명 in 타입스크립트 가이드](https://www.typescriptlang.org/ko/docs/handbook/decorators.html)

### 데코레이터란?
파이썬의 데커레이터나 자바의 애너테이션과 유사한 기능을 한다.  
클래스, 메서드, 접근자, 프로퍼티, 매개 변수에 적용 가능하다.  
각 요소의 선언부 앞에 @로 시작하는 데커레이터를 선언하면 데커레이터로 구현된 코드를 함께 실행한다.  

### 데커레이터의 합성
데커레이터는 수학의 함수의 합성과 같이 동작한다.

**소스코드**
```typescript
function first() {
  console.log("first(): factory evaluated");
  return function (target: any, propertyKey: string, descriptor: PropertyDescriptor) {
    console.log("first(): called");
  }
}

function second() {
  console.log("second(): factory evaluated");
  return function (target: any, propertyKey: string, descriptor: PropertyDescriptor) {
    console.log("second(): called");
  }
}

class ExampleClass {
  @first
  @second
  method() {
    console.log("method is called");
  }
}
```
**실행순서**
```
first(): factory evaluated
second(): factory evaluated
second(): called
first(): called
method is called
```

### 클래스 데커레이터
```typeScript
function reportableClassDecorator<T extends { new (...args: any[]): {} }>(constructor: T) {
  return class extends constructor {
    reportingURL = "http://www...";
  };
}

@reportableClassDecorator
class BugReport {
  type = "report";
  title: string;

  constructor(t: string) {
    this.title = t;
  }
}

const bug = new BugReport("Needs dark mode");
console.log(bug.title); // Prints "Needs dark mode"
console.log(bug.type); // Prints "report"

// 클래스 타입이 변경된 것은 아니기 때문에 reportingURL을 직접 호출 할 수는 없다.
// Note that the decorator *does not* change the TypeScript type
// and so the new property `reportingURL` is not known
// to the type system:
console.log(bug.reportingURL);
// prints "Property 'reportingURL' does not exist on type 'BugReport'."

console.log(bug);
// prints {type: "report", title: "Needs dark mode", reportingURL: "http://www..."}
```

### 메서드 데커레이터
target: 정적 멤버가 속한 클래스의 생성자 함수 또는 인스턴스 멤머에 대한 클래스의 프로토타입  
propertyKey: 멤버의 이름  
descriptor: 멤버의 속성 설명자  
[PropertyDescriptor란?](https://developer.mozilla.org/ko/docs/Web/JavaScript/Reference/Global_Objects/Object/getOwnPropertyDescriptors#%EC%84%A4%EB%AA%85)
```typescript
class Greeter {
  greeting: string;
  constructor(message: string) {
    this.greeting = message;
  }

  @enumerable(false)
  greet() {
    return "Hello, " + this.greeting;
  }
}

function enumerable(value: boolean) {
  return function (target: any, propertyKey: string, descriptor: PropertyDescriptor) {
    descriptor.enumerable = value;
  };
}
```

### 접근자 데커레이터
```typescript
class Point {
  private _x: number;
  private _y: number;
  constructor(x: number, y: number) {
    this._x = x;
    this._y = y;
  }

  @configurable(false)
  get x() {
    return this._x;
  }

  @configurable(false)
  get y() {
    return this._y;
  }
}

function configurable(value: boolean) {
  return function (target: any, propertyKey: string, descriptor: PropertyDescriptor) {
    descriptor.configurable = value;
  };
}
```

### 속성 데커레이터
target: 정적 멤버가 속한 클래스의 생성자 함수 또는 인스턴스 멤버에 대한 클래스의 프로토타입  
propertyKey: 멤버의 이름

```typescript
function format(formatString: string) {
  return function (target: any, propertyKey: string): any {
    let value = target[propertyKey];

    function getter() {
      return `${formatString} ${value}`
    }

    function setter(newVal: string) {
      value = newVal;
    }

    return {
      get: getter,
      set: setter,
      enumerable: true,
      configurable: true,
    }
  }
}

class Greeter {
  @format("Hello")
  greeting: string;
}

const t = new Greeter();
t.greeting = "World";
console.log(t.greeting);
// prints "Hello World"
```
또는

[메타데이터란?](https://www.typescriptlang.org/ko/docs/handbook/decorators.html#%EB%A9%94%ED%83%80%EB%8D%B0%EC%9D%B4%ED%84%B0-metadata)
```typescript
import "reflect-metadata";

const formatMetadataKey = Symbol("format");

function format(formatString: string) {
  return Reflect.metadata(formatMetadataKey, formatString);
}

function getFormat(target: any, propertyKey: string) {
  return Reflect.getMetadata(formatMetadataKey, target, propertyKey);
}

class Greeter {
  @format("Hello, %s")
  greeting: string;
  constructor(message: string) {
    this.greeting = message;
  }
  greet() {
    let formatString = getFormat(this, "greeting");
    return formatString.replace("%s", this.greeting);
  }
}
```

### 매개변수 데커레이터
target: 정적 멤버가 속한 클래스의 생성자 함수 또는 인스턴스 멤버에 대한 클래스의 프로토타입  
propertyKey: 멤버의 이름  
parameterIndex: 매개변수가 함수에서 몇번째 위치에 선언되었는지를 나타내는 인덱스  
```typescript
import { BadRequestException } from "@nestjs/common";

function MinLength(min: number) {
  return function (target: any, propertyKey: string, parameterIndex: number) {
    target.validators = {
      minLength: function (args: string[]) {
        return args[parameterIndex].length >= min;
      }
    }
  }
}

function Validate(target: any, propertyKey: string, descriptor: PropertyDescriptor) {
  const method = descriptor.value;

  descriptor.value = function(...args) {
    Object.keys(target.validators).forEach(key => {
      if (!target.validators[key](args)) {
        throw new BadRequestException();
      }
    })
    method.apply(this, args);
  }
}

class User {
  private name: string;

  @Validate
  setName(@MinLength(3) name: string) {
    this.name = name;
  }
}

const t = new User();
t.setName("Dexter");
t.setName("De"); // error: BadRequestException
```

### 데커레이터 요약

|데커레이터|역할|호출 시 전달되는 인수|선언 불가능한 위치|
|---|---|---|---|
|클래스 데커레이터|클래스의 정의 r, u|constructor|d.ts 파일, declare 클래스|
|메서드 데커레이터|메서드의 정의 r, u|target, propertyKey, propertyDescriptor|d.ts 파일, declare 클래스, 오버로드 메서드|
|접근자 데커레이터|접근자의 정의 r, u|target, propertyKey, propertyDescriptor|d.ts 파일, declare 클래스|
|속성 데커레이터|속성의 정의 r|target, propertyKey|d.ts 파일, declare 클래스|
|매개변수 데커레이터|매개변수의 정의 r|target, propertyKey, parameterIndex|d.ts 파일, declare 클래스|

### 데코레이터 평가 및 적용 순서
1. 메서드, 접근자 또는 프로퍼티 데코레이터가 다음에 오는 매개 변수 데코레이터는 각 인스턴스 멤버에 적용된다.
2. 메서드, 접근자 또는 프로퍼티 데코레이터가 다음에 오는 매개 변수 데코레이터는 각 정적 멤버에 적용된다.
3. 매개 변수 데코레이터는 생성자에 적용된다.
4. 클래스 데코레이터는 클래스에 적용된다.

**요약**  
평가 순서: 클래스 -> 속성/접근자/메서드/생성자 -> 매개변수  
적용 순서: 매개변수 -> 속성/접근자/메서드(인스턴스 멤버 -> 정적 멤버) -> 생성자 -> 클래스  

속성/접근자/메서드 데코레이터의 호출 순서는 코드에서 나타나는 순서에 따라 달라진다.