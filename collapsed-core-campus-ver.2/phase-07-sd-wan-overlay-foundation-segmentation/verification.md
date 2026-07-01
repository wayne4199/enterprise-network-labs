# Phase 07 — Verification

## 1. Purpose

This document records the final verification results for Phase 07.

Phase 07 validated the Cisco Catalyst SD-WAN overlay foundation and multi-VPN segmentation.

The following items were verified:

- SD-WAN controller control-plane readiness
- HQ WAN Edge onboarding
- Branch WAN Edge onboarding
- Root CA and device certificate installation
- Control connections
- OMP peering
- TLOC advertisement
- BFD overlay tunnels
- Service VPN 10 route exchange
- VPN 20 segmentation route exchange
- Same-VPN overlay reachability
- Cross-VPN isolation

---

## 2. Final Device Summary

| Device | Role | System IP | Site ID | Final State |
|---|---|---:|---:|---|
| SDWAN-MGR | vManage / Manager | 1.1.1.10 | 100 | Operational |
| SDWAN-CTRL | vSmart / Controller | 1.1.1.20 | 100 | Operational |
| SDWAN-VALID | vBond / Validator | 1.1.1.30 | 100 | Operational |
| HQ-SDWAN-EDGE | HQ WAN Edge | 1.1.1.101 | 100 | Onboarded |
| BR-SDWAN-EDGE | Branch WAN Edge | 1.1.1.102 | 200 | Onboarded |

---

## 3. Final Verification Summary

| Verification Item | Result |
|---|---|
| Controller bring-up | PASS |
| Control component certificates | PASS |
| HQ WAN Edge certificate | PASS |
| Branch WAN Edge certificate | PASS |
| HQ control connections | PASS |
| Branch control connections | PASS |
| vSmart OMP peers | PASS |
| TLOC advertisement | PASS |
| TLOC resolution | PASS |
| BFD overlay tunnels | PASS |
| VPN 10 OMP routes | PASS |
| VPN 20 OMP routes | PASS |
| VPN 10 reachability | PASS |
| VPN 20 reachability | PASS |
| VPN 10 / VPN 20 segmentation | PASS |
| vManage GUI reachability | PASS |
| SDWAN-VALID valid WAN Edge list | PASS |

---

## 4. Controller Verification

## 4.1 SDWAN-CTRL — OMP Peers

Command:

```cisco
show omp peers
```

Expected result:

```text
PEER       TYPE    SITE ID   STATE
1.1.1.101  vedge   100       up
1.1.1.102  vedge   200       up
```

Observed result:

```text
1.1.1.101  vedge  site 100  up
1.1.1.102  vedge  site 200  up
```

Interpretation:

```text
PASS

SDWAN-CTRL successfully formed OMP peer relationships with both WAN Edges.
```

---

## 4.2 SDWAN-CTRL — VPN 10 OMP Routes

Command:

```cisco
show omp routes vpn 10
```

Expected VPN 10 routes:

```text
10.10.0.0/16
10.20.10.0/24
172.16.7.0/30
```

Observed route sources:

```text
10.10.0.0/16
  received from 1.1.1.101
  origin-proto static
  status C,R

10.20.10.0/24
  received from 1.1.1.102
  origin-proto connected
  status C,R

172.16.7.0/30
  received from 1.1.1.101
  origin-proto connected
  status C,R
```

Interpretation:

```text
PASS

SDWAN-CTRL received VPN 10 service routes from both HQ and Branch.
HQ advertised the HQ service summary route.
Branch advertised the Branch service LAN route.
```

---

## 4.3 SDWAN-CTRL — VPN 20 OMP Routes

Command:

```cisco
show omp routes vpn 20
```

Expected VPN 20 routes:

```text
10.10.20.1/32
10.20.20.1/32
```

Observed route sources:

```text
10.10.20.1/32
  received from 1.1.1.101
  origin-proto connected
  status C,R

10.20.20.1/32
  received from 1.1.1.102
  origin-proto connected
  status C,R
```

Interpretation:

```text
PASS

SDWAN-CTRL received VPN 20 loopback routes from both WAN Edges.
VPN 20 route exchange through OMP is working.
```

---

## 4.4 SDWAN-CTRL — TLOC Verification

Command:

```cisco
show omp tlocs
```

Expected TLOC entries:

```text
1.1.1.101 public-internet ipsec
1.1.1.101 biz-internet    ipsec
1.1.1.102 public-internet ipsec
1.1.1.102 biz-internet    ipsec
```

Observed final TLOC status:

```text
HQ-SDWAN-EDGE:
1.1.1.101 public-internet  C,I,R
1.1.1.101 biz-internet     C,I,R

BR-SDWAN-EDGE:
1.1.1.102 public-internet  C,I,R
1.1.1.102 biz-internet     C,I,R
```

Status meaning:

```text
C = Chosen
I = Installed
R = Resolved
```

Interpretation:

```text
PASS

SDWAN-CTRL successfully learned and resolved both transport TLOCs from both WAN Edges.
```

---

## 5. SDWAN-VALID Verification

## 5.1 Orchestrator Connections

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
SDWAN-CTRL up
SDWAN-MGR up
```

Observed result:

```text
vedge   1.1.1.101  public-internet  up
vedge   1.1.1.101  biz-internet     up
vedge   1.1.1.102  public-internet  up
vedge   1.1.1.102  biz-internet     up
vsmart  1.1.1.20   up
vmanage 1.1.1.10   up
```

Interpretation:

```text
PASS

SDWAN-VALID successfully sees both WAN Edges and both controllers.
```

---

## 5.2 Valid WAN Edge List

Command:

```cisco
show orchestrator valid-vedges
```

Expected result:

```text
HQ-SDWAN-EDGE validity valid
BR-SDWAN-EDGE validity valid
```

Observed result:

```text
C8K-PAYG-A28-7E91-401D-9B4F-9F969C250194
 serial-number 4EA1C49401D8E3529D4A242E737E4C6D2D74B075
 validity      valid
 org           WAYNE-SDWAN-LAB

C8K-PAYG-C9D-4A2C-4F8A-ABE5-63814E7C969D
 serial-number 52FC0BF2
 validity      valid
 org           WAYNE-SDWAN-LAB
```

Interpretation:

```text
PASS

Both WAN Edges are valid from the Validator/vBond perspective.
```

---

## 6. HQ-SDWAN-EDGE Verification

## 6.1 HQ Control Connections

Command:

```cisco
show sdwan control connections
```

Expected result:

```text
vsmart  up
vbond   up
vmanage up
```

Observed result:

```text
vsmart   1.1.1.20      public-internet  up
vsmart   1.1.1.20      biz-internet     up
vbond    203.0.113.22  public-internet  up
vbond    203.0.113.22  biz-internet     up
vmanage  1.1.1.10      public-internet  up
```

Interpretation:

```text
PASS

HQ-SDWAN-EDGE has working control connections to vSmart, vBond, and vManage.
```

---

## 6.2 HQ OMP Peer

Command:

```cisco
show sdwan omp peers
```

Expected result:

```text
Peer 1.1.1.20
Type vsmart
State up
```

Observed result:

```text
1.1.1.20  vsmart  up
R/I/S = 4/4/6
```

Interpretation:

```text
PASS

HQ-SDWAN-EDGE successfully formed an OMP peer relationship with SDWAN-CTRL.
```

---

## 6.3 HQ BFD Sessions

Command:

```cisco
show sdwan bfd sessions
```

Expected result:

```text
Remote system IP 1.1.1.102
Remote site ID 200
State up
```

Observed result:

```text
HQ public-internet -> BR public-internet  up
HQ public-internet -> BR biz-internet     up
HQ biz-internet    -> BR public-internet  up
HQ biz-internet    -> BR biz-internet     up
```

Interpretation:

```text
PASS

HQ-SDWAN-EDGE formed four BFD data-plane sessions toward BR-SDWAN-EDGE.
This confirms SD-WAN overlay tunnel establishment.
```

---

## 6.4 HQ VPN 10 Routing Table

Command:

```cisco
show ip route vrf 10
```

Expected result:

```text
10.10.0.0/16 local static route
10.20.10.0/24 learned from Branch through OMP
172.16.7.0/30 directly connected
```

Observed result:

```text
S 10.10.0.0/16 via 172.16.7.2
m 10.20.10.0/24 via 1.1.1.102, Sdwan-system-intf
C 172.16.7.0/30 directly connected, GigabitEthernet3
```

Interpretation:

```text
PASS

HQ VPN 10 has both local and remote service routes.
The Branch VPN 10 route is installed as an SD-WAN OMP route.
```

---

## 6.5 HQ VPN 20 Routing Table

Command:

```cisco
show ip route vrf 20
```

Expected result:

```text
10.10.20.1/32 local Loopback20
10.20.20.1/32 learned from Branch through OMP
```

Observed result:

```text
C 10.10.20.1/32 directly connected, Loopback20
m 10.20.20.1/32 via 1.1.1.102, Sdwan-system-intf
```

Interpretation:

```text
PASS

HQ VPN 20 learned the Branch VPN 20 loopback route through OMP.
```

---

## 6.6 HQ VPN 10 OMP Routes

Command:

```cisco
show sdwan omp routes vpn 10
```

Expected result:

```text
HQ local VPN 10 routes
Branch remote VPN 10 routes
```

Observed important routes:

```text
10.10.0.0/16
  TLOC 1.1.1.101 public-internet
  TLOC 1.1.1.101 biz-internet

172.16.7.0/30
  TLOC 1.1.1.101 public-internet
  TLOC 1.1.1.101 biz-internet

10.20.10.0/24
  TLOC 1.1.1.102 public-internet
  TLOC 1.1.1.102 biz-internet
```

Interpretation:

```text
PASS

HQ is advertising local VPN 10 routes and receiving the Branch VPN 10 route.
```

---

## 6.7 HQ VPN 20 OMP Routes

Command:

```cisco
show sdwan omp routes vpn 20
```

Expected result:

```text
10.10.20.1/32
10.20.20.1/32
```

Observed important routes:

```text
10.10.20.1/32
  TLOC 1.1.1.101 public-internet
  TLOC 1.1.1.101 biz-internet

10.20.20.1/32
  TLOC 1.1.1.102 public-internet
  TLOC 1.1.1.102 biz-internet
```

Interpretation:

```text
PASS

HQ VPN 20 route exchange is working through OMP.
```

---

## 7. BR-SDWAN-EDGE Verification

## 7.1 Branch Control Connections

Command:

```cisco
show sdwan control connections
```

Expected result:

```text
vsmart  up
vbond   up
vmanage up
```

Observed result:

```text
vsmart   1.1.1.20      public-internet  up
vsmart   1.1.1.20      biz-internet     up
vbond    203.0.113.22  public-internet  up
vbond    203.0.113.22  biz-internet     up
vmanage  1.1.1.10      biz-internet     up
```

Interpretation:

```text
PASS

BR-SDWAN-EDGE has working control connections to vSmart, vBond, and vManage.
```

---

## 7.2 Branch OMP Peer

Command:

```cisco
show sdwan omp peers
```

Expected result:

```text
Peer 1.1.1.20
Type vsmart
State up
```

Observed result:

```text
1.1.1.20  vsmart  up
R/I/S = 6/6/4
```

Interpretation:

```text
PASS

BR-SDWAN-EDGE successfully formed an OMP peer relationship with SDWAN-CTRL.
```

---

## 7.3 Branch BFD Sessions

Command:

```cisco
show sdwan bfd sessions
```

Expected result:

```text
Remote system IP 1.1.1.101
Remote site ID 100
State up
```

Observed result:

```text
BR public-internet -> HQ public-internet  up
BR public-internet -> HQ biz-internet     up
BR biz-internet    -> HQ public-internet  up
BR biz-internet    -> HQ biz-internet     up
```

Interpretation:

```text
PASS

BR-SDWAN-EDGE formed four BFD data-plane sessions toward HQ-SDWAN-EDGE.
This confirms SD-WAN overlay tunnel establishment.
```

---

## 7.4 Branch VPN 10 Routing Table

Command:

```cisco
show ip route vrf 10
```

Expected result:

```text
10.10.0.0/16 learned from HQ through OMP
10.20.10.0/24 directly connected
172.16.7.0/30 learned from HQ through OMP
```

Observed result:

```text
m 10.10.0.0/16 via 1.1.1.101, Sdwan-system-intf
C 10.20.10.0/24 directly connected, GigabitEthernet3
m 172.16.7.0/30 via 1.1.1.101, Sdwan-system-intf
```

Interpretation:

```text
PASS

Branch VPN 10 has both local and remote service routes.
HQ VPN 10 routes are installed as SD-WAN OMP routes.
```

---

## 7.5 Branch VPN 20 Routing Table

Command:

```cisco
show ip route vrf 20
```

Expected result:

```text
10.10.20.1/32 learned from HQ through OMP
10.20.20.1/32 local Loopback20
```

Observed result:

```text
m 10.10.20.1/32 via 1.1.1.101, Sdwan-system-intf
C 10.20.20.1/32 directly connected, Loopback20
```

Interpretation:

```text
PASS

Branch VPN 20 learned the HQ VPN 20 loopback route through OMP.
```

---

## 7.6 Branch VPN 10 OMP Routes

Command:

```cisco
show sdwan omp routes vpn 10
```

Expected result:

```text
HQ remote VPN 10 routes
Branch local VPN 10 routes
```

Observed important routes:

```text
10.10.0.0/16
  TLOC 1.1.1.101 public-internet
  TLOC 1.1.1.101 biz-internet

172.16.7.0/30
  TLOC 1.1.1.101 public-internet
  TLOC 1.1.1.101 biz-internet

10.20.10.0/24
  TLOC 1.1.1.102 public-internet
  TLOC 1.1.1.102 biz-internet
```

Interpretation:

```text
PASS

Branch is advertising its local VPN 10 route and receiving HQ VPN 10 routes.
```

---

## 7.7 Branch VPN 20 OMP Routes

Command:

```cisco
show sdwan omp routes vpn 20
```

Expected result:

```text
10.10.20.1/32
10.20.20.1/32
```

Observed important routes:

```text
10.10.20.1/32
  TLOC 1.1.1.101 public-internet
  TLOC 1.1.1.101 biz-internet

10.20.20.1/32
  TLOC 1.1.1.102 public-internet
  TLOC 1.1.1.102 biz-internet
```

Interpretation:

```text
PASS

Branch VPN 20 route exchange is working through OMP.
```

---

## 8. VPN 10 Reachability Verification

## 8.1 HQ to Branch VPN 10

Command on HQ-SDWAN-EDGE:

```cisco
ping vrf 10 10.20.10.254
```

Expected result:

```text
Success rate is 100 percent
```

Observed result:

```text
Success rate is 100 percent
```

Interpretation:

```text
PASS

HQ VPN 10 can reach Branch VPN 10.
```

---

## 8.2 Branch to HQ VPN 10

Command on BR-SDWAN-EDGE:

```cisco
ping vrf 10 172.16.7.1
```

Expected result:

```text
Success rate is 100 percent
```

Observed result:

```text
Success rate is 100 percent
```

Interpretation:

```text
PASS

Branch VPN 10 can reach HQ VPN 10.
```

---

## 9. VPN 20 Reachability Verification

## 9.1 HQ to Branch VPN 20

Command on HQ-SDWAN-EDGE:

```cisco
ping vrf 20 10.20.20.1 source Loopback20
```

Expected result:

```text
Success rate is 100 percent
```

Observed result:

```text
Success rate is 100 percent
```

Interpretation:

```text
PASS

HQ VPN 20 can reach Branch VPN 20.
```

---

## 9.2 Branch to HQ VPN 20

Command on BR-SDWAN-EDGE:

```cisco
ping vrf 20 10.10.20.1 source Loopback20
```

Expected result:

```text
Success rate is 100 percent
```

Observed result:

```text
Success rate is 100 percent
```

Interpretation:

```text
PASS

Branch VPN 20 can reach HQ VPN 20.
```

---

## 10. VPN Segmentation Verification

## 10.1 HQ VPN 20 to Branch VPN 10

Command on HQ-SDWAN-EDGE:

```cisco
ping vrf 20 10.20.10.254 source Loopback20
```

Expected result:

```text
Success rate is 0 percent
```

Observed result:

```text
Success rate is 0 percent
```

Interpretation:

```text
PASS

VPN 20 cannot reach VPN 10.
This is expected because no route leaking was configured between VPN 20 and VPN 10.
```

---

## 10.2 Branch VPN 20 to HQ VPN 10

Command on BR-SDWAN-EDGE:

```cisco
ping vrf 20 172.16.7.1 source Loopback20
```

Expected result:

```text
Success rate is 0 percent
```

Observed result:

```text
Success rate is 0 percent
```

Interpretation:

```text
PASS

VPN 20 cannot reach VPN 10.
This confirms SD-WAN service VPN segmentation.
```

---

## 11. vManage GUI Verification

Path:

```text
Configuration > Devices > WAN Edge List
```

Expected state:

```text
HQ-SDWAN-EDGE: Reachable / In Sync
BR-SDWAN-EDGE: Reachable / In Sync or certificate-valid operational state
```

Path:

```text
Configuration > Certificates > WAN Edge List
```

Expected state:

```text
HQ-SDWAN-EDGE certificate installed and valid
BR-SDWAN-EDGE certificate installed and valid
Both WAN Edges validated
```

Observed result:

```text
HQ-SDWAN-EDGE visible in vManage
BR-SDWAN-EDGE visible in vManage
WAN Edge certificate serials present
Certificate expiration dates present
```

Interpretation:

```text
PASS

vManage GUI confirms that both WAN Edges were successfully onboarded and recognized.
```

---

## 12. Final End-to-End Verification Matrix

| Area | Command / Test | Expected | Result |
|---|---|---|---|
| HQ control | `show sdwan control connections` | controllers up | PASS |
| BR control | `show sdwan control connections` | controllers up | PASS |
| HQ OMP | `show sdwan omp peers` | vSmart up | PASS |
| BR OMP | `show sdwan omp peers` | vSmart up | PASS |
| vSmart peers | `show omp peers` | HQ and BR up | PASS |
| vSmart VPN 10 | `show omp routes vpn 10` | HQ/BR routes present | PASS |
| vSmart VPN 20 | `show omp routes vpn 20` | HQ/BR routes present | PASS |
| vSmart TLOC | `show omp tlocs` | 4 TLOCs C,I,R | PASS |
| vBond edges | `show orchestrator valid-vedges` | both valid | PASS |
| HQ BFD | `show sdwan bfd sessions` | 4 sessions up | PASS |
| BR BFD | `show sdwan bfd sessions` | 4 sessions up | PASS |
| HQ VPN 10 route | `show ip route vrf 10` | BR route installed | PASS |
| BR VPN 10 route | `show ip route vrf 10` | HQ route installed | PASS |
| HQ VPN 20 route | `show ip route vrf 20` | BR loopback installed | PASS |
| BR VPN 20 route | `show ip route vrf 20` | HQ loopback installed | PASS |
| VPN 10 reachability | HQ ↔ BR ping | 100% success | PASS |
| VPN 20 reachability | HQ ↔ BR ping | 100% success | PASS |
| Cross-VPN isolation | VPN 20 → VPN 10 ping | 0% success | PASS |

---

## 13. Final Verification Result

Phase 07 verification is complete.

Final result:

```text
PASS
```

The SD-WAN overlay foundation is operational.

Verified final state:

```text
- SD-WAN controllers are operational.
- HQ-SDWAN-EDGE is onboarded.
- BR-SDWAN-EDGE is onboarded.
- Both WAN Edges have valid certificates.
- Both WAN Edges form control connections.
- OMP peering is up.
- OMP routes are exchanged for VPN 10 and VPN 20.
- TLOCs are advertised and resolved.
- BFD overlay sessions are up.
- VPN 10 communication works between HQ and Branch.
- VPN 20 communication works between HQ and Branch.
- VPN 10 and VPN 20 remain isolated from each other.
```

Therefore, Phase 07 successfully validates:

```text
SD-WAN Overlay Foundation
OMP
TLOC
BFD
Service VPN 10
Multi-VPN Segmentation
```
