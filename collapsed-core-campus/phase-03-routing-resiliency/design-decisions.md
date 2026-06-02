# PHASE 03 — Routing Resiliency

# Design Decisions

This document explains the major architectural and operational design decisions made during Phase 03.

The goal of this phase was not only to configure resiliency technologies, but also to understand why specific enterprise design choices are made.

---

# 1. Collapsed-Core Architecture Retained

## Decision

The existing collapsed-core campus design from Phase 02 was retained.

The topology continues to use:

- EDGE1
- CORE1
- CORE2
- Access Layer switches

without introducing additional distribution-layer devices.

---

## Reasoning

The collapsed-core design is appropriate for:

- Small-to-medium enterprise environments
- Simpler operational management
- Reduced complexity
- Lower resource requirements in CML

This approach also keeps the focus on routing resiliency concepts rather than large-scale campus hierarchy.

---

# 2. Dual ISP WAN Design

## Decision

Two ISP routers were introduced:

- ISP1
- ISP2

EDGE1 was connected redundantly to both ISPs.

---

## Reasoning

The dual ISP design provides:

- WAN redundancy
- Automatic failover capability
- Reduced dependency on a single upstream provider

This simulates a realistic enterprise internet edge architecture.

---

# 3. Floating Static Route with IP SLA Tracking

## Decision

The WAN edge initially used:

- Floating static routes
- Administrative Distance manipulation

Later enhanced with:

- IP SLA
- Object Tracking

---

## Reasoning

Simple floating static routes alone do not reliably detect upstream reachability failures.

IP SLA tracking was introduced to provide:

- Active reachability verification
- Reliable failover
- Automatic route withdrawal
- Automatic route restoration

This reflects real-world enterprise WAN edge practices.

---

# 4. OSPF Selected as the Dynamic Routing Protocol

## Decision

OSPF was deployed between:

- EDGE1
- CORE1
- CORE2

using:

- Single Area 0
- Loopback-based Router IDs

---

## Reasoning

OSPF was selected because it provides:

- Fast convergence
- Dynamic topology awareness
- Enterprise scalability
- Vendor-neutral standards-based routing

Single Area 0 was intentionally chosen to:

- Reduce complexity
- Focus on core OSPF fundamentals
- Simplify LSDB behavior during early resiliency studies

---

# 5. Loopback Interfaces Used for Router IDs

## Decision

Loopback interfaces were added:

| Device | Loopback |
|---|---|
| EDGE1 | 1.1.1.1 |
| CORE1 | 2.2.2.2 |
| CORE2 | 3.3.3.3 |

---

## Reasoning

Loopback-based Router IDs provide:

- Stable OSPF identification
- Predictable neighbor relationships
- Reduced risk of Router ID changes during interface failures

This is considered enterprise best practice.

---

# 6. Passive Interface Strategy

## Decision

Passive-interface default was implemented on CORE1 and CORE2.

Only routed uplinks toward EDGE1 were configured as non-passive.

---

## Reasoning

This design prevents unnecessary OSPF hello traffic toward:

- User VLANs
- Server VLANs
- Management VLANs

Benefits include:

- Reduced control-plane noise
- Improved stability
- Reduced unnecessary neighbor discovery attempts
- Improved security posture

This reflects common enterprise campus OSPF design practices.

---

# 7. Static Internal Routes Removed

## Decision

Legacy static routes on EDGE1 were removed after OSPF deployment.

---

## Reasoning

The design objective was to transition from:

```text
Static Routing
```

to:

```text
Dynamic Routing
```

This allowed:

- Dynamic route learning
- Automatic convergence
- Redundant path utilization
- Simplified operations

OSPF became the authoritative internal routing mechanism.

---

# 8. ECMP Initially Allowed

## Decision

OSPF Equal Cost Multi Path (ECMP) was initially allowed between:

- EDGE1 ↔ CORE1
- EDGE1 ↔ CORE2

---

## Reasoning

ECMP was used to demonstrate:

- Multi-path routing
- Redundant forwarding behavior
- Dynamic convergence capability

This provided a foundation before implementing preferred-path manipulation.

---

# 9. Preferred Path Design Introduced

## Decision

OSPF cost manipulation was applied on:

```cisco
EDGE1 interface G0/2
 ip ospf cost 50
```

to prefer CORE1 as the primary path.

---

## Reasoning

Real enterprise networks rarely use all paths equally.

Preferred-path design allows:

- Predictable forwarding behavior
- Controlled traffic engineering
- Backup path designation
- Deterministic convergence

This mirrors real-world enterprise routing policy design.

---

# 10. HSRP Selected for First-Hop Redundancy

## Decision

HSRP was implemented for:

- VLAN10
- VLAN20
- VLAN40

using:

- Active/Standby gateway roles
- Preempt
- Priority-based failover

---

## Reasoning

Dynamic routing protocols protect router-to-router communication.

However, end hosts still require:

- Stable default gateways
- Gateway failover capability

HSRP provides:

- Virtual gateway continuity
- Fast failover
- Transparent gateway recovery

This reflects standard enterprise campus gateway design.

---

# 11. Load Sharing Between CORE1 and CORE2

## Decision

HSRP active roles were intentionally distributed:

| VLAN | Active Router |
|---|---|
| VLAN10 | CORE1 |
| VLAN20 | CORE2 |
| VLAN40 | CORE1 |

---

## Reasoning

This design prevents one core switch from becoming:

- Over-utilized
- A constant active gateway bottleneck

Benefits include:

- Better resource utilization
- Improved operational balance
- More realistic enterprise behavior

---

# 12. Hierarchical Stability Preserved

## Decision

Access-layer devices were intentionally excluded from:

- OSPF neighbor formation
- Dynamic routing participation

---

## Reasoning

This limits the routing failure domain.

Benefits include:

- Reduced route churn
- Faster convergence
- Stable control-plane behavior
- Better scalability

Access-layer failures should not destabilize the core routing domain.

This reflects enterprise hierarchical design principles.

---

# 13. Failure Isolation Validated

## Decision

Access-layer uplink failures were intentionally tested.

---

## Reasoning

The goal was to verify that:

- Core OSPF adjacencies remain stable
- HSRP remains operational
- Failure impact remains localized

This validates proper failure-domain isolation.

---

# 14. Summary-Friendly Addressing Discussed

## Decision

The current VLAN addressing scheme was intentionally retained for lab simplicity.

---

## Reasoning

The existing VLAN addressing:

```text
10.10.10.0/24
10.20.20.0/24
10.40.40.0/24
```

is not optimized for route summarization.

However, redesigning addressing was intentionally deferred to keep focus on resiliency behavior.

The concept of:

- hierarchical addressing
- summary-friendly allocation
- scalable route design

was introduced conceptually for future design phases.

---

# 15. Operational Validation Emphasized

## Decision

Every major resiliency mechanism was validated through simulated failures.

---

## Reasoning

Enterprise resiliency design must be operationally verified.

Configuration alone is insufficient.

Validation confirmed:

- automatic failover
- automatic recovery
- stable convergence
- predictable routing behavior

under failure conditions.

---

# Final Design Philosophy

This phase reinforced the following enterprise design principles:

- Failures are inevitable
- Recovery must be automatic
- Failure domains must remain limited
- Routing behavior must remain predictable
- Control-plane stability is critical
- Hierarchical design improves scalability and resiliency

The resulting topology now closely resembles a resilient enterprise collapsed-core campus architecture suitable for advanced routing and design studies.
