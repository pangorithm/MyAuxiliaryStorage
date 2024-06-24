# performance-analysis

### slow-query
보통 `/etc/my.cnf.d/server.cnf` 파일을 통해 설정을 관리하지만 아래 명령과 같은 SQL문을 통해서도 설정 조회 또는 변경이 가능하다.
```sql
show global variables like '%slow_query%'; # 슬로우 쿼리에 관한 옵션을 조회.
set global slow_query_log=1; # 슬로우 쿼리 기록 활성화
show global variables like '%long_query%'; # 슬로우 쿼리 기준 조회
set global variables 'long_query_time'; # 슬로우 쿼리 기준 설정
```

### explain command
explain 명령 뒤에 따라오는 쿼리문에 대한 정보를 반환한다.  
기본적으로 select 쿼리에만 적용할 수 있지만 mariaDB 10.0.5 버전 부터는 update와 delete 문 또한 사용할 수 있다.  
```sql
explain select 컬럼1, 컬럼2 from 테이블 where 조건절; # explain 명령어 뒤에 분석하길 원하는 쿼리를 입력하여 실행한다.
```
실행 결과로 아래 목록과 같은 결과를 조회할 수 있다.  
id, select_type, table, type, possible_keys, key, key_len, ref, rows, Extra  
[explain 명령어 공식 문서 설명](https://mariadb.com/kb/en/explain/)  

```sql
show processlist\G # 실행 중인 쿼리(프로세스) 조회, \G를 통해 세로로 출력 설정
show explain for 쿼리id값\G # 실행 중인 쿼리에 대한 explain 명령 수행, 슬로우 쿼리 결과가 나오기 전에 분석이 가능하다는 장점이 있다.
```

### profiling
한 세션동안 리소스 사용을 보여주는 정보를 벤치마크 한다.  
블록 I/O, 컨텍스트 스위치, CPU, IPC, 메모리, 페이지 폴트, 소스, 스왑, all 의 정보를 얻을 수 있다.  
하지만 프로파일링 자체가 리소스를 차지하여 성능을 저하시키기 때분에 실서버에서 사용하는 것은 추천하지 않는다.  
```sql
set profiling=1; # 프로파일링 활성화
show profiles; # 프로파일 목록 조회
show profile 리소스 for query 쿼리id값; # 해당 쿼리의 리소스 사용 정보 프로파일
```

### 벤치마킹 툴
sysbench: fileio, cpu, memory, threads, mutex, olltp에 대한 벤치마크 기능을 제공한다.  
percona-toolkit: mysql과 mariaDB를 위한 툴 패키지 [페르코나 공식문서](https://www.percona.com/percona-toolkit)  
pt-query-digest:  mariaDB 슬로우 쿼리와 바이너리 로그파일을 분석해준다.  
pt-stalk: 특정 트리거가 발생했을 때의 데이터를 수집해준다.  
pt-summary: 시스템과 관련된 정보를 제공한다.  
pt-mysql-summary: 스키마와 데이터베이스를 포함한 MariaDB 인스턴스의 요약을 보여준다.  
pt-duplicate-key-checker: 데이터베이스 테이블에서 중복 인덱스와 외부키를 찾아준다.  
pt-index-usage: 슬로우 쿼리 로그를 이용해 어떤 인덱스가 사용되지 않는지 분석한다.  
mytop, innotop: show processlist; 쿼리의 강화버전  
mysqlsla: mysql, mariaDB의 슬로우, 바이너라, 마이크로슬로우 로그를 파싱해준다.  
