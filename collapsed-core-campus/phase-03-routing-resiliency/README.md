# PHASE 03 — Routing Resiliency

## Overview

Phase 03 focuses on implementing routing resiliency mechanisms in an enterprise collapsed-core campus topology.  
The objective of this phase is to provide stable and predictable network behavior during link, device, and gateway failures.

This phase introduces:

- Dual ISP edge resiliency
- Floating static routes
- IP SLA and object tracking
- OSPF dynamic routing
- OSPF convergence behavior
- Equal Cost Multi Path (ECMP)
- Preferred and backup path design
- HSRP first-hop redundancy
- Failure isolation concepts
- Hierarchical stability and scalability principles

The lab emphasizes not only CLI configuration, but also enterprise design philosophy and operational behavior during failures.

---

# Topology Changes from Phase 02

The following topology enhancements were introduced:

- Added ISP1 router
- Added ISP2 router
- Added ext-conn-0
- Added ext-conn-1
- EDGE1 connected redundantly to:
  - ISP1
  - ISP2
  - CORE1
  - CORE2

The collapsed-core campus topology now includes:

- Dual WAN uplinks
- Dual core routing paths
- Redundant first-hop gateway services
- Dynamic routing convergence mechanisms

---

# Design Goals

## 1. WAN Edge Resiliency

Provide redundant internet connectivity using:

- Primary ISP path
- Secondary ISP backup path
- Floating static routes
- IP SLA tracking

Goals:

- Automatic WAN failover
- Minimal traffic interruption
- Automatic recovery after restoration

---

## 2. Dynamic Routing Resiliency

OSPF was implemented between:

- EDGE1
- CORE1
- CORE2

Goals:

- Dynamic route learning
- Automatic convergence
- ECMP path support
- Fast failover and recovery

---

## 3. First Hop Redundancy

HSRP was implemented on:

- VLAN10
- VLAN20
- VLAN40

Goals:

- Virtual default gateway
- Gateway failover
- Active/Standby redundancy
- Load sharing between core switches

---

## 4. Preferred Path Design

OSPF cost manipulation was used to create:

- Preferred primary routing path
- Backup secondary routing path

Goals:

- Predictable traffic forwarding
- Controlled failover behavior
- Basic traffic engineering concepts

---

## 5. Failure Isolation

The network was validated against access-layer failures to confirm:

- Stable core routing operation
- Limited failure domains
- Minimal route churn
- Hierarchical stability

---

# Technologies Implemented

| Technology | Purpose |
|---|---|
| Floating Static Route | WAN backup routing |
| IP SLA | WAN reachability monitoring |
| Object Tracking | Conditional route failover |
| OSPF | Dynamic routing protocol |
| ECMP | Equal-cost load balancing |
| OSPF Cost Manipulation | Preferred path selection |
| HSRP | First-hop gateway redundancy |
| Passive Interface | OSPF stability and security |

---

# OSPF Design

## Area Design

- Single Area 0 topology

## OSPF Router IDs

| Device | Router ID |
|---|---|
| EDGE1 | 1.1.1.1 |
| CORE1 | 2.2.2.2 |
| CORE2 | 3.3.3.3 |

## OSPF Objectives

- Dynamic route learning
- Fast convergence
- Redundant path utilization
- Automatic failover and recovery

---

# HSRP Design

## Virtual Gateway Addresses

| VLAN | Virtual IP |
|---|---|
| VLAN10 | 10.10.10.254 |
| VLAN20 | 10.20.20.254 |
| VLAN40 | 10.40.40.254 |

## Active/Standby Roles

| VLAN | Active Router | Standby Router |
|---|---|
| VLAN10 | CORE1 | CORE2 |
| VLAN20 | CORE2 | CORE1 |
| VLAN40 | CORE1 | CORE2 |

## HSRP Features Used

- Priority
- Preempt
- Active/Standby failover

---

# WAN Edge Design

## ISP Connectivity

| ISP | EDGE1 Interface |
|---|---|
| ISP1 | G0/0 |
| ISP2 | G0/3 |

## WAN Redundancy Mechanism

- Primary default route via ISP1
- Floating static backup via ISP2
- IP SLA reachability monitoring
- Object tracking for route failover

---

# Operational Validation Performed

The following validation scenarios were tested successfully:

## WAN Failover

- ISP1 failure
- Automatic backup route activation through ISP2
- Automatic recovery after restoration

## OSPF Convergence

- CORE1 uplink failure
- Neighbor adjacency loss detection
- SPF recalculation
- Backup path promotion

## HSRP Failover

- CORE1 SVI shutdown
- HSRP Active role transition
- Virtual gateway continuity verification

## Preferred Path Recovery

- Primary OSPF path restoration
- Preferred route reinstallation

## Failure Isolation

- Access-layer uplink failure
- Core routing stability maintained
- No major routing instability observed

---

# Enterprise Design Concepts Learned

This phase focused heavily on enterprise routing design concepts:

- Routing resiliency
- Gateway resiliency
- Hierarchical network design
- Failure domain isolation
- Stable convergence
- Traffic engineering fundamentals
- Scalable routing architecture
- Operational stability

---

# Key Takeaways

This phase demonstrated that enterprise networks must be designed with the assumption that failures will occur.

The objective of resilient design is not to eliminate failures, but to:

- Detect failures quickly
- Contain failure impact
- Recover automatically
- Maintain service continuity

The combination of:

- OSPF
- HSRP
- Dual Core
- Dual ISP
- Preferred/Backup routing paths

provides a stable and scalable enterprise campus routing architecture.

---

# Phase Completion Status

| Component | Status |
|---|---|
| Dual ISP Connectivity | Complete |
| Floating Static Routing | Complete |
| IP SLA Tracking | Complete |
| OSPF Deployment | Complete |
| OSPF Convergence Validation | Complete |
| ECMP Validation | Complete |
| Preferred Path Design | Complete |
| HSRP Deployment | Complete |
| Gateway Failover Validation | Complete |
| Failure Isolation Validation | Complete |
| Operational Validation | Complete |
