# 스파이더 엔진: 데이터 샤딩

### 스파이더 엔진의 기능
* 테이블 링크: 여러개의 MariaDB 서버의 테이블에 싱글 테이블 인스턴스 처럼 접근 가능
* XA 트랜잭션: 여러 개의 MariaDB 인스턴스에 걸쳐 데이터를 동기화하거나 업데이트 하기 위해 사용
* 테이블 파티셔닝: 여러 개의 서버에 파티션을 만들 수 있다.
* 속도: 성능 향상을 위해 하나가 아닌 여러개의 서버를 사용한다.

#### 스파이더 기반 인프라 구조
* 데이터 노드: 데이터 저장 노드와 같이 행동한다.
* 스파이더 노드: 사용자 접근의 입구에 놓여있다.(로드 밸런스, 장애 극복 등)
* 모니터링 노드: 높은 가용성을 위해 데이터 노드를 감시한다.

### 스파이더 설정
1. 스파이더 엔진은 mariaDB 10에 기본적으로 포함되어 있지만 활성화 하기 위해서는 새로운 테이블을 생성하고 선행작업이 필요하다. 이는 sql 명령어로 수행 가능하다. [마리아디비 공식 문서](https://mariadb.com/kb/en/spider-installation/)
2. 샤드 생성
	1. 스파이더 사용자 생성	   
	   ```sql
	   create user 'spider'@'ip주소' identified by '비밀번호';
	   grant all privileges on *.* to 'spider'@'ip주소' identified by '비밀번호';
	   flush privileges;
	   ```
	2.  모든 서버에 백엔드 디비 생성 및 테이블 삽입
	   ```sql
	   create database backend;
	   create table backend.sbtest (
	   id int(10) unsigned not null auto_incresment,
	   k int(10) unsigned not null default '0',
	   c char(120) not null default '',
	   pwd char(60) not null default '',
	   primary key (id),
	   key k (k)
	   ) engine=InnoDB; # 백엔드에 어떠한 종류의 엔진이라도 사용 가능하다.
	   # 스파이더 서버에만 스파이더 엔진을 사용하면 된다
```
	3.  스파이더 서버에 백엔드 서버 정보 입력
	   ```sql
	   create server backend1
	   foreign data wrapper mysql
	   options(
	     host '백엔드1서버ip',
	     database '디비명',
	     user 'spider',
	     passwd '비밀번호',
	     port 3306
	   );
	   
	   create server backend2
	   foreign data wrapper mysql
	   options(
	     host '백엔드2서버ip',
	     database '디비명',
	     user 'spider',
	     passwd '비밀번호',
	     port 3306
	   );
```
	4. 샤딩 정보 확인
	   `select DB, Server_name, Host, Username, Password, Port from mysql.servers;`
	5. 스파이더 디비에 백엔드 디비와 같은 테이블을 스파이더 엔진으로 생성
	   ```sql
	   create table backend.sbtest (
	   id int(10) unsigned not null auto_incresment,
	   k int(10) unsigned not null default '0',
	   c char(120) not null default '',
	   pwd char(60) not null default '',
	   primary key (id),
	   key k (k)
	   ) engine=spider comment='database "backend", table "sbtest"' # 스파이더 엔진 사용
	   partition by key (id) # 테이블의 id 칼럼으로 샤딩하도록 스파이더에 요청
	   (
	   partition pt1 comment = 'srv "백엔드1"',
	   partition pt2 comment = 'srv "백엔드2'
	   );
```
	6. 스파이더 테이블 설정 확인
	   ```sql
	   select
	   db_name, table_name, link_id, server, tgt_table_name, link_status 
	   from mysql.spider_tables;
```
	7. 스파이더 모니터링 설정
	   ```sql
   	   create server mon
	   foreign data wrapper mysql
	   options(
	     host '모니터링서버ip',
	     database '디비명',
	     user 'spider',
	     passwd '비밀번호',
	     port 3306
	   );
```
	8.  모니터링 활성화
	   ```sql
	   insert into mysql.spider_link_mon_servers values
	   ('%','%','%',3306,'mon',null,null,null,null,null,null,null,null,null,null,null,0,null,null);
```


### 성능 튜닝
#### bgs모드
기본적으로 메모리 사용을 최적화 하기 위해 비활성화 되어 있지만 RAM이 충분할 경우 수정 가능하다. 기본 값을 바꾸면 플랜이 여러 파티션을 prune 할 때 읽기 쿼리를 병렬로 수행할 수 있다.
`set spider_bgs_mode=2` : 활성화 명령, 설정파일을 통해서도 활성화 가능하다.
#### 연결 재활용 모드
`spider_conn_recycle_mode=1`: 모든 세션에서 스파이더가 재활요하도록 설정, 설정파일을 통해서도 활성화 가능하다.

#### 상태정보 테이블
독립적인 저장 엔진을 활성화, 옵티마이저에 의해 10%까지 추가적인 성능향상을 시킬 수 있다.
`set global use_stat_tables='preferably';` : 활성화 명령, 설정파일을 통해서도 활성화 가능하다.

#### 원격 SQL 로그
원격의 백엔드로 로그를 전송한다. 하지만 성능에 악영향을 끼치므로 비활성화를 권장한다.
`spider_remote_sql_log_off=1`: 비활성화 명령, 설정파일을 통해서도 활성화 가능하다.

#### 샤드의 개수
샤드가 많을수록 병렬 처리가 가능해지기 때문에 성능상 이점을 가질 수 있다.
