
### NATS의 설정파일
JSON과 YAML의 장점을 결합한 conf 파일을 사용한다.
* `#` 또는 `//` 를 사용하여 주석을 달 수 있다.
* `=`, `:`, `(공백)`를 사용하여 값을 할당 할 수 있다.
* 배열은 `[]`으로 선언한다
* Map은 `{}`으로 선언한다
* Map은 구분자 없이 할당할 수 있다.
* 라인의 마침 표시인 `;`는 생략해도 된다.
* 설정 파일은 UTF-8로 파싱된다.
* 유니코드 사용이 가능하나, ASCII 사용이 권장된다.
* 문자열이 숫자로 시작되는 경우 반드시 `" "`으로 감싸야 한다.

### 주요 설정 예시
구성 Property 주요 항목
* 기본 연결: host, port, 또는 listen: host:port
* JetStream 사용 시: jetstream { store_dir: <경로>; max_memory_store: 1GB; max_file_store: 10GB } 처럼 설정 
* TLS/웹소켓/MQTT 등 확장 프로토콜: tls {}, websocket {}, mqtt {} 등의 블록을 별도로 활성화 가능 
* PID 파일 쓰기: pid_file: /경로/to/pid.txt
* Lame duck, 포트 파일 출력, 재연결 알림 등 고급 옵션: lame_duck_grace_period, port_file_dir, connect_error_reports 등 

### 문자열과 숫자
* 숫자는 K를 사용하여 1000단위를, KB를 사용하여 1024단위를 나타낸다.
* 문자열은 `" "`으로 묶어 주어야 한다.

### 변수
* 변수를 할당하고 `$`으로 호출하여 사용할 수 있다.
* 단, `" "`으로 감쌀 경우 문자열로 인식하여 치환이 이루어 지지 않는다

### include 
* 설정파일을 재사용 단위로 분리하고 include 지시어를 통해 하나의 설정파일 처럼 사용할 수 있다. 단, include는 진입 파일 기준 상대경로를 사용해야 한다.
* include 뒤에 `=` 또는 `:`를 사용하지 않아야 한다.

### 설정 리로드
`nats-server --signal reload`: 서버를 재시작하거나 커넥션을 끊지 않고도 대부분의 설정 변경 사항을 적용할 수 있다.

##### 상세 설정 참조
https://docs.nats.io/running-a-nats-service/configuration

### JetStream 리소스 제한 설정
* JetStream 리소스는 글로벌 또는 계정별로 제한하여 관리 할 수 있다.

```
jetstream {
    store_dir: /data/jetstream
    max_mem: 1G
    max_file: 100G
}

accounts {
    HR: {
        jetstream {
            max_mem: 512M
            max_file: 1G
            max_streams: 10
            max_consumers: 100
        }
    }
}
```

### NATS 서버 클러스터링
NATS는 각 서버를 클러스터 모드로 실행할 수 있다. NATS 서버는 자신이 알고 있는 모든 서버에 대해 정보를 주고 받고 연결하여 동적으로 full mesh를 형성한다. 이로 인해 클라이언트가 특정 서버에 연결되면 현재 클러스터 구성원에 대한 정보를 받기 때문에, 클러스터는 확장, 축소 및 자가 복구가 가능하고 full mesh가 반드시 명시적으로 구성될 필요가 없다.
* cluster 옵션: 클러스터의 구성원으로 동작할 네트워크 설정을 명시한다.
* routes 옵션: 시드 서버로의 경로를 설정한다.

### super-cluster
* cluster 프로토콜이 인접 서버 간 전체 메시지를 라우팅 한다면, 게이트웨이는 클러스터 간 메시지를 라우팅 해준다.
* gateway는 이름 기반으로 클러스터를 식별하며, 서버 간에 전체 메시지 토폴로지를 생성하지 않고 클러스터 단위로 전체 메시지 mesh를 형성한다.

### gateway connection
* gateway는 clustering과 서로 다른 포트를 통해 listen 한다.
* 클러스터의 모든 시드 서버를 gateways에 정의하고, 클러스터 내의 모든 게이트웨이 노드는 양방향 연결이 가능할 것을 권장한다.
* gateway를 사용하므로 인해 모든 노드가 full-mesh로 연결되는 것보다 훨씬 적은 연결로 대규모 토폴로지를 구성 가능하다.

### monitoring
NATS는 모니터링을 위한 경량 HTTP 서버를 제공한다.

### subject mapping
* 매핑 기능을 사용하여 A 주제에 대해 발행되는 메시지를 B 주제로 재매핑하여 전달 할 수 있다.
* 재매핑 시 `*` 토큰을 `$1`, `$2` 등으로 참조해 순서를 변경 가능하다.

