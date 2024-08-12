### 목적

객체의 내부 상태가 변경될 때 그 객체의 행동을 변경하도록 한다. 객체의 상태를 나타내는 일련의 동작을 상태 객체에 위임함으로써, 객체의 상태에 따라 객체의 행동을 변경할 수 있게 한다. 이로 인해 조건문의 사용을 줄여 코드의 유지보수를 용이하게 하고, 더 명확하게 상태에 따른 행동을 구분할 수 있게 한다.

### 구성 요소

1.  **Context (문맥)**: 사용자가 직접 상호작용하는 객체로, 현재 상태를 가지고 있으며, 상태가 변경됨에 따라 다른 행동을 할 수 있습니다.
2.  **State (상태)**: 여러 상태들의 공통 인터페이스를 정의합니다. 이 인터페이스는 각 상태에서 실행될 수 있는 메서드를 선언합니다.
3.  **Concrete States (구체적인 상태들)**: State 인터페이스를 구현하거나 상속하여 만들어진 클래스들로, 특정 상태에 대한 구체적인 행동을 정의합니다.

### 타입스크립트 예시 코드

```
// State 인터페이스 정의
interface State {
  write(text: string): void;
}

// ConcreteState: 편집 상태
class EditingState implements State {
  write(text: string) {
    console.log(`Editing: ${text}`);
  }
}

// ConcreteState: 기본 상태
class DefaultState implements State {
  write(text: string) {
    console.log(`Viewing: ${text}`);
  }
}

// Context 클래스
class TextEditor {
  private state: State;

  constructor(state: State) {
    this.state = state;
  }

  setState(state: State) {
    this.state = state;
  }

  type(text: string) {
    this.state.write(text);
  }
}
```

```
// 사용 예
const editor = new TextEditor(new DefaultState());

console.log("Initially in default state:");
editor.type("First line");

console.log("Changing to editing state...");
editor.setState(new EditingState());
editor.type("Editing first line");

console.log("Going back to default state...");
editor.setState(new DefaultState());
editor.type("Second line");
```