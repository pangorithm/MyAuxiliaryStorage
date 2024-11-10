러스트는 운영체제에 독립적으로 설계되어 있어 다양한 플랫폼에서 호환성을 유지한다. 그러나 특정 상황에서는 운영체제의 고유한 기능을 활용해야 할 때가 있다. 새로운 프로세스를 실행하거나 해당 운영체제의 특별한 기능에 접근해야 하는 경우 등이 있다. 이를 위해 러스트는 시스템 콜 기능을 제공한다.

### std::os
std::os::windows, std::os::linux와 같은 서브 모듈이 존재해 각 운영체제별로 특화된 기능과 인터페이스를 제공한다. 이런 구조로 인해 std::os의 사용은 플랫폼에 종속적이다.
* `std::os::unix`: unix와 호환되는 모든 시스템에서 제공하는 기능을 제공
* `std::os::unix::fs`: DirEntry, FileType, Metadata, Permissions, ReadDir 등의 확장 기능을 제공
* `std::os::unix::io`: AsRawFd, FromRawFd, IntoRawFd, RawFd 등의 유형 및 트레잇에 대한 확장을 제공해 unix 파일 디스크립터와 상호작용하는 방법을 제공
* `std::os::unix::net`: SocketAddr, UnixDatagram, UnixListener, UnixStream 등 unix 소켓과 관련된 확장 기능을 제공
* `std::os::unix::thread`: 운영체제 스레드 기능을 제공
* `std::os::windows`: 윈도우 특화 기능을 제공
* `std::os::raw`: C언어 원시 자료형에 접근할 때 사용

### std::env
운영체제의 환경 및 실행 중인 프로세스에 관한 정보에 접근할 때 활용한다. 주로 환경 변수의 조회나 설정, 그리고 프로그램 실행 시 전달된 명령줄 인자를 가져오는데 사용된다. 또한 시스템의 핵심 디렉터리 경로 조회하는 기능을 제공한다.

### std::process
프로세스와 관련된 다양한 작업과 관련된 기능을 제공한다. 예를 들어 프로세스의 생성, 관리, 종료 또는 다른 프로세스의 표준 입출력 스트림에 접근하는 기능을 지원한다.

### std::fs
파일 시스템 관련 작업을 지원하는 표준 라이브러리의 일부이다. 파일을 열고, 읽고, 쓰는 작업 등을 위한 File 구조체와 디렉터리의 항목을 나타낸느 DirEntry 구조체가 포함되어 있다.

### std::path
파일 경로 조작 모듈. 경로 구성, 결합, 확장자 추출 등을 지원한다.