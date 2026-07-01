# Phase 07 — Configs

## 1. Phase Scope

This file records the final working configuration for Phase 07.

Phase 07 focused on the Cisco Catalyst SD-WAN overlay foundation:

- Controller bring-up
- WAN Edge onboarding
- Root CA and device certificate workflow
- Control connections
- OMP
- TLOC
- BFD overlay tunnel
- Service VPN 10
- Multi-VPN segmentation with VPN 20

Failed commands and failed attempts are intentionally excluded from this file.

---

## 2. Final Device Inventory

| Device | Role | System IP | Site ID | Notes |
|---|---|---:|---:|---|
| SDWAN-MGR | vManage / Manager | 1.1.1.10 | 100 | GUI, certificate operations, WAN Edge list |
| SDWAN-CTRL | vSmart / Controller | 1.1.1.20 | 100 | OMP control plane |
| SDWAN-VALID | vBond / Validator | 1.1.1.30 | 100 | WAN Edge orchestration |
| HQ-SDWAN-EDGE | WAN Edge | 1.1.1.101 | 100 | HQ site |
| BR-SDWAN-EDGE | WAN Edge | 1.1.1.102 | 200 | Branch site |

---

## 3. Common SD-WAN Parameters

```text
Organization Name: WAYNE-SDWAN-LAB
vBond / Validator Public IP: 203.0.113.22

Transport Colors:
- public-internet
- biz-internet

Service VPNs:
- VPN 10
- VPN 20
```

---

## 4. Transport Addressing

| Device | Interface | Color | IP Address |
|---|---|---|---:|
| HQ-SDWAN-EDGE | GigabitEthernet1 | public-internet | 100.64.11.2/30 |
| HQ-SDWAN-EDGE | GigabitEthernet2 | biz-internet | 100.64.12.2/30 |
| BR-SDWAN-EDGE | GigabitEthernet1 | public-internet | 100.64.21.2/30 |
| BR-SDWAN-EDGE | GigabitEthernet2 | biz-internet | 100.64.22.2/30 |

---

## 5. Controller Public Addressing

| Device | Public / Transport IP |
|---|---:|
| SDWAN-MGR | 203.0.113.20 |
| SDWAN-CTRL | 203.0.113.21 |
| SDWAN-VALID | 203.0.113.22 |
| CML Host / Temporary CA HTTP Server | 203.0.113.254 |

---

## 6. CML Host — Temporary CA HTTP Server

The temporary HTTP server was used to provide the root CA certificate to WAN Edge devices.

```bash
cd ~/phase07-sdwan-ca
python3 -m http.server 8000 --bind 203.0.113.254
```

The terminal running this command must remain open while WAN Edges download the certificate.

Root CA download URL:

```text
http://203.0.113.254:8000/WAYNE-SDWAN-LAB-CA.crt
```

---

## 7. CML Host — Branch Return Routes

The CML host needed return routes to the Branch WAN transport subnets.

```bash
sudo ip route replace 100.64.21.0/30 via 203.0.113.1
sudo ip route replace 100.64.22.0/30 via 203.0.113.2
```

Verification:

```bash
ip route get 100.64.21.2
ip route get 100.64.22.2

ping -c 2 100.64.21.2
ping -c 2 100.64.22.2
```

Expected result:

```text
100.64.21.2 reachable via 203.0.113.1
100.64.22.2 reachable via 203.0.113.2
```

---

## 8. CML Host — CA File Directory

The local CA files were stored under:

```bash
cd ~/phase07-sdwan-ca
```

Final CA-related files:

```text
SDWAN-CTRL.crt
SDWAN-CTRL.csr
SDWAN-MGR.crt
SDWAN-MGR.csr
SDWAN-VALID.crt
SDWAN-VALID.csr
WAYNE-SDWAN-LAB-CA.crt
WAYNE-SDWAN-LAB-CA.key
WAYNE-SDWAN-LAB-CA.srl
BR-SDWAN-EDGE.csr
BR-SDWAN-EDGE.crt
```

Important note:

```text
Do not upload private key files such as WAYNE-SDWAN-LAB-CA.key to a public GitHub repository.
```

---

## 9. Branch WAN Edge Certificate Signing

The Branch WAN Edge CSR was saved as:

```text
BR-SDWAN-EDGE.csr
```

The CSR was checked with:

```bash
openssl req -in BR-SDWAN-EDGE.csr -noout -subject
```

The Branch WAN Edge certificate was signed with the local CA:

```bash
openssl x509 -req \
  -in BR-SDWAN-EDGE.csr \
  -CA WAYNE-SDWAN-LAB-CA.crt \
  -CAkey WAYNE-SDWAN-LAB-CA.key \
  -CAcreateserial \
  -out BR-SDWAN-EDGE.crt \
  -days 3650 \
  -sha256
```

Certificate verification:

```bash
openssl x509 -in BR-SDWAN-EDGE.crt -noout -subject -issuer -serial -dates
```

Final validated properties:

```text
Subject CN: vedge-C8K-PAYG-a28-7e91-401d-9b4f-9f969c250194-1.viptela.com
Issuer CN: WAYNE-SDWAN-LAB-ROOT-CA
Certificate validity: valid
```

---

## 10. Time Synchronization Requirement

Before signing and installing certificates, CML host, vManage, and WAN Edge clocks must be aligned.

CML host UTC check:

```bash
date -u
```

If the CML host time must be manually adjusted in the lab:

```bash
sudo timedatectl set-ntp false
sudo date -u -s "2026-06-27 22:25:00"
date -u
```

WAN Edge time check:

```text
show clock
show sdwan system | include Current time
```

---

## 11. HQ-SDWAN-EDGE — System and Transport Configuration

Final identity:

```text
Hostname: HQ-SDWAN-EDGE
System IP: 1.1.1.101
Site ID: 100
Organization: WAYNE-SDWAN-LAB
vBond: 203.0.113.22
```

Configuration:

```cisco
config-transaction

system
 host-name HQ-SDWAN-EDGE
 system-ip 1.1.1.101
 site-id 100
 organization-name WAYNE-SDWAN-LAB
 vbond 203.0.113.22
 exit

interface GigabitEthernet1
 ip address 100.64.11.2 255.255.255.252
 no shutdown
 exit

interface GigabitEthernet2
 ip address 100.64.12.2 255.255.255.252
 no shutdown
 exit

ip route 0.0.0.0 0.0.0.0 100.64.11.1
ip route 0.0.0.0 0.0.0.0 100.64.12.1

sdwan
 interface GigabitEthernet1
  tunnel-interface
   encapsulation ipsec
   color public-internet
   allow-service all
   exit
  exit
 interface GigabitEthernet2
  tunnel-interface
   encapsulation ipsec
   color biz-internet
   allow-service all
   exit
  exit
 exit

commit
end
```

---

## 12. HQ-SDWAN-EDGE — Root CA Installation

Download root CA certificate:

```cisco
copy http://203.0.113.254:8000/WAYNE-SDWAN-LAB-CA.crt bootflash:WAYNE-SDWAN-LAB-CA.crt
```

Install root CA chain:

```cisco
request platform software sdwan root-cert-chain install bootflash:WAYNE-SDWAN-LAB-CA.crt
```

Verification:

```cisco
show sdwan control local-properties
```

Expected result:

```text
root-ca-chain-status        Installed
```

---

## 13. HQ-SDWAN-EDGE — Service VPN 10

VPN 10 was used as the primary enterprise service VPN.

HQ service-side link:

```text
GigabitEthernet3: 172.16.7.1/30
Next hop toward HQ campus/core side: 172.16.7.2
HQ service summary route: 10.10.0.0/16
```

Configuration:

```cisco
config-transaction

vrf definition 10
 address-family ipv4
 exit-address-family
 exit

interface GigabitEthernet3
 vrf forwarding 10
 ip address 172.16.7.1 255.255.255.252
 no shutdown
 exit

ip route vrf 10 10.10.0.0 255.255.0.0 172.16.7.2

commit
end
```

Final VPN 10 route state:

```text
S 10.10.0.0/16 via 172.16.7.2
m 10.20.10.0/24 via 1.1.1.102, Sdwan-system-intf
C 172.16.7.0/30 directly connected, GigabitEthernet3
```

---

## 14. HQ-SDWAN-EDGE — Service VPN 20

VPN 20 was used to validate multi-VPN segmentation.

HQ VPN 20 loopback:

```text
Loopback20: 10.10.20.1/32
```

Configuration:

```cisco
config-transaction

vrf definition 20
 address-family ipv4
 exit-address-family
 exit

interface Loopback20
 vrf forwarding 20
 ip address 10.10.20.1 255.255.255.255
 no shutdown
 exit

commit
end
```

Final VPN 20 route state:

```text
C 10.10.20.1/32 directly connected, Loopback20
m 10.20.20.1/32 via 1.1.1.102, Sdwan-system-intf
```

---

## 15. BR-SDWAN-EDGE — System and Transport Configuration

Final identity:

```text
Hostname: BR-SDWAN-EDGE
System IP: 1.1.1.102
Site ID: 200
Organization: WAYNE-SDWAN-LAB
vBond: 203.0.113.22
```

Configuration:

```cisco
config-transaction

system
 host-name BR-SDWAN-EDGE
 system-ip 1.1.1.102
 site-id 200
 organization-name WAYNE-SDWAN-LAB
 vbond 203.0.113.22
 exit

interface GigabitEthernet1
 ip address 100.64.21.2 255.255.255.252
 no shutdown
 exit

interface GigabitEthernet2
 ip address 100.64.22.2 255.255.255.252
 no shutdown
 exit

ip route 0.0.0.0 0.0.0.0 100.64.21.1
ip route 0.0.0.0 0.0.0.0 100.64.22.1

sdwan
 interface GigabitEthernet1
  tunnel-interface
   encapsulation ipsec
   color public-internet
   allow-service all
   exit
  exit
 interface GigabitEthernet2
  tunnel-interface
   encapsulation ipsec
   color biz-internet
   allow-service all
   exit
  exit
 exit

commit
end
```

---

## 16. BR-SDWAN-EDGE — Root CA Installation

Download root CA certificate:

```cisco
copy http://203.0.113.254:8000/WAYNE-SDWAN-LAB-CA.crt bootflash:WAYNE-SDWAN-LAB-CA.crt
```

Install root CA chain:

```cisco
request platform software sdwan root-cert-chain install bootflash:WAYNE-SDWAN-LAB-CA.crt
```

Verification:

```cisco
show sdwan control local-properties
```

Expected result:

```text
root-ca-chain-status        Installed
```

---

## 17. BR-SDWAN-EDGE — Chassis Number and Token

The Branch WAN Edge was matched with the vManage WAN Edge List entry.

Final values:

```text
Chassis Number:
C8K-PAYG-a28-7e91-401d-9b4f-9f969c250194

Token:
1ff98b349a344145a9a4ad742ee15c6a
```

Configuration command:

```cisco
request platform software sdwan vedge_cloud activate chassis-number C8K-PAYG-a28-7e91-401d-9b4f-9f969c250194 token 1ff98b349a344145a9a4ad742ee15c6a
```

Verification:

```cisco
show sdwan control local-properties
```

Expected result:

```text
chassis-num/unique-id       C8K-PAYG-a28-7e91-401d-9b4f-9f969c250194
token                       1ff98b349a344145a9a4ad742ee15c6a
```

---

## 18. BR-SDWAN-EDGE — Service VPN 10

VPN 10 Branch LAN gateway:

```text
GigabitEthernet3: 10.20.10.254/24
```

Configuration:

```cisco
config-transaction

vrf definition 10
 address-family ipv4
 exit-address-family
 exit

interface GigabitEthernet3
 vrf forwarding 10
 ip address 10.20.10.254 255.255.255.0
 no shutdown
 exit

commit
end
```

Final VPN 10 route state:

```text
m 10.10.0.0/16 via 1.1.1.101, Sdwan-system-intf
C 10.20.10.0/24 directly connected, GigabitEthernet3
L 10.20.10.254/32 directly connected, GigabitEthernet3
m 172.16.7.0/30 via 1.1.1.101, Sdwan-system-intf
```

---

## 19. BR-SDWAN-EDGE — Service VPN 20

VPN 20 Branch loopback:

```text
Loopback20: 10.20.20.1/32
```

Configuration:

```cisco
config-transaction

vrf definition 20
 address-family ipv4
 exit-address-family
 exit

interface Loopback20
 vrf forwarding 20
 ip address 10.20.20.1 255.255.255.255
 no shutdown
 exit

commit
end
```

Final VPN 20 route state:

```text
m 10.10.20.1/32 via 1.1.1.101, Sdwan-system-intf
C 10.20.20.1/32 directly connected, Loopback20
```

---

## 20. vManage GUI — WAN Edge Onboarding Workflow

The following vManage GUI workflow was used.

### WAN Edge List

Path:

```text
Configuration > Devices > WAN Edge List
```

Actions:

```text
1. Add or confirm WAN Edge entry
2. Generate Bootstrap Configuration
3. Confirm chassis number and token
4. Install WAN Edge certificate
5. Send to Controllers
6. Validate Device
```

Final WAN Edge list result:

```text
HQ-SDWAN-EDGE:
- Hostname: HQ-SDWAN-EDGE
- System IP: 1.1.1.101
- Site ID: 100
- Certificate: Installed / Valid

BR-SDWAN-EDGE:
- Hostname: BR-SDWAN-EDGE
- System IP: 1.1.1.102
- Site ID: 200
- Certificate: Installed / Valid
```

---

## 21. vManage GUI — Certificate Installation

Path:

```text
Configuration > Certificates > WAN Edge List
```

Process:

```text
1. Select WAN Edge
2. Install Certificate
3. Paste signed device certificate
4. Install
5. Send to Controllers
6. Validate Device
```

Important:

```text
Paste the signed certificate, not the CSR.
The certificate must start with:
-----BEGIN CERTIFICATE-----

The certificate must end with:
-----END CERTIFICATE-----
```

---

## 22. vManage GUI — Send to Controllers

After installing certificates, the WAN Edge list was pushed to controllers.

Path:

```text
Configuration > Certificates > WAN Edge List > Send to Controllers
```

Expected task result:

```text
Total Task: 3
Success: 3
```

Controllers receiving the serial list:

```text
SDWAN-VALID
SDWAN-MGR
SDWAN-CTRL
```

---

## 23. vManage GUI — Validate Device

After the certificate was installed and the serial list was pushed to controllers, the WAN Edge device was validated.

Path:

```text
Configuration > Certificates > WAN Edge List > Actions > Validate Device
```

Final expected state:

```text
Validate: valid
```

---

## 24. SDWAN-VALID — Final Validation State

Command:

```cisco
show orchestrator valid-vedges
```

Expected final state:

```text
orchestrator valid-vedges C8K-PAYG-A28-7E91-401D-9B4F-9F969C250194
 serial-number  4EA1C49401D8E3529D4A242E737E4C6D2D74B075
 validity       valid
 org            WAYNE-SDWAN-LAB

orchestrator valid-vedges C8K-PAYG-C9D-4A2C-4F8A-ABE5-63814E7C969D
 serial-number  52FC0BF2
 validity       valid
 org            WAYNE-SDWAN-LAB
```

Command:

```cisco
show orchestrator connections
```

Expected result:

```text
HQ-SDWAN-EDGE public-internet up
HQ-SDWAN-EDGE biz-internet    up
BR-SDWAN-EDGE public-internet up
BR-SDWAN-EDGE biz-internet    up
SDWAN-CTRL connection         up
SDWAN-MGR connection          up
```

---

## 25. SDWAN-CTRL — Final OMP Peer State

Command:

```cisco
show omp peers
```

Expected result:

```text
PEER       TYPE   SITE ID   STATE
1.1.1.101  vedge  100       up
1.1.1.102  vedge  200       up
```

Final route count example:

```text
1.1.1.101  R/I/S  6/0/4
1.1.1.102  R/I/S  4/0/6
```

---

## 26. SDWAN-CTRL — Final VPN 10 OMP Routes

Command:

```cisco
show omp routes vpn 10
```

Expected routes:

```text
10.10.0.0/16
  received from 1.1.1.101
  origin-proto static
  status C,R
  tloc 1.1.1.101, biz-internet, ipsec
  tloc 1.1.1.101, public-internet, ipsec

10.20.10.0/24
  received from 1.1.1.102
  origin-proto connected
  status C,R
  tloc 1.1.1.102, biz-internet, ipsec
  tloc 1.1.1.102, public-internet, ipsec

172.16.7.0/30
  received from 1.1.1.101
  origin-proto connected
  status C,R
  tloc 1.1.1.101, biz-internet, ipsec
  tloc 1.1.1.101, public-internet, ipsec
```

---

## 27. SDWAN-CTRL — Final VPN 20 OMP Routes

Command:

```cisco
show omp routes vpn 20
```

Expected routes:

```text
10.10.20.1/32
  received from 1.1.1.101
  origin-proto connected
  status C,R
  tloc 1.1.1.101, biz-internet, ipsec
  tloc 1.1.1.101, public-internet, ipsec

10.20.20.1/32
  received from 1.1.1.102
  origin-proto connected
  status C,R
  tloc 1.1.1.102, biz-internet, ipsec
  tloc 1.1.1.102, public-internet, ipsec
```

---

## 28. SDWAN-CTRL — Final TLOC State

Command:

```cisco
show omp tlocs
```

Expected TLOC entries:

```text
1.1.1.101 biz-internet ipsec
  status C,I,R
  public-ip 100.64.12.2
  public-port 12386
  private-ip 100.64.12.2
  private-port 12386

1.1.1.101 public-internet ipsec
  status C,I,R
  public-ip 100.64.11.2
  public-port 12386
  private-ip 100.64.11.2
  private-port 12386

1.1.1.102 biz-internet ipsec
  status C,I,R
  public-ip 100.64.22.2
  public-port 12386
  private-ip 100.64.22.2
  private-port 12386

1.1.1.102 public-internet ipsec
  status C,I,R
  public-ip 100.64.21.2
  public-port 12386
  private-ip 100.64.21.2
  private-port 12386
```

---

## 29. HQ-SDWAN-EDGE — Final Verification Commands

```cisco
show sdwan system
show ip interface brief
show ip route
show ip route vrf 10
show ip route vrf 20
show sdwan control local-properties
show sdwan control connections
show sdwan omp peers
show sdwan omp routes vpn 10
show sdwan omp routes vpn 20
show sdwan bfd sessions
```

Expected control connections:

```text
vsmart  1.1.1.20  public-internet  up
vsmart  1.1.1.20  biz-internet     up
vbond   203.0.113.22 public-internet up
vbond   203.0.113.22 biz-internet    up
vmanage 1.1.1.10  public-internet  up
```

Expected OMP peer:

```text
Peer: 1.1.1.20
Type: vsmart
State: up
```

Expected BFD sessions:

```text
Remote System IP: 1.1.1.102
Remote Site ID: 200
State: up

public-internet to public-internet
public-internet to biz-internet
biz-internet to public-internet
biz-internet to biz-internet
```

---

## 30. BR-SDWAN-EDGE — Final Verification Commands

```cisco
show sdwan system
show ip interface brief
show ip route
show ip route vrf 10
show ip route vrf 20
show sdwan control local-properties
show sdwan control connections
show sdwan omp peers
show sdwan omp routes vpn 10
show sdwan omp routes vpn 20
show sdwan bfd sessions
```

Expected control connections:

```text
vsmart  1.1.1.20  public-internet  up
vsmart  1.1.1.20  biz-internet     up
vbond   203.0.113.22 public-internet up
vbond   203.0.113.22 biz-internet    up
vmanage 1.1.1.10  biz-internet     up
```

Expected OMP peer:

```text
Peer: 1.1.1.20
Type: vsmart
State: up
```

Expected BFD sessions:

```text
Remote System IP: 1.1.1.101
Remote Site ID: 100
State: up

public-internet to public-internet
public-internet to biz-internet
biz-internet to public-internet
biz-internet to biz-internet
```

---

## 31. VPN 10 Reachability Tests

HQ to Branch VPN 10:

```cisco
ping vrf 10 10.20.10.254
```

Expected result:

```text
Success rate is 100 percent
```

Branch to HQ VPN 10:

```cisco
ping vrf 10 172.16.7.1
```

Expected result:

```text
Success rate is 100 percent
```

---

## 32. VPN 20 Reachability Tests

HQ to Branch VPN 20:

```cisco
ping vrf 20 10.20.20.1 source Loopback20
```

Expected result:

```text
Success rate is 100 percent
```

Branch to HQ VPN 20:

```cisco
ping vrf 20 10.10.20.1 source Loopback20
```

Expected result:

```text
Success rate is 100 percent
```

---

## 33. Cross-VPN Segmentation Test

VPN 20 to VPN 10 test from HQ:

```cisco
ping vrf 20 10.20.10.254 source Loopback20
```

Expected result:

```text
Success rate is 0 percent
```

VPN 20 to VPN 10 test from Branch:

```cisco
ping vrf 20 172.16.7.1 source Loopback20
```

Expected result:

```text
Success rate is 0 percent
```

This failure is expected because VPN 10 and VPN 20 are separate service VPNs and no route leaking was configured.

---

## 34. Final Working State Summary

### Control Plane

```text
HQ-SDWAN-EDGE control connections: up
BR-SDWAN-EDGE control connections: up
SDWAN-CTRL OMP peers: up
SDWAN-VALID orchestrator connections: up
```

### OMP

```text
VPN 10 routes exchanged:
- HQ 10.10.0.0/16
- HQ 172.16.7.0/30
- Branch 10.20.10.0/24

VPN 20 routes exchanged:
- HQ 10.10.20.1/32
- Branch 10.20.20.1/32
```

### TLOC

```text
HQ-SDWAN-EDGE:
- 1.1.1.101 public-internet ipsec
- 1.1.1.101 biz-internet ipsec

BR-SDWAN-EDGE:
- 1.1.1.102 public-internet ipsec
- 1.1.1.102 biz-internet ipsec
```

### BFD

```text
HQ to Branch BFD sessions: up
Branch to HQ BFD sessions: up

Four overlay paths are active:
- public-internet to public-internet
- public-internet to biz-internet
- biz-internet to public-internet
- biz-internet to biz-internet
```

### Service VPN

```text
VPN 10 same-VPN communication: PASS
VPN 20 same-VPN communication: PASS
VPN 10 to VPN 20 communication: BLOCKED as expected
```

---

## 35. Final Phase 07 Result

Phase 07 reached the final working SD-WAN overlay baseline.

The following items were successfully configured and verified:

- SD-WAN controllers operational
- HQ WAN Edge onboarded
- Branch WAN Edge onboarded
- Root CA installed
- WAN Edge certificates installed and valid
- WAN Edge list pushed to controllers
- WAN Edge devices validated
- Control connections up
- OMP peers up
- OMP routes exchanged
- TLOCs advertised and received
- BFD sessions up
- VPN 10 service route exchange working
- VPN 20 segmentation working
- Cross-VPN isolation confirmed
