### 목적

파사드는 건물의 출입구가 있는 정면을 가리키는 단어이다.

이 뜻을 의역해서 풀어보자면 파사드 객체는 은행 창구의 은행원 역할을 한다고 할 수 있다.

복잡한 시스템에 대한 간단한 인터페이스를 제공하는 구조적 디자인 패턴.

시스템과의 상호작용을 단순화하고, 클라이언트가 시스템의 복잡한 부분을 직접 다루지 않도록 하는 것.

파사드는 클라이언트와 시스템의 나머지 부분 사이에 있는 유일한 통신 포인트 역할을 하여, 클라이언트는 파사드를 통해 시스템과 상호작용한다.

### **예제 코드**

```
class SubsystemA {
    operationA1(): string {
        return 'Subsystem A, Method A1\n';
    }

    operationA2(): string {
        return 'Subsystem A, Method A2\n';
    }
}

class SubsystemB {
    operationB1(): string {
        return 'Subsystem B, Method B1\n';
    }

    operationB2(): string {
        return 'Subsystem B, Method B2\n';
    }
}

class Facade {
    private subsystemA: SubsystemA = new SubsystemA();
    private subsystemB: SubsystemB = new SubsystemB();

    operation1(): string {
        let result = 'Facade initializes subsystems:\n';
        result += this.subsystemA.operationA1();
        result += this.subsystemB.operationB1();
        return result;
    }

    operation2(): string {
        let result = 'Facade orders subsystems to perform the action:\n';
        result += this.subsystemA.operationA2();
        result += this.subsystemB.operationB2();
        return result;
    }
}

// 클라이언트 코드
const facade = new Facade();
console.log(facade.operation1());
console.log(facade.operation2());
```

위의 예제 코드에서는 단순히 서브시스템의 실행 값을 더해주고 있다.

Facade 클래스는 SubsystemA와 SubsystemB의 복잡한 기능을 간단한 인터페이스(operation1과 operation2)로 제공한다. 이로인해 클라이언트는 Facade를 통해 서브시스템과 상호작용하므로, 서브시스템의 복잡성을 직접 다루지 않아도 된다.

### 구현방법

1.  기존 하위시스템이 이미 제공하는 것보다 더 간단한 인터페이스를 제공하는 것이 가능한지 확인한다.
2.  새 퍼사드 패턴 클래스에서 이 인터페이스를 선언하고 구현한다. 이 퍼사드는 클라이언트 코드의 호출들을 하위 시스템의 적절한 객체들로 리다이렉션해야 한다. 클라이언트가 아래 작업을 이미 수행하지 않은 한, 퍼사드는 하위 시스템을 초기화하고 추가 수명 주기를 관리하는 적업의 책임을 맡아야 한다.
3.  최대한 모든 클라이언트 코드가 퍼사드 패턴을 통해서만 하위시스템과 통신하도록 한다. 그렇게 되면 클라이언트 코드는 하위 시스템 코드의 변경 사항들로부터 보호된다.
4.  퍼사드가 과하게 커질 경우, 일부 행동들을 새로운 퍼사드 클래스로 추출한다.