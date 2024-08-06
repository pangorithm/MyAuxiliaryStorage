# 오류 처리
러스트는 다른 언어와는 달리 복구 가능한 오류와 복구 불가능한 오류로 나누어 오류를 처리한다.

### 복구 가능한 오류
Result 열거형을 사용해 복구 가능한 오류를 제어한다.
```rust
enum Result<T, E> {
  Ok(T),
  Err(E)
}
```
Ok는 함수의 실행이 성공할 경우 반환되고 T는 함수 호출이 성공한 경우 반환될 값의 타입을 의미한다. 반대로 Err는 함수의 실행이 실패할 경우 반환되며, E는 오류의 타입을 의미한다.
```rust
use std::fs::File;

fn main() {
    // "test.txt" 파일을 열려고 시도
    let result = File::open("test.txt");

    // result는 Result 타입이므로, 이를 통해 파일 열기의 성공 또는 실패를 확인할 수 있습니다.
    let f = match result {
        Ok(f) => f, // 파일 열기에 성공하면 File 인스턴스를 반환합니다.
        Err(err) => {
            // 파일 열기에 실패하면 에러 정보를 출력하고 프로그램을 종료합니다.
            panic!("파일 열기 실패: {:?}", err) // 파일 열기 실패: Os { code: 2, kind: NotFound, message: "No such file or directory" }
        }
    };

    // 여기에 도달하면 파일 열기에 성공했음을 의미한다.
    println!("파일 열기 성공!");
}
```
##### unwrap
unwrap은 Result가 Ok(T)일 경우 T의 값을 반환한다.
위의 코드에서 unwrap을 사용하면 아래와 같이 축약 할 수 있다.
```rust
use std::fs::File;

fn main() {
    // unwrap 메서드는 Result가 Ok 값이면 그 값을 반환하고, Err 값이면 패닉을 일으킨다.
    let f = File::open("test.txt").unwrap();
    // called `Result::unwrap()` on an `Err` value: Os { code: 2, kind: NotFound, message: "No such file or directory" }

    // 여기에 도달하면 파일 열기에 성공했음을 의미한다.
    println!("파일 열기 성공!");
}
```

##### expect
expect는 Result가 Err(E)일 경우 지정된 오류 메시지를 출력한다. 그리고 unwrap과 마찬가지로 Ok(T)일 경우 T의 값을 반환한다.
```rust
use std::fs::File;

fn main() {
    // unwrap 메서드는 Result가 Ok 값이면 그 값을 반환하고, Err 값이면 사용자 정의 에러 메시지와 함께 패닉을 일으킨다.
    let f = File::open("test.txt").expect("에러");
    // 에러: Os { code: 2, kind: NotFound, message: "No such file or directory" }

    // 여기에 도달하면 파일 열기에 성공했음을 의미한다.
    println!("파일 열기 성공!");
}
```

##### ? 키워드와 함께 사용하는 오류 전파
러스트는 `Result<T, E>`를 반환하는 방식으로 복구 가능한 오류를 처리한다. 그래서 `Result<T, E>` 형을 반환하도록 함수를 구현하면 오류를 전파할 수 있다. 함수 내에 여러가지 오류가 복합적으로 발생한다 하더라도 동일한 형식으로 처리한다.

* ? 키워드 사용 전
```rust
// I/O 작업과 관련된 트레잇과 타입을 사용하기 위해 std::io를 가져온다.
// 파일 작업을 위해 std::fs::File을 가져온다.
use std::fs::File;
use std::io;
use std::io::Read;

// 파일에서 문자열을 읽어 반환하는 함수
// 파일 열기 또는 읽기 중 오류가 발생하면 오류를 반환한다.
fn read_from_file(path: String) -> Result<String, io::Error> {
    let mut s = String::new();
    let mut f = match File::open(path) {
        Ok(f) => f,
        Err(e) => return Err(e), // 파일 열기 실패 시 오류 반환
    };

    match f.read_to_string(&mut s) {
        Ok(_len) => return Ok(s), // 파일 읽기 성공 시 내용 반환
        Err(e) => return Err(e),  // 파일 읽기 실패 시 오류 반환
    };
}

fn main() {
    // 실패 시 사용자 정의 메시지와 함께 프로그램 종료
    let ret = read_from_file(String::from("test.txt")).expect("파일 읽기 중 오류가 발생했습니다.");
    // 파일 읽기 중 오류가 발생했습니다.: Os { code: 2, kind: NotFound, message: "No such file or directory" }

    // 여기에 도달하면 파일 열기에 성공했음을 의미한다.
    println!("test.txt: {}", ret);
}
```

* ? 키워드 사용 후
```
use std::fs::File;
use std::io;
use std::io::Read;

// 파일 열기 또는 읽기 중 오류가 발생하면 오류를 반환한다.
fn read_from_file(path: String) -> Result<String, io::Error> {
    let mut s = String::new(); // 읽은 문자열을 저장할 문자열 객체
    let mut f = File::open(path)?; // 파일 열기. 실패하면 즉시 오류 반환
    let mut _ret = f.read_to_string(&mut s)?; // 파일 읽기. 실패하면 즉시 오류 반환
    Ok(s) // 파일 읽기 성공 시 문자열 반환
}

fn main() {
    // 실패 시 사용자 정의 메시지와 함께 프로그램 종료
    let ret = read_from_file(String::from("test.txt")).expect("파일이 없습니다");
    // 파일이 없습니다: Os { code: 2, kind: NotFound, message: "No such file or directory" }

    // 여기에 도달하면 파일 열기에 성공했음을 의미한다.
    println!("test.txt: {}", ret);
}
```

### 복구 불가능한 오류
`panic!` 매크로를 사용하여 임의로 복구 불가능한 오류를 발생 시킬 수 있다.  
복구 불가능한 오류는 프로그램이 더 이상 정상적으로 실행할 수 없는 상황을 의미한다. 이는 프로그램의 설계 단계에서 예측하지 못한 시스템 오류를 의미한다.
##### 복구 불가능한 오류의 예
* 잘못된 메모리 접근: 허가되지 않거나 비정상적인 데이터가 기록된 메모리에 접근 시 발생
* 잘못된 배열 인덱스 참조: 배열 크기를 벗어난 인덱스를 참조 시 발생
* 잘못된 파일/소켓/파이프 접근: 대상 자원이 존재하지 않거나 예기치 못한 상황이 발생해 더 이상 자원을 사용할 수 없는 상황이 발생했음에도 불구하고 프로그램이 해당 자원에 접근 시 발생
* 시스템 메모리 부족: 시스템의 물리적 메모리가 부족해 더 이상 메모리를 할당받지 못할 시 발생
* 0으로 나누기: 0으로 나눌 시 발생
```rust
fn div(a: i32, b: i32) -> i32 {
    a / b
}

fn main() {
    let ret = div(1, 0); // 1 나누기 0을 시도해 오류 발생
                         // thread 'main' panicked at src/main.rs:2:5:
                         // attempt to divide by zero
                         // note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
    println!("{}", ret);
}
```

##### 백트레이스 얻기
`cargo run` 앞에 `RUST_BACKTRACE=1`을 넣거나 환경 변수에 `RUST_BACKTRACE`를 설정하면 백트레이스를 얻어 프로그램에 오류가 발생할 경우 오류가 발생한 원인을 추적할 수 있도록 호출한 함수를 따라가며 오류가 발생한 위치를 찾을 수 있다.  
cargo build를 사용하면 디버그 모드로 빌드되고 심벌이 패키지에 포함된다. 하지만 디버그 심벌로 인해 프로그래므이 사이즈가 커지며 불필요한 정보들이 패키지에 포함되므로 릴리스 모드에는 기본적으로 디버그 심벌이 탑재되지 않는다.

* `RUST_BACKTRACE=1 cargo run`
```stderr
thread 'main' panicked at src/main.rs:2:5:
attempt to divide by zero
stack backtrace:
   0: rust_begin_unwind
             at /rustc/129f3b9964af4d4a709d1383930ade12dfe7c081/library/std/src/panicking.rs:652:5
   1: core::panicking::panic_fmt
             at /rustc/129f3b9964af4d4a709d1383930ade12dfe7c081/library/core/src/panicking.rs:72:14
   2: core::panicking::panic_const::panic_const_div_by_zero
             at /rustc/129f3b9964af4d4a709d1383930ade12dfe7c081/library/core/src/panicking.rs:180:21
   3: exception::div
             at ./src/main.rs:2:5
   4: exception::main
             at ./src/main.rs:6:15
   5: core::ops::function::FnOnce::call_once
             at /rustc/129f3b9964af4d4a709d1383930ade12dfe7c081/library/core/src/ops/function.rs:250:5
note: Some details are omitted, run with `RUST_BACKTRACE=full` for a verbose backtrace.
```

* `RUST_BACKTRACE=1 cargo run --release`
```stderr
thread 'main' panicked at src/main.rs:2:5:
attempt to divide by zero
stack backtrace:
   0: rust_begin_unwind
   1: core::panicking::panic_fmt
   2: core::panicking::panic_const::panic_const_div_by_zero
   3: exception::main
note: Some details are omitted, run with `RUST_BACKTRACE=full` for a verbose backtrace.
```

* `RUST_BACKTRACE=full cargo run --release
```stderr
thread 'main' panicked at src/main.rs:2:5:
attempt to divide by zero
stack backtrace:
   0:     0x5606947ce715 - <std::sys_common::backtrace::_print::DisplayBacktrace as core::fmt::Display>::fmt::h1e1a1972118942ad
   1:     0x5606947eb7ab - core::fmt::write::hc090a2ffd6b28c4a
   2:     0x5606947cc99f - std::io::Write::write_fmt::h8898bac6ff039a23
   3:     0x5606947ce4ee - std::sys_common::backtrace::print::ha96650907276675e
   4:     0x5606947cf7a9 - std::panicking::default_hook::{{closure}}::h215c2a0a8346e0e0
   5:     0x5606947cf4ed - std::panicking::default_hook::h207342be97478370
   6:     0x5606947cfc43 - std::panicking::rust_panic_with_hook::hac8bdceee1e4fe2c
   7:     0x5606947cfaeb - std::panicking::begin_panic_handler::{{closure}}::h00d785e82757ce3c
   8:     0x5606947cebd9 - std::sys_common::backtrace::__rust_end_short_backtrace::h1628d957bcd06996
   9:     0x5606947cf857 - rust_begin_unwind
  10:     0x5606947b4b63 - core::panicking::panic_fmt::hdc63834ffaaefae5
  11:     0x5606947b50e7 - core::panicking::panic_const::panic_const_div_by_zero::h0158edae44a9fd47
  12:     0x5606947b526e - exception::main::hc7aaa9322410a514
  13:     0x5606947b5223 - std::sys_common::backtrace::__rust_begin_short_backtrace::hc0b3e44c4ea8e345
  14:     0x5606947b5239 - std::rt::lang_start::{{closure}}::h201322a765f062bc
  15:     0x5606947cabb0 - std::rt::lang_start_internal::h3ed4fe7b2f419135
  16:     0x5606947b5295 - main
  17:     0x7faad02ca590 - __libc_start_call_main
  18:     0x7faad02ca640 - __libc_start_main_alias_1
  19:     0x5606947b5155 - _start
  20:                0x0 - <unknown>
``` 

릴리즈 모드에서도 디버그 심벌을 탑재하고 싶다면 Cargo.tmol에 별도의 프로파일을 등록해 사용해야 한다.  
```
[profile.release-with-debug]
inherits = "release"
debug = true
```

* `RUST_BACKTRACE=full cargo run --profile=release-with-debug`
```
thread 'main' panicked at src/main.rs:2:5:
attempt to divide by zero
stack backtrace:
   0:     0x55f6bce58715 - std::backtrace_rs::backtrace::libunwind::trace::h1a07e5dba0da0cd2
                               at /rustc/129f3b9964af4d4a709d1383930ade12dfe7c081/library/std/src/../../backtrace/src/backtrace/libunwind.rs:105:5
   1:     0x55f6bce58715 - std::backtrace_rs::backtrace::trace_unsynchronized::h61b9b8394328c0bc
                               at /rustc/129f3b9964af4d4a709d1383930ade12dfe7c081/library/std/src/../../backtrace/src/backtrace/mod.rs:66:5
   2:     0x55f6bce58715 - std::sys_common::backtrace::_print_fmt::h1c5e18b460934cff
                               at /rustc/129f3b9964af4d4a709d1383930ade12dfe7c081/library/std/src/sys_common/backtrace.rs:68:5

...중략...
                               at /rustc/129f3b9964af4d4a709d1383930ade12dfe7c081/library/std/src/panic.rs:149:14
  30:     0x55f6bce54bb0 - std::rt::lang_start_internal::h3ed4fe7b2f419135
                               at /rustc/129f3b9964af4d4a709d1383930ade12dfe7c081/library/std/src/rt.rs:141:20
  31:     0x55f6bce3f29c - main
  32:     0x7f2fc47f6590 - __libc_start_call_main
  33:     0x7f2fc47f6640 - __libc_start_main_alias_1
  34:     0x55f6bce3f155 - _start
  35:                0x0 - <unknown>
```

### 복구 가능한 오류 vc 복구 불가능한 오류
복구 가능한 오류의 경우 `Result<T, E>`를 반환하기에 호출자가 복구 가능한 오류를 발생할 수 있다는 것을 고려해 개발한다.  
복구 불가능한 오류를 발생 시키는 경우 함수 명세나 가이드 페이지에 `panic!`이 발생할 수 있음을 명시할 것을 공식 가이드 북에서 권장한다. 그 이유는 panic!이 발생한다는 것을 공지하지 않으면 함수의 호출자가 영문도 모른 채 panic!을 맞닥뜨릴 수 있기 때문이다.  