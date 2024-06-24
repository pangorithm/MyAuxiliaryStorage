# Performance Optimization

### 상태 정보 재설정

1. 동작 중 수행  
```bash
# bash 명령어로 수행
mysqladmin flush-all-statistics
# 글로벌 상태정보 간략히 확인
mysqladmin status

# sql 명령어로 수행
FLUSH STATUS;
```  
2. 설정파일 수정 후 MariaDB 인스턴스 재시작  
```bash
# 설정파일 수정
vi /etc/my.cnf # 또는 아래 명령
vi /etc/my.cnf.d/server.cnf

# 인스턴스 재시작
systemctl restart mariadb
```

### 최대 연결  
1. 설정파일에서 `max_connections` 옵션 수정 (재부팅 필요)    
2. `set global max_connections=커넥션_수` 쿼리 수행(재부팅 필요 없음)    
3. `select @@global.max_connextoins;` 쿼리를 통해 최대 커넥션 수 조회  

### 빈로그 캐시  
빈로그 캐시의 분류  
1. 트랜잭션 캐시: InnoDB/XtraDB 엔진에서 사용된다.  
2. 비트랜잭션 캐시: MyISAM 엔진에서 사용된다.  

빈로그는 복제 시스템에서 필수라고 할 수 있으며, 캐시가 제대로 사용되지 않으면 디스크가 사용되고 점차 느려지게 된다.  

`show global status like 'binlog_cache%';` : 트랜잭션 캐시 전역 변수 조회  
Binlog_cache_use: 캐시 사용량(바이트 단위)  
Binlog_cache_disk_use: 디스크 사용량(0으로 멈춰 있지 않으면 트랜잭션이 캐시에 들어가기에 너무 크다는 의미이며 성능 저하를 일으킨다.)  

### 쿼리 캐시  
이전에 수행한 select 문의 결과를 저장한다. 같은 쿼리를 사용할 경우가 생기면 이것을 재사용하고 데이터가 테이블에 업데이트나 추가되면 쿼리 캐시는 지워진다. 그리고 다음 select 쿼리 때 디스크로부터 데이터를 다시 한번 가져온다.  

만약 동시에 너무 많이 실행되면 쿼리 캐시는 병목 현상을 일으키며 여기에는 두가지 시나리오가 있다.  
1. 작은 DB, 낮은 부하 트래픽, 저비용: 쿼리 캐시는 DB 접근을 빠르게 도와준다.  
2. 거대한 DB, 높은 부하 트래픽, 비용: 쿼리 캐시를 비활성화 하고 인덱스를 최적화하며, 읽기 전용 복제를 두어 읽기 로드를 분산시키고 Memcache/Redis와 같은 향상된 캐싱 머신 사용이 권장된다.  


`show variables like 'query_cache_type';`: 쿼리 캐시 활성화 여부 조회 쿼리  
0/disabled: 모든 쿼리를 캐시하지 않는다.  
1/enabled: SQL_NO_CACHE 절을 포함한 쿼리 이외의 모든 쿼리를 캐시한다.  
2/demand: SQL_CACHE 절을 포함한 쿼리 이외의 모든 쿼리를 캐시하지 않는다.  
`show global status like 'Qc%'; :  쿼리 캐시값 조회  

`show global variables like 'query_cache%';`와 `set global 옵션=값` 쿼리를 통해 쿼리 캐시 설정 조회와 수정이 가능하다.  

### 풀 크기와 상태 정보  
`show engine innodb status \G`: 엔진 상태정보 조회  
조회 결과 버퍼 풀이 충분하지 않다면 설정파일에서 `innodb_buffer_pool_size` 옵션을 수정해서 버퍼 풀 크기를 더 크게 잡을 수 있다.(재부팅 필요)  

### 트랜잭션 커밋과 로그  
`show session variables like 'innodb_flush_log_at_trx_commit';` : 트랜잭션 안전성 설정 조회  
1: ACID를 준수하여 가장 안전하지만 fsync가 매순간 변화를 리두 로그로 쓰기에 디스크가 느릴 경우 오버헤드가 발생한다.  
2: 초당 한 번의 트랜잭션을 리두 로그로 작성하기에 덜 안전하지만 복제를 이용하고 있다면 문제되지 않는다.  
0: 가장 빠르지만 데이터를 잃을 수 있어 가장 위험하기 때문에 슬레이브에서만 사용하기를 권장한다.  

`show session variables like 'sync_binlog';` : 빈로그 동기화 설정 조회  
1: 매 sync_binlog 쓰기 이후 바이너리 로그를 디스크에 동기화 한다.  
0: 성능으로는 더 좋지만 안전성을 위해 배터리가 있는 RAID 카드를 사용할 것을 추천한다.  
