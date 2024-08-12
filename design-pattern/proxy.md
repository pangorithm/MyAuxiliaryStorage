### 목적

어떤 다른 객체에 대한 접근을 제어하기 위해 대리인 또는 자리표시자를 제공하기 위한 패턴. 이는 해당 객체의 생명주기를 관리하거나, 실제 객체의 호출을 가로채서 추가적인 기능을 수행하기 위함이다. 프록시 패턴은 실제 객체와 같은 인터페이스를 구현함으로써, 클라이언트가 실제 객체와 프록시를 동일하게 다룰 수 있도록 한다.

### 유형

-   **가상 프록시 (Virtual Proxy):**  실제 객체의 생성 비용이 클 때 초기화 지연을 사용하여 리소스를 절약한다. 객체가 실제로 필요할 때까지 생성을 지연시킨다.
-   **보호 프록시 (Protection Proxy):** 실제 객체에 대한 접근 권한을 제어한다.
-   **원격 프록시 (Remote Proxy):** 네트워크 연결을 통해 객체에 대한 로컬 대표를 제공한다. 원격 서버에 있는 객체를 로컬 객체처럼 사용할 수 있게 해준다.
-   **로깅** **프록시 (Logging Proxy):**  서비스 객체에 대한 요청을 기록한다.
-   **캐싱 프록시 (Caching Proxy):**  항상 같은 결과를 생성하는 반복 요청들에 대한 캐싱을 수행한다.
-   **스마트 참조 프록시 (Smart Reference Proxy):**  클라이언트를 점검하여 활성화 상태인지 확인한다.

### 구성요소

-   **Subject:** 클라이언트가 사용하는 인터페이스입니다. 실제 객체와 프록시 모두 이 인터페이스를 구현해야 한다.
-   **RealSubject:** 프록시가 대신하여 요청을 전달하는 실제 객체.
-   **Proxy:** Subject의 인터페이스를 구현하며, 실제 객체에 대한 참조를 유지한다. 클라이언트의 요청을 받고, 필요에 따라 이 요청을 RealSubject에 전달하기 전이나 후에 추가적인 처리를 할 수 있다.

### 다른 패턴과의 차이점

-   **인터페이스의 관점**
    -   **어댑터**: 다른 인터페이스 제공
    -   **프록시**: 같은 인터페이스 제공
    -   **데코레이터**: 향상된 인터페이스를 래핑된 객체에 제공
-   **제어의 관점**
    -   **프록시**: 자체적으로 자신의 서비스 객체의 수명 주기를 관리한다. 
    -   **데코레이터**: 항상 클라이언트에 의해 제어된다. 
    -   **공통점**: 한 객체가 일부 작업을 다른 객체에 위힘해야 하는 합성 원칙을 기반으로 원래 객체와 동일한 인터페이스를 요구한다.
-   **목적의 관점**
    -   **프록시**: 객체에 대한 접근 제어
    -   **데코레이터**: 기능의 확장, 추가, 변경

### 예시 코드

```
interface Image {
  display(): void;
}

class RealImage implements Image {
  private fileName: string;

  constructor(fileName: string) {
    this.fileName = fileName;
    this.loadFromDisk(fileName);
  }

  display(): void {
    console.log(`Displaying ${this.fileName}`);
  }

  private loadFromDisk(fileName: string): void {
    console.log(`Loading ${fileName}`);
  }
}

class ProxyImage implements Image {
  private realImage: RealImage | null = null;
  private fileName: string;

  constructor(fileName: string) {
    this.fileName = fileName;
  }

  display(): void {
    if (this.realImage === null) {
      this.realImage = new RealImage(this.fileName);
    }
    this.realImage.display();
  }
}

// 사용 예
function clientCode(image: Image) {
  // 이미지를 표시하는 작업
  image.display();
}

const image1 = new ProxyImage("testImage1.jpg");
const image2 = new ProxyImage("testImage2.jpg");

clientCode(image1); // 처음 호출할 때는 로딩 과정을 거칩니다.
clientCode(image1); // 이미 로딩이 되어있기 때문에 바로 표시됩니다.
clientCode(image2); // 새로운 이미지는 로딩 과정을 거칩니다.
```