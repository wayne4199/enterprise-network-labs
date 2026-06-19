# Phase 05 - QoS Design Decisions

## 1. Design Scope

This document records the major design decisions made during:

```text
collapsed-core-campus-ver.2/phase-05-qos-ver.2
```

Phase 05 focuses on enterprise WAN edge QoS.

The main design goals were:

```text
- Understand Cisco MQC structure.
- Classify traffic into different QoS classes.
- Mark traffic using DSCP.
- Apply WAN egress queuing.
- Understand trust boundary placement.
- Verify QoS behavior using policy-map counters.
- Troubleshoot QoS classification issues in a NAT/PAT environment.
```

QoS was implemented only on:

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

## 2. Baseline Decision

### Decision

Phase 05 starts from the final state of:

```text
Phase 04 Security
+
Phase 03 Case 2 - Multi-Area OSPF
```

### Reason

Phase 03 had two OSPF learning cases:

```text
Case 1 - Single Area OSPF
Case 2 - Multi-Area OSPF
```

Single Area OSPF was used as a comparison and learning case.

The operational baseline for Phase 04 and later phases is Multi-Area OSPF because it provides:

```text
- ABR behavior
- Route summarization
- Better scaling model
- More realistic enterprise design
```

Therefore, Phase 05 does not return to the Single Area OSPF baseline.

### Result

Phase 05 QoS was added on top of the following stable baseline:

```text
- VLANs and inter-VLAN routing
- HSRP gateway redundancy
- Multi-Area OSPF
- Campus route summarization
- WAN primary/backup routing
- IP SLA and object tracking
- NAT/PAT
- Phase 04 Layer 2 and management security controls
```

---

## 3. QoS Placement Decision

### Decision

QoS was implemented on `EDGE1` only.

### Reason

EDGE1 is the WAN edge device and is the most appropriate location to study WAN QoS behavior.

WAN links are usually the most likely congestion point in an enterprise design.

Applying QoS only on EDGE1 keeps the design focused and avoids unnecessary changes to the already completed Phase 04 Security baseline.

### Result

QoS policy placement:

```text
EDGE1 Gi0/2 input  -> Ingress marking from CORE1 side
EDGE1 Gi0/3 input  -> Ingress marking from CORE2 side
EDGE1 Gi0/0 output -> WAN egress queuing toward ISP1
EDGE1 Gi0/1 output -> WAN egress queuing toward ISP2
```

---

## 4. No New Node Decision

### Decision

No additional nodes were added for Phase 05.

### Reason

The lab already had enough hosts to simulate different QoS traffic classes.

Existing hosts were reused:

```text
EXEC        -> Voice-like traffic
SALES       -> Video-like traffic
IT-OPS      -> Business Critical traffic
MGMT-SRV1   -> Management traffic
ROGUE-PC    -> Scavenger traffic
```

This also respects the CML Personal 20-node limit.

### Result

No IP Phone or video endpoint was required.

Voice and video QoS were simulated using test traffic and QoS class counters.

---

## 5. Traffic Class Design Decision

### Decision

Five main QoS traffic classes were created:

```text
VOICE
VIDEO
BUSINESS-CRITICAL
MGMT
SCAVENGER
```

A default class was also retained for all other traffic.

### Reason

These classes represent common enterprise QoS categories:

```text
VOICE:
- Low latency traffic
- Highest priority treatment

VIDEO:
- High bandwidth media traffic
- Important but lower priority than voice

BUSINESS-CRITICAL:
- Important enterprise application traffic
- Should receive guaranteed bandwidth

MGMT:
- Network operations and management traffic
- Should be protected for operational stability

SCAVENGER:
- Low priority or non-critical traffic
- Should receive limited bandwidth
```

### Result

Final traffic class mapping:

| Traffic Class     | Source Host |    Source IP |  Destination | DSCP |
| ----------------- | ----------- | -----------: | -----------: | ---: |
| Voice-like        | EXEC        | 10.10.10.117 | 203.0.113.10 |   EF |
| Video-like        | SALES       | 10.10.20.197 | 203.0.113.10 | AF41 |
| Business Critical | IT-OPS      | 10.10.30.140 | 203.0.113.10 | AF31 |
| Management        | MGMT-SRV1   |  10.10.40.10 | 203.0.113.10 |  CS2 |
| Scavenger         | ROGUE-PC    | 10.10.20.126 | 203.0.113.10 |  CS1 |

---

## 6. DSCP Marking Decision

### Decision

The following DSCP values were used:

| Class             |        DSCP |
| ----------------- | ----------: |
| VOICE             |          EF |
| VIDEO             |        AF41 |
| BUSINESS-CRITICAL |        AF31 |
| MGMT              |         CS2 |
| SCAVENGER         |         CS1 |
| DEFAULT           | Best Effort |

### Reason

The selected DSCP values follow common QoS design logic:

```text
EF:
- Used for voice-like low latency traffic.

AF41:
- Used for video-like traffic.

AF31:
- Used for business-critical traffic.

CS2:
- Used for network management traffic.

CS1:
- Used for scavenger or low priority traffic.
```

### Result

Traffic was marked on EDGE1 before NAT/PAT and later matched by DSCP on the WAN egress policy.

---

## 7. Trust Boundary Decision

### Decision

The QoS trust boundary was placed at:

```text
EDGE1 campus-facing input interfaces
```

Specifically:

```text
EDGE1 Gi0/2 input
EDGE1 Gi0/3 input
```

### Reason

EDGE1 is the first routed WAN edge point where traffic can be centrally classified before NAT/PAT.

This allows the policy to match original internal source IP addresses such as:

```text
10.10.10.117
10.10.20.197
10.10.30.140
10.10.40.10
10.10.20.126
```

The Access and Core switches were not modified, which preserves the Phase 04 Security configuration.

### Result

Ingress marking policy:

```text
CAMPUS-QOS-MARK-IN
```

was applied to:

```text
EDGE1 Gi0/2 input
EDGE1 Gi0/3 input
```

---

## 8. Initial WAN Output Classification Decision and Rejection

### Initial Decision

At first, one combined WAN output policy was attempted.

The initial model was:

```text
EDGE1 WAN output
→ Match original 10.10.x.x source IP
→ Mark traffic
→ Apply queuing
```

### Observed Problem

Traffic successfully reached the destination, but QoS custom class counters did not increase.

Observed behavior:

```text
- Pings to 203.0.113.10 succeeded.
- VOICE / VIDEO / BUSINESS / MGMT / SCAVENGER counters stayed at 0.
- Only class-default increased.
- QoS ACL hit counts did not increase.
```

### Root Cause

EDGE1 performs NAT/PAT.

By the WAN output stage, the policy did not reliably see the original internal source IP addresses.

Therefore, ACL-based matching on the WAN output interface was not suitable.

### Final Decision

The initial combined WAN output policy was rejected.

The final design separates QoS into two stages:

```text
1. Ingress marking before NAT
2. WAN egress queuing after NAT using DSCP
```

### Result

The initial test policy was removed:

```cisco
no policy-map WAN-QOS-POLICY
```

Final remaining policy maps:

```text
CAMPUS-QOS-MARK-IN
WAN-QOS-EGRESS
```

---

## 9. Marking and Queuing Separation Decision

### Decision

QoS was split into two policy stages.

### Stage 1 - Ingress Marking

Applied on:

```text
EDGE1 Gi0/2 input
EDGE1 Gi0/3 input
```

Purpose:

```text
- Match original internal source IP addresses.
- Mark traffic before NAT/PAT.
```

Policy:

```text
CAMPUS-QOS-MARK-IN
```

### Stage 2 - WAN Egress Queuing

Applied on:

```text
EDGE1 Gi0/0 output
EDGE1 Gi0/1 output
```

Purpose:

```text
- Match DSCP-marked traffic.
- Apply WAN queuing behavior.
```

Policy:

```text
WAN-QOS-EGRESS
```

### Reason

This design follows a more practical QoS model:

```text
Classify and mark early.
Queue at the congestion point.
```

### Result

Final QoS traffic flow:

```text
Campus Host
→ CORE1 or CORE2
→ EDGE1 Gi0/2 or Gi0/3 input
→ Original IP classification
→ DSCP marking
→ NAT/PAT
→ EDGE1 Gi0/0 or Gi0/1 output
→ DSCP-based queuing
→ WAN
```

---

## 10. WAN Queuing Decision

### Decision

The WAN egress policy uses the following queuing treatments:

| WAN Class                | Match             | Treatment            |
| ------------------------ | ----------------- | -------------------- |
| CM-WAN-VOICE             | DSCP EF           | priority percent 10  |
| CM-WAN-VIDEO             | DSCP AF41         | bandwidth percent 20 |
| CM-WAN-BUSINESS-CRITICAL | DSCP AF31         | bandwidth percent 20 |
| CM-WAN-MGMT              | DSCP CS2          | bandwidth percent 10 |
| CM-WAN-SCAVENGER         | DSCP CS1          | bandwidth percent 5  |
| class-default            | Any other traffic | fair-queue           |

### Reason

Voice-like traffic should receive priority queue treatment because it represents low-latency traffic.

Video and Business Critical traffic should receive bandwidth guarantees.

Management traffic should receive protected bandwidth for operational stability.

Scavenger traffic should receive limited bandwidth because it is low priority.

Default traffic remains best effort with fair-queue.

### Result

The WAN egress policy was applied to both WAN interfaces:

```text
EDGE1 Gi0/0 output
EDGE1 Gi0/1 output
```

---

## 11. ISP1 and ISP2 Policy Consistency Decision

### Decision

The same WAN egress QoS policy was applied to both ISP-facing interfaces.

```text
EDGE1 Gi0/0 -> ISP1 primary WAN
EDGE1 Gi0/1 -> ISP2 backup WAN
```

### Reason

Phase 03 already introduced WAN resiliency using IP SLA and object tracking.

If failover occurs from ISP1 to ISP2, QoS should remain consistent.

### Result

Both ISP links use:

```text
WAN-QOS-EGRESS
```

This prepares the lab for future SD-WAN Underlay Enhancement work in Phase 06.

---

## 12. Access/Core QoS Decision

### Decision

No QoS policy was applied to Access or Core switches.

### Reason

Phase 05 focuses on WAN edge QoS.

Access and Core switches already contain important Phase 04 Security controls, including:

```text
- Port Security
- DHCP Snooping
- Dynamic ARP Inspection
- BPDU Guard
- Management ACL
```

Adding QoS trust or marking at the access layer would increase complexity and risk changing the security baseline.

### Result

Access/Core QoS remains out of scope for Phase 05.

Possible future enhancement:

```text
- Access layer trust boundary
- IP phone trust model
- CoS-to-DSCP mapping
- Campus QoS policy expansion
```

---

## 13. Simulated Voice and Video Decision

### Decision

Real IP phones and video endpoints were not added.

### Reason

The goal of Phase 05 was to learn QoS policy structure, classification, marking, and queuing.

Real voice/video devices are not required to validate:

```text
- class-map matching
- DSCP marking
- WAN egress DSCP matching
- policy-map counters
```

### Result

Voice-like and video-like traffic were simulated using ICMP traffic from existing hosts.

This kept the topology simple and avoided unnecessary node additions.

---

## 14. Scavenger Traffic Decision

### Decision

ROGUE-PC was reused as the Scavenger traffic source.

### Reason

ROGUE-PC already existed from Phase 04 Security testing.

It provided a useful host to represent low-priority or non-critical traffic.

Scavenger traffic was marked as:

```text
DSCP CS1
```

### Result

ROGUE-PC traffic was successfully classified and marked as Scavenger traffic.

---

## 15. DHCP Address Handling Decision

### Decision

QoS ACLs were updated to match actual DHCP-assigned host IP addresses.

### Reason

Some test hosts did not use the originally assumed IP addresses.

Observed active IP addresses:

```text
EXEC      10.10.10.117
SALES     10.10.20.197
IT-OPS    10.10.30.140
```

Stable IP addresses:

```text
MGMT-SRV1 10.10.40.10
ROGUE-PC  10.10.20.126
```

### Result

QoS ACLs were updated to match the actual active lab state.

Important note:

```text
If DHCP addresses change after a lab restart, the QoS ACLs may need to be updated.
```

Possible future improvement:

```text
Use DHCP reservations for QoS test hosts.
```

---

## 16. Verification Method Decision

### Decision

QoS was verified using MQC counters and ACL hit counts.

Verification commands included:

```cisco
show policy-map interface GigabitEthernet0/2
show policy-map interface GigabitEthernet0/3
show policy-map interface GigabitEthernet0/0
show policy-map interface GigabitEthernet0/1
show access-lists
```

### Reason

CML IOSv is suitable for validating MQC policy structure and counters, but it is not intended to fully simulate real WAN congestion behavior.

Therefore, the main validation targets were:

```text
- ACL match count
- class-map packet count
- DSCP marking count
- WAN egress DSCP class count
```

### Result

All five traffic classes were successfully verified.

```text
VOICE: PASS
VIDEO: PASS
BUSINESS-CRITICAL: PASS
MGMT: PASS
SCAVENGER: PASS
```

---

## 17. CML Lab Constraint Decision

### Decision

Phase 05 did not attempt to validate real congestion behavior or real voice/video media quality.

### Reason

The lab is running in CML using virtual IOSv/IOSvL2 nodes.

The practical goal of this phase is to validate QoS logic, not real physical link performance.

### Result

The lab focused on:

```text
- MQC configuration
- Policy placement
- Classification
- Marking
- DSCP-based queuing
- Counter verification
- Troubleshooting NAT/QoS interaction
```

Real congestion testing and advanced shaping can be considered in a later phase if needed.

---

## 18. NAT/PAT Failover Carry-Forward Decision

### Decision

NAT/PAT failover remains out of scope for Phase 05.

### Reason

Phase 05 focuses on QoS.

The existing NAT/PAT configuration is still tied to the ISP1-facing interface.

Current known limitation:

```text
NAT/PAT failover is not fully implemented.
```

### Result

This item remains carried forward to:

```text
Phase 06 - SD-WAN Underlay Enhancement
```

QoS was applied to both ISP1 and ISP2 interfaces so that the QoS policy is ready for future WAN failover enhancement.

---

## 19. Final Design Result

Final Phase 05 QoS design:

```text
Ingress classification and marking before NAT
+
WAN egress queuing after NAT based on DSCP
```

Final policy placement:

```text
EDGE1 Gi0/2 input:
- CAMPUS-QOS-MARK-IN

EDGE1 Gi0/3 input:
- CAMPUS-QOS-MARK-IN

EDGE1 Gi0/0 output:
- WAN-QOS-EGRESS

EDGE1 Gi0/1 output:
- WAN-QOS-EGRESS
```

Final verification result:

```text
VOICE QoS: PASS
VIDEO QoS: PASS
BUSINESS CRITICAL QoS: PASS
MANAGEMENT QoS: PASS
SCAVENGER QoS: PASS
```

Phase 05 successfully demonstrated enterprise WAN edge QoS using Cisco MQC.
