
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