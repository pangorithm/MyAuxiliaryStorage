### 목적

알고리즘을 객체의 구조에서 분리시켜 객체 구조의 변화에 영향을 받지 않으면서도 새로운 연산을 쉽게 추가할 수 있게 해준다. 주로 복잡한 객체 구조에 대해 연산을 수행해야 할 때 유용하게 사용된다.

### 구성 요소

1.  **Visitor Interface**: 방문자가 수행할 연산을 선언하는 인터페이스. 구체적인 방문자 클래스들이 이 인터페이스를 구현한다.
2.  **Concrete Visitor**: Visitor 인터페이스를 구현하는 클래스, 구조의 각 요소에 대해 방문하며 수행할 구체적인 연산을 구현한다.
3.  **Element Interface**: accept 메서드를 선언하는 인터페이스. 이 메서드는 방문자(Visitor)를 인자로 받는다.
4.  **Concrete Element**: Element 인터페이스를 구현하는 클래스로, 실제 데이터와 구조를 포함한다. accept 메서드는 방문자에게 자신을 인자로 전달하여, 방문자가 수행할 연산을 이 객체에 대해 실행할 수 있도록 한다.
5.  **Object Structure**: 객체 구조를 나타내며, Element들의 집합을 관리한다. 이 구조는 방문자가 구조의 각 요소를 방문할 수 있도록 한다.

### 작동 방식

1.  방문자 패턴은 객체 구조에 있는 각 요소에 대해 특정 연산을 수행하고자 할 때 사용된다. 각 요소 클래스는 accept 메서드를 통해 방문자 객체를 받아들일 수 있으며, 이 메서드는 방문자에게 자신을 전달한다.
2.  방문자는 전달받은 요소에 대해 특정 연산을 수행한다. 이 때, 방문자는 요소의 구체적인 클래스 타입에 따라 오버로딩된 visit 메서드 중 적절한 메서드를 실행한다.
3.  이 패턴을 사용하면 구조 내의 각 요소에 대해 수행해야 하는 연산을 추가하기 위해 요소의 클래스를 변경할 필요가 없다. 대신, 새로운 방문자를 추가함으로써 새로운 연산을 쉽게 추가할 수 있다.

### 장점과 단점

**장점**:

-   연산을 객체 구조에서 분리시켜 객체 구조의 변경 없이 새로운 연산을 추가할 수 있다.
-   복잡한 객체 구조에 대한 연산을 캡슐화하여 코드의 재사용성과 분리를 향상시킬 수 있다.

**단점**:

-   새로운 Concrete Element를 추가할 때마다 모든 Concrete Visitor 클래스를 수정해야 할 수도 있습니다. 이는 오픈/클로즈 원칙에 반할 수 있다.
-   복잡한 객체 구조와 다양한 방문자를 관리해야 하므로 구현이 복잡해질 수 있다.

### 타입스크립트 예제코드

```
// Component Interfaces
interface ComputerPart {
  accept(computerPartVisitor: ComputerPartVisitor): void;
}

interface ComputerPartVisitor {
  visit(computer: Computer): void;
  visit(mouse: Mouse): void;
  visit(keyboard: Keyboard): void;
  visit(monitor: Monitor): void;
}

// Concrete Components
class Keyboard implements ComputerPart {
  accept(computerPartVisitor: ComputerPartVisitor): void {
    computerPartVisitor.visit(this);
  }
}

class Monitor implements ComputerPart {
  accept(computerPartVisitor: ComputerPartVisitor): void {
    computerPartVisitor.visit(this);
  }
}

class Mouse implements ComputerPart {
  accept(computerPartVisitor: ComputerPartVisitor): void {
    computerPartVisitor.visit(this);
  }
}

class Computer implements ComputerPart {
  parts: ComputerPart[];

  constructor() {
    this.parts = [new Mouse(), new Keyboard(), new Monitor()];
  }

  accept(computerPartVisitor: ComputerPartVisitor): void {
    for (let part of this.parts) {
      part.accept(computerPartVisitor);
    }
    computerPartVisitor.visit(this);
  }
}

// Concrete Visitor
class ComputerPartDisplayVisitor implements ComputerPartVisitor {
  visit(element: ComputerPart): void {
    if (element instanceof Computer) {
      console.log("Displaying Computer.");
    } else if (element instanceof Mouse) {
      console.log("Displaying Mouse.");
    } else if (element instanceof Keyboard) {
      console.log("Displaying Keyboard.");
    } else if (element instanceof Monitor) {
      console.log("Displaying Monitor.");
    }
  }
}
```

```
const computer: ComputerPart = new Computer();
const computerPartVisitor: ComputerPartVisitor = new ComputerPartDisplayVisitor();
computer.accept(computerPartVisitor);
```

### 자바 예시 코드

[https://refactoring.guru/ko/design-patterns/visitor/java/example](https://refactoring.guru/ko/design-patterns/visitor/java/example)
