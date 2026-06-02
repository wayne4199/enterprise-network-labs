# PHASE 03 — Routing Resiliency

# Verification Guide

This document contains the operational verification procedures used during Phase 03.

The purpose of this phase was to validate:

- WAN edge resiliency
- Dynamic routing convergence
- Gateway redundancy
- Preferred/backup routing behavior
- Failure isolation
- Stable enterprise routing operations

---

# 1. WAN Edge Verification

## Objective

Verify:

- ISP connectivity
- WAN redundancy
- Primary/backup default routes

---

# Verify ISP Reachability

## EDGE1

```cisco
ping 100.64.1.1
ping 100.64.2.1
```

---

# Expected Result

```text
Success rate is 100 percent
```

for both ISP next-hop addresses.

---

# Verify Default Route Installation

## EDGE1

```cisco
show ip route 0.0.0.0
```

---

# Expected Result

Primary route installed:

```text
Known via "static", distance 1
* 100.64.1.1
```

Backup route available through Administrative Distance 200.

---

# 2. IP SLA Verification

## Objective

Verify:

- WAN reachability tracking
- Automatic failover behavior
- Automatic recovery behavior

---

# Verify IP SLA Operation

## EDGE1

```cisco
show ip sla statistics
```

---

# Expected Result

```text
Latest operation return code: OK
```

---

# Verify Track Object Status

## EDGE1

```cisco
show track 1
```

---

# Expected Result

```text
Reachability is Up
```

---

# WAN Failover Test

## Simulate ISP1 Failure

### ISP1

```cisco
conf t
interface g0/0
 shutdown
end
```

---

# Verify Failover

## EDGE1

```cisco
show ip route 0.0.0.0
```

---

# Expected Result

Backup route installed:

```text
Known via "static", distance 200
* 100.64.2.1
```

---

# Verify Recovery

### ISP1

```cisco
conf t
interface g0/0
 no shutdown
end
```

---

# Verify Primary Restoration

## EDGE1

```cisco
show ip route 0.0.0.0
```

---

# Expected Result

Primary route restored:

```text
Known via "static", distance 1
* 100.64.1.1
```

---

# 3. OSPF Neighbor Verification

## Objective

Verify:

- OSPF adjacency
- Neighbor stability
- Dynamic routing operation

---

# Verify OSPF Neighbors

## EDGE1

```cisco
show ip ospf neighbor
```

---

# Expected Result

```text
2.2.2.2 FULL
3.3.3.3 FULL
```

---

# Verify OSPF Routing Table

## EDGE1

```cisco
show ip route ospf
```

---

# Expected Result

OSPF-learned routes:

```text
O 10.10.10.0/24
O 10.20.20.0/24
O 10.40.40.0/24
```

---

# Verify OSPF Database

## EDGE1

```cisco
show ip ospf database
```

---

# Expected Result

Router LSAs present for:

- EDGE1
- CORE1
- CORE2

---

# 4. ECMP Verification

## Objective

Verify Equal Cost Multi Path routing.

---

# Verify ECMP Routes

## EDGE1

```cisco
show ip route ospf
```

---

# Expected Result

Before OSPF cost manipulation:

```text
via 10.255.255.2
via 10.255.255.6
```

for the same destination networks.

---

# 5. OSPF Preferred Path Verification

## Objective

Verify:

- OSPF cost manipulation
- Preferred path selection
- Backup path promotion

---

# Verify OSPF Interface Cost

## EDGE1

```cisco
show ip ospf interface brief
```

---

# Expected Result

Higher OSPF cost on:

```text
GigabitEthernet0/2
```

---

# Verify Preferred Routing Path

## EDGE1

```cisco
show ip route ospf
```

---

# Expected Result

Preferred routes installed through:

```text
10.255.255.2
```

with lower metric values.

---

# Verify Backup Path Promotion

## Simulate CORE1 Failure

### CORE1

```cisco
conf t
interface g0/0
 shutdown
end
```

---

# Verify OSPF Re-Convergence

## EDGE1

```cisco
show ip ospf neighbor
show ip route ospf
```

---

# Expected Result

- CORE1 neighbor removed
- Routes converge through CORE2
- Backup path activated

Example:

```text
via 10.255.255.6
```

---

# Verify Preferred Path Recovery

### CORE1

```cisco
conf t
interface g0/0
 no shutdown
end
```

---

# Verify Route Restoration

## EDGE1

```cisco
show ip route ospf
```

---

# Expected Result

Preferred routes restored through:

```text
10.255.255.2
```

---

# 6. HSRP Verification

## Objective

Verify:

- Gateway redundancy
- Active/Standby operation
- Gateway failover
- Preempt recovery

---

# Verify HSRP Status

## CORE1 / CORE2

```cisco
show standby brief
```

---

# Expected Result

| VLAN | Active Router | Standby Router |
|---|---|---|
| VLAN10 | CORE1 | CORE2 |
| VLAN20 | CORE2 | CORE1 |
| VLAN40 | CORE1 | CORE2 |

---

# Verify HSRP Failover

## Simulate CORE1 VLAN10 Failure

### CORE1

```cisco
conf t
interface vlan10
 shutdown
end
```

---

# Verify HSRP Transition

## CORE2

```cisco
show standby brief
```

---

# Expected Result

```text
Vlan10 Active
```

on CORE2.

---

# Verify HSRP Recovery

### CORE1

```cisco
conf t
interface vlan10
 no shutdown
end
```

---

# Verify Preempt Recovery

## CORE1 / CORE2

```cisco
show standby brief
```

---

# Expected Result

CORE1 resumes Active role due to:

- higher priority
- preempt enabled

---

# 7. Failure Isolation Verification

## Objective

Verify:

- limited failure domains
- hierarchical stability
- control-plane protection

---

# Simulate Access-Layer Failure

### CORE1

```cisco
conf t
interface g0/2
 shutdown
end
```

---

# Verify OSPF Stability

## EDGE1

```cisco
show ip ospf neighbor
```

---

# Expected Result

OSPF core adjacencies remain stable.

Expected neighbors:

```text
2.2.2.2 FULL
3.3.3.3 FULL
```

---

# Verify Minimal Routing Impact

## EDGE1

```cisco
show ip route ospf
```

---

# Expected Result

No major route churn observed.

Core routing remains operational.

---

# 8. General Operational Verification Commands

## Interface Verification

```cisco
show ip interface brief
```

---

## Routing Verification

```cisco
show ip route
show ip route ospf
```

---

## OSPF Verification

```cisco
show ip ospf neighbor
show ip ospf interface brief
show ip ospf database
```

---

## HSRP Verification

```cisco
show standby brief
show standby
```

---

## IP SLA Verification

```cisco
show ip sla statistics
show track
```

---

## Connectivity Testing

```cisco
ping
traceroute
```

---

# Final Verification Summary

The following operational behaviors were successfully validated:

| Feature | Status |
|---|---|
| Dual ISP Connectivity | Verified |
| Floating Static Routing | Verified |
| IP SLA Failover | Verified |
| Automatic Recovery | Verified |
| OSPF Neighbor Formation | Verified |
| OSPF Convergence | Verified |
| ECMP Operation | Verified |
| Preferred Path Selection | Verified |
| Backup Path Promotion | Verified |
| HSRP Failover | Verified |
| HSRP Preempt Recovery | Verified |
| Failure Isolation | Verified |
| Stable Core Routing | Verified |

---

# Verification Conclusion

The topology successfully demonstrated:

- enterprise routing resiliency
- dynamic convergence
- gateway redundancy
- predictable failover behavior
- operational stability
- failure isolation

The final topology behaves as a resilient enterprise collapsed-core campus routing architecture.
