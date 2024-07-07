### 상태 변수를 통한 조회
`show global status like 'wsrep_%';` : 쓰기 세트 관련 상태 변수 조회
##### 상태 변수 설명
###### 클러스터 무결성 확인
* `wsrep_cluster_state_uuid` : 클러스터 상태 uuid, 노드가 클러스터의 일부인지 확인 가능하다.
* `wsrep_cluster_conf_id` : 클러스터의 변경 발생 횟수
* `wsrep_cluster_size` : 클러스터를 이루고 있는 노드 멤버의 수
* `wsrep_cluster_status` : 클러스터가 primary 상태인지 확인 가능하다.
###### 노드 상태 확인
* `wsrep_ready` : 노드가 쿼리를 받아 들일 수 있는 상태인지 나타낸다.
* `wsrep_connected` : 노드가 다른 노드와 네트워크로 연결되어 있는지 여부 표시
* `wsrep_local_state_comment` : 노드의 상태(Joining, Waiting on SST, Joined, Synced or Donor) 표기
###### 복제 상태 확인
* `wsrep_local_recv_queue_avg/max/min` : 마지막 상태 쿼리 수신 후 로컬 수신 대기열의 크기
* `wsrep_flow_control_paused` : flush status 이후 노드가 중지된 시간(분 단위) 표기
* `wsrep_cert_deps_distance` : 노드가 병렬로 적용할 수 있는 가장 낮은 seqno 와 가장 높은 seqno 사이의 평균 거리(높을 수록 병렬 처리 능력이 높다)
###### 느린 네트워크 문제 감지
* `wsrep_local_send_queue_avg` : flush status 쿼리 실행 이후 전송 대기열 길이의 평균 표시

### DB 서버 로그
my.cnf 의 설정 파일을 통해 설정 가능
```
# wsrep Log Options 
wsrep_log_conflicts=ON # 충돌 로그에 대한 로깅 활성화
wsrep_provider_options="cert.log_conflicts=ON" # 복제 중 인증 실패에 대한 로깅 활성화
wsrep_debug=ON # 서버 로그에 디버깅 정보 활성화
```

### 갈레라 관리자
갈레라 관리자 다운로드 : `curl https://galeracluster.com/galera-manager/gm-installer`
갈레라 관리자 설치 : `chmod a+x gm-installer && sudo ./gm-installer install`
갈레라 관리자가 설치된 서버의 80, 8081, 8082포트를 허용해줘야 하며 80포트(http)를 통해 web으로 갈레라 클러스터를 관리할 수 있다.