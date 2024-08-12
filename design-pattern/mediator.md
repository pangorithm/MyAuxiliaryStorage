### 목적

객체 간의 복잡한 통신을 캡슐화하는 방법을 제공. 여러 컴포넌트 또는 객체 사이의 직접적인 참조를 제거하여, 그들 사이의 결합도를 낮추는 대신, 모든 통신은 중재자(Mediator)라고 불리는 중앙 집중화된 객체를 통해 이루어진다. 이로 인해 각 객체는 오직 Mediator 인터페이스만을 알고 있으며, 다른 객체들의 존재나 내부 구현에 대해서는 알 필요가 없다.

### 패턴의 구성 원리

중재자 패턴은 서로 독립적으로 작동해야 하는 컴포넌트 간의 모든 직접 통신을 중단한 후, 대신 이러한 컴포넌트들은 호출들을 적절한 컴포넌트들로 리다이렉션하는 특수 중재자 객체를 호출하여 간접적으로 협력하게 하도록 한다. 이로 인해 컴포넌트들은 수십 개의 동료 컴포넌트들과 결합되는 대신 단일 중재자 클래스에만 의존한다.

### Mediator 패턴의 구성 요소

1.  **Mediator 인터페이스:** 중재자의 역할을 정의하는 인터페이스입니다. 이 인터페이스는 여러 컴포넌트 간의 통신을 조정하는 메서드를 선언한다.
2.  **Concrete Mediator:** Mediator 인터페이스를 구현하는 클래스, 구체적인 컴포넌트 간의 통신 로직을 캡슐화한다.
3.  **Colleague (컴포넌트):** 중재자를 통해 서로 통신하는 클래스들. Mediator 인터페이스를 통해 중재자와 통신하며, 다른 컴포넌트들과는 직접 통신하지 않는다.

### 장점

-   **낮은 결합도:** 컴포넌트들이 직접 서로를 참조하지 않고 중재자를 통해서만 통신하기 때문에 결합도가 낮아져 시스템의 유지보수성과 확장성을 향상시킨다.
-   **재사용성 향상:** 각 컴포넌트가 독립적이기 때문에 다른 시나리오나 프로젝트에서 재사용하기가 더 쉽다.
-   **중앙 집중화된 통제:** 복잡한 통신 로직이 Mediator 내부에 캡슐화되어 있기 때문에, 통신의 흐름을 쉽게 이해하고 관리할 수 있다.

### 단점

-   **중재자의 복잡성 증가:** 모든 통신 로직이 중재자 내부에 구현되므로, 중재자 자체가 매우 복잡해질 수 있다. 이로 인해 중재자의 관리와 유지보수가 어려울 수 있다.

### 타입 스크립트 예제 코드

```
// Mediator Interface
interface Mediator {
    notify(sender: object, event: string): void;
}

// Concrete Mediator
class ConcreteMediator implements Mediator {
    private component1: Component1;

    private component2: Component2;

    constructor(c1: Component1, c2: Component2) {
        this.component1 = c1;
        this.component1.setMediator(this);
        this.component2 = c2;
        this.component2.setMediator(this);
    }

    public notify(sender: object, event: string): void {
        if (event === 'A') {
            console.log('Mediator reacts on A and triggers following operations:');
            this.component2.doC();
        }

        if (event === 'D') {
            console.log('Mediator reacts on D and triggers following operations:');
            this.component1.doB();
            this.component2.doC();
        }
    }
}

// Colleague Component
class BaseComponent {
    protected mediator: Mediator;

    constructor(mediator?: Mediator) {
        this.mediator = mediator!;
    }

    public setMediator(mediator: Mediator): void {
        this.mediator = mediator;
    }
}

// Concrete Components
class Component1 extends BaseComponent {
    public doA(): void {
        console.log('Component 1 does A.');
        this.mediator.notify(this, 'A');
    }

    public doB(): void {
        console.log('Component 1 does B.');
        this.mediator.notify(this, 'B');
    }
}

class Component2 extends BaseComponent {
    public doC(): void {
        console.log('Component 2 does C.');
        this.mediator.notify(this, 'C');
    }

    public doD(): void {
        console.log('Component 2 does D.');
        this.mediator.notify(this, 'D');
    }
}

// Mediator와 Colleague가 서로를 필드에 두고 있다.
```

```
// 실행 예시
const c1 = new Component1();
const c2 = new Component2();
const mediator = new ConcreteMediator(c1, c2);

console.log('Client triggers operation A.');
c1.doA();
/*
    Component 1 does A.
    Mediator reacts on A and triggers following operations:
    Component 2 does C.
*/

console.log('');
console.log('Client triggers operation D.');
c2.doD();
/*
    Component 2 does D.
    Mediator reacts on D and triggers following operations:
    Component 1 does B.
    Component 2 does C.
*/
```

#### 자바 코드 예시

[https://refactoring.guru/ko/design-patterns/mediator/java/example](https://refactoring.guru/ko/design-patterns/mediator/java/example)

 [자바로 작성된 중재자 / 디자인 패턴들

/ 디자인 패턴들 / 중재자 / 자바 자바로 작성된 중재자 중재자는 행동 디자인 패턴이며 프로그램의 컴포넌트들이 특수 중재자 객체를 통하여 간접적으로 소통하게 함으로써 해당 컴포넌트 간의

refactoring.guru](https://refactoring.guru/ko/design-patterns/mediator/java/example)

### 중재자와 퍼사드 비교

#### 중재자 패턴 (Mediator Pattern)

-   **목적**: 객체 간의 복잡한 통신을 단순화하는 것. 여러 컴포넌트 간의 직접적인 참조를 제거하여, 객체 사이의 결합도를 낮추고, 유지 보수와 확장성을 향상시킨다.
-   **방식**: 중재자 패턴은 객체들 사이의 상호작용을 캡슐화하는 중재자 객체를 도입하여, 모든 통신을 중재자를 통해 이루어지게 한다. 각 객체는 다른 객체들과 직접 통신하는 대신 중재자를 통해 필요한 작업을 수행하도록 요청한다.
-   **적용 예**: 채팅 시스템에서 여러 사용자 간의 메시지 전송을 관리하는 채팅 서버, UI 컴포넌트 간의 상호작용을 관리하는 이벤트 매니저 등.

#### 퍼사드 패턴 (Facade Pattern)

-   **목적**: 복잡한 서브시스템에 대한 단순화된 인터페이스를 제공하는 것. 클라이언트가 서브시스템의 복잡성을 이해하지 않고도 사용할 수 있도록 한다.
-   **방식**: 복잡한 서브시스템의 기능을 하나의 인터페이스 뒤로 숨긴다. 클라이언트는 이 인터페이스를 통해 서브시스템과 상호작용하며, 서브시스템 내부의 복잡한 작업들은 퍼사드 뒤에서 처리된다.
-   **적용 예**: 컴퓨터 시스템을 시작할 때 BIOS가 제공하는 간단한 인터페이스를 통해 다양한 하드웨어 초기화 작업을 수행하는 과정, 복잡한 라이브러리 또는 프레임워크에 대한 간단한 API 제공 등.

#### 비교

-   **목적의 차이**: 중재자 패턴은 객체 간의 복잡한 상호작용을 중재자를 통해 단순화하는 데 중점을 두는 반면, 퍼사드 패턴은 복잡한 시스템에 대해 단순화된 인터페이스를 제공하는 데 초점을 맞춘다.
-   **사용하는 관점의 차이**: 중재자 패턴은 시스템 내부의 컴포넌트 간의 상호작용을 개선하기 위해 사용되는 반면, 퍼사드 패턴은 외부 클라이언트가 시스템을 더 쉽게 사용할 수 있도록 하는 데 사용된다.
-   **결합도 관리**: 두 패턴 모두 결합도를 관리하는 방식에서 유사성을 보이지만, 중재자는 객체 간의 직접적인 연결을 줄이는 반면, 퍼사드는 복잡한 서브시스템과의 상호작용을 단순화한다.