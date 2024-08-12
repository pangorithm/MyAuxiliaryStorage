# Chain of Responsibility

##  의도

### 목적

여러 처리기가 있을 때, 어떤 처리기가 요청을 처리할 지 미리 결정하지 않고, 각 처리기가 자신이 처리할 수 없는 요청은 다음 처리기에게 전달하는 방식. 이를 통해 요청을 보내는 측과 처리하는 측의 결합도를 낮출 수 있고 요청 처리기의 체인을 동적으로 재구성하는 것이 가능해진다.

### 구성요소

-   **Handler:** 요청을 처리하는 인터페이스. 모든 처리기는 이 인터페이스를 구현한다. 일반적으로 Handler 인터페이스에는 요청을 처리하는 메소드와 다음 처리기를 설정하는 메소드가 포함된다.
-   **ConcreteHandler:** Handler 인터페이스를 구현하는 실제 처리 클래스. 요청을 처리할 수 있으면 처리하고, 그렇지 않으면 요청을 체인상의 다음 객체에 전달한다.
-   **Client:** 요청을 생성하고 체인의 첫 번째 처리기에게 전달하는 역할을 한다.

### 대표 사용 예시

-   웹 서버에서 HTTP 요청을 처리하는 미들웨어 체인 구성
-   로깅 시스템에서 로그 메시지를 다양한 방식(콘솔, 파일, 네트워크 등)으로 처리하는 경우
-   GUI에서 이벤트를 처리할 때, 이벤트가 발생한 컴포넌트에서 부모 컴포넌트로 이벤트를 전파하는 경우

### 예시 코드

```
abstract class Logger {
  static readonly INFO = 1;
  static readonly DEBUG = 2;
  static readonly ERROR = 3;

  protected level: number;

  // 다음 요소 in 체인
  protected nextLogger: Logger | null = null;

  constructor(level: number) {
    this.level = level;
  }

  setNextLogger(nextLogger: Logger): void {
    this.nextLogger = nextLogger;
  }

  logMessage(level: number, message: string): void {
    if (this.level <= level) {
      this.write(message);
    }
    if (this.nextLogger != null) {
      this.nextLogger.logMessage(level, message);
    }
  }

  protected abstract write(message: string): void;
}

class ConsoleLogger extends Logger {
  constructor(level: number) {
    super(level);
  }

  protected write(message: string): void {
    console.log('Standard Console::Logger: ' + message);
  }
}

class ErrorLogger extends Logger {
  constructor(level: number) {
    super(level);
  }

  protected write(message: string): void {
    console.error('Error Console::Logger: ' + message);
  }
}

class FileLogger extends Logger {
  constructor(level: number) {
    super(level);
  }

  protected write(message: string): void {
    console.log('File::Logger: ' + message);
  }
}

// 클라이언트 코드
function clientCode() {
  const errorLogger = new ErrorLogger(Logger.ERROR);
  const fileLogger = new FileLogger(Logger.DEBUG);
  const consoleLogger = new ConsoleLogger(Logger.INFO);

  errorLogger.setNextLogger(fileLogger);
  fileLogger.setNextLogger(consoleLogger);

  // 체인의 시작점
  errorLogger.logMessage(Logger.INFO, 'This is an information.');
  errorLogger.logMessage(Logger.DEBUG, 'This is a debug level information.');
  errorLogger.logMessage(Logger.ERROR, 'This is an error information.');
}

clientCode();
```