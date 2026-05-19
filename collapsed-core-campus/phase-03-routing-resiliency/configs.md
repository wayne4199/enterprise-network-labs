# PHASE 03 — Routing Resiliency

# Device Configuration Summary

This document contains the major configuration changes introduced during Phase 03.

Phase 03 focuses on:

- WAN edge resiliency
- Dynamic routing resiliency
- OSPF convergence
- Preferred path selection
- HSRP first-hop redundancy
- Failure isolation and operational stability

---

# SECTION 1 — WAN Edge Resiliency

## Objective

Implement:

- Dual ISP connectivity
- Floating static routes
- IP SLA tracking
- Automatic WAN failover

---

# ISP1 Configuration

```cisco
hostname ISP1

no ip domain-lookup

interface Loopback0
 ip address 11.11.11.11 255.255.255.255

interface GigabitEthernet0/0
 description TO-EDGE1
 ip address 100.64.1.1 255.255.255.252
 no shutdown

interface GigabitEthernet0/1
 description TO-EXT-CONN-0
 ip address dhcp
 no shutdown

ip route 10.0.0.0 255.0.0.0 100.64.1.2
```

---

# ISP2 Configuration

```cisco
hostname ISP2

no ip domain-lookup

interface Loopback0
 ip address 22.22.22.22 255.255.255.255

interface GigabitEthernet0/0
 description TO-EDGE1
 ip address 100.64.2.1 255.255.255.252
 no shutdown

interface GigabitEthernet0/1
 description TO-EXT-CONN-1
 ip address dhcp
 no shutdown

ip route 10.0.0.0 255.0.0.0 100.64.2.2
```

---

# EDGE1 WAN Uplink Configuration

```cisco
interface Loopback0
 ip address 1.1.1.1 255.255.255.255

interface GigabitEthernet0/0
 description TO-ISP1
 ip address 100.64.1.2 255.255.255.252
 no shutdown

interface GigabitEthernet0/3
 description TO-ISP2
 ip address 100.64.2.2 255.255.255.252
 no shutdown
```

---

# Floating Static Routes

```cisco
ip route 0.0.0.0 0.0.0.0 100.64.1.1
ip route 0.0.0.0 0.0.0.0 100.64.2.1 200
```

---

# IP SLA + Object Tracking

```cisco
ip sla 1
 icmp-echo 100.64.1.1 source-interface GigabitEthernet0/0
 frequency 5
exit

ip sla schedule 1 life forever start-time now

track 1 ip sla 1 reachability

no ip route 0.0.0.0 0.0.0.0 100.64.1.1

ip route 0.0.0.0 0.0.0.0 100.64.1.1 track 1
ip route 0.0.0.0 0.0.0.0 100.64.2.1 200
```

---

# SECTION 2 — OSPF Fundamentals

## Objective

Implement:

- OSPF Area 0
- Dynamic route learning
- OSPF adjacency
- ECMP
- OSPF convergence

---

# EDGE1 OSPF Configuration

```cisco
router ospf 1
 router-id 1.1.1.1

 network 10.255.255.0 0.0.0.3 area 0
 network 10.255.255.4 0.0.0.3 area 0
 network 1.1.1.1 0.0.0.0 area 0
```

---

# CORE1 OSPF Configuration

```cisco
interface Loopback0
 ip address 2.2.2.2 255.255.255.255

router ospf 1
 router-id 2.2.2.2

 network 10.255.255.0 0.0.0.3 area 0
 network 2.2.2.2 0.0.0.0 area 0

 network 10.10.10.0 0.0.0.255 area 0
 network 10.20.20.0 0.0.0.255 area 0
 network 10.40.40.0 0.0.0.255 area 0

 passive-interface default
 no passive-interface GigabitEthernet0/0
```

---

# CORE2 OSPF Configuration

```cisco
interface Loopback0
 ip address 3.3.3.3 255.255.255.255

router ospf 1
 router-id 3.3.3.3

 network 10.255.255.4 0.0.0.3 area 0
 network 3.3.3.3 0.0.0.0 area 0

 network 10.10.10.0 0.0.0.255 area 0
 network 10.20.20.0 0.0.0.255 area 0
 network 10.40.40.0 0.0.0.255 area 0

 passive-interface default
 no passive-interface GigabitEthernet0/0
```

---

# Removal of Legacy Static Internal Routes

EDGE1 internal static routes were removed to allow OSPF dynamic route learning.

```cisco
no ip route 10.10.10.0 255.255.255.0 10.255.255.2
no ip route 10.20.20.0 255.255.255.0 10.255.255.6
no ip route 10.40.40.0 255.255.255.0 10.255.255.2
```

---

# SECTION 3 — HSRP First Hop Redundancy

## Objective

Implement:

- Gateway redundancy
- Active/Standby failover
- Gateway resiliency

---

# CORE1 HSRP Configuration

```cisco
interface vlan10
 standby 10 ip 10.10.10.254
 standby 10 priority 110
 standby 10 preempt

interface vlan20
 standby 20 ip 10.20.20.254
 standby 20 priority 90
 standby 20 preempt

interface vlan40
 standby 40 ip 10.40.40.254
 standby 40 priority 110
 standby 40 preempt
```

---

# CORE2 HSRP Configuration

```cisco
interface vlan10
 standby 10 ip 10.10.10.254
 standby 10 priority 90
 standby 10 preempt

interface vlan20
 standby 20 ip 10.20.20.254
 standby 20 priority 110
 standby 20 preempt

interface vlan40
 standby 40 ip 10.40.40.254
 standby 40 priority 90
 standby 40 preempt
```

---

# SECTION 4 — OSPF Path Manipulation

## Objective

Implement:

- Preferred routing path
- Backup routing path
- OSPF metric manipulation

---

# Preferred Path Configuration

EDGE1 prefers CORE1 as the primary routing path.

```cisco
interface GigabitEthernet0/2
 ip ospf cost 50
```

Result:

- CORE1 path preferred
- CORE2 path used as backup

---

# SECTION 5 — Scalability Concepts

## Objective

Study:

- Route summarization concepts
- Hierarchical addressing
- LSDB scalability
- Stable convergence

---

# Design Notes

Current addressing:

```text
10.10.10.0/24
10.20.20.0/24
10.40.40.0/24
```

is not optimized for route summarization.

Enterprise best practice recommends:

```text
Hierarchical addressing
Summary-friendly subnet allocation
```

for scalable routing design.

---

# SECTION 6 — Failure Domains & Stability

## Objective

Validate:

- Failure isolation
- Stable convergence
- Limited failure domains

---

# Failure Isolation Test

Validated:

- Access-layer link failure
- Core OSPF stability
- Minimal routing disruption

Test example:

```cisco
CORE1(config)# interface g0/2
CORE1(config-if)# shutdown
```

Observed behavior:

- OSPF core adjacency remained stable
- No major route churn
- Failure impact isolated to access layer

---

# SECTION 7 — Operational Validation

## Objective

Validate:

- Routing resiliency
- Gateway resiliency
- Dynamic convergence
- Recovery operations

---

# Validation Scenarios Completed

| Scenario | Result |
|---|---|
| ISP1 Failure | Success |
| Automatic ISP2 Failover | Success |
| IP SLA Recovery | Success |
| OSPF Neighbor Failure | Success |
| OSPF Re-Convergence | Success |
| Preferred Path Recovery | Success |
| HSRP Active/Standby Failover | Success |
| HSRP Preempt Recovery | Success |
| Access Failure Isolation | Success |

---

# Final Operational State

The topology successfully provides:

- Dual ISP redundancy
- Dynamic routing resiliency
- Gateway redundancy
- Preferred and backup routing paths
- Failure isolation
- Stable convergence behavior

The final design closely resembles a resilient enterprise collapsed-core campus architecture.
