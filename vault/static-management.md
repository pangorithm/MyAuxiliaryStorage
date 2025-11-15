# HashiCorp Vault 운영 가이드: 정적 비밀(비밀번호) 관리를 위한 A-to-Z

## I. 서론: 클러스터 구성 완료 후, 정적 비밀 관리 운영의 시작

HashiCorp Vault 클러스터 구성을 성공적으로 완료한 것은 Vault 도입 여정의 첫 번째 주요 마일스톤을 달성한 것입니다. 이제 인프라 구축(Setup) 단계를 넘어, Vault의 핵심 가치를 실현하는 운영(Operation) 단계로 전환할 시점입니다.

운영의 첫 번째 과제는 조직 내에 산재한 '비밀번호', 즉 정적 비밀(Static Secrets)을 Vault로 중앙화하고 안전하게 관리하는 것입니다. Vault의 맥락에서 '비밀번호' 또는 '정적 비밀'이란, API 키, 데이터베이스의 고정된 사용자/비밀번호, 서비스 계정의 Private Key, TLS 인증서와 같이 생성된 후 수동으로 변경되기 전까지 그 값이 변하지 않는 모든 민감한 데이터를 의미합니다.   

이러한 정적 비밀을 관리하기 위해 Vault는 **KV(Key/Value) Secrets Engine**이라는 핵심 구성 요소를 제공합니다. 이는 Vault에서 가장 기본적이고 널리 사용되는 엔진으로, 임의의 데이터를 Key-Value 쌍으로 저장하는, 암호화된 고성능 저장소(마치 암호화된 Redis나 Memcached)처럼 작동합니다.   

KV 엔진은 Vault의 모든 기능을 마스터하기 위한 필수적인 "디딤돌(stepping stone)" 역할을 합니다. 정적 비밀 관리(KV)는 그 자체로 완벽한 해결책이라기보다는, Vault 운영의 기본기를 다지는 과정입니다. 정적 비밀은 "이 비밀번호를 누가, 언제 생성했는가?", "이 비밀번호가 유출되었을 때 즉시 폐기(revoke)할 수 있는가?", "현재 이 비밀번호를 몇 개의 애플리케이션이 사용하고 있는가?"와 같은 근본적인 질문에 대한 답을 주지 못합니다.   

본 보고서는 먼저 이 KV 엔진을 완벽하게 마스터하여 정적 비밀을 올바르게 관리하는 방법론에 집중합니다. 이 기본기가 확립된 후, Vault의 진정한 강점인 '동적 비밀'(Dynamic Secrets) - 예를 들어, 요청 시마다 고유한 DB 자격 증명을 생성하고 만료시키는 Database Secrets Engine  또는 임시 클라우드 접근 키를 발급하는 AWS Secrets Engine  - 로 확장해 나가는 것이 올바른 수순입니다.   

## II. 1단계: 비밀번호 금고(KV Secrets Engine) 활성화 및 구성

### Secrets Engine의 핵심 아키텍처

Vault는 단일 목적의 도구가 아니라, 다양한 유형의 비밀을 관리하기 위한 '플러그인' 기반 아키텍처를 채택하고 있습니다. 이러한 플러그인을 **Secrets Engine**이라고 부릅니다. KV 엔진은 여러 Secrets Engine 중 하나일 뿐이며, 그 외에도 데이터베이스, PKI, AWS, SSH 등 다양한 엔진이 존재합니다.   

Secrets Engine은 Vault 내의 특정 경로(path)에 '활성화(enable)' 또는 '마운트(mount)'되어야 작동합니다. 예를 들어, `vault secrets enable -path=kv-prod kv` 명령은 `kv-prod/`라는 격리된 경로에 KV 엔진의 새 인스턴스를 활성화합니다.   

여기서 Vault의 강력한 보안 모델이 작동합니다. 각 Secrets Engine 인스턴스가 활성화될 때마다 고유한 UUID가 생성되며, 모든 데이터는 해당 UUID를 접두사로 하여 물리적 스토리지에 격리(chroot)됩니다. 이는 하나의 엔진 인스턴스(예: `kv-prod/`)가 다른 인스턴스(예: `kv-dev/`)의 데이터에 `../` 같은 상대 경로로 접근하는 것을 원천적으로 불가능하게 만듭니다. 즉, `kv-prod/` 엔진이 해킹당하더라도 `kv-dev/`의 비밀은 안전하게 보호됩니다.   

### 핵심 결정: KV v1과 KV v2 중 무엇을 선택해야 하는가?

정적 비밀 관리를 시작하기 전, 운영자는 두 가지 버전의 KV 엔진 중 하나를 선택해야 하는 첫 번째 아키텍처 결정을 내려야 합니다: **v1**과 **v2**입니다.   

- **KV v1 (Non-versioned):**
    
    - 가장 고전적이고 단순한 Key-Value 저장소입니다.   
        
    - `put` (쓰기) 작업 시, 기존 값을 즉시 덮어씁니다. 이전 값은 영구적으로 사라지며 복구할 수 없습니다.   
        
    - **장점:** 추가 메타데이터나 히스토리를 저장하지 않으므로 스토리지 공간을 덜 차지하며, 잠금(locking) 메커니즘이 없어 v2 대비 약간의 런타임 성능 이점이 있습니다.   
        
- **KV v2 (Versioned):**
    
    - 현대의 Vault 운영을 위한 표준입니다. v1과 달리, 모든 `put` 작업은 기존 값을 덮어쓰는 대신 새로운 '버전'을 생성합니다. 기본적으로 최근 10개의 버전을 유지합니다 (설정 가능).   
        
    - **핵심 기능 1: 버전 관리(Versioning):** 실수로 비밀을 덮어쓰거나 잘못된 값으로 변경했을 때, 이전 버전의 비밀 값을 즉시 조회하거나 롤백(rollback)할 수 있습니다.   
        
    - **핵심 기능 2: 소프트 삭제(Soft Delete):** `delete` 명령이 데이터를 즉시 파기하지 않습니다. 대신 해당 버전을 '삭제됨'으로 표시(marking)하고 `deletion_time`을 기록합니다. 이는 실수로 삭제한 비밀을 복원(`undelete`)할 수 있는 강력한 안전장치입니다.   
        
    - **핵심 기능 3: Check-and-Set (CAS):** 여러 자동화 스크립트가 동시에 동일한 비밀을 수정하려 할 때, "내가 읽은 버전이 최신 버전일 때만 쓰기"를 수행하여 의도치 않은 데이터 덮어쓰기를 방지합니다. 이는 CI/CD 파이프라인의 데이터 무결성을 보장하는 데 매우 중요합니다.   
        

### KV v1 vs. KV v2 기능 비교 매트릭스

다음 표는 운영자의 결정을 돕기 위해 두 버전의 핵심 기능 차이를 요약한 것입니다.   

|기능 (Feature)|KV v1 (Non-versioned)|KV v2 (Versioned)|운영상의 의미|
|---|---|---|---|
|**버전 관리**|없음 (항상 덮어쓰기)|**지원** (버전 기록 유지)|실수로 덮어쓴 비밀을 롤백할 수 있습니다.|
|**삭제 동작**|**영구 삭제** (즉시 파기)|**소프트 삭제** (복구 가능)|'oops' 삭제로부터 복구할 수 있는 안전장치입니다.|
|**복원 (Undelete)**|불가능|**지원** (`undelete` 명령어)|감사(Audit) 및 재해 복구에 절대적으로 유리합니다.|
|**Check-and-Set (CAS)**|미지원|**지원**|자동화된 CI/CD 파이프라인의 동시성(concurrency) 충돌을 방지합니다.|
|**API 경로 (중요)**|`MOUNT_PATH/SECRET_PATH`|`MOUNT_PATH/data/SECRET_PATH`|**v2는 `data`라는 하위 경로가 추가됩니다.**|

### 전문가의 강력한 권고: 지금 즉시 KV v2를 선택하십시오

사용자는 _이제 막_ 클러스터를 구성한 '그린필드(Greenfield)' 환경에 있습니다. 이는 막대한 기술 부채를 피할 수 있는 절호의 기회입니다. **새로운 모든 Vault 배포는 반드시 KV v2로 시작해야 합니다.**

v1으로 시작했을 때 발생하는 운영상의 고통은 v1에서 v2로의 _업그레이드_ 과정에 있습니다.   

1. **서비스 중단(Downtime):** 기존 v1 엔진을 v2로 업그레이드하는 `vault kv enable-versioning <path>` 명령을 실행하면, Vault가 기존 데이터를 v2 구조로 마이그레이션하는 동안 해당 Secrets Engine은 일시적으로 **사용 불가능(unavailable)** 상태가 됩니다. 이는 운영 중인 서비스의 즉각적인 다운타임을 의미합니다.   
    
2. **치명적인 API 경로 변경:** 업그레이드 과정에서 가장 치명적인 문제는 API 경로가 `MOUNT_PATH/SECRET_PATH`에서 `MOUNT_PATH/data/SECRET_PATH`로 변경된다는 점입니다.   
    
3. **전방위적 리팩토링 필요:** 이 경로 변경은 해당 Secrets Engine에 접근하는 **모든 애플리케이션의 코드**와 **모든 Vault 접근 정책(Policy)**을 수정해야 함을 의미합니다. 이는 "엄청난 리팩토링 및 재설계 노력"을 요구하며 , 수백 개의 마이크로서비스가 얽혀있다면 사실상 불가능에 가까운 작업이 될 수 있습니다.   
    

결론적으로, 지금 v2를 선택하는 것은 미래에 발생할 막대한 운영 비용과 서비스 중단 위험을 원천적으로 차단하는, 클러스터 구축 직후에 내릴 수 있는 가장 현명하고 중요한 아키텍처 결정입니다.   

### 실행: CLI를 통한 KV v2 Secrets Engine 활성화

먼저, `vault secrets list` 명령어로 현재 활성화된 엔진을 확인합니다. (기본적으로 `cubbyhole/`, `identity/`, `sys/` 등이 있습니다).   

다음 명령어를 사용하여 `kv-v2/`라는 경로에 버전 2의 KV 엔진을 활성화합니다. (기본 경로인 `secret/` 대신, 용도를 명확히 알 수 있는 `kv-v2/`, `kv-prod/`, `kv-dev/` 등 명시적인 경로를 사용하는 것이 모범 사례입니다 ).   


```
# -path=kv-v2 : 'kv-v2/' 라는 명시적 경로에 마운트합니다.
# -version=2  : KV 버전 2를 사용하도록 지정합니다.
# kv          : 활성화할 엔진의 타입(Type)입니다.

$ vault secrets enable -path=kv-v2 -version=2 kv
Success! Enabled the kv secrets engine at: kv-v2/
```

## III. 2단계: 비밀번호 생명주기 관리 (CRUDL 실무)

KV v2 엔진이 활성화되었으므로, 이제 비밀번호의 전체 생명주기(생성, 조회, 목록화, 삭제)를 관리하는 실무 명령어를 마스터해야 합니다.

### 비밀번호 생성 및 업데이트 (Create/Update: `vault kv put`)

`vault kv put` 명령어는 지정된 경로에 새로운 버전의 비밀을 작성합니다. 만약 해당 경로에 비밀이 이미 존재한다면, 새로운 버전을 생성하며 값을 업데이트합니다 (v1이었다면 덮어썼을 것입니다).   

- **단일 값 저장 (버전 1 생성):**
    
        
    ```
    # 'kv-v2/` 마운트 경로 아래 'my-app/db-pass'라는 경로에 비밀 저장
    $ vault kv put kv-v2/my-app/db-pass username=db_user password=S3cr3t!
    
    # KV v2의 응답 (버전 등 메타데이터가 포함됨) [12, 22]
    Key                Value
    ---                -----
    created_time       2023-10-27T10:00:00.123456Z
    custom_metadata    <nil>
    deletion_time      n/a
    destroyed          false
    version            1
    ```
    
- **여러 값 업데이트 (버전 2 생성):** 동일한 경로에 다시 `put`을 실행하면, 기존 데이터(v1)는 보존된 채 새로운 버전(v2)이 생성됩니다.
    
        
    ```
    # 동일한 경로에 'password'를 업데이트하고 'hostname' 필드 추가
    $ vault kv put kv-v2/my-app/db-pass username=db_user password=NewS3cr3t! hostname=db.example.com
    
    # 응답 (version이 2로 증가한 것을 확인)
    Key                Value
    ---                -----
    created_time       2023-10-27T10:05:00.789012Z
    ```
    

... version 2 ```

- **전문가 팁 (JSON을 통한 입력):** 복잡한 구조의 비밀(예: JSON 형태의 서비스 계정 키)을 저장하거나 자동화 스크립트에서 사용할 경우, `@` 기호를 사용하여 JSON 파일을 직접 입력할 수 있습니다.   
    
        
    ```
    $ cat payload.json
    {
      "api_key": "xyz-123-long-key",
      "api_secret": "abc-789-super-secret",
      "region": "us-east-1"
    }
    
    $ vault kv put kv-v2/api/service-key @payload.json
    ```
    

### 비밀번호 조회 (Read: `vault kv get`)

`vault kv get` 명령어는 지정된 경로의 비밀을 조회합니다. KV v2에서는 기본적으로 **가장 최신 버전**의 비밀을 반환합니다.   

- **최신 버전 조회:**
    
        
    ```
    $ vault kv get kv-v2/my-app/db-pass
    
    # 응답 (메타데이터와 데이터 섹션이 명확히 분리됨) [13, 23]
    ====== Metadata ======
    Key                Value
    ---                -----
    created_time       2023-10-27T10:05:00.789012Z
    custom_metadata    <nil>
    deletion_time      n/a
    destroyed          false
    version            2
    
    ====== Data ======
    Key                Value
    ---                -----
    hostname           db.example.com
    password           NewS3cr3t!
    username           db_user
    ```
    
- **특정 과거 버전 조회:** `-version` 플래그를 사용하여 이전 버전을 조회할 수 있습니다. 이는 v2 엔진의 핵심 가치(롤백)를 보여줍니다.   
    
        
    ```
    # 버전 1의 데이터 (업데이트 전 비밀번호) 조회
    $ vault kv get -version=1 kv-v2/my-app/db-pass
    
    ====== Data ======
    Key                Value
    ---                -----
    password           S3cr3t!
    username           db_user
    ```
    
- **특정 필드만 추출 (자동화용):** 애플리케이션 환경 변수 등에 비밀번호 값을 주입할 때, `-field` 플래그는 셸 스크립팅을 매우 간결하게 만들어 줍니다.   
    
        
    ```
    $ export DB_PASSWORD=$(vault kv get -field=password kv-v2/my-app/db-pass)
    $ echo $DB_PASSWORD
    NewS3cr3t!
    ```
    

### 비밀번호 목록 확인 (List: `vault kv list`)

`vault kv list` 명령어는 특정 경로 _아래_에 존재하는 모든 비밀 키(key)와 하위 경로(folder)를 나열합니다.   

- **경고: 키 이름(Key Name)에 민감 정보를 포함하지 마십시오.** `list` 명령어는 키의 _이름_만 반환하며, 값(value)은 반환하지 않습니다. 하지만 `list` 권한은 `read` 권한보다 광범위하게 부여되는 경우가 많습니다. 만약 키 이름 자체에 `kv-v2/db-password-is-S3cr3t!`과 같이 민감한 정보를 포함하면, `read` 권한이 없는 사용자에게도 해당 정보가 노출될 수 있습니다.   
    
        
    ```
    $ vault kv put kv-v2/my-app/another-secret value=123
    $ vault kv put kv-v2/my-app/sub-folder/test value=abc
    
    $ vault kv list kv-v2/my-app/
    Keys
    ----
    another-secret
    db-pass
    sub-folder/
    ```
    
- **비밀 구조화 전략 (Architecture Insight):** 운영자는 비밀을 어떻게 구조화할지 초기에 결정해야 합니다. 이는 향후 정책 관리의 복잡도를 결정합니다.   
    
    - **안티 패턴 (단일 경로, 다중 필드):** `kv-v2/app1`이라는 경로 하나에 `db_pass=...`, `api_key=...`, `redis_pass=...` 등 모든 비밀을 저장하는 방식입니다.
        
        - **문제점:** `db_pass`만 업데이트하고 싶은데 실수로 `vault kv put kv-v2/app1 db_pass=newpass`를 실행하면, `api_key`와 `redis_pass` 필드가 **모두 삭제**됩니다 (덮어쓰기). 이를 방지하려면 `vault kv patch...` 명령어를 사용해야 하지만, 이는 실수를 유발하기 쉽습니다. 또한 접근 정책을 "A팀은 `db_pass`만 읽기"와 같이 세분화하기가 극도로 어렵습니다.   
            
    - **모범 사례 (계층적 경로, 분리된 비밀):** `kv-v2/app1/db-pass` (value=...), `kv-v2/app1/api-key` (value=...), `kv-v2/app1/redis-pass` (value=...) 처럼 경로 자체를 계층화하고 각 비밀을 분리하는 방식입니다.
        
        - **장점:** `list`로 해당 앱의 모든 비밀 목록을 볼 수 있고, `get`으로 개별 비밀을 안전하게 가져옵니다. `put`으로 인한 덮어쓰기 위험이 없으며, "A팀은 `kv-v2/app1/db-pass` 경로만 `read` 가능"과 같이 **매우 세분화된(granular) 정책**을 적용하기 용이합니다.   
            

### 심층 분석: KV v2의 3단계 삭제 (Delete vs. Destroy vs. Metadata Delete)

KV v2 운영 시 가장 혼동하기 쉬운 지점이 바로 '삭제'입니다. KV v1에서 `delete`는 영구 삭제였지만, v2에서는 '휴지통으로 이동'을 의미합니다. 운영자는 컴플라이언스 및 보안 사고 대응을 위해 이 3단계 삭제 모델을 명확히 이해해야 합니다.   

- **1단계: 소프트 삭제 (Soft Delete - `vault kv delete`)**
    
    - 이 명령어는 KV v2의 기본 삭제 동작입니다.
        
    - 이는 데이터를 스토리지에서 파기하는 것이 아니라, _가장 최신 버전_의 비밀에 `deletion_time` 메타데이터를 설정하여 '삭제됨'으로 표시(marking)합니다.   
        
    - 데이터는 스토리지에 **여전히 존재하며** 복구 가능합니다.   
        
    
        
    ```
    # 현재 최신 버전이 v2라고 가정
    $ vault kv delete kv-v2/my-app/db-pass
    Success! Data deleted (if it existed) at: kv-v2/my-app/db-pass
    
    # 이제 'get'을 시도하면 "no value found" 오류가 발생합니다.
    $ vault kv get kv-v2/my-app/db-pass
    No value found at kv-v2/data/my-app/db-pass
    ```
    
- **2단계: 삭제 복원 (Undelete - `vault kv undelete`)**
    
    - 소프트 삭제된 버전을 복원(Restore)할 수 있습니다.   
        
    
        
    ```
    # 방금 'delete'로 표시한 2번 버전을 복원
    $ vault kv undelete -versions=2 kv-v2/my-app/db-pass
    Success! Data written to: kv-v2/my-app/db-pass
    
    # 'get'을 하면 v2가 다시 조회됩니다. (v2가 복원되며 v3가 생성될 수도 있습니다)
    ```
    
- **3단계: 영구 삭제 (Hard/Permanent Delete)**
    
    - GDPR과 같은 컴플라이언스 요구사항이나, 비밀번호가 유출되어 스토리지에서 _완전히_ 제거해야 할 때 사용합니다. 이는 복구 불가능합니다.
        
    - **옵션 A: `vault kv destroy` (특정 버전 영구 삭제)**
        
        - 지정된 버전의 데이터만 스토리지에서 영구적으로 제거합니다.   
            
        - 이는 "휴지통에서 특정 파일만 선택하여 영구 삭제"하는 것과 같습니다.   
            

        ```
        # 1번 버전과 2번 버전을 영구적으로 파기
        $ vault kv destroy -versions="1,2" kv-v2/my-app/db-pass
        Success! Data written to: kv-v2/my-app/db-pass
        ```
        
    - **옵션 B: `vault kv metadata delete` (키의 모든 버전 영구 삭제)**
        
        - 가장 강력한 삭제 명령어입니다.
            
        - 이는 `my-app/db-pass`라는 키(Key) 자체와, 그에 속한 **모든 버전 (v1, v2,...)** 및 모든 메타데이터를 스토리지에서 영구적으로 제거합니다.   
            
        - 이는 "휴지통을 비우는" 것을 넘어, 해당 "폴더 자체를 모든 역사 기록과 함께 삭제"하는 것과 같습니다.   
            

        ```
        $ vault kv metadata delete kv-v2/my-app/db-pass
        Success! Data deleted (if it existed) at: kv-v2/my-app/db-pass
        ```
        

## IV. 3단계: 접근 주체 정의 (인증: Authentication)

비밀번호를 저장했다면, 이제 '누가(Who)' 이 비밀번호에 접근할 수 있는지 정의해야 합니다.

### 운영의 제1원칙: 루트 토큰(Root Token)을 즉시 폐기하라

Vault 클러스터를 초기화(`vault operator init`)할 때 'Initial Root Token'이 생성됩니다. 이 토큰은 Vault 내의 _모든_ 작업을 수행할 수 있는 '신(God)' 권한을 가지며, 만료 기간이 없을 수도 있습니다.   

**이 루트 토큰은 절대 운영 환경에서 사용되어서는 안 됩니다.**

루트 토큰은 오직 초기 설정(예: 이 가이드의 3단계, 4단계를 수행하여 첫 번째 관리자 계정과 정책을 생성하는 것)에만 사용하고, 해당 작업이 완료되는 즉시 폐기(`vault token revoke <root-token>`)해야 합니다.   

모든 운영 작업은 루트 토큰이 아닌, **최소 권한의 원칙(Principle of Least Privilege)**에 따라 생성된 토큰으로 수행해야 합니다.   

### Vault 인증 워크플로우 이해

Vault의 모든 접근은 토큰(Token)을 기반으로 합니다. 사용자(인간 또는 머신)는 Vault에 구성된 `Auth Method`(인증 방법)를 통해 자신을 인증합니다. 인증에 성공하면, Vault는 해당 사용자에게 **정책(Policies)**이 연결된 **토큰(Token)**을 발급합니다.   

사용자는 이 토큰을 사용하여 Vault API를 요청하며, 토큰에 연결된 정책이 '무엇을(What)' 할 수 있는지(인가)를 결정합니다.   

### 실무: 인증 방법(Auth Method) 활성화 및 사용자 생성

- **인간 사용자용: `userpass` 인증 방법**
    
    - 가장 간단한 사용자명/비밀번호 인증 방식입니다. (대규모 운영 환경에서는 Active Directory와 연동되는 `ldap` 또는 Okta 등과 연동되는 `oidc` 인증 방법을 권장하지만 , 초기 관리자 설정에는 `userpass`가 유용합니다.)   
        
    - **1. `userpass` 인증 활성화:** (이 작업은 루트 토큰으로 수행합니다.)

        ```
        # 'userpass/' 기본 경로에 활성화
        $ vault auth enable userpass
        Success! Enabled userpass auth method at: userpass/
        ```
        
    - **2. 관리자 사용자 생성 (및 정책 할당):**
        
        - 핵심은 사용자를 생성할 때 `policies` 인수를 통해 "어떤 정책을 가질 것인지"를 명시적으로 연결하는 것입니다.   
            
        - 여기서는 `admin-policy`라는 정책을 연결합니다. (이 정책은 V 단계에서 생성할 것입니다.)
            

        ```
        # 'my-admin' 사용자를 생성하고 'admin-policy' 정책을 할당
        # (이 작업도 루트 토큰으로 수행합니다.)
        $ vault write auth/userpass/users/my-admin \
            password="<매우-강력하고-안전한-비밀번호-입력>" \
            policies="admin-policy"
        Success! Data written to: auth/userpass/users/my-admin
        ```
        
    - **3. 로그인 (루트 토큰 사용 중단):** 이제 루트 토큰을 폐기(revoke)하고, 방금 생성한 `my-admin` 사용자로 로그인합니다.
        ```
        # (루트 토큰 폐기: vault token revoke <root-token>)
        
        # 'my-admin' 사용자로 로그인
        $ vault login -method=userpass username=my-admin
        Password (will be hidden): <매우-강력하고-안전한-비밀번호-입력>
        
        Success! You are now authenticated. The token information displayed below
        is already stored in the token helper. You do NOT need to run "vault login"
        again. Future Vault requests will automatically use this token.
        
        Key                  Value
        ---                  -----
        token                hvs.THIS_IS_YOUR_NEW_TOKEN...
        token_accessor      ...
        token_duration       768h
        token_renewable      true
        token_policies       ["admin-policy", "default"]
        identity_policies   
        policies             ["admin-policy", "default"]
        ```
        
        이제부터 모든 CLI 작업은 `admin-policy` 권한을 가진 이 새 토큰으로 수행됩니다.
        
- **애플리케이션(머신)용: `AppRole` 인증 방법 (개념)**
    
    - 인간이 아닌 머신(서버, 컨테이너, CI/CD 파이프라인)이 Vault에 인증하는 데 사용되는 표준 방식입니다.   
        
    - `RoleID`(사용자명에 해당)와 `SecretID`(비밀번호에 해당)의 조합을 사용하여 인증하고 토큰을 발급받습니다.
        
    - 운영 원칙은 '인간은 `userpass`/`ldap`/`oidc`, 머신은 `AppRole`/`kubernetes`/`aws-iam`'으로 분리하는 것입니다.   
        

## V. 4단계: 접근 권한 통제 (인가: Authorization)

인증(Authentication)이 "당신이 누구인가?"(Who you are)를 확인하는 과정이라면, 인가(Authorization)는 "당신이 무엇을 할 수 있는가?"(What you can do)를 통제하는 과정입니다.

### Vault 정책(Policy)의 핵심 원리: Deny-by-Default

Vault의 모든 접근 통제는 정책(Policy)을 통해 이루어집니다. 이 정책은 HCL(HashiCorp Configuration Language)이라는 선언적 언어로 작성되며, 경로(path)를 기반으로 합니다.   

가장 중요한 보안 원칙은 **Deny-by-Default** (기본적으로 거부)입니다. 빈 정책(empty policy)은 _아무 권한도_ 부여하지 않습니다. 사용자는 오직 하나 이상의 정책에 명시적으로 허용된(explicitly defined) 경로와 작업만 수행할 수 있습니다.   

### 정책(Policy) HCL 구문 해부: `path`와 `capabilities`

모든 정책은 하나 이상의 `path` 블록으로 구성됩니다.
```
# 정책(Policy) HCL의 기본 구조
path "PATH_STRING" {
  capabilities =
}
```

- **`path`:** Vault 내의 API 경로 (예: `kv-v2/data/my-app`). 와일드카드(`*`)나 세그먼트 매칭(`+`) 같은 특수 문자를 사용할 수 있습니다.   
    
- **`capabilities`:** 해당 경로에서 수행할 수 있는 작업(Operation)의 목록입니다.   
    

### 핵심 Policy Capabilities 해부

다음 표는 가장 자주 사용되는 `capabilities`와 그 의미를 설명합니다.   

|Capability|HTTP Method|설명|
|---|---|---|
|`create`|`POST`/`PUT`|**새로운** 데이터를 해당 경로에 작성합니다. (키가 존재하지 않을 때)|
|`read`|`GET`|해당 경로에서 데이터를 읽습니다.|
|`update`|`POST`/`PUT`|**기존** 데이터를 해당 경로에서 수정합니다. (키가 이미 존재할 때)|
|`delete`|`DELETE`|해당 경로의 데이터를 삭제합니다. (KV v2의 경우 Soft Delete)|
|`list`|`LIST`|해당 경로 _아래_의 키 목록을 조회합니다. (UI 탐색에 필수)|
|`sudo`|(다양함)|루트 권한이 필요한 민감한 경로(예: `sys/`)에 접근할 때 사용합니다.|
|`deny`|(N/A)|**명시적 거부.** 다른 어떤 정책이 허용하더라도, `deny`는 항상 우선합니다.|

  

**주의:** Vault 정책에는 `write`라는 Capability가 없습니다. 이는 실수로 인한 덮어쓰기를 방지하기 위해 의도적으로 `create`와 `update`로 분리되었습니다.   

### [!!!] 전문가 핵심 경고: KV v2 정책 경로의 함정 (`/data/`)

KV v2를 사용하기로 한 결정(II 단계)이 정책(V 단계)에 미치는 가장 중요하고도 혼란스러운 영향입니다. 이는 Vault를 처음 운영하는 엔지니어들이 99% 겪는 문제입니다.   

사용자가 CLI에서 `vault kv get kv-v2/my-secret`를 입력할 때, Vault 내부적으로 이 요청은 `v1/kv-v2/data/my-secret`라는 실제 API 엔드포인트로 라우팅됩니다.   

**결과적으로, 접근 정책(Policy)은 _반드시_ 사용자가 입력하는 경로가 아닌, 실제 API 경로인 `.../data/...`를 포함해야 합니다**.   

- **잘못된 정책 (작동 안 함 - v1 방식):**
    ```
    # 이 정책은 KV v1 용입니다. v2에서는 "Permission Denied"를 반환합니다! [52]
    path "kv-v2/my-app/db-pass" {
      capabilities = ["read"]
    }
    ```
    
- **올바른 정책 (KV v2 방식):**
    ```
    # 'data'가 마운트 경로와 비밀 경로 사이에 삽입되어야 합니다. [51, 53, 54]
    path "kv-v2/data/my-app/db-pass" {
      capabilities = ["read"]
    }
    ```
    
- **`list` 권한의 함정:** `list` 작업(예: `vault kv list kv-v2/my-app/`)은 `.../data/...` 경로가 아닌 `.../metadata/...` 경로를 사용합니다.         
    ```
    # 'kv-v2/my-app/` 경로에서 `list`를 허용하는 올바른 정책
    path "kv-v2/metadata/my-app/*" {
      capabilities = ["list"]
    }
    ```
    

### 전문가 인사이트: UI/CLI 탐색을 위한 계층적 `list` 권한

또 다른 주요 함정은 UI 탐색입니다. `kv-v2/data/apps/webapp/db` 경로에 `read` 권한을 부여했더라도, 해당 사용자는 Vault UI에서 `kv-v2/` -> `apps/` -> `webapp/`으로 클릭하여 이동할 수 없습니다.

이동(탐색)을 위해서는 **모든 상위 경로**에 `list` 권한이 필요합니다.   
```
# 'kv-v2/data/apps/webapp/db`를 읽기 위한 최소한의 '탐색' 정책
path "kv-v2/metadata/" {            # 1. 루트에서 'list'
  capabilities = ["list"]
}
path "kv-v2/metadata/apps/" {       # 2. 'apps/'에서 'list'
  capabilities = ["list"]
}
path "kv-v2/metadata/apps/webapp/" { # 3. 'webapp/'에서 'list'
  capabilities = ["list"]
}

# 최종 목적지 데이터에 대한 'read' 권한
path "kv-v2/data/apps/webapp/db" {
  capabilities = ["read"]
}
```

### 실행: 정책 파일 작성 및 Vault에 적용

정책은 HCL 파일로 작성하여 Vault에 업로드합니다.

- **1. 로컬에 HCL 파일 작성:**  `read-only-app.hcl`이라는 이름의 파일을 생성합니다.   
    ```
    # 'read-only-app.hcl'
    # 'my-app/db-pass' 비밀에 대한 읽기 전용 권한 부여 [52, 54]
    
    path "kv-v2/data/my-app/db-pass" {
      capabilities = ["read"]
    }
    
    # UI 탐색을 위한 list 권한 
    path "kv-v2/metadata/" {
      capabilities = ["list"]
    }
    path "kv-v2/metadata/my-app/" {
      capabilities = ["list"]
    }
    ```
    
- **2. Vault에 정책 업로드:** `vault policy write` 명령어를 사용하여 로컬 파일을 Vault에 업로드하고 `read-only-app`이라는 이름의 정책을 생성합니다. (이 작업은 `admin-policy` 권한을 가진 `my-admin` 사용자로 수행합니다.)    
    ```
    $ vault policy write read-only-app./read-only-app.hcl
    Success! Uploaded policy: read-only-app
    ```
    
- **3. 정책을 사용자(또는 AppRole)에 연결:** 이제 이 `read-only-app` 정책을 실제 인증 주체(예: `userpass` 사용자 또는 `AppRole`)에 연결합니다.   
    
        
    ```
    # 'app-user'라는 새 사용자를 만들고 'read-only-app' 정책을 할당
    $ vault write auth/userpass/users/app-user \
        password="<app-user-password>" \
        policies="read-only-app"
    ```
    

### 사례별 정책 예시 (Policy Cookbook)

- **예시 1: `admin-policy` (IV 단계에서 사용한 관리자 정책)** 이 정책은 Vault의 핵심 기능을 관리(인증, 정책, 시크릿 엔진)하고 모든 KV 비밀에 접근할 수 있는 강력한 권한을 부여합니다.       
    ```
    # 'admin-policy.hcl'
    
    # 모든 KV v2 경로에 대한 전체 권한 (CRUDL + Destroy)
    path "kv-v2/data/*"    { capabilities = ["create", "read", "update", "delete", "list"] }
    path "kv-v2/metadata/*" { capabilities = ["list", "delete"] }
    path "kv-v2/destroy/*"  { capabilities = ["update"] } # 'destroy'는 'update' cap임
    
    # 인증 방법(Auth Methods) 관리 권한
    path "auth/*" { capabilities = ["create", "read", "update", "delete", "list", "sudo"] }
    
    # 정책(Policies) 관리 권한
    path "sys/policies/acl/*" { capabilities = ["create", "read", "update", "delete", "list"] }
    
    # Secrets Engines 관리 권한
    path "sys/mounts/*" { capabilities = ["create", "read", "update", "delete", "list", "sudo"] }
    ```
    
- **예시 2: 특정 팀을 위한 개발 환경 접근 (개발자용)** `dev-team/` 경로 아래의 모든 비밀에 대한 모든 권한을 부여합니다.   
    ```
    # 'policy_dev_team.hcl'
    
    # dev-team/ 경로 아래 모든 비밀에 대한 모든 권한
    path "kv-v2/data/dev-team/*" {
      capabilities = ["create", "read", "update", "delete", "list"]
    }
    
    # 탐색을 위한 list 권한
    path "kv-v2/metadata/dev-team/*" {
      capabilities = ["list"]
    }
    path "kv-v2/metadata/" {
      capabilities = ["list"]
    }
    ```
    

## VI. 결론: 보안 비밀 관리의 운영 정착

### 핵심 워크플로우 요약

Vault 클러스터 구성 후, 정적 비밀(비밀번호)을 안전하게 관리하고 운영하는 것은 다음의 3단계 워크플로우로 요약할 수 있습니다.

1. **금고 선택 (Secrets Engine):** `vault secrets enable -version=2 -path=<path> kv` 명령을 실행하여, 버전 관리, 소프트 삭제, CAS 기능을 제공하는 **KV v2** 엔진을 활성화합니다. 이는 실수로부터의 복원력과 자동화 안정성을 보장하는 첫걸음입니다.   
    
2. **접근 주체 정의 (Authentication):** `userpass`(인간용) 또는 `AppRole`(머신용) 같은 인증 방법을 활성화하고, **루트 토큰을 대체**할 관리자 및 애플리케이션 ID를 생성합니다. 이때 `policies="..."` 인수를 통해 적절한 정책을 연결하는 것이 핵심입니다.   
    
3. **권한 통제 (Authorization):** HCL 파일을 작성하고 `vault policy write`를 실행하여 세분화된 접근 정책을 적용합니다. 이때 KV v2의 정책 경로는 _반드시_ `.../data/...` (CRUD용)  또는 `.../metadata/...` (List용)  형태여야 함을 명심해야 합니다.   
    

### 전문가의 최종 권고: 정적 비밀을 넘어 동적 비밀로

이 가이드를 통해 업계 모범 사례에 따른 정적 비밀 관리 시스템을 성공적으로 구축했습니다. 하지만 과 에서 강조했듯이, KV 엔진에 저장된 비밀번호는 여전히 '정적'입니다. 유출 시 비밀번호를 수동으로 교체(rotate)하고, 유출된 버전을 `destroy`해야 하는 운영 부담이 남아있습니다.   

귀하의 다음 Vault 여정은 이 정적 비밀의 _필요성 자체를 제거_하는 것이어야 합니다. KV v2를 통해 정적 비밀을 완벽하게 통제하고 중앙화하는 것은, 이러한 고급 동적 비밀 관리로 나아가기 위한 가장 중요하고 견고한 첫걸음입니다.

**다음 확장 로드맵:**

1. **Database Secrets Engine (동적 DB 자격 증명):** 애플리케이션이 Vault에 요청할 때마다 _즉시_ 5분(예시) 수명의 고유한 데이터베이스 유저/패스워드를 동적으로 생성하고, 만료 시 Vault가 자동으로 삭제합니다.   
    
2. **Cloud Secrets Engines (AWS/Azure/GCP):** CI/CD 파이프라인이 배포를 위해 Vault에 요청하면, 30초 수명의 임시 IAM 자격 증명을 동적으로 생성합니다.   
    
3. **Transit Secrets Engine (암호화 서비스):** 민감한 데이터를 Vault에 _저장_하는 대신, Vault를 '암호화 서비스(Encryption-as-a-Service)'로 사용하여 애플리케이션 데이터를 안전하게 암/복호화합니다.