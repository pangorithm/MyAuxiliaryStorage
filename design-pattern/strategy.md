### 목적 

알고리즘의 군을 정의하고, 각각을 하나의 클래스로 캡슐화하여, 이들이 상호 교체 가능하도록 만드는 디자인 패턴. 전략 패턴을 사용하면 알고리즘을 사용하는 클라이언트와 독립적으로 알고리즘을 변경할 수 있다. 이 패턴은 객체의 행동을 변경해야 할 때 직접 객체의 클래스를 변경하지 않고도, 객체가 사용하는 전략(즉, 메서드)을 변경함으로써 객체의 행동을 유연하게 확장할 수 있는 방법을 제공한다.

### 구성 요소

1.  **Context (문맥)**: 전략 패턴의 사용자로, 필요에 따라 구체적인 전략을 변경하여 사용할 수 있는 클래스. 이 클래스는 자신의 기능 일부를 전략에 위임한다.
2.  **Strategy (전략)**: 여러 알고리즘을 동일한 인터페이스 아래에 정의하는 인터페이스입니다. Context는 이 인터페이스를 통해 알고리즘을 호출한다.
3.  **ConcreteStrategy (구체적인 전략)**: Strategy 인터페이스를 구현하는 클래스로, 구체적인 알고리즘을 포함다.

### 장점

-   알고리즘을 사용하는 클라이언트와 알고리즘을 분리하여, 알고리즘의 교체가 용이하다.
-   동일 계열의 알고리즘을 쉽게 추가하거나 변경할 수 있다.
-   조건문 대신 전략을 사용함으로써, 코드의 유연성과 재사용성을 향상시킨다.

### 단점

-   애플리케이션에 전략 클래스가 많아지면 클래스가 과도하게 증가할 수 있다.
-   클라이언트가 ConcreteStrategy 객체를 생성하고, Context에 주입해주어야 하므로, 클라이언트 코드가 복잡해질 수 있다.

### 타입스크립트 예시코드

```
// Strategy Interface
interface LogStrategy {
  log(message: string): void;
}

// Concrete Strategies
class ConsoleLogStrategy implements LogStrategy {
  log(message: string): void {
    console.log(`ConsoleLogStrategy: ${message}`);
  }
}

class FileLogStrategy implements LogStrategy {
  log(message: string): void {
    // 여기서는 실제 파일 로깅 로직을 구현한다고 가정합니다.
    console.log(`FileLogStrategy: ${message}`);
  }
}

class NetworkLogStrategy implements LogStrategy {
  log(message: string): void {
    // 네트워크를 통해 로그 메시지를 전송하는 로직을 구현한다고 가정합니다.
    console.log(`NetworkLogStrategy: ${message}`);
  }
}

// Context
class Logger {
  private logStrategy: LogStrategy;

  constructor(logStrategy: LogStrategy) {
    this.logStrategy = logStrategy;
  }

  log(message: string): void {
    this.logStrategy.log(message);
  }

  setLogStrategy(logStrategy: LogStrategy): void {
    this.logStrategy = logStrategy;
  }
}
```

```
const message = "Hello, Strategy Pattern!";

const consoleLogger = new Logger(new ConsoleLogStrategy());
consoleLogger.log(message); // ConsoleLogStrategy를 사용하여 로그 기록

const fileLogger = new Logger(new FileLogStrategy());
fileLogger.log(message); // FileLogStrategy를 사용하여 로그 기록

const networkLogger = new Logger(new NetworkLogStrategy());
networkLogger.log(message); // NetworkLogStrategy를 사용하여 로그 기록

// 실행 시간에 로그 전략 변경
consoleLogger.setLogStrategy(new NetworkLogStrategy());
consoleLogger.log(message); // 변경된 NetworkLogStrategy를 사용하여 로그 기록
```

### 전략패턴 사용 라이브러리: Passport

[https://www.passportjs.org/](https://www.passportjs.org/)

 [Passport.js

Simple, unobtrusive authentication for Node.js

www.passportjs.org](https://www.passportjs.org/)

전략 패턴의 개념을 활용하여, 개발자가 애플리케이션에 필요한 인증 방식을 쉽게 추가하거나 교체할 수 있도록 설계되었다.