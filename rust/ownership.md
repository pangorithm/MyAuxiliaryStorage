# 소유권
### 특징
* 메모리 안정성: 강력하고 엄격한 소유권 검사기를 통해 메모리 안정성을 보장하고 프로그램에서 발생 가능한 오류를 미연에 방지.
* 성능: GC의 오버헤드 없이 빠르고 효율적으로 메모리를 관리
* 스레드 안전성: 소유권은 하나의 값을 하나의 스레드에만 접근 가능하도록 한정한다. 따라서 안전한 동시성을 지원하며 자원 경쟁과 같은 문제점을 미연에 방지한다.
* 단점: 러스트만의 독창적인 개념이므로 러닝커브가 높고 소유권을 만족시키기 위해 코드 구조가 장황해지고 복잡해진다.

### 소유권 이전
```rust
fn main() {
  // 새로운 문자열 변수 생성
  let s1 = String::from("Hello Rust");

  // s2로 소유권 이동
  let s2 = s1;

  // s1은 소유권을 상실했기 때문에 s1에접근하는 순간 컴파일 에러 발생
  println!("{}", s1);
}
```


```rust
fn main() {
  // 새로운 문자열 변수 생성
  let s1 = String::from("Hello Rust");

  // s1을 복제해 s2에 저장
  let s2 = s1.clone();

  // s1은 여전히 소유권을 갖고 있기 때문에 사용 가능
  println!("{}", s1);
}
```

### 소유권 빌림
```rust
fn main() {
  let s = String::from("Hello");
  push_str(s); // push_str 함수에 소유권 이관

  // s를 사용하는 순간 컴파일 오류 발생
  println!("{}", s);
}

fn push_str(mut s: String) {
  s.push_str(" Rust!");
}
```


```rust
fn main() {
  let s = String::from("Hello");
  let s = push_str(s); // push_str 함수에 소유권을 전달하고, 섀도잉으로 s를 획득

  // s를 사용하는 순간 컴파일 오류 발생
  println!("{}", s);
}

fn push_str(mut s: String) -> String {
  s.push_str(" Rust!");
  s
}
```

```rust
fn main() {
  let s = String::from("Hello");
  
  // s의 소유권을 push_str 함수에 대여
  push_str(&mut s); 

  // s는 소유권을 유지하므로 정상 작동
  println!("{}", s);
}

fn push_str(s: &mut String) {
  s.push_str(" Rust!");
}
```

### 데이터 복제

```rust
// copy 트레잇을 적용한다.
#[derive(Copy, Clone)]
struct Student {
  age: i32,
}


fn main() {
  let mut s1 = Student { age: 10 };
  // copy 트레잇이 적용되어 있기 때문에 소유권 이전이 아니라 s1을 복사해 s2에 저장한다.
  let s2 = s1;

  println("s1: {}, s2: {}", s1.age, s2.age); // s1: 10, s2: 10

  s1.age = 20;
  println("s1: {}, s2: {}", s1.age, s2.age); // s1: 20, s2: 10
}
```

단, Copy 트레잇은 모든 자료형에 적용이 어렵다. 대표적으로 String의 경우 sopy 트레잇이 구현되어 있지 않기 때문에 위의 예제와 같이 copy 트레잇을 적용하더라도 컴파일 오류가 발생한다. 따라서 이런 경우 명시적으로 clone 함수를 사용해 값을 복제해야 한다.