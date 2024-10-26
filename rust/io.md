# 입출력

### 동기식 입출력
```rust
use std::fs::File;
use std::io::{Read, Write};

fn main() {
    let mut file = File::open("input.txt").unwrap();
    let mut contents = String::new();
    file.read_to_string(&mut contents).unwrap();
    // 파일을 읽을 때까지 대기합니다.

    println!("{}", contents);

    let mut file = File::create("output.txt").unwrap();
    file.write_all(contents.as_bytes()).unwrap();
    // 파일을 쓸 때까지 대기합니다.
}
```

### 비동기식 입출력
```rust
use tokio::fs::File;
use tokio::io::{AsyncReadExt, AsyncWriteExt};

#[tokio::main]
async fn main() {
    let mut file = File::open("input.txt").await.unwrap(); // 비동기 방식으로 file handel을 얻습니다.
    let mut contents = String::new();
    file.read_to_string(&mut contents).await.unwrap(); // 비동기 방식으로 file을 읽습니다.

    println!("{}", contents);

    let mut file = File::create("output.txt").await.unwrap(); // 비동기 방식으로 file을 생성합니다.
    file.write_all(contents.as_bytes()).await.unwrap(); // 비동기 방식으로 file에 내용을 저장합니다.
}

```

### 비동기식 이벤트루프
```rust
use tokio::fs::File;
use tokio::io::{stdin, BufReader, AsyncBufReadExt};

#[tokio::main]
async fn main() {
    let mut reader = BufReader::new(stdin());
    let mut lines = reader.lines();

    loop { // quit가 입력될 때까지 입력을 받음
        match lines.next_line().await.unwrap() {
            Some(input) => {
                println!("입력: {}", input);

                if input == "quit" { // quit을 입력받으면 종료합니다.
                    break;
                }
            }
            None => {
                break;
            }
        };

    }
}
```


### 데이터 버퍼링
```rust
use std::fs::File;
use std::io::{BufRead, BufReader};

fn main() {
    let file = File::open("input.txt").unwrap();
    let reader = BufReader::new(file); // BufReader를 생성합니다.

    for line in reader.lines() {
        // file을 읽습니다.
        let line = line.unwrap();
        println!("{}", line);
    }
}
```

### 데이터 직렬화
```rust
use serde::{Deserialize, Serialize};

#[derive(Serialize, Deserialize)]
struct Point {
    x: i32,
    y: i32,
}

fn main() {
    let pt = Point { x: 10, y: 20 };
    let json = serde_json::to_string(&pt).unwrap(); // pt를 json으로 변환
    println!("json: {}", json);

    let pt: Point = serde_json::from_str(&json).unwrap(); // json을 사용해 Point를 생성
    println!("point: [{}, {}]", pt.x, pt.y);
}
```

### SQLite 사용하기
```rust
use sqlite;
use sqlite::State;

fn main() {
    let connection = sqlite::open(":memory:").unwrap(); // 메모리에 sqlite db를 생성

    let query = "
      CREATE TABLE users (name TEXT, age INTEGER);
      INSERT INTO users VALUES ('루나', 3);
      INSERT INTO users VALUES ('러스트', 13);
    ";
    connection.execute(query).unwrap(); // 테이블 생성 쿼리를 실행

    let query = "SELECT * FROM users WHERE age > ?"; // ?는 1에 바인딩 됨
    let mut statement = connection.prepare(query).unwrap(); // 쿼리를 실행
    statement.bind((1, 5)).unwrap(); // age > 5

    while let Ok(State::Row) = statement.next() {
        // 테이블의 데이터를 조회
        println!("name = {}", statement.read::<String, _>("name").unwrap());
        println!("age = {}", statement.read::<i64, _>("age").unwrap());
    }
}
```