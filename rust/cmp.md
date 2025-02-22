[rust cmp 모듈 공식문서](https://doc.rust-lang.org/std/cmp/index.html)

기본적으로 최소값을 우선하는 Reverse 래퍼를 지원한다.

비교 방식을 커스텀 하기 위해서는 `Ord와` `PartialOrd` 트레이트를 직접 구현해야 한다.

```rust
use std::cmp::Ordering;

#[derive(Eq, PartialEq)] // 해당 트레이트의 필요한 메서들을 rust 컴파일러가 자동 구현
struct MinHeapNode(i32);

impl Ord for MinNode {
    fn cmp(&self, other: &Self) -> Ordering {
        other.0.cmp(&self.0) // 기본 정렬 순서를 반대로 만듦
					         // MinNode는 래핑 객체이므로 .0으로 원본을 꺼내야함
    }
}

impl PartialOrd for MinNode {
    fn partial_cmp(&self, other: &Self) -> Option<Ordering> {
        Some(self.cmp(other))
    }
}
```

### `Ord` vs `PartialOrd` 차이점
* Ord
	* 전체 순서 (Total Ordering)
	* partial_cmp(): `Ordering` 항상 반환 (`None` 불가능)
	* cmp(): `Ordering` 반환
	* `f64`, `f32` 사용: 불가능 (`NaN` 때문에)
	* 항상 모든 값이 비교 가능 해야한다
* PartialOrd
	* 부분 순서 (Partial Ordering)
	* partial_cmp(): `Option<Ordering>` 반환 가능 (`None` 가능)
	* cmp(): 없음
	* `f64`, `f32` 사용: 가능 (`NaN` 처리)
	* 비교할 수 없는 값이 있을 수 있다.