# 갈레라 클러스터 관리

### Node Provisioning
새 노드 또는 실페한 노드의 상태가 클러스터와 다른 경우 클러스터와의 동기화가 필요한데, 이때 새 노드를 프로비저닝하거나 실패한 노드를 복구하는 과정은 기본적으로 노드를 클러스터에 연결하는 프로세스와 동일하다.
Galera는 `grastate.dat` 파일에서 초기 노드 상태 ID를 읽어오는데, 이 파일은 `wsrep_data_dir` 매개변수로 지정된 디렉토리에 저장된다. 노드가 정상적으로 종료될 때마다 Galera는 이 파일에 상태를 저장한다.
ps. UUID가 유효한 상태라면 클러스터가 해당 노드를 인식할 수 있기 때문에 seqno가 -1(undifined)이라 하더라도 IST를 통해서 복구시킬 수 있다.

### 노드 합류 방식
1. 자동: 노드가 클러스터에 합류하려고 시도할 때, 그룹 통신 계층은 Primary Component에 있는 노드 중 하나를 상태 제공자로 자동 선택한다.
2. 수동: `wsrep_sst_donor` 매개변수를 사용하여 수동으로 상태 제공자를 지정할 수 있다. 지정된 상태 제공자가 Primary Component의 일부가 아니면 상태 전송이 실패하고 합류 시도가 중단된다.
상태 전송은 노드와 제공자에게 큰 부담을 준다. 따라서 수동으로 상태 제공자를 선택하고, 전송 중에는 클라이언트 연결을 다른 노드로 전환하는 것이 권장된다. 전송 완료 후 노드는 최신 Primary Component 상태로 동기화되고 클라이언트 연결을 받을 수 있다.

`wsrep_sst_method = rsync` : sst 전송 방식 설정, 수신 노드에서 설정한다.
`wsrep_sst_donor = "node1, node2"` : 기본 기여자 노드 설정

### 논리적 상태 스냅샷
#### mysqldump
* 라이브 서버에서 사용 가능하며 완전히 초기화된 서버만 논리 상태 스냅샷을 수신할 수 있다.
* 송신 노드와 수신 노드가 같은 데이터베이스 구성(버전, 파일 형식 등)일 필요가 없다.
* 수신 디비 서버가 송신 노드의 루트 연결을 허용해야 한다.
* 수신 노드에 완전한 기능의 디비가 있어야 한다.
* 규모가 큰 DB에서는 다른 방식보다 몇배는 느리지만 매우 작은 DB의 경우 오히려 다른 방식보다 빠를 수 있다.
##### SST 권한 설정
mysqldump를 통해 SST 전송을 하기 위해서는 루트 연결이 필요하므로 아래의 설정을 설정파일에 추가해줘야 한다.
`wsrep_sst_auth = wsrep_sst_username:password`

이전에 클러스터에서 이 노드를 사용한 적이 없을 경우 복제를 비활성화 한 상태로 시작해야한다.
`systemctl start mysql --wsrep-on=OFF`
각 노드에 mysqldump SST를 위한 권한을 부여해준다. 
```sql
GRANT ALL ON *.* TO 'wsrep_sst_user'@'node1_IP_address' IDENTIFIED BY 'password'; GRANT ALL ON *.* TO 'wsrep_sst_user'@'node2_IP_address' IDENTIFIED BY 'password'; GRANT ALL ON *.* TO 'wsrep_sst_user'@'node3_IP_address' IDENTIFIED BY 'password';
```
다른 노드를 구성하는 동안 디비를 정지한다.
`systemctl stop mysql`


### 물리적 상태 스냅샷
* 한 노드에서 다른 노드의 디스크로 데이터를 물리적으로 복사하므로 각 DB 서버가 상호작용할 필요가 없다.
* 기여자 노드가 이전에 결합한 노드의 디스크에 내용을 덮어쓰기 때문에 디비가 작동 상태일 필요가 없다.
* 더 빠르다.
* 기여자 노드와 조인 노드가 동일한 디비 디렉토리 레이아웃과 스토리지 엔진을 사용해야 한다.
* 초기화된 스토리지 엔진이 있는 서버에서는 허용되지 않는다. 즉, SST가 필요한 경우 변경 사항을 적용하기 위해서는 DB 서버를 다시 시작해야 한다. 디비 서버는 스토리지 엔진 없이는 인증을 수행할 수 없으므로 SST가 완료될때 까지 클라이언트에서 연결할 수 없다.

#### rsync
SST를 위한 가장 빠른 방법.  
전송 중 기여자 노드를 차단하지만, 디비 구성이나 루트 액세스가 필요 없으므로 구성이 간편하다.  
xtrabackup에 비해 1.5~2배 빠르다.  단, 버전간 비호환 문제가 자주 발생한다.
```
wsrep_sst_method=rsync
```
#### xtrabackup
시스템 테이블을 복사하는데 걸리는 짧은 시간 동안만 도너 노드를 차단시킨다.  
복사하는 데이터의 양이 많을 경우 기여자 노드의 성능이 저하될 수 있다.  
기여자 서버에 대한 루트 접근 권한이 있어야 한다.
```
[mysqld] 
wsrep_sst_auth = sst_user:sst_password
wsrep_sst_method = xtrabackup-v2 
datadir = /path/to/datadir
```
```sql
mysql> CREATE USER ''@'localhost' IDENTIFIED BY ''; 
mysql> GRANT BACKUP_ADMIN, PROCESS, RELOAD ON *.* TO ''@'localhost'; 
mysql> GRANT SELECT ON performance_schema.keyring_component_status TO 'sst_user'@'localhost' ; mysql> GRANT SELECT ON performance_schema.log_status TO 'sst_user'@'localhost' ;
```

#### clone
8.0.22버전 이상의 mySQL Galera cluster 에서 사용할 수 있다. xtrabackup보다 훨씬 빠르지만 DDL이 실행되는동안 기여자를 차단한다.
```
# 조인 노드 설정파일
wsrep_sst_method=clone

# 기여자 노드 설정파일
wsrep_sst_auth = sst_user:sst_password
```
```sql
mysql> CREATE USER ''@'localhost' IDENTIFIED BY ''; mysql> GRANT CREATE USER, SUPER ON *.* TO ''@'localhost'; 
mysql> GRANT INSERT, DELETE ON mysql.plugin TO 'sst_user'@'localhost'; 
mysql> GRANT UPDATE ON performance_schema.setup_instruments TO ''@'localhost'; mysql> GRANT UPDATE ON performance_schema.setup_consumers TO ''@'localhost'; mysql> GRANT BACKUP_ADMIN ON *.* TO ''@'localhost' WITH GRANT OPTION; 
mysql> GRANT EXECUTE ON *.* TO 'sst_user'@'localhost' WITH GRANT OPTION; 
mysql> GRANT SELECT ON performance_schema.* TO 'sst_user'@'localhost' WITH GRANT OPTION;
```


### Scriptable State Snapshot Transfers
DB 서버 외부에서 실행되는 프로레스를 통해서 관리한다.  기본 동작이 제공하는 것보다 더많은 프로세스가 필요한 경우 SST를 관리하기 위해 사용자 정의 쉘 스트립트용 인터페이스를 제공한다.

#### 공통 SST 스크립트
wsrep_sst_common이란 파일명으로 리눅스의 경우 /usr/bin에 설치한다.  
일반적으로 인수 목록 구분 분석, 오류 로깅 등을 위한 기능을 제공한다.  
수신 노드의 스토리지 엔진 초기화는 상태 전송이 완료된 후에만 발생한다고 가정한다. 이는 소스 데이터 디렉터리의 내용을 대상 데이터 디렉터리에 복사한다는 의미이다.

#### 상태 전송 스크립트 매개변수
갈레라 클러스터는 SST를 위한 외부 프로세스를 시작할 때 자체 상태 전송 스크립트를 구성하는 데 사용할 수 있는 여러 매개변수를 스크립트로 전달한다.

#### 일반 매개변수 (General Parameters)

1. **--role**: 스크립트에 노드의 역할을 지정하는 문자열(`donor` 또는 `joiner`)을 전달하여 노드가 상태 스냅샷을 보내는지(receive) 받는지(send) 여부를 나타낸다.
2. **--address**: 조인 노드의 IP 주소를 전달한다.
    - **joiner 실행 시**: `wsrep_sst_receive_address` 매개변수의 값 또는 `<ip_address>:<port>` 형식의 기본값을 사용한다.
    - **donor 실행 시**: 상태 전송 요청의 값을 사용한다.
3. **--auth**: 노드 인증 정보를 전달한다.
    - **joiner 실행 시**: `wsrep_sst_auth` 매개변수의 값을 사용한다.
    - **donor 실행 시**: 상태 전송 요청의 값을 사용한다.
4. **--datadir**: 데이터 디렉토리 경로를 전달한다. `mysql_real_data_home` 매개변수로 설정된다.
5. **--defaults-file**: `my.cnf` 구성 파일의 경로를 전달한한다.

#### Donor 전용 매개변수 (Donor-specific Parameters)

1. **--gtid**: 노드가 마지막으로 커밋된 트랜잭션의 상태 UUID와 시퀀스 번호(seqno)로 구성된 글로벌 트랜잭션 ID를 전달.
2. **--socket**: 필요한 경우, 통신을 위한 로컬 서버 소켓을 전달.
3. **--bypass**: 스크립트가 실제 데이터 전송을 건너뛰고 글로벌 트랜잭션 ID만 수신 노드에 전달할지 여부를 지정. 즉, 노드가 Incremental State Transfer를 시작할지 여부를 나타낸다.

#### 논리적 상태 전송 전용 매개변수 (Logical State Transfer-specific Parameters)
 `wsrep_sst_mysqldump` 상태 전송 스크립트에만 사용되며, 보내는 노드와 받는 노드 모두에 의해 전달된다.

1. **--user**: 스크립트가 donor와 joiner 데이터베이스 서버에 연결하는 데 사용하는 데이터베이스 사용자 이름을 전달. 이 사용자는 양쪽 서버에서 동일해야 하며 `wsrep_sst_auth` 매개변수에서 정의된다.
2. **--password**: 데이터베이스 사용자의 비밀번호를 전달합니다. 이 비밀번호는 `wsrep_sst_auth` 매개변수에 의해 구성된다.
3. **--host**: joiner 노드의 IP 주소를 전달.
4. **--port**: joiner 노드와 함께 사용할 포트 번호를 전달합니다.
5. **--local-port**: 상태 전송을 보내는 데 사용할 포트 번호를 전달합니다.

### 호출 규칙
상태 스냅샷 전송을 위해 사용자 정의 스크립트를 작성할 때 Galera Cluster가 스크립트를 호출하는  위해 따라야 하는 특정 규칙

#### 조인 노드 (Receiver)

조인 노드가 상태 스냅샷 전송을 요청할 때, 다음 단계가 수행된다  
1. **일반 매개변수 전달**
    - 조인 노드가 상태 스냅샷 전송을 요청하면 일반 매개변수를 포함한 여러 인수가 SST 스크립트에 전달된다. 이 매개변수는 필요에 따라 사용할 수 있다.
2. **노드 준비**
    - 스크립트는 노드가 상태 스냅샷 전송을 받을 준비. 예를 들어, `wsrep_sst_rsync`의 경우, 스크립트가 rsync를 서버 모드로 시작한다.
3. **수신 준비 신호**
    - 노드가 상태 전송을 받을 준비가 되면 표준 출력에 다음 문자열을 출력한다: `ready <address>:port\n`
    - 예시: `ready 192.168.1.1:4444`
4. **전송 요청**
    - 조인 노드는 donor 노드로 상태 전송 요청을 보낸다. 이 요청에는 조인 노드의 주소와 포트 번호, `wsrep_sst_auth` 매개변수의 값, 스크립트 이름이 포함된다.
5. **상태 전송 완료**
    - 조인 노드가 상태 전송을 받고 적용을 완료하면, 수신한 상태의 글로벌 트랜잭션 ID를 표준 출력에 출력한다.
    - 예시: `e2c9a15e-5485-11e0-0800-6bbb637e7211:8823450456`
    - 스크립트는 상태 전송이 성공했음을 나타내기 위해 0 상태 코드로 종료한다.

#### 도너 노드 (Sender)

도너 노드가 상태 스냅샷 전송을 수행할 때, 다음 단계가 수행된다:

1. **일반 매개변수 전달**
    - 도너 노드가 상태 스냅샷 전송을 요청하면 일반 매개변수를 포함한 여러 인수가 SST 스크립트에 전달됩니다. 이 매개변수는 필요에 따라 사용할 수 있다.
2. **신호 수신**
    - 스크립트가 실행되는 동안 Galera 클러스터는 다음 신호를 수신합니다. 이 신호는 표준 출력으로 출력된다:
        - `flush tables\n`: 데이터베이스 서버에 FLUSH TABLES를 실행하도록 요청합니다. 완료되면 데이터 디렉토리에 `tables_flushed` 파일이 생성된다.
        - `continue\n`: 데이터베이스 서버에 트랜잭션 커밋을 계속할 수 있음을 알린다.
        - `done\n`: 상태 전송이 완료되고 성공했음을 데이터베이스 서버에 알리는 필수 신호.
3. **상태 전송 완료**
    - `done\n` 신호를 보낸 후 스크립트는 0 반환 코드로 종료한다.
4. **실패 시 처리**
    - 실패 시 스크립트는 발생한 오류에 해당하는 코드를 반환해야 한다. 도너 노드는 이 코드를 그룹 통신을 통해 조인 노드에 반환한다. 데이터 디렉토리가 일관되지 않은 상태가 되면 조인 노드는 클러스터를 떠나고 상태 전송이 중단한다.

#### 스크립트 사용 설정 (Enabling Scriptable SST’s)

커스텀 SST 스크립트를 직접 작성하거나 `wsrep_sst_common`을 사용하는 경우, 이를 활성화하는 과정은 동일하다. 스크립트 파일명은 `wsrep_sst_<name>` 형식을 따라야 하며, `<name>`은 구성 파일의 `wsrep_sst_method` 매개변수에 지정된 값이다.

예를 들어, `wsrep_sst_galera-sst`라는 스크립트를 작성한 경우, `my.cnf`에 다음 줄을 추가한다
`wsrep_sst_method = galera-sst`
노드가 시작될 때 커스텀 스크립트를 사용하여 상태 스냅샷 전송을 수행합니다.