### 팩토리 패턴

객체를 사용하는 코드에서 객체 생성 부분을 떼어내 추상화한 패턴. 생성할 객체의 정확한 클래스를 명시하지 않고도 객체를 생성할 수 있도록 하는 디자인 패턴이다. 주로 단일 클래스가 객체 생성을 책임지며, 생성할 객체의 클래스가 서브클래스 중 하나인 경우에 유용하다. 상속 관계에 있는 두 클래스에서 상위 클래스가 중요한 뼈대를 결정하고, 하위 클래스에서 객체 생성에 관한 구체적인 내용을 결정한다.

이는 의존성 주입이라고 볼 수 도 있는데, 하위 클래스에서 생성한 인스턴스를 생위 클래스에 주입하기 때문이다.

```
class VehicleFactory {
  // Factory Method
  createVehicle(type) {
    switch (type) {
      case "car":
        return new Car();
      case "truck":
        return new Truck();
      default:
        throw new Error("Vehicle type not supported");
    }
  }
}

class Car {
  drive() {
    console.log("Driving a car");
  }
}

class Truck {
  drive() {
    console.log("Driving a truck");
  }
}

// 사용 예
const factory = new VehicleFactory();
const car = factory.createVehicle("car");
car.drive(); // Output: Driving a car

const truck = factory.createVehicle("truck");
truck.drive(); // Output: Driving a truck
```

###   

### 추상 팩토리 패턴

추상 팩토리 패턴은 관련성이 있거나 의존성이 있는 여러 종류의 객체를 생성하기 위한 인터페이스를 제공합니다. 이 패턴은 서로 다른 팩토리들을 그룹화하여 각각의 팩토리가 특정 객체 그룹을 생성하도록 합니다.

```
// AbstractFactory
class GUIFactory {
  createButton() {}
  createCheckbox() {}
}

// ConcreteFactory1
class WinFactory extends GUIFactory {
  createButton() {
    return new WinButton();
  }
  createCheckbox() {
    return new WinCheckbox();
  }
}

// ConcreteFactory2
class MacFactory extends GUIFactory {
  createButton() {
    return new MacButton();
  }
  createCheckbox() {
    return new MacCheckbox();
  }
}

// AbstractProductA
class Button {
  paint() {}
}

// AbstractProductB
class Checkbox {
  paint() {}
}

// ConcreteProductA1
class WinButton extends Button {
  paint() {
    console.log("Rendering a button in Windows style.");
  }
}

// ConcreteProductA2
class MacButton extends Button {
  paint() {
    console.log("Rendering a button in MacOS style.");
  }
}

// ConcreteProductB1
class WinCheckbox extends Checkbox {
  paint() {
    console.log("Rendering a checkbox in Windows style.");
  }
}

// ConcreteProductB2
class MacCheckbox extends Checkbox {
  paint() {
    console.log("Rendering a checkbox in MacOS style.");
  }
}

// Client code
function application(factory) {
  const button = factory.createButton();
  const checkbox = factory.createCheckbox();
  
  button.paint();
  checkbox.paint();
}

// Example usage
const winFactory = new WinFactory();
application(winFactory); // Windows style

const macFactory = new MacFactory();
application(macFactory); // MacOS style
```
  

  

### 공통점

생성 로직을 클라이언트로부터 분리하여 코드의 유연성을 높이고, 결합도를 낮춘다.

  

### 차이점

팩토리 패턴은 하나의 객체를 생성하는 데 중점을 둔 반면, 추상 팩토리 패턴은 관련 있는 여러 종류의 객체를 일관된 방식으로 생성하는 데 중점을 둔다. 따라서 새로운 종류의 객체를 추가할 때, 팩토리 패턴은 새로운 생성자 클래스만 추가하면 되지만, 추상 팩토리 패턴은 팩토리 인터페이스와 모든 구체 팩토리 클래스에 해당 객체에 해당하는 생성 메서드를 추가해야 한다.  이러한 특성으로 인해 팩토리 패턴은 단일 제품 생성에 초점을 맞춘 보다 단순한 시나리오에 적합한 반면, 추상 팩토리 패턴은 여러 관련 제품군을 효율적으로 생성하고 관리할 수 있는 더 복잡한 시나리오에 적합하다.