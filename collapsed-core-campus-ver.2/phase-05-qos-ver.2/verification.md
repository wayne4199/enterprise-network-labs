# Phase 05 - QoS Verification

## 1. Verification Scope

This document records the verification results for:

```text
collapsed-core-campus-ver.2/phase-05-qos-ver.2
```

Phase 05 verifies QoS using Cisco MQC on `EDGE1`.

Main verification goals:

```text
- Confirm the Phase 04 / Phase 03 Multi-Area OSPF baseline is stable.
- Confirm QoS policies are correctly applied on EDGE1.
- Confirm traffic is classified before NAT/PAT.
- Confirm traffic is marked with the correct DSCP value.
- Confirm WAN egress policy matches traffic using DSCP.
- Confirm all planned QoS classes work as expected.
```

QoS was configured only on:

```text
EDGE1
```

No QoS configuration was added to:

```text
CORE1
CORE2
ACC1
ACC2
ACC3
SRV-ACC1
ISP1
ISP2
INTERNET-SRV
```

---

## 2. Baseline Verification Before QoS

Before applying QoS, the existing routing, WAN, NAT, and security baseline was verified.

### 2.1 EDGE1 Interface Status

Verified EDGE1 interfaces:

```text
EDGE1 Gi0/0 100.64.1.2  up/up  -> ISP1
EDGE1 Gi0/1 100.64.2.2  up/up  -> ISP2
EDGE1 Gi0/2 172.16.0.1  up/up  -> CORE1
EDGE1 Gi0/3 172.16.0.5  up/up  -> CORE2
```

Verification command:

```cisco
show ip interface brief
```

Result:

```text
PASS
```

---

### 2.2 OSPF Neighbor Verification

EDGE1 OSPF neighbors were verified.

Expected neighbors:

```text
CORE1 router-id 2.2.2.2 via EDGE1 Gi0/2
CORE2 router-id 3.3.3.3 via EDGE1 Gi0/3
```

Observed state:

```text
Neighbor 2.2.2.2 FULL via 172.16.0.2 Gi0/2
Neighbor 3.3.3.3 FULL via 172.16.0.6 Gi0/3
```

Verification command:

```cisco
show ip ospf neighbor
```

Result:

```text
PASS
```

---

### 2.3 Campus Summary Route Verification

EDGE1 learned the summarized campus route from the Multi-Area OSPF design.

Expected route:

```text
O IA 10.10.0.0/16
```

Observed route:

```text
O IA 10.10.0.0/16 [110/2] via 172.16.0.2, GigabitEthernet0/2
```

Verification command:

```cisco
show ip route ospf
```

Result:

```text
PASS
```

---

### 2.4 WAN Primary / Backup Verification

EDGE1 still uses ISP1 as the primary WAN path.

Observed routing behavior:

```text
Primary default route via ISP1 100.64.1.1
Backup floating default route via ISP2 100.64.2.1 AD 10
```

IP SLA and object tracking were also verified.

Observed result:

```text
IP SLA return code: OK
Track reachability: Up
```

Verification commands:

```cisco
show ip route
show ip sla statistics
show track
show running-config | section ip route
```

Result:

```text
PASS
```

---

### 2.5 NAT/PAT Baseline Verification

EDGE1 NAT/PAT was verified.

Observed NAT/PAT configuration:

```cisco
ip nat inside source list ENTERPRISE-NAT interface GigabitEthernet0/0 overload
```

Important note:

```text
NAT/PAT failover is still not fully implemented.
Current NAT/PAT remains tied to the ISP1-facing interface Gi0/0.
This item is carried forward to Phase 06 - SD-WAN Underlay Enhancement.
```

Verification command:

```cisco
show running-config | include ip nat
```

Result:

```text
PASS with known design limitation
```

---

### 2.6 Existing QoS Baseline

Before Phase 05 QoS was applied, no existing service-policy was observed on EDGE1, CORE1, CORE2, or access switches.

Verification command:

```cisco
show policy-map interface
```

Result:

```text
PASS
```

---

## 3. Final QoS Policy Placement Verification

Final QoS policy placement on EDGE1:

```text
Gi0/2 input  -> CAMPUS-QOS-MARK-IN
Gi0/3 input  -> CAMPUS-QOS-MARK-IN

Gi0/0 output -> WAN-QOS-EGRESS
Gi0/1 output -> WAN-QOS-EGRESS
```

Verification commands:

```cisco
show running-config interface GigabitEthernet0/0
show running-config interface GigabitEthernet0/1
show running-config interface GigabitEthernet0/2
show running-config interface GigabitEthernet0/3
```

Expected result:

```cisco
interface GigabitEthernet0/0
 service-policy output WAN-QOS-EGRESS

interface GigabitEthernet0/1
 service-policy output WAN-QOS-EGRESS

interface GigabitEthernet0/2
 service-policy input CAMPUS-QOS-MARK-IN

interface GigabitEthernet0/3
 service-policy input CAMPUS-QOS-MARK-IN
```

Observed result:

```text
PASS
```

---

## 4. Final QoS Policy Structure Verification

### 4.1 Ingress Marking Policy

Expected policy:

```text
CAMPUS-QOS-MARK-IN
```

Expected behavior:

| Class                | Match Type        |             Marking |
| -------------------- | ----------------- | ------------------: |
| CM-VOICE             | ACL               |             DSCP EF |
| CM-VIDEO             | ACL               |           DSCP AF41 |
| CM-BUSINESS-CRITICAL | ACL               |           DSCP AF31 |
| CM-MGMT              | ACL               |            DSCP CS2 |
| CM-SCAVENGER         | ACL               |            DSCP CS1 |
| class-default        | Any other traffic | No explicit marking |

Verification command:

```cisco
show policy-map
show running-config | section policy-map
```

Result:

```text
PASS
```

---

### 4.2 WAN Egress Queuing Policy

Expected policy:

```text
WAN-QOS-EGRESS
```

Expected behavior:

| Class                    | Match Type        | WAN Treatment        |
| ------------------------ | ----------------- | -------------------- |
| CM-WAN-VOICE             | DSCP EF           | priority percent 10  |
| CM-WAN-VIDEO             | DSCP AF41         | bandwidth percent 20 |
| CM-WAN-BUSINESS-CRITICAL | DSCP AF31         | bandwidth percent 20 |
| CM-WAN-MGMT              | DSCP CS2          | bandwidth percent 10 |
| CM-WAN-SCAVENGER         | DSCP CS1          | bandwidth percent 5  |
| class-default            | Any other traffic | fair-queue           |

Verification command:

```cisco
show policy-map
show running-config | section policy-map
```

Result:

```text
PASS
```

---

## 5. QoS Verification Method

Each QoS class was verified using the following process:

```text
1. Clear interface counters and ACL counters.
2. Generate ICMP traffic from the test host to INTERNET-SRV.
3. Verify ACL hit count.
4. Verify EDGE1 campus-facing input policy counter.
5. Verify DSCP marking counter.
6. Verify EDGE1 WAN-facing output policy counter.
7. Confirm the correct WAN egress class matched the marked DSCP value.
```

Counter reset commands:

```cisco
clear counters GigabitEthernet0/0
clear counters GigabitEthernet0/1
clear counters GigabitEthernet0/2
clear counters GigabitEthernet0/3
clear access-list counters
```

Main verification commands:

```cisco
show policy-map interface GigabitEthernet0/2
show policy-map interface GigabitEthernet0/3
show policy-map interface GigabitEthernet0/0
show policy-map interface GigabitEthernet0/1
show access-lists
```

Traffic destination:

```text
INTERNET-SRV 203.0.113.10
```

---

## 6. Voice QoS Verification

### 6.1 Test Traffic

```text
Source Host: EXEC
Source IP: 10.10.10.117
Destination: 203.0.113.10
Traffic Type: ICMP
QoS Class: Voice-like
Expected DSCP: EF
Expected WAN Treatment: priority percent 10
```

Test command on EXEC:

```bash
ping 203.0.113.10
```

### 6.2 Expected Path

```text
EXEC
→ ACC1
→ CORE1
→ EDGE1 Gi0/2 input
→ DSCP EF marking
→ EDGE1 Gi0/0 output
→ WAN-QOS-EGRESS CM-WAN-VOICE
```

### 6.3 Observed Ingress Marking Result

Observed on EDGE1 Gi0/2:

```text
Class-map: CM-VOICE
  3 packets, 294 bytes
  Match: access-group name QOS-VOICE-LIKE
  QoS Set:
    dscp ef
    Packets marked 3
```

### 6.4 Observed WAN Egress Result

Observed on EDGE1 Gi0/0:

```text
Class-map: CM-WAN-VOICE
  4 packets, 384 bytes
  Match: dscp ef
  Priority: 10%
```

### 6.5 Observed ACL Match

```text
QOS-VOICE-LIKE
permit icmp host 10.10.10.117 host 203.0.113.10 (3 matches)
```

### 6.6 Result

```text
VOICE QoS Verification: PASS
```

---

## 7. Video QoS Verification

### 7.1 Test Traffic

```text
Source Host: SALES
Source IP: 10.10.20.197
Destination: 203.0.113.10
Traffic Type: ICMP
QoS Class: Video-like
Expected DSCP: AF41
Expected WAN Treatment: bandwidth percent 20
```

Test command on SALES:

```bash
ping 203.0.113.10
```

### 7.2 Expected Path

```text
SALES
→ ACC2
→ CORE2
→ EDGE1 Gi0/3 input
→ DSCP AF41 marking
→ EDGE1 Gi0/0 output
→ WAN-QOS-EGRESS CM-WAN-VIDEO
```

### 7.3 Observed Ingress Marking Result

Observed on EDGE1 Gi0/3:

```text
Class-map: CM-VIDEO
  6 packets, 588 bytes
  Match: access-group name QOS-VIDEO-LIKE
  QoS Set:
    dscp af41
    Packets marked 6
```

### 7.4 Observed WAN Egress Result

Observed on EDGE1 Gi0/0:

```text
Class-map: CM-WAN-VIDEO
  3 packets, 294 bytes
  Match: dscp af41
  Bandwidth: 20%
```

### 7.5 Observed ACL Match

```text
QOS-VIDEO-LIKE
permit icmp host 10.10.20.197 host 203.0.113.10 (3 matches)
```

### 7.6 Result

```text
VIDEO QoS Verification: PASS
```

---

## 8. Business Critical QoS Verification

### 8.1 Test Traffic

```text
Source Host: IT-OPS
Source IP: 10.10.30.140
Destination: 203.0.113.10
Traffic Type: ICMP
QoS Class: Business Critical
Expected DSCP: AF31
Expected WAN Treatment: bandwidth percent 20
```

Test command on IT-OPS:

```bash
ping 203.0.113.10
```

### 8.2 Expected Path

```text
IT-OPS
→ ACC3
→ CORE1
→ EDGE1 Gi0/2 input
→ DSCP AF31 marking
→ EDGE1 Gi0/0 output
→ WAN-QOS-EGRESS CM-WAN-BUSINESS-CRITICAL
```

### 8.3 Observed Ingress Marking Result

Observed on EDGE1 Gi0/2:

```text
Class-map: CM-BUSINESS-CRITICAL
  3 packets, 294 bytes
  Match: access-group name QOS-BUSINESS-CRITICAL
  QoS Set:
    dscp af31
    Packets marked 3
```

### 8.4 Observed WAN Egress Result

Observed on EDGE1 Gi0/0:

```text
Class-map: CM-WAN-BUSINESS-CRITICAL
  3 packets, 294 bytes
  Match: dscp af31
  Bandwidth: 20%
```

### 8.5 Observed ACL Match

```text
QOS-BUSINESS-CRITICAL
permit icmp host 10.10.30.140 host 203.0.113.10 (3 matches)
```

### 8.6 Result

```text
BUSINESS CRITICAL QoS Verification: PASS
```

---

## 9. Management QoS Verification

### 9.1 Test Traffic

```text
Source Host: MGMT-SRV1
Source IP: 10.10.40.10
Destination: 203.0.113.10
Traffic Type: ICMP
QoS Class: Management
Expected DSCP: CS2
Expected WAN Treatment: bandwidth percent 10
```

Test command on MGMT-SRV1:

```bash
ping -c 20 203.0.113.10
```

### 9.2 Expected Path

```text
MGMT-SRV1
→ SRV-ACC1
→ CORE1
→ EDGE1 Gi0/2 input
→ DSCP CS2 marking
→ EDGE1 Gi0/0 output
→ WAN-QOS-EGRESS CM-WAN-MGMT
```

### 9.3 Observed Ingress Marking Result

Observed on EDGE1 Gi0/2:

```text
Class-map: CM-MGMT
  4 packets, 392 bytes
  Match: access-group name QOS-MGMT
  QoS Set:
    dscp cs2
    Packets marked 4
```

### 9.4 Observed WAN Egress Result

Observed on EDGE1 Gi0/0:

```text
Class-map: CM-WAN-MGMT
  4 packets, 392 bytes
  Match: dscp cs2
  Bandwidth: 10%
```

### 9.5 Observed ACL Match

```text
QOS-MGMT
permit icmp host 10.10.40.10 host 203.0.113.10 (4 matches)
```

### 9.6 Result

```text
MANAGEMENT QoS Verification: PASS
```

---

## 10. Scavenger QoS Verification

### 10.1 Test Traffic

```text
Source Host: ROGUE-PC
Source IP: 10.10.20.126
Destination: 203.0.113.10
Traffic Type: ICMP
QoS Class: Scavenger
Expected DSCP: CS1
Expected WAN Treatment: bandwidth percent 5
```

Test command on ROGUE-PC:

```bash
ping 203.0.113.10
```

### 10.2 Expected Path

```text
ROGUE-PC
→ ACC2
→ CORE2
→ EDGE1 Gi0/3 input
→ DSCP CS1 marking
→ EDGE1 Gi0/0 output
→ WAN-QOS-EGRESS CM-WAN-SCAVENGER
```

### 10.3 Observed Ingress Marking Result

Observed on EDGE1 Gi0/3:

```text
Class-map: CM-SCAVENGER
  5 packets, 490 bytes
  Match: access-group name QOS-SCAVENGER
  QoS Set:
    dscp cs1
    Packets marked 5
```

### 10.4 Observed WAN Egress Result

Observed on EDGE1 Gi0/0:

```text
Class-map: CM-WAN-SCAVENGER
  5 packets, 490 bytes
  Match: dscp cs1
  Bandwidth: 5%
```

### 10.5 Observed ACL Match

```text
QOS-SCAVENGER
permit icmp host 10.10.20.126 host 203.0.113.10 (5 matches)
```

### 10.6 Result

```text
SCAVENGER QoS Verification: PASS
```

---

## 11. Initial Failed Verification and Resolution

### 11.1 Initial Failed Design

The first QoS design attempted to apply classification, marking, and queuing directly on the WAN output interface.

Initial model:

```text
EDGE1 Gi0/0 output
→ Match original 10.10.x.x source IP
→ Mark traffic
→ Queue traffic
```

### 11.2 Observed Failure

Campus hosts successfully reached `203.0.113.10`.

However, QoS custom classes did not match.

Observed behavior:

```text
CM-VOICE              0 packets
CM-VIDEO              0 packets
CM-BUSINESS-CRITICAL  0 packets
CM-MGMT               0 packets
CM-SCAVENGER          0 packets

class-default         increased
```

ACL hit counts also did not increase.

### 11.3 Cause

EDGE1 performs NAT/PAT.

The WAN output service-policy did not reliably see the original internal source IP addresses.

Therefore, ACL-based classification using original `10.10.x.x` addresses failed on the WAN output interface.

### 11.4 Resolution

QoS was redesigned into two stages:

```text
1. EDGE1 campus-facing input:
   - Match original internal source IP.
   - Mark DSCP before NAT.

2. EDGE1 WAN-facing output:
   - Match DSCP after marking.
   - Apply queuing.
```

Final working policy placement:

```text
Gi0/2 input  -> CAMPUS-QOS-MARK-IN
Gi0/3 input  -> CAMPUS-QOS-MARK-IN

Gi0/0 output -> WAN-QOS-EGRESS
Gi0/1 output -> WAN-QOS-EGRESS
```

Result:

```text
PASS
```

---

## 12. Final Verification Summary

| QoS Class         | Source                | DSCP | Ingress Interface | WAN Interface | Result |
| ----------------- | --------------------- | ---: | ----------------- | ------------- | ------ |
| Voice-like        | EXEC 10.10.10.117     |   EF | Gi0/2 input       | Gi0/0 output  | PASS   |
| Video-like        | SALES 10.10.20.197    | AF41 | Gi0/3 input       | Gi0/0 output  | PASS   |
| Business Critical | IT-OPS 10.10.30.140   | AF31 | Gi0/2 input       | Gi0/0 output  | PASS   |
| Management        | MGMT-SRV1 10.10.40.10 |  CS2 | Gi0/2 input       | Gi0/0 output  | PASS   |
| Scavenger         | ROGUE-PC 10.10.20.126 |  CS1 | Gi0/3 input       | Gi0/0 output  | PASS   |

Overall result:

```text
Phase 05 QoS Verification: PASS
```

---

## 13. Useful Verification Commands

### EDGE1 QoS Policy Verification

```cisco
show class-map
show policy-map
show running-config | section class-map
show running-config | section policy-map
```

### EDGE1 Interface Policy Verification

```cisco
show running-config interface GigabitEthernet0/0
show running-config interface GigabitEthernet0/1
show running-config interface GigabitEthernet0/2
show running-config interface GigabitEthernet0/3
```

### EDGE1 QoS Counter Verification

```cisco
show policy-map interface GigabitEthernet0/0
show policy-map interface GigabitEthernet0/1
show policy-map interface GigabitEthernet0/2
show policy-map interface GigabitEthernet0/3
```

### EDGE1 ACL Hit Count Verification

```cisco
show access-lists QOS-VOICE-LIKE
show access-lists QOS-VIDEO-LIKE
show access-lists QOS-BUSINESS-CRITICAL
show access-lists QOS-MGMT
show access-lists QOS-SCAVENGER
```

### Counter Reset

```cisco
clear counters GigabitEthernet0/0
clear counters GigabitEthernet0/1
clear counters GigabitEthernet0/2
clear counters GigabitEthernet0/3
clear access-list counters
```

### Routing / WAN Baseline Verification

```cisco
show ip interface brief
show ip route
show ip route ospf
show ip ospf neighbor
show ip sla statistics
show track
show running-config | section ip route
```

### NAT/PAT Verification

```cisco
show running-config | include ip nat
show ip nat translations
show ip nat statistics
```

### Host IP Verification

On Linux-based hosts:

```bash
ip -4 addr
ip route
```

---

## 14. Verification Notes

### Note 1 - DHCP Address Dependency

Some QoS ACLs depend on DHCP-assigned host addresses.

Observed active host IPs:

```text
EXEC      10.10.10.117
SALES     10.10.20.197
IT-OPS    10.10.30.140
MGMT-SRV1 10.10.40.10
ROGUE-PC  10.10.20.126
```

If DHCP addresses change after a lab restart, update the QoS ACLs or configure DHCP reservations.

---

### Note 2 - ISP2 Counter Behavior

`Gi0/1` has `WAN-QOS-EGRESS` applied, but normal test traffic uses ISP1 through `Gi0/0`.

This is expected because ISP1 is the primary WAN path.

`Gi0/1` is prepared for backup WAN behavior and future failover testing.

---

### Note 3 - CML / IOSv Limitation

This lab verifies QoS logic using:

```text
- ACL hit counts
- class-map matches
- DSCP marking counters
- WAN egress DSCP class counters
```

This lab does not prove real physical WAN congestion behavior or real voice/video media quality.

---

## 15. Final Status

Final Phase 05 verification result:

```text
VOICE QoS: PASS
VIDEO QoS: PASS
BUSINESS CRITICAL QoS: PASS
MANAGEMENT QoS: PASS
SCAVENGER QoS: PASS
```

Final QoS architecture verified:

```text
Ingress classification and DSCP marking before NAT
+
WAN egress queuing after NAT using DSCP
```

Phase 05 QoS verification is complete.
