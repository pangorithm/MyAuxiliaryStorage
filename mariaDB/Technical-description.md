
### 동기식 복제와 비동기식 복제
* 동기식 복제: 노드는 단일 트랜잭션에서 모든 복제본을 업데이트 하여 모든 복제본을 동기화한다. 즉, 트랜잭션이 커밋되면 모든 노드가 동일한 값을 갖는다.
* 비동기식 복제: 지연 복제 방식. 마스터 노드가 복제본 업데이트를 다른 노드에 비동기적으로 전파한다. 즉, 트랜잭션이 커밋되고 짧은 시간동안 일부 노드는 다른 값을 갖는다.

### 동기식 복제의 장단점
#### 장점
* 고가용성: 노드 충돌 시 데이터 손실이 없다. 데이터 복제본은 일관성을 유지한다. 복잡하고 시간이 많이 걸리는 장애복구가 필요가 없다.
* 성능 향상: 모든 노드에서 트랜잭션을 서로 병렬로 실행하여 성능을 향상 시킬 수 있다.
* 모든 노드의 인과성 보장: 트랜잭션 직후 다른 노드에서 실행된 쿼리 또한 항상 트랜잭션의 효과를 볼 수 있다.
#### 단점
* 노드 수가 증가할 수록 트랜잭션 응답 시간과 충돌 및 교착 상태 발생 확률이 기하급수적으로 증가한다.

### 동기식 복제의 문제 해결방법

* 그룹통신: 복제 데이터의 일관성을 보장하기 위해 노드의 통신 패턴을 높은 수준으로 추상화 시킨다.
* 쓰기 세트: DB 쓰기를 단일write-set으로 묶어 노드가 한번의 하나씩 조정하는 것을 방지한다.
* 디비 상태머신: 읽기 전용 트랜잭션을 로컬에서 처리하도록 한다. update 가 실행될 경우 먼저 DB에서 얕은 복사를 실행한 뒤에 다른 DB에서 읽기 세트로 통신하여 인증을 커밋 할 수 있게 한다.
* 트랜잭션 재정렬: DB가 트랜잭션을 커밋하고 다른 DB에 브로트캐스트 하기 전에 트랜잭션을 재정렬하여, 인증을 성공적으로 통과하는 트랜잭션의 수를 증가시킨다.

### 인증복제 기반
#### 인증 기반 복제의 조건
1. 트랜잭션 데이터베이스: 적용되지 않은 변경 사항을 롤백할 수 있어야 한다.
2. 원자적 변경: 복제이벤트는 DB를 원자 단위로 변경할 수 있어야 한다.
3. 전역 순서: 복제 이벤트는 전역적으로 순서를 지정해야 한다. 즉, 모든 인스턴스에서 동일한 순서여야 한다.

#### 인증기반 복제의 작동 방식
1. 인증 기반 복제는 충돌이 없다는 가정하에 트랜잭션이 커밋 지점에 도달할 때까지 통상적으로 트랜잭션을 실행한다(낙관적 실행). 
2. 클라이언트가 커밋 명령을 실행하지만 실제로는 커밋이 발생하기 전에 트랜잭션과 변경된 행의 기본 키에 의해 DB에 수행된 모든 변경 사항이 쓰기 집합에 수집되고, 이 쓰기 집합을 다른 모든 노드에 전송한다.
3. 쓰기 집합은 기본키를 사용하여 결정론적 인증 테스트를 클러스터의 모든 노드에서 거친다. 이 테스트는 노드가 해당 쓰기집합을 적용할 수 있는지 여부를 결정한다.
4. 인증 테스트가 실패하면 노드는 해당 쓰기집합을 삭제하고 트랜잭션을 롤백한다. 성공할 경우 트랜잭션이 커밋되고 쓰기 집합이 나머지 클러스터에 적용된다.

### Galera 클러스터의 인증 기반 복제
갈레라 클러스터에서의 인증 기반 복제 구현은 트랜잭션의 전역 순서에 따라 달라진다.

1. 갈레라 클러스터는 복제하는 동안 각 트랜잭션에 전역 순차 번호를 할당한다(GTID). 
2. 트랜잭션이 커밋 지점에 도달하면 노드는 마지막으로 성공한 트랜잭션의 순차 번호와 비교한다. 이 두 지점 사이의 트랜잭션들은 서로 영향을 끼치지 못한다.
3. 두 지점 사이의 모든 트랜잭션을 해당 트랜잭션과 기본 키 충돌이 있는지 검사한다. 충돌이 있을 경우 인증 테스트는 실패한다.
4. 모든 복제본에서 동일한 순서로 트랜잭션을 수신하므로 모든 노드는 인증 테스트에서 동일한 결정을 내린다. 트랜잭션을 시작한 노드는 클라이언트에 트랜잭션이 커밋되었는지 여부를 알릴 수 있다.

### 복제 API
동기식 복제 시스템은 일반적으로 빠른 복제를 사용한다. 클러스터의 노드는 단일 트랜잭션을 통해 복제본을 업데이트하여 다른 모든 노드와 동기화한다. 즉, 트랜잭션이 커밋되면 모든 노드가 동일한 값을 갖게된다.  
#### 갈레라 클러스터의 구성요소
* DBMS: 개별 노드에서 실행되는 DB 서버. 갈레라 클러스터는 MySQL, MairaDB, Percona XtraDB를 사용할 수 있다.
* wsrep API: DB 서버에 대한 인터페이스로서, 복제 공급자. 
* wsrep Hook: 쓰기 세트 복제를 위해 DB 서버 엔진과 통합된다.
- dlopen():  wsrep Hook에서 wsrep 공급자를 사용할 수 있게 한다.
- 갈레라 복제 플러그인: 사용할 경우 쓰기 설정 복제 서비스를 사용할 수 있다.
- 그룹 통신 플러그인: 갈레라 클러스터에서 사용할 수 있는 여러 그룹통신 시템(gcomm, spread 등)

#### wsrep API
애플리케이션 콜백 및 복제 플러그인 호출 집합을 정의한다.  
wsrep API는 DB 서버에 상태가 있는 것으로 간주하는 복제 모델을 사용한다. 이 상태는 DB의 콘텐츠를 나타내며 DB가 사용중이고 클라이언트가 DB 콘텐츠를 수정하면 DB의 상태가 변경된다. wsrep API는 DB의 상태 변경을 일련의 원자적 변경 또는 트랜잭션으로 나타낸다. 
Galera 클러스터의 상태변경 처리방식은 다음과 같다.
1. 클러스터의 한 노드에서 DB의 상태 변경이 발생한다.
2. DB에서 wsrep 혹은 변경사항을 write-set으로 변환한다.
3. dlopen()을 호출하면 wsrep 공급자 함수를 wsrep Hook에서 사용할 수 있게 된다.
4. 갈레라 복제 플러그인이 클러스터에 대한 쓰기 설정 인증 및 복제를 처리한다.

#### 갈레라 복제 플러그인
wsrep API를 구현하며, wsrep 공급자로 작동한다. 아래의 요소로 이루어져 있다.
* 인증 레이어 : write set을 준비하고 인증 검사를 수행하여 적용할 수 있는지를 확인한다.
* 복제 계층: 복제 프로토콜을 관리하고 전체 주문 기능을 제공한다.
* 그룹 커뮤니케이션 프레임워크: Galera 클러스터에 연결되는 다양한 그룹 커뮤니케이션 시스템을 위한 플러그인 아키텍쳐를 제공한다.

#### 그룹 커뮤니케이션 플러그인
갈레러 클러스터는 독점적인 그룹 통신 시스템 레이어 위에 구축되어 가상 동기화를 구현한다. 가상 동기화는 데이터 전송과 클러스터 멤버쉽 서비스를 통합한다. 가상 동기화는 일관성을 보장하지만 시간적 동기화는 보장하지 않으며, 이는 원활한 다중 마스터 작업에 필요하다. 이를 해결하기 위해 갈레라 클러스터는 자체 런타임을 구성 가능한 시간 흐름 제어를 구현한다.  
그룹 커뮤니케이션 프레임워크는 여러 소스로부터 온 메시지의 전역 순서를 제공하며, 이를 통해 GTID를 생성한다.  
갈레라 클러스터는 대칭형 무방향 그래프이며 모든 DB 노드는 TCP를 통해 연결된다. 그러나 LAN에서의 복제를 위해 UDP를 사용할 수도 있다.


### 격리 수준
#### Galera 클러스터의 노드내 격리 vs 노드 간 격리
##### 개별 클러스터 노드 내의 격리 수준
* MySQL/InnoDB에서 지원하는 모든 격리 수준
##### 클러스터 전체에 지원되는 전체 격리 수준
복제 프로토콜의 영향을 받으므로 다른 노드에서 발급된 트랜잭션은 동일한 노드에서 발급된 트랜잭션과 동일하게 격리되지 않을 수 있다.
* read-uncommited: 다른 트랜잭션이 만든 아직 커밋되지 않은 데이터의 변경사항을 볼 수 있다,
* read-commited: 커밋되지 않은 변경 내용은 다른 트랜잭션에 표시되지 않는다. 
* repeatable-read: select 쿼리가 실행된 시점의 스냅샷에 의해 격리된다.
다른 노드에서 발행된 트랜잭션의 경우, "첫 번째 커미터가 승리한다"는 규칙으로 격리가 강화되어 "업데이트 손실 이상"이 제거되는 반면, 동일한 노드에서 발행된 트랜잭션의 경우 이 규칙이 적용되지 않는다.

### 상태 전송
#### 상태 스냅샷 전송(SST)
한 노드에서 다른 노드로 전체 데이터 사본을 전송하여 노드를 프로비저닝 한다.

#### 증분 상태 전송(IST)
노드에 누락된 트랜잭션을 식별하여 노드를 프로비저닝 하고 전체 상태 대신 해당 트랜잭션만 전송한다. 다만 아래의 조건을 만족해야한다.
1. 조인어의 UUID는 그룹 상태와 동일해야 한다.
2. 누락된 모든 쓰기 집합을 기증자의 쓰기 집합 캐시에서 사용할 수 있어야 한다.

##### Write-set 캐시(GCache)
Galera 클러스터는 쓰기 집합을 write-set 또는 GCache라는 특수 캐시에 저장한다. GCache는 write-set을 위한 메모리 할당기이다. 주요 목적은 RAM의 write-set 풋프린트를 최소화하는 것이며 아래의 세가지 유형의 스토리지를 사용한다.
1. Permanent In-Memory Store: 운영 체제의 기본 메모리 할당자. 기본적으로 비활성화 되어있다.
2. Permanent Ring-Buffer File: 캐시 초기화 중 디스크에 미리 할당된다.
3. On-Demand Page Store: 필요에 따라 런타임 중 메모리 메핑된 페이지 파일. 다른 모든 저장소가 비활성화 되어있어도 적어도 하나의 페이지 파일은 디스크에 남아있다.

### 흐름제어
갈레라 클러스터는 클러스터 전체 순서에 따라 트랜잭션이 모든 노드에 복사되도록 하여 복제를 달성한다.  하지만 복제된 트랜잭션의 적용과 커밋은 비동기적으로 발생한다.  
노드는 write-set을 수신하여 GTID 순서로 정리한다.  클러스터로부터 받았지만 아직 커밋하지 않은 트랜잭션들은 수신 큐에 유지된다.  
수신 큐의 크기가 특정 값에 도달하면 노드는 흐름제어를 트리거한다. 이때 노드는 복제를 일시 중지하고 수신 큐의 트랜잭션들을 처리한다. 수신 큐의 크기를 관리하기 쉬운 크리로 줄일 때까지 작업을 수행한 후 복제를 재개한다.  

갈레라 클러스터는 여러가지 흐름 제어 구현을 통해 가상동기화가 제공하는 논리적 동기화가 아닌 시간적 동기화와 일관성을 보장한다.
##### 흐름 제어의 종류
* No Flow control: 노드가 open 또는 primary 상태에 있을 때 적용된다. 해당 노드를 클러스터의 일부로 간주하지 않으며, write-set을 복제, 적용 또는 캐시 할 수 없다.  
* Write-set Caching: 노드가 joiner 및 doner 상태에 있을 때 적용된다. 해당 노드는 이 상태에서 write-set을 적용할 수 없으며 나중에 사용할 수 있도록 캐시한다. 모든 복제를 중지하는 것 외에는 노드를 클러스터와 동기화 할 수 있는 방법이 없다. 다음 매개변수를 사용해 복제 속도를 제한하여 write-set 캐시가 구성된 크기를 초과하지 않도로 제어할 수 있다.
	* gcs.resc_q_hard_limit: 최대 쓰기 설정 캐시 크기
	* gcs.max_throttle: 클러스터에서 노드가 허용할 수 있는 정상 복제 속도에 대한 최소 분수
	* gcs.recv_q_soft_limit: 노드의 평균 복제 속도 추정치
* Catching Up: 노드가 joined 상태에 있을 때 적용된다. write-set을 적용할 수 있으며 노드가 결국 클러스터를 따라잡을 수 있음을 보장한다. 특히 write-set 캐시가 절대하지 않음을 보장한다. 따라서 클러스터 전체 복제 속도는 이 상태의 노드가 write-set을 적용할 수있는 속도에 의해 제한된다. write-set을 적용하는 것이 일반적으로 트랜잭션을 처리하는 것보다 몇 배는 더 빠르기 때문에 이 상태의 노드는 클러스터 성능에 거의 영향을 끼치지 않는다.
* Cluster Sync: 노드가 동기화 상태에 있을 때 적용된다. 슬레이브 대기열을 최소한으로 유지하려고 시도한다. 다음의 매개변수를 사용할 수 있다.
	* gcs.fc_limit: 흐름 제어가 작동하는 지점을 결정한다.
	* gcs.fc_factor: 흐름 제어가 해제되는 지점을 결정한다.

### 노드의 상태 변화  

![galera-cluster-node-state.png](https://pangorithm.github.io/MyAuxiliaryStorage/image/galera-cluster-node-state.png)  

1. 노드가 시작되고 기본 컴포넌트에 대한 연결을 설정하다.
2. 노드가 상태 전송에 성공하면 쓰기 집합을 캐시하기 시작한다.
3. 노드가 SST를 받는다. 이제 모든 클러스터 데이터를 호가보하고 캐시된 쓰기 집합을 적용하기 시작한다. 이때 노드는 흐름 제어를 활성화하여 슬레이브 대기열의 최종 감소를 보장한다.
4. 노드가 클러스터를 따라잡는 작업을 완료한다. 이제 슬레이브 대기열이 비어있고 흐름 제어가 이를 비워두돌고 설정한다. 노드는 wsrep_ready 값을 1로 설정하여, 이제 노드가 트랜잭션을 처리할 수 있다.
5. 노드가 SST 요청을 받는다. 흐름 제어가 Doner로 완화된다. 적용할 수 없는 모든 write-set을 캐시한다.
6. 노드가 Joiner 노드로 상태 전송을 완료한다.

### 노드 장애 및 복구
개별 노드가 클러스터와 연결이 끊어지면 작동하지 않는다.
#### 단일 노드 장애 감지
클러스터 관점에서 주요 수성 요소를 구성하는 노드들이 해당 노드를 더 이상 볼 수 없을 때 그 노드는 장애가 발생한 것이다. 장애가 발생한 노드의 관점에서는 해당 노드가 다운되지 않은 상태라면 주요 구성 요소와의 연결을 잃은 것이다.  
갈레라 클러스터 노드 상태를 모니터링 하려면 wsrep_local_state 상태 벼수를 풀링하거나 Notification Command 를 통해 확인할 수 있다.  
클러스터는 노드로부터 네트워크 패킷을 마지막으로 수신한 시점을 기준으로 노드의 연결을 결정한다. evs.inactive_check_period 매개변수를 사용하여 클러스터가 이를 확인하는 빈도를 구성할 수 있다. 호가인 중 클러스터가 노드로부터 네트워크 패킷을 마지막으로 수신한 이후 시간이 evs.keepalive_period 매개변수의 값보다 크면 하트비트 비콘을 방출하기 시작한다. 클러스터가 evs.suspect_timeout 매개 변수 기간 동안 노드로부터 네트워크 패킷을 계속 수신하지 못하면 노드는 의심 노드로 선언된다. 기본 구성요소의 모든 구성원이 해당 노드를 의심 노드로 간주하면 비활성 선언된다. WAN과 같이 불안정한 네트워크를 사용하는 경우 이러한 옵션의 수정을 통해 가용성과 파티션 허용 오차 사이의 절충이 필요하다.

#### 단인 노드 장애 복구
한 노드에 장애가 발생하도 다른 노드는 정상적으로 계속 작동하므로, 장애가 발생한 노드가 다시 온라인 상태가 되면 다른 노드와 자동으로 동기화 된 후 클러스터에 다시 허용된다.

### 쿼럼 구성 요소
#### 자동 장애 조치 조건
1. 단일 스위치 클러스터는 최소 3개의 노드를 사용해야 한다.
2. 스위치에 걸쳐 있는 클러스터는 최소 3개의 스위치를 사용해야 한다.
3. 네트워크에 걸쳐 있는 클러스터는 최소 3개의 네트워크를 사용해야 한다.
4. 데이터 센터에 걸쳐 있는 클러스터는 최소 3개의 데이터 센터를 사용해야 한다.

#### 분할 뇌 상태
DB 노드가 서로 독립적으로 작동하는 클러스터 장애를 스플릿 브레인 상태라고 한다. 갈레라 클러스터는 이를 방지하기 위해 동일한 크기의 두 파티션으로 분할되는 경우 (명시적으로 구성하지 않는 한) 어느 파티션도 기본 구성 요소가 되지 않는다. 노드 수가 짝수인 클러스터에서 이런 일이 발생할 위험을 최소화하기 위해 전체 노드 수를 홀수로 구성해야 한다.
`SET GLOBAL wsrep_provider_options="pc.weight=3";` 명령어를 통해 런타임중에 해당 노드의 가중치를 변경할 수 있다.

### 스트리밍 복제(Galera cluster 4.0부터 지원)
대규모 데이터 세트에 대한 장기간의 쓰기 및 변경 작업은 문제가 발생할 수 있다. 따라서 스트리밍 복제에서는 트랜잭션을 조각으로 나눈 다음 트랜잭션이 계속 진행 중인 동안 이를 인증하고 슬레이브에 복제한다. 일단 인증된 조각은 더 이상 충돌하는 트랜잭션에 의해 중단 될 수 없다.

#### 스트리밍 복제를 사용해야 하는 경우
1. 장기 실행 쓰기 트랜잭션: 노드가 트랜잭션을 커밋하는 시간이 오래 걸릴 수록 클러스터가 더 긴 트랜잭션을 클러스터에 복제하기 전에 충돌하는 트랜잭션을 적용할 가능성이 높아진다.
2. 대용량 데이터 쓰기 트랜잭션: 대규모 트랜잭션을 적용하는 동안에는 수신한 다른 트랜잭션을 커밋할 수 없으므로 전체 클러스터의 흐름 제어 스로틀링이 발생할 수 있다.
3. 핫 레코드: 동일한 테이블에서 동일한 레코드를 자주 업데이트 하는 경우 중요한 업데이트를 전체 클러스터에 강제로 복제할 수 있다.

#### 제한사항
1. 트랜잭션 중 성능: wsrep_streaming_log 테이블에 write_set을 기록하므로 복제 업데이트가 충돌하는 경우 지속성을 보장하지만, 노드에 부하를 증가시키므로 스트리밍 복제는 세션 수준에서만 사용하도록 설정한 다음, 필수적인 트랜잭션에 대해서만 사용하는 것이 권장된다.
2. 롤백 중 성능: 장기간 실행되는 쓰기 트랜잭션을 자주 롤백해야 하는 경우, 성능 문제가 될 수 있다. 따라서 일괄 처리 또는 예약 작업의 경우 스트리밍 복제 외에 이를 더 작은 트랜잭션으로 분할하는 것을 고려해야 한다.