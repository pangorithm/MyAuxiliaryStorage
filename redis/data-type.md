### 주요 자료형
* String: 문자열(숫자값 포함). 간단한 키와 값의 조합
* List: 문자열 리스트
* Hash: 연관 배열이나 딕셔너리와 비슷한 개념
* Set: 복수의 값을 순서와 중복 없이 저장한다
* SortedSet: 정렬된 집합

### 보조 자료형
* BitMap: 비트맵(비트 배열)
* 지리적 공간 인덱스

### 레디스의 폭 넓은 데이터 모델 표현성
데이터베이스 번호로 식별하는 방식으로 네임스페이스 같은 데이터를 관리할 수도 있다. 하지만 기본적으로 전역에서 키와 값의 쌍으로 관리하며, RDBMS의 테이블과 같은 개념은 없다.

### 레디스 자료형과 명령어
데이터를 저장하고 가져올 때 기본적으로 데이터별 독립된 명령어를 사용한다.

### 레디스 유틸리티 명령어
* `KEYS pattern` : 레디스에서 키 목록을 확인하고자 할 때 사용한다. 패턴에는 와일드카드`*` 를 사용할 수 있지만 CPU 사용에에 영향을 주기 때문에 운용 중인 경우에는 사용하지 않는 것이 바람직하다. 
* `SCAN`: 운영환경에서 키 목록 정보를 확인하고 싶을 때는 `SCAN/SSCAN/HSCAN/ZSCAN` 명령어와 같이 SCAN 계열 명령어의 사용을 권장한다.
* `EXISTS`: 키 존재 여부를 확인할 때 사용하며 매칭된 갯수를 반환한다.
* `TYPE`: 해당 키를 사용하는 자료형과 기능을 확인할 수 있다.
* `DEL`: 키의 삭제는 모든 자료형에서 공통적으로 DEL을 사용한다. 삭제한 키의 개수를 반환한다.


## String
String이란 이름을 가지고 있지만 이진 안전 문자열이기 때문에 binary data 등 문자열 이외의 데이터도 저장할 수 있다. 숫자 값도 String형에 저장하기 때문에 String에 저장한 부동소수점을 조작하기 위한 별도의 명령어가 존재한다.
key -> value 형식으로 저장된다.
또한 비트 단위로 조작할 수 있는 Bitmap이라는 보조 자료형이 존재한다.
* 특징: 
  * 키와 값이 일대일 대응하는 가장 간단한 자료형
  * 이진 안전 문자열
* 유즈케이스
  * 캐시
  * 카운터
  * 실시간 메트릭스
레디스의 String 형은 기본적으로 512MB의 크기 제한을 갖지만 4.0.7버전 이후부터 설정 변경으로 최대 크기를 변경할 수 있다.

##### String 명령어
```redis
# 키값 가져오기
GET key

# 키에 값 저장하기(저장시간, TTL 등 옵션을 지정할 수도 있다)
SET key value

# 여러 개의 키값 가져오기
MGET key1 [key2 key3 ......]

# 여러 개의 키에 값을 저장하기
MSET key1 value1 [key2 value2 ......]

# 키에 값 덮어쓰기(키가 존재하는 경우, 키값 끝에 인수 내용을 추가한다. 존재하지 않는 경우 인수를 값으로 하는 새로운 String형 키를 만든다.)
APPEND key value

# 키의 길이 가져오기
STRLEN key

# 범위를 지정하여 키값 가져오기
GETRANGE key start end

# 범위를 지정하여 키값 저장하기
SETRANGE key offset value


### String형에서 값이 숫자인 경우만 사용할 수 있는 명령어 ###

# 값을 1만큼 증가시키기
INCR key

# 값을 지정한 정수만큼 증가시키기
INCRBY key increment

# 값을 지정한 부동소수점만큼 증가시키기
INCRBYFLOAT key increment

# 값을 1만큼 감소시키기
DECR key

# 값을 지정한 정수만큼 감소시키기
DECRBY key decrement


### 기타 명령어 ###

# TTL(초 단위)을 설정한 키값 가져오기(옵션에서 유호기간 설정 가능)
GETEX key

# 키값을 가져온 후 그 키를 삭제하기
GETDEL key

# 여러 개의 키가 존재하지 않는 것을 확인하고 값을 저장하기
MSETNX key1 value1 [key2 value2 ......]


### 옵션 ###

# 키가 존재하지 않는 경우 저장
NX

# 키가 존재하는 경우 저장
XX

# TTL(Time To Live) 설정하기
# 옵션
# 초 단위 설정
EX ttl
# 밀리 초 단위 설정
XX ttl
# 유닉스 시간을 사용하여 유효 시간을 설정(6.2 버전 이상)
EXAT/PXAT
# 키관련 TTL을 변경하지 않고 조작(6.0 버전 이상)
KEEPTTL

# 명령어
EXPIRE key ttl
```


## List
String의 리스트 프로그래밍 언어의 리스트에 가깝다.
* 특징
  * 문자열 컬렉션으로 삽입 순서를 유지한다
* 유즈케이스
  * 스택, 큐
  * 최신 데이터 캐싱
  * 로그
새로운 요소를 리스트 앞이나 뒤에 추가하는 동작은 상수 시간으로 완료되나 중간 부분으로 접근하는 것은 느리다.

##### List 명령어
```redis
# 리스트 왼쪽부터 값을 가져오고 삭제하기 (6.2 이후는 갯수 지정 가능)
LPOP ket [count]

# 리스트 왼쪽부터 값을 삽입하기
LPUSH key element [element ...]

# 리스트의 오른쪽부터 값을 가져오고 삭제하기 (6.2 이후는 갯수 지정 가능)
RPOP ket [count]

# 리스트 오른쪽부터 값을 삽입하기
RPUSH key element [element ...]

# 리스트의 왼쪽 혹은 오른쪽부터 여러 개의 값을 가져오고 삭제하기(7.0 이상)
LMPOP numkeys key [key ...] <LEFT | RIGHT> [COUNT count]

# 블록 기능을 갖춘 LMPOP (7.0 이상) (리스트에 요소가 없으면 처리를 블록하고, 순서 집합에 요소가 추가될 때까지 처리를 대기한다. 단, 최대 대기 시간은 지정한 값으로 제한된다.)
BLMPOP timeout numkeys key [key ...] <LEFT | RIGHT> [COUNT count]

# 리스트에서 지정한 인덱스에 값을 조회하기
LINDEX key index

# 리스트에서 지정한 인덱스에 값을 삽입하기. 키로 지정한 리스트에 지정한 요소의 바로 앞 혹은 뒤에 같은 요소를 삽입한다. 찾을 수 없는 경우 수행되지 않는다.
LINSERT key BEFORE|AFTER pivot element

# 리스트의 길이 가져오기
LLEN key

# 리스트에서 지정한 범위의 인덱스에 있는 값 가져오기(끝부분 부터 가져오고 싶은 경우 음수 index 지정)
LRANGE key start end

# 리스트에서 지정한 요소를 지정한 수만큼 삭제하기
LREM key count element

# 리스트에서 지정한 인덱스에 있는 값을 지정한 값으로 저장하기
LSET key index element

# 지정한 범위 인덱스에 포함된 요소로 리스트 갱신하기
LTRIM key start stop

# 리스트 중 지정한 인덱스에 있는 값 가져오기(몇번째 일치 요소인지, 몇개의 인덱스를 반환할지, 최대 탐색 요소는 몇 개인지)
LPOS key element [RANK rank] [COUNT num-matches] [MAXLEN len]

# 리스트가 있는 경우에만 값을 삽입하기
LPUSHX key element [element ...]
RPUSHX key element [element ...]

# 리스트 간 요소로 이동하기(6.2 이상)
LMOVE source destination LEFT|RIGHT LEFT|RIGHT

# 블록 기능을 갖춘 POP(리스트에 요소가 없는 경우 처리를 블록하고 리스트에 요소가 추가될 때까지 대기한다.)
BLPOP key [key ...] timeout
BRPOP key [key ...] timeout
```

## Hash
* 특징
  * 필드와 값의 쌍의 집합
  * 필드와 연결된 값으로 구성된  맵. 필드와 값 모두 문자열이다.
* 유스케이스
  * 객체 표현

##### Hash 명령어
```redis
# 키로 지정한 해시에서 지정한 필드를 삭제
HDEL key field [field ......]

# 키로 지정한 해시에 지정한 필드가 존재하면 1을 반환, 그렇지 않으면 0을 반환
HEXISTS key field

# 키로 지정한 해시에서 지정한 필드에 저장된 값을 반환
HGET key field

# 키로 지정한 해시에 포함된 모든 필드와 값 쌍을 반환
HGETALL key

# 해시에 포함된 필드 이름 무작위로 가져오기(옵션 지정시 값도 함께 가져온다)
HRANDFIELD key [count [WITHVALUES]]

# 해시에서 모든 필드 가져오기
HKEYS key

# 해시에 포함된 필드 수 가져오기
HLEN key

# 해시에 지정한 필드값 문자열의 길이 가져오기
HSTRLEN key field

# 해시에서 여러 필드와 값의 쌍 저장하기
HMSET key field value [field value ...]

# 해시에 지정한 필드값 저장하기
HSET key field value [field value ...]

# 해시에 필드가 존재하지 않는 것을 확인한 후 값 저장하기
HSETNX key field value

# 해시의 모든 필드값 가져오기
HVALS key

# 키로 지정한 해시의 필드 집합을 반복 처리하여 필드 이름과 저장된 값의 쌍 목록을 반환
HSCAN key cursor [MATCH pattern] [COUNT count]


### 값이 숫자인 경우에만 사용 가능한 명령어 ###

# 해시에 지정한 필드 값을 지정한 정수만큼 증가(키가 존재 하지 않는 경우 동작 전에 0을 저장) 
HINCRBY key field increment

# 해시에 지정한 필드값을 지정한 부동소수점 수만큼 증가(키가 존재 하지 않는 경우 동작 전에 0을 저장)
HINCREBYFLOAT key field increment
```

## Set
* 특징
  * 순서 없는 고유한 문자열 집합
  * 대상 요소 포함 여부를 확인하거나, 집합 간 합집합, 차집합 등 작업을 통해 공통 요소 또는 차이점 추출이 가능하다.
* 유즈케이스
  * 멤버십
  * 태그 관리

##### Set 명령어
```redis
# 집합에 하나 이상의 멤버 추가하기
SADD key member [member ...]

# 집합에 포함된 멤버의 수 가져오기
SCARD key

# 집합에 지정한 멤버가 포함되었는지 판단하기
SISMEMBER key member

# 집합에 지정한 여러 멤버가 포함되어 있는지 판단하기
SMISMEMBER key member [member ...]

# 집합에 포함된 모든 멤버 가져오기
SMEMBERS key

# 집합에 포함된 멤버를 무작위로 가져오기
SPOP key [count]

# 멤버를 삭제하지 않고 무작위로 멤버 추출하기
SRANDMEMBER key [count]

# 집합에서 하나 이상의 멤버를 삭제하기
SREM key member [member ...]

# 키로 지정한 집합에 각 멤버를 반복 처리하여 멤버 목록을 반환한다.
SSCAN key member [member ...]

# 집합 간 멤버 이동하기
SMOVE source destination member


### Set에서 사용 가능한 집합 연산 명령어 ###

# 하나 이상의 집합들의 차집합 가져오기
SDIFF key [key ...]

# 집합 간 차집합을 가져오고 저장하기
SDIFFSTORE destination key [key ...]

# 하나 이상의 집합들의 교집합 가져오기
SINTER key [key ...]

# 집합 간 교집합을 가져오고 저장하기
SINTERSTORE destination key [key ...]

# 하나 이상의 집합들의 교집합의 멤버 수를 반환
SINTERCARD numkeys key [key ...] [LIMIT limit]

# 하나 이상의 집합들의 합집합 가져오기
SUNION key [key ...]

# 집합 간의 합집합을 가져오고 저장하기
SUNIONSTORE destination key [key ...]
```

## Sorted Set
* 특징
  * 순서가 있는 고유한 문자열 집합
  * Set와 유사하지만 모든 요소에는 점수라는 부동소수점을 가진다. 요소는 항상 점수를 통해 정렬되며 Set와는 다른 특정 범위의 요소를 추출할 수 있다.
* 유즈케이스
  * 랭킹
  * 활동 피드

##### Sorted Set 명령어
```redis
# 순서 집합에 하나 이상의 점수와 멤버 쌍 추가하기(GT 옵션은 큰 값으로 갱신, LT 옵션은 작은 값으로 갱신한다.)
ZADD key [NX | XX] [GT | LT] [CH] [INCR] score member [score member ...]

# 순서 집합에 포함된 멤버 수 가져오기
ZCARD key

# 순서 집합에 지정한 멤버의 점수 순위를 오름차순으로 가져오기
ZRANK key member

# 순서 집합에 지정한 멤버의 점수 순위를 높은 순서대로 가져오기
ZREVRANK key member

# 순서 집합에 지정한 멤버의 점수 범위에 있는 멤버를 오름 차순으로 가져오기
ZRANGE key min max [BYSCORE | BYLEX] [REV] [LIMIT offset count] [WITHSCORES]

# 순서 집합에 지정한 멤버의 점수 범위에 있는 멤버 목록을 오른차순으로 가져오고 저장하기
ZRANGESTORE dst src min max [BYSCORE | BYLEX] [REV] [LIMIT offset count]

# 순서 집합에 지정한 멤버 삭제하기
ZREM key member [member ...]

# 순서 집합에서 지정한 점수 범위에 있는 멤버 수 가져오기
ZCOUNT key min max

# 순서 집합에서 점수가 최대인 멤버 삭제하고 가져오기
ZPOPMAX key [count]

# 순서 집합에서 점수가 최소인 멤버 삭제하고 가져오기
ZPOPMIN key [count]

# 순서 집합에서 지정한 멤버 점수 가져오기
ZCORE key member

# 순서 집합에서 여러 멤버 점수 가져오기
ZMSCORE key member [member ...]

# 순서 집합에서 반복 처리하여 멤버 목록 가져오기
ZSCAN key cursor [MATCH pattern] [COUNT count]

# 순서 집합에서 점수가 최대 혹은 최소인 여러 멤버를 삭제하고 가져오기
ZMPOP numkeys key [key ...] <MIN|MAX> [COUNT count]

# 블록 기능을 갖춘 ZMPOP(순서 집합에 요소가 없는 경우 요소가 추가될 때까지 처리를 대기한다.)
BZMPOP timeout numkeys key [key ...] <MIN|MAX> [COUNT count]


### Sorted Set에서 사용 가능한 집합 연산 명령어 ###
### Set에서 사용 가능한 집합 연산 명령어에서 접두어만 Z로 바꾸면 된다 ###


### 기타 명령어 ###

# 순서 집합에서 지정한 사전 순 범위에 있는 멤버 모두 삭제하기
ZREMRANGEBYLEX key min max

# 순서 집합에서 지정한 순위의 범위에 있는 멤버 모두 삭제하기
ZREMRANGEBYRANK key start stop

# 순서 집합에서 지정한 점수 범위에 있는 멤버를 모두 삭제하기 
ZREMRANGEBYSCORE key min max

# 순서 집합에서 지정한 사전 순 범위에 있는 멤버 수 가져오기
ZLEXCOUNT key [count [WITHSCORES]]

# 순서 집합의 점수를 지정한 정수만큼 증가시키기
ZINCRBY key increment member

# 블록 기능을 갖춘 ZPOPMIN
BZPOPMIN key [key ...] timeout

# 블록 기능을 갖춘 ZPOPMAX
BZPOPMAX key [key ...] timeout
```

## bitmap
이 보조 자료형은 다섯가지 자료형 내부에서 모두 사용할 수 있다. 비트맵은 비트 배열이라고도 불리며 데이터 모델을 비트의 존재 여부나 그 위치에 따라 표현하여 메모리를 절약할 수 있다. 비트맵은 독립적인 자료형처럼 보이나 실제로는 String 형으로 정의되어 있으며, 비트 연산 등 특정 용도에 특화된 보조 자료형이다.

* 특징
  * 비트열 작업에 사용한다.
  * 개별 비트를 설정 또는 초기화하거나 비트 수를 세거나 처음 0 또는 1로 저장된 비트 위치 검색, 연산 등을 처리할 수 있다.
  * 비트맵을 여러 키로 분해하여 샤딩하기 용이하다.
* 유즈케이스
  * 모든 종류의 실시간 분석
  * 객체 ID 관련 이진 정보 저장
비트맵은 메모리 공간을 효율적으로 다루는 장점이 있으나, 처리하는 대상의 숫자가 적다면 희소 상태가 되기 때문에 메모리 공간 관리 측면에서 비효율적일 수 있다. 또한 기존 비트맵의 크기가 작은 상태에서 큰 오프셋으로 비트를 설정하는 경우, 비트맵에 메모리가 추가로 할당되어 확장되고, 나머지 부분은 0으로 채워진다. 이 과정에서 레디스 서버가 블록될 수도 있다.

##### bitmap 명령어
```redis
# 지정한 오프셋의 비트값 가져오기
GETBIT key offset

# 지정한 오프셋의 비트값 설정하기
SETBIT key offset value

# 비트맵의 비트 수 가져오기
BITCOUNT key [start end [BYTE | BIT]]

# 지정한 비트의 처음 위치 가져오기
BITPOS key bit [start [end [BYTE | BIT]]]

# 여러 비트 필드 동시에 조작하기
BITFIELD key GET encoding offset | [OVERFLOW WRAP | SAT | FAIL] SET encoding offset value | INCRBY encoding offset increment [GET encoding offset | [OVERFLOW WRAP | SAT | FAIL] SET encoding offset value | INCRBY encoding offset increment ...]
# BITFIELD 하위 명령어
GET type offset
SET type offset value
INCRBY type offset increment
OVERFLOW [WRAP|SAT|FAIL]

# 읽기 전용 BITFIELD 명령어
BITFIELD_RO key GET type offset

# 비트연산 명령어
BITTOP [AND | OR | XOR | NOT]
 
```


## Geohash
경도와 위도 두 개의 좌표를 하나의 문자열로 합친 것으로 그리드 내의 영역을 표시한다.

##### Geohash 명령어
```redis
# 위치 정보 추가하기
GEOADD key [NS|XX] [CH] longitude latitude member [longitude latitude member ...]

# 위치 정보 Geohash 값 가져오기
GEOHASH key member [member ...]

# 경도 위도 값 가져오기
GEOPOS key member [member ...]

# 멤버 간 거리 가져오기(미터 단위)
GEODIST key member1 member2 [M | KM | FT | MI]

# 특정 경도 위도 지점에서 지정한 조건에 있는 멤버 목록 가져오기
GEOSEARCH key FROMMEMBER member| FROMLONLAT logitude latitude BYRADIUS radius M | KM | FT | MI | MYBOX width height M | KM | FT | MI | [ASC | DESC] [COUNT count [ANY]] [WITHDIST] [WITHHASH]

# 특정 경도 위도 지점에서 지정한 조건에 있는 멤버 목록 가져오고 저장하기
GEOSEARCHSTORE destination source FROMMEMBER member | FROMLONLAT longitude latitude BYRADIUS M | KM | FT | MI | MYBOX width height M | KM | FT | MI | [ASC | DESC] [COUNT count [ANY]] [STOREDIST]
```