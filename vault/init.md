### 1단계: 호스트에 디렉터리 구조 생성

먼저 `podman-compose` 파일과 각 노드의 설정 및 데이터를 저장할 디렉터리를 생성합니다. 이 구조는 매우 중요합니다.

Bash

```
# 메인 디렉터리 생성
mkdir ha-vault
cd ha-vault

# 각 노드를 위한 디렉터리 생성
mkdir vault1/
mkdir vault2/
mkdir vault3/
```

이 단계가 끝나면 다음과 같은 구조가 됩니다.

```
ha-vault/
├── vault1/
├── vault2/
└── vault3/
```

(이후 단계에서 각 `vaultX` 디렉터리 안에 `config.hcl` 파일을 생성할 것입니다.)

### 2단계: `podman-compose.yaml` 파일 작성

`ha-vault` 디렉터리 최상위에 `podman-compose.yaml` 파일을 생성합니다.

YAML

```
version: '3.8'

services:
  vault1:
    image: docker.io/hashicorp/vault:latest  # [1, 2]
    container_name: vault1
    hostname: vault1
    command: server
    cap_add:
      - "IPC_LOCK"  #
    ports:
      - "8200:8200" # 호스트 포트 8200 -> vault1
    volumes:
      - ./vault1/config.hcl:/vault/config/config.hcl:ro
      - ./vault1/data:/vault/data:rw,Z  # :Z는 SELinux 레이블링 [3, 4]
    networks:
      - vault-net
    restart: always

  vault2:
    image: docker.io/hashicorp/vault:latest
    container_name: vault2
    hostname: vault2
    command: server
    cap_add:
      - "IPC_LOCK"
    ports:
      - "8201:8200" # 호스트 포트 8201 -> vault2
    volumes:
      - ./vault2/config.hcl:/vault/config/config.hcl:ro
      - ./vault2/data:/vault/data:rw,Z
    networks:
      - vault-net
    restart: always

  vault3:
    image: docker.io/hashicorp/vault:latest
    container_name: vault3
    hostname: vault3
    command: server
    cap_add:
      - "IPC_LOCK"
    ports:
      - "8202:8200" # 호스트 포트 8202 -> vault3
    volumes:
      - ./vault3/config.hcl:/vault/config/config.hcl:ro
      - ./vault3/data:/vault/data:rw,Z
    networks:
      - vault-net
    restart: always

networks:
  vault-net:
    driver: bridge #
```

### 3단계: 각 노드의 HCL 설정 파일 작성

이제 각 노드의 고유한 `config.hcl` 파일을 생성합니다. 이 파일들은 `podman-compose.yaml`이 참조하는 경로(예: `./vault1/config.hcl`)에 위치해야 합니다.

**경고:** 이 예제는 `tls_disable = true`를 사용합니다. 이는 **테스트 및 개발용**이며, 프로덕션 환경에서는 절대 사용해서는시
```
# 리스너 (API 및 UI 트래픽) - TLS 활성화
listener "tcp" { 
  address = "0.0.0.0:8200" 
  
  # 프로덕션 필수: TLS 인증서 경로 
  tls_cert_file = "/etc/vault.d/tls/vault.crt" 
  tls_key_file = "/etc/vault.d/tls/vault.key" 
  
  # 보안 강화: 최소 TLS 버전 지정 
  tls_min_version = "tls12" 
}
```

### 4단계: 클러스터 시작 및 초기화

모든 파일이 준비되었다면, `ha-vault` 디렉터리에서 다음 명령을 실행합니다.

**1. 컨테이너 시작:**
```
$ podman-compose up -d
```

이제 3개의 Vault 컨테이너가 실행 중입니다. 하지만 모두 '봉인(Sealed)' 상태입니다.

**2. 클러스터 초기화 (vault1에서 실행):** `podman exec`를 사용해 `vault1` 컨테이너 내부에서 `init` 명령을 실행합니다.  
```
# VAULT_ADDR은 podman-compose.yaml에서 노출한 호스트 포트를 사용합니다.
$ export VAULT_ADDR='http://127.0.0.1:8200'

# vault1 컨테이너에서 초기화를 실행합니다.
# (컨테이너 이름을 사용해도 됩니다: podman exec -it vault1 vault operator init)
$ vault operator init

# [중요] 출력된 Unseal Key 5개와 Initial Root Token 1개를 
#       안전한 곳에 즉시 복사하여 저장합니다.
```

**3. 모든 노드 봉인 해제 (Unseal):** 클러스터가 작동하려면 임계값(기본값 3개) 이상의 노드가 봉인 해제되어야 합니다. 초기화 시 받은 5개의 Unseal Key 중 3개를 사용하여 **3개의 모든 노드**를 각각 봉인 해제해야 합니다.

- **vault1 봉인 해제 (호스트 포트 8200):**    
    ```
    $ vault operator unseal <Unseal Key 1>
    $ vault operator unseal <Unseal Key 2>
    $ vault operator unseal <Unseal Key 3>
    ```
    
- **vault2 봉인 해제 (호스트 포트 8201):**    
    ```
    # VAULT_ADDR을 vault2의 호스트 포트로 변경합니다.
    $ export VAULT_ADDR='http://127.0.0.1:8201'
    
    $ vault operator unseal <Unseal Key 1>
    $ vault operator unseal <Unseal Key 2>
    $ vault operator unseal <Unseal Key 3>
    ```
    
- **vault3 봉인 해제 (호스트 포트 8202):**    
    ```
    # VAULT_ADDR을 vault3의 호스트 포트로 변경합니다.
    $ export VAULT_ADDR='http://127.0.0.1:8202'
    
    $ vault operator unseal <Unseal Key 1>
    $ vault operator unseal <Unseal Key 2>
    $ vault operator unseal <Unseal Key 3>
    ```
    

**4. 클러스터 상태 확인:** 모든 노드가 봉인 해제되면, HCL 파일의 `retry_join` 설정 덕분에 자동으로 클러스터가 구성됩니다. `vault1`에서 상태를 확인합니다.
```
# VAULT_ADDR을 다시 vault1로 설정
$ export VAULT_ADDR='http://127.0.0.1:8200'

# Root Token으로 로그인
$ vault login <Initial Root Token>

# Raft 피어 목록 확인
$ vault operator raft list-peers
#
Node      Address          State       Voter
----      -------          -----       -----
vault1    vault1:8201      leader      true
vault2    vault2:8201      follower    true
vault3    vault3:8201      follower    true
```