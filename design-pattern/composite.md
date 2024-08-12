### **특징**

클라이언트가 개별 객체와 객체의 조합을 동일하게 다룰 수 있도록 한다. 이 패턴의 핵심은 개체와 개체의 컬렉션 간의 작업을 일관된 방식으로 처리할 수 있게 하는 것이다. 즉, 클라이언트는 개별 객체와 그 객체들의 조합을 구분하지 않고 동일한 인터페이스를 통해 다룰 수 있다. 컴포지트 패턴은 트리 구조를 이용해 전체-부분(Part-Whole) 계층을 표현하며, 이를 통해 사용자가 단일 객체와 객체의 집합을 동일하게 처리 한다.

### **구성 요소**

-   Component
    -   개별 객체와 객체의 컬렉션에 공통적인 인터페이스 또는 추상 클래스
    -   클라이언트는 이 인터페이스를 통해 개별 객체와 그 조합을 동일하게 처리할 수 있다.
    -   일반적으로, 컴포넌트는 자식을 추가, 제거하거나, 자식을 가지고 있는지 조회하는 등의 인터페이스를 제공
-   Leaf
    -   컴포넌트 인터페이스를 구현하며, 자식을 가질 수 없는 개별 객체를 나타낸다.
    -   구조의 기본적인 동작을 정의하며, 컴포지션의 '잎사귀' 역할을 합니다.
-   Composite
    -   컴포넌트 인터페이스의 구현체, 자식 컴포넌트를 가질 수 있는 객체
    -   복합체는 자식 컴포넌트들을 저장하는 컨테이너 역할을 하며, 컴포넌트 인터페이스에서 정의된 동작을 자식에게 위임한다.
    -   이는 부분-전체 계층을 표현하는 데 사용된다.

### **동작원리**

1.  복합체 객체에 인터페이스에서 정의된 동작을 실행시킨다.
2.  복합체는 leaf에 도달할 때까지 재귀적으로 자식 객체들에 해당 동작을 실행시킨다.
3.  잎 객체에 도달 하면 잎 객체가 해당 동작을 수행하고 결과를 다시 부모 객체에 반환한다.
4.  복합체에 내린 명령은 결과적으로 모든 하위 객체에 명령을 내린 것과 동일하게 수행된다.

### **사용 사례**

-   파일 시스템에서 폴더(Composite)와 파일(Leaf)의 관계를 표현할 때
-   UI 컴포넌트에서 복잡한 위젯(Composite)과 간단한 컨트롤(Leaf)의 관계를 모델링할 때
-   회사 조직도에서 부서(Composite)와 직원(Leaf)의 계층 구조를 나타낼 때

예시코드

```
// Component 인터페이스
interface FileSystemComponent {
    void printName(): void;
}

// Leaf 클래스
class File implements FileSystemComponent {
    constructor(private name: string) {}

    printName(): void {
        console.log(`File: ${this.name}`);
    }
}

// Composite 클래스
class Directory implements FileSystemComponent {
    private children: FileSystemComponent[] = [];

    constructor(private name: string) {}

    add(child: FileSystemComponent): void {
        this.children.push(child);
    }

    printName(): void {
        console.log(`Directory: ${this.name}`);
        this.children.forEach(child => child.printName());
    }
}
```

```
// 파일과 디렉토리 생성
const file1 = new File("File1.txt");
const file2 = new File("File2.txt");
const file3 = new File("File3.txt");

const directory1 = new Directory("Directory1");
const directory2 = new Directory("Directory2");

// 디렉토리에 파일과 다른 디렉토리 추가
directory1.add(file1);
directory1.add(file2);
directory1.add(directory2);

directory2.add(file3);

// 구조 출력
directory1.printName(); // 디렉토리와 그 내용물의 이름을 출력
```