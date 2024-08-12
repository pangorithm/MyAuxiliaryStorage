### 목적

컬랙션의 내부 표현 방식과 상관 없이, 외부에서는 통일된 방법으로 해당 컬렉션의 요소들을 순회할 수 있게 하는 것. 이를 통해 컬렉션의 구현과 순회 로직 사이의 결합도를 낮출 수 있으며, 다양한 컬렉션에 대해 일관된 인터페이스를 제공할 수 있다.

### 핵심 구성 요소

-   **Iterator(이터레이터):** 컬렉션을 순회하는 인터페이스. 일반적으로 next(), hasNext()와 같은 메소드를 제공하여, 컬렉션의 다음 요소에 접근하고, 더 이상 순회할 요소가 있는지를 확인할 수 있다.
-   **ConcreteIterator(구체적 이터레이터):** Iterator 인터페이스를 구현하는 클래스로, 특정 컬렉션의 순회 로직을 구현.
-   **Aggregate(집합체):** 컬렉션의 인터페이스를 정의. 일반적으로 이터레이터 객체를 생성하고 반환하는 메소드(createIterator())를 제공.
-   **ConcreteAggregate(구체적 집합체):** Aggregate 인터페이스를 구현하는 클래스로, 실제 컬렉션을 구현. createIterator() 메소드를 통해 해당 컬렉션을 순회할 수 있는 ConcreteIterator 인스턴스를 생성하여 반환.

### 타입스크립트 예시코드

```
// Aggregate Interface
interface Aggregate {
    createIterator(): Iterator<number>;
}

// ConcreteAggregate
class NumberCollection implements Aggregate {
    private items: number[] = [];

    constructor(items: number[]) {
        this.items = items;
    }

    public createIterator(): Iterator<number> {
        return new NumberIterator(this);
    }

    public getItems(): number[] {
        return this.items;
    }
}

// Iterator Interface
interface Iterator<T> {
    next(): T | undefined;
    hasNext(): boolean;
}

// ConcreteIterator
class NumberIterator implements Iterator<number> {
    private collection: NumberCollection;
    private position: number = 0;

    constructor(collection: NumberCollection) {
        this.collection = collection;
    }

    public next(): number | undefined {
        // Return the next element if it exists, otherwise return undefined
        if (this.hasNext()) {
            return this.collection.getItems()[this.position++];
        } else {
            return undefined;
        }
    }

    public hasNext(): boolean {
        // Check if the current position is less than the length of the collection
        return this.position < this.collection.getItems().length;
    }
}

// Usage
const numbers = new NumberCollection([1, 2, 3, 4, 5]);
const iterator = numbers.createIterator();

console.log("Iterating over numbers:");
while (iterator.hasNext()) {
    const number = iterator.next();
    console.log(number);
}
```

### 구현 예시

대표적인 구현 예시는 JAVA의 Iterator 인터페이스가 있다.  
[https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/Iterator.html](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/Iterator.html)
