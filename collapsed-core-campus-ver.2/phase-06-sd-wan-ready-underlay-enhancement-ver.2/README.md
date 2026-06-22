# Phase 06 — SD-WAN-Ready Underlay Enhancement

## 1. Overview

This phase enhances the traditional dual-ISP WAN underlay so that it becomes ready for SD-WAN overlay learning in the next phase.

Phase 06 does **not** build the SD-WAN overlay itself.
There is no vManage, vSmart, vBond, cEdge, OMP, TLOC, VPN template, or centralized policy in this phase.

Instead, this phase focuses on strengthening the existing WAN underlay with:

* Dual ISP NAT/PAT failover
* IP SLA and object tracking for both ISP paths
* Track-aware backup default route
* Track-aware Policy-Based Routing
* Application-aware path selection simulation
* QoS and WAN egress integration
* Failure and recovery validation

The goal is to understand how a traditional WAN underlay can be made more resilient and policy-driven before moving into a real SD-WAN overlay architecture.

---

## 2. Lab Position in the Overall Series

This phase continues from the final state of:

```text
Phase 05 — QoS
```

The Phase 05 baseline already included:

* Multi-Area OSPF from Phase 03
* Security controls from Phase 04
* QoS marking and WAN queuing from Phase 05
* EDGE1 dual-ISP topology
* Campus VLAN segmentation
* Centralized services through MGMT-SRV1
* Internet reachability through INTERNET-SRV

Phase 06 extends this by improving WAN resiliency and adding policy-based traffic steering.

---

## 3. Phase 06 Title

```text
Phase 06 — SD-WAN-Ready Underlay Enhancement
```

This title is intentional.

The phase prepares the network for SD-WAN learning by enhancing the underlay, but it does not yet deploy the SD-WAN overlay.

---

## 4. Topology Summary

The topology remains based on the existing collapsed-core campus design.

### WAN / Edge

| Device       | Interface |   IP Address | Role                 |
| ------------ | --------: | -----------: | -------------------- |
| EDGE1        |     Gi0/0 |   100.64.1.2 | ISP1-facing outside  |
| EDGE1        |     Gi0/1 |   100.64.2.2 | ISP2-facing outside  |
| EDGE1        |     Gi0/2 |   172.16.0.1 | CORE1-facing inside  |
| EDGE1        |     Gi0/3 |   172.16.0.5 | CORE2-facing inside  |
| ISP1         |     Gi0/1 |   100.64.1.1 | ISP1 next-hop        |
| ISP2         |     Gi0/1 |   100.64.2.1 | ISP2 next-hop        |
| INTERNET-SRV |        E0 | 203.0.113.10 | Internet test server |

### Key Campus Hosts

| Host      |   IP Address | Traffic Role      |
| --------- | -----------: | ----------------- |
| EXEC      | 10.10.10.117 | VOICE-like        |
| SALES     | 10.10.20.197 | VIDEO-like        |
| IT-OPS    | 10.10.30.140 | BUSINESS-CRITICAL |
| MGMT-SRV1 |  10.10.40.10 | MGMT              |
| ROGUE-PC  | 10.10.20.126 | SCAVENGER         |

---

## 5. Phase Objectives

The objectives of this phase are:

1. Identify the limitation of single-interface NAT/PAT in a dual-ISP design.
2. Replace single-interface PAT with route-map based dual-ISP NAT/PAT.
3. Validate NAT/PAT failover from ISP1 to ISP2.
4. Validate NAT/PAT failback from ISP2 to ISP1.
5. Add IP SLA and tracking for both ISP1 and ISP2.
6. Tie the backup default route to Track 2.
7. Add track-aware PBR for traffic steering.
8. Simulate SD-WAN-like application-aware path selection.
9. Verify that PBR, NAT/PAT, and QoS work together.
10. Confirm that failed WAN paths are not used by policy routing.

---

## 6. What This Phase Is Not

This phase is **not** SD-WAN overlay deployment.

The following are intentionally out of scope for Phase 06:

* vManage deployment
* vSmart deployment
* vBond deployment
* Catalyst SD-WAN Manager configuration
* Catalyst SD-WAN Controller configuration
* Catalyst SD-WAN Validator configuration
* cEdge onboarding
* OMP
* TLOC extension
* SD-WAN VPN segmentation
* Centralized policy
* Localized policy
* App-Aware Routing using SD-WAN SLA classes
* BFD-based SD-WAN tunnel health
* Control connections

Those topics belong to Phase 07.

---

## 7. Section Summary

### SECTION 1 — Baseline Review and Problem Identification

The initial WAN resiliency design allowed routing failover from ISP1 to ISP2.

However, NAT/PAT was still tied only to ISP1:

```cisco
ip nat inside source list ENTERPRISE-NAT interface GigabitEthernet0/0 overload
```

This caused a problem:

```text
Routing failover = PASS
NAT/PAT failover = FAIL
```

When ISP1 failed, the default route moved to ISP2, but NAT/PAT still depended on Gi0/0.

This proved that default route failover alone is not enough in a dual-ISP Internet edge design.

---

### SECTION 2 — NAT/PAT Failover Enhancement

The single-interface PAT design was replaced with route-map based dual-ISP NAT/PAT.

Final NAT/PAT structure:

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

Result:

```text
ISP1 path used → PAT to 100.64.1.2
ISP2 path used → PAT to 100.64.2.2
```

Validation:

| Scenario      | Result                         |
| ------------- | ------------------------------ |
| ISP1 normal   | NAT/PAT through 100.64.1.2     |
| ISP1 failure  | NAT/PAT through 100.64.2.2     |
| ISP1 recovery | NAT/PAT returned to 100.64.1.2 |

SECTION 2 result:

```text
PASS
```

---

### SECTION 3 — IP SLA / Object Tracking Enhancement

Before this section, only ISP1 was tracked.

Phase 06 enhanced the design by adding ISP2 tracking.

Final tracking design:

```text
Track 1 → ISP1 reachability
Track 2 → ISP2 reachability
```

IP SLA 1 monitors ISP1 primary path:

```cisco
ip sla 1
 icmp-echo 203.0.113.10 source-interface GigabitEthernet0/0
 threshold 1000
 frequency 5
ip sla schedule 1 life forever start-time now
```

IP SLA 2 monitors ISP2 next-hop:

```cisco
ip sla 2
 icmp-echo 100.64.2.1 source-interface GigabitEthernet0/1
 threshold 1000
 timeout 1000
 frequency 5
ip sla schedule 2 life forever start-time now
```

Track configuration:

```cisco
track 1 ip sla 1 reachability
 delay down 10 up 5

track 2 ip sla 2 reachability
 delay down 10 up 5
```

Final default route structure:

```cisco
ip route 0.0.0.0 0.0.0.0 100.64.1.1 track 1
ip route 0.0.0.0 0.0.0.0 100.64.2.1 10 track 2
```

Validation:

| Scenario           | Expected Result                           | Actual Result |
| ------------------ | ----------------------------------------- | ------------- |
| ISP1 Up, ISP2 Up   | Default route via ISP1                    | PASS          |
| ISP2 Down          | Track 2 Down, default route remains ISP1  | PASS          |
| ISP1 Down, ISP2 Up | Track 1 Down, default route moves to ISP2 | PASS          |
| ISP1 restored      | Default route returns to ISP1             | PASS          |

SECTION 3 result:

```text
PASS
```

---

### SECTION 4 — Policy-Based Routing / SLA-Based WAN Policy

This section added track-aware PBR.

The goal was to steer selected traffic to preferred WAN paths while avoiding failed next-hops.

Initial policy:

| Traffic              | Preferred ISP | Track   |
| -------------------- | ------------- | ------- |
| MGMT-SRV1            | ISP1          | Track 1 |
| ROGUE-PC / Scavenger | ISP2          | Track 2 |

PBR was applied inbound on both campus-facing EDGE1 interfaces:

```cisco
interface GigabitEthernet0/2
 ip policy route-map WAN-BUSINESS-INTENT

interface GigabitEthernet0/3
 ip policy route-map WAN-BUSINESS-INTENT
```

Key PBR logic:

```cisco
route-map WAN-BUSINESS-INTENT permit 10
 description SCAVENGER_TRAFFIC_PREFER_ISP2
 match ip address PBR-SCAVENGER-TO-ISP2
 set ip next-hop verify-availability 100.64.2.1 10 track 2

route-map WAN-BUSINESS-INTENT permit 20
 description MGMT_TRAFFIC_PREFER_ISP1
 match ip address PBR-MGMT-TO-ISP1
 set ip next-hop verify-availability 100.64.1.1 10 track 1

route-map WAN-BUSINESS-INTENT permit 100
 description DEFAULT_TRAFFIC_USE_NORMAL_ROUTING
```

The `verify-availability` keyword was important because it prevents PBR from forcing traffic toward a failed ISP next-hop.

Validation:

| Test                                        | Result |
| ------------------------------------------- | ------ |
| MGMT-SRV1 prefers ISP1                      | PASS   |
| ROGUE-PC prefers ISP2                       | PASS   |
| ISP2 failure → ROGUE-PC falls back to ISP1  | PASS   |
| ISP1 failure → MGMT-SRV1 falls back to ISP2 | PASS   |

SECTION 4 result:

```text
PASS
```

---

### SECTION 5 — Application-Aware Path Selection Simulation

This section expanded the PBR policy to simulate SD-WAN-like application-aware path selection.

Final business intent policy:

| Traffic Type      | Host                    | Preferred Path |
| ----------------- | ----------------------- | -------------- |
| VOICE-like        | EXEC / 10.10.10.117     | ISP1           |
| VIDEO-like        | SALES / 10.10.20.197    | ISP2           |
| BUSINESS-CRITICAL | IT-OPS / 10.10.30.140   | ISP1           |
| MGMT              | MGMT-SRV1 / 10.10.40.10 | ISP1           |
| SCAVENGER         | ROGUE-PC / 10.10.20.126 | ISP2           |
| Default           | Other traffic           | Normal routing |

Final route-map structure:

```text
permit 5    VOICE-like        → ISP1 / Track 1
permit 10   SCAVENGER         → ISP2 / Track 2
permit 20   MGMT              → ISP1 / Track 1
permit 30   VIDEO-like        → ISP2 / Track 2
permit 40   BUSINESS-CRITICAL → ISP1 / Track 1
permit 100  DEFAULT           → Normal routing
```

Validation:

| Traffic                               | Expected Path     | Result |
| ------------------------------------- | ----------------- | ------ |
| EXEC / VOICE-like                     | ISP1 / 100.64.1.2 | PASS   |
| SALES / VIDEO-like                    | ISP2 / 100.64.2.2 | PASS   |
| IT-OPS / BUSINESS-CRITICAL            | ISP1 / 100.64.1.2 | PASS   |
| MGMT-SRV1 / MGMT                      | ISP1 / 100.64.1.2 | PASS   |
| ROGUE-PC / SCAVENGER                  | ISP2 / 100.64.2.2 | PASS   |
| ISP2 failure → SALES fallback to ISP1 | ISP1 / 100.64.1.2 | PASS   |

SECTION 5 result:

```text
PASS
```

---

### SECTION 6 — QoS + WAN Integration Final Verification

This section verified that QoS still worked after adding NAT/PAT failover and PBR.

The final traffic flow is:

```text
Campus host
→ EDGE1 ingress classification
→ DSCP marking
→ PBR path selection
→ NAT/PAT based on egress interface
→ WAN egress QoS queuing
→ ISP path
```

QoS policies remained in place:

```text
Gi0/2 input  → CAMPUS-QOS-MARK-IN
Gi0/3 input  → CAMPUS-QOS-MARK-IN
Gi0/0 output → WAN-QOS-EGRESS
Gi0/1 output → WAN-QOS-EGRESS
```

Validated DSCP marking:

| Traffic           | DSCP Marking |
| ----------------- | ------------ |
| VOICE-like        | EF           |
| VIDEO-like        | AF41         |
| BUSINESS-CRITICAL | AF31         |
| MGMT              | CS2          |
| SCAVENGER         | CS1          |

Validated WAN egress classes:

| Traffic                    | WAN Interface                             | Egress Class             |
| -------------------------- | ----------------------------------------- | ------------------------ |
| EXEC / VOICE-like          | Gi0/0                                     | CM-WAN-VOICE             |
| SALES / VIDEO-like         | Gi0/1 normally, Gi0/0 during ISP2 failure | CM-WAN-VIDEO             |
| IT-OPS / BUSINESS-CRITICAL | Gi0/0                                     | CM-WAN-BUSINESS-CRITICAL |
| MGMT-SRV1 / MGMT           | Gi0/0 normally, Gi0/1 during ISP1 failure | CM-WAN-MGMT              |
| ROGUE-PC / SCAVENGER       | Gi0/1 normally, Gi0/0 during ISP2 failure | CM-WAN-SCAVENGER         |

SECTION 6 result:

```text
PASS
```

---

## 8. Final Phase 06 Result

Phase 06 successfully enhanced the WAN underlay.

Final result:

```text
Routing failover       PASS
NAT/PAT failover       PASS
IP SLA / Track         PASS
PBR path steering      PASS
Application-aware path PASS
QoS + WAN integration  PASS
```

The final EDGE1 design supports:

* ISP1 primary default route
* ISP2 backup default route
* Track-based route installation
* NAT/PAT based on actual WAN egress path
* Track-aware PBR
* Application-aware path preference simulation
* QoS marking and egress queuing preservation
* Failure-based fallback to available WAN path

---

## 9. Key Lessons Learned

### 9.1 Routing failover alone is not enough

A backup default route can move traffic to ISP2, but NAT/PAT must also follow the active egress interface.

Without dual-ISP NAT/PAT logic, Internet failover remains incomplete.

---

### 9.2 NAT/PAT must match the actual egress path

Route-map based NAT/PAT allows the router to translate traffic using the correct WAN interface address.

```text
Gi0/0 egress → 100.64.1.2
Gi0/1 egress → 100.64.2.2
```

---

### 9.3 Backup paths should also be tracked

A floating static route without tracking may remain configured even when the backup ISP is unusable.

Adding Track 2 ensures that the ISP2 backup route is only valid when ISP2 is actually reachable.

---

### 9.4 PBR must be track-aware

Traditional PBR can be dangerous if it forces traffic to a failed next-hop.

Using `set ip next-hop verify-availability` prevents traffic from being sent toward a failed ISP.

---

### 9.5 This phase simulates SD-WAN business intent

The PBR policy in this phase is not SD-WAN, but it introduces similar thinking:

```text
Traffic class → Preferred WAN path → Health check → Fallback
```

This prepares the lab for Phase 07 SD-WAN overlay concepts.

---

## 10. Relationship to Phase 07

Phase 06 prepares the underlay for Phase 07.

In Phase 07, these traditional mechanisms will be compared with SD-WAN overlay concepts:

| Phase 06 Traditional WAN | Phase 07 SD-WAN Overlay        |
| ------------------------ | ------------------------------ |
| IP SLA / Track           | BFD / TLOC health              |
| PBR                      | Centralized Data Policy        |
| Static default routes    | OMP route advertisements       |
| Preferred ISP            | Preferred color / TLOC         |
| NAT/PAT failover         | DIA / service-side NAT policy  |
| QoS MQC                  | SD-WAN QoS / App-Aware Routing |
| Manual policy            | Controller-based policy        |

---

## 11. Verification Commands

Useful verification commands:

```cisco
show ip interface brief
show ip route 0.0.0.0
show ip sla statistics
show track

show ip nat translations
show ip nat statistics
show running-config | include ip nat
show running-config | section route-map NAT

show ip policy
show route-map WAN-BUSINESS-INTENT
show running-config | section route-map WAN-BUSINESS-INTENT

show policy-map interface GigabitEthernet0/0
show policy-map interface GigabitEthernet0/1
show policy-map interface GigabitEthernet0/2
show policy-map interface GigabitEthernet0/3
```

---

## 12. Final Status

```text
Phase 06 — SD-WAN-Ready Underlay Enhancement
Status: Completed
Result: PASS
```

Phase 06 is now ready to be documented in:

```text
configs.md
design-decisions.md
troubleshooting.md
verification.md
```

The next major lab phase is:

```text
Phase 07 — SD-WAN Overlay
```
