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
* `KEYS pattern` : 레디스에서 키 목록을 확인하고자 할 때 사용한다. 패턴에는 와일드카드`*` 를 사용할 수 있지만 CPU 사용여 영향을 주기 때문에 운용 중인 경우에는 사용하지 않는 것이 바람직하다. 
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