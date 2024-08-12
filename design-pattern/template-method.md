### 목적

알고리즘의 구조를 메서드에 정의하고, 알고리즘의 일부 단계를 서브클래스에서 재정의할 수 있게 함으로써, 알고리즘의 구조를 변경하지 않고도 특정 단계를 재정의할 수 있도록 한다.

### 구성 요소

-   **추상 클래스(Abstract Class):** 알고리즘의 골격을 정의하는 메서드(탬플릿 메서드)와 알고리즘의 일부 단계를 구현하는 하나 이상의 추상 메서드 또는 구체적 메서드를 포함합니다.
-   **구체 클래스(Concrete Class):** 추상 클래스에서 정의한 알고리즘의 단계 중 일부를 구현하는 클래스입니다. 필요한 경우 알고리즘의 특정 단계를 재정의하여 구체적인 행동을 제공합니다.

```
abstract class DataProcessor {
    // 탬플릿 메서드
    public process(): void {
        const data = this.loadData();
        const parsedData = this.parseData(data);
        this.saveData(parsedData);
    }

    // 기본 단계: 데이터 로드
    private loadData(): string {
        // 데이터 로드 로직 구현 (예시)
        console.log("Data loaded.");
        return "raw data";
    }

    // 추상 단계: 데이터 파싱 (서브클래스에서 구현해야 함)
    protected abstract parseData(data: string): string;

    // 기본 단계: 데이터 저장
    private saveData(data: string): void {
        // 데이터 저장 로직 구현 (예시)
        console.log(`Data saved: ${data}`);
    }
}

class JsonDataProcessor extends DataProcessor {
    // `parseData` 메서드 구현
    protected parseData(data: string): string {
        // JSON 파싱 로직 구현 (예시)
        console.log("Data parsed as JSON.");
        return `{ "data": "${data}" }`;
    }
}

class CsvDataProcessor extends DataProcessor {
    // `parseData` 메서드 구현
    protected parseData(data: string): string {
        // CSV 파싱 로직 구현 (예시)
        console.log("Data parsed as CSV.");
        return `"data", "${data}"`;
    }
}
```

```
const jsonDataProcessor = new JsonDataProcessor();
jsonDataProcessor.process(); // 데이터 처리 과정: 로드 -> JSON 파싱 -> 저장

const csvDataProcessor = new CsvDataProcessor();
csvDataProcessor.process(); // 데이터 처리 과정: 로드 -> CSV 파싱 -> 저장
```

### 템플릿 메서드 패턴과 전략 패턴 비교

#### 템플릿 메서드 패턴

-   **정의**: 템플릿 메서드 패턴은 알고리즘의 뼈대를 정의하는 메서드를 제공. 이 패턴은 알고리즘의 일부 단계를 서브클래스에서 구현하도록 한다. 즉, 알고리즘의 구조는 변경하지 않으면서 특정 단계의 구현을 서브클래스에 위임한다.
-   **사용 시나리오**: 공통의 작업 처리 로직을 정의할 때 유용하며, 특정 단계에서의 변화만을 허용한다. 예를 들어, 데이터 처리 작업에서 데이터를 로드하고 저장하는 과정은 동일하지만, 데이터를 파싱하는 방식이 다른 경우에 사용할 수 있다.
-   **구현 방식**: 추상 클래스와 상속을 통해 구현합니다. 추상 클래스에서 알고리즘의 골격을 정의하고, 하나 또는 그 이상의 추상 메서드를 서브클래스에서 구현하도록 한다.

#### 전략 패턴

-   **정의**: 전략 패턴은 동일한 문제를 해결하는 여러 알고리즘(전략)을 정의하고, 이를 실행 시간에 교체할 수 있도록 한다. 전략 패턴은 알고리즘을 사용하는 클라이언트와 독립적으로 알고리즘을 변경할 수 있게 한다.
-   **사용 시나리오**: 클라이언트가 실행 시간에 다른 알고리즘을 선택해야 할 때 유용하다. 예를 들어, 다양한 종류의 정렬 알고리즘이 있을 때 실행 시간에 사용자의 선택에 따라 정렬 방법을 바꿀 수 있다.
-   **구현 방식**: 인터페이스와 구성(composition)을 통해 구현. 여러 알고리즘을 각각 다른 클래스로 구현하고, 이들이 공통 인터페이스를 구현하게 한다. 클라이언트는 필요에 따라 이 전략들 중 하나를 선택하여 사용할 수 있다.

#### 차이점

-   **상속 vs 구성**: 템플릿 메서드 패턴은 상속을 통해 확장하는 반면, 전략 패턴은 구성을 통해 알고리즘을 교체할 수 있다
-   **변경의 유연성**: 전략 패턴은 실행 시간에 알고리즘을 자유롭게 교체할 수 있어 더 유연한 반면, 템플릿 메서드 패턴은 컴파일 시간에 결정된 알고리즘의 구조 내에서만 변경이 가능하다.
-   **사용 목적**: 템플릿 메서드 패턴은 알고리즘의 골격을 정의하는 데 중점을 두는 반면, 전략 패턴은 클라이언트가 사용할 수 있는 알고리즘의 가족을 정의하고 이를 쉽게 교체할 수 있게 하는 데 중점을 둔다.