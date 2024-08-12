### **특징**

객체에 동적으로 새로운 책임(기능)을 추가할 수 있는 패턴. 클래스의 수정 없이 기능 확장을 원할 때 유용하게 사용된다. 객체의 세부 사항을 클라이언트로부터 숨기면서도 새로운 기능을 추가할 수 있게 한다.

### **구성 요소**

-   **Component**: 기본 기능을 정의하는 인터페이스 또는 추상 클래스. 데코레이터와 구체적인 컴포넌트 모두 이 인터페이스를 구현한다.
-   **ConcreteComponent**: Component 인터페이스를 구현하며, 데코레이션(추가 기능)이 적용되는 기본 객체.
-   **Decorator**: Component 인터페이스를 가지며 내부에 Component 타입의 객체를 포함한다. 이 클래스는 Component 인터페이스의 메서드를 오버라이드하고, 자신이 장식할 컴포넌트를 호출하여 그 결과에 무언가를 추가함으로써 새로운 기능을 구현한다.
-   **ConcreteDecorator**: Decorator를 상속받아 추가적인 기능을 구현하는 클래스

### **작동 원리**

1.  데코레이터 패턴은 구성(Composition)을 사용하여 기능을 확장한다. 즉, Decorator 객체는 내부적으로 Component 객체를 가지고 있으며, 이를 통해 메서드 호출을 위임한다.
2.  클라이언트는 Component 인터페이스를 통해 객체를 사용한다. 이때, 클라이언트가 사용하는 객체는 실제 ConcreteComponent일 수도 있고, 하나 이상의 Decorator로 감싸진 ConcreteComponent일 수도 있다.
3.  데코레이터는 ConcreteComponent의 메서드를 호출하기 전후에 추가적인 기능을 수행할 수 있으며, 이를 통해 기능을 동적으로 확장한다.

### **장점**

-   기존 코드를 수정하지 않고도 객체에 새로운 책임을 추가할 수 있다.
-   서브클래스 만들기 대신 데코레이터를 통해 유연하게 기능을 확장할 수 있다.
-   단일 책임 원칙과 개방-폐쇄 원칙을 지킬 수 있다.

### **단점**

-   시스템에 많은 작은 객체가 추가되어, 전체 구조가 복잡해질 수 있다.
-   데코레이터의 연쇄적 사용은 디버깅을 어렵게 만들 수 있다.
-   데코레이터 패턴은 유연성을 극대화하고 기능 확장을 용이하게 하지만, 설계가 복잡해지고 많은 수의 객체를 도입하는 부담이 따르므로, 신중하게 사용해야 한다.

### **예시코드**

```
// Component 인터페이스
// 기본 인터페이스인 Coffee를 정의
interface Coffee {
    getCost(): number;
    getDescription(): string;
}

// ConcreteComponent 클래스
// Coffee 인터페이스의 기본 구현을 제공
class SimpleCoffee implements Coffee {
    getCost(): number {
        return 10; // 기본 가격
    }

    getDescription(): string {
        return 'Simple coffee';
    }
}

// Decorator 클래스
// Coffee 인터페이스를 구현하며, 추가될 기능을 가진 클래스들의 기본 클래스 역할
abstract class CoffeeDecorator implements Coffee {
    constructor(protected baseCoffee: Coffee) {}

    getCost(): number {
        return this.baseCoffee.getCost();
    }

    getDescription(): string {
        return this.baseCoffee.getDescription();
    }
}

// ConcreteDecorator 클래스
// CoffeeDecorator를 상속받아 구체적인 기능(여기서는 추가 재료)을 제공하는 클래스
class MilkCoffee extends CoffeeDecorator {
    getCost(): number {
        return this.baseCoffee.getCost() + 2; // 우유 비용 추가
    }

    getDescription(): string {
        return this.baseCoffee.getDescription() + ', milk';
    }
}

class WhipCoffee extends CoffeeDecorator {
    getCost(): number {
        return this.baseCoffee.getCost() + 5; // 휘핑 크림 비용 추가
    }

    getDescription(): string {
        return this.baseCoffee.getDescription() + ', whip';
    }
}
```

```
// 커피를 만들고, 여러 데코레이터로 커피에 추가 재료를 넣어 가격과 설명을 업데이트하는 방법
const simpleCoffee = new SimpleCoffee();
console.log(simpleCoffee.getCost()); // 10
console.log(simpleCoffee.getDescription()); // Simple coffee

let milkCoffee = new MilkCoffee(simpleCoffee);
console.log(milkCoffee.getCost()); // 12
console.log(milkCoffee.getDescription()); // Simple coffee, milk

let whipCoffee = new WhipCoffee(milkCoffee);
console.log(whipCoffee.getCost()); // 17
console.log(whipCoffee.getDescription()); // Simple coffee, milk, whip
```

자바의 InputStream에서 사용된 데코레이터 패턴 예시

```
// 기본 Component
public abstract class InputStream {
    public abstract int read() throws IOException;
    // 클래스 내 다른 메소드들...
}

// Concrete Component
// FileInputStream은 InputStream의 구현체로, 파일에서 바이트 단위로 데이터를 읽는다.
public class FileInputStream extends InputStream {
    // 파일에서 데이터를 읽는 구현...
    public int read() throws IOException {
        // 실제 읽기 작업 구현...
    }
    // 클래스 내 다른 메소드들...
}

// Decorator
/* 
FilterInputStream은 모든 InputStream 데코레이터의 기본 클래스. 
이 클래스 자체는 InputStream을 확장하며, 
내부에 또 다른 InputStream 객체를 유지하여 추가적인 기능을 제공하기 위한 준비한다.
*/
public class FilterInputStream extends InputStream {
    protected volatile InputStream in;

    protected FilterInputStream(InputStream in) {
        this.in = in;
    }

    public int read() throws IOException {
        return in.read(); // 내부 InputStream 객체에 위임
    }
    // 클래스 내 다른 메소드들...
}

// Concrete Decorator
// BufferedInputStream과 DataInputStream은 FilterInputStream을 확장하여 구체적인 기능을 추가한다.

// BufferedInputStream은 버퍼링을 추가하여 데이터를 읽는 효율성을 향상
public class BufferedInputStream extends FilterInputStream {
    // 버퍼링을 위한 구현...
    public int read() throws IOException {
        // 버퍼에서 데이터를 읽는 구현...
    }
    // 클래스 내 다른 메소드들...
}

// DataInputStream은 기본 데이터 타입을 읽을 수 있는 메소드를 추가
public class DataInputStream extends FilterInputStream {
    // 데이터 타입별 읽기 메소드 구현...
    public final int readInt() throws IOException {
        // 정수를 읽는 구현...
    }
    // 클래스 내 다른 메소드들...
}
```

```
import java.io.*;

public class InputStreamExample {
    public static void main(String[] args) throws IOException {
        int data;

        try (InputStream is = new BufferedInputStream(new FileInputStream("data.txt"))) {
            while((data = is.read()) != -1) {
                // 데이터 처리...
            }
        }

        try (DataInputStream dis = new DataInputStream(new BufferedInputStream(new FileInputStream("data.txt")))) {
            int intData = dis.readInt();
            // 정수 데이터 처리...
        }
    }
}
```