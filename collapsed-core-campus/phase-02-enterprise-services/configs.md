# Phase 02 - Enterprise-Services Configurations

## Build Flow

- MGMT-SRV1 Ubuntu
- EDGE1
- CORE1
- CORE2
- ACC1
- ACC2
- ACC3
- SRV-ACC1

---
  
## STEP 1 — MGMT-SRV1 Ubuntu

Purpose
- MGMT-SRV1은 Phase 02에서 통합 관리 서버 역할을 수행한다.
- DHCP, DNS resolver, Syslog receiver, SNMP manager 역할을 하나의 Ubuntu 노드에 통합하여 실제 Enterprise Management Server 구조를 학습하기 위해 사용한다.

```bash
sudo ip link set ens2 up
sudo ip addr add 10.20.20.31/24 dev ens2
sudo ip route add default via 10.20.20.254 dev ens2
```
Why
- ens2를 활성화하여 네트워크 연결을 사용 가능하게 한다.
- 10.20.20.31/24는 MGMT-SRV1의 고정 관리 IP이다.
- default route는 MGMT-SRV1이 CORE/EDGE를 통해 외부 인터넷과 내부 VLAN으로 통신하기 위해 필요하다.

---

```bash  
sudo nano /etc/resolv.conf

nameserver 8.8.8.8
nameserver 1.1.1.1
```
Why
- Ubuntu에서 apt update, 패키지 설치, 외부 도메인 해석을 하기 위해 DNS resolver를 수동 지정했다.

---

```bash  
sudo apt update
sudo apt install dnsmasq -y
sudo apt install snmp snmp-mibs-downloader -y
```
Why
- dnsmasq : DHCP/DNS 서비스 제공
- snmp : MGMT-SRV1에서 Cisco 장비를 SNMP로 조회하기 위한 도구
- snmp-mibs-downloader : MIB/OID 해석 보조 도구

---

```bash
sudo nano /etc/dnsmasq.conf
```
```cisco
port=0
interface=ens2

dhcp-range=10.10.10.100,10.10.10.199,255.255.255.0,12h
dhcp-option=option:router,10.10.10.254
dhcp-option=option:dns-server,8.8.8.8,1.1.1.1
log-dhcp
```
Why
- port=0 : systemd-resolved와 DNS 포트 53 충돌을 피하고 DHCP 기능 중심으로 사용한다.
- interface=ens2 : DHCP 서비스를 MGMT-SRV1의 실제 연결 인터페이스에 바인딩한다.
- dhcp-range : VLAN10 사용자 단말에 자동 할당할 IP 범위이다.
- router : 클라이언트 기본 게이트웨이를 HSRP VIP로 지정한다.
- dns-server : 클라이언트가 외부 도메인을 해석할 수 있도록 DNS 서버를 배포한다.
- log-dhcp : DHCP lease 과정을 로그로 확인하기 위해 필요하다.

---

```bash
sudo systemctl restart dnsmasq
sudo systemctl enable dnsmasq
```
Why
- dnsmasq 설정을 적용하고, 재부팅 후에도 DHCP 서비스가 자동 시작되도록 한다.

---

```bash  
sudo nano /etc/rsyslog.conf
```
```cisco
module(load="imudp")
input(type="imudp" port="514")
```
Why
- Cisco 장비들이 UDP/514로 전송하는 Syslog 메시지를 MGMT-SRV1이 수신할 수 있도록 한다.

---

```bash  
sudo systemctl restart rsyslog
```
Why
- rsyslog 설정 변경 사항을 적용한다.

---

## STEP 2 - EDGE1

Purpose
- EDGE1은 내부 캠퍼스망과 External Connector 사이의 인터넷 경계 라우터 역할을 수행한다.
- NAT/PAT, default route, 내부망 return route, SSH, NTP, Syslog, SNMP agent 기능을 담당한다.

---
  
conf t
hostname EDGE1

interface g0/0
 description TO-EXTERNAL-CONNECTOR
 no ip address
 ip address dhcp
 ip nat outside
 no shutdown

 Why
 - External Connector로 부터 DHCP 주소를 받아 인터넷 연결을 수행한다.
 - NAT outside 인터페이스로 사용한다.

---

interface g0/1
 description TO-CORE1
 ip address 10.255.255.1 255.255.255.252
 no shutdown
 ip nat inside

interface g0/2
 description TO-CORE2
 ip address 10.255.255.5 255.255.255.252
 no shutdown
 ip nat inside

Why
- CORE1~2과 EDGE1 사이의 point-to-point routed link이다.
- 내부망에서 외부로 나가는 트래픽의 NAT inside 구간이다.

---
  
ip route 0.0.0.0 0.0.0.0 dhcp
ip route 10.10.10.0 255.255.255.0 10.255.255.2
ip route 10.20.20.0 255.255.255.0 10.255.255.6
ip route 10.40.40.0 255.255.255.0 10.255.255.2

Why
- External Connector DHCP를 통해 받은 gateway를 default route로 사용하여 EDGE1이 인터넷으로 패킷을 보낼 수 있게 한다.
- 외부에서 돌아오는 응답 패킷을 내부 VLAN으로 되돌려 보내기 위한 return route다.
- 이 경로가 없으면 NAT 이후 응답 트래픽이 내부망으로 돌아가지 못한다.

---

access-list 1 permit 10.0.0.0 0.255.255.255
ip nat inside source list 1 interface g0/0 overload

Why 
- 10.0.0.0/8 내부 사설망 주소를 EDGE1 외부 인터페이스 주소로 PAT 변환하여 인터넷 접속이 가능하게 한다.

---

ntp server 10.40.40.1

Why
- 내부 캠퍼스의 기준 시간 서버인 CORE1과 시간을 동기화하기 위해 사용한다.

---

end
wr

---

## STEP 3 - CORE1

Purpose
- CORE1은 VLAN10/VLAN20/VLAN40 SVI, HSRP, DHCP relay, NTP master, SSH/SNMP/Syslog 관리 기능을 제공한다.

---
  
conf t
hostname CORE1

interface g0/0
 no switchport
 description TO-EDGE1
 ip address 10.255.255.2 255.255.255.252
 no shutdown

Why
- CORE1과 EDGE1 사이를 L3 routed port로 구성한다.
- default route의 next-hot인 EDGE1과 직접 연결되기 위해 필요하다.

---

interface vlan 10
 ip helper-address 10.20.20.31

 Why
 - VLAN10 사용자의 DHCP broadcast를 MGMT-SRV1의 dnsmasq DHCP 서버로 relay하기 위해 필요하다.

---

ip route 0.0.0.0 0.0.0.0 10.255.255.1

Why
- CORE1이 모르는 외부 목적지 트래픽을 EDGE1으로 전달하기 위한 default route 이다.

---

ntp master 1

Why
- CORE1을 내부 기준 시간 소스로 사용한다.
- 모든 장비가 CORE1 시간을 기준으로 동기화되어 Syslog와 장애 분석의 시간 기준이 맞게 된다.

---

end
wr

---

## STEP 4 - CORE2

Purpose
- CORE2는 HSRP redundancy, VLAN SVI, DHCP relay, EDGE1 연결, NTP client, SSH/SNMP/Syslog 기능을 제공한다.

---

conf t
hostname CORE2

interface g0/0
 no switchport
 description TO-EDGE1
 ip address 10.255.255.6 255.255.255.252
 no shutdown

Why
- CORE2와 EDGE1 사이를 L3 routed port로 구성하여 CORE2 active VLAN 트래픽도 외부망으로 전달할 수 있게 한다.

---

interface vlan 10
 ip helper-address 10.20.20.31

Why
- VLAN10 DHCP 요청을 중앙 DHCP 서버인 MGMT-SRV1으로 relay한다.

---

ip route 0.0.0.0 0.0.0.0 10.255.255.5

Why
- CORE2가 외부 목적지 트래픽을 EDGE1로 전달하기 위한 default route 이다.

---

ntp server 10.40.40.1

Why
- CORE1을 기준 시간 서버로 사용하여 CORE2의 로그 시간이 CORE1과 일치하도록 한다.

---

end
wr

---

## STEP 5 - ACC1

Purpose
- Access Switch는 사용자 단말 또는 관리 단말을 수용하고, VLAN40 관리 IP를 통해 원격 관리된다.

---

conf t
hostname ACC1

interface vlan 40
 ip address 10.40.40.11 255.255.255.0
 no shutdown

Why
- 각 Access Switch를 Management VLAN에서 SSH/SNMP/Syslog/NTP 대상으로 관리하기 위한 SVI 이다. 

---

ip default-gateway 10.40.40.254

Why
- 각 Access Switch는 L2 스위치이므로 라우팅 테이블 대신 default-gateway를 사용하여 원격 관리 트래픽을 CORE HSRP VIP로 전달한다.

---

ntp server 10.40.40.1

Why
- 각 Access Switch 로그의 시간 기준을 CORE1 NTP master와 동기화한다.

---

end
wr

---

## STEP 6 - ACC2

conf t
hostname ACC2

interface vlan 40
 ip address 10.40.40.12 255.255.255.0
 no shutdown

---

ip default-gateway 10.40.40.254

---

ntp server 10.40.40.1

---

end
wr

---

## STEP 7 - ACC3

conf t
hostname ACC3

interface vlan 40
 ip address 10.40.40.13 255.255.255.0
 no shutdown

---

ip default-gateway 10.40.40.254

---

ntp server 10.40.40.1

---

end
wr

---

## STEP 8 - SRV-ACC1

Purpose
- SRV-ACC1은 MGMT-SRV1이 연결되는 서버 액세스 스위치이다.
- 서버 VLAN20과 관리 VLAN40을 함께 수용한다.

---
  
conf t
hostname SRV-ACC1

vlan 40
 name MGMT

Why
- SRV-ACC1에서 VLAN40이 없어 VLAN40 SVI가 down/down 상태였기 때문에, 관리 VLAN을 생성하여 SVI를 활성화하기 위해 필요했다.

---

interface g0/0
 description TO-ACC2
 switchport trunk encapsulation dot1q
 switchport trunk native vlan 999
 switchport mode trunk
 no shutdown

Why
- ACC2와 SRV-ACC1 사이에 VLAN20 서버 트래픽과 VLAN40 관리 트래픽을 전달하기 위한 trunk uplink 이다.

---

interface vlan 40
 ip address 10.40.40.21 255.255.255.0
 no shutdown

Why
- SRV-ACC1 자체를 Management VLAN에서 관리하기 위한 SVI 이다.

---
  
ip default-gateway 10.40.40.254

Why
- SRV-ACC1이 원격 관리 트래픽을 CORE HSRP VIP로 전달하기 위해 필요하다.

---

ntp server 10.40.40.1

Why
- 각 Access Switch 로그의 시간 기준을 CORE1 NTP master와 동기화한다.
  
---

end
wr

---

## STEP 9 - 공통 SSH/Syslog/SNMP 설정
- EDGE1, CORE1, CORE2, ACC1, ACC2, ACC3, SRV-ACC1

---

ip domain-name lab.local
username admin privilege 15 secret <ADMIN_SECRET>
crypto key generate rsa modulus 2048
ip ssh version 2
service password-encryption

line vty 0 4
 login local
 transport input ssh

end
wr 

Why
- ip domain-name : RSA key 생성에 필요하다.
- username : local login 인증 계정을 만든다.
- crypto key : SSH 암호화를 위한 RSA key를 생성한다.
- ip ssh version 2 : SSHv2 만 사용하도록 한다.
- service password-encryption : 평문 password 노출을 줄인다.
- login local : VTY 접속 시 local user DB를 사용한다.
- transport input ssh : Telnet을 차단하고 SSH만 허용한다.

---

service timestamps log datetime msec
logging host 10.20.20.31
logging trap informational
logging on

end
wr

Why
- service timestamps : 로그에 시간 정보를 남긴다.
- logging host : MGMT-SRV1을 중앙 Syslog 서버로 지정한다.
- logging trap informational : informational 레벨 이상의 로그를 전송한다.
- logging on : Syslog 전송 기능을 활성화한다.

---

ip access-list standard SNMP-MGMT
 permit 10.20.20.0 0.0.0.255

snmp-server community NMS-RO RO SNMP-MGMT
snmp-server location LAB
snmp-server contact admin@lab.local

end
wr

Why
- SNMP-MGMT ACL : SNMP 접근을 MGMT-SRV1이 위치한 관리망으로 제한한다.
- NMS-RO : 읽기 전용 SNMP community 이다.
- location : 장비 위치 정보를 SNMP로 제공한다.
- contact : 운영자 연락처 정보를 SNMP로 제공한다.

---



