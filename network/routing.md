

### Linux Server Multiple Gateway

참조 [multiple_gateway_설정](https://atl.kr/dokuwiki/doku.php/multiple_gateway_%EC%84%A4%EC%A0%95)  

서버에 연결된 여러개의 네트워크 인터페이스로 외부에서 접속하기 위해서는 각 인터페이스 별로 게이트웨이를 설정해줄 필요가 있다.  

처음에는 static route로 시도하였으나 static route의 경우 외부로의 요청은 정상적으로 동작하였으나 외부에서 서버로의 route 경로를 올바르게 찾아주지 못하였고 이를 해결하기 위해서는 위에 각 인터페이스 별로 라우트 테이블을 생성하고 정책 라우트를 설정하여야 한다.

##### 설정 명령어 예시

```bash
ip route add <공인ip대역>/<subnet> dev eno1 table 1
ip route add default via <공인게이트웨이ip> dev eno1 table 1
ip rule add iif eno1 priority 100 table 1
ip rule add from <현재서버공인ip> priority 100 table 1

ip route add <사설ip대역>/<subnet> dev eno3 table 3
ip route add default via <사설게이트웨이ip> dev eno3 table 3
ip rule add iif eno3 priority 100 table 3
ip rule add from <현재서버사설ip> priority 100 table 3
```

##### 재부팅 시 자동 적용 설정 (설정하지 않을경우 재부팅 시 설정이 초기화되므로 필수)

`vi /etc/NetworkManager/dispatcher.d/10-multiple-gw.sh`  

```bash
#!/bin/bash

INTERFACE=$1
ACTION=$2

if [ "$INTERFACE" == "eno1" ]; then
    if [ "$ACTION" == "up" ]; then
      ip route add <공인ip대역>/<subnet> dev eno1 table 1
      ip route add default via <공인게이트웨이ip> dev eno1 table 1
      ip rule add iif eno1 priority 100 table 1
      ip rule add from <현재서버ip> priority 100 table 1
    fi
fi

if [ "$INTERFACE" == "eno3" ]; then
    if [ "$ACTION" == "up" ]; then
      ip route add <사설ip대역>/<subnet> dev eno3 table 3
      ip route add default via <사설게이트웨이ip> dev eno3 table 3
      ip rule add iif eno3 priority 100 table 3
      ip rule add from <현재서버ip> priority 100 table 3
    fi
fi
```


### L3 스위치 설정
1. 초기화하는 법 : 부팅 시 mode 버튼 꾹 누르고 있기
2. 초기 설정 하는 법: [[cisco공식문서](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst3650/hardware/installation/guide/Cat3650hig_book/HGcliSET.html)]를 따라서 하면 된다.
3. 스위치는 모드에 따라 입력 가능한 명령어가 다르니 주의가 필요하다.
- **User EXEC Mode (`>`)**
    - **모드 표시:** `Switch>`
    - **설명:** 기본 모드로, 장비에 접속한 직후에 진입하는 모드입니다. 사용자에게는 제한된 명령만 실행할 수 있는 권한이 주어집니다. 주로 장비 상태 확인과 같은 간단한 작업을 할 수 있습니다. 예를 들어, `show` 명령어로 상태나 설정을 확인할 수 있습니다.
- **Privileged EXEC Mode (`#`)**
    - **모드 표시:** `Switch#`
    - **설명:** `enable` 명령어를 통해 진입하는 모드로, 장비의 대부분의 설정을 변경할 수 있는 권한이 주어집니다. 여기에서는 고급 명령어를 사용하여 장비 설정을 변경하거나, 다른 구성 모드로 전환할 수 있습니다. 예를 들어, `show running-config` 명령어로 현재 설정을 확인하거나, `reload` 명령어로 장비를 재부팅할 수 있습니다.
- **Global Configuration Mode (`(config)`)**
    - **모드 표시:** `Switch(config)`
    - **설명:** `configure terminal` 명령어를 통해 진입하는 모드로, 장비의 설정을 변경할 수 있는 상태입니다. 이 모드에서 네트워크 설정을 변경할 수 있습니다. 예를 들어, `hostname` 명령어로 장비의 이름을 변경하거나, 인터페이스 설정을 수정할 수 있습니다.
- **Interface Configuration Mode (`(config-if)`)**
    - **모드 표시:** `Switch(config-if)`
    - **설명:** 인터페이스 설정을 변경할 수 있는 모드입니다. `interface` 명령어를 사용하여 특정 포트나 인터페이스에 대한 설정을 할 수 있습니다. 예를 들어, `ip address`로 IP 주소를 설정하거나, `shutdown`으로 인터페이스를 비활성화할 수 있습니다.
- **Line Configuration Mode (`(config-line)`)**
    - **모드 표시:** `Switch(config-line)`
    - **설명:** 콘솔, VTY(가상 터미널) 등 장치의 접속 방식에 대한 설정을 변경할 수 있는 모드입니다. 여기에서 `password`나 `login` 설정을 통해 접근 제어를 할 수 있습니다.
- Interface Range Configuration Mode (`(config-if-range)`)
    - **모드 표시:** `Switch(config-if-range)`
    - **설명:** `interface range` 명령어를 사용하여 여러 개의 인터페이스를 동시에 설정할 수 있는 모드입니다. 이 모드에서는 여러 개의 인터페이스를 한 번에 선택하여 동일한 설정을 적용할 수 있습니다. 예를 들어, 여러 포트에 대해 `shutdown`이나 `speed` 설정을 한 번에 할 수 있습니다.
4. vlan 설정
```bash
# 설정할 각 vlan을 아래와 같이 설정한다.
Switch(config)# interface vlan 10                           # 설정할 가상 인터페이스 지정
Switch(config-if)# ip address 192.168.10.1 255.255.255.0    # 인터페이스 ip 설정
Switch(config-if)# no shutdown                              # 인터페이스 활성화.
Switch(config-if)# exit
```
5. vlan에 포트 할당
```bash
Switch(config)# interface range gigabitethernet/0/1 - 4     # 1 ~ 4번 포트 선택 (이름은 조금 다를 수 있음)
Switch(config-if-range)# switchport mode access             # 1:1 지정 모드
Switch(config-if-range)# switchport access vlan 10          # 선택한 포트들을 vlan10에 속하도록 설정
Switch(config-if-range)# exit
```
6. L3 라우팅 기능 활성화
```bash
Switch(config)# ip routing          # ip 라우팅 기능 활성화(다른 vlan 간에 통신 가능)

Switch# show ip route               # 설정된 라우팅 테이블 확인
```
7. DHCP 설정
```bash
# DHCP 풀(DHCP 서버) 생성
Switch(config)# ip dhcp pool VLAN10_Pool
Switch(config-dhcp)# network 192.168.10.0 255.255.255.0
Switch(config-dhcp)# default-router 192.168.10.1
Switch(config-dhcp)# dns-server 8.8.8.8
Switch(config-dhcp)# exit

# 다른 서브넷에 DHCP 서버가 있을 경우
Switch(config)# interface vlan 10
Switch(config-if)# ip helper-address <DHCP 서버 IP>   # DHCP 서버의 IP 주소 입력 (스위치가 DHCP 요청을 해당 서버로 포워딩)
Switch(config-if)# exit
```
8. 라우팅 설정
```bash
ip default-gateway <게이트웨이 ip>      # 기본 게이트웨이 설정
ip route 0.0.0.0 0.0.0.0 <목적지 ip>    # 기본경로 정적 라우팅 설정

show access-lists                       # acl 목록 조회
access-list <acl 번호> permit|deny ip 192.168.1.0 0.0.0.255  # acl 규칙 생성

interface <interface 선택>
ip access-group <acl 번호> in/out       # 해당 인터페이스에 in/out 규칙 추가
```
9. NAT 설정
```bash
ip route 0.0.0.0 0.0.0.0 <상위 라우터 ip> # L3 라우터 설정
```
10. 트렁크 모드 설정
```bash
(config-if-range)#switchport trunk encapsulation dot1q
(config-if-range)#switchport mode trunk
```
11. standby 설정
```bash
# active 스위치 설정
interface vlan <vlan id>
ip address <기본 vlan ip> 255.255.255.0
standby <standby group id> ip <standby group에서 사용할 vip>
standby <standby group id> preemprion                                               # active로 자동 복귀하기 위한 사전 점유 설정
standby <standby group id> priority 150                          # 미설정시 기본값 100
standby <standby group id> timer <hello-time> <hold-time>        # time 각각 5, 15 로 설정 (미설정 시 기본값 3, 10)
standby <standby group id> track <track id> decrement 60         # track id 를 모니터링하여 이상 발생 시 우선도 60 감소
exit
track <track id> interface <포트이름> line-protocol              # 해당 포트를 감시하는 track 등록(위의 standby 설정의 track id와 동일하게 설정)
exit
spanning-tree vlan <vlan id> root primary

# standby 스위치 설정
interface vlan <vlan id>
ip address <기본 vlan ip> 255.255.255.0                          # active와 겹치지 않도록 설정
standby <standby group id> ip <standby group에서 사용할 vip>      # active와 동일하게 설정
standby <standby group id> preemprion
standby <standby group id> timer <hello-time> <hold-time>        # time 각각 5, 15 로 설정 (미설정 시 기본값 3, 10)
exit
track <track id> interface <포트이름> line-protocol              # 해당 포트를 감시하는 track 등록(위의 standby 설정의 track id와 동일하게 설정)
exit
spanning-tree vlan <vlan id> root secondary
```