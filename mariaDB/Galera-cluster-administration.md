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


### Galera System Tables

```sql
# 갈레라 복제 관련 시스템 테이블 조회
show tables from mysql like 'wsrep%';
```
#### 조회 결과 테이블
* wsrep_cluster : 클러스터의 UUID와 기타 식별정보, 클러스터의 기능 등 저장
	```
	select colum_name from information_schema.columns
	where table_schema='mysql' and table_name='wsrep_cluster';
	# 조회 결과 : cluster_uuid, view_id, view_seqno, protocol_version, capabilities
	``` 
* wsrep_cluster_members : 클러스터 구성원 정보 저
	```sql
	select colum_name from information_schema.columns
	where table_schema='mysql' and table_name='wsrep_cluster_members';
	# 조회 결과 : node_uuid, cluster_uuid, node_name, node_incoming_address
	```
* wsrep_streaming_log : 진행 중인 스트리밍 트랜잭션에 대한 메타 데이터와 행 이벤트, 쓰기 집합 저장
  ```sql
    select colum_name from information_schema.columns
	where table_schema='mysql' and table_name='wsrep_streaming_log';
	# 조회 결과 : node_uuid, trx_id, seqno, flags, frag

	```

### 스키마 업그레이드
#### TOI(Total Order Isolation)
온라인 스키마의 변경 내용을 클러스터를 통해 복제하고 클러스터가 DDL문을 처리하는 동안 다른 트랜잭션의 차단을 신경쓰지 않아도 된다.  
`set global wsrep_OSU_method='TOI';`  
##### 고려사항
1. 클러스터가 모든 이전 트랜잭션을 커밋한 후에만 실행되기 때문에 이전 트랜잭션과 충돌하지 않는다.
2. DDL이 실행되는 동안 진행 중이던 트랜잭션과 동일한 DB 리소스가 관련된 트랜잭션은 커밋 시 교착 상태 오류가 발생하여 롤백된다.
3. 클러스터는 스키마 변경 쿼리를 실행하기 전에 DDL 쿼리를 복제한다. 개별 노드가 쿼리 처리에 성공했는지 여부는 알 수 없고 스키마 변경에 대한 오류 검사를 방지한다.
#### Rolling Schema Upgrade(RSU)
스키마 업그레이드 중에 고가용성을 유지하고 새 스키마 정의와 이전 스키마 정의 간의 충돌을 방지할 수 있다,  
`set global wsrep_OSU_method='RSU';`  
스키마를 변경하는 쿼리가 로컬 노드에서만 처리되고 그동안 클러스터와의 동기화가 해제된다. 스키마 변경 처리가 완료되면 지연된 복제 이벤트를 적용하여 클러스터와 동기화 한다.  
클러스터 전체의 스키마를 변경하기 위해서는 각 노드에서 차례로 쿼리를 실행해야 한다. 이때 쿼리가 아직 실행되지 않은 노드는 이전 스키마 구조를 사용하고 있다는 것을 주의해야 한다.  
한 번에 하나의 노드만 차단된다는 장점이 있지만 새 스키마 정의와 이전 스키마 정의가 복제 이벤트에서 호환되지 않을 경우 퇴출 당할 수 있다는 위험이 있다.

#### Non-Blocking Operations(Galera cluster Enterprise Edition)
장기 실행 DDL문이 클러스터 업데이트를 차단하는 경우 사용하는 비차단 방법
`set global wsrep_OSU_method='NBO';` 
DDL 쿼리가 클러스터의 모든 노드에 복제된 뒤 이전의 모든 트랜잭션이 동시에 커밋될 때까지 기다린 다음 별도로 스키마 변경을 실행한다. DDL 처리 중에는 다른 트랜잭션을 커밋할 수 없다.
##### 장점
1. NBO를 사용하여 다른 테이블을 변경할 수 있다.
2. 변경 중이 아닌 테이블에 데이터를 삽입할 수 있다.
3. 한 노드가 충돌하면 다른 노드에서 작업이 계속되고 , 성공하면 지속된다.
##### 고려사항
1. 지원되는 DDL 명령문
   ```sql
   ALTER TABLE table_name LOCK = {SHARED|EXCLUSIVE}, 추가적인_변경사항; 
   ALTER TABLE table_name LOCK = {SHARED|EXCLUSIVE} PARTITION; 
   ANALYZE TABLE 
   OPTIMIZE TABLE
	 ```
2.  지원되지 않는 DDL 명령문
   ```sql
   ALTER TABLE LOCK = {DEFAULT|NONE}; # 잠금 절을 생략하는 것이 금지된다.
   CREATE 
   RENAME 
   DROP 
   REPAIR
   ```
3. Lock 요소 없이 Create 문을 실행할 경우 오류가 발생할수 있으므로 서버 전체에서 NBO를 사용하는 것은 권장되지 않는다.
4. 변경 중인 테이블은 쓰기를 할 수 없다. 잠금 수준에 따라 읽기도 차단될 수 있다.
5. 변경되는 테이블에 대해 긴 트랜잭션이 진행 중인 경우 클러스터가 차단될 수 있다.
6. DDL 작업이 실행되는 동안 노드는 SST 기여자 노드가 될 수 없다.
7. NBO DDL 작업이 진핸되는 동안 노드가 클러스터를 떠나면 데이터 일관성이 사라녀 IST가 아닌 SST를 통해서만 클러스터에 다시 참여할 수 있다.
8. 한 번에 두 개 이상의 테이블에서 NBO 작업을 수행해서는 안된다.
9. NBO 방식으로 DDL문이 수행되는 동안 RSU 방식으로 스키마 업그레이드를 수행하면 안된다.

### Galera cluster 업그레이드
#### Rolling Upgrade
클러스터를 활성 상태로 유지해야 하고 각 노드를 업그레이드 하는데 걸리는 시간을 고려하지 않은 경우 사용하는 방식.  
각 노드를 개별적으로 종료하고 소프트웨어를 업그레이드 한 다음 노드를 재시작 한다.  
#### Bulk Upgrade
모든 노드를 종료하고 업그레이드를 수행한 다음 클러스터를 다시 온라인으로 전환한다. 빠르게 업그레이드를 진행할 수 있지만, 완전히 서비스가 중단된다.
#### Upgrading Galera Replication Plugin
```bash
# redhat
yum update galera
# debian
apt-get update
apt-get upgrade galera
```
#### Update Galera cluster
1. 모든 클러스트의 load 중지
2. 클러스터의 각 노드에서 다음 쿼리 실행
   `SET GLOBAL wsrep_provider='none';` 
   `SET GLOBAL wsrep_provider='/usr/lib64/galera/libgalera_smm.so';`
3. 클러스터의 노드 중 하나에서 다음 쿼리 실행
   `SET GLOBAL wsrep_cluster_address='gcomm://';`
4. 클러스터의 다른 모든 노드에서 다음 쿼리 실행
   `SET GLOBAL wsrep_cluster_address='gcomm://node1addr';`
5. 클러스터에서 로드를 재개

### 기본 구성요소 복구
1. 가장 최신 상태의 노드 찾기
   `SHOW STATUS LIKE 'wsrep_last_committed';`
2-1. 가장 최신 상태의 노드에서 자동 부트스트랩 설정 
   `SET GLOBAL wsrep_provider_options='pc.bootstrap=YES';`
2-2. 수동 부트스트랩
	 2-2-1. 가장 최신 상태의 노드를 마지막으로 모든 노드 종료 
	 `systemctl stop mariadb`
	 2-2-2. 가장 최신 상태의 노드에서만 클러스터 생성 명령 실행
	 `galera_new_cluster`
	 2-2-3. 나머지 노드에서 순차적으로 실행 명령
	 `systemctl start mariadb`

### 흐름제어 관리
흐름제어 모니터링 상태 변수 조: `SHOW STATUS LIKE 'wsrep_flow_control_%';`
레플리케이션 상태 확인 bash 명령어 : `myq_status wsrep`

### Auto-Eviction
##### 자동 퇴출 옵션
`evs.delay_margin` : 클러스터가 지연된 목록에 노드를 추가할 때까지 노드가 예상하는 응답 지연시간 설정
`evs.delay_keep_period` : 노드가 지연된 목록에서 제거될 때까지 응답을 유지해야 하는 기간 설정
`evs.evivt` : 클러스터가 특정 노드 값에 대한 수동 제거를 트리거 하는 지점 설정(빈 문자열로 설정할 경우 지연 목록 삭제)
`evs.auto_evict` : 자동 퇴출 발생 전 지연된 노드에서 허용되는 항목 수 설정
`evs.version` : 노드가 사용하는 EVS 프로토콜 버전 설정
##### 퇴출 상태 확인
`SHOW STATUS LIKE 'wsrep_evs_delayed';`
* wsrep_evs_state : EVS 프로토콜 내부 상태 확인
* wsrep_evs_delayed : 지연된 노드 목록 확인
* wsrep_evs_evict_list :  퇴출된 노드의 UUID 목록 확인

### 갈레라 중재인
갈레라 중재인은 복제에 참여하지 않지만 다른 모든 노드와 동일한 데이터를 수신한다. 이는 스플릿 브레인 방지 등에 사용된다.  
갈레라 중재인은 Galera Cluster와 별개의 데몬이므로 my.cnf 파일로 구성할 수 없고 클러스터와 별개로 시작해야 한다.  
##### 갈레라 중재인을 구성하는 방법
1-1. shell 명령 사용
   ```bash
garbd --group=example_cluster \
--address="gcomm://192.168.1.1,192.168.1.2,192.168.1.3" \
--option="socket.ssl_key=/etc/ssl/galera/server-key.pem;socket.ssl_cert=/etc/ssl/galera/server-cert.pem;socket.ssl_ca=/etc/ssl/galera/ca-cert.pem; socket.ssl_cipher=AES128-SHA256""
```
1-2 명령어를 실행할 때마다 입력하기 싫을 경우 설정 파일(arbitrator.config) 구성
   ```bash
arbitrator.config group = example_cluster 
address = gcomm://192.168.1.1,192.168.1.2,192.168.1.3
```
1-2-1. 갈레라 중재자 시작 : `garbd --cfg /path/to/arbitrator.config`
1-2-2. 갈레라 중재자 쉘 명령 옵션 확인 : `garbd --help`
2. 갈레라 중재자를 서비스로 시작하기, 설정 파일 작성
   설정 파일은 배포판마다 다르지만 /etc 하위 경로에 존재한다.(일반적으로 아래 위치 중에 존재)
   • /etc/defaults/ 
   • /etc/init.d/ 
   • /etc/systemd/ 
   • /etc/sysconfig/
   ```
   # Copyright (C) 2013-2015 Codership Oy 
   # This config file is to be sourced by garbd service script. 
   # A space-separated list of node addresses (address[:port]) in the cluster: 
   GALERA_NODES="192.168.1.1:4567 192.168.1.2:4567" 
   # Galera cluster name, should be the same as on the rest of the node. 
   GALERA_GROUP="example_wsrep_cluster" 
   # Optional Galera internal options string (e.g. SSL settings) 
   # see https://galeracluster.com/documentation/galera-parameters.html 
   GALERA_OPTIONS="socket.ssl_cert=/etc/galera/cert/cert.pem;socket.ssl_key=/$" 
   # Log file for garbd. Optional, by default logs to syslog 
   LOG_FILE="/var/log/garbd.log"
   ```
3. 서비스 시작
   `systemctl start garb`