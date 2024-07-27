# Galera Cluster
갈레라 클러스터는 Codership이 만든 동기적 다중 마스터 방법이다. MySQL과 MariaDB 등의 innoDB에서 사용할 수 있다. 갈레라 클러스터는 인증 기반 복제를 제공하여, 노드는 즉각 복제된 쓰기 세트를 다른 쓰기 세트에 대해 인증한다. MySQL 클러스터와는 별개의 것으로 MySQL 클러스터는 MariaDB 이식에 실패하였으며, MySQL 클러스터는 가용성과 파티셔닝을 제공하지만 갈레라 클러스터는 일관성과 가용성을 제공한다.

### 갈레라 클러스터의 특징
* 진정한 다중 마스터: 아무 때나 어떤 노드에서도 읽기와 쓰기가 가능하다.
* 동기적 복제: 슬레이브 지연이 없고 노드 충돌시에 데이터 손실이 없다.
* 일관적 데이터: 모든 노드는 같은 상태를 유짛나다.
* 멀티스레드 슬레이브: 어떤 워크로드에서도 더 나은 성능을 가능하게 한다.
* 고가용성을 위한 HA 클러스터가 필요 없다: 마스터-슬레이브 장애 극복이 없다.
* 상시대기: 장애 극복 시 정지시간이 없다.
* 애플리케이션에 있어서 편리하다: 특별한 드라이버나 애플리케이션 수정이 필요없다.
* 읽기와 쓰기 스플릿이 필요 없다: 읽기와 쓰기 요청을 스플릿할 필요가 없다.
* WAN: 갈레라 클러스터는 WAN 복제를 지원한다.
  
#### 갈레라 클러스터의 제약사항
* 3개 이상의 노드가 필요하다.(쿼럼, 투표 등 으로 인해) 또는 중재자가 필요하다.
* InnoDB/XtraDB가 아닌 다른 스토리지 엔진을 사용할 경우
  * 일관성 유지를 위해 하나의 싱글 노드에서만 쓰기 할 수 있다.
  * 다른 노드들과 복제는 완전히 지원하지 않을 수 있다.
  * 충돌 관리는 지원되지 않을 것이다.
  * 갈레라에 연결하는 애플리케이션은 동시에 하나의 싱글 노드에만 쓰기가 가능하다.

### 갈레라 클러스터의 복제 메커니즘
* 트랜잭션 재배열: 다른 노드와 트랜잭션 완료 전에 트랜잭션을 재배열한다. 이로써 성공적인 트랜잭션 인증 테스트의 숫자를 증가시킬 수 있다.
* 기록 집합: 너무 많은 노드 조정을 피하기 위해 단일 기록 집합에서 집합을 작성하여 노드 간의 작업 수를 줄인다.
* 데이터베이스 상태 장치: 읽기 전용 트랜잭션은 로컬 노드에서 처리한다. 쓰기 트랜잭션은 로컬에서 섀도우 복사본으로 수행되어 인증과 커밋을 위해 다른 노드에 읽기 집합으로 브로드캐스트 한다.
* 그룹 커뮤니케이션: 일관성을 보장하면서 노드 간의 통신을 위해 높은 단계의 추상화(gcomm 또는 spread)

노드 간의 일관성과 유사한 ID를 얻기 위해 GTID를 사용하지만 MariaDB의 복제 메커니즘이 아닌 자신만의 구현체를 사용한다.

#### 갈레라 클러스터의 제한사항
* InnoDB 테이블만 완벽히 지원한다.
* 모든 노드 간의 서로 다른 쿼리 실행 순서를 피하기 위해 모든 테이블에서 PK를 사용한다. 기본 키 없는 삭제 작업은 지원하지 않는다.
* 테이블을 잠그고 해제하는 것과 잠금 기능은 지원하지 않는다. 만약 사용한다면 무시된다.
* 갈레라 클러스터는 쿼리 캐시를 비활성화한다.
* XA(전역) 트랜잭션은 지원하지 않는다.
* 쿼리 로그는 테이블로 갈 수 없지만, 대신에 파일로 갈 수 있다.
* 저금 덜 일반적인 제약이 존재(공식페이지 참조)하지만 대부분 상황에는 신경 쓸 필요가 없다.

### 설정
기존의 my.cnf 파일과 갈레라 클러스터를 위한 설정 파일을 분리하여 my.cnf의 일부 옵션을 오버라이드 할 수 있다.  
설정파일: /etc/mysql/conf.d/galera.cnf
* wsrep_provider: 갈레라 플러그인의 위치. MariaDB 부팅 시 갈레라를 로딩할 수 있게 한다.
* wsrep_cluster_name: 클러스터의 이름. 같은 네트워크 서브넷에 여러개의 서버가 있을 경우, 노드가 잘못된 클러스터로 들어가는 것을 막기 위해 사용한다.
* wsrep_node_name: 현재 노드의 고유한 이름
* wsrep_node_address: 갈레라 통신을 위해 전용이며 사설인 네트워크를 사용하고 있다면 이 옵션을 사용할 수 있다. 고유한 값이어야 한다.
* wsrep_cluster_address: 클러스터의 멤버 리스트
* wsrep_provider_option: 추가적인 옵션
  * gcache.size: 갈레라를 위한 전용 캐시.높은 트래픽 부하에서도 항상 모든 노드를 동기화하고 싶다면 이 값을 키워야한다. 그렇지 않을 경우 동기화 모드에 들어가지 못할 수 있다.
  * gcache.name: gcache가 저장될 위치
  * gcache.page_size: 페이지 스토리지에 있는 페이지 파일의 크기
* wsrep_retry_autocommit: 충돌이 발견되면 실패 이전에 재시도의 횟수를 설정
* wsrep_set_method: 노드 간의 전송 방법
* wsrep_slave_threads: 슬레이브 기록 집합을 적용하기 위해 사용하는 스레드의 수
* swrep_replication_myisam: MyISAM 복제를 활성화한다. 하지만 MyISAM에 관한 트랜잭션이 없으므로 무시해도 된다.
* wsrep_sst_receive_address: IP를 지정할 수 있으며 일반적으로 VIP를 이용하여 다른 서버에 접근할 때 원격 서버 입장에서 지금 서버가 정확한 IP주소로 오는지 알 수 없을 때 사용한다.
* wsrep_notify_cmd: 매번 갈레라 이벤트 발생 시 스크립트를 실행한다. 노드가 더는 클러스터 멤버가 아닐 때 사용할 수 있다.
MariaDB 오버라이드 옵션
* binlog_format: 로그 형식을 정의
* innodb_autoinc_lock_mode: 락 메커니즘 설정
* innodb_flush_log_at_trx_commit: 성능을 최정화 할 수 있지만, 전체 갈레라 클러스터에 전력 손실이 있을 경우 데이터를 잃을 수 있다.
* query_cach_size: 0으로 설정하여 쿼리 캐시 비활성화

### 클러스터의 상태 정보(SHOW STATUS LIKE 'wsrep_%';)
* wsrep_local_state_comment: 노드의 현재 상태
  * Joining: 노드가 클러스터에 들어가고 있다.
  * Donor/Desynced: 노드가 Donor 모드(다른 노드에 데이터를 복제)에 있거나 다른 노드와 동기화되지 않았다.
  * Joined: 노드가 클러스터에 들어왔다.
  * Synced: 노드가 클러스터의 멤버다.
* wsrep_cluster_size: 갈레라 클러스터에 있는 노드 멤버의 수를 나타낸다.
* wsrep_cluster_state_uuid: 고유한 클러스터 ID. 모든 노드가 서로 연결됨을 확신하려면 같은 UUID를 가지고 있어야 한다.
* wsrep_cluster_conf_id: 설정 ID로 모든 노드에서 설정이 같음을 확신하기 위해 같은 값을 가져야 한다.
* wsrep_cluster_conf_id: 노드에 관하여 현재 복제 상태를 알려준다.
  * Primary: 마스터 상태의 노드
  * Non-Primary: 마스터가 아닌 노드
  * Disconnected: 클러스터에 연결되지 않은 노드
* wsrep_connected: 복제를 위한 네트워크 연결을 나타낸다.
* wsrep_ready: 노드가 준비되고 SQL 트랜잭션을 다룰 수 있음을 나타낸다.

### 전송 방법
클러스터에서 어떤 노드를 통합해야 할 때 동작 중인 노드는 상태를 바꾸고 Doner가 되는 것으로 설계되어 있다. Doner는 새로운 노드에 데이터를 전송하는 전용 노드가 된다. 노드가 도너 모드일 때, 데이터 전송이 끝날 때까지 이 노드에서 트랜잭션은 잠긴다.

#### 데이터 전송 방식
* SST(State Snapshot Transfer): 전체 백업을 하기 위한 방법
* IST(Incremental State Transfer): 잃어버린 데이터를 전송하기 위한 방법
모든 복제 메커니즘은 SST와 IST를 동시에 할 수 없다.

* mysqldump: SST는 가능하지만 IST는 불가능하다. 클러스터에 노드를 추가하면 일정 시간 동안 정지되어 복제를 재개하거나 잃어버린 데이터를 다시 찾을 수 없다. 클러스터에 노드를 통합할 때 IST 대신 SST 전송을 한다.
* Xtrabackup: SST와 IST 전송을 수행하는 빠른 방법이다. 가장 적은 잠금 시간이 걸린다.
* rsync: SST와 IST를 전송하기 위한 가장 빠른 방법이다. Xtrabackup에 비해 도너 노드에서 긴 시간 동안 트랜잭션을 잠근다.

### 이중 설계
* 갈레라 클러스터를 마스터로 두고 클러스터의 노드들을 복제한 읽기 슬레이브를 두는 구조
* 노드가 여러 네트워크에 분산된 WAN 복제 구조의 갈레라 클러스터. 지연과 타임아웃을 고려해 다음의 옵션을 수정해야 한다.
  * evs.keepalive_period: 생존 확인 신호 전송 주기
  * evs.inactive_check_period: 피어의 정지 여부 확인 주기
  * evs.suspect_timeout: 다른 노드에 의해 특정 노드가 죽었다고 고려할 수 있는 정지 기간. 모든 노드가 특정 노드가 죽었다고 확인하면 클러스터에서 해당 노드를 퇴출 시킨다.
  * evs.inactive_timeout: 노드가 죽었다고 설정될 정지의 최대 제한치(evs.suspect_timeout 값은 이 값을 건너뛸 수 있다.)
  * evs.install_timeout: 인스톨 메시지 응답을 기다리는 타임아웃
* 갈레라 클러스터를 일반적 복제한 DR 갈레라 클러스터 구조
  * 갈레라 클러스터는 모든 노드간에 동기화 되어있다
  * DR은 마스터 클러스터의 속도를 줄이지 않는다.
  * 구성 방법
    1. 갈레라 클러스터를 구성한다
    2. DR 갈레라 클러스터를 구성한다
    3. 갈레라의 마스터 노드와 갈레라 DR 슬레이브 노드 간의 데이터를 동기화 한다.
    4. 이중 마스터 복제를 생성한다.(복잡한 스플릿 브레인을 피하기 위해 VIP 또는 클러스터 소프트웨어 없이 복제한다)