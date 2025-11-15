# Vault CLI 명령어를 HTTP API로 직접 변환하기 위한 전문가 가이드

## I. 서론: Vault CLI에서 HTTP API로의 아키텍처 전환

HashiCorp Vault는 현대적인 인프라스트럭처에서 시크릿 관리를 위한 핵심 도구로 자리 잡았습니다. 대부분의 사용자는 `vault` 커맨드 라인 인터페이스(CLI)를 통해 Vault와 상호작용하지만, Vault의 진정한 힘과 유연성은 HTTP API에서 나옵니다. Vault CLI는 이 HTTP API를 감싼 "씬 래퍼(thin wrapper)"에 불과하며, CLI의 모든 명령어는 내부적으로 하나 이상의 API 호출로 변환됩니다.   

이러한 API 우선(API-First) 설계는 Vault CLI 바이너리를 설치할 필요 없이 `curl`과 같은 기본 HTTP 클라이언트만으로도 Vault의 모든 기능을 활용할 수 있음을 의미합니다.   

### 1.1. API 직접 호출의 전략적 중요성

CLI 대신 API를 직접 호출하는 것은 단순한 기술적 선호의 문제가 아니라, 자동화 및 보안 아키텍처 설계의 핵심적인 결정입니다.

- **CI/CD 파이프라인의 효율성:** `vault` 바이너리를 다운로드하고, 설치하며, 인증 토큰을 설정하는 오버헤드 없이, `alpine/curl`과 같은 경량 컨테이너 이미지를 사용하여 파이프라인 스크립트에서 직접 시크릿을 주입할 수 있습니다.
    
- **불변성 및 최소 권한의 원칙:** `vault` 바이너리 자체가 존재하지 않는 'Distroless' 또는 최소한의 스크래치(Scratch) 기반 컨테이너 내부에서 실행되는 애플리케이션이 Vault와 직접 통신할 수 있습니다. 이는 공격 표면을 최소화하는 데 기여합니다.
    
- **정교한 프로그래밍 방식의 제어:** API를 직접 사용하면, CLI가 제공하는 단순한 문자열 파싱(parsing)을 넘어서는 제어가 가능합니다. HTTP 상태 코드 , 응답 헤더, 그리고 `{"errors":[...]}` 와 같은 상세한 JSON 오류 메시지를 기반으로 정교한 오류 처리, 재시도 로직, 조건부 워크플로우를 구현할 수 있습니다.   
    

### 1.2. 궁극의 학습 도구: `-output-curl-string` 플래그 활용

Vault CLI 명령어를 API 호출로 변환하는 방법을 배우는 가장 강력하고 정확한 방법은 Vault CLI 자체에 내장된 `-output-curl-string` 플래그를 사용하는 것입니다. 이 플래그는 명령어를 실제로 실행하는 대신, 해당 작업을 수행하는 `curl` 명령어를 표준 출력(stdout)으로 인쇄합니다.   

예제 1: KVv2 시크릿 쓰기    

- **CLI 명령어:**
    
        
    ```
    vault kv put -output-curl-string secret/test foo=bar
    ```
    
- **생성된 `curl` 명령어:**
    
        
    ```
    curl -X PUT -H "X-Vault-Request: true" \
         -H "X-Vault-Token: $(vault print token)" \
         -d '{"data":{"foo":"bar"},"options":{}}' \
         http://127.0.0.1:8200/v1/secret/data/test
    ```
    

예제 2: 마운트된 시크릿 엔진 목록 보기    

- **CLI 명령어:**
    
        
    ```
    vault secrets list -output-curl-string
    ```
    
- **생성된 `curl` 명령어:**
    
        
    ```
    curl -H "X-Vault-Request: true" \
         -H "X-Vault-Token: $(vault print token)" \
         http://127.0.0.1:8200/v1/sys/mounts
    ```
    

예제 3: KVv2 시크릿 읽기 (특정 필드)    

- **CLI 명령어:**
    
        
    ```
    vault kv get -output-curl-string -field=server kv/myapp/database
    ```
    
- **생성된 `curl` 명령어:**
    
        
    ```
    curl -H "X-Vault-Request: true" \
         -H "X-Vault-Token: $(vault print token)" \
         http://vaultserver.dev:8200/v1/kv/data/myapp/database
    ```
    

이 도구는 단순한 번역기를 넘어, CLI가 사용자를 위해 _추상화하는 모든 숨겨진 로직_을 노출합니다. 예를 들어, `vault kv put secret/test...`  명령이 API 경로 `.../v1/secret/data/test` 로 변환되는 것을 통해, KVv2 쓰기 작업 시 CLI가 자동으로 `/data/` 경로를 삽입한다는 사실을 명확히 알 수 있습니다. 이는 API의 구조를 학습하는 강력한 디버깅 도구입니다.   

### 1.3. 모든 `curl` 요청의 기본 구성 요소

Vault API와 `curl`을 사용하여 상호작용할 때, 다음 요소들은 모든 요청의 기본이 됩니다.

- **`VAULT_ADDR` (대상 주소):** `curl` 명령어는 `VAULT_ADDR` 환경 변수를 자동으로 읽지 않습니다. 따라서 `curl` 명령어에는 항상 `http://127.0.0.1:8200`와 같은 전체 Vault 서버 URL을 명시적으로 포함해야 합니다. 자동화 스크립트에서는 `curl $VAULT_ADDR/v1/...` 형식을 사용하는 것이 일반적입니다.   
    
- **`X-Vault-Token` 헤더:** Vault에 대한 모든 인증된 요청은 `X-Vault-Token` HTTP 헤더에 유효한 토큰을 포함해야 합니다.   
    
- **`/v1/` API 접두사:** Vault의 모든 HTTP API 경로는 `/v1/` 접두사로 시작합니다.   
    
- **타임아웃 처리:** `VAULT_CLIENT_TIMEOUT` 환경 변수는 Vault CLI 전용입니다. `curl`을 사용할 때는 서버가 응답하기까지 클라이언트가 대기하는 시간을 제어하기 위해 `--max-time <seconds>` 플래그(예: `--max-time 20`)를 사용하여 동일한 클라이언트 측 타임아웃 효과를 구현해야 합니다.   
    

---

## II. Vault 클러스터 운영 및 상태 관리 (System Backend API)

자동화된 클러스터 관리, 모니터링, 부트스트래핑(bootstrapping) 스크립트를 작성할 때, `sys` 백엔드 엔드포인트는 가장 먼저 호출되는 API입니다.

### 2.1. 상태 확인 (Health Check): `GET /v1/sys/health`

이 엔드포인트는 Vault 인스턴스의 현재 상태를 확인하는 데 사용되며, 특히 로드 밸런서(LB)나 서비스 디스커버리 도구(예: Consul)와의 통합에 필수적입니다.

- **CLI 명령어 (부분적):** `vault status`
    
- **API 엔드포인트:** `GET /v1/sys/health`    
    

**`curl` 예제 (비인증):**

```
curl http://127.0.0.1:8200/v1/sys/health
```

**핵심 분석: HTTP 상태 코드를 통한 모니터링** 이 엔드포인트는 Consul HTTP 상태 확인과 동일한 시맨틱(semantics)을 갖도록 설계되었습니다. 자동화된 모니터링 시스템은 JSON 응답 본문(body)을 파싱하기보다, 반환되는 HTTP 상태 코드를 확인해야 합니다.   

- **`200`:** 초기화됨(Initialized), 봉인 해제됨(Unsealed), 활성(Active) 상태    
    
- **`429`:** 봉인 해제됨(Unsealed), 대기(Standby) 상태    
    
- **`501`:** 초기화되지 않음(Not Initialized)    
    
- **`503`:** 봉인됨(Sealed)    
    

**`curl` 예제 (인증):**

```
curl -H "X-Vault-Token: s.xxxxxxxxxxxx" \
     http://127.0.0.1:8200/v1/sys/health
```

인증된 상태로 `/sys/health` 엔드포인트를 호출하면 , 비인증 호출 시의 기본 정보 와 달리, `version`, `cluster_name`, `cluster_id`, `license` (Enterprise) 등 운영자에게 유용한 상세 정보가 포함된 풍부한 JSON 응답을 반환합니다. 이는 `/sys/health` 엔드포인트가 (1) 외부 모니터링 시스템을 위한 '상태 코드' 반환, (2) 내부 운영자를 위한 '상세 정보' 반환이라는 두 가지 용도로 사용됨을 보여줍니다.   

### 2.2. 봉인 상태 확인 (Seal Status): `GET /v1/sys/seal-status`

이 엔드포인트는 Vault의 봉인(seal) 상태와 관련된 상세 정보(예: Shamir 키 임계값, 진행 상태)를 반환합니다.

- **CLI 명령어:** `vault status`    
    
- **API 엔드포인트:** `GET /v1/sys/seal-status`    
    
- **`curl` 예제:**
    
        
    ```
    curl -H "X-Vault-Token: $(vault print token)" \
         https://some.host.com:8200/v1/sys/seal-status
    ```
    
- 응답 예시 :   
    
    JSON
    
    ```
    {"type":"shamir","initialized":true,"sealed":true,"t":3,"n":5,"progress":0,"nonce":"...","version":"1.6.1"}
    ```
    

이 API는 `curl`로의 전환 시 가장 흔히 발생하는 네트워크 및 TLS 관련 문제를 진단하는 첫 번째 관문이 됩니다.

- 오류 1: `http: server gave HTTP response to HTTPS client`    
    
    - **원인:** `curl`이 `https://...` (HTTPS)로 요청했으나, Vault 리스너가 `tls_disable=1` (HTTP)로 구성되어 있습니다. 스크립트의 URL과 Vault 리스너 설정이 불일치합니다.
        
- 오류 2: `x509: certificate signed by unknown authority`    
    
    - **원인:** Vault가 사설 CA 또는 자체 서명 인증서를 사용 중이며, `curl`을 실행하는 시스템이 해당 CA를 신뢰하지 않습니다.
        
    - **해결책:** (보안) `curl --cacert /path/to/ca.crt...`로 CA 인증서를 명시하거나, (비보안/테스트용) `curl -k...`로 검증을 건너뜁니다.
        
- 오류 3: `no route to host`    
    
    - **원인:** `curl` 실행 환경과 Vault 서버 간의 근본적인 네트워크 경로(방화벽, 라우팅 테이블)가 존재하지 않습니다.
        

### 2.3. 서버 봉인 해제 (Unseal): `POST /v1/sys/unseal`

초기화된 Vault 서버가 재시작되면 봉인 상태가 되며, 운영자가 봉인 키(unseal key)를 제공하여 봉인을 해제해야 합니다.

- **CLI 명령어:** `vault operator unseal`    
    
- **API 엔드포인트:** `POST /v1/sys/unseal`    
    
- **`curl` 페이로드 (단순):**
    
        
    ```
    curl -X POST --data '{"key": "YOUR_UNSEAL_KEY"}' \
         http://127.0.0.1:8200/v1/sys/unseal
    ```
    

CLI와 마찬가지로, 봉인 해제 임계값(Threshold)에 도달할 때까지(예: 3/5) 서로 다른 봉인 키를 사용하여 이 API 엔드포인트를 여러 번 호출해야 합니다. API 응답의 `"progress": 1`과 같은 값을 모니터링하여 진행 상태를 확인할 수 있습니다.

**치명적인 보안 위험 및 전문가 수준의 해결책** `vault operator unseal` CLI 명령어는 의도적으로 키 입력을 위한 프롬프트(prompt)를 사용하여, 봉인 키가 셸 히스토리(`.bash_history`)에 평문으로 남는 것을 방지합니다.   

하지만 위의 단순한 `curl` 예제 는 봉인 키를 명령어 인수로 직접 노출시켜, `history` 명령어는 물론 `ps aux`와 같은 프로세스 목록에도 키가 노출될 수 있는 심각한 보안 사고를 유발합니다.   

자동화 스크립트에서 이 위험을 완화하는 전문가 수준의 `bash` 패턴은 다음과 같습니다.   

**전문가 수준의 `curl` 봉인 해제 (보안):**

```
# /my-unseal-key 파일에 키가 저장되어 있다고 가정
cat /my-unseal-key | curl -X PUT \
     -d @<(echo "{\"key\":\"$(cat /dev/stdin)\"}") \
     $VAULT_ADDR/v1/sys/unseal
```

**위 명령어 상세 분석:**

1. `cat /my-unseal-key |...`: 키 파일의 내용을 읽어 표준 입력(stdin) 파이프로 전달합니다.
    
2. `... $(cat /dev/stdin)...`: 파이프로 전달된 입력을 읽습니다.
    
3. `echo "{\"key\":\"...\"}"`: 읽어들인 키를 포함하는 JSON 페이로드를 동적으로 생성합니다.
    
4. `@<(...)`: `bash`의 '프로세스 치환(Process Substitution)'입니다. `echo` 명령의 출력을 임시 파일처럼 취급하여, `curl`의 `-d @...` (파일에서 데이터를 읽어 POST) 옵션에 전달합니다.
    
5. **결과:** 봉인 키 자체가 셸 히스토리에 기록되거나 `ps` 명령어의 인수로 노출되지 않습니다. 이는 CLI의 보안 프롬프트와 동등한 수준의 보안을 API 기반으로 달성하는 방법입니다.
    

---

## III. 핵심 전략: KV Secrets Engine (Version 2) API 마스터하기

KVv2(Key-Value Version 2) 시크릿 엔진은 Vault API 사용의 가장 일반적인 사례입니다. KVv2의 핵심은 시크릿의 *내용(data)*과 *메타데이터(metadata)*를 관리하는 API 경로가 완전히 분리되어 있다는 점입니다.

### 3.1. KVv2 API의 핵심 아키텍처: `/data/`와 `/metadata/`의 분리

KVv2 API를 처음 사용하는 개발자가 겪는 가장 큰 혼란의 원천은 이 경로 분리입니다.   

- **`/data/` 경로:** 시크릿의 _내용_을 다룹니다. (예: 특정 버전의 K-V 쌍 읽기, 새 버전 쓰기, 최신 버전 삭제).
    
- **`/metadata/` 경로:** 시크릿의 _컨테이너_를 다룹니다. (예: 버전 목록 보기, 시크릿 영구 파괴, ACL 관리, 메타데이터 수정).
    

또한, `v1/secret/data/...`와 같은 경로에서 `secret`은 API 키워드가 아니라, KVv2 엔진이 마운트된 *경로(mount path)*를 나타내는 변수입니다. 만약 `vault secrets enable -path=dev kv` 로 마운트했다면, API 경로는 `v1/dev/data/...`가 됩니다. 본 가이드에서는 일반성을 위해 마운트 경로를 `<mount>`로 표기합니다.   

### 3.2. 시크릿 생성 및 업데이트: `POST /v1/<mount>/data/<path>`

- **CLI 명령어:** `vault kv put <mount>/<path> foo=bar`    
    
- **API 엔드포인트:** `POST /v1/<mount>/data/<path>`  (또는 `PUT` )   
    
- **`curl` 예제:**
    
        
    ```
    # 'secret' 마운트의 'baz' 경로에 시크릿 생성
    curl -X POST -H "X-Vault-Token: s.xxxxxxxxxxxx" \
         -H "Content-Type: application/json" \
         -d '{"data":{"value":"bar"}}' \
         http://127.0.0.1:8200/v1/secret/data/baz
    ```
    

**핵심 페이로드 분석:** `{"data": {"key": "value"}}` 페이로드는 _반드시_ `data`라는 최상위 키로 래핑(wrapping)되어야 합니다. 이것이 KVv1과의 핵심적인 차이점입니다. `vault kv put`의 `curl` 출력 은 `{"data":{...}, "options":{}}` 구조를 보여주며, `options` 객체는 CAS(Check-And-Set)와 같은 고급 쓰기 옵션에 사용됩니다.   

### 3.3. 최신 버전 시크릿 읽기: `GET /v1/<mount>/data/<path>`

- **CLI 명령어:** `vault kv get <mount>/<path>`    
    
- **API 엔드포인트:** `GET /v1/<mount>/data/<path>`
    
- **`curl` 예제:**
    
        
    ```
    curl -H "X-Vault-Token: s.xxxxxxxxxxxx" \
         http://127.0.0.1:8200/v1/kv/data/tool-common/dev/svc-DeployDev
    ```
    

**응답 구조 분석: `jq`가 필수적인 이유 (data.data)** `vault kv get` CLI 명령어는 시크릿 값만 깔끔하게 반환하지만 , `curl` API 호출의 원시(raw) JSON 응답은 다음과 같은 이중 중첩 구조를 가집니다.   

**응답 예시:**

JSON

```
{
  "request_id": "...",
  "lease_id": "",
  "data": {
    "data": {
      "password": "...",
      "username": "..."
    },
    "metadata": {
      "created_time": "...",
      "version": 3,
     ...
    }
  }
}
```

스크립트에서 실제 시크릿 값(예: `password`)에 접근하려면 **`.data.data`**라는 경로를 파싱해야 합니다.   

- 첫 번째 `.data`: Vault API 응답의 표준 래퍼입니다.
    
- 두 번째 `.data.data`: KVv2 시크릿의 실제 K-V 페이로드입니다.
    
- `.data.metadata`: 버전 번호, 생성 시간 등 시크릿의 메타데이터입니다.
    

`curl`과 `jq`를 사용한 값 추출 :   

```
curl --silent -H "X-Vault-Token:...".../v1/kv/data/path | \
jq -r.data.data.password
```

### 3.4. 시크릿 목록화 (Listing): `LIST /v1/<mount>/metadata/<path>`

- **CLI 명령어:** `vault kv list <mount>/<path>`    
    
- **API 엔드포인트:**
    
    1. `LIST /v1/<mount>/metadata/<path>`  (권장)   
        
    2. `GET /v1/<mount>/metadata/<path>?list=true`    
        

**`curl` 예제:**

```
# 'GET' 메서드와 'list=true' 쿼리 파라미터 사용
curl -H "X-Vault-Token:..." \
     "https://vault-tst.com/v1/kv-v2/metadata/folder?list=true" | jq.data
```

**핵심 경로 분석:** `LIST` 작업은 시크릿의 _내용_이 아닌 _존재_를 확인하는 것이므로, `/data/`가 아닌 **`/metadata/`** 경로를 사용해야 합니다. 이는 KVv2 API 사용 시 가장 흔한 실수 중 하나입니다. 응답은 `{"data": {"keys": ["key-a", "key-b"]}}`  형식입니다.   

### 3.5. 시크릿 버전 관리 및 삭제 전략

KVv2는 CLI의 `delete`와 `destroy`에 해당하는 두 가지 삭제 개념을 API로 제공합니다.   

#### 3.5.1. 최신 버전 삭제 (Soft Delete): `DELETE /v1/<mount>/data/<path>`

- **CLI 명령어:** `vault kv delete <mount>/<path>`    
    
- **API 엔드포인트:** `DELETE /v1/<mount>/data/<path>`
    
- **`curl` 예제:**
    
        
    ```
    curl -X DELETE -H "X-Vault-Token:..." \
         https://vault.example.com/v1/secret/data/creds
    ```
    
- **분석:** 이 작업은 최신 버전을 "삭제됨"으로 표시(mark)할 뿐, 데이터를 스토리지에서 제거하지 않습니다. `vault kv undelete` CLI 또는 API로 복구할 수 있습니다.
    

#### 3.5.2. 특정 버전 영구 파괴 (Destroy): `POST /v1/<mount>/destroy/<path>`

- **CLI 명령어:** `vault kv destroy -versions=<n> <mount>/<path>`    
    
- **API 엔드포인트:** `POST /v1/<mount>/destroy/<path>`    
    
- **`curl` 예제:**
    
        
    ```
    # 'creds' 시크릿의 버전 2와 3을 영구적으로 파괴
    curl -X POST -H "X-Vault-Token:..." \
         -d '{"versions":[1, 2]}' \
         https://vault.example.com/v1/secret/destroy/creds
    ```
    
- **분석:** 이 작업은 데이터를 스토리지에서 영구적으로 삭제하며 복구할 수 없습니다. 이는 `/data/`가 아닌 별도의 `/destroy/` 엔드포인트를 사용합니다.
    

---

## IV. 레거시 및 단순 사용: KV Secrets Engine (Version 1) API

KVv1 시크릿 엔진은 버전 관리 기능이 없는 단순한 K-V 저장소입니다. API 구조 또한 KVv2보다 훨씬 단순합니다.

### 4.1. KVv1 API 아키텍처: 단일 경로 모델

KVv1은 `/data/`와 `/metadata/`의 구분이 없습니다. 모든 읽기, 쓰기, 목록 보기, 삭제 작업이 `v1/<mount>/<path>`라는 단일 경로에서 발생합니다.   

### 4.2. 시크릿 쓰기 (KVv1): `POST /v1/<mount>/<path>`

- **CLI 명령어:** `vault write <mount>/<path> foo=bar`
    
- **API 엔드포인트:** `POST /v1/<mount>/<path>`    
    
- **`curl` 예제:**
    
        
    ```
    curl -X POST -H "X-Vault-Token: s.xxxxxxxxxxxx" \
         -d '{"password":"mypassword"}' \
         https://myvault.mydomain.com:8200/v1/secret/path
    ```
    
- **페이로드 비교:** KVv2의 `{"data":{"key":"value"}}` 와 달리, KVv1의 페이로드는 `{"key":"value"}` 로, 래핑(wrapping) 계층이 없습니다.   
    

### 4.3. 시크릿 읽기 (KVv1): `GET /v1/<mount>/<path>`

- **CLI 명령어:** `vault read <mount>/<path>`
    
- **API 엔드포인트:** `GET /v1/<mount>/<path>`    
    
- **`curl` 예제:**
    
        
    ```
    curl -H "X-Vault-Token: s.xxxxxxxxxxxx" \
         https://my-vault.company.com/v1/secret/mysuperuberrandomsecret
    ```
    
- **응답 분석:** 응답은 `{"data": {"key": "value"}}`  구조입니다. KVv2의 `.data.data` 와 달리, `.data`만 파싱하면 시크릿 값에 접근할 수 있습니다.   
    
    - `... | jq -r.data.password`
        

### 4.4. 시크릿 목록 (KVv1): `GET /v1/<mount>/<path>?list=true`

- **CLI 명령어:** `vault list <mount>/<path>`
    
- **API 엔드포인트:** `GET /v1/<mount>/<path>?list=true`  또는 `LIST /v1/<mount>/<path>`    
    
- **`curl` 예제:**
    
        
    ```
    curl -H "X-Vault-Token:..." -X GET \
         "http://10.16.12.111:8200/v1/secret/path/?list=true"
    ```
    

---

## V. 고급 자동화: 동적 시크릿(Dynamic Secrets) API 활용

Vault의 가장 강력한 기능 중 하나는 데이터베이스 자격 증명이나 클라우드 API 키와 같은 시크릿을 _동적으로_ 생성하는 것입니다.

### 5.1. 동적 데이터베이스 자격 증명 요청

애플리케이션이나 CI/CD 잡이 시작될 때마다 고유하고 수명(TTL)이 짧은 데이터베이스 자격 증명을 발급받아 사용하는 것이 모범 사례입니다.   

- **CLI 명령어:** `vault read database/creds/<role-name>`    
    
- **API 엔드포인트:** `GET /v1/database/creds/<role-name>`    
    
- **`curl` 예제:**
    
        
    ```
    # 'database' 엔진에 정의된 'my-role'에 대한 자격 증명 요청
    curl -H "X-Vault-Token: s.xxxxxxxxxxxx" \
         http://127.0.0.1:8200/v1/database/creds/my-role
    ```
    

`vault read`  CLI 명령어는 HTTP `GET` 메서드와 시맨틱이 일치하며, 공식 API 문서 역시 'Generate credentials' API의 샘플 요청으로 `GET /v1/database/creds/my-role`을 명확히 보여줍니다.   

일부 2차 자료에서는 `POST` 메서드를 사용하라고 제안하기도 하지만 , 이는 이 작업이 REST 시맨틱 관점에서 '새로운 리소스(데이터베이스 사용자) 생성'에 해당하기 때문일 수 있습니다. 혼동을 피하기 위해, 자동화 스크립트 작성 시에는 공식 문서 에 명시된 `GET` 메서드를 사용하는 것이 가장 안전하고 권장되는 방법입니다.   

**응답 분석 및 활용** API는 `GET` 요청에 대해 다음과 같은 JSON을 반환합니다.   

JSON

```
{
  "request_id": "...",
  "lease_id": "database/creds/my-role/...",
  "renewable": true,
  "lease_duration": 3600,
  "data": {
    "username": "v-temp-user-...",
    "password": "..."
  }
}
```

자동화 스크립트는 이 응답을 파싱하여 `.data.username`과 `.data.password`를 추출하고, 이를 애플리케이션의 환경 변수나 설정 파일에 주입해야 합니다. 또한, `.lease_id`를 저장해 두었다가 애플리케이션이 종료될 때 해당 자격 증명을 조기에 폐기(revoke)하는 데 사용할 수 있습니다.

---

## VI. 결론: API 우선(API-First) Vault 자동화를 위한 프레임워크

`curl`을 사용하여 Vault API와 직접 상호작용하는 것은 단순한 CLI 대체가 아닙니다. 이는 Vault를 독립적인 플랫폼으로 취급하고, 어떤 환경(CI 파이프라인, Distroless 컨테이너, 레거시 시스템)에서든 일관된 방식으로 자동화를 구현하는 **API 우선(API-First) 접근 방식**으로의 아키텍처 전환을 의미합니다.

CLI는 사용자의 편의를 위해 복잡한 API 로직(예: KVv2의 `/data/` 경로)을 추상화하는 편리한 도구이지만, API는 Vault의 모든 기능을 노출하는 **핵심 계약(core contract)**입니다. 자동화 및 통합 작업을 수행할 때는 이 핵심 계약에 직접 의존하는 것이 장기적으로 더 안정적이고 강력한 시스템을 구축하는 길입니다.

다음은 본 보고서에서 다룬 핵심적인 CLI 명령어를 `curl` API 호출로 변환하는 '로제타석(Rosetta Stone)' 테이블입니다.

### [표 1] Vault CLI-API 핵심 변환 매트릭스

|작업 (Operation)|Vault CLI 명령어 (예)|HTTP 메서드|API 엔드포인트 및 `curl` 옵션|핵심 페이로드 (Payload) / `jq` 경로|
|---|---|---|---|---|
|**서버 상태 확인**|`vault status` (Health)|`GET`|`/v1/sys/health`|(HTTP 상태 코드 200/429/503 확인)|
|**봉인(Seal) 상태**|`vault status` (Seal)|`GET`|`/v1/sys/seal-status`|`jq.sealed` (true/false)|
|**봉인 해제 (Unseal)**|`vault operator unseal`|`POST`|`/v1/sys/unseal`|`-d '{"key": "..."}'`|
|**KVv2 쓰기**|`vault kv put secret/foo a=b`|`POST` / `PUT`|`/v1/secret/data/foo`|`-d '{"data":{"a":"b"}}'`|
|**KVv2 읽기**|`vault kv get secret/foo`|`GET`|`/v1/secret/data/foo`|`jq -r.data.data.a`|
|**KVv2 목록**|`vault kv list secret/`|`LIST` / `GET`|`/v1/secret/metadata/?list=true`|`jq.data.keys`|
|**KVv2 삭제 (Soft)**|`vault kv delete secret/foo`|`DELETE`|`/v1/secret/data/foo`|(N/A)|
|**KVv2 파괴 (Destroy)**|`vault kv destroy -v=1 secret/foo`|`POST`|`/v1/secret/destroy/foo`|`-d '{"versions":[3]}'`|
|**KVv1 쓰기**|`vault write secret/foo a=b`|`POST` / `PUT`|`/v1/secret/foo`|`-d '{"a":"b"}'`|
|**KVv1 읽기**|`vault read secret/foo`|`GET`|`/v1/secret/foo`|`jq -r.data.a`|
|**동적 시크릿 읽기**|`vault read db/creds/my-role`|`GET`|`/v1/db/creds/my-role`|`jq.data` (username/password)|