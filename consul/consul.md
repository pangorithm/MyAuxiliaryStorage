```bash
# consul help
Usage: consul [--version] [--help] <command> [<args>]

Available commands are:
    acl             Interact with Consul's ACLs
    agent           Runs a Consul agent
    catalog         Interact with the catalog
    config          Interact with Consul's Centralized Configurations
    connect         Interact with Consul Connect
    debug           Records a debugging archive for operators
    event           Fire a new event
    exec            Executes a command on Consul nodes
    force-leave     Forces a member of the cluster to enter the "left" state
    info            Provides debugging information for operators.
    intention       Interact with Connect service intentions
    join            Tell Consul agent to join cluster
    keygen          Generates a new encryption key
    keyring         Manages gossip layer encryption keys
    kv              Interact with the key-value store
    leave           Gracefully leaves the Consul cluster and shuts down
    lock            Execute a command holding a lock
    login           Login to Consul using an auth method
    logout          Destroy a Consul token created with login
    maint           Controls node or service maintenance mode
    members         Lists the members of a Consul cluster
    monitor         Stream logs from a Consul agent
    operator        Provides cluster-level tools for Consul operators
    peering         Create and manage peering connections between Consul clusters
    reload          Triggers the agent to reload configuration files
    resource        Interact with Consul's resources
    rtt             Estimates network round trip time between nodes
    services        Interact with services
    snapshot        Saves, restores and inspects snapshots of Consul server state
    tls             Builtin helpers for creating CAs and certificates
    troubleshoot    CLI tools for troubleshooting Consul service mesh
    validate        Validate config files/directories
    version         Prints the Consul version
    watch           Watch for changes in Consul
```

### 설치  
```bash
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo <https://rpm.releases.hashicorp.com/RHEL/hashicorp.repo>
sudo yum -y install consul

```

### 설정파일 공식 문서  
[https://developer.hashicorp.com/consul/docs/agent](https://developer.hashicorp.com/consul/docs/agent)  
```bash
# 서버 설정 예시
node_name = "consul-server1"
server = true
bootstrap_expect = 3
datacenter = "dc1"
data_dir = "/opt/consul"
bind_addr = "0.0.0.0"
client_addr = "0.0.0.0"
retry_join = ["consul-server-ip", ...] # 다른 서버의 IP 주소
retry_join_wan = ["wan-cluster-consul-server-ip1", ...] # wan federation 설정

ui_config {
  enabled = true
}

acl {
  enabled = true
  default_policy = "allow"
  enable_token_persistence = true
}
```

### consul node 등록  
```bash
consul agent -config-dir=/etc/consul.d >> /var/log/consul.log &

# 리더 노드 확인
consul operator raft list-peers

# 리더 변경
consul operator raft transfer-leader
```

### 서비스 도메인 등록  
```bash
# consul 노드 수동 참여
consul join <consul서버ip>

# 등록된 노드 조회
consul members
consul catalog nodes
```

### 서비스 헬스체크 설정 공식문서  
[https://developer.hashicorp.com/consul/docs/services/usage/checks](https://developer.hashicorp.com/consul/docs/services/usage/checks)  
```bash
# 서비스 등록
consul services register /etc/consul.d/service-definition.json

# 등록된 서비스 조회
consul catalog services

# 서비스 노드 도메인 조회
# yum install -y bind-utils
dig @127.0.0.1 -p 8600 서비스명.service.consul

# 등록된 서비스 헬스체크
curl <http://127.0.0.1:8500/v1/health/checks/:service>
```

### wan federation 공식문서  
[https://developer.hashicorp.com/consul/tutorials/archive/federation-gossip-wan](https://developer.hashicorp.com/consul/tutorials/archive/federation-gossip-wan)  
```bash
# wan federation 멤버 조회
consul members -wan
```

### ACL 적용  
```bash
# ACL 마스터 토큰 생성
consul acl bootstrap

AccessorID:       08dd1174-cab7-09e4-af06-e1e2476a970a # 토큰을 식별하는 고유 ID
SecretID:         b630575f-e0e3-3326-f9f4-a1d0c4ac687c # 이게 토큰임(테스트용)
Description:      Bootstrap Token (Global Management)
Local:            false
Create Time:      2024-11-13 11:25:30.571102698 +0900 KST
Policies:
   00000000-0000-0000-0000-000000000001 - global-management
   
# ACL 토큰을 사용해서 요청하기
# 아래처럼 X-Consul-Token을 헤더로 넣어줘야 함
curl --header "X-Consul-Token: <your-token>" <http://localhost:8500/v1/agent/services>

# 기존 consul 명령어도 ACL 토큰을 사용해야 함
consul members -token=<your-token>
```

### ACL 공식문서  
[https://developer.hashicorp.com/consul/docs/security/acl](https://developer.hashicorp.com/consul/docs/security/acl)  
```bash
# dc2에서 아래 요청을 내릴 경우 dc1의 acl키가 필요함
curl --header "X-Consul-Token: b630575f-e0e3-3326-f9f4-a1d0c4ac687c" <http://127.0.0.1:8500/v1/catalog/services?dc=dc1>

# dc2의 acl 키 
ff43b3ac-235a-6fe7-530d-3acf43733945
```

### ACL 정책 설정  
```bash
# 정책 설정 파일 수정
vi /etc/consul.d/rules.hcl

# 정책 설정 파일 적용
consul acl policy create -name "정책명" -description "정책 주석" -rules @/etc/consul.d/rules.hcl -token "토큰값"

# 정책 삭제
consul acl policy delete -name "정책명" -token "토큰값"

```

### systemctl deamon 파일  
`vi /etc/systemd/system/consul.service`
```bash
[Unit]
Description=Consul Agent
After=network.target

[Service]
User=root
ExecStart=/usr/bin/consul agent -config-dir=/etc/consul.d -log-file=/var/log/consul.log
ExecReload=/bin/kill -HUP $MAINPID
KillMode=process
Restart=on-failure
RestartSec=5s

[Install]
WantedBy=multi-user.target
```