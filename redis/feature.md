
# 레디스 지원 기능
## Pub/Sub
구독자는 관심 있는 주제에 대해 하나 이상의 채널을 구독할수 있으며 발행자는 특정 채널을 지정하여 이 채널을 통해 메시지를 발행한다. 발행자가 발행한 메시지는 해당 채널을 구독하고 있는 모든 구독자가 메시지 형태로 받아볼 수 있다.

##### Pub/Sub 기능에서 사용 가능한 명령어
```redis
# 발행자가 메시지 발행하기
PUBLISH channel message

# 구독자가 채널 구독하기
SUBSCRIBE channel [channel ...]

# 구독자가 채널 구독 종료하기
UNSUBSCRIBE [channel [channel ...]]

# 구독자가 패턴으로 지정한 채널 구독하기
PSUBSCRIBE pattern [pattern ...]

# 구독자가 패턴으로 지정한 채널 구독 종료하기
PUNSUBSCRIBE [pattern [pattern ...]]

# 지정한 샤드 채널에 발행자가 메시지를 발행하기
SPUBLISH shardchannel message

# 지정한 샤드 채널에 구독자가 채널 구독하기
SSUBSCRIBE shardchannel [shardchannel ...] 

# 지정한 샤드 채널에 구독자가 구독 종료하기
SUNSUBSCRIBE [channel [channel ...]]

# Pub/Sub 기능 상태 조사하기
PUBSUB [CHANNELS | NUMSUB | NUMPAT | SHARDCHANNELS | SHARDNUMSUB]
```

##### 주의사항
구독자가 채널을 구독할 때, 과거에 그 채널에 발행한 메시지는 받을 수 없다.

##### Shared Pub/Sub 기능 (7.0 이상)
레디스 7.0 미만에서 레디스 클러스터를 사용하는 경우, 노드에 발행된 메시지는 다른 모든 노드에 전달되므로 모든 노드에서 구독된다. 따라서 높은 처리량이 필요한 Pub/Sub 워크로드를 처리할 때, 클러스터 버스의 버퍼 관리로 인해 메인 스레드 CPU 부하가 높아질 수 있다. 레디스 7.0 이후에는 Shared Pub/Sub 기능이 도입되어 레디스 클러스터를 사용하는 경우, 메시지 반영은 샤드 내로 제한된다. 키가 특정 슬롯에 할당되는 것 처럼 샤드 채널도 슬롯에 할당된다. 이렇게 함으로써 클러스터 버스 내 데이터 양이 제한되고, 샤드를 추가하여 수평 방향으로 쉽게 확장할 수 있다.

## HyperLogLog
고유한 수를 효과적으로 계산할 수 있는 확률적 계산 방법. 오차가 다소 있을 수 있지만, 메모리 공간을 효율적으로 사용할 수 있다. 대략적인 값을 알기만 하면 괜찮을 경우 활용할 수 있다. 레디스 키로 참조하는 HyperLogLog 자료구조에 요소를 추가해나가는 방식으로 사용한다.

* 특징 
  * 고유한 수를 계산하는 확률적 계산 방법에 사용한다. 메모리 공간 효율성을 위해 오차는 1% 미만으로 포함한다.
* 장점
  * 계산해야 하는 수에 비례하지 않는 일정량의 메모리만 필요하다.
  * 필요한 메모리는 많아야 12KB 정도로, 요소 개수가 매우 적을 때는 메모리가 더 적을 수 있다.
* 단점
  * 표준 오차 1% 미만의 계산을 할 때 오차가 발생한다.
* 유즈케이스
  * 고유 방문자 수

##### HyperLogLog 명령어
```redis
# HyperLogLog에 값 추가하기
PFADD key element [element ...]

# HyperLogLog로 카운트한 값의 근사치 가져오기
PFCOUNT key [key ...]

# HyperLogLog 자료구조 통합하기
PFMERGE destkey sourcekey [sourcekey ...]

# HyperLogLog 기능을 디버그하기(레디스 자체 개발과 관련된 경우가 아니면 사용할일 없음)
PFDEBUG subcommand key
```


## Redis Stream
강력한 메시지 처리 기능을 가진 추가형 자료구조를 사용한다. 로그라고 하는 연속한 순서로 끝에 불변하는 레코드를 추가하는 자료구조가 있다.
* 특징 
  * 데이터가 연속으로 대량 발생하는 상황에서 데이터를 추가할 때 특화된 자료구조. 기존 데이터를 변경하지 않고 추가할 수 있다.
  * 각 엔트리에 여러 필드를 갖도록 구조화된 데이터를 가질 수 있다.
  * 과거 데이터도 유지할 수 있다
  * 엔트리 ID 및 유닉스 시간으로 엔트리 조회 범위를 지정할 수 있다.
  * 레디스만의 기능으로 Consumer Grop 기능을 지원하여 그룹 내 클라이언트 협동을 할 수 있다.
    * Consumer가 지정한 엔트리를 처리할 수 있다.
    *  Consumer가 지정한 엔트리가 제대로 처리되었는지 확인할 수 있다.
    * 제대로 처리되지 않은 경우, 다른 Consumer에 할당하여 처리를 계속할 수 있다.
    * Kafka와 비슷한 개념이지만 Kafka가 메시지를 읽는 소스에서 파티션으로 분할하는 반면, 레디스는 미리 준비된 Consumer의 상태를 기반으로 논리적으로 파티션을 구현한다. Kafka와 비슷하게 구현하기 위해서는 키를 여러 개로 분할하여 유사한 파티셔닝을 할 수 있다.
* 유즈케이스
  * 메시징 시스템
  * 시계열 데이터
  * Consumer Group

메시징 시스템 쿼리 모드는 스트림에 추가된 메시지와 같은 내용을 여러 Consumer에 전달할 수 있다.  
시계열 데이터 쿼리 모드에서는 시간 범위를 지정하거나 커서로 과거 이력에 따라 메시지를 추적할 수 있다.  
Consumer Group 쿼리 모드에서는 Consumer Group 내 하나 이상의 Consumer에서 스트림 내 엔트리를 분산시키면서 함께 처리할 수 있다. Consumer Group을 하나의 단위로, 동일 그룹 내 하나 이상의 Consumer는 파티션으로 나뉘어 있다. 메시지 처리는 각 파티션 내 Consumer로만 할 수 있다. 이를 통해 여러 Comsumer에서 작업을 분할하여 처리할 수 있으며, Consumer는 스케일 아웃 구성이 가능하도록 설계되었다. 또한 레디스 서버 내에서 여러 Comsumer의 엔트리 처리 상황을 관리할 수 있다. 하나의 스트림 내 여러 Comsumer Group을 만들 수도 있고, 같은 스트림의 데이터를 각기 다른 진행 상황으로 처리하는 것이 가능하다.

##### 쿼리 모드와 상관없이 사용 가능한 명령어
```redis
# 스트림에 엔트리 추가하기
XADD key [NOMKSTREAM] [<MAXLEN | MINID> [ = | ~] threshold [LIMIT count]] <* | id> field value [field value ...]

# 스트림에서 지정한 범위에 있는 엔트리 ID를 오름차순으로 가져오기
XRANGE key start end [COUNT count]
# 스트림에서 지정한 범위에 있는 엔트리 ID를 내림차순으로 가져오기
XREVRANGE key start end [COUNT count]

# 지정한 하나 이상의 키 스트림에서 지정한 엔트리ID 이후의 엔트리 데이터 가져오기
XREAD [COUNT count] [BLOCK milliseconds] STREAM key [key ...] ID [ID ...]

# 스트림 내 엔트리 수 가져오기
XLEN key

# 스토리 내 엔트리 삭제하기
XDEL key ID [ID ...]

# 지정한 키 스트림 내에 임계값을 지정하여 엔트리를 삭제하고 엔트리 수를 제한하기
XTRIM key MAXLEN | MINID [= | ~] threshold [LIMIT count]

# 스트림이나 Consumer Group의 상세 정보 가져오기
XINFO CONSUMERS key groupname
XINFO GROUPS key
XINFO STREAM key [FUILL [COUNT count]]
XINFO HELP

```

* 특수 ID
  * `-` : 모든 스트림 내에서 가장 작은 엔트리 ID
  * `+` : 모든 스트림 내에서 가장 큰 엔트리 ID
  * `$` : 명령어를 실행한 후 스트림에 가장 최근에 도착한 메시지의 엔트리 ID
  * `>` : XREADGROUP 명령어. Consumer Group 내에 어떤 Consumer에게도 전달되지 않은 메시지 ID
  * `*` : XADD 명령어로 추가한 새로운 엔트리에 부여된 신규 ID

##### Consumer Group 쿼리 모드 명령어
```redis
### Consumer Group 관리하기
# 지정한 그룹명으로 Consumer Group 생성
XGROUP CREATE key groupname id | $ [MKSTREAM] [ENTRIESREAD entries_read]
# 지정한 Consumer Group 이름으로 Consumer 생성
XGROUP CREATECONSUMER key groupname consumername
# Consumer Group 삭제
XGROUP DESTROY key groupname
# Consumer Group이 읽기 시작할 스트림 ID를 수종으로 설정하기
XGROUP SETID key groupname id | $ [ENTRIESREAD entries_read]
# 도움말
XGROUP HELP

# Consumer Group 내 각 Consumer가 공동으로 각 엔트리 가져오기
XREADGROUP GROUP group consumer [COUNT count] [BLOCK milliseconds] [NOACK] STREAMS key [key ...] id [id ...]

# Consumer Group 내에서 엔트리 처리 완료 보고하기
XACK key group ID [ID ...]

# Consumer Group 내에서 엔트리 소유권을 다른 Consumer로 변경하기
XCLAIM key group consumer min-idle-time id [id ...] [IDLE ms] [TIME unix-time-milliseconds] [RETRYCOUNT count] [FORCE] [JUSTID]

# Consumer Group 내에서 보류 상태인 엔트리 수 가져오기
XPENDIMG key group [[IDLE min-idle-time] start end count [consumer]]

# Consumer Group 내에 시간 만료된 엔트리 소유권을 자동적으로 다른 Consumer로 변경하기
XAUTOCLAIM key group consumer min-idle-time start [COUNT count] [JUSTID]
```
   