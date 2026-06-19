# Phase 05 - QoS

## 1. Overview

This phase implements and verifies Quality of Service (QoS) in the `collapsed-core-campus-ver.2` enterprise campus lab.

The main goal of this phase is to understand how QoS is designed, applied, verified, and troubleshot in an enterprise WAN edge environment.

This phase focuses on:

* MQC
* Traffic classification
* Traffic marking
* DSCP-based QoS
* Trust boundary
* Queuing
* Voice QoS
* Video QoS
* Business Critical QoS
* Management QoS
* Scavenger QoS
* WAN edge QoS policy
* QoS verification
* QoS troubleshooting

QoS was implemented on `EDGE1` only.
No QoS configuration was added to CORE or Access switches in this phase.

---

## 2. Lab Repository Context

Repository:

```text
https://github.com/wayne4199/enterprise-network-labs
```

Active lab path:

```text
collapsed-core-campus-ver.2/phase-05-qos-ver.2
```

Previous completed phases:

```text
phase-01-foundation-ver.2
phase-02-enterprise-services-ver.2
phase-03-routing-resiliency-ver.2
phase-04-security-ver.2
```

This phase builds on the final state of:

```text
Phase 04 Security
+ Phase 03 Case 2 Multi-Area OSPF
```

Single Area OSPF was used only as a learning comparison case in Phase 03.
Phase 05 continues from the Multi-Area OSPF operational baseline.

---

## 3. Baseline Assumptions

Phase 05 starts from the following stable baseline:

```text
- VLANs and inter-VLAN routing are already configured.
- HSRP is already configured across CORE1 and CORE2.
- Multi-Area OSPF is already configured.
- CORE1 and CORE2 act as ABRs.
- EDGE1 learns the summarized campus route 10.10.0.0/16.
- WAN primary/backup routing is already configured with IP SLA and object tracking.
- NAT/PAT is already configured on EDGE1.
- Phase 04 access security features are preserved.
```

Important routing baseline:

```text
OSPF Area 0:
- EDGE1 <-> CORE1
- EDGE1 <-> CORE2

OSPF Area 10:
- Campus VLAN SVIs on CORE1 and CORE2

ABRs:
- CORE1
- CORE2

Summary route:
- 10.10.0.0/16
```

Expected EDGE1 route:

```text
O IA 10.10.0.0/16
```

---

## 4. Phase 05 Learning Objectives

The learning objectives of this phase are:

```text
1. Understand QoS design goals in an enterprise WAN edge.
2. Classify different traffic types.
3. Mark traffic using DSCP values.
4. Understand the QoS trust boundary.
5. Understand DSCP and CoS concepts.
6. Configure QoS using MQC.
7. Configure class-map, policy-map, and service-policy.
8. Configure basic queuing behavior.
9. Simulate Voice QoS.
10. Simulate Video QoS.
11. Simulate Business Critical QoS.
12. Protect Management traffic.
13. Mark Scavenger traffic with low priority.
14. Apply WAN egress QoS policy.
15. Verify QoS counters.
16. Troubleshoot QoS classification issues.
```

---

## 5. QoS Concepts Covered

### MQC

MQC stands for:

```text
Modular QoS CLI
```

MQC uses the following structure:

```text
class-map
→ identifies traffic

policy-map
→ defines how traffic should be treated

service-policy
→ applies the policy to an interface
```

### DSCP

DSCP is a Layer 3 QoS marking field in the IP header.

In this phase, DSCP values were used to mark and classify traffic across the WAN edge.

### CoS

CoS is a Layer 2 QoS marking value carried in the 802.1Q tag.

CoS was covered conceptually, but the actual lab implementation focused on DSCP because the QoS policy was applied at the routed WAN edge.

### Trust Boundary

The trust boundary defines where the network begins to trust or remark QoS markings.

In this phase, the trust boundary was placed on EDGE1 campus-facing interfaces.

Final trust boundary:

```text
EDGE1 Gi0/2 input
EDGE1 Gi0/3 input
```

Reason:

```text
- Original internal source IP addresses are still visible before NAT.
- Classification can be performed before NAT/PAT.
- DSCP marking can be preserved through WAN egress.
- Existing Access/Core security configuration is not changed.
```

---

## 6. Topology Impact

No new nodes were required for this phase.

Existing nodes were reused:

```text
EDGE1
CORE1
CORE2
ACC1
ACC2
ACC3
SRV-ACC1
ISP1
ISP2
INTERNET-SW
INTERNET-SRV
EXEC
SALES
IT-OPS
MGMT-SRV1
ROGUE-PC
```

ROGUE-PC was reused as the Scavenger traffic source.

ROGUE-SW-TEST from Phase 04 Security is not required for Phase 05 QoS and can be powered off or removed.

---

## 7. QoS Traffic Classes

The following QoS traffic classes were implemented.

| Class             | Source Host       |    Source IP |  Destination |        DSCP | Treatment     |
| ----------------- | ----------------- | -----------: | -----------: | ----------: | ------------- |
| Voice-like        | EXEC              | 10.10.10.117 | 203.0.113.10 |          EF | Priority 10%  |
| Video-like        | SALES             | 10.10.20.197 | 203.0.113.10 |        AF41 | Bandwidth 20% |
| Business Critical | IT-OPS            | 10.10.30.140 | 203.0.113.10 |        AF31 | Bandwidth 20% |
| Management        | MGMT-SRV1         |  10.10.40.10 | 203.0.113.10 |         CS2 | Bandwidth 10% |
| Scavenger         | ROGUE-PC          | 10.10.20.126 | 203.0.113.10 |         CS1 | Bandwidth 5%  |
| Default           | Any other traffic |          Any |          Any | Best Effort | Fair Queue    |

Note:

```text
EXEC, SALES, and IT-OPS received DHCP addresses during the lab.
The QoS ACLs were updated to match the actual active IP addresses observed during verification.
```

---

## 8. Final QoS Design

The final QoS design uses a two-stage model.

### Stage 1 - Ingress Classification and Marking

Applied on EDGE1 campus-facing interfaces:

```text
EDGE1 Gi0/2 input
EDGE1 Gi0/3 input
```

Purpose:

```text
- Match original internal source IP addresses.
- Classify traffic before NAT/PAT.
- Mark traffic with the correct DSCP value.
```

### Stage 2 - WAN Egress Queuing

Applied on EDGE1 WAN-facing interfaces:

```text
EDGE1 Gi0/0 output
EDGE1 Gi0/1 output
```

Purpose:

```text
- Match traffic by DSCP.
- Apply WAN egress queuing treatment.
- Ensure ISP1 and ISP2 both have the same WAN QoS policy.
```

Final traffic flow:

```text
Campus Host
→ CORE1 or CORE2
→ EDGE1 Gi0/2 or Gi0/3 input
→ ACL-based classification
→ DSCP marking
→ NAT/PAT
→ EDGE1 Gi0/0 or Gi0/1 output
→ DSCP-based WAN queuing
→ ISP / INTERNET-SRV
```

---

## 9. Why the Initial QoS Design Was Changed

The initial design attempted to classify traffic directly on the WAN output interface using original internal source IP addresses.

Initial failed model:

```text
EDGE1 Gi0/0 output
→ Match original 10.10.x.x source IP
→ Mark and queue traffic
```

Problem:

```text
EDGE1 performs NAT/PAT.
By the WAN output stage, packets were no longer reliably matched using original 10.10.x.x inside source addresses.
```

Observed result:

```text
- Ping traffic succeeded.
- Custom QoS class counters stayed at 0.
- class-default counters increased.
- ACL hit counts did not increase.
```

Final solution:

```text
Classify and mark traffic before NAT on EDGE1 inside-facing interfaces.
Queue traffic after NAT on WAN-facing interfaces using DSCP.
```

This became an important QoS troubleshooting lesson in this phase.

---

## 10. EDGE1 QoS Policy Placement

Final EDGE1 interface policy placement:

```text
GigabitEthernet0/0
- To ISP1 primary WAN
- service-policy output WAN-QOS-EGRESS

GigabitEthernet0/1
- To ISP2 backup WAN
- service-policy output WAN-QOS-EGRESS

GigabitEthernet0/2
- To CORE1 campus
- service-policy input CAMPUS-QOS-MARK-IN

GigabitEthernet0/3
- To CORE2 campus
- service-policy input CAMPUS-QOS-MARK-IN
```

---

## 11. QoS Marking Policy

Ingress marking policy:

```text
CAMPUS-QOS-MARK-IN
```

Marking behavior:

| Class                |        DSCP Marking |
| -------------------- | ------------------: |
| CM-VOICE             |                  EF |
| CM-VIDEO             |                AF41 |
| CM-BUSINESS-CRITICAL |                AF31 |
| CM-MGMT              |                 CS2 |
| CM-SCAVENGER         |                 CS1 |
| class-default        | No explicit marking |

---

## 12. WAN Queuing Policy

WAN egress queuing policy:

```text
WAN-QOS-EGRESS
```

Queuing behavior:

| Class                    | Match             | Treatment            |
| ------------------------ | ----------------- | -------------------- |
| CM-WAN-VOICE             | DSCP EF           | priority percent 10  |
| CM-WAN-VIDEO             | DSCP AF41         | bandwidth percent 20 |
| CM-WAN-BUSINESS-CRITICAL | DSCP AF31         | bandwidth percent 20 |
| CM-WAN-MGMT              | DSCP CS2          | bandwidth percent 10 |
| CM-WAN-SCAVENGER         | DSCP CS1          | bandwidth percent 5  |
| class-default            | Any other traffic | fair-queue           |

---

## 13. Verification Summary

All major QoS classes were successfully verified.

### Voice QoS

```text
Source: EXEC 10.10.10.117
Destination: 203.0.113.10
Ingress class: CM-VOICE
Marking: DSCP EF
WAN class: CM-WAN-VOICE
Treatment: priority percent 10
Result: PASS
```

### Video QoS

```text
Source: SALES 10.10.20.197
Destination: 203.0.113.10
Ingress class: CM-VIDEO
Marking: DSCP AF41
WAN class: CM-WAN-VIDEO
Treatment: bandwidth percent 20
Result: PASS
```

### Business Critical QoS

```text
Source: IT-OPS 10.10.30.140
Destination: 203.0.113.10
Ingress class: CM-BUSINESS-CRITICAL
Marking: DSCP AF31
WAN class: CM-WAN-BUSINESS-CRITICAL
Treatment: bandwidth percent 20
Result: PASS
```

### Management QoS

```text
Source: MGMT-SRV1 10.10.40.10
Destination: 203.0.113.10
Ingress class: CM-MGMT
Marking: DSCP CS2
WAN class: CM-WAN-MGMT
Treatment: bandwidth percent 10
Result: PASS
```

### Scavenger QoS

```text
Source: ROGUE-PC 10.10.20.126
Destination: 203.0.113.10
Ingress class: CM-SCAVENGER
Marking: DSCP CS1
WAN class: CM-WAN-SCAVENGER
Treatment: bandwidth percent 5
Result: PASS
```

---

## 14. Verification Commands

The following commands were used to verify QoS operation on EDGE1:

```cisco
show class-map
show policy-map
show policy-map interface GigabitEthernet0/0
show policy-map interface GigabitEthernet0/1
show policy-map interface GigabitEthernet0/2
show policy-map interface GigabitEthernet0/3
show access-lists
show running-config | section class-map
show running-config | section policy-map
show running-config interface GigabitEthernet0/0
show running-config interface GigabitEthernet0/1
show running-config interface GigabitEthernet0/2
show running-config interface GigabitEthernet0/3
```

Counter reset commands:

```cisco
clear counters GigabitEthernet0/0
clear counters GigabitEthernet0/1
clear counters GigabitEthernet0/2
clear counters GigabitEthernet0/3
clear access-list counters
```

---

## 15. Important Troubleshooting Lesson

### Problem

The first QoS design used a WAN output policy that matched original inside source IP addresses.

Result:

```text
Custom class counters stayed at 0.
Only class-default increased.
ACL hit counts did not increase.
```

### Cause

EDGE1 performs NAT/PAT.

The WAN output policy did not reliably see the original internal source IP addresses.

### Resolution

The policy was redesigned:

```text
Before NAT:
- Classify traffic by original source IP.
- Mark traffic with DSCP.

After NAT:
- Queue traffic based on DSCP at WAN egress.
```

Final corrected design:

```text
EDGE1 Gi0/2 and Gi0/3 input:
- CAMPUS-QOS-MARK-IN

EDGE1 Gi0/0 and Gi0/1 output:
- WAN-QOS-EGRESS
```

---

## 16. Known Notes and Constraints

### CML / IOSv Limitation

This lab validates QoS using MQC counters and DSCP-based class matching.

It does not simulate real WAN congestion or real voice/video media behavior.

The purpose is to validate:

```text
- MQC structure
- Classification
- Marking
- Queuing policy placement
- Policy-map counters
- NAT/QoS order-of-operation behavior
```

### DHCP Address Note

Some hosts received DHCP addresses during the lab.

If the lab is restarted and DHCP addresses change, the QoS ACLs may need to be updated.

Affected hosts:

```text
EXEC
SALES
IT-OPS
```

Stable hosts:

```text
MGMT-SRV1 10.10.40.10
ROGUE-PC 10.10.20.126
```

### NAT/PAT Failover Note

NAT/PAT failover is still not fully implemented.

Current NAT/PAT remains tied to the ISP1-facing interface:

```text
EDGE1 Gi0/0
```

This item is carried forward to:

```text
Phase 06 - SD-WAN Underlay Enhancement
```

---

## 17. Devices Modified in This Phase

QoS configuration was applied only to:

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

This was intentional to keep the Phase 04 Security baseline stable and to focus on WAN edge QoS behavior.

---

## 18. Final Status

Phase 05 QoS implementation and verification are complete.

Final result:

```text
VOICE QoS: PASS
VIDEO QoS: PASS
BUSINESS CRITICAL QoS: PASS
MANAGEMENT QoS: PASS
SCAVENGER QoS: PASS
```

Final QoS architecture:

```text
Ingress classification and marking before NAT
+
WAN egress queuing after NAT based on DSCP
```

This phase successfully demonstrates enterprise WAN edge QoS using Cisco MQC.
