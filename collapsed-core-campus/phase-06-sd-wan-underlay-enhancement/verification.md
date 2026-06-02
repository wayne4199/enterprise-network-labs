# PHASE 06 — Verification Guide

## Overview

This document validates all SD-WAN Underlay Enhancement objectives implemented in Phase 06.

Verification focuses on:

* Policy Based Routing (PBR)
* SLA-Based WAN Selection
* Application-Aware Path Selection
* QoS and WAN Integration
* Business Intent Traffic Steering
* Traffic Segmentation Policy
* Direct Internet Access (DIA)
* Local Internet Breakout (LIB)
* WAN Edge Design Concepts
* IP SLA Enhancement

---

# 1. OSPF Reachability Verification

## Command

```cisco
show ip route
```

## Expected Result

Routes learned from CORE1 and CORE2 should appear.

Example:

```text
O 10.10.10.0/24
O 10.20.20.0/24
O 10.30.30.0/24
O 10.40.40.0/24
O 10.50.50.0/24
```

## PASS Criteria

```text
All VLAN networks learned through OSPF.
```

---

# 2. HSRP Verification

## Command

```cisco
show standby brief
```

## Expected Result

CORE1

```text
VLAN10 Active
VLAN30 Active
VLAN40 Active
VLAN50 Active
```

CORE2

```text
VLAN20 Active
```

## PASS Criteria

```text
Correct active/standby gateway ownership.
```

---

# 3. INTERNET-SRV1 Connectivity Verification

## Command

```bash
ping 203.0.113.1
ping 203.0.113.2
```

## Expected Result

```text
Success rate 100%
```

## PASS Criteria

```text
INTERNET-SRV1 reachable from both ISP routers.
```

---

# 4. Guest VLAN Reachability

## Command

```bash
ifconfig
route -n
ping 10.50.50.254
```

## Expected Result

```text
IP Address : 10.50.50.10/24
Gateway    : 10.50.50.254
Ping Success
```

## PASS Criteria

```text
Guest host reaches HSRP gateway.
```

---

# 5. Guest Segmentation Policy Verification

## Test 1 — User VLAN Access

### Command

```bash
ping 10.10.10.10
```

### Expected Result

```text
100% packet loss
```

---

## Test 2 — Server VLAN Access

### Command

```bash
ping 10.20.20.31
```

### Expected Result

```text
100% packet loss
```

---

## Test 3 — Management VLAN Access

### Command

```bash
ping 10.40.40.10
```

### Expected Result

```text
100% packet loss
```

---

## PASS Criteria

```text
Guest VLAN cannot access internal enterprise resources.
```

---

# 6. Direct Internet Access Verification

## Command

```bash
ping 203.0.113.10
```

## Expected Result

```text
Replies received successfully.
```

## PASS Criteria

```text
Guest VLAN can access Internet resources.
```

---

# 7. NAT Verification

## Command

```cisco
show ip nat translations
```

## Expected Result

```text
Inside Local  10.50.50.10
Inside Global 100.64.2.2
```

or

```text
Inside Global 100.64.1.2
```

depending on WAN selection.

---

## Command

```cisco
show ip nat statistics
```

## Expected Result

```text
Dynamic mappings present
Hits increasing
```

## PASS Criteria

```text
Guest traffic translated successfully.
```

---

# 8. PBR Verification

## Command

```cisco
show route-map
```

## Expected Result

```text
Policy routing matches increasing
```

Example:

```text
Policy routing matches: 17128 packets
```

## PASS Criteria

```text
Traffic matches route-map entries.
```

---

# 9. Guest Traffic Steering Verification

## Command

```cisco
show route-map
```

## Expected Result

```text
route-map GUEST-PBR
ip next-hop verify-availability 100.64.2.1 track 2
ip next-hop verify-availability 100.64.1.1 track 1
```

## PASS Criteria

```text
Guest traffic prefers ISP2.
ISP1 acts as backup.
```

---

# 10. Application-Aware Routing Verification

## Video Application

### Command

```cisco
show route-map VIDEO-PBR
```

### Expected Result

```text
ip next-hop verify-availability 100.64.1.1
```

---

## Voice Application

### Command

```cisco
show route-map VOICE-PBR
```

### Expected Result

```text
ip next-hop verify-availability 100.64.1.1
```

---

## PASS Criteria

```text
Voice and Video traffic prefer ISP1.
```

---

# 11. IP SLA Verification

## Command

```cisco
show ip sla statistics
```

## Expected Result

```text
Operation ID: 1
Latest operation return code: OK

Operation ID: 2
Latest operation return code: OK
```

## PASS Criteria

```text
Both SLA probes operational.
```

---

# 12. Tracking Object Verification

## Command

```cisco
show track
```

## Expected Result

```text
Track 1 Reachability Up
Track 2 Reachability Up
```

## PASS Criteria

```text
Both WAN paths monitored successfully.
```

---

# 13. ISP2 Failure Verification

## Failure Injection

```cisco
ISP2(config)# interface g0/0
ISP2(config-if)# shutdown
```

---

## Verification

```cisco
show track
```

Expected:

```text
Track 2 Reachability Down
```

---

## Verification

```cisco
show route-map
```

Expected:

```text
100.64.2.1 track 2 [down]
100.64.1.1 track 1 [up]
```

---

## Verification

```bash
ping 203.0.113.10
```

Expected:

```text
Ping still successful.
```

---

## PASS Criteria

```text
Automatic failover to ISP1.
```

---

# 14. ISP2 Recovery Verification

## Recovery

```cisco
ISP2(config)# interface g0/0
ISP2(config-if)# no shutdown
```

---

## Verification

```cisco
show track
```

Expected:

```text
Track 2 Reachability Up
```

---

## Verification

```cisco
show route-map
```

Expected:

```text
100.64.2.1 track 2 [up]
```

---

## PASS Criteria

```text
Traffic automatically returns to preferred ISP2 path.
```

---

# 15. QoS Verification

## Command

```cisco
show policy-map interface g0/0
```

## Expected Result

```text
Class-map VOICE
Class-map VIDEO
Class-map SCAVENGER
```

Statistics should show packet matches.

---

## PASS Criteria

```text
QoS policy active on WAN interface.
```

---

# Final Validation Checklist

```text
[PASS] OSPF reachability verified
[PASS] HSRP operational
[PASS] INTERNET-SRV1 reachable
[PASS] Guest VLAN operational
[PASS] Guest segmentation enforced
[PASS] Direct Internet Access working
[PASS] NAT translations observed
[PASS] PBR policy matching traffic
[PASS] Voice traffic prefers ISP1
[PASS] Video traffic prefers ISP1
[PASS] Guest traffic prefers ISP2
[PASS] ISP2 failover verified
[PASS] ISP2 recovery verified
[PASS] IP SLA operational
[PASS] Tracking objects operational
[PASS] QoS policy operational
```

## Phase 06 Completion Status

```text
PHASE 06 — SD-WAN UNDERLAY ENHANCEMENT
STATUS: COMPLETED
```
