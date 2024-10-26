# 동시성
동시성과 병렬성은 비슷하지만 조금 다른 개념이다.  동시성은 시스템이 여러 작업을 동시에 수행할 수 있는 것(번갈아 가며 작업한다 하더라도)을 말하며, 병렬성은 여러 작업이 실제로 동시에 수행하는 것을 말한다.  

러스트는 동시성에서 발생할 수 있는 문제를 소유권 개념을 활용해 여러 스레드가 동시에 자료에 접근할 수 없도록 강제하여, 교착 상태나 경쟁 상태 같은 문제를 사전에 예방한다.  

안전하고 효율적인 방식으로 동시성 프로그래밍을 할 수 있도록 스레드, 채널, async/await 등 다양한 기능을 제공한다.  

### 이벤트루프 vs 스레드
* 특징: 
  * 이벤트루프: 단일 스레드를 사용하고 이벤트 큐에 수신된 이벤트를 애플리케이션이 종료될 때까지 무한 반복해 처리하는 방식
  * 스레드: 스레드 별로 작업이 할당되는 방식.
* 장점:
  * 이벤트루프: 최소한의 오버헤드로 많은 이벤트를 처리할 수 있음. 복잡한 동기화 메커니즘이 필요하지 않음
  * 스레드: CPU 코어가 충분할 경우 복수의 작업을 병렬화 해 빠르게 전체 작업을 수행할 수 있음
* 단점:
  * 이벤트루프: 단일 스레드를 사용하므로 CPU 연산이 많은 작업에는 적합하지 않음.
  * 스레드: CPU 코어 개수를 넘어설 정도로 많은 양의 처리를 동시에 수행하면 컨텍스트 스위칭 비용이 발생해 성능이 지연될 수 있음. 복잡한 동기화 메터니즘이 필요
* 사용 예:
  * 이벤트루프: UI가 있는 애플리케이션. I/O 작업이 많은 서비스 등
  * 스레드: CPU 연산량이 많은 작업(알고리즘 등)

### 동시성 제어 기법
* 뮤텍스: 상호 배제(mutual exclusion)의 약자로, 여러 스레드가 공유 자원에 동시에 접근하지 못하도록 막는 기법. 잠금과 해제 두 가지 상태가 존재하며, 뮤텍스가 어떤 자원을 잠그면 다른 스레드는 자원이 해제될 때까지 대기한다. 하나의 자원은 하나의 스레드만 접근할 수 있어 효과적으로 동시성을 제어할 수 있다. 하지만 개발자의 부주의로 해제하지 않을 경우 다른 스레드는 자원이 해제될 때까지 무한정 대기하는 교착 상태가 발생할 수 있다.
```runst
use std::sync::Mutex;
use std::thread;

static counter: Mutex<i32> = Mutex::new(0); // conuter를 전역 변수로 정의

fn inc_counter() {
    let mut num = counter.lock().unwrap();
    *num = *num + 1;
} // inc_counter를 벗어나는 순간 counter는 unlock됩니다.

fn main() {
    let mut thread_vec = vec![];

    for _ in 0..100 {
        let th = thread::spawn(inc_counter); // counter를 증가합니다.
        thread_vec.push(th);
    }

    for th in thread_vec {
        th.join().unwrap();
    }

    println!("결과: {}", *counter.lock().unwrap()); // counter 값을 획득합니다.
}
```
* 세마포어: 복수의 제한된 자원에 다수의 스레드가 동시에 접근하는 것을 막는 동시성 제어 방법. 뮤텍스와 비슷하게 보일 수 있으나, 가장 큰 차이점은 세마포어는 복수의 자원을 보호하는데 사용하는 기법이고, 뮤텍스는 단일 자원을 보호하는 데 사용하는 기법이다. 또한 세마포어는 스레드를 넘어 프로세스 간의 공유 자원을 보호하는데도 사용할 수 있다. 세마포어는 뮤텍스와 달리 임계 지점(critical section)을 직접 지정해야 한다.
```rust
use std::sync::{Arc, Mutex};
use tokio::sync::Semaphore;

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
* Arc(Atomic Reference Count): 원자적 참조 카운트는 동시적 상황에서 사용 가능한 참조 카운팅 스마트 카운터이다. Arc는 참조 카운트를 원자적으로 수행함으로써 여러 스레드에서 동시에 접근하더라도 안전하게 참조 횟수를 관리할 수 있다. 참조 횟수 계산이 원자적이지 않다면 동시에 여러 스레드가 접근할 경우 자원 경쟁 문제가 발생해 예상하지 못한 문제가 발생할 수 있다.
```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
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

    println!("결과: {}", *counter.lock().unwrap()); // 최종 카운터 값을 출력한다.
}
``` 

##### 다중 스레드에서 발생하는 문제
교착상태: 두개 이상의 스레드가 각각 자원을 해제하기를 기다리느라 시스템이 정지한 상황.
```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let lock_a = Arc::new(Mutex::new(0));
    let lock_b = Arc::new(Mutex::new(0));

    let lock_a_ref = lock_a.clone();
    let lock_b_ref = lock_a.clone();

    let thread1 = thread::spawn(move || {
        // 강제로 교착상태를 만든다.
        let a = lock_a.lock().unwrap();
        let b = lock_b_ref.lock().unwrap(); // lock_b는 thread2에 의해 잠겨있는 상태
    });

    let thread2 = thread::spawn(move || {
        let a = lock_b.lock().unwrap();
        let b = lock_a_ref.lock().unwrap(); // lock_a는 thread1에 의해 잠겨있는 상태
    });

    thread1.join().unwrap();
    thread2.join().unwrap();

    println!("프로그램 종료");
}

```


