# 프로젝트 구조
표준 러스트 프로젝트 구조는 src 디렉터리와 Cargo.toml 파일을 중심으로 구성된다.  
src 디렉터리에는 러스트의 소스 파일이 위치하며, 특히 main.rs 또는 lib.rs와 같은 엔트리 포인트 파일이 프로그램의 시작점을 정의한다.  
Cargo.toml 파일은 프로젝트의 메타데이터를 포함하며, 프로젝트가 의존하는 외부 라이브러리나 패키지에 대한 정보를 명시한다. 이 파일은 러스트의 패키지 관리자인 Cargo에 의해 참조되며, Cargo는 이 정보를 바탕으로 필요한 패키지를 다운로드 하고 빌드 과정에서 이를 연결한다.  
러스트는 코드를 모듈로 구분해 관리한다. 모듈은 `mod.rs` 파일이나 모듈의 이름과 동일한 `.rs` 파일로 정의할 수 있다. Cargo.tml 파일과 연관된 소스코드 디렉터리를 가지며 이는 Cargo의 기본 단위이다.  
의존성 관리와 버전 컨트롤 또한 러스트 프로젝트에서 매우 중요한 부분이다. Cargo.toml 파일 내의 의존성 섹션에 프로젝트가 필요로 하는 외부 패키지의 이름과 버전을 정확히 명시하면 Cargo는 이를 자동으로 관리한다. 또한 Cargo.lock 파일을 통해 각 의존성의 정확한 버전이 기록되어, 다른 개발 환경에서도 동일한 조건과 환경으로 프로젝트를 빌드하고 실행하도록 도와준다.

### 모듈화
러스트는 모듈화를 위해 `mod`와 `use`라는 키워드를 제공한다. 
##### 장점
* 재사용성: 하나의 모듈은 서로 다른 응용 프로그램에서 재사용 가능하다.
* 유지 보수성: 모듈마다 기능이 독립적으로 구성된다.
* 개발 생산성: 개발자 본인이 맡은 모듈만 책임지기 때문에 협업이 용이하다.
* 자원 효율성: 필요한 모듈만 로드아혀 쓸 수 있기 때문에 메모리 사용량을 줄일 수 있다.
##### 단점
* 성능 저하: 모듈 간 통신하기 위한 비용이 추가로 들어간다. 단순 호출의 경우 성능에 큰 영향을 주지 않을 수도 있지만, 상황에 따라 IPC(Inter Process Communication)을 해야 하는 경우 성능에 직접적인 영향을 준다.
* 의존성 관리: 과도한 모듈화는 모듈 간의 의존성 관리를 어렵게 만든다.

### 모듈 생성
`cargo new adder --lib`: 라이브러리 크레이트를 만들려면  --lib 옵션을 추가해야 한다.  
main 함수가 없으므로 `cargo run`으로 실행 시 오류 메시지가 나오기 때문에 `cargo test`로 실행해야 한다.  
해당 모듈을 사용하기 위해서는 해당 모듈을 사용할 크레이트의 `Cargo.toml`의 `dependencies`에 `adder = { path = "../adder" }`와 같이 의존성을 추가해야 한다.

### mod로 계층 구성하기
계층화 아키텍처는 소프트웨어 시스템 설계 및 구현에서 널리 사용되는 설계 패턴 중 하나이다. 이 아키텍처 패턴은 시스템을 여러 계층으로 나누고 각 계층이 특정 책임과 역할을 수행하도록 설계한다. 이러한 계층화 구조는 백엔드 시스템뿐만 아니라 애플리케이션 및 응용 시스템에서도 적용 가능하며 여러 이점이 있다.  
주요 특징 및 이점으로는 모듈화를 통해 각 계층이 독립적으로 개발, 테스트, 유지보수 가능하며 변경 또는 확장 시 해당 계층만 수정할 수 있어 모듈화를 강화한다. 또한 각 계층의 재사용성 또한 높이며 다른 프로젝트나 시스템에서 동일한 계층을 활용할 수 있다.  
일반적으로 사용되는 계층에는 표현 계층, 응용 계층, 데이터 계층이 있다.  
```rust
// 프레젠테이션 레이어를 나타내는 모듈
pub mod presentation {
    // 뷰에 관련된 기능을 담당하는 모듈
    pub mod view {
        // 렌더링에 관련된 함수
        pub fn render() {
            println!("mysystem::presentation::view::render");
        }
    }
}

// 비즈니스 로직을 담당하는 모듈
pub mod business {
    // 사용자 관련 비즈니스 로직을 담당하는 모듈
    pub mod user {
        // 사용자 생성에 관련된 함수
        pub fn create() {
            println!("mysystem::business::user::create");
        }
    }
}

// 데이터베이스 작업을 담당하는 모듈
pub mod database {
    // 사용자 데이터 액세스 객체(DAO)를 나타내는 모듈
    pub mod user_dao {
        // 사용자 생성에 관련된 데이터베이스 함수
        pub fn create() {
            println!("mysystem::database::user_dao::create");
        }
    }
}

#[test]
fn it_works() {
    presentation::view::render();
    business::user::create();
    database::user_dao::create();
}
// 테스트 모드는 println을 출력하지 않으므로 cargo test -- --nocapture으로 실행해야 출력 결과가 확인 가능하다.
```

### 모듈 단위로 파일 분리
러스트에서는 폴더명과 파일명을 기준으로 모듈을 찾는다. 예를 들어 business 라는 모듈을 사용한다면, business라는 폴더가 있는지 확인한 다음 business라는 폴더가 있고 mod.rs라는 파일이 있다면 해당 파일을 불러온다. 그 다음 business.rs라는 파일이 있는지 찾는다.

##### use 사용하기
```rust
// 렌더링에 관련된 함수
pub fn render() {
    println!("mysystem::presentation::view::render");
    // business::user::create(); // 빌드 오류 발생
    super::super::business::user::create(); // 각 모듈은 현재 위치를 기준으로 모듈을 찾으므로 ../와 같은 역할을 하는 super 키워드 사용
    crate::business::user::create(); // crate라는 키워드를 사용해 크레이트를 기준으로 모듈을 탐색
}
```


### 가시성 제어
러스트는 가시성(visibility) 라는 방법으로 객체지향의 정보 은닉과 캡슐화를 지웒나다.
* `pub`: 정보를 제한 없이 외부에 공개한다.
* `pub(in 대상_모듈)`: 제공된 경로에 한정해 정보를 제공한다.
* `pub(crate)`: 현재 crate에 한정해 정보를 제공한다.
* `pub(super)`: 상위 모듈에 정보를 제공한다.
* `생략 혹은 pub(self)`: 정보를 외부에 공개하지 않는다.
```rust
pub mod my_module {
    // 이 함수는 외부에서 접근 가능합니다.
    pub fn public_fn() {}

    // 이 함수는 my_module에서만 접근 가능합니다.
    fn public_fn() {}

    // 이 함수는 현재 크레이트에 정보를 제공합니다.
    pub(crate) fn public_fn() {}

    // 이 함수는 상위 모듈에 정보를 제공합니다.
    pub(super) fn public_fn() {}
}
```