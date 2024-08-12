### 싱글톤 패턴

하나의 클래스에 오직 하나의 인스턴트만 가지는 패턴. 일반적으로 데이터베이스 연결 모듈이나 쓰레드 풀 등의 공용 자원에 많이 사용된다. 하나의 인스턴스를 만들어 놓고 해당 인스턴스를 다른 모듈들이 공유하며 사용하기 때문에 인스턴스 생성 비용이 줄어든다는 장점과 의존성이 높아진다는 단점이 있다.

```
class Singleton {
  constructor() {
    if (!Singleton.instance) {
      Singleton.instance = this;
    }
    return Singleton.instance;
  }

  // 예제 메서드
  someMethod() {
    console.log('Doing something...');
  }
}

// 인스턴스를 생성하는 함수
const getInstance = (function() {
  let instance;

  return function() {
    if (!instance) {
      instance = new Singleton();
    }
    return instance;
  };
})();

// 예제 사용
const singletonInstance1 = getInstance();
const singletonInstance2 = getInstance();

singletonInstance1.someMethod(); // Doing something...

console.log(singletonInstance1 === singletonInstance2); // true
```

### 장점

인스턴스가 하나만 생성 되기 때문에  인스턴스 생성에 의한 오버헤드와 메모리 낭비를 방지할 수 있다.

다양한 컴포넌트나 클래스에서 어디서나 해당 인스턴스를 공유할 수 있다.

인스턴스의 초기화 시점을 제어할 수 있으며, 필요할 때 생성되도록 지연 초기화를 구현할 수 있다.

### 단점

사실상 전역 상태를 만들어 코드의 결합도를 높여 테스트의 격리성을 해친다.

멀티 스레드 환경에서 추가적인 동기화 처리가 필요하기 때문에 성능 저하가 일어날 수 있다.

결합이 강해진다는 단점은 의존성 주입을 통해 모듈간의 결합을 느슨하게 만들어 해결 할 수있다.

### 의존성 주입 (Dependency Inversion, DI)

-   장점: 구현할 때 추상화 레이어를 넣고 이를 기반으로 구현체를 넣어 주기 때문에 애플리케이션 의존성 방향이 일관되고, 애플리케이션을 쉽게 추론할 수 있으며, 모듈간의 관계들이 더 명확해진다. 이로인해 모듈들을 쉽게 교체할 수 있는 구조가 되어 테스팅이나 마이그레이션 하기에 수월해진다.
-   단점: 모듈들이 더욱더 분리되므로 클래스 수가 늘어나 복잡성이 증가 될 수 있으며 이로 인해 약간의 런타임 패널티가 생기기도 한다.
-   원칙: 상위 모듈은 하위 모듈에서 어떠한 것도 가져오지 않아야 한다. 또한 둘 다 추상화에 의존해야 하며, 이때 추상화는 세부 사항에 의존하지 말아야 한다.