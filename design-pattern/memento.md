### 목적

객체의 상태를 캡처하여 외부에 저장함으로써, 나중에 객체를 그 상태로 되돌릴 수 있는 메커니즘을 제공한다. 이를 통해 객체의 내부 구조를 노출시키지 않고도 객체 상태의 복원과 관리가 가능다.

### 주요 구성 요소

-   **Originator**: 원본 객체로, 자신의 상태를 저장하거나 이전 상태로 복원하는 데 필요한 로직을 포함한다.
-   **Memento**: 메멘토 객체로, Originator 객체의 상태를 저장한다. 이 객체는 Originator의 상태를 캡처하여 외부에 저장하되, 그 상태에 대한 직접적인 접근이나 수정을 허용하지 않는다.
-   **Caretaker**: 메멘토를 관리하는 객체로, 메멘토의 저장과 복원을 담당한다. Caretaker는 메멘토의 내용을 모르며, 단지 메멘토를 보관하는 역할만 수행다.

### 작동 원리

1.  **상태 저장**: Originator 객체의 현재 상태를 저장하려면, Originator는 자신의 상태를 Memento 객체에 저장합니다. 이 Memento 객체는 Caretaker에 의해 관리됩니다.
2.  **상태 복원**: 특정 시점의 상태로 객체를 복원하고자 할 때, Caretaker는 해당 시점에 해당하는 Memento 객체를 Originator에게 전달합니다. Originator는 이 Memento 객체의 상태를 사용하여 자신을 해당 상태로 복원합니다.

### 예제코드

```
class Memento {
    private state: string;

    constructor(state: string) {
        this.state = state;
    }

    public getState(): string {
        return this.state;
    }
}

class Originator {
    private state: string;

    constructor() {
        this.state = '';
    }

    public setState(state: string): void {
        this.state = state;
    }

    public getState(): string {
        return this.state;
    }

    public saveToMemento(): Memento {
        return new Memento(this.state);
    }

    public restoreFromMemento(memento: Memento): void {
        this.state = memento.getState();
    }
}

class Caretaker {
    private mementos: Memento[] = [];

    public addMemento(memento: Memento): void {
        this.mementos.push(memento);
    }

    public getMemento(index: number): Memento {
        return this.mementos[index];
    }
}
```

```
// 사용 예제
const originator = new Originator();
const caretaker = new Caretaker();

originator.setState("State #1");
originator.setState("State #2");
caretaker.addMemento(originator.saveToMemento());

originator.setState("State #3");
console.log("Current State: " + originator.getState()); // State #3

caretaker.addMemento(originator.saveToMemento());

originator.setState("State #4");
console.log("Current State: " + originator.getState()); // State #4

originator.restoreFromMemento(caretaker.getMemento(0));
console.log("First saved State: " + originator.getState()); // State #2
originator.restoreFromMemento(caretaker.getMemento(1));
console.log("Second saved State: " + originator.getState()); // State #3
```