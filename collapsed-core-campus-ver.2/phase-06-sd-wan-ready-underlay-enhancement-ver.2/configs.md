# Phase 06 — SD-WAN-Ready Underlay Enhancement Configs

## 1. Purpose

This document records the final configuration used in:

```text
Phase 06 — SD-WAN-Ready Underlay Enhancement
```

This phase enhanced the existing dual-ISP WAN underlay with:

* Route-map based dual ISP NAT/PAT
* IP SLA and Object Tracking for ISP1 and ISP2
* Track-aware default route failover
* Track-aware Policy-Based Routing
* Application-aware path selection simulation
* QoS and WAN path integration

Phase 06 continues from the final state of:

```text
Phase 05 — QoS
```

---

## 2. Configuration Scope

Most Phase 06 configuration changes were applied on:

```text
EDGE1
```

The following devices were used for traffic generation and validation, but did not require new Phase 06 configuration changes:

```text
EXEC
SALES
IT-OPS
MGMT-SRV1
ROGUE-PC
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

## 3. EDGE1 Interface Role Summary

| Interface | IP Address | Role                |
| --------- | ---------: | ------------------- |
| Gi0/0     | 100.64.1.2 | ISP1-facing outside |
| Gi0/1     | 100.64.2.2 | ISP2-facing outside |
| Gi0/2     | 172.16.0.1 | CORE1-facing inside |
| Gi0/3     | 172.16.0.5 | CORE2-facing inside |

---

## 4. EDGE1 Final Relevant Configuration

> This section records the final relevant EDGE1 configuration for Phase 06.
> Existing Phase 01–05 baseline configuration is not repeated unless it directly affects Phase 06 behavior.

```cisco
hostname EDGE1

!
! =========================================================
! INTERFACE ROLES
! =========================================================
!

interface GigabitEthernet0/0
 description TO-ISP1
 ip address 100.64.1.2 255.255.255.252
 ip nat outside
 service-policy output WAN-QOS-EGRESS
 no shutdown

interface GigabitEthernet0/1
 description TO-ISP2
 ip address 100.64.2.2 255.255.255.252
 ip nat outside
 service-policy output WAN-QOS-EGRESS
 no shutdown

interface GigabitEthernet0/2
 description TO-CORE1
 ip address 172.16.0.1 255.255.255.252
 ip nat inside
 ip policy route-map WAN-BUSINESS-INTENT
 service-policy input CAMPUS-QOS-MARK-IN
 no shutdown

interface GigabitEthernet0/3
 description TO-CORE2
 ip address 172.16.0.5 255.255.255.252
 ip nat inside
 ip policy route-map WAN-BUSINESS-INTENT
 service-policy input CAMPUS-QOS-MARK-IN
 no shutdown
```

---

## 5. EDGE1 Static Routes

```cisco
!
! =========================================================
! DEFAULT ROUTES WITH TRACKING
! =========================================================
!

ip route 0.0.0.0 0.0.0.0 100.64.1.1 track 1
ip route 0.0.0.0 0.0.0.0 100.64.2.1 10 track 2

!
! Force IP SLA 1 probe toward ISP1 path
!
ip route 203.0.113.10 255.255.255.255 100.64.1.1
```

### Route Meaning

| Route                                    | Purpose                              |
| ---------------------------------------- | ------------------------------------ |
| `0.0.0.0/0 via 100.64.1.1 track 1`       | Primary default route through ISP1   |
| `0.0.0.0/0 via 100.64.2.1 AD 10 track 2` | Backup default route through ISP2    |
| `203.0.113.10/32 via 100.64.1.1`         | Keeps IP SLA 1 tied to the ISP1 path |

---

## 6. EDGE1 IP SLA and Object Tracking

```cisco
!
! =========================================================
! IP SLA FOR ISP1
! =========================================================
!

ip sla 1
 icmp-echo 203.0.113.10 source-interface GigabitEthernet0/0
 threshold 1000
 frequency 5

ip sla schedule 1 life forever start-time now

track 1 ip sla 1 reachability
 delay down 10 up 5

!
! =========================================================
! IP SLA FOR ISP2
! =========================================================
!

ip sla 2
 icmp-echo 100.64.2.1 source-interface GigabitEthernet0/1
 threshold 1000
 timeout 1000
 frequency 5

ip sla schedule 2 life forever start-time now

track 2 ip sla 2 reachability
 delay down 10 up 5
```

### Tracking Design

| Track   | Monitored Path    | Purpose                                              |
| ------- | ----------------- | ---------------------------------------------------- |
| Track 1 | ISP1 reachability | Controls primary default route and ISP1 PBR next-hop |
| Track 2 | ISP2 reachability | Controls backup default route and ISP2 PBR next-hop  |

---

## 7. EDGE1 NAT/PAT Configuration

### 7.1 Enterprise NAT ACL

```cisco
!
! =========================================================
! ENTERPRISE NAT ACL
! =========================================================
!

ip access-list standard ENTERPRISE-NAT
 permit 10.10.0.0 0.0.255.255
```

### 7.2 Route-map Based Dual ISP NAT/PAT

```cisco
!
! =========================================================
! DUAL ISP NAT/PAT
! =========================================================
!

ip nat inside source route-map NAT-ISP1 interface GigabitEthernet0/0 overload
ip nat inside source route-map NAT-ISP2 interface GigabitEthernet0/1 overload

route-map NAT-ISP1 permit 10
 match ip address ENTERPRISE-NAT
 match interface GigabitEthernet0/0

route-map NAT-ISP2 permit 10
 match ip address ENTERPRISE-NAT
 match interface GigabitEthernet0/1
```

### NAT/PAT Behavior

| Egress Path  | NAT/PAT Address |
| ------------ | --------------: |
| Gi0/0 / ISP1 |      100.64.1.2 |
| Gi0/1 / ISP2 |      100.64.2.2 |

---

## 8. EDGE1 PBR ACLs

The following ACLs are used by the `WAN-BUSINESS-INTENT` route-map.

```cisco
!
! =========================================================
! PBR ACLS
! =========================================================
!

ip access-list extended PBR-VOICE-TO-ISP1
 permit ip host 10.10.10.117 host 203.0.113.10

ip access-list extended PBR-SCAVENGER-TO-ISP2
 permit ip host 10.10.20.126 host 203.0.113.10

ip access-list extended PBR-MGMT-TO-ISP1
 permit ip host 10.10.40.10 host 203.0.113.10

ip access-list extended PBR-VIDEO-TO-ISP2
 permit ip host 10.10.20.197 host 203.0.113.10

ip access-list extended PBR-BUSINESS-TO-ISP1
 permit ip host 10.10.30.140 host 203.0.113.10
```

---

## 9. EDGE1 Track-Aware PBR Route-map

```cisco
!
! =========================================================
! TRACK-AWARE BUSINESS INTENT PBR
! =========================================================
!

route-map WAN-BUSINESS-INTENT permit 5
 description VOICE_TRAFFIC_PREFER_ISP1
 match ip address PBR-VOICE-TO-ISP1
 set ip next-hop verify-availability 100.64.1.1 10 track 1

route-map WAN-BUSINESS-INTENT permit 10
 description SCAVENGER_TRAFFIC_PREFER_ISP2
 match ip address PBR-SCAVENGER-TO-ISP2
 set ip next-hop verify-availability 100.64.2.1 10 track 2

route-map WAN-BUSINESS-INTENT permit 20
 description MGMT_TRAFFIC_PREFER_ISP1
 match ip address PBR-MGMT-TO-ISP1
 set ip next-hop verify-availability 100.64.1.1 10 track 1

route-map WAN-BUSINESS-INTENT permit 30
 description VIDEO_TRAFFIC_PREFER_ISP2
 match ip address PBR-VIDEO-TO-ISP2
 set ip next-hop verify-availability 100.64.2.1 10 track 2

route-map WAN-BUSINESS-INTENT permit 40
 description BUSINESS_CRITICAL_TRAFFIC_PREFER_ISP1
 match ip address PBR-BUSINESS-TO-ISP1
 set ip next-hop verify-availability 100.64.1.1 10 track 1

route-map WAN-BUSINESS-INTENT permit 100
 description DEFAULT_TRAFFIC_USE_NORMAL_ROUTING
```

### PBR Policy Summary

| Sequence | Traffic           | Preferred ISP  | Track   |
| -------: | ----------------- | -------------- | ------- |
|        5 | VOICE-like        | ISP1           | Track 1 |
|       10 | SCAVENGER         | ISP2           | Track 2 |
|       20 | MGMT              | ISP1           | Track 1 |
|       30 | VIDEO-like        | ISP2           | Track 2 |
|       40 | BUSINESS-CRITICAL | ISP1           | Track 1 |
|      100 | Other traffic     | Normal routing | N/A     |

---

## 10. EDGE1 QoS ACLs

These ACLs were originally used in Phase 05 and remained active in Phase 06.

```cisco
!
! =========================================================
! QOS CLASSIFICATION ACLS
! =========================================================
!

ip access-list extended QOS-VOICE-LIKE
 permit icmp host 10.10.10.117 host 203.0.113.10

ip access-list extended QOS-VIDEO-LIKE
 permit icmp host 10.10.20.197 host 203.0.113.10

ip access-list extended QOS-BUSINESS-CRITICAL
 permit icmp host 10.10.30.140 host 203.0.113.10

ip access-list extended QOS-MGMT
 permit icmp host 10.10.40.10 host 203.0.113.10

ip access-list extended QOS-SCAVENGER
 permit icmp host 10.10.20.126 host 203.0.113.10
```

---

## 11. EDGE1 QoS Class Maps

```cisco
!
! =========================================================
! CAMPUS INGRESS CLASS MAPS
! =========================================================
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
! =========================================================
! WAN EGRESS CLASS MAPS
! =========================================================
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
```

---

## 12. EDGE1 QoS Policy Maps

### 12.1 Campus Ingress Marking Policy

```cisco
!
! =========================================================
! CAMPUS INGRESS QOS MARKING
! =========================================================
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
```

### 12.2 WAN Egress Queuing Policy

```cisco
!
! =========================================================
! WAN EGRESS QOS POLICY
! =========================================================
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
```

---

## 13. Final Host-to-Policy Mapping

| Host      |   IP Address | QoS Class         | DSCP | Preferred ISP | PBR Sequence |
| --------- | -----------: | ----------------- | ---- | ------------- | -----------: |
| EXEC      | 10.10.10.117 | VOICE-like        | EF   | ISP1          |            5 |
| SALES     | 10.10.20.197 | VIDEO-like        | AF41 | ISP2          |           30 |
| IT-OPS    | 10.10.30.140 | BUSINESS-CRITICAL | AF31 | ISP1          |           40 |
| MGMT-SRV1 |  10.10.40.10 | MGMT              | CS2  | ISP1          |           20 |
| ROGUE-PC  | 10.10.20.126 | SCAVENGER         | CS1  | ISP2          |           10 |

---

## 14. Final Traffic Behavior

### 14.1 Normal State

| Traffic                    | Expected Path | NAT/PAT Address | WAN QoS Class            |
| -------------------------- | ------------- | --------------: | ------------------------ |
| EXEC / VOICE-like          | ISP1 / Gi0/0  |      100.64.1.2 | CM-WAN-VOICE             |
| SALES / VIDEO-like         | ISP2 / Gi0/1  |      100.64.2.2 | CM-WAN-VIDEO             |
| IT-OPS / BUSINESS-CRITICAL | ISP1 / Gi0/0  |      100.64.1.2 | CM-WAN-BUSINESS-CRITICAL |
| MGMT-SRV1 / MGMT           | ISP1 / Gi0/0  |      100.64.1.2 | CM-WAN-MGMT              |
| ROGUE-PC / SCAVENGER       | ISP2 / Gi0/1  |      100.64.2.2 | CM-WAN-SCAVENGER         |

### 14.2 Failure State

| Failure                   | Expected Behavior                         |
| ------------------------- | ----------------------------------------- |
| ISP1 Down                 | Track 1 Down, default route moves to ISP2 |
| ISP2 Down                 | Track 2 Down, default route remains ISP1  |
| MGMT preferred ISP1 Down  | MGMT falls back to ISP2                   |
| ROGUE preferred ISP2 Down | ROGUE falls back to ISP1                  |
| VIDEO preferred ISP2 Down | VIDEO falls back to ISP1                  |

---

## 15. Verification Commands

```cisco
!
! =========================================================
! INTERFACE AND ROUTING
! =========================================================
!

show ip interface brief
show ip route 0.0.0.0
show ip sla statistics
show track

!
! =========================================================
! NAT/PAT
! =========================================================
!

show ip nat translations
show ip nat translations | include 10.10.10.117
show ip nat translations | include 10.10.20.197
show ip nat translations | include 10.10.30.140
show ip nat translations | include 10.10.40.10
show ip nat translations | include 10.10.20.126
show ip nat statistics
show running-config | include ip nat
show running-config | section route-map NAT

!
! =========================================================
! PBR
! =========================================================
!

show ip policy
show route-map WAN-BUSINESS-INTENT
show running-config | section route-map WAN-BUSINESS-INTENT

!
! =========================================================
! QOS
! =========================================================
!

show policy-map interface GigabitEthernet0/0
show policy-map interface GigabitEthernet0/1
show policy-map interface GigabitEthernet0/2
show policy-map interface GigabitEthernet0/3
```

---

## 16. Migration Notes

### 16.1 Removed Previous Single-Interface PAT

The previous single-interface PAT design was replaced.

Old design:

```cisco
ip nat inside source list ENTERPRISE-NAT interface GigabitEthernet0/0 overload
```

New design:

```cisco
ip nat inside source route-map NAT-ISP1 interface GigabitEthernet0/0 overload
ip nat inside source route-map NAT-ISP2 interface GigabitEthernet0/1 overload
```

### 16.2 Replaced Untracked Backup Default Route

Old design:

```cisco
ip route 0.0.0.0 0.0.0.0 100.64.2.1 10
```

New design:

```cisco
ip route 0.0.0.0 0.0.0.0 100.64.2.1 10 track 2
```

---

## 17. Save Configuration

After final validation, save the configuration.

```cisco
write memory
```

or

```cisco
copy running-config startup-config
```

---

## 18. Final Config Status

```text
Phase 06 — SD-WAN-Ready Underlay Enhancement
Config Status: Completed
Result: PASS
```
