# 분류
* Sequence
  * Vec : 크기 조정이 가능한 배열
  * VecDeque : 앞뒤 양쪽에 자료를 추가, 삭제할 수 있는 큐
  * LinkedList : 자료가 한 줄로 연결되게 관리하는 자료구조, 삽입, 삭제가 빈번하게 발생할 때 사용
* Map
  * HashMap : key와 value의 쌍으로 이뤄진 자료를 관리하는 자료구조
  * BTreeMap : B-Tree를 기반으로 하는 정렬된 Map
* Set
  * HashSet : 하나의 자료만 담을 수 있는 집합 구조
  * BTreeSet : B-Tree를 기반으로 하는 정렬된 Set
* 기타
  * BinaryHeap : 우선 순위 큐
  * String : 문자열

### Vec
미리 capacity만큼 공간을 할당하고 그 이상 자료가 추가되면 크기를 재조정한다. 인접 메모리를 사용하기에 랜덤 액세스 성능이 빠르나, 자료를 검색할 때 모든 자료를 탐색한다.
##### vec! 매크로
```rust
fn main() {
    // let mut v: Vec<i32> = Vec::new(); // 빈 i32 타입의 벡터 v를 생성한다.
    // let mut v: Vec<i32> = vec![1, 2, 3]; // 초기값을 가지는 벡터
    let mut v: Vec<i32> = vec![]; // 빈 벡터

    for i in 1..10 {
        // 1부터 9 까지의 숫자를 반복한다.
        v.push(i);
    }

    for d in &v {
        // 벡터 v의 각 요소에 대해 반복한다.
        print!("{}, ", d);
    }
    // 1, 2, 3, 4, 5, 6, 7, 8, 9,
}
```

##### 벡터 개별 요소 읽기
```rust
fn main() {
    let v = vec![1, 2, 3]; // i32 타입의 벡터 v를 생성하고 초깃값으로 1, 2, 3을 설정한다.
    let one = v[0]; // 벡터의 첫 번째 요소를 one에 할당한다.
    let two = v.get(1); // 벡터의 두 번째 요소를 가져온다. Option 타입을 반환하므로 결과는 Some(2)이다.
    let nine = v.get(9); // 벡터 크기를 벗어나는 인덱스

    println!("One: {:?}, Two: {:?}, Nine: {:?}", one, two, nine);
    // One: 1, Two: Some(2), Nine: None

    println!("{:?}", v[9]); // panic! 발생
                            // thread 'main' panicked at src/main.rs:9:29:
                            // index out of bounds: the len is 3 but the index is 9
}
```

##### 벡터의 값 변경
```rust
fn main() {
    let mut v = vec![1, 2, 3]; // i32 타입의 벡터 v를 생성하고 초깃값으로 1, 2, 3을 설정한다.
    v[0] = 2;
    v[1] = 3;
    v[2] = 4;

    println!("{}, {}, {}", v[0], v[1], v[2]);
    // 2, 3, 4

    for i in &mut v {
        *i += 1;
    }
    println!("{}, {}, {}", v[0], v[1], v[2]);
    // 3, 4, 5
}
```

### LinkedList
리스트는 벡터와 다르게 인덱스를 사용해 값을 찾는 함수가 제공되지 않는다. 따라서 원하는 인덱스의 값을 얻으려면 전체 목록을 순회하거나 iterator의 nth( ) 함수로 데이터를 얻어야 한다.

##### LinkedList 생성 및 조회
```rust
use std::collections::LinkedList;

fn main() {
    let mut list: LinkedList<i32> = LinkedList::new(); // i32 타입의 빈 연결 리스트 생성
    list.push_back(1); // 리스트의 뒤에 숫자 1을 추가
    list.push_back(2); // 리스트의 뒤에 숫자 2을 추가
    list.push_back(3); // 리스트의 뒤에 숫자 3을 추가

    for i in &list {
        print!("{}, ", i);
    }
    // 1, 2, 3,

    list = LinkedList::new();
    for i in 0..10 {
        list.push_back(i);
    }

    // 9번째 인덱스 찾기
    let idx = 9;
    let mut i = 0;
    let mut target: i32 = 0;

    for data in &list {
        if i == idx {
            target = *data;
        }

        i += 1;
    }
    println!("target: {:?}", target);
    // target: 9

    println!("iterator target: {:?}", list.iter().nth(9));
    // iterator target: Some(9)
}
```

##### LinkedList 내부의 값 변경
```rust
use std::collections::LinkedList;

fn main() {
    let mut list: LinkedList<i32> = LinkedList::new(); // i32 타입의 빈 연결 리스트 생성
    for i in 0..10 {
        list.push_back(i);
    }

    for d in list.iter_mut() {
        // 수정 가능한 반복자를 얻는다.
        *d += 10;
    }

    for d in list.iter() {
        print! {"{:?}, ", d};
    }
    // 10, 11, 12, 13, 14, 15, 16, 17, 18, 19,
}
```

### HashMap
키를 사용해 값을 쉽게 찾을 수 있기에 보통 cache를 구현할 때 많이 사용한다.
```rust
use std::collections::HashMap;

fn main() {
    let mut books: HashMap<i32, String> = HashMap::new(); // i32 타입의 key와 String 타입의 값을 가지는 빈 HashMap 생성

    books.insert(10, String::from("Rust"));
    books.insert(20, String::from("Java"));
    books.insert(30, String::from("Python"));

    for (key, value) in &books {
        println!("Key: {:?}, Value: {:?}", key, value);
    }
    // 조회 순서는 보장되지 않는다.
    // # cargo run
    // Key: 10, Value: "Rust"
    // Key: 30, Value: "Python"
    // Key: 20, Value: "Java"
    // # cargo run
    // Key: 30, Value: "Python"
    // Key: 10, Value: "Rust"
    // Key: 20, Value: "Java"
    // # cargo run
    // Key: 20, Value: "Java"
    // Key: 30, Value: "Python"
    // Key: 10, Value: "Rust"

    let rust = books.get(&10);
    println!("key 10은 {:?}", rust);
    // key 10은 Some("Rust")
}
```

### HashSet
```rust
use std::collections::HashSet;

fn main() {
    let mut book: HashSet<String> = HashSet::new(); // String 타입의 값을 가지는 빈 HashSet 생성

    book.insert(String::from("Rust"));
    book.insert(String::from("Java"));
    book.insert(String::from("Python"));
    for data in &book {
        println!("data: {:?}", data);
    }
    // 조회 순서는 보장되지 않는다
    // # cargo run
    // data: "Java"
    // data: "Python"
    // data: "Rust"
    // # cargo run
    // data: "Rust"
    // data: "Java"
    // data: "Python"
    // # cargo run
    // data: "Python"
    // data: "Rust"
    // data: "Java"

    // 값이 있는지 확인하기
    if book.contains("JavaScript") == false {
        println!("JavaScript가 없습니다.");
        // JavaScript가 없습니다.
    }
}
```

### BinaryHeap
이진힙은 우선순위 큐라고도 부른다. 이진힙은 주어진 자료에서 가장 큰 값 혹은 가장 작은 값을 찾을 때 사용된다.
```rust
use std::collections::BinaryHeap;

fn main() {
    let mut heap: BinaryHeap<i32> = BinaryHeap::new(); // String 타입의 값을 가지는 빈 HashSet 생성

    heap.push(3);
    heap.push(9);
    heap.push(2);
    heap.push(5);

    while heap.is_empty() == false {
        print!("{:?}, ", heap.pop()); // 힙의 최대값을 꺼내어 출력한다. pop()은 Option<T>를 반환하므로 {:?}를 사용해 출력한다.
    }
    // Some(9), Some(5), Some(3), Some(2),
}
```

### String
##### 문자열 만들기
```rust
fn main() {
    let mut eng = String::new();
    eng.push_str("hello");
    let chn = "你好".to_string();
    let kor = String::from("안녕하세요");

    println!("{}, {}, {}", eng, chn, kor);
}
// hello, 你好, 안녕하세요
```
##### 문자열 포매팅
```rust
fn main() {
    let str = String::from("안녕");
    let idx = 123;
    let s = format!("{} {}", str, idx); // str과 idx를 결함한다.
    println!("{}", s);
    // 안녕 123
}
```

##### 문자열 내 문자 탐색
```rust
fn main() {
    let txt = String::from("안녕하세요");
    for c in txt.chars() {
        print!("{} ", c);
    }
    // 안 녕 하 세 요
}
```

##### 인코딩 변경하기
러스트는 기본적으로 UST-8 인코딩을 사용한다. 이는 유니코드 문자를 표현하기 위한 가변 길이 문자 인코딩 방식으로, 전세계 대부분의 문자를 1부터 4바이트의 범위로 인코딩 할 수 있다.  

* 인코딩을 변경하기 위한 Cargo.toml 수정
  ```rust
  [dependencies]
  encoding_rs = "*"
```
* 인코딩 변환 코드
```rust
extern crate encoding_rs;

use encoding_rs::{EUC_KR, UTF_8};
use std::str;

fn main() {
    let utf8_string = "안녕하세요";

    // UTF-8로 인코딩된 바이트로 변환
    let utf8_bytes = utf8_string.as_bytes();

    // EUC-KR로 인코딩된 바이트로 변환
    let (euc_kr_bytes, _, _) = EUC_KR.encode(utf8_string);

    // 결과 출력
    println!("UTF-8 {:?}", utf8_bytes);
    println!("EUC_KR {:?}", euc_kr_bytes);
    // UTF-8 [236, 149, 136, 235, 133, 149, 237, 149, 152, 236, 132, 184, 236, 154, 148]
    // EUC_KR [190, 200, 179, 231, 199, 207, 188, 188, 191, 228]

    // EUC_KR 바이트 배열을 UTF-8 문자열로 디코딩
    let (utf8_string, _, _) = EUC_KR.decode(&euc_kr_bytes);

    // 결과 출력
    println!("EUC_KR to UTF-8: {:?}", utf8_string);
    // EUC_KR to UTF-8: "안녕하세요"
}
```

### 컬렉션의 소유권
컬렉션에서도 '소유권'이라는 중요한 개념이 있다. iter( )와 into_iter( )는 둘 다 반복자를 생성하는 메서드이지만, 작동 방식이 약간 다르다
iter( )는 불변 빌림을 사용해 반복자를 생성하는 반면, into_iter( )는 컬렉션의 소유권을 반복자로 이동시킨다.
* iter( )
```rust
fn main() {
    let vec = vec![1, 2, 3];
    for item in vec.iter() {
        // vec의 불변 빌림 반복자 생성
        println!("{}", item);
    }

    println!("{:?}", vec); // vec 접근 가능
                           // [1, 2, 3]
}
```
* into_inter( )
```rust
fn main() {
    let vec = vec![1, 2, 3];
    for item in vec.into_iter() {
        // vec의 소유권은 이동했으므로 이후에 vec을 사용할 수 없음
        println!("{}", item);
    }

    // println!("{:?}", vec);
    // 8   |     println!("{:?}", vec);
    //     |                      ^^^ value borrowed here after move
}
```
