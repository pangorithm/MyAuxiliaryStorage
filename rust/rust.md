##### 러스트의 오류
* 러스트 오류의 분류: 
	1. 복구 가능한 오류
	2. 복구 불가능한 오류
* 러스트 오류 처리 방식: 
	1. `Reault<T, E>`를 반환한다
	2. `panic!`을 발생시켜 프로그램을 명시적으로 종료한다.

##### 러스트의 비동기 처리
* sync/await 구문 내장 지원
* 비동기 함수를 실행할 실행자가 Future 객체의 형태로 존재하며, 객체 생성 순간에는 아무것도 실행하지 않기에 실행자를 명시적으로 지정해야 한다. 또한 Future는 스레드를 넘나들 수 있어 작동 중간에 잠시 멈추고 다른 스레드에서 재개하는 것도 가능하다.


##### cargo 명령어 도움말
```bash
> cargo
Rust's package manager

Usage: cargo [+toolchain] [OPTIONS] [COMMAND]
       cargo [+toolchain] [OPTIONS] -Zscript <MANIFEST_RS> [ARGS]...

Options:
  -V, --version                  Print version info and exit
      --list                     List installed commands
      --explain <CODE>           Provide a detailed explanation of a rustc error message
  -v, --verbose...               Use verbose output (-vv very verbose/build.rs output)
  -q, --quiet                    Do not print cargo log messages
      --color <WHEN>             Coloring: auto, always, never
  -C <DIRECTORY>                 Change to DIRECTORY before doing anything (nightly-only)
      --locked                   Assert that `Cargo.lock` will remain unchanged
      --offline                  Run without accessing the network
      --frozen                   Equivalent to specifying both --locked and --offline
      --config <KEY=VALUE|PATH>  Override a configuration value
  -Z <FLAG>                      Unstable (nightly-only) flags to Cargo, see 'cargo -Z help' for details
  -h, --help                     Print help

Commands:
    build, b    Compile the current package
    check, c    Analyze the current package and report errors, but don't build object files
    clean       Remove the target directory
    doc, d      Build this package's and its dependencies' documentation
    new         Create a new cargo package
    init        Create a new cargo package in an existing directory
    add         Add dependencies to a manifest file
    remove      Remove dependencies from a manifest file
    run, r      Run a binary or example of the local package
    test, t     Run the tests
    bench       Run the benchmarks
    update      Update dependencies listed in Cargo.lock
    search      Search registry for crates
    publish     Package and upload this package to the registry
    install     Install a Rust binary
    uninstall   Uninstall a Rust binary
    ...         See all commands with --list

See 'cargo help <command>' for more information on a specific command.
```