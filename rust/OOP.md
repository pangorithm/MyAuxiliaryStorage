# Object Oriented Programming Rust
러스트는 객체지향 언어가 아니기에 완벽하게 캡슐화, 다형성, 상속 등을 지원하지는 못하지만, 객체지향의 핵심적인 개념을 트레잇과 같은 형태로 차용해 객체지향에 가까운 프로그램을 작성할 수 있다.

### 캡슐화
러스트는 pub 키워드를 사용해 함수와 데이터를 캡슐화 할 수 있다. pub 키워드를 사용한 부분은 외부로 노출되고 그렇지 않은 부분은 내부로 은닉된다.
```rust
struct Student {
    id: i32,           // private field
    pub name: String,  // public field
    pub email: String, // public field
}

impl Student {
    // public 생성자
    pub fn new(id: i32, name: String, email: String) -> Student {
        Student { id, name, email }
    }

    // public 메서드
    pub fn get_name(&self) -> &String {
        &self.name
    }

    // private 메서드
    fn set_name(&mut self, name: String) {
        self.name = name.clone();
    }
}

fn main() {
    let student = Student::new(1, String::from("luna"), String::from("luna@email.me"));
    println!("이름: {}", student.get_name());
    // 이름: luna
}
```

### 다형성
다형성은 객체가 문맥에 따라 다른 자료형으로 형태를 취할 수 있게 하는 것을 말한다. 러스트는 trait 키워드와 dyn 키워드를 사용해 다형성을 제공한다. 트레잇은 객체지향의 인터페이스와 유사하다. 트레잇은 함수를 추상화해 트레잇을 구현하는 구조체들이 같은 이름의 API를 제공하도록 한다. dyn 키워드는 컴파일러에 트레잇의 구현체를 런타임에 찾아야 한다고 알려주는 키워드이다.
```rust
// Hello라는 트레잇을 정의한다. 이 트레잇은 hello_msg 메서드를 가져야 한다.
trait Hello {
    fn hello_msg(&self) -> String; // hello_msg 메서드는 String을 반환해야 하며, 이를 구현하는 타입에서 정의해야 한다.
}

// say_hello 함수는 Hello 트레잇을 구현하는 어떠한 타입의 참조도 받을 수 있다.
// 이 함수는 전달받은 타입의 hello_msg 메서드를 호출하고 그 결과를 출력한다.
fn say_hello(say: &dyn Hello) {
    println!("{}", say.hello_msg());
}

struct Student {}

impl Hello for Student {
    fn hello_msg(&self) -> String {
        String::from("안녕하세요! 선생님,")
    }
}

struct Teacher {}

impl Hello for Teacher {
    fn hello_msg(&self) -> String {
        String::from("안녕하세요! 오늘 수업은...")
    }
}

fn main() {
    let student = Student {};
    let teacher = Teacher {};

    say_hello(&student);
    // 안녕하세요! 선생님,
    say_hello(&teacher);
    // 안녕하세요! 오늘 수업은...
}
```

### 상속
상속은 기존에 정의된 자료형을 기반으로 새로운 자료형을 만드는 것을 말한다. 상속을 활용하면 기존 코드를 재사용하면서 수정이 필요한 부분만 재정의해 사용할 수 있기에 유지 보수성을 크게 높일 수 있다. 다만 상속이 너무 지나치면 상위 클래스와 하위 클래스 가느이 결합도가 필요 이상으로 높아지는 문제가 있다.
불행히도 러스트에서는 상속을 구현하기가 쉽지 않다. 데코레이터나 프록시 패턴 등을 사용해 상속과 유사하게 작동하도록 구현하는 방법이 현재로서는 최선이다.
```rust
trait Pointable {
    // 인터페이스 정의
    fn x(&self) -> i32;
    fn y(&self) -> i32;
}

struct Point {
    x: i32,
    y: i32,
}

impl Pointable for Point {
    // Point에 Pointable 인터페이스를 구현한다.
    fn x(&self) -> i32 {
        self.x
    }

    fn y(&self) -> i32 {
        self.y
    }
}

fn print_pointable(pointable: &dyn Pointable) {
    println!("x: {}, y: {}", pointable.x(), pointable.y());
}

// ColorPoint는 Point를 확장한 하위 클래스처럼 작동한다.
// 러스트는 상속을 제공하지 않기 때문에 내부적으로 point 인스턴스를 가지고 있다.
// 그리고 부모 클래스의 함수를 호출하는 것처럼 만들기 위해 point 인스턴스의 함수를 호출한다.
struct ColorPoint {
    color: String,
    point: Point,
}

impl ColorPoint {
    fn new(color: String, x: i32, y: i32) -> ColorPoint {
        ColorPoint {
            color: color,
            point: Point { x: x, y: y },
        }
    }

    fn color(&self) -> &String {
        &self.color
    }
}

impl Pointable for ColorPoint {
    fn x(&self) -> i32 {
        self.point.x
    }

    fn y(&self) -> i32 {
        self.point.y
    }
}

fn main() {
    let pt = ColorPoint::new(String::from("red"), 1, 2);
    print_pointable(&pt);
    // x: 1, y: 2
}
```