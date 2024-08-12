### 목적

객체의 상태 변화를 관찰하는 관찰자들에게 상태 변화를 알려주는 행위를 자동으로 수행하도록 한다. 일대다 의존 관계를 생성하여 한 객체의 상태 변화가 발생할 때, 모든 의존 객체들이 자동으로 알림을 받고 갱신될 수 있게 한다.

### 구성 요소

-   **Subject (Observable)**: 관찰 대상 객체. 여러 Observer 객체를 자신의 관찰자 목록에 등록할 수 있다. Subject는 내부 상태에 변화가 있을 때 등록된 모든 Observer에게 변화를 알린다.
-   **Observer**: Subject의 상태 변화를 관찰하는 객체. Subject로부터 상태 변화를 알림받으면 이에 대응하는 업데이트를 수행한다.
-   **ConcreteSubject**: Subject 인터페이스를 구현하는 실제 객체. 내부 상태와 Observer 목록을 관리한다.
-   **ConcreteObserver**: Observer 인터페이스를 구현하는 실제 객체. Subject의 상태 변화에 대응하는 구체적인 업데이트 로직을 포함한다.

### 작동 원리

1.  Observer는 Subject에 자신을 등록(구독)하여, Subject의 상태 변화를 관찰한다.
2.  Subject의 상태가 변화하면, Subject는 등록된 모든 Observer에게 상태 변화를 알리는 알림 메서드를 호출한다.
3.  각 Observer는 알림을 받고, 필요한 업데이트나 처리를 수행한다.

타입스크립트 예시 코드

```
interface Observer {
    update: (state: any) => void;
}

class Subject {
    private observers: Observer[] = [];
    private state: any;

    public getState(): any {
        return this.state;
    }

    public setState(state: any): void {
        this.state = state;
        this.notifyAllObservers();
    }

    public attach(observer: Observer): void {
        const isExist = this.observers.includes(observer);
        if (isExist) {
            return console.log('Observer has been attached already.');
        }

        this.observers.push(observer);
    }

    public detach(observer: Observer): void {
        const observerIndex = this.observers.indexOf(observer);
        if (observerIndex !== -1) {
            this.observers.splice(observerIndex, 1);
        }
    }

    public notifyAllObservers(): void {
        for (const observer of this.observers) {
            observer.update(this.state);
        }
    }
}

class ConcreteObserver implements Observer {
    private name: string;

    constructor(name: string) {
        this.name = name;
    }

    public update(state: any): void {
        console.log(`${this.name} has been notified. New state: ${state}`);
    }
}
```

```
// Usage
const subject = new Subject();

const observer1 = new ConcreteObserver('Observer 1');
const observer2 = new ConcreteObserver('Observer 2');

subject.attach(observer1);
subject.attach(observer2);

subject.setState('state 1');
// Observer 1 has been notified. New state: state 1
// Observer 2 has been notified. New state: state 1

subject.detach(observer1);

subject.setState('state 2');
// Observer 2 has been notified. New state: state 2
```

### 메시지 큐와 옵저버 패턴의 유사점

-   **일대다 통신**: 옵저버 패턴에서 한 Subject가 여러 Observer에게 변화를 알림과 유사하게, 메시지 큐 시스템에서 한 메시지 발행자가 여러 구독자에게 메시지를 전달할 수 있다.
-   **비동기 처리**: 옵저버 패턴에서 Subject의 상태 변화를 Observer가 비동기적으로 처리할 수 있듯, 메시지 큐는 발행된 메시지를 비동기적으로 구독자에게 전달하며, 구독자는 자신의 처리 속도에 맞춰 메시지를 처리할 수 있다.
-   **느슨한 결합**: 두 패턴 모두 시스템의 결합도를 낮추는 데 기여한다. 옵저버 패턴에서 Subject는 Observer의 구체적인 구현을 알 필요가 없으며, 메시지 큐에서도 메시지의 발행자와 구독자는 서로를 직접적으로 알 필요가 없다.

### 메시지 큐와 옵저버 패턴의 차이점

-   **통신 메커니즘**: 옵저버 패턴은 주로 메모리 내에서 객체 간의 직접적인 호출을 통해 이루어지는 반면, 메시지 큐는 네트워크를 통해 프로세스나 서비스 간에 메시지를 전달한다.
-   **메시지 지속성**: 대부분의 메시지 큐 시스템은 메시지를 일정 기간 동안 저장하고, 구독자가 준비될 때까지 메시지를 보관할 수 있다. 반면, 옵저버 패턴은 일반적으로 메모리 내에서 실시간으로 동작하며, Observer가 준비되지 않은 상태에서 발생한 이벤트는 유실될 수 있다.
-   **확장성 및 관리 기능**: 메시지 큐 시스템은 대규모 분산 시스템에서의 확장성, 메시지 라우팅, 메시지 상태 관리 등 추가적인 기능을 제공다.

### 메시지 큐의 구독 메커니즘

-   푸시 기반 (Push-Based) 방식
    -   메시지 큐 시스템은 새 메시지가 큐에 도착하면 자동으로 해당 메시지를 구독자에게 전달한다.
    -   이 방식에서 구독자는 메시지를 받을 준비가 되어 있어야 하며, 메시지 큐 시스템이 직접 구독자의 대기 중인 리스너(listener)나 콜백(callback) 함수를 호출하여 메시지를 전달한다.
    -   구독자는 특정 조건이나 이벤트에 따라 메시지를 즉시 처리할 수 있어야 한다.
    -   푸시 기반 방식은 실시간 처리가 중요한 애플리케이션에 적합할 수 있으나, 구독자가 처리할 수 있는 속도보다 메시지가 빠르게 도착하는 경우 문제가 발생할 수 있다.
-   풀 기반 (Pull-Based) 방식
    -   구독자는 메시지 큐 시스템에 주기적으로 연결하여 대기 중인 메시지가 있는지 확인(폴링)하고, 메시지가 있을 경우 이를 직접 가져간다(retrieve).
    -   이 방식에서 구독자는 자신의 처리 용량과 속도에 맞춰 새 메시지를 요청하고 가져가므로, 메시지 처리의 속도를 더 잘 제어할 수 있다.
    -   풀 기반 방식은 구독자가 메시지를 처리할 준비가 되었을 때만 메시지를 요청하기 때문에, 자원의 효율적 사용과 흐름 제어(flow control) 측면에서 이점을 가질 수 있다.
    -   다만, 폴링 간격이 길어질 경우 메시지 처리에 지연이 발생할 수 있으며, 너무 짧으면 불필요한 네트워크 트래픽과 리소스 낭비를 초래할 수 있다.
-   선택 기준
    -   **푸시 기반** 방식은 실시간 처리가 중요하고, 구독자가 빠르게 메시지를 소비할 수 있는 환경에 적합하다.
    -   **풀 기반** 방식은 구독자가 처리 용량에 따라 메시지 소비를 조절해야 하거나, 폴링 방식의 지연이 크게 문제되지 않는 경우에 유리하다.