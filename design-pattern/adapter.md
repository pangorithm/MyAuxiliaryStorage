### 목적

호환성이 없는 인터페이스를 가진 클래스들이 함께 작동할 수 있도록 해주는 것

### 구성요소

**어댑티(Adaptee) 클래스 -> **어댑터(Adapter) 클래스 -> **타깃(Target) 인터페이스******

어댑티 클래스를 어댑터 클래스로 래핑하여 타깃 인터페이스의 형태로 반환한다.

### 종류

1.  **클래스 어댑터 패턴**: 상속을 사용하여 어댑티의 기능을 타깃 인터페이스에 맞게 확장한다. 주로 다중 상속을 지원하는 언어에서 사용된다.
2.  **객체 어댑터 패턴**: 구성을 사용하여 어댑티 객체를 어댑터 클래스 내부에 보유함으로써 작동한다. 비교적 더 유연하며, 대부분의 프로그래밍 언어에서 사용할 수 있다.

### nest.js에서 사용되는 express.js 어댑터 코드 

[https://github.com/nestjs/nest/blob/master/packages/platform-express/adapters/express-adapter.ts](https://github.com/nestjs/nest/blob/master/packages/platform-express/adapters/express-adapter.ts)