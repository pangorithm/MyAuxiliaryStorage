### 목적

요청을 객체의 형태로 캡슐화하여 사용자가 요청을 발생시키는 객체와 요청을 처리하는 객체 사이의 의존성을 줄이고, 요청을 더 유연하게 관리할 수 있도록 한다. 이 패턴은 요청을 수행하는 호출자(Invoker)와 요청에 대해 특정 작업을 수행하는 수신자(Receiver) 사이의 연결을 간접적으로 만든다. 커맨드 객체는 이 둘 사이에서 요청을 캡슐화하며, 실행될 필요가 있는 모든 정보를 포함한다.

### 주요 구성 요소

-   **Command**: 실행될 모든 명령에 대한 인터페이스. 일반적으로 이 인터페이스에는 execute 메소드가 선언되어 있다.
-   **ConcreteCommand**: Command 인터페이스를 구현하며, 특정 Receiver 객체에 대한 작업을 호출. 이 객체는 Receiver 객체와 액션 사이의 연결고리 역할을 한다.
-   **Invoker**: Command 객체를 사용하고, Command 객체의 execute 메소드를 호출하여 요청을 실행한다.
-   **Receiver**: 요청에 대한 실제 작업을 수행. ConcreteCommand는 이 Receiver의 메소드를 호출하여 작업을 실행한다.
-   **Client**: ConcreteCommand 객체를 생성하고, 그것의 수신자를 결정한다. 또한, 이 객체를 Invoker에 할당한다.

### 타입스크립트 예제 코드

```
// Command 인터페이스
interface Command {
  execute(): void;
}

// Receiver 클래스
class Light {
  on(): void {
    console.log('Light is on');
  }

  off(): void {
    console.log('Light is off');
  }
}

// ConcreteCommand 클래스
class LightOnCommand implements Command {
  private light: Light;

  constructor(light: Light) {
    this.light = light;
  }

  execute(): void {
    this.light.on();
  }
}

class LightOffCommand implements Command {
  private light: Light;

  constructor(light: Light) {
    this.light = light;
  }

  execute(): void {
    this.light.off();
  }
}

// Invoker 클래스
class RemoteControl {
  private slot: Command;

  constructor() {}

  setCommand(command: Command): void {
    this.slot = command;
  }

  pressButton(): void {
    this.slot.execute();
  }
}

// Client 코드
const light = new Light();
const lightOn = new LightOnCommand(light);
const lightOff = new LightOffCommand(light);

const remote = new RemoteControl();
remote.setCommand(lightOn);
remote.pressButton(); // 출력: Light is on

remote.setCommand(lightOff);
remote.pressButton(); // 출력: Light is off
```

### 요약

invoker는 concreateCommand의 excute 메서드를 실행할 뿐이고 receiver에서 어떤 명령이 실행될 지는 invoker가 어떤 concreateCommand 인스턴스를 받았느냐에 따라 결정된다.