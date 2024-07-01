# 멀티 마스터 복제: 갈레라 클러스터

### 갈레라 클러스터의 장점
* 진정한 다중 마스터: 언제나 어떤 노드에서도 읽기와 스기가 가능하다.
* 동기적 복제: 슬레이브 지연이 없고 노드 충돌 시에 데이터 손실이 없다.
* 일관적 데이터: 모든 노드는 같은 상태를 유지한다.
* 멀티스레드 슬레이브: 어떠한 워크로드에서도 더 나은 성능을 가능케 한다.
* 별도의 HA 클러스터가 필요 없다: 마스터-슬레이브 장애 극복이 없다.
* 상시대기: 장애 극복 시 정지시간이 없다.
* 편리하다: 특별한 드라이버나 애플리케이션 수정이 필요 없다
* 읽기와 쓰기 스플릿이 필요 없다: 읽기와 쓰기 요청을 분리할 필요가 없다
* WAN: WAN 복제를 지원한다.

### 다른 스토리지 엔진을 사용할 경우의 제약사항
1. 일관성을 유지하기 위해 하나의 싱글 노드에서만 쓰기를 할 수 있다.
2. 다른 노드와의 복제는 완전히 지원하지 않을 수 있다.
3. 충돌 관리는 지원되지 않을 것이다.
4. 갈레라에 연결하는 애플리케이션은 동시에 하나의 싱글 노드에만 쓰기가 가능하다.

### 갈레라 클러스터가 복제를 위해 사용하는 메커니즘 요약
* 트랜잭션 재배열: 다른 노드와 트랜잭션 완료 전에 트랜잭션을 재배열한다. 이로써 성공적인 인증 테스트의 수를 증가시킬 수 있다.
* write-set: 너무 많은 노드 조정을 피하기 위해 단일 기록집합에서 집합을 작성하여 노드 간의 작업 수를 줄인다.
* 데이터베이스 상태 장치: 읽기 전용 트랜잭션은 로컬 노드에서 처리한다. 쓰기 트랜잭션은 로컬에서 섀도우 복사본으로 수행되어 인증과 커밋을 위해 다른 노드에 읽기 집합으로 브로드캐스트한다.
* 그룹 커뮤니케이션: 일관성을 보장하며 노드 간의 통신을 위한 높은 단계의 추상화

### 갈레라 클러스터의 제한사항
* InnoDB 테이블만 완벽히 지원한다. MyISAM은 부분적으로 지원하며 TokuDB는 계획에 있지만 아직 지원하지 않는다.
* 모든 노드 간의 서로 다른 쿼리 실행 순서를 피하기 위해 모든 테이블에서 기본 키를 사용한다.
* 테이블을 잠그고 해제하는 것과 잠금 기능은 지원하지 않는다. 만일 사용할 경우 무시된다.
* 쿼리 캐시를 비활성화 한다.
* XA 트랜잭션은 지원하지 않는다.
* 쿼리 로그는 테이블로 갈 수 없지만 파일로는 갈 수 있다.
* 특수한 경우의 제약이 추가적으로 존재하지만 일반적으로는 신경 쓸 필요 없다.

### 갈레라 설정
* wsrep_provider: 갈레라 플러그인의 경로
* wsrep_cluster_name: 클러스터의 이름. 같은 네트워크 서브넷에 여러개의 서버가 있을 경우 원치 않는 노드가 잘못된 클러스터로 들어가는 것을 막기 위해 사용한다.
* wsrep_node_name: 현재 노드의 고유한 이름
* wsrep_node_adress: 갈레라 통신을 위한 사설 네트워크를 사용한다면 설정 할 수 있다. 그렇지 않을 경우 공인 IP를 사용해야 한다.
* wsrep_cluster_address: 클러스터의 멤버 리스트
* wsrep_provider_options: "추가 옵션"
	* gcache.size: 갈레라 전용 캐시 크기. 높은 부하에서도 항상 모든 노드를 동기화하고 싶다면 이를 키워야 한다.
	* gcache.name: gcache가 저장될 위치와 이름
	* gcache.page_size: 페이지 스토리지에 있는 페이지 파일의 크기
* wsrep_retry_autocommit: 충돌이 발견되면 실패 이전에 재시도의 횟수를 설정할 수 있다.
* wsrep_sst_method: 노드간의 전송 방법
* wsrep_slave_threads: 슬레이브 기록 집합을 적용하기 위한 스레드 수
* wsrep_replication_myisam: MyISAM 복제 활성화. 하지만 MyISAM에 관한 트랜잭션이 없으므로 무시해되 된다.
* wsrep_sst_receive_adress: IP 또는 VIP를 이용하여 다른 서버에서 접근할 때 원격 서버 입장에서 지금 서버가 정확한 IP 주소로 알 수 없을 때 사용한다.
* wsrep_notifi_cmd: 매번 갈레라 이벤트 발생 시 스크립트를 실행한다.

#### 오버라이드 되는  MariaDB 옵션
* binlog_fomat: 로그 형식 정의
* innodb_autoinc_lock_mode: 락 메커니즘 사용 정의
* innodb_flush_log_at_trx_commit: 이 옵션으로 성능을 최적화 할 수 있지만, 전체 갈레라 클러스터에 전력 손실이 있을 경우 위험할 수 있다.
* query_cache_size: 쿼리 캐시 활성화 여부

### 첫번째 부팅

`service mysql start --wsrep_cluster_address='gcomm://'` : 첫번째 노드에서 갈레라 클러스터 생성
`service mysql start` : 나머지 노드
`show status like 'wsrep_%'` : 클러스터 상태 확인
##### 클러스터 상태 정보
* wsrep_local_state_comment: 노드의 현재상태
	* Joining: 클러스터에 들어가는 중
	* Donor/Desynced: 노드가 기여자 모드에 있거나 다른 노드와 동기화 되지 않았다.
	* Joined: 노드가 클러스터에 들어왔다
	* Synced: 노드가 클러스터의 멤버이다
* wsrep_cluster_size: 갈레라 클러스터에 있는 노드 멤버의 수
* wsrep_cluster_state_uuid: 고유한 클러스터ID. 모든 노드는 서로 연결됨을 확신하려면 같은  UUID를 갖져야 한다.
* wsrep_cluster_conf_id: 설정 ID
* wsrep_cluster_status: 노드의 현재 복제 상태
	* Primary: 마스터 상태
	* Non-Primary: 마스터가 아닌 노드
	* Disconnected: 클러스터에 연결되지 않은 노드
* wsrep_conneted: 복제를 위한 네트워크 연결 상태
* wsrep_ready: 노드가 준비되어 SQL 트랜잭션을 다룰 수 있음을 나타낸다.

### 전송방법
* SST(Staet Snapshot Transfer): 전체 백업을 위한 방법
* IST(Incremental State Transfer): 잃어버린 데이터를 전송하기 위한 방법
모든 복제 메커니즘은 이 두가지를 동시에 쓸 수는 없다.  

* mysqldump: SST만 가능하며 가장 기본적이고 느린 방법
* Xtrabackup: 별도의 설치가 필요한 SST와 IST를 모두 지원하고 가장 적은 잠금 시간을 갖는 빠른 방법
* rsync: IST와 SST 전송을 위한 가장 빠른 방법

### 도너 노드로 만들기
`wsrep_sst_donor=노드이름` : 설정파일 사용
`set global wsrep_sst_donor=노드이름` : 명령어 사용

### Garb: 쿼럼 방식
가짜 노드를 제공하는 방법
/etc/default/garb : garb 설정파일 경로

### 성능 튜닝
#### 병렬 스렐이브 스레드
물리적 코어 당 4개의 스레드를 두는 것을 권장한다.
`wsrep_slave_threads=코어수*4`
위의 값을 `wsrep_cert_deps_distance` 값 이상으로 설정하지 않아야 한다.
#### Gcache 크기
계산 방법
Gcache크기 = (received_by_value2 - received_by_value1) / (time2 - time1)

