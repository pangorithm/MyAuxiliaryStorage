### Rust/Cargo 자체 업데이트
```bash
rustup update
rustup update stable # 안정화 버전으로 업데이트
```

### cargo 명령어 목록 출력
```bash
cargo --list
```
### Rustup 관리 컴포넌트 확인
```bash
rustup component list --installed
```
### Cargo 애드온
| 이름                | 기능                 | 설치 명령어                                      |
| ----------------- | ------------------ | ------------------------------------------- |
| `cargo-edit`      | 의존성 추가/삭제/업데이트     | `cargo install cargo-edit --version 0.13.3` |
| `cargo-clippy`    | 린터, 코드 분석 도구       | `rustup component add clippy`               |
| `cargo-fmt`       | 자동 코드 포맷팅          | `rustup component add rustfmt`              |
| `cargo-watch`     | 파일 변경 감지 후 자동 빌드 등 | `cargo install cargo-watch`                 |
| `cargo-audit`     | 보안 취약점 검사          | `cargo install cargo-audit`                 |
| `cargo-tree`      | 의존성 트리 출력          | `cargo install cargo-tree`                  |
| `cargo-expand`    | 매크로 확장 결과 보기       | `cargo install cargo-expand`                |
| `cargo-criterion` | 정밀한 벤치마크           | `cargo install cargo-criterion`             |


### cargo-edit 관련 명령어 확인
```
cargo add --help
cargo rm --help
cargo upgrade --help
```