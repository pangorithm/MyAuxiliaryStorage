# mariaDB-replication

### 레플리케이션의 목적
* 읽기 스케일 아웃: 읽을 수 있는 여러 ㄷ데이터베이스 복제를 통해 읽기 능력 향상
* 고가용성 제공: 여러 개의 슬레이브로 단일 고장점 문제 해결
* 데이터 분석: 실제 운영환경에 영향을 주지 않고 쿼리를 수행할 수 있는 복제 환경 구성
* 재해 복구: 데이터베이스가 있는 주요 위치에서 문제가 발생했을 경우를 대비한 복사본
* 보안성: DMZ에 보안성 위험을 줄인 읽기 전용 DB 제공 

### 마스터 노드 설정
마스터 노드로 사용하기 위해서는 설정파일에서 아래의 옵션 설정이 필요하다.

#### 필수 옵션
* server-id : 유일한 값이어야하며 1 ~ 2^32-1 설정 가능
* bind-adress : mariaDB 인스턴스가 커넥션 연결을 허용할 ip 주소
* log-bin : 바이너리 로그 저장 위치

#### 추가 옵션
* expire_log_days : 빈로그 보관 일수
* sync_binlog : 쓰기할 때마다 이벤트 후에 바이너리 로그 파일을 디스크에 동기화(flush)한다. (1일 경우 활성화)
* slave_compressed_protocol : 슬레이브가 이 옵션을 함 사용할 경우 압축을 사용한다.
* binlog_fomat : [빈로그 형식](https://mariadb.com/kb/en/binary-log-formats/)(row: 행 기반 데이터 로깅, statement: 명령문 기반 데이터 로깅, mixed: 혼합 로깅)
* transaction-isolation : [트랜젝션 격리 수준](https://mariadb.com/kb/en/set-transaction/)

### 마스터 노드 준비
기존 계정에 권한을 추가해 줄 수 있지만 보안을 위해 복제 권한만 있는 복제용 계정 생성을 추천한다.
```sql
create user 'replication'@'slaveDB의ip' identified by '비밀번호'; # 계정생성
grant replication slave on *.* to 'replication'@'slaveDB의ip'; # 레플리케이션 권한 부여
flush privileges; # 권한 동기화

show master status; # 마스터 노드의 상태 확인
# 최신 상태의 binlog 파일과 binlog의 정확한 포지션을 보여준다.
```

### 슬레이브 노드 설정
* server-id : 마스터 노드와 다른 값이어야 한다.
* bind-adress : 0.0.0.0일 경우 모두 허용
* slave_compressed_protocol : 마스터와 함께 사용해야 적용된다.
* read_only : 읽기 작업만 허용한다.

### 슬레이브 생성

1. mysqldump : 전통적인 방식. 덤프하는 동안 데이터 베이스의 크기와 테이블 잠금에 의해 아주 긴 시간이 걸릴 수 있다. 전체를 덤프하는 동안 테이블을 잠가야 한다.  
```sql
flush tables with read lock; # 모든 테이블을 닫고 최신 데이터를 동기화하고 쓰기 작업을 막는다.
```

```bash
mysqldump -uroot -p --opt --routiness --triggers --events --single-transaction --master-data=2 -A > 생성파일명.sql
```
> --opt : 백업 속도와 복원 속도를 최적화 하기 위한 아래 옵션들의 묶음 옵션이다.  
	--add-locks : 복원 시 빠른 삽입  
	--create-options : create 문의 모든 mariaDB 옵션을 추가한다.  
	--disable-keys : 임포트 이후 인덱스를 생성하게 하여 덤프 복원 속도를 높여준다  
	--extended-insert : 다중 열 삽입을 사용한다.  
	--lock-tables : 덤프하기 전에 테이블을 잠근다.  
	--quick : 테이블 데이터를 한 줄씩 읽어 메모리 사용량을 줄인다.  
	--set-charset : 덤프에 문자셋을 추가한다.  

--routines : 저장 프로시저와 함수를 포함하여 백업합니다.  
--triggers : 덤프에 트리거를 추가한다.  
--events : mysql.event 테이블도 덤프한다.  
--single-transaction: InnoDB를 위한 일관적인 상태를 유지한다. 읽기가 잠긴 테이블은 플러시하지 않기 위해 InnoDB/XtraDB 상태에서만 사용한다.  
--master-data : 1은 import된 서버에 'change master to' 명령문을 수행하고, 2는 덤프에 binlog 파일과 위치 정보를 코멘트로 추가한다.  
-A : 모든 DB를 덤프한다.  

```sql
unlock tables; # 위의 mysqldump 실행 후 테이블 잠금을 해제한다.
```
  
```bash
mysql -uroot -p < 생성파일명.sql # 복원 명령

# master-data 옵션을 1로 했을 경우 빈로그 파일과 포지션을 가져온다
grep -m 1"^-- CHANGE MASTER" 생성파일명.sql 
```

2. Xtrabackup :  아주 짦은 시간동안 테이블을 잠그고 덤프를 빠르게 생성하여 압축된 DB를 전송한다.  
Xtrabackup을 사용하기 위해서는 마스터와 슬레이브 서버 모두에 Xtrabackup을 설치해야 한다.  

다음은 Xtrabackup 을 진행하는 과정이다  
```bash
슬레이브> systemctl stop mysql # 디비 인스턴스 정지
슬레이브> systemctl status mysql # 디비 정지상태 확인
슬레이브> rm -Rf /var/lib/mysql/* # 마리아 디비의 모든 data 삭제

마스터> ssh-copy-id 슬레이브ip # 슬레이브의 ssh 키 복제
마스터> innobackupex --stream=xbstream ./ | ssh root@슬레이브ip "xbstream -x -C /var/lib/mysql/" # 백업 생성 및 백업 테이터 전송

슬레이브> innobackupex --apply-log /var/lib/mysql # 백업 데이터에 로그 적용
슬레이브> chown -Rf mysql. /var/lib/mysql # 디비로 소유권 변경
슬레이브> systemctl start mysql # 인스턴스 시작
```


```sql
stop slave;
reset slave;
change master to master_host='마스터ip', master_user='replication', master_password='비밀번호', master_log_file='빈로그파일', master_log_pos=포지션값; # 슬레이브에 마스터에 대한 정보 제공
start slave;

show slave status; # 슬레이브 상태 확인
```


### GTID 복제  

Global ID를 사용하여 모든 노드에서 동일한 트랜잭션 ID를 얻어 쉽게 마스터를 변경할 수 있다.  
GTID 형식: [도메인ID]-[서버ID]-[숫자]  
GTID 복제를 활성화 하기 위해서는 기존 replication 설정에 'gtid_strict_mode'를 (마스터와 슬레이브 모두) 활성화 시키면 된다.  
`select @@global.gtid_strict_mode;` : GTID 복제 활성화 상태를 조회할 수 있다.  
`select @@global.gtid_current_pos;` : GTID 포지션 확인  

GTID 슬레이브 시작  

```sql
stop slave;
reset slave;
set global gtid_slave_pos='GTID포지션'; # 이부분이 전통 방식과 다르다.
change master to master_host='마스터ip', master_user='replication', master_password='비밀번호', master_use_gtid=slave_pos; # 슬레이브에 마스터에 대한 정보 제공
start slave;

select @@gtid_slave_pos; # 슬레이브의 gtid 포지션 확인인

show slave status; # 슬레이브 상태 확인, 전통 방식과 같은 명령어지만 더 다양한 값이 조회된다.  
```
[master_use_gtid](https://mariadb.com/kb/en/change-master-to/#master_use_gtid) : 세가지 파라미터가 사용 가능하다.  
* slave_pos: 슬레이브는 자체적으로 유지하는 마지막 GTID 위치에서 복제를 시작한다. 슬레이브 노드를 위한 안전한 방법이다.  
* current_pos: 슬레이브는 가장 최근의 GTID 위치에서 복제를 시작한다. 이는 현재 마스터의 복제 상태와 직접 동기화. gtid_strict_mode 사용 시 슬레이브 서버의 빈로그에 추가적인  트랜잭션이 삽입될 수도 있다.  
* no: GTID를 비활성화  

### 일반 복제에서 GTID 복제 방식으로의 이전
```sql
stop slave;
select @@gtid_slave_pos;
set global gtid_slave_pos='위에서 확인한 gtid pos값';
change master to master_host='마스터ip', master_user='replication', master_password='비밀번호', master_use_gtid=slave_pos;
start slave;

show slave status; # 변환 여부 확인
```

### 병렬 복제  
기본적으로 복제는 단일 스레드로 동작하지만 병력 복제를 활성화 시키면 최대 10배까지 빨라질 수 있다.  

#### 설정파일 필수옵션  
slave-parallel-threads=복제시 사용하는 스레드 수(0 비활성화 ~ 16383 최대) 너무 크게 잡으면 오히려 성능 저하가 발생할 수 있다.  

#### 추가 옵션  
slave_parallel_max_queued : SQL 스레드를 위한 메모리 제한 값  
slave_domain_parallel_threads : 마스터가 한번에 최대로 예약할 수 있는 연결의 수  
binlog_commit_wait_count : 바이너리 로그에서 I/O를 설정한다.(설정한 값 만큼의 commit 후 binlog 쓰기 작업을 시도한다.)  
binlog_commit_wait_usec : binlog를 작성하기전 대기하는 시간을 마이크로초 단위로 설정 해준다.  

### 로드벨런싱
여러 레플리케이션 디비를 두고 로드벨런서를 통해 부하분산을 수행할 수 있다.  
이때에 haproxy 또는 proxySQL 등을 사용하여 설정할 수 있고 tcpdump 또는 `watch -n1 'netstat -auntpl | grep 3306'` 명령어를 통해 로드벨런싱이 정상적으로 작동하는지 확인할 수 있다.  

### 에러처리
복제에서 에러가 발생할 경우 SQL문을 실행할 수 없으므로 복제가 정지하게 된다.  

해결방법  

```sql
show slave status\G # 슬레이브 상태 확인
stop slave;

# 해결책 1번: 바로잡기
Last_SQL_Error 등을 확인하여 문제를 보고 SQL 문을 실행해 충돌이 나지 않도록 해당 문제를 해결해 다시 복제를 시작하게 한다.

# 해결책 2-1번: 지나치기
set global sql_slave_skip_count=1; # 설정한 값만큼 복제 이벤트를 건너뛴다.
# 해결책 2-2번: GTID를 사용할 경우 sql_slave_skip_count 옵션을 사용할 수 없다.
select @@gtid_slave_pos;
set global gtid_slave_pos='위에서 확인한 포지션 값 + 건너뛸 수';

start slave; # 문제 해결 후 다시 복제 시작
```


### 빈로그 분석
```bash
mysqlbinlog 빈로그파일명 # 이 명령어를 통해 빈로그 정보 확인이 가능하다.
```


### GTID: 슬레이브를 마스터로 교체하고 복구하기
1. 마스터 교체  
  
```sql
# 슬레이브 디비
set global read_only=0; # 구슬레이브가 쓰기 가능하도록 설정한다.
stop slave; 
create user 'replication'@'slaveDB의ip' identified by '비밀번호'; # 구마스터가 신마스터를 복제할 수 있도록 신마스터 디비에 복제용 계정 생성
grant replication slave on *.* to 'replication'@'신슬레이브ip'; # 복제 권한 부여
flush privileges; # 설정 동기화

# 이전의 마스터 디비
select @@gtid_slave_pos; # 슬레이브였던 적이 없기 때문에 비어있다.
change master to master_host='신마스터ip', master_user='replication', master_password='비밀번호', master_use_gtid=current_pos;
start slave;
```

2. 이후 데이터가 손실되었던 이전의 마스터가 복제를 통해 데이터가 복구되었다.  
3. 신 마스터 디비를 정지시킨다.  
4. 복구가 완료된 구마스터 디비를 다시 마스터로 설정한다.  
5. `stop slave;` 다시 마스터가 되었으니 복제를 중지시킨다.  
6. 복구를 위해 사용했던 신 마스터를 다시 슬레이브로 시작한다.  