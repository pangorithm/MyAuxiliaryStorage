### Copy
Copy 트레잇은 묵시적 복사를 수행하며 이때 바이트 단위의 복제인 깊은 복사를 수행한다. 따라서 소유권이 아닌 값만 복제하여 전달하기 때문에 코드를 간결하게 만든다는 장점이 있지만, 성능 저하의 원인이 될 수 있다.
```rust
#[derive(Debug, Clone, Copy)]
struct Point {
    x: i32,
    y: i32,
}

fn add_points(p1: Point, p2: Point) -> Point {
    Point {
        x: p1.x + p2.x,
        y: p1.y + p2.y,
    }
}

fn main() {
    let a = Point { x: 1, y: 2 };
    let b = Point { x: 3, y: 4 };

    // add_point의 인자로 들어가는 a, b는 copy 트레잇에 의해 복제됩니다.
    // 소유권을 잃지 않습니다.
    let result = add_points(a, b);

    println!("{:?}", a); // a에 접근 가능
    println!("{:?}", b); // b에 접근 가능
    println!("{:?}", result);
}
```

### Clone
Clone 트레잇은 객체를 복사하는 기능을 제공한다. Copy와 다른점은 Clone은 명시적으로 clone() 함수를 호출해야 복사된다. 이로 인해 사용자가 직접 clone()을 구현하여 값을 복제하는 방식을 결정할 수 있다.
```rust
#[derive(Debug)]
struct Person {
    name: String,
    age: u32,
    cloned: bool,
}

// Person을 복제한다. Copy과는 다르게 값을 직접 설정할 수 있다
impl Clone for Person {
    fn clone(&self) -> Self {
        Person {
            name: self.name.clone(),
            age: self.age,
            cloned: true,
        }
    }
}

fn main() {
    let person1 = Person {
        name: String::from("루나"),
        age: 10,
        cloned: false,
    };

    // person1을 복제합니다. 소유권을 잃지 않습니다.
    let person2 = person1.clone();

    println!("{:?}", person1);
    println!("{:?}", person2);
}
```

### Drop
러스트는 메모리 관리를 위해 소유권 시스템을 활용한다. 그러나 파일, 소켓, 비트맵과 같은 특정 자원은 소유권의 라이프사이클과 다를때가 있다. 이러한 리소스나 객체가 메모리에서 해제될 때 특정 작동을 실행하기 위해 Drop 트레잇이 사용된다. 이런 작업은 직접 호출할 수 없기 때문에, std::mem::drop() 함수를 통해 리소스를 명시적으로 해제하거나 정리할 수 있다.
```rust
struct Book {
    title: String,
}

impl Drop for Book {
    fn drop(&mut self) {
        println!("Book 객체 해제: {}", self.title);
    }
}

fn main() {
    {
        let book = Book {
            title: String::from("러스트"),
        };
    } // book이 스코프를 벗어나며 book의 Drop 트레잇이 자동으로 호출된다.
}
```


### From 과 Into
From 트레잇은 한 자료형을 다른 자료형으로 변환하는 로직을 정의할 수 있게 해준다. 반대로 Into 트레잇은 From의 역방향으로 자료형을 변환한다. 이때 Into 트레잇은 별도로 구현하지 않더라도 컴파일러가 자동으로 구현해주어 활용할 수 있다.
```rust
#[derive(Debug)]
struct Point {
    x: i32,
    y: i32,
}

impl From<(i32, i32)> for Point {
    fn from(tuple: (i32, i32)) -> Self {
        Point {
            x: tuple.0,
            y: tuple.1,
        }
    }
}

fn main() {
    let tuple = (1, 2);

    // 주어진 tuple을 바탕으로 Point 객체 생성
    let pt: Point = Point::from(tuple);

    println!("Point::from = {:?}", pt);

    // tuple을 기반으로 point를 생성. 이때 Point::from이 호출된다.
    let pt: Point = tuple.into();

    println!("tuple.into = {:?}", pt);
}
```

### AsRef
AsRef 트레잇은 객체를 참조값으로 변환하는 용도로 사용된다. 이를 통해 함수가 다양한 자료형의 참조를 받아들일 수 있다. `AsRef<T>`를 파라미터로 취하는 함수는 T의 참조뿐만 아니라 T를 대상으로 AsRef를 구현한 다른 타입들의 참조도 인자로 받아들일 수 있다.
```rust
#[derive(Debug)]
struct Person {
    name: String,
    age: u32,
}

impl AsRef<str> for Person {
    // Person의 name을 str 형태로 참조 할 수 있다.
    fn as_ref(&self) -> &str {
        &self.name
    }
}

fn greet_person<P: AsRef<str>>(person: P) {
    println!("안녕! {}", person.as_ref());
}

fn main() {
    let person = Person {
        name: String::from("루나"),
        age: 30,
    };

    // Person 구조체에 AsRef<str>을 구현했기 때문에,
    // greet_person 함수는 Person을 인자로 받아 사용할 수 있다
    greet_person(person);

    // String과 &str도 greet_person 함수 호출이 가능하다.
    greet_person(String::from("러스트"));
    greet_person("하이!");
}
```


### AsMut
AsRef 와 비슷하지만 수정 가능한 참조로 바꾼다는 차이가 있다.
```rust
#[derive(Debug)]
struct Person {
    name: String,
    age: u32,
}

impl AsMut<String> for Person {
    // Person의 name을 str 형태로 참조 할 수 있다.
    fn as_mut(&mut self) -> &mut String {
        &mut self.name
    }
}

fn name_change<P: AsMut<String>>(person: &mut P, new_name: &str) {
    let name = person.as_mut();
    name.clear();
    name.push_str(new_name);
    println!("안녕! {}", person.as_mut());
}

fn main() {
    let mut person = Person {
        name: String::from("루나"),
        age: 10,
    };

    println!("변경 전: {}", person.name);
    name_change(&mut person, "러스트");
    println!("변경 후: {}", person.name);
}

```