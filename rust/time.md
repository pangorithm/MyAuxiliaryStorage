### SystemTime
SystemTime은 std::time 모듈에 포함된 구조체로 현재 시스템의 시간을 표현하느데 사용된다. 운영체제 내부 시계를 기반으로 동작하며, 리눅스의 경우 유닉스 에포크를 기준으로 경과 시간을 계산한다.
 주의할 점은 SystemTime은 UTC를 기준으로 계산되기 때문에 사용자의 시간대에 맞는 정보를 얻으려면 따로 시간 연산을 해야 한다.
 
##### 현재 시간 
```rust
 use std::time::SystemTime;

fn main() {
    // 현재 시스템 시간을 가져온다.
    let now = SystemTime::now();

    // 현재 시스템 시간을 디버그 형식으로 출력한다.
    println!("{:?}", now);
    // SystemTime { tv_sec: 1730547487, tv_nsec: 510123098 }
}
```

##### 구간 시간 측정
```rust
use std::thread::sleep;
use std::time::Duration;
use std::time::SystemTime;

fn main() {
    let tm = SystemTime::now();

    println!("1초 대기!");
    sleep(Duration::from_secs(1));

    match tm.elapsed() {
        // 시간 측정
        Ok(elapsed) => {
            println!(
                "대기 시간: {}.{}초",
                elapsed.as_secs(),
                elapsed.subsec_millis()
            );
        }
        Err(e) => {
            println!("오류 발생: {:?}", e);
        }
    }
}
// 1초 대기!
// 대기 시간: 1.0초
```

##### 시간 연산
```rust
use std::time::Duration;
use std::time::SystemTime;

fn main() {
    let now = SystemTime::now();
    let after = now + Duration::from_secs(3);

    println!("현재시간: {:?}", now);
    println!("+3초: {:?}", after);
    // 현재시간: SystemTime { tv_sec: 1730547893, tv_nsec: 96390848 }
    // +3초: SystemTime { tv_sec: 1730547896, tv_nsec: 96390848 }
}
```

##### 날짜 포매팅
```rust
use chrono::Local;

fn main() {
    // 현재 로컬 날짜와 시간을 가져온다.
    let now = Local::now();

    println!("{}", now.format("%Y-%m-%d")); // YYYY-MM-DD

    println!("{}", now.format("%H:%M:%S")); // HH:MM:SS

    println!(
        "{}",
        now.format("오늘은 %A, %B, %d, %y. 현재 시간은%Y-%m-%d")
    );
}
// 2024-11-02
// 20:49:43
// 오늘은 Saturday, November, 02, 24. 현재 시간은2024-11-02
```

#####  시간대 계산
```rust
use chrono::offset::TimeZone;
use chrono::{FixedOffset, Local, Utc};

fn main() {
    // UTC 시간 획득
    let now_utc = Utc::now();
    println!("Utc 시각: {}", now_utc);

    // 로컬 시간
    let now_local = Local::now();
    println!("Local 시각: {}", now_local);

    // 서울 시간 획득 UTC+9
    let seoul_offset = FixedOffset::east(9 * 3600); // +9
    let seoul = seoul_offset.from_utc_datetime(&now_utc.naive_utc());
    println!("한국 시각: {}", seoul);
}
// Utc 시각: 2024-11-02 11:56:14.679871697 UTC
// Local 시각: 2024-11-02 20:56:14.679901283 +09:00
// 한국 시각: 2024-11-02 20:56:14.679871697 +09:00
```