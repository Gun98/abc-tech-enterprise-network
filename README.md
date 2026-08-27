# ABC Tech Enterprise Network

Cisco Packet Tracer 기반 **본사–지사 기업 네트워크 구축 및 장애 대응 프로젝트**입니다.

약 80~100명 규모의 가상 기업 환경을 가정하여 부서별 네트워크 분리부터
본사–지사 Routing, 인터넷 연결, 접근제어, 이중화 및 중앙 로그 관리까지 구성했습니다.

단순히 통신이 되는 환경을 구축하는 데 그치지 않고,
VLAN / Trunk / OSPF / DHCP / NAT / ACL / EtherChannel 등의 장애를 의도적으로 발생시킨 뒤
`show` 명령과 통신 테스트를 통해 정상 구간을 제거하고 Root Cause를 찾아 복구하는 과정에 중점을 두었습니다.

> 본 프로젝트는 실제 상용망이 아닌 Cisco Packet Tracer 기반 시뮬레이션 프로젝트입니다.

---

## Network Topology

![Network Topology](pic/topology.png)

### 전체 구조

- HQ : DEV / SALES / ADMIN / SERVER / MGMT 네트워크
- Branch : 별도 지사 LAN
- HQ ↔ Branch : OSPF 기반 Dynamic Routing
- HQ ↔ ISP : Default Route + NAT/PAT
- HQ 내부 : Router-on-a-Stick 기반 Inter-VLAN Routing

---

## IP / VLAN Design

| 구분 | VLAN | Network | Gateway |
|---|---:|---|---|
| DEV | 10 | `10.10.10.0/24` | `10.10.10.1` |
| SALES | 20 | `10.10.20.0/24` | `10.10.20.1` |
| ADMIN | 30 | `10.10.30.0/24` | `10.10.30.1` |
| SERVER | 40 | `10.10.40.0/24` | `10.10.40.1` |
| MGMT | 99 | `10.10.99.0/24` | `10.10.99.1` |
| Branch | - | `10.20.10.0/24` | `10.20.10.1` |
| HQ ↔ Branch | - | `172.16.0.0/30` | - |
| HQ ↔ ISP | - | `203.0.113.0/30` | - |
| External Network | - | `198.51.100.0/24` | - |

본사는 `10.10.x.x`, 지사는 `10.20.x.x`로 주소 영역을 구분하고,
본사 VLAN 번호와 세 번째 Octet을 대응시켜 주소만으로도 소속 네트워크를 쉽게 식별할 수 있도록 구성했습니다.

---

## Layer 2

### VLAN / Trunk

DEV, SALES, ADMIN, SERVER, MGMT 네트워크를 VLAN으로 분리하고
802.1Q Trunk를 이용해 여러 VLAN을 스위치 및 Router 간 전달했습니다.

![VLAN](pic/vlan.png)

![Trunk](pic/trunk.png)

### LACP EtherChannel

SW1–SW2 사이의 두 물리 링크를 LACP EtherChannel `Po1`으로 구성했습니다.

```text
Fa0/23 ─┐
        ├── Po1
Fa0/24 ─┘

정상 상태에서 두 인터페이스가 Port-Channel Member로 참여하는 것을 확인하고,
멤버 링크 하나를 Down시켜도 나머지 링크를 통해 통신이 유지되는 것을 검증했습니다.

STP

Layer 2 이중 경로에서 발생할 수 있는 Loop를 방지하기 위해 STP를 사용하고,
Primary Path 장애 시 Backup Path가 Forwarding 상태로 전환되는 과정을 확인했습니다.

Layer 3 / OSPF

본사 VLAN 간 통신은 HQ-R1의 Subinterface를 이용한
Router-on-a-Stick 방식으로 구성했습니다.

본사와 지사 연결은 먼저 Static Routing으로 동작을 검증한 뒤,
향후 네트워크 확장 시 수동 Route 관리 부담을 줄이기 위해 OSPF로 전환했습니다.

HQ-R1
172.16.0.1/30
     |
     | OSPF Area 0
     |
172.16.0.2/30
BR-R1

사용자 VLAN에는 passive-interface를 적용하고
실제 Router가 연결된 WAN 인터페이스에서만 OSPF Neighbor를 형성하도록 구성했습니다.

Routing Table에서 상대 Site의 네트워크가 OSPF Route로 학습되는 것도 확인했습니다.

또한 HQ-R1의 Internet Default Route를:

router ospf 1
 default-information originate

를 통해 BR-R1에 광고하여 지사 사용자도 HQ를 경유해 인터넷에 접근하도록 구성했습니다.

DHCP / DHCP Relay

SERVER VLAN의 중앙 DHCP Server에서 DEV / SALES 사용자 IP를 할당하도록 구성했습니다.

서로 다른 VLAN의 DHCP Broadcast가 Server까지 전달될 수 있도록 HQ-R1에:

ip helper-address 10.10.40.10

을 적용했습니다.

DHCP Relay 설정을 제거했을 때 특정 VLAN에서만 IP 할당이 실패하는 장애도 직접 검증했습니다.

NAT / PAT

본사와 지사의 Private IP가 외부망과 통신할 수 있도록 HQ-R1에서 PAT를 구성했습니다.

HQ / Branch Private IP
        |
        v
      HQ-R1
      NAT/PAT
        |
        v
   203.0.113.1
        |
        v
      ISP-R1

본사 10.10.0.0/16 및 지사 10.20.10.0/24를 NAT 대상으로 지정하고,
HQ-R1의 외부 인터페이스 주소를 여러 내부 사용자가 공유하도록 overload를 적용했습니다.

Management / Access Control
Management VLAN & SSH

네트워크 장비 관리 트래픽을 일반 사용자망과 분리하기 위해 VLAN99를 사용했습니다.

HQ-R1 : 10.10.99.1
SW1   : 10.10.99.11
SW2   : 10.10.99.12

SW1과 SW2에 SSH를 활성화하고,
VTY ACL을 이용해 지정된 ADMIN-PC 10.10.30.10에서만 관리 접속이 가능하도록 구성했습니다.

Extended ACL

역할에 따라 접근정책을 분리했습니다.

Source	SERVER HTTP/HTTPS	SERVER Other	MGMT	Internet
DEV	Allow	Deny	Deny	Allow
SALES	Allow	Deny	Deny	Allow
ADMIN-PC	Allow	Allow	Allow	Allow

Named Extended ACL을 각 사용자 VLAN의 Router Subinterface에 Inbound 방향으로 적용했습니다.

NTP / Syslog

SERVER01을 이용해 네트워크 장비의 시간을 동기화하고,
Router와 Switch의 로그를 중앙 Syslog Server로 전달하도록 구성했습니다.

EtherChannel 멤버 포트를 Down / Up하여 실제 인터페이스 상태 변경 이벤트가
중앙 Syslog Server에 기록되는 것을 확인했습니다.

Troubleshooting

정상 구성 후 여러 장애를 의도적으로 발생시켜 원인 분석과 복구를 진행했습니다.

Incident	Symptom	Root Cause
Trunk VLAN 누락	SALES만 전체 통신 실패	VLAN20 Allowed VLAN 누락
OSPF Area 불일치	WAN Ping O / Branch 통신 X	Area 0 / Area 1 불일치
OSPF Network 누락	Neighbor FULL / 특정 Route 없음	Network Statement 누락
DHCP Relay 누락	특정 VLAN만 DHCP 실패	ip helper-address 누락
NAT ACL 누락	Branch 내부통신 O / Internet X	Branch 대역 NAT 대상 누락
NAT Outside 누락	Router Internet O / 사용자 Internet X	ip nat outside 누락
ACL Rule Order	허용된 HTTP 서비스 차단	상위 Deny Rule 선매칭
ACL 적용 누락	SALES가 MGMT까지 접근	Interface에 ACL 미적용
LACP 불일치	Po1 Down / Member Individual	active / on Mode 불일치
STP Backup 장애	Primary 장애 후 Failover 실패	Backup Interface Shutdown
Troubleshooting Flow
증상 확인
   ↓
영향 범위 확인
   ↓
정상 구간 제거
   ↓
관련 장비 상태 확인
   ↓
Root Cause 확인
   ↓
최소 범위 설정 수정
   ↓
End-to-End 재검증

장애가 발생하면 설정을 바로 변경하기보다
먼저 정상적으로 동작하는 구간을 확인하고 문제 범위를 단계적으로 좁히는 방식으로 접근했습니다.

Verification Commands
show interfaces status
show vlan brief
show interfaces trunk
show etherchannel summary
show spanning-tree vlan 30

show ip interface brief
show ip route
show ip ospf neighbor

show access-lists
show ip nat translations
show ip nat statistics
Repository Structure
.
├── README.md
├── ABC_Tech_Enterprise_Network_Final.pkt
│
├── configs/
│   ├── BR-R1.txt
│   ├── BR_SW1.txt
│   ├── HQ_R1.txt
│   ├── ISP_R1.txt
│   ├── SW1.txt
│   └── SW2.txt
│
└── pic/
    ├── acl.png
    ├── etherchannel.png
    ├── nat.png
    ├── ospf_neighbor.png
    ├── routing.png
    ├── ssh.png
    ├── stp.png
    ├── syslog.png
    ├── syslog2.png
    ├── topology.png
    ├── trunk.png
    └── vlan.png
What I Learned

각 기술을 개별적으로 설정하는 것보다
VLAN, Routing, DHCP, NAT, ACL 등 여러 기능이 End-to-End 통신 과정에서
서로 연결되어 동작한다는 점을 확인했습니다.

특히 OSPF Neighbor가 FULL이어도 특정 Network Statement가 누락되면 Route가 학습되지 않고,
ACL이 정의되어 있어도 인터페이스에 적용되지 않으면 실제 정책으로 동작하지 않는다는 점을
장애 실습을 통해 확인했습니다.

정상망 구축뿐 아니라 의도적으로 장애를 발생시키고,
정상 구간을 먼저 제거하면서 Root Cause 범위를 단계적으로 좁혀가는 문제 해결 방식을 반복했습니다.

Files
ABC_Tech_Enterprise_Network_Final.pkt : 최종 Packet Tracer 프로젝트
configs/ : Router / Switch 최종 Running Configuration
pic/ : 구축 및 검증 결과 이미지
