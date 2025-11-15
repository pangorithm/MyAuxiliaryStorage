# Vault 동적 시크릿을 이용한 MySQL 접근 제어: 전문가급 구현 가이드

## I. 파운데이션 아키텍처: 'How'를 넘어 'Why'의 이해

### A. 도입: 정적 시크릿을 넘어서

현대 인프라에서 정적 시크릿(Static Secrets), 즉 설정 파일, 환경 변수 또는 코드 저장소에 하드코딩된 데이터베이스 사용자 이름과 암호는 가장 심각하고 일반적인 공격 벡터입니다. 이러한 자격 증명은 대부분 수명이 길고(long-lived), 과도한 권한(high-standing privileges)을 가지며, 거의 순환(rotate)되지 않습니다. 이는 곧, 단 하나의 자격 증명 유출이 전체 데이터베이스의 침해로 이어질 수 있음을 의미합니다.

이 보고서에서 제시하는 Vault 동적 시크릿(Dynamic Secrets) 모델은 이러한 패러다임을 근본적으로 전환합니다. 이는 단순한 암호 관리의 개선이 아닌, 'Identity-Driven Infrastructure' (신원 기반 인프라)로의 아키텍처적 변화입니다. 이 모델에서 자격 증명은 더 이상 영구적이지 않습니다. 대신, 필요할 때(on-demand) 실시간으로 생성되고, 수명이 극도로 짧으며(ephemeral, short-lived), 사용 후 자동으로 파기됩니다.

### B. Vault 동적 시크릿 모델: 하이레벨 개요

이 아키텍처의 작동 방식은 명확한 워크플로우를 따릅니다.

1. **클라이언트 인증 (Client Authentication):** 애플리케이션이나 사용자는 AppRole, OIDC, Kubernetes 인증 등과 같은 신뢰할 수 있는 방법을 사용하여 먼저 Vault에 자신을 인증하고 토큰을 발급받습니다.
    
2. **시크릿 요청 (Credential Request):** 클라이언트는 특정 '역할(Role)'에 매핑된 동적 자격 증명을 Vault에 요청합니다. (예: `vault read database/creds/app-readonly`)
    
3. **Vault 서버 처리 (Vault Server Processing):** Vault는 이 요청을 수신하고, 해당 요청이 허용된 정책(Policy)에 부합하는지 확인합니다.
    
4. **데이터베이스 연결 (Database Connection):** Vault는 자체적으로 보유한, 고도로 특권화된 '루트' 계정을 사용하여 대상 MySQL 데이터베이스에 연결합니다.
    
5. **동적 생성 (Dynamic Creation):** Vault는 요청받은 '역할'에 정의된 SQL 문(`creation_statements`)을 실행하여, MySQL 내에 완전히 새로운 사용자(예: `v-app-abc-123...`)를 _실시간으로 생성_하고 필요한 권한을 부여합니다.
    
6. **리스(Lease) 발급 (Lease Issuance):** Vault는 새로 생성된 임시 사용자 이름과 암호, 그리고 이 자격 증명의 유효 기간(Lease)을 정의하는 `lease_id`를 클라이언트에게 반환합니다.
    
7. **데이터베이스 접근 (Database Access):** 클라이언트는 이 임시 자격 증명을 사용하여 MySQL에 직접 접근하고 작업을 수행합니다.
    
8. **자동 폐기 (Automated Revocation):** 리스 기간이 만료되거나 클라이언트가 명시적으로 리스를 폐기(`revoke`)하면, Vault는 다시 MySQL에 연결하여 해당 임시 사용자를 _즉시 삭제_(`DROP USER...`)합니다.
    

### C. 본 가이드의 행위자 (The Actors)

이 구현 과정을 이해하기 위해, 우리는 네 가지 주요 구성 요소를 정의합니다.

- **Vault 서버 (The Vault Server):** 시크릿의 생성, 리스 관리, 폐기를 총괄하는 중앙 권한 기관입니다.
    
- **MySQL 데이터베이스 (The MySQL Database):** 우리가 보호하려는 대상 리소스입니다.
    
- **"Vault 루트" MySQL 사용자 (The "Vault Root" MySQL User):** Vault가 MySQL을 관리하기 위해 사용하는, 높은 권한을 가진 전용 계정입니다. (본 가이드에서는 `v-admin`으로 명명)
    
- **동적 사용자 (The Dynamic User):** Vault가 애플리케이션을 위해 생성하는, 수명이 짧은 임시 사용자입니다. (예: `v-app-readonly-xyz-456...`)
    
- **클라이언트 애플리케이션 (The Client Application):** 데이터베이스 접근이 필요한 최종 서비스 또는 사용자입니다.
    

## II. Phase 1: 사전 준비: Vault 통합을 위한 MySQL 설정

Vault가 데이터베이스를 관리하도록 허용하기 전, 데이터베이스 자체에 대한 선행 작업이 필요합니다. 이는 Vault가 사용할 '루트' 계정을 생성하고, 이 계정에 적절한(그리고 매우 높은) 권한을 부여하는 작업입니다.

### A. "Vault 루트" 사용자: 필수적인 특권 계정 생성

Vault는 동적 사용자를 생성(`CREATE USER`)하고, 권한을 부여(`GRANT`)하며, 삭제(`DROP USER`)할 수 있어야 합니다. 이를 위해 Vault가 사용하는 자체 계정은 이러한 관리 작업을 수행할 수 있는 권한을 가져야 합니다.

여기서 핵심은 `WITH GRANT OPTION`입니다. Vault의 `v-admin` 계정은 단순히 `SELECT`, `INSERT` 권한을 갖는 것이 아니라, _다른 사용자에게 이러한 권한을 부여할 수 있는 권한_을 가져야 합니다. 이것이 없으면 Vault는 동적 사용자를 생성할 수는 있지만, 그 사용자에게 아무런 권한도 줄 수 없어 전체 모델이 실패합니다.

**주석이 포함된 SQL 명령어 블록:**

SQL

```
/*
 * 1. Vault 전용 관리자 사용자를 생성합니다.
 *    식별을 위해 'v-admin'과 같은 명확한 이름을 권장합니다.
 *    'vault-admin-password'는 즉시 교체될 것이므로,
 *    초기 부트스트랩을 위한 임시 고-엔트로피 암호입니다.
 */
CREATE USER 'v-admin'@'%' IDENTIFIED BY 'vault-admin-password'; 


/*
 * 2. Vault에 필요한 모든 권한을 부여합니다.
 *    이것이 동적 시크릿 엔진의 핵심입니다.
 *    Vault는 새 사용자를 생성(CREATE USER)하고,
 *    권한을 부여(GRANT)하며, 사용자를 삭제(DROP USER)해야 합니다.
 *    `WITH GRANT OPTION`은 `v-admin`이 생성한 동적 사용자에게
 *    권한을 부여할 수 있도록 하는 필수 옵션입니다.
 */
GRANT ALL PRIVILEGES ON *.* TO 'v-admin'@'%' WITH GRANT OPTION;


/*
 * 3. 권한을 즉시 적용합니다.
 */
FLUSH PRIVILEGES;
```

### B. 연결 강화: 네트워크 범위 제한

위의 예시에서 사용된 `'@'%'`는 '모든 호스트에서'의 연결을 허용합니다. 이는 보안상 심각한 위험입니다. 프로덕션 환경에서는 Vault 서버가 실행되는 특정 IP 주소 또는 서브넷으로 이 범위를 엄격하게 제한해야 합니다.

예를 들어, Vault 서버의 IP가 `10.1.1.5` 및 `10.1.1.6`이라면, 다음과 같이 사용자를 생성해야 합니다.

SQL

```
CREATE USER 'v-admin'@'10.1.1.5' IDENTIFIED BY '...';
CREATE USER 'v-admin'@'10.1.1.6' IDENTIFIED BY '...';
GRANT ALL PRIVILEGES ON *.* TO 'v-admin'@'10.1.1.5' WITH GRANT OPTION;
GRANT ALL PRIVILEGES ON *.* TO 'v-admin'@'10.1.1.6' WITH GRANT OPTION;
```

이는 Vault와 MySQL 간의 연결을 네트워크 수준에서 보호하는 최소 권한의 원칙을 적용하는 것입니다. 추가적으로, `...REQUIRE SSL;` 옵션을 사용하여 Vault와 MySQL 간의 모든 트래픽이 암호화되도록 강제하는 것이 강력히 권장됩니다.

## III. Phase 2: Vault 구성: 데이터베이스 연결

이제 MySQL이 준비되었으므로, Vault에게 MySQL에 연결하고 관리하는 방법을 알려줄 차례입니다.

### A. 데이터베이스 시크릿 엔진 활성화

먼저 Vault에서 `database` 시크릿 엔진을 활성화합니다.


```
# 기본 경로인 'database/'에 데이터베이스 시크릿 엔진을 활성화합니다.
vault secrets enable database

# (출력) Success! Enabled the database secrets engine at: database/
```

엔터프라이즈 환경에서는 여러 데이터베이스를 관리해야 할 수 있습니다. 이 경우, 식별 가능한 경로를 사용하는 것이 좋습니다. (예: `vault secrets enable -path=mysql-prod database`, `vault secrets enable -path=postgres-dev database`) 이는 명확한 정책 분리와 관리를 가능하게 합니다.

### B. 연결 설정 작성 (부트스트랩)

다음으로, `database/config` 엔드포인트에 '어떻게' MySQL에 연결할지 작성합니다. 이 단계에서 Phase 1에서 생성한 `v-admin` 사용자의 자격 증명을 Vault에 전달합니다.

**주석이 포함된 명령어 블록:**


```
vault write database/config/my-mysql-db \
    plugin_name="mysql-database-plugin" \
    connection_url="{{username}}:{{password}}@tcp(mysql.prod.internal:3306)/" \
    username="v-admin" \
    password="vault-admin-password" \
    allowed_roles="app-readonly,app-readwrite,schema-migration"

```

**파라미터 상세 분석:**

- `database/config/my-mysql-db`: `my-mysql-db`는 이 연결 설정의 _이름_입니다. 이 이름은 임의로 지정할 수 있으며, 이후 '역할(Role)'을 정의할 때 이 이름을 참조하게 됩니다.
    
- `plugin_name`: MySQL의 경우, `mysql-database-plugin`을 정확히 사용해야 합니다.
    
- `connection_url`: Vault가 사용할 연결 문자열의 _템플릿_입니다. 여기서 `{{username}}`과 `{{password}}`는 동적 사용자의 것이 _아니라_, 바로 아래 `username`과 `password` 파라미터의 값으로 채워집니다. 이 템플릿 방식은 Vault가 자신의 루트 암호를 순환(rotate)할 수 있도록 하는 핵심 메커니즘입니다.
    
- `username="v-admin"`: Phase 1에서 생성한 "Vault 루트" 사용자입니다.
    
- `password="..."`: `v-admin`의 _초기 부트스트랩 암호_입니다.
    
- `allowed_roles`: 이 연결을 사용할 수 있는 Vault 역할 이름의 _화이트리스트_입니다. 이는 중요한 보안 경계입니다.
    

### C. "루트 자격 증명 순환" 활성화: "Vault 루트" 사용자 보안

가장 중대한 단계입니다. 방금 우리는 Vault 설정에 `v-admin`의 정적 암호를 입력했습니다. 이 암호는 이제 Vault의 암호화된 스토리지에 저장되었지만, MySQL 데이터베이스 자체에는 여전히 _정적_인 상태입니다. 이는 심각한 보안 허점입니다. 만약 누군가 Vault의 스토리지(백업 등)에 접근할 수 있다면 이 "신(God)" 계정의 암호를 알아낼 수 있습니다.

따라서, _즉시_ Vault에게 이 암호의 제어권을 넘기고 순환(rotate)하도록 명령해야 합니다.

**주석이 포함된 명령어 블록:**


```
# Vault에게 'my-mysql-db' 연결에 설정된 'v-admin' 사용자의
# 암호를 MySQL 데이터베이스에서 즉시 변경하도록 명령합니다.
# Vault는 변경된 새 암호를 내부 스토리지에 자동으로 업데이트합니다.
vault write -f database/rotate-root/my-mysql-db


# (출력) Success! Data written to: database/rotate-root/my-mysql-db
```

이 명령이 실행되는 순간, 부트스트랩에 사용된 `vault-admin-password`는 더 이상 유효하지 않습니다. Vault는 자체적으로 생성한 고-엔트로피 암호로 `v-admin`의 암호를 변경했으며, 이 새로운 암호는 _오직 Vault만이 알고 있습니다._ 이제 운영자를 포함한 그 누구도 이 특권 계정의 암호를 알지 못합니다. 이로써 'Vault 루트' 계정 자체가 완벽하게 Vault의 통제 하에 들어가게 됩니다.

## IV. Phase 3: 동적 역할 작성 (쿼리의 핵심)

이제 Vault가 MySQL에 연결되었고 자신의 루트 계정을 확보했으므로, 애플리케이션이 요청할 '역할(Role)'을 정의할 차례입니다. 이것이 바로 사용자 쿼리의 핵심입니다.

### A. `database/roles` 정의의 해부

'역할'은 Vault가 동적 사용자를 생성하기 위해 사용하는 "레시피"입니다. 주요 파라미터는 다음과 같습니다.

- `db_name`: 이 역할이 사용할 `database/config`의 이름 (예: `my-mysql-db`).
    
- `creation_statements`: 사용자를 생성하고 권한을 부여하는 SQL 문(들)의 배열. `{{username}}`, `{{password}}` 템플릿을 사용합니다.
    
- `default_ttl`: 자격 증명의 기본 리스 기간입니다. (예: `1h`)
    
- `max_ttl`: 갱신을 포함하여 자격 증명이 존재할 수 있는 최대 시간입니다. (예: `24h`)
    

### B. 예제 A: 'Read-Only' 애플리케이션 역할 (`app-readonly`)

가장 일반적인 사용 사례입니다. 애플리케이션 백엔드가 데이터베이스에서 `SELECT`만 수행해야 하는 경우입니다.

**주석이 포함된 명령어 블록:**


```
vault write database/roles/app-readonly \
    db_name="my-mysql-db" \
    creation_statements="CREATE USER '{{username}}'@'%' IDENTIFIED BY '{{password}}'; \
                         GRANT SELECT ON app_db.* TO '{{username}}'@'%';" \
    default_ttl="1h" \
    max_ttl="24h"

```

**분석:**

- `db_name`: Phase 2에서 설정한 `my-mysql-db` 연결을 사용하도록 지정합니다.
    
- `creation_statements`:
    
    1. `CREATE USER...`: Vault가 생성한 임시 `{{username}}`과 `{{password}}`로 사용자를 생성합니다.
        
    2. `GRANT SELECT...`: 이 사용자에게 `app_db` 데이터베이스의 _모든 테이블_에 대해 `SELECT` 권한만 부여합니다. 이는 완벽한 최소 권한의 원칙을 따릅니다.
        
- `default_ttl="1h"`: 자격 증명은 기본 1시간 동안 유효하며, 갱신(renew)할 수 있습니다.
    
- `max_ttl="24h"`: 갱신을 하더라도, 이 자격 증명은 24시간 이상 존재할 수 없습니다. 24시간이 지나면 애플리케이션은 _반드시_ 새로운 자격 증명을 다시 요청해야 합니다.
    

### C. 예제 B: 'Read-Write' 서비스 역할 (`app-readwrite`)

사용자 정보 수정 API와 같이, 레코드를 `INSERT`하거나 `UPDATE`해야 하는 서비스에 사용됩니다.

**주석이 포함된 명령어 블록:**


```
vault write database/roles/app-readwrite \
    db_name="my-mysql-db" \
    creation_statements="CREATE USER '{{username}}'@'%' IDENTIFIED BY '{{password}}'; \
                         GRANT SELECT, INSERT, UPDATE ON services_db.user_table TO '{{username}}'@'%';" \
    default_ttl="30m" \
    max_ttl="2h"
```

**분석:**

- 권한이 `INSERT`, `UPDATE`로 증가했습니다.
    
- 권한 범위가 `services_db.*`가 아닌, `services_db.user_table`이라는 _특정 테이블_로 더욱 엄격하게 제한되었습니다.
    
- 리스 기간이 더 짧아졌습니다 (`default_ttl="30m"`, `max_ttl="2h"`). 더 높은 권한은 더 짧은 수명 주기를 가져야 한다는 보안 원칙을 반영합니다.
    

### D. 예제 C: 'Schema-Migration' 임시 역할

CI/CD 파이프라인(예: Flyway, Liquibase)이 `ALTER TABLE`과 같은 스키마 변경을 수행할 때 사용하는, 매우 높은 권한의 역할입니다.

**주석이 포함된 명령어 블록:**


```
vault write database/roles/schema-migration \
    db_name="my-mysql-db" \
    creation_statements="CREATE USER '{{username}}'@'%' IDENTIFIED BY '{{password}}'; \
                         GRANT ALTER, CREATE, DROP, INDEX, REFERENCES ON app_db.* TO '{{username}}'@'%';" \
    default_ttl="60s" \
    max_ttl="300s"
```

**분석:**

- 권한이 `ALTER`, `CREATE`, `DROP` 등 사실상 관리자에 가깝습니다.
    
- 이러한 막강한 권한에 대한 유일한 통제 수단은 _극도로 공격적인 TTL_입니다.
    
- `default_ttl="60s"`, `max_ttl="300s"`(5분)는 이 자격 증명이 "유출"되더라도 5분 이상 존재할 수 없음을 보장합니다. CI/CD 작업은 이 자격 증명을 발급받아 _즉시_ 사용하고, 작업이 끝나면 (또는 5분이 지나면) 자격 증명은 자동으로 소멸됩니다.
    

### E. 사용자 정의 폐기 구문 (`revocation_statements`)

기본적으로 Vault는 동적 사용자를 제거하기 위해 `DROP USER '{{username}}'@'%'`를 실행할 만큼 "똑똑"합니다. 하지만 `creation_statements`에서 사용자를 생성하는 것 외에 다른 복잡한 작업을 수행했다면, 별도의 정리 로직이 필요할 수 있습니다.


```
vault write database/roles/app-readonly \
   ... \
    revocation_statements="DROP USER '{{username}}'@'%';"
```

대부분의 MySQL 사용 사례에서는 이 구문이 암시적으로 처리되므로 명시적으로 정의할 필요가 없지만, 완전한 제어를 위해 존재합니다.

### F. 고급 패턴: 저장 프로시저 (Stored Procedures)

`creation_statements`에 원시 SQL을 배열로 포함하는 것은 Vault 구성과 데이터베이스 스키마 간의 "취약한 결합(fragile glue)"을 만듭니다. 만약 DBA가 권한 정책을 변경(예: 새로운 테이블에 `SELECT` 추가)해야 한다면, Vault 운영자가 `vault write` 명령을 다시 실행하여 _인프라 설정_을 변경해야 합니다. 이는 DevSecOps 안티패턴입니다.

전문가 수준의 솔루션은 이 책임을 분리하는 것입니다.

1. **DBA (데이터베이스 팀):** 데이터베이스 내에 저장 프로시저를 생성하고 관리합니다.
    
    SQL
    
    ```
    CREATE PROCEDURE sp_create_readonly_user(IN p_user VARCHAR(255), IN p_pass VARCHAR(255))
    BEGIN
        CREATE USER p_user@'%' IDENTIFIED BY p_pass;
        GRANT SELECT ON app_db.table1 TO p_user@'%';
        GRANT SELECT ON app_db.table2 TO p_user@'%';
        -- 'read-only'에 대한 정의가 여기에 중앙 집중화됨
    END
    ```
    
2. **Vault 운영자 (DevSecOps 팀):** Vault의 역할 정의를 극도로 단순화합니다.
    
        
    ```
    vault write database/roles/app-readonly \
        db_name="my-mysql-db" \
        creation_statements="CALL sp_create_readonly_user('{{username}}', '{{password}}');" \
        default_ttl="1h" \
        max_ttl="24h"
    ```
    

이 패턴은 두 시스템을 완벽하게 **분리(decouple)**합니다. Vault는 더 이상 'read-only'가 _무엇_을 의미하는지 알 필요가 없습니다. 단지 _어떻게_ 해당 프로시저를 호출하는지만 알면 됩니다. 'read-only'의 실제 정의는 이제 DBA가 관리하는 데이터베이스 내부에 안전하게 존재합니다.

### 동적 역할 예제 요약

|**역할 이름**|**목적**|**예제 권한**|**기본 TTL**|**최대 TTL**|**핵심 보안 고려 사항**|
|---|---|---|---|---|---|
|`app-readonly`|애플리케이션 백엔드 (읽기 전용)|`GRANT SELECT ON app_db.*`|1h|24h|가장 일반적. 데이터 유출 방지를 위해 `SELECT`로 엄격히 제한.|
|`app-readwrite`|데이터 수정 서비스 (API 등)|`GRANT SELECT, INSERT, UPDATE ON services_db.user_table`|30m|2h|더 높은 권한(쓰기)은 더 짧은 TTL과 더 좁은 범위(테이블)로 완화.|
|`schema-migration`|CI/CD 데이터베이스 스키마 변경|`GRANT ALTER, CREATE, DROP ON app_db.*`|60s|300s|극도로 높은 권한. 5분 미만의 매우 짧은 TTL로만 위험을 완화.|
|`sp-readonly` (고급)|저장 프로시저 기반 역할|`CALL sp_create_readonly_user(...)`|1h|24h|Vault와 DB 스키마의 책임 분리. 가장 성숙하고 권장되는 패턴.|

## V. Phase 4: 실제 라이프사이클 시연: "작동 증명"

이제 설계한 시스템이 실제로 어떻게 작동하는지, "요람에서 무덤까지" 전체 라이프사이클을 검증합니다.

### A. 생성: `vault read` (결실)

클라이언트 애플리케이션이 자격 증명을 얻는 방법입니다.

**주석이 포함된 명령어 블록:**


```
# 'app-readonly' 역할로부터 새로운 동적 자격 증명을 요청합니다.
vault read database/creds/app-readonly


# (예상 출력)
# Key                Value
# ---                -----
# lease_id           database/creds/app-readonly/abc-123...
# lease_duration     1h
# lease_renewable    true
# password           a3-T9cZqR8v...
# username           v-app-readonly-xyz-456...
```

**분석:**

클라이언트는 이 응답을 받습니다. `username`과 `password`는 즉시 데이터베이스에 연결하는 데 사용됩니다. `lease_id`는 이 자격 증명의 "영수증"이며, 갱신(renew) 및 폐기(revoke)에 사용됩니다.

### B. 검증 (1부): MySQL에서의 생성 증명

이 단계는 시스템에 대한 신뢰를 구축하는 데 필수적입니다. MySQL 관리자로 로그인하여 Vault가 수행한 작업을 직접 확인합니다.

**주석이 포함된 명령어 블록:**


```
# 1. MySQL 관리자로, 해당 동적 사용자가 실제로 생성되었는지 확인합니다.
mysql -u root -p -e "SELECT user, host FROM mysql.user WHERE user LIKE 'v-app-readonly-%';"
# (출력) -> v-app-readonly-xyz-456... | %
# (성공: 사용자가 존재함)

# 2. 이 사용자에게 올바른 권한이 부여되었는지 확인합니다.
mysql -u root -p -e "SHOW GRANTS FOR 'v-app-readonly-xyz-456...'@'%';"
# (출력) -> GRANT SELECT ON `app_db`.* TO `v-app-readonly-xyz-456...`@`%`
# (성공: 정확히 SELECT 권한만 부여됨)

# 3. 새로 발급된 동적 사용자로 직접 연결을 시도합니다. (읽기 시도)
mysql -u 'v-app-readonly-xyz-456...' -p'a3-T9cZqR8v...' \
      -e "USE app_db; SELECT * FROM some_table LIMIT 1;"
# (출력) -> (테이블 데이터가 성공적으로 조회됨)
# (성공: 읽기 권한이 작동함)

# 4. 금지된 작업을 시도합니다. (삭제 시도)
mysql -u 'v-app-readonly-xyz-456...' -p'a3-T9cZqR8v...' \
      -e "USE app_db; DELETE FROM some_table WHERE id=1;"
# (출력) -> ERROR 1142 (42000): DELETE command denied to user...
# (성공!: 최소 권한의 원칙이 작동함을 증명)
```

이 네 단계의 검증은 시스템이 의도한 대로 정확하게 작동하고 있음을 증명합니다.

### C. 갱신: `vault lease renew`

`default_ttl` (1시간)보다 오래 실행되어야 하는 애플리케이션은 리스를 갱신해야 합니다.


```
# 'lease_id'를 사용하여 리스를 갱신합니다.
# max_ttl(24h)에 도달할 때까지 1시간씩 연장할 수 있습니다.
vault lease renew database/creds/app-readonly/abc-123...

```

이는 중요한 아키텍처적 함의를 가집니다. 애플리케이션은 더 이상 자격 증명을 "발급받고 잊어버리는(fire-and-forget)" 방식이어서는 안 됩니다. 애플리케이션은 "리스를 인식(lease-aware)"하도록 설계되어야 합니다. 즉, 리스 만료 전에 갱신을 시도하는 백그라운드 로직이 필요합니다. (또는 `vault-agent`와 같은 사이드카를 사용하여 이 로직을 애플리케이션에서 분리할 수 있습니다.)

### D. 폐기: `vault lease revoke` (소멸)

애플리케이션이 종료되거나, 보안 침해가 감지되었을 때 리스를 즉시 종료시킬 수 있습니다.

**주석이 포함된 명령어 블록:**


```
# 리스가 만료되기 전에 강제로 폐기합니다.
vault lease revoke database/creds/app-readonly/abc-123...

# (출력) Success! Revoked lease:...
```

### E. 검증 (2부): MySQL에서의 삭제 증명

폐기 명령의 "마법"은 Vault가 즉시 MySQL에 연결하여 해당 사용자를 정리하는 것입니다.

**주석이 포함된 명령어 블록:**


```
# 폐기 명령 직후, MySQL 관리자로 사용자가 사라졌는지 확인합니다.
mysql -u root -p -e "SELECT user, host FROM mysql.user WHERE user = 'v-app-readonly-xyz-456...';"
# (출력) -> (empty set)
# (성공: 사용자가 즉시 삭제되었음)
```

이로써 "요람에서 무덤까지"의 전체 라이프사이클이 완벽하게 자동화되었으며, 데이터베이스에 고아(orphaned) 자격 증명이 남지 않음이 증명되었습니다.

## VI. 프로덕션 강화: 정책, 감사, 그리고 고가용성

지금까지의 과정은 "기능"을 구현한 것입니다. 이제부터는 이를 "프로덕션" 수준으로 만드는 비기능적 보안 요구사항입니다.

### A. Vault ACL 정책: 자판기 잠그기

이것은 전체 아키텍처에서 _가장 중요한_ 내부 보안 통제입니다.

**문제점:** 우리는 `app-readonly`, `app-readwrite`, `schema-migration`이라는 세 가지 역할을 만들었습니다. 만약 Vault ACL 정책이 없다면, `app-readonly` 애플리케이션이 인증받은 후 `vault read database/creds/schema-migration`을 요청하는 것을 막을 방법이 없습니다. 이는 _치명적인_ 권한 상승(privilege escalation)입니다.

**해결책:** Vault의 ACL 정책(HCL 언어)을 사용하여 "자판기"의 버튼을 제한해야 합니다.

app-readonly 애플리케이션을 위한 정책 (HCL):

(이 정책은 app-readonly의 AppRole 또는 인증 토큰에 연결됨)

Terraform

```
# 정책 이름: app-readonly-policy.hcl

# 'app-readonly' 역할에 대한 자격 증명 읽기('read')만 허용
path "database/creds/app-readonly" {
  capabilities = ["read"]
}

# 다른 모든 database/creds 경로는 암시적으로 거부됨
# (예: database/creds/schema-migration 접근 불가)
```

CI/CD 파이프라인을 위한 정책 (HCL):

(이 정책은 CI/CD 작업의 OIDC/JWT 토큰에 연결됨)

Terraform

```
# 정책 이름: cicd-migration-policy.hcl

# 'schema-migration' 역할에 대한 자격 증명 읽기만 허용
path "database/creds/schema-migration" {
  capabilities = ["read"]
}

# (예: database/creds/app-readonly 접근 불가)
```

이 HCL 정책은 `app-readonly` 애플리케이션이 `schema-migration` 역할을 _요청조차 할 수 없도록_ 보장합니다. 이는 Vault 자체에 대한 최소 권한의 원칙을 적용하는 것이며, 동적 시크릿 아키텍처의 핵심 보안 제어입니다.

### B. 감사 및 모니터링: "누가, 무엇을, 언제"

Vault는 모든 요청과 응답을 불변의 감사 로그(Audit Logs)에 기록합니다.

애플리케이션이 `database/creds/app-readonly`에서 자격 증명을 읽을 때, 감사 로그에는 다음과 같은 명확한 기록이 남습니다.

- `timestamp`: 요청이 발생한 정확한 시간.
    
- `auth.entity_id` (또는 `auth.display_name`): _누가_ 요청했는가? (예: `approle-backend-prod`)
    
- `request.path`: _무엇을_ 요청했는가? (예: `database/creds/app-readonly`)
    
- `response.data.username`: 어떤 동적 사용자가 발급되었는가? (예: `v-app-readonly-pqr-789...`)
    

이는 정적 자격 증명으로는 불가능했던, _모든 데이터베이스 접근 시도_에 대한 100% 완전한 감사 추적을 제공합니다.

### C. 고가용성(HA) 및 장애 모드

프로덕션 시스템은 장애를 가정하고 설계되어야 합니다.

- MySQL이 다운된다면?
    
    Vault가 MySQL에 연결할 수 없으므로 database/creds 요청이 실패합니다. Vault는 동적 사용자를 생성할 수 없습니다. 애플리케이션은 재시도(Retry) 및 서킷 브레이커(Circuit Breaker) 패턴으로 이를 처리해야 합니다.
    
- **Vault가 다운된다면? (더 심각함)**
    
    1. **신규 발급 실패:** 새로운 애플리케이션이나 기존 앱의 재시작 시 _신규 자격 증명을 발급받지 못합니다._
        
    2. **폐기 실패 (중요):** 이것이 핵심 위험입니다. `default_ttl`이 1시간인 리스가 만료되더라도, Vault(폐기의 주체)가 다운되었기 때문에 MySQL의 동적 사용자는 _삭제되지 않습니다._
        
    3. 이 고아 사용자들은 Vault가 다시 온라인 상태가 되어 만료된 리스 백로그를 처리할 때까지 MySQL에 _활성 상태_로 남아있게 됩니다.
        

이러한 위험은 단 하나의 결론으로 귀결됩니다. 프로덕션 환경에서 데이터베이스 동적 시크릿을 운영하려면, **고가용성 Vault 클러스터(예: 3-5개 노드)가 필수 전제조건**입니다. 단일 Vault 서버로 프로덕션 DB를 관리하는 것은 또 다른 단일 장애점(SPOF)을 만드는 행위입니다.

## VII. 결론: 새로운 보안 태세의 확립

이 보고서의 가이드를 따름으로써, 조직의 데이터베이스 보안 태세는 근본적으로 변화합니다.

**이전 (Before):**

- 수명이 긴 정적 `DB_USER` / `DB_PASS`.
    
- 설정 파일, Git 저장소, CI/CD 변수에 저장된 자격 증명.
    
- 공유 계정 및 과도한 권한.
    
- 수동적이고 고통스러운 암호 순환 프로세스.
    
- "누가" 이 자격 증명을 사용했는지 알 수 없는, 불가능한 감사.
    

**이후 (After):**

1. **정적 자격 증명 소멸:** 애플리케이션 레벨에서는 더 이상 정적 DB 자격 증명이 존재하지 않습니다.
    
2. **완전 자동화된 라이프사이클:** "요람에서 무덤까지" 모든 자격 증명이 자동으로 생성, 갱신, 폐기됩니다.
    
3. **세분화된 권한:** Vault 역할(`app-readonly`, `schema-migration` 등)을 통해 MySQL 권한이 세분화됩니다.
    
4. **엄격한 접근 제어:** Vault ACL 정책을 통해 "누가" "어떤 역할"을 요청할 수 있는지 중앙에서 제어합니다.
    
5. **100% 감사 가능성:** _모든_ 데이터베이스 접근 _요청_이 명확한 신원과 함께 로깅됩니다.
    

이는 단순한 도구의 도입이 아니라, 제로 트러스트(Zero Trust) 원칙을 데이터베이스 접근에 적용하는 현대적인 아키텍처로의 진화입니다.