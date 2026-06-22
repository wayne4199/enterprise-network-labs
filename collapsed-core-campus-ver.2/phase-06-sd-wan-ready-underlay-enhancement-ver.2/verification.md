# Phase 06 — SD-WAN-Ready Underlay Enhancement Verification

## 1. Purpose

This document records the verification results for:

```text
Phase 06 — SD-WAN-Ready Underlay Enhancement
```

This phase verified that the existing dual-ISP WAN underlay can support:

* Routing failover
* NAT/PAT failover
* IP SLA and Object Tracking
* Track-aware backup default route
* Track-aware Policy-Based Routing
* Application-aware path selection simulation
* QoS and WAN path integration
* Failure and recovery behavior

Final result:

```text
Phase 06 Verification Status: PASS
```

---

## 2. Verification Scope

Most verification was performed on:

```text
EDGE1
```

Traffic was generated from the following hosts:

| Host         |   IP Address | Role                      |
| ------------ | -----------: | ------------------------- |
| EXEC         | 10.10.10.117 | VOICE-like                |
| SALES        | 10.10.20.197 | VIDEO-like                |
| IT-OPS       | 10.10.30.140 | BUSINESS-CRITICAL         |
| MGMT-SRV1    |  10.10.40.10 | MGMT                      |
| ROGUE-PC     | 10.10.20.126 | SCAVENGER                 |
| INTERNET-SRV | 203.0.113.10 | Internet test destination |

---

## 3. EDGE1 Interface Verification

### Command

```cisco
show ip interface brief
```

### Expected Result

```text
GigabitEthernet0/0    100.64.1.2    up/up
GigabitEthernet0/1    100.64.2.2    up/up
GigabitEthernet0/2    172.16.0.1    up/up
GigabitEthernet0/3    172.16.0.5    up/up
```

### Result

```text
PASS
```

EDGE1 interface roles were confirmed:

| Interface | Role                |
| --------- | ------------------- |
| Gi0/0     | ISP1-facing outside |
| Gi0/1     | ISP2-facing outside |
| Gi0/2     | CORE1-facing inside |
| Gi0/3     | CORE2-facing inside |

---

## 4. OSPF Baseline Verification

### Command

```cisco
show ip ospf neighbor
show ip route ospf
```

### Expected Result

EDGE1 should have OSPF neighbors with CORE1 and CORE2.

```text
Neighbor 2.2.2.2 via Gi0/2 FULL
Neighbor 3.3.3.3 via Gi0/3 FULL
```

EDGE1 should learn the summarized campus route:

```text
O IA 10.10.0.0/16 via 172.16.0.2
```

### Result

```text
PASS
```

The Phase 03 Multi-Area OSPF baseline remained stable.

---

## 5. Default Route Verification

### Command

```cisco
show ip route 0.0.0.0
show running-config | include ^ip route
```

### Expected Normal State

```text
0.0.0.0/0 via 100.64.1.1
```

Expected route configuration:

```cisco
ip route 0.0.0.0 0.0.0.0 100.64.1.1 track 1
ip route 0.0.0.0 0.0.0.0 100.64.2.1 10 track 2
ip route 203.0.113.10 255.255.255.255 100.64.1.1
```

### Result

```text
PASS
```

Primary default route used ISP1.
Backup default route existed in the running configuration and was tied to Track 2.

---

## 6. IP SLA and Track Verification

### Command

```cisco
show ip sla statistics
show track
```

### Expected Result

```text
IP SLA 1 = OK
Track 1 = Up

IP SLA 2 = OK
Track 2 = Up
```

### Verified Design

| Object   | Target       | Source Interface | Purpose                     |
| -------- | ------------ | ---------------- | --------------------------- |
| IP SLA 1 | 203.0.113.10 | Gi0/0            | ISP1 path health            |
| Track 1  | IP SLA 1     | N/A              | ISP1 route and PBR tracking |
| IP SLA 2 | 100.64.2.1   | Gi0/1            | ISP2 next-hop health        |
| Track 2  | IP SLA 2     | N/A              | ISP2 route and PBR tracking |

### Result

```text
PASS
```

Both ISP tracking objects were operational.

---

## 7. Baseline NAT/PAT Problem Verification

### Purpose

Verify the original problem before applying Phase 06 NAT/PAT enhancement.

### Original NAT/PAT Configuration

```cisco
ip nat inside source list ENTERPRISE-NAT interface GigabitEthernet0/0 overload
```

### ISP1 Failure Test

EDGE1:

```cisco
configure terminal
interface GigabitEthernet0/0
 shutdown
end
```

### Expected Result Before Fix

```text
Track 1 Down
Default route moves to ISP2
MGMT-SRV1 ping fails
No valid ISP2 PAT translation
```

### Observed Result

```text
Routing failover: PASS
NAT/PAT failover: FAIL
```

Observed failure:

```text
From 172.16.0.1 icmp_seq=1 Destination Host Unreachable
```

NAT translations:

```text
Total active translations: 0
```

### Result

```text
PROBLEM CONFIRMED
```

This confirmed that routing failover alone was not enough.

---

## 8. NAT/PAT Failover Enhancement Verification

### Final NAT/PAT Configuration

```cisco
ip nat inside source route-map NAT-ISP1 interface GigabitEthernet0/0 overload
ip nat inside source route-map NAT-ISP2 interface GigabitEthernet0/1 overload
```

Route-map logic:

```cisco
route-map NAT-ISP1 permit 10
 match ip address ENTERPRISE-NAT
 match interface GigabitEthernet0/0

route-map NAT-ISP2 permit 10
 match ip address ENTERPRISE-NAT
 match interface GigabitEthernet0/1
```

---

### 8.1 ISP1 Normal NAT/PAT Verification

MGMT-SRV1:

```bash
ping -c 4 203.0.113.10
```

EDGE1:

```cisco
show ip nat translations
show ip nat statistics
```

Expected translation:

```text
Inside local  : 10.10.40.10
Inside global : 100.64.1.2
```

Observed NAT state:

```text
NAT-ISP1 refcount 1
NAT-ISP2 refcount 0
```

Result:

```text
PASS
```

---

### 8.2 ISP1 Failure NAT/PAT Failover Verification

EDGE1:

```cisco
clear ip nat translation *

configure terminal
interface GigabitEthernet0/0
 shutdown
end
```

Verification commands:

```cisco
show track
show ip route 0.0.0.0
show ip nat translations
show ip nat statistics
```

Expected state:

```text
Track 1 Down
Default route via 100.64.2.1
```

MGMT-SRV1:

```bash
ping -c 4 203.0.113.10
```

Expected translation:

```text
Inside local  : 10.10.40.10
Inside global : 100.64.2.2
```

Observed NAT state:

```text
NAT-ISP1 refcount 0
NAT-ISP2 refcount 1
```

Result:

```text
PASS
```

---

### 8.3 ISP1 Recovery / Failback Verification

EDGE1:

```cisco
configure terminal
interface GigabitEthernet0/0
 no shutdown
end
```

Verification commands:

```cisco
show ip interface brief
show track
show ip route 0.0.0.0
```

Expected state:

```text
Gi0/0 up/up
Track 1 Up
Default route via 100.64.1.1
```

MGMT-SRV1:

```bash
ping -c 4 203.0.113.10
```

Expected translation:

```text
Inside local  : 10.10.40.10
Inside global : 100.64.1.2
```

Observed NAT state:

```text
NAT-ISP1 refcount 1
NAT-ISP2 refcount 0
```

Result:

```text
PASS
```

---

## 9. ISP2 Track Verification

### Purpose

Verify that ISP2 is explicitly monitored by IP SLA 2 and Track 2.

### ISP2 Failure Test

EDGE1:

```cisco
configure terminal
interface GigabitEthernet0/1
 shutdown
end
```

Verification:

```cisco
show ip sla statistics
show track
show ip route 0.0.0.0
```

Expected state:

```text
IP SLA 2 return code: Timeout
Track 2 Down
Track 1 Up
Default route remains via 100.64.1.1
```

MGMT-SRV1:

```bash
ping -c 4 203.0.113.10
```

Expected result:

```text
Ping success through ISP1
```

NAT expected:

```text
10.10.40.10 → 100.64.1.2
```

Result:

```text
PASS
```

---

### ISP2 Recovery Test

EDGE1:

```cisco
configure terminal
interface GigabitEthernet0/1
 no shutdown
end
```

Verification:

```cisco
show ip interface brief
show ip sla statistics
show track
show ip route 0.0.0.0
```

Expected state:

```text
Gi0/1 up/up
IP SLA 2 OK
Track 2 Up
Default route remains via 100.64.1.1
```

Result:

```text
PASS
```

---

## 10. Track-Aware PBR Configuration Verification

### Command

```cisco
show ip policy
show route-map WAN-BUSINESS-INTENT
```

### Expected Policy Placement

```text
Interface        Route map
Gi0/2            WAN-BUSINESS-INTENT
Gi0/3            WAN-BUSINESS-INTENT
```

### Expected Route-Map Structure

```text
permit 5    VOICE_TRAFFIC_PREFER_ISP1
permit 10   SCAVENGER_TRAFFIC_PREFER_ISP2
permit 20   MGMT_TRAFFIC_PREFER_ISP1
permit 30   VIDEO_TRAFFIC_PREFER_ISP2
permit 40   BUSINESS_CRITICAL_TRAFFIC_PREFER_ISP1
permit 100  DEFAULT_TRAFFIC_USE_NORMAL_ROUTING
```

### Expected Track-Aware Next-Hop State

```text
100.64.1.1 track 1 [up]
100.64.2.1 track 2 [up]
```

### Result

```text
PASS
```

PBR was correctly applied inbound on EDGE1 inside-facing interfaces.

---

## 11. PBR Normal Path Verification

### 11.1 MGMT-SRV1 to ISP1

MGMT-SRV1:

```bash
ping -c 4 203.0.113.10
```

EDGE1:

```cisco
show ip nat translations
show ip nat statistics
show route-map WAN-BUSINESS-INTENT
show policy-map interface GigabitEthernet0/0
```

Expected:

```text
Inside local  : 10.10.40.10
Inside global : 100.64.1.2
PBR sequence 20 match
Gi0/0 CM-WAN-MGMT match
```

Observed:

```text
NAT-ISP1 refcount 1
NAT-ISP2 refcount 0
```

Result:

```text
PASS
```

---

### 11.2 ROGUE-PC to ISP2

ROGUE-PC:

```bash
ping -c 4 203.0.113.10
```

EDGE1:

```cisco
show ip nat translations
show ip nat statistics
show route-map WAN-BUSINESS-INTENT
show policy-map interface GigabitEthernet0/1
```

Expected:

```text
Inside local  : 10.10.20.126
Inside global : 100.64.2.2
PBR sequence 10 match
Gi0/1 CM-WAN-SCAVENGER match
```

Observed:

```text
NAT-ISP1 refcount 0
NAT-ISP2 refcount 1
```

Result:

```text
PASS
```

---

## 12. PBR Fallback Verification

### 12.1 ISP2 Failure — ROGUE-PC Falls Back to ISP1

EDGE1:

```cisco
clear ip nat translation *

configure terminal
interface GigabitEthernet0/1
 shutdown
end
```

Expected state:

```text
Track 2 Down
Track 1 Up
Default route via 100.64.1.1
PBR sequence 10 next-hop 100.64.2.1 track 2 [down]
```

ROGUE-PC:

```bash
ping -c 4 203.0.113.10
```

Expected NAT:

```text
10.10.20.126 → 100.64.1.2
```

Expected QoS:

```text
Gi0/0 CM-WAN-SCAVENGER match
```

Result:

```text
PASS
```

Recovery:

```cisco
configure terminal
interface GigabitEthernet0/1
 no shutdown
end
```

Recovery verification:

```text
Gi0/1 up/up
Track 2 Up
Default route via 100.64.1.1
```

Recovery result:

```text
PASS
```

---

### 12.2 ISP1 Failure — MGMT-SRV1 Falls Back to ISP2

EDGE1:

```cisco
clear ip nat translation *

configure terminal
interface GigabitEthernet0/0
 shutdown
end
```

Expected state:

```text
Track 1 Down
Track 2 Up
Default route via 100.64.2.1
PBR sequence 20 next-hop 100.64.1.1 track 1 [down]
```

MGMT-SRV1:

```bash
ping -c 4 203.0.113.10
```

Expected NAT:

```text
10.10.40.10 → 100.64.2.2
```

Expected QoS:

```text
Gi0/1 CM-WAN-MGMT match
```

Observed:

```text
NAT-ISP1 refcount 0
NAT-ISP2 refcount 2
```

Result:

```text
PASS
```

Recovery:

```cisco
configure terminal
interface GigabitEthernet0/0
 no shutdown
end
```

Recovery verification:

```text
Gi0/0 up/up
Track 1 Up
Track 2 Up
Default route via 100.64.1.1
```

Recovery result:

```text
PASS
```

---

## 13. Application-Aware Path Selection Verification

### 13.1 EXEC / VOICE-like to ISP1

EXEC:

```bash
ping -c 4 203.0.113.10
```

EDGE1:

```cisco
show ip nat translations | include 10.10.10.117
show ip nat statistics
show route-map WAN-BUSINESS-INTENT
show policy-map interface GigabitEthernet0/0
```

Expected:

```text
10.10.10.117 → 100.64.1.2
PBR sequence 5 match
Gi0/0 CM-WAN-VOICE match
```

Observed:

```text
Inside local  : 10.10.10.117
Inside global : 100.64.1.2
```

Result:

```text
PASS
```

---

### 13.2 SALES / VIDEO-like to ISP2

SALES:

```bash
ping -c 4 203.0.113.10
```

EDGE1:

```cisco
show ip nat translations | include 10.10.20.197
show ip nat statistics
show route-map WAN-BUSINESS-INTENT
show policy-map interface GigabitEthernet0/1
```

Expected:

```text
10.10.20.197 → 100.64.2.2
PBR sequence 30 match
Gi0/1 CM-WAN-VIDEO match
```

Observed:

```text
Inside local  : 10.10.20.197
Inside global : 100.64.2.2
NAT-ISP1 refcount 0
NAT-ISP2 refcount 1
```

Result:

```text
PASS
```

---

### 13.3 IT-OPS / BUSINESS-CRITICAL to ISP1

IT-OPS:

```bash
ping -c 4 203.0.113.10
```

EDGE1:

```cisco
show ip nat translations | include 10.10.30.140
show ip nat statistics
show route-map WAN-BUSINESS-INTENT
show policy-map interface GigabitEthernet0/0
```

Expected:

```text
10.10.30.140 → 100.64.1.2
PBR sequence 40 match
Gi0/0 CM-WAN-BUSINESS-CRITICAL match
```

Observed:

```text
Inside local  : 10.10.30.140
Inside global : 100.64.1.2
NAT-ISP1 refcount 1
NAT-ISP2 refcount 0
```

Result:

```text
PASS
```

---

### 13.4 MGMT-SRV1 / MGMT to ISP1

MGMT-SRV1:

```bash
ping -c 4 203.0.113.10
```

EDGE1:

```cisco
show ip nat translations | include 10.10.40.10
show ip nat statistics
show route-map WAN-BUSINESS-INTENT
show policy-map interface GigabitEthernet0/0
```

Expected:

```text
10.10.40.10 → 100.64.1.2
PBR sequence 20 match
Gi0/0 CM-WAN-MGMT match
```

Result:

```text
PASS
```

---

### 13.5 ROGUE-PC / SCAVENGER to ISP2

ROGUE-PC:

```bash
ping -c 4 203.0.113.10
```

EDGE1:

```cisco
show ip nat translations | include 10.10.20.126
show ip nat statistics
show route-map WAN-BUSINESS-INTENT
show policy-map interface GigabitEthernet0/1
```

Expected:

```text
10.10.20.126 → 100.64.2.2
PBR sequence 10 match
Gi0/1 CM-WAN-SCAVENGER match
```

Result:

```text
PASS
```

---

## 14. Application-Aware Fallback Verification

### ISP2 Failure — SALES / VIDEO-like Falls Back to ISP1

EDGE1:

```cisco
clear ip nat translation *

configure terminal
interface GigabitEthernet0/1
 shutdown
end
```

Expected state:

```text
Track 2 Down
Track 1 Up
Default route via 100.64.1.1
PBR sequence 30 next-hop 100.64.2.1 track 2 [down]
```

SALES:

```bash
ping -c 4 203.0.113.10
```

EDGE1:

```cisco
show ip nat translations | include 10.10.20.197
show ip nat statistics
show policy-map interface GigabitEthernet0/0
show route-map WAN-BUSINESS-INTENT
```

Expected NAT:

```text
10.10.20.197 → 100.64.1.2
```

Expected QoS:

```text
Gi0/0 CM-WAN-VIDEO match
```

Result:

```text
PASS
```

Recovery:

```cisco
configure terminal
interface GigabitEthernet0/1
 no shutdown
end
```

Recovery verification:

```text
Gi0/1 up/up
Track 2 Up
Track 1 Up
Default route via 100.64.1.1
```

Recovery result:

```text
PASS
```

---

## 15. QoS + WAN Integration Verification

### Purpose

Verify that QoS marking and WAN egress queuing still work after adding NAT/PAT failover and PBR.

Final traffic flow:

```text
Campus host
→ EDGE1 ingress QoS marking
→ PBR path selection
→ NAT/PAT based on egress WAN interface
→ WAN egress QoS queuing
→ ISP path
```

---

### 15.1 QoS Policy Placement

Command:

```cisco
show policy-map interface GigabitEthernet0/0
show policy-map interface GigabitEthernet0/1
show policy-map interface GigabitEthernet0/2
show policy-map interface GigabitEthernet0/3
```

Expected:

```text
Gi0/2 input  → CAMPUS-QOS-MARK-IN
Gi0/3 input  → CAMPUS-QOS-MARK-IN
Gi0/0 output → WAN-QOS-EGRESS
Gi0/1 output → WAN-QOS-EGRESS
```

Result:

```text
PASS
```

---

### 15.2 Ingress QoS Marking Verification

Expected marking:

| Traffic           | Ingress Interface | DSCP |
| ----------------- | ----------------- | ---- |
| VOICE-like        | Gi0/2             | EF   |
| BUSINESS-CRITICAL | Gi0/2             | AF31 |
| MGMT              | Gi0/2             | CS2  |
| VIDEO-like        | Gi0/3             | AF41 |
| SCAVENGER         | Gi0/3             | CS1  |

Observed:

```text
Gi0/2 marked VOICE EF, BUSINESS AF31, MGMT CS2.
Gi0/3 marked VIDEO AF41, SCAVENGER CS1.
```

Result:

```text
PASS
```

---

### 15.3 WAN Egress QoS Verification

Expected WAN class behavior:

| Traffic                    | Normal WAN | Egress Class             |
| -------------------------- | ---------- | ------------------------ |
| EXEC / VOICE-like          | Gi0/0      | CM-WAN-VOICE             |
| SALES / VIDEO-like         | Gi0/1      | CM-WAN-VIDEO             |
| IT-OPS / BUSINESS-CRITICAL | Gi0/0      | CM-WAN-BUSINESS-CRITICAL |
| MGMT-SRV1 / MGMT           | Gi0/0      | CM-WAN-MGMT              |
| ROGUE-PC / SCAVENGER       | Gi0/1      | CM-WAN-SCAVENGER         |

Fallback behavior:

| Failure   | Traffic              | Fallback WAN | Egress Class     |
| --------- | -------------------- | ------------ | ---------------- |
| ISP2 Down | SALES / VIDEO-like   | Gi0/0        | CM-WAN-VIDEO     |
| ISP2 Down | ROGUE-PC / SCAVENGER | Gi0/0        | CM-WAN-SCAVENGER |
| ISP1 Down | MGMT-SRV1 / MGMT     | Gi0/1        | CM-WAN-MGMT      |

Observed:

```text
Gi0/0 showed VOICE, VIDEO fallback, BUSINESS, MGMT, and SCAVENGER fallback counters.
Gi0/1 showed VIDEO, MGMT fallback, and SCAVENGER counters.
```

Result:

```text
PASS
```

---

## 16. Final Integrated Verification Matrix

| Test                   | Expected Result                       | Status |
| ---------------------- | ------------------------------------- | ------ |
| EDGE1 interfaces up/up | Gi0/0, Gi0/1, Gi0/2, Gi0/3 up/up      | PASS   |
| OSPF neighbors         | CORE1 and CORE2 FULL                  | PASS   |
| OSPF summary route     | `O IA 10.10.0.0/16` on EDGE1          | PASS   |
| Primary default route  | ISP1 via 100.64.1.1                   | PASS   |
| Backup default route   | ISP2 via 100.64.2.1 AD 10 track 2     | PASS   |
| IP SLA 1               | OK                                    | PASS   |
| Track 1                | Up                                    | PASS   |
| IP SLA 2               | OK                                    | PASS   |
| Track 2                | Up                                    | PASS   |
| NAT/PAT ISP1 normal    | Inside global 100.64.1.2              | PASS   |
| NAT/PAT ISP2 failover  | Inside global 100.64.2.2              | PASS   |
| NAT/PAT failback       | Return to 100.64.1.2                  | PASS   |
| PBR policy placement   | Gi0/2 and Gi0/3 inbound               | PASS   |
| MGMT preferred path    | ISP1                                  | PASS   |
| ROGUE preferred path   | ISP2                                  | PASS   |
| MGMT fallback          | ISP2 when ISP1 fails                  | PASS   |
| ROGUE fallback         | ISP1 when ISP2 fails                  | PASS   |
| VOICE-like path        | ISP1                                  | PASS   |
| VIDEO-like path        | ISP2                                  | PASS   |
| BUSINESS-CRITICAL path | ISP1                                  | PASS   |
| VIDEO fallback         | ISP1 when ISP2 fails                  | PASS   |
| QoS ingress marking    | EF, AF41, AF31, CS2, CS1              | PASS   |
| WAN egress QoS         | Correct class on active WAN interface | PASS   |

---

## 17. Final Verification Commands

Use the following commands for final review.

```cisco
!
! Interface and routing
!
show ip interface brief
show ip route 0.0.0.0
show ip route ospf
show ip ospf neighbor

!
! IP SLA and tracking
!
show ip sla statistics
show track

!
! NAT/PAT
!
show ip nat translations
show ip nat statistics
show running-config | include ip nat
show running-config | section route-map NAT

!
! PBR
!
show ip policy
show route-map WAN-BUSINESS-INTENT
show running-config | section route-map WAN-BUSINESS-INTENT

!
! QoS
!
show policy-map interface GigabitEthernet0/0
show policy-map interface GigabitEthernet0/1
show policy-map interface GigabitEthernet0/2
show policy-map interface GigabitEthernet0/3
```

---

## 18. Final Verification Result

Final Phase 06 result:

```text
Routing failover:       PASS
NAT/PAT failover:       PASS
IP SLA / Track:         PASS
PBR path steering:      PASS
Application-aware path: PASS
QoS + WAN integration:  PASS
Failure recovery:       PASS
```

Final status:

```text
Phase 06 — SD-WAN-Ready Underlay Enhancement
Verification Status: Completed
Result: PASS
```
