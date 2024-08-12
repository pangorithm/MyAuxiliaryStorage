### 프로토타입 패턴

기존 객체를 복제하여 새로운 객체를 생성하는 방법을 제공한다. 특히 객체 생성 비용이 크거나, 객체의 타입이 런타임에 결정되는 경우에 유용하다. 프로토타입 패턴은 자바스크립트와 같은 프로토타입 기반 언어에서 자주 사용된다. 자바스크립트에서는 객체 리터럴이나 생성자 함수를 사용하여 객체를 생성한 후, 이를 복제하여 새로운 객체를 만드는 과정이 프로토타입 패턴의 개념과 일치한다.

프로토타입 패턴으로 복제를 구현해야 하는 이유로는 객체의 필드 중 일부가 비공개일 경우 외부에서 볼 수 없으며, 해당 객체의 클래스를 알아야 하므로 클래스에 의존하게 되고 그 객체가 따르는 인터페이스만 알고, 해당 객체의 구상 클래스는 알지 못할 수 있기 때문이다.

```
interface Product {
    clone(): Product;
}

class ConcreteProduct implements Product {
    private name: string;

    constructor(name: string) {
        this.name = name;
    }

    // 복제 메서드 구현
    public clone(): ConcreteProduct {
        // `new ConcreteProduct(this.name)`를 사용해도 되지만,
        // 더 복잡한 객체의 경우, 상태를 정확히 복제하기 위해
        // 복사 생성자 또는 직렬화/역직렬화 방식을 고려할 수 있습니다.
        return new ConcreteProduct(this.name);
    }

    public getName(): string {
        return this.name;
    }

    // 필요한 경우 여기에 더 많은 메서드를 추가할 수 있습니다.
}

// 사용 예
const originalProduct = new ConcreteProduct("Original Product");
const clonedProduct = originalProduct.clone();

console.log(originalProduct.getName()); // Original Product
console.log(clonedProduct.getName()); // Original Product
console.log(originalProduct !== clonedProduct); // true, 다른 인스턴스임을 확인
```