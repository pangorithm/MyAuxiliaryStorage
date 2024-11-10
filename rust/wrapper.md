![alt text](https://pangorithm.github.io/MyAuxiliaryStorage/image/rust-cheat-sheet.webp)

### `Option<T>`
러스트는 null 이 없기 때문에 값이 있을 수도 있고 없을 수도 있는 경우 사용한다.  
Some은 값이 있는 경우를, None은 값이 없는 경우를 의미한다. 기본 자료형은 값이 없는 경우가 고려되지 않기 때문에 Option은 기본 자료형과 호환되지 않는다. 따라서 Option 열거형을 사용해 값을 지정한 경우에는 match 키워드를 함께 사용해 값을 추출한다.
```rust
fn print_optional(val: Option<String>) {
	match val {
		Some(val) => println!("{}", val),
		None => println!("None"),
	}
}
```

### `Result<T,E>`
러스트는 Result 열거형을 사용해 복구 가능한 오류를 제어한다.  
Ok(T)는 함수의 실행이 성공한 경우 반호나하며, T는 함수 호출이 성공한 경우 반환될 값의 타입을 의미한다. 반대로 Err(E)는 함수의 실행이 실패한 경우 반환되며 E는 오류의 타입을 의미한다.
```rust
let result = File::open("test.txt");

let f = match result {
	Ok(f) => f,
	Err(err) => panic!("함수 실행 실패: {:?}", err)
}
```
match 키워드를 통해 오류를 처리하는 것은 유연하게 대응할 수 있게 해주지만 코드가 장황해지는 단점이 있다. 러스트는 이를 unwrap과 expect, ? 키워드를 제공해 match를 사용하지 않고도 오류에 간편히 대응할 수 있게 해준다.
* `unwrap`: **Result가 Ok(T)일 경우 T의 값을 반환**한다.
```rust
let f = File::open("test.txt").unwrap();
```
* `expect`: Result가 **Err(E)일 경우 지정된 오류 메시지를 출력**한다. 그리고 unwrap과 마찬가지로 **Ok(T)일 경우 T의 값을 반환**한다.
```rust
let f = File::open("test.txt").expect("에러");
```
* `?`: Result 타입의 값이  Ok이면 그 값을 반환하고, Err이면 즉시 함수에서 Err값을 반환한다.
```rust
let mut f = File::open("test.txt")?;

let result = f.expect("파일이 없습니다.");
```

### Box
Box를 사용해 힘 영역에 동적으로 메모리를 할당 받을 수 있다. 일종의 스마트 포인터 역할을 한다. 메모리를 할당받으려면 `Box::new` 를 사용하고, 할당받은 메모리에 접근하려면 `*` 키워드를 사용한다. 동적으로 생성한 x를 따로 해제해 주지 않아도 데이터가 더이상 사용되지 않는 시점에 알아서 해제된다.
```rust
let mut x = Box::new(10);
*x = 20;
```

### Rc
Rc는 레퍼런스 카운팅 포인터이다. Rc로 관리되는 데이터는 공유가 가능해 여러 변수가 동일한 값을 참조할 수 있도록 한다. 이때 Rc는 데이터를 참조하는 모든 변수들이 메모리에 존재하는 한 해당 값을 해제하지 않는다.  
Box와 다른 점은 Box는 빌림(`&`) 방식 외에는 공유가 불가능한 반면, Rc는 공유가 가능하다.
```rust
let person = Rc::new(Person { age: 10 });

let p1 = person.clone();
println!("{}", Rc::strong_count(&person)); // 2
let p2 = person.clone();
println!("{}", Rc::strong_count(&person)); // 3
{
let p3 = person.clone();
  println!("{}", Rc::strong_count(&person)); // 4
}
println!("{}", Rc::strong_count(&person)); // 3
```

### RefCell
Rc는 불변성을 가진 참조형이어서 공유 데이터를 변경할 수 없다. 이를 해결하기 위해 RefCell은 변경 불가능한 변수를 임시로 변경 가능하게 해주는 기능을 제공하여 Rc와 함께 쓰는 경우가 많다.
```rust
struct Person {
	name: String,
	age: i32,
	next: RecCell<Option<Rc<Person>>>, // RefCell으로 인해 불변 참조에서도 수정 가능
}

let mut p1 = Rc::new(Person {
	name: String::from("Luna"),
	age: 30,
	next: RefCell::new(None),
});
let mut p2 = Rc::new(Person {
	name: String::from("Rust"),
	age: 20,
	next: RefCell::new(None),
});
let mut next = p1.next.borrow_mut(); // 가변 참조 얻기
*next = Some(p2.clone());
```

### Weak
Rc의 경우 참조 카운트가 0이 되어야지만 해제되기 때문에 순환참조가 발생할 경우 메모리에서 해제되지 않는 문제가 발생한다. 따라서 순환 참조를 이용할 경우 Weak 객체를 이용해서 이 문제를 해결할 수 있다. Weak 객체는 참조 카운트를 증가시키지 않기 때문에 원본 Rc 객체가 해제될 때 함께 해제된다.
```rust
let weak = RC::doungrade(&p);
match weak.upgrade() {
    Some(rc) => rc,
    None => None
}
```

### `Mutex<T>`
lock과 unlock으로 하나의 스레드만 자원에 접근할 수 있도록 하여 동시성을 제어한다. 자원에 접근하기 위해서는 `*`키워드를 사용한다.
```rust
static counter: Mutex<i32> = Mutex::new(0); // conuter를 전역 변수로 정의

fn inc_counter() {
    let mut num = counter.lock().unwrap();
    *num = *num + 1;
} // inc_counter를 벗어나는 순간 counter는 unlock됩니다.
```

### Semaphore
뮤텍스와는 다르게 복수의 자원을 보호하는데 사용한다. 세마포어를 사용하기 위해서는 tokio 크레이트를 사용해야 한다. 세마포어는 뮤텍스와 달리 임계 지점을 직접 지정해야 하므로 획득 시점(`acquire_owned`)과 해제 시점(`drop`)을 직접 설정해야 한다.
```rust
static counter: Mutex<i32> = Mutex::new(0); // conuter를 전역 변수로 정의

#[tokio::main]
async fn main() {
    // 동시에 2개의 스레드가 접근 가능하도록 세마포어 설정
    let semaphore = Arc::new(Semaphore::new(2));
    let mut future_vec = vec![];

    for _ in 0..100 {
        // 세마포어 획득
        let permit = semaphore.clone().acquire_owned().await.unwrap();
        let future = tokio::spawn(async move {
            let mut num = counter.lock().unwrap(); // 뮤텍스로부터 안전한 참조를 획득
            *num = *num + 1; // 카운터 증가

            drop(permit); // 세마포어 해제
        });
        future_vec.push(future); // 생성된 future를 벡터에 저장
    }

    for future in future_vec {
        future.await.unwrap(); // 모든 future가 완료될 때까지 대기
    }

    println!("결과: {}", *counter.lock().unwrap()); // 최종 결과 출력
}

```

### `Arc<T>`
원자적 참조 카운트를 의미한다. Arc는 동시적 상황에 사용 가능한 Rc라고 생각하면 된다. 여러 스레드에 걸쳐 데이터의 소유권을 공유하는데 사용할 수 있다.
```rust
    let counter = Arc::new(Mutex::new(0)); // 공유될 카운터를 Arc와 Mutex로 감싸준다.
    let mut thread_vec = vec![]; // 스레드를 저장할 벡터

    for _ in 0..100 {
        let _cnt = counter.clone(); // 현재 카운터의 클론을 생성한다. Arc를 사용하면 여러 스레드 간에 안전하게 공유할 수 있다.
        let th = thread::spawn(move || {
            let mut num = _cnt.lock().unwrap(); // 뮤텍스로부터 안전하게 락을 얻어와 참조를 획득한다.
            *num = *num + 1;
        });
        thread_vec.push(th);
    }

    for th in thread_vec {
        th.join().unwrap(); // 모든 스레드가 완료될 때까지 기다린다.
    }
```
