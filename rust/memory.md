# 동적 메모리 할당

### Box
힙 영역에 동적으로 메모리를 할당받으며 일종의 스마트 포인터 역할을 한다. 할당 받은 메모리에 접근하려면 `*` 키워드를 사용한다.
```rust
fn main() {
  let mut x = Box::new(10); //i32형식의 데이터를 힙에 할당
  println!("x: {}", x); // x: 10

  *x = 20; // x를 변경
  println!("x: {}", x); // x: 20
}
```

### Rc(reference-counting pointer)
Rc로 관리되는 데이터는 공유가 가능해 여러 변수가 동일한 값을 참조할 수 있도록 한다. 이때 RC는 데이터를 참조하는 모든 변수들이 메모리에 존재하는 동안 해당 값을 해제하지 않는다.
Box는 빌림 방식 외에는 공유가 불가능한 데에 반해, RC는 공유가 가능하다. 따라서 여러 변수에 값을 공유해야 하는 경우 유연하게 사용 할 수 있다. 또한 Rc는 불변성을 가진 참조형이다.

```rust
use std::rc::Rc;

struct Person {
  age: i32,
}

fn main() {
  // person을 공유 객체로 생성
  let person = Rc::new(Person { age: 10 });

  // person 복제
  let p1 = person.clone();
  println!("person: {}, p1: {}", person.age, p1.age); // person: 10, p1: 10
  // 참조 횟수를 반환하여 출력
  println!("RefCount: {}", Rc::strong_count(&person)); // RefCount: 2
  
  // person 복제
  let p2 = person.clone();
  println!("person: {}", person.age, p1.age);
  println!("RefCount: {}", Rc::strong_count(&person)); // RefCount: 3
  
  {
    // person 복제
    let p3 = person.clone();
    println!("person: {}", person.age, p1.age);
    println!("RefCount: {}", Rc::strong_count(&person)); // RefCount: 4
    // p3 소멸
  }
  
  println!("RefCount: {}", Rc::strong_count(&person)); // RefCount: 3
}
```


### RefCell
Rc는 불변성을 가진 참조형이기 때문에 공유 데이터를 변경할 수 없다. 이러한 문제를 해결하기 위해 러스트는 RefCell이라는 자료형을 제공하며 RefCell은 변경 불가능한 변수를 임시로 변경 가능하게 해주는 기능을 제공한다. 그래서 Rc와 함께 쓰이는 경우가 많다. `*` 키워드를 사용하면 RefCell로 선언한 자료에 접근할 수 있다.
```rust
use std::rc::Rc;
use std::cell::RefCell; // RefCell은 내부 가변성을 제공하며, 런타임에 빌림 규칙을 검사한다.

struct Person {
  name: String,
  age: i32,
  next: RefCell<Option<Rc<Person>>>, // RefCell로 래핑되어 있기 때문에 불변 참조에서도 수정 가능
}

fn main() {
  let p1 = Rc::new(Person{
    name: String::from("Luna"),
    age: 30,
    next: RefCell::new(None),
  });
  
  // person 복제
  let p2 = Rc::new(Person{
    name: String::from("Rust"),
    age: 10,
    next: RefCell::new(None),
  });

  // p1의 next 필드에 대한 가변 참조를 얻음
  let mut next = p1.next.borrow_mut();
  *next = Some(p2.clone()); // p1 뒤에 p2를 추가해 연결 리스트를 연결
}
```

### 약한 참조
Rc는 참조 횟수를 추적해 참조 횟수가 0이 되면 데이터를 삭제하지만 순환 참조 등의 경우 참조 횟수가 0이 되지 않아 메모리 누수가 발생한다.

##### 참조 순환으로 인한 메모리 누수
```rust
use std::rc::Rc;
use std::cell::RefCell;

struct Person {
  id: i32,
  next: RefCell<Option<Rc<Person>>>,
}

// 참조 횟수가 0이 되는 순간 호출됨
impl Drop for Person {
  fn drop(&mut self) {
    println!("p{} Drop!", self.id);
  }
}

fn main() {
  let mut p1 = Rc::new(Person {
    id: 1,
    next: RefCell::new(None),
  });
  
  let mut p2 = Rc::new(Person {
    id: 2,
    next: RefCell::new(None),
  });

  let mut p3 = Rc::new(Person {
    id: 3,
    next: RefCell::new(None),
  });

  // 순환참조 발생
  let mut next = p1.next.borrow_mut();
  *next = Some(p2.clone());
  
  let mut next = p2.next.borrow_mut();
  *next = Some(p1.clone());

  println!("p1 RefCount: {}, p2 RefCount: {}",
    Rc::strong_count(&p1), Rc::strong_count(&p2));
  // p1 RefCount:2, p2 RefCount:2
  // p3 Drop!
}
```

##### 약한 참조를 사용한 순환 참조 해결
Weak타입은 참조 카운트를 증가 시키지 않는다.
```rust
use std::rc::{Rc, Weak};
use std::cell::RefCell;

struct Person {
  id: i32,
  next: RefCell<Option<Weak<Person>>>,
}

// 참조 횟수가 0이 되는 순간 호출됨
impl Drop for Person {
  fn drop(&mut self) {
    println!("p{} Drop!", self.id);
  }
}

fn main() {
  let mut p1 = Rc::new(Person {
    id: 1,
    next: RefCell::new(None),
  });
  
  let mut p2 = Rc::new(Person {
    id: 2,
    next: RefCell::new(None),
  });

  println!("p1 RefCount: {}, p2 RefCount: {}",
    Rc::strong_count(&p1), Rc::strong_count(&p2));
  // p1 RefCount:1, p2 RefCount:1

  // 순환참조 발생
  let mut next = p1.next.borrow_mut();
  *next = Some(Rc::downgrade(&p2)); // weak 방식으로 p2 추가
  
  let mut next = p2.next.borrow_mut();
  *next = Some(Rc::downgrade(&p2)); // weak 방식으로 p1 추가

  println!("p1 RefCount: {}, p2 RefCount: {}",
    Rc::strong_count(&p1), Rc::strong_count(&p2));
  // p1 RefCount:1, p2 RefCount:1
  // p2 Drop!
  // p1 Drop!
}
```

### Box와 Rc
* 공통점
  1. 힙 메모리에 할당된다.
* 차이점
  1. Box: 소유권을 가지고 있다. Rc: 참조 횟수를 관리해 데이터를 여러 변수와 공유할 수 있다.
  2. Box: 가변 변수로 사용 가능하다. Rc: 불변성으로 인해 변경이 불가능하다.
이러한 특성으로 인해 외부 공유가 필요한 경우 Rc를 사용하고, 그렇지 않다면 Box를 사용하는 것이 효율적이다. (Rc 변수를 변경하려면 RefCell과 같은 기법을 사용해야 하여 코드가 복잡해질 수 있으며 참조 횟수 추적에 비용이 발생하기 때문)

### 라이프타임 지시자
lifetime 지시자는 변수를 대여할 때 대여 기간을 명시적으로 지정하기 위해 사용한다. 라이프타임 지시자는 `'`를 사용해 정의한다.  

라이프타임 매개변수는 아무 이름이나 지정할 수 있지만 관례적으로 `'a, 'b, 'c`와 같은 알파벳을 사용한다. 매개변수 참조는 `&'a str` 형식으로 선언되며 이는 문자열 슬라이스의 라이프 타임이 `'a`와 같다는 의미이다.  

함수에서의 라이프타임 지시자 사용
```rust
fn 함수이름<'a> (매개변쉬 &'a str) -> 'a str {
   (...생략...)
}
```

구조체에서의 라이프타임 지시자 사용
```rust
struct 구조체이름<'a> {
  변수: &'a str,
}
```

라이프타임 지시자는 변수의 소멸 시점이 명확하지 않은 경우 발생하는 오류를, 반환 시점을 명시해 줌으로서 해결해준다.

```rust
// 런타임에 판단해 빌림을 반환하는 케이스
// x, y, 반환 타입 모두 소멸 시점이 명확히 드러나지 않아 컴파일 오류 발생
// error[E0106]: missing lifetime specifier
/*
fn longest(x: &str, y: &str) -> &str {
    if x.len() > y.len() {
        x
    } else {
        y
    }
}
*/

fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() {
        x
    } else {
        y
    }
}

fn main() {
    let s1 = String::from("Hello");
    let s2 = String::from("Rust");

    let result = longest(&s1, &s2);
    println!("{}와 {} 중 더 긴 문자열은 '{}'", s1, s2, result); 
    // Hello와 Rust 중 더 긴 문자열은 'Hello'
}
```

### 정적 변수
정적 변수는 프로그램이 종료될 때까지 메모리에 유지된다. 정적 라이프타임 지시자`&'static`를 사용해 쉽게 생성할 수 있다.
```rust
static GLOBAL_CONST: i32 = 10; // 프로그램이 종료될 때까지 메모리에서 해제되지 않음

fn main() {
  let x: &'static str = "Hello Rust!"; // x는 프로그램이 종료될 때까지 메모리에서 해제되지 않음
  println!("x: {}", x);
  println!("GLOBAL_CONST: {}", GLOBAL_CONST);
}
```