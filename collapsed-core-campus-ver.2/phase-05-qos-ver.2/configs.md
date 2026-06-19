# Phase 05 - QoS Configurations

## 1. Configuration Scope

This document records the final QoS configuration for:

```text
collapsed-core-campus-ver.2/phase-05-qos-ver.2
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

Reason:

```text
Phase 05 focuses on WAN edge QoS behavior.
Existing Phase 04 Security baseline should remain unchanged.
```

---

## 2. Final QoS Design

The final QoS design uses two separate policy stages.

```text
Stage 1:
EDGE1 campus-facing input
- Classify traffic using original internal source IP addresses.
- Mark traffic with DSCP before NAT/PAT.

Stage 2:
EDGE1 WAN-facing output
- Match traffic using DSCP.
- Apply WAN egress queuing.
```

Final QoS flow:

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

This design was selected because the initial attempt to match original inside source IP addresses directly on the WAN output interface failed due to NAT/PAT behavior.

---

## 3. QoS Class Summary

| Traffic Class     | Source Host |    Source IP |  Destination |        DSCP | WAN Treatment |
| ----------------- | ----------- | -----------: | -----------: | ----------: | ------------- |
| Voice-like        | EXEC        | 10.10.10.117 | 203.0.113.10 |          EF | Priority 10%  |
| Video-like        | SALES       | 10.10.20.197 | 203.0.113.10 |        AF41 | Bandwidth 20% |
| Business Critical | IT-OPS      | 10.10.30.140 | 203.0.113.10 |        AF31 | Bandwidth 20% |
| Management        | MGMT-SRV1   |  10.10.40.10 | 203.0.113.10 |         CS2 | Bandwidth 10% |
| Scavenger         | ROGUE-PC    | 10.10.20.126 | 203.0.113.10 |         CS1 | Bandwidth 5%  |
| Default           | Any         |          Any |          Any | Best Effort | Fair Queue    |

Note:

```text
EXEC, SALES, and IT-OPS received DHCP addresses during the lab.
The QoS ACLs were updated to match the actual active host IP addresses observed during verification.
```

---

## 4. EDGE1 - QoS Classification ACLs

```cisco
enable
configure terminal

ip access-list extended QOS-VOICE-LIKE
 permit icmp host 10.10.10.117 host 203.0.113.10
exit

ip access-list extended QOS-VIDEO-LIKE
 permit icmp host 10.10.20.197 host 203.0.113.10
exit

ip access-list extended QOS-BUSINESS-CRITICAL
 permit icmp host 10.10.30.140 host 203.0.113.10
exit

ip access-list extended QOS-MGMT
 remark MGMT traffic simulation: MGMT-SRV1 to INTERNET-SRV
 permit icmp host 10.10.40.10 host 203.0.113.10
exit

ip access-list extended QOS-SCAVENGER
 remark SCAVENGER traffic simulation: ROGUE-PC to INTERNET-SRV
 permit icmp host 10.10.20.126 host 203.0.113.10
exit
```

---

## 5. EDGE1 - ACL-Based Class Maps

These class maps match the original internal source IP addresses before NAT/PAT.

```cisco
class-map match-any CM-VOICE
 match access-group name QOS-VOICE-LIKE
exit

class-map match-any CM-VIDEO
 match access-group name QOS-VIDEO-LIKE
exit

class-map match-any CM-BUSINESS-CRITICAL
 match access-group name QOS-BUSINESS-CRITICAL
exit

class-map match-any CM-MGMT
 match access-group name QOS-MGMT
exit

class-map match-any CM-SCAVENGER
 match access-group name QOS-SCAVENGER
exit
```

---

## 6. EDGE1 - DSCP-Based WAN Class Maps

These class maps match DSCP values after traffic has already been marked.

```cisco
class-map match-any CM-WAN-VOICE
 match dscp ef
exit

class-map match-any CM-WAN-VIDEO
 match dscp af41
exit

class-map match-any CM-WAN-BUSINESS-CRITICAL
 match dscp af31
exit

class-map match-any CM-WAN-MGMT
 match dscp cs2
exit

class-map match-any CM-WAN-SCAVENGER
 match dscp cs1
exit
```

---

## 7. EDGE1 - Ingress Marking Policy

This policy is applied inbound on the EDGE1 campus-facing interfaces.

Purpose:

```text
- Match original internal source IP before NAT/PAT.
- Mark traffic with the intended DSCP value.
```

```cisco
policy-map CAMPUS-QOS-MARK-IN
 class CM-VOICE
  set dscp ef
 exit

 class CM-VIDEO
  set dscp af41
 exit

 class CM-BUSINESS-CRITICAL
  set dscp af31
 exit

 class CM-MGMT
  set dscp cs2
 exit

 class CM-SCAVENGER
  set dscp cs1
 exit

 class class-default
 exit
exit
```

---

## 8. EDGE1 - WAN Egress Queuing Policy

This policy is applied outbound on the EDGE1 WAN-facing interfaces.

Purpose:

```text
- Match DSCP-marked traffic.
- Apply WAN egress queuing behavior.
```

```cisco
policy-map WAN-QOS-EGRESS
 class CM-WAN-VOICE
  priority percent 10
 exit

 class CM-WAN-VIDEO
  bandwidth percent 20
 exit

 class CM-WAN-BUSINESS-CRITICAL
  bandwidth percent 20
 exit

 class CM-WAN-MGMT
  bandwidth percent 10
 exit

 class CM-WAN-SCAVENGER
  bandwidth percent 5
 exit

 class class-default
  fair-queue
 exit
exit
```

---

## 9. EDGE1 - Interface Service-Policy Assignment

### 9.1 Gi0/0 - ISP1 Primary WAN

```cisco
interface GigabitEthernet0/0
 description TO-ISP1-PRIMARY-WAN
 ip address 100.64.1.2 255.255.255.252
 ip nat outside
 ip virtual-reassembly in
 duplex auto
 speed auto
 media-type rj45
 service-policy output WAN-QOS-EGRESS
exit
```

### 9.2 Gi0/1 - ISP2 Backup WAN

```cisco
interface GigabitEthernet0/1
 description TO-ISP2-BACKUP-WAN
 ip address 100.64.2.2 255.255.255.252
 ip nat outside
 ip virtual-reassembly in
 duplex auto
 speed auto
 media-type rj45
 service-policy output WAN-QOS-EGRESS
exit
```

### 9.3 Gi0/2 - CORE1 Campus Link

```cisco
interface GigabitEthernet0/2
 description TO-CORE1-CAMPUS
 ip address 172.16.0.1 255.255.255.252
 ip nat inside
 ip virtual-reassembly in
 duplex auto
 speed auto
 media-type rj45
 service-policy input CAMPUS-QOS-MARK-IN
exit
```

### 9.4 Gi0/3 - CORE2 Campus Link

```cisco
interface GigabitEthernet0/3
 description TO-CORE2-CAMPUS
 ip address 172.16.0.5 255.255.255.252
 ip nat inside
 ip virtual-reassembly in
 ip ospf cost 50
 duplex auto
 speed auto
 media-type rj45
 service-policy input CAMPUS-QOS-MARK-IN
exit
```

Important note:

```text
Do not remove "ip ospf cost 50" from Gi0/3.
It is part of the Phase 03 Multi-Area OSPF routing resiliency design.
```

---

## 10. EDGE1 - Complete QoS Configuration Block

This is the final QoS configuration block for EDGE1.

```cisco
!
! ============================================================
! Phase 05 QoS - Classification ACLs
! ============================================================
!

ip access-list extended QOS-VOICE-LIKE
 permit icmp host 10.10.10.117 host 203.0.113.10

ip access-list extended QOS-VIDEO-LIKE
 permit icmp host 10.10.20.197 host 203.0.113.10

ip access-list extended QOS-BUSINESS-CRITICAL
 permit icmp host 10.10.30.140 host 203.0.113.10

ip access-list extended QOS-MGMT
 remark MGMT traffic simulation: MGMT-SRV1 to INTERNET-SRV
 permit icmp host 10.10.40.10 host 203.0.113.10

ip access-list extended QOS-SCAVENGER
 remark SCAVENGER traffic simulation: ROGUE-PC to INTERNET-SRV
 permit icmp host 10.10.20.126 host 203.0.113.10

!
! ============================================================
! Phase 05 QoS - ACL-Based Class Maps
! ============================================================
!

class-map match-any CM-VOICE
 match access-group name QOS-VOICE-LIKE

class-map match-any CM-VIDEO
 match access-group name QOS-VIDEO-LIKE

class-map match-any CM-BUSINESS-CRITICAL
 match access-group name QOS-BUSINESS-CRITICAL

class-map match-any CM-MGMT
 match access-group name QOS-MGMT

class-map match-any CM-SCAVENGER
 match access-group name QOS-SCAVENGER

!
! ============================================================
! Phase 05 QoS - DSCP-Based WAN Class Maps
! ============================================================
!

class-map match-any CM-WAN-VOICE
 match dscp ef

class-map match-any CM-WAN-VIDEO
 match dscp af41

class-map match-any CM-WAN-BUSINESS-CRITICAL
 match dscp af31

class-map match-any CM-WAN-MGMT
 match dscp cs2

class-map match-any CM-WAN-SCAVENGER
 match dscp cs1

!
! ============================================================
! Phase 05 QoS - Ingress Marking Policy
! ============================================================
!

policy-map CAMPUS-QOS-MARK-IN
 class CM-VOICE
  set dscp ef
 class CM-VIDEO
  set dscp af41
 class CM-BUSINESS-CRITICAL
  set dscp af31
 class CM-MGMT
  set dscp cs2
 class CM-SCAVENGER
  set dscp cs1
 class class-default

!
! ============================================================
! Phase 05 QoS - WAN Egress Queuing Policy
! ============================================================
!

policy-map WAN-QOS-EGRESS
 class CM-WAN-VOICE
  priority percent 10
 class CM-WAN-VIDEO
  bandwidth percent 20
 class CM-WAN-BUSINESS-CRITICAL
  bandwidth percent 20
 class CM-WAN-MGMT
  bandwidth percent 10
 class CM-WAN-SCAVENGER
  bandwidth percent 5
 class class-default
  fair-queue

!
! ============================================================
! Phase 05 QoS - Interface Service-Policy Placement
! ============================================================
!

interface GigabitEthernet0/0
 description TO-ISP1-PRIMARY-WAN
 service-policy output WAN-QOS-EGRESS

interface GigabitEthernet0/1
 description TO-ISP2-BACKUP-WAN
 service-policy output WAN-QOS-EGRESS

interface GigabitEthernet0/2
 description TO-CORE1-CAMPUS
 service-policy input CAMPUS-QOS-MARK-IN

interface GigabitEthernet0/3
 description TO-CORE2-CAMPUS
 service-policy input CAMPUS-QOS-MARK-IN
```

---

## 11. Removed Initial Test Policy

The following initial test policy was removed after troubleshooting:

```text
WAN-QOS-POLICY
```

Reason:

```text
The initial design attempted to classify traffic on the WAN output interface using original 10.10.x.x inside source addresses.

Because EDGE1 performs NAT/PAT, the WAN output policy did not reliably match the original internal source IP addresses.

Result:
- Custom QoS class counters stayed at 0.
- Only class-default increased.
- ACL hit counts did not increase.
```

Cleanup command used:

```cisco
configure terminal
no policy-map WAN-QOS-POLICY
end
write memory
```

Final remaining policy maps:

```text
CAMPUS-QOS-MARK-IN
WAN-QOS-EGRESS
```

---

## 12. Final Policy Placement

```text
EDGE1 Gi0/2 input:
- CAMPUS-QOS-MARK-IN
- Campus traffic from CORE1
- Original source IP classification
- DSCP marking before NAT

EDGE1 Gi0/3 input:
- CAMPUS-QOS-MARK-IN
- Campus traffic from CORE2
- Original source IP classification
- DSCP marking before NAT

EDGE1 Gi0/0 output:
- WAN-QOS-EGRESS
- ISP1 primary WAN
- DSCP-based queuing

EDGE1 Gi0/1 output:
- WAN-QOS-EGRESS
- ISP2 backup WAN
- DSCP-based queuing
```

---

## 13. Verification Commands

The following commands were used to verify the configuration.

```cisco
show class-map
show policy-map
show policy-map interface GigabitEthernet0/0
show policy-map interface GigabitEthernet0/1
show policy-map interface GigabitEthernet0/2
show policy-map interface GigabitEthernet0/3
show access-lists
show running-config | section ip access-list extended QOS
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

## 14. Final Save Command

```cisco
write memory
```

---

## 15. Final Configuration Status

```text
Phase 05 QoS configuration: COMPLETE
```

Verified QoS classes:

```text
VOICE QoS: PASS
VIDEO QoS: PASS
BUSINESS CRITICAL QoS: PASS
MANAGEMENT QoS: PASS
SCAVENGER QoS: PASS
```

Final architecture:

```text
Ingress classification and marking before NAT
+
WAN egress queuing after NAT based on DSCP
```
