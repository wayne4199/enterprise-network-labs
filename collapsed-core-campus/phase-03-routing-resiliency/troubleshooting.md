# PHASE 03 — Routing Resiliency

# Troubleshooting Guide

This document summarizes the major troubleshooting scenarios encountered during Phase 03.

The focus of this phase was understanding:

- routing resiliency
- dynamic convergence
- gateway failover
- operational stability
- failure isolation

---

# 1. ISP Reachability Failure

## Symptom

EDGE1 could not reach:

```text
100.64.2.1
```

Ping failed toward ISP2.

---

## Cause

Incorrect or unstable CML link connection between:

```text
EDGE1 G0/3
↔
ISP2 G0/0
```

---

## Verification

```cisco
show ip interface brief
```

Observed:

```text
EDGE1 G0/3 = down/down
```

while ISP2 interface remained operational.

---

## Resolution

- Removed and recreated the CML link
- Verified correct interface mapping
- Re-enabled interfaces using:

```cisco
interface g0/x
 no shutdown
```

---

# 2. Floating Static Route Did Not Fail Over

## Symptom

After shutting down ISP1 interface:

```text
EDGE1 still preferred ISP1 route
```

and backup route was not activated.

---

## Cause

The initial floating static configuration relied only on:

```cisco
ip route 0.0.0.0 0.0.0.0 100.64.1.1
```

This recursive static route did not actively verify upstream reachability.

Additionally, CML interface-state propagation behavior delayed link-state updates.

---

## Verification

```cisco
show ip route 0.0.0.0
show ip interface brief
```

Observed:

```text
EDGE1 interface remained up/up
```

even when ISP1 interface was administratively down.

---

## Resolution

Implemented:

- IP SLA
- Object Tracking

Configuration:

```cisco
ip sla 1
 icmp-echo 100.64.1.1 source-interface GigabitEthernet0/0

track 1 ip sla 1 reachability

ip route 0.0.0.0 0.0.0.0 100.64.1.1 track 1
```

This allowed reliable failover based on actual reachability.

---

# 3. OSPF Neighbor Missing Between CORE1 and CORE2

## Symptom

Expected OSPF adjacency between CORE1 and CORE2 did not form.

---

## Cause

CORE1 and CORE2 interconnections were Layer 2 switch links, not Layer 3 routed interfaces.

No IP addressing existed on the inter-switch links.

---

## Verification

```cisco
show ip interface brief
```

Observed:

```text
CORE interconnect interfaces had no IP addresses
```

---

## Resolution

Confirmed this behavior was intentional.

OSPF adjacency was correctly designed only between:

```text
EDGE1 ↔ CORE1
EDGE1 ↔ CORE2
```

This aligned with the collapsed-core campus architecture.

---

# 4. OSPF Routes Not Appearing on EDGE1

## Symptom

EDGE1 did not display VLAN routes as OSPF-learned routes.

Expected:

```text
O 10.10.10.0/24
O 10.20.20.0/24
O 10.40.40.0/24
```

---

## Cause

Legacy static routes from Phase 02 still existed on EDGE1.

Static routes had lower Administrative Distance than OSPF routes.

---

## Verification

```cisco
show ip route
show running-config | include ip route
```

Observed:

```text
Known via "static"
```

for internal VLAN networks.

---

## Resolution

Removed legacy static routes:

```cisco
no ip route 10.10.10.0 255.255.255.0 10.255.255.2
no ip route 10.20.20.0 255.255.255.0 10.255.255.6
no ip route 10.40.40.0 255.255.255.0 10.255.255.2
```

OSPF routes then appeared correctly.

---

# 5. OSPF ECMP Behavior Confusion

## Symptom

EDGE1 displayed multiple next-hop entries for the same destination network.

Example:

```text
via 10.255.255.2
via 10.255.255.6
```

---

## Cause

OSPF Equal Cost Multi Path (ECMP) was functioning normally.

Both paths had identical OSPF cost values.

---

## Verification

```cisco
show ip route ospf
```

Observed:

```text
[110/2]
```

for both paths.

---

## Resolution

Confirmed ECMP was expected behavior.

Later, OSPF cost manipulation was applied:

```cisco
interface g0/2
 ip ospf cost 50
```

to create a preferred path design.

---

# 6. HSRP Failover Verification

## Symptom

Need to verify gateway failover during CORE1 failure.

---

## Verification

```cisco
show standby brief
```

Observed:

```text
Vlan10 Active → CORE1
```

before failure.

---

## Test Procedure

Shutdown:

```cisco
interface vlan10
 shutdown
```

on CORE1.

---

## Expected Behavior

CORE2 became:

```text
Active
```

for VLAN10 HSRP group.

---

## Resolution

Confirmed:

- HSRP failover
- Virtual IP continuity
- Gateway resiliency

operated correctly.

---

# 7. HSRP Preempt Recovery Validation

## Symptom

Need to verify automatic Active role restoration after CORE1 recovery.

---

## Verification

```cisco
show standby brief
```

---

## Cause

HSRP preempt functionality controls automatic active-role restoration.

---

## Resolution

Confirmed:

```cisco
standby X preempt
```

was correctly configured.

After recovery:

```text
CORE1 resumed Active role
```

based on higher priority.

---

# 8. Access-Layer Failure Isolation Validation

## Symptom

Need to validate that access-layer failures do not destabilize the routing core.

---

## Test Procedure

Shutdown:

```cisco
CORE1(config)# interface g0/2
CORE1(config-if)# shutdown
```

---

## Verification

```cisco
show ip ospf neighbor
```

on EDGE1.

Observed:

- OSPF core adjacencies remained stable
- No major route churn occurred
- CORE routing plane remained operational

---

## Resolution

Confirmed proper:

- failure isolation
- hierarchical stability
- limited failure domain behavior

---

# 9. CML Interface-State Synchronization Behavior

## Symptom

Interface shutdowns on one side did not immediately reflect on the opposite device.

---

## Cause

CML virtual link-state propagation delay.

This behavior is occasionally observed with:

- IOSv
- ext-connector nodes
- modified virtual links

---

## Verification

```cisco
show ip interface brief
```

showed inconsistent interface states between connected devices.

---

## Resolution

Workarounds included:

- interface shutdown/no shutdown
- recreating links
- waiting for line protocol synchronization
- relying on IP SLA rather than physical interface state

---

# 10. Key Troubleshooting Lessons Learned

This phase reinforced several important operational concepts:

- Interface state does not always equal reachability
- Dynamic routing requires proper route preference understanding
- IP SLA improves WAN failover reliability
- OSPF convergence depends on topology stability
- HSRP protects host gateway continuity
- Hierarchical design limits failure impact
- Failure isolation improves operational stability
- Enterprise resiliency requires validation, not just configuration

---

# Recommended Troubleshooting Commands

## WAN Edge

```cisco
show ip route
show track
show ip sla statistics
show ip interface brief
```

---

## OSPF

```cisco
show ip ospf neighbor
show ip ospf interface brief
show ip route ospf
show ip ospf database
```

---

## HSRP

```cisco
show standby brief
show standby
```

---

## General

```cisco
show running-config
show logging
ping
traceroute
```
