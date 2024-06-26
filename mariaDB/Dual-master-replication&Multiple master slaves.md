### 이중 마스터 복제

이중 마스터 복제는 간단히 말해 양뱡향으로 복제하는 것이다. 이론상 양쪽의 mariaDB는 인스턴스에 동시에 쓰기를 할 수 있다. 하지만 이렇게 하면 양쪽 호스트간의 데이터가 불일치하거나 키 충돌로 인해 복제가 깨질 수 있다. 이를 피하기 위해 다음의 옵션을 추가함으로서 이런 문제를 최소화(완전 방지가 아니다.) 할 수 있다.
`auto-increment-increment=마스터의개수` ,`auto-increment-offset=디비의id` : 이 두가지 옵션을 동시에 적용함으로 인해 각각의 마스터 디비에서 insert시 pk가(자동증가는 주로 pk에 사용된다.) 충돌하지 않도록 해준다. 

이후로는 두 디비에서 서로 슬레이브 설정을 진행하면 된다.

### 자동관리
이중 마스터 복제의 목적 중 하나는 문제가 발생했을 때 양쪽 서버에 수동으로 개입하지 않고, 서버 오류가 생겼을 때 최대한 스위칭 시간을 줄이는 것이다.
이때 사용할 수 있는 방법은 아래의 목록이 있다.
1. haproxy : 강력한 로드벨런서
2. pacemaker & corosync : 가용성 높은 오픈소스 클러스터 리소스 매니저 
3. DRBD : 네트워크 대역폭 문제가 있을 경우 이중 마스터 복제를 대신할 블록복제 시스템


### mariaDB 다중 마스터 슬레이브
마리아디비는 GTID를 사용하여 하나의 슬레이브가 여러개의 마스터를 가질 수 있다.

* 주의: 각각의 마스터 서버는 유일한 server-id와 database명을 가져야한다.

1. 각각의 마스터 노드에 복제를 위한 사용자를 생성한다.
  ```sql
  create user 'replication'@'slaveDB의ip' identified by '비밀번호';
  grant replication slave on *.* to 'replication'@'slaveDB의ip'; 
  flush privileges;
  ```
2.	슬레이브 설정
```sql
마스터1,2> select @@GLOBAL.GTID_CURRENT_POS; # 마스터의 GTID 확인

슬레이브> select @@DEFAULT_MASTER_CONNECTION; # 현재는 비어있다.

SET @@DEFAULT_MASTER_CONNECTION='마스터1'; # 현재 마스터1의 복제를 작업하고 있다고 표기
select @@DEFAULT_MASTER_CONNECTION; # 이제 마스터1이 조회된다.
set global GTID_SLAVE_POS = '위에서 확인한 마스터1의 GTID';
change master '마스터1' to master_host='마스터1의 ip', master_user='replication', master_password='비밀번호', master_use_gtid=slave_pos;

#마스터2에 대해서도 위와 동일하게 수행해준다.

start all slaves; # 모든 슬레이브(복제) 시작 

show warnings; # 메세지 확인

show all slave status\G # 모든 슬레이브의 상태 확인

# 기타 명령
stop all slaves; # 모든 슬레이브 정지

reset slave '마스터이름'; 해당 복제를 재설정
```


### 복제의 제한

다중 마스터 슬레이브 또한 마스터 디비에 `replication_do_~`, `replication_ignore_~`, `replication_wild_~` 옵션을 사용하여 복제에 제한을 둘 수 있다.

