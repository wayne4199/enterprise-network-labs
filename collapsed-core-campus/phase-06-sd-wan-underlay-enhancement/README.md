# PHASE 06 — SD-WAN Underlay Enhancement

## Overview

This phase extends the enterprise campus WAN edge by introducing SD-WAN underlay concepts using traditional Cisco Enterprise technologies. The objective is to build practical experience with Policy-Based Routing (PBR), IP SLA, application-aware forwarding, traffic steering, segmentation policies, Direct Internet Access (DIA), and Local Internet Breakout (LIB) while maintaining the existing collapsed-core campus architecture.

Unlike a full SD-WAN overlay deployment, this phase focuses on underlay intelligence and WAN edge decision-making using routing, tracking, QoS, and policy mechanisms already available on enterprise routers.

---

## Learning Objectives

### 1. Policy-Based Routing (PBR)

* Understand destination-independent forwarding decisions.
* Override normal routing-table behavior.
* Forward traffic according to business requirements rather than shortest path.

### 2. SLA-Based WAN Policy

* Monitor WAN health using IP SLA.
* Track ISP reachability.
* Dynamically adjust forwarding behavior when WAN conditions change.

### 3. Application-Aware Path Selection

* Classify applications using ACLs.
* Apply forwarding decisions per application category.
* Simulate SD-WAN style application-aware routing.

### 4. QoS + WAN Integration

* Reuse MQC QoS policies from Phase 05.
* Integrate traffic classification with WAN forwarding decisions.
* Validate QoS behavior at the WAN edge.

### 5. Business Intent Traffic Steering

* Translate business requirements into forwarding policies.
* Route different traffic classes through different WAN links.

### 6. Traffic Segmentation Policy

* Isolate guest traffic from internal resources.
* Enforce security boundaries between user groups.
* Validate segmentation using ACL-based policy enforcement.

### 7. Direct Internet Access (DIA)

* Provide direct Internet connectivity without centralized backhauling.
* Enable Internet access through WAN edge policy decisions.

### 8. Local Internet Breakout (LIB)

* Allow selected traffic to exit locally through a designated ISP.
* Reduce unnecessary WAN traversal.
* Simulate branch Internet breakout behavior.

### 9. WAN Edge Design Concepts

* Understand ISP redundancy.
* Implement primary/secondary WAN design.
* Build resilient forwarding policies.

### 10. IP SLA Enhancement

* Deploy multiple SLA probes.
* Use tracking objects for policy decisions.
* Simulate WAN availability monitoring and automatic failover.

---

## Topology Enhancements

### Existing Infrastructure

* EDGE1
* ISP1
* ISP2
* CORE1
* CORE2
* ACC1
* ACC2
* ACC3
* SRV-ACC1
* MGMT-SRV1
* USER1
* MGMT1

### New Nodes Added

#### INTERNET-SRV1

Simulated Internet server used to validate:

* DIA
* LIB
* WAN path selection
* NAT behavior

#### INTERNET-SW

Unmanaged switch used to connect:

* ISP1
* ISP2
* INTERNET-SRV1

This provides a shared Internet segment for WAN testing.

#### GUEST1

Guest endpoint connected to VLAN 50.

Used to validate:

* Traffic Segmentation
* DIA
* LIB
* PBR
* WAN failover behavior

---

## WAN Design

### ISP1

Primary WAN path.

Used for:

* Voice traffic
* Video traffic
* Default enterprise services

### ISP2

Guest Internet breakout path.

Used for:

* Guest traffic
* Local Internet Breakout

---

## Business Intent Policy

### Voice Traffic

Source Network:

* VLAN 30
* 10.30.30.0/24

Preferred Path:

* ISP1

Reason:

* Lowest latency
* Business-critical communications

---

### Video Traffic

Source Network:

* VLAN 20
* 10.20.20.0/24

Preferred Path:

* ISP1

Reason:

* Consistent bandwidth requirements

---

### Guest Traffic

Source Network:

* VLAN 50
* 10.50.50.0/24

Preferred Path:

* ISP2

Backup Path:

* ISP1

Reason:

* Preserve enterprise WAN resources
* Enable Local Internet Breakout

---

## Traffic Segmentation Policy

Guest users are restricted from accessing internal resources.

### Blocked Networks

* VLAN 10 Users
* VLAN 20 Servers
* VLAN 30 Voice
* VLAN 40 Management

### Allowed Destinations

* Internet resources
* External services

This policy enforces isolation between guest and enterprise traffic.

---

## IP SLA Deployment

### IP SLA 1

Target:

* ISP1 (100.64.1.1)

Purpose:

* Monitor primary WAN availability.

### IP SLA 2

Target:

* ISP2 (100.64.2.1)

Purpose:

* Monitor guest breakout WAN availability.

### Tracking Objects

* Track 1 → ISP1 Reachability
* Track 2 → ISP2 Reachability

These tracking objects are referenced directly by PBR route-maps.

---

## Local Internet Breakout (LIB)

Guest traffic exits directly through ISP2.

Traffic Flow:

GUEST1 → EDGE1 → ISP2 → INTERNET-SRV1

Benefits:

* Reduced WAN utilization
* Faster Internet access
* Simplified breakout architecture

---

## Direct Internet Access (DIA)

Internet connectivity is provided directly from the WAN edge.

Traffic Flow:

Enterprise User → EDGE1 → ISP → Internet

No centralized Internet hub is required.

---

## Verification Highlights

Successful validation was completed for:

* PBR policy forwarding
* SLA-based failover
* IP SLA tracking
* Application-aware path selection
* Guest traffic steering
* Traffic segmentation
* NAT translation
* DIA operation
* LIB operation
* ISP failover and recovery

---

## Key Design Decisions

* Preserve collapsed-core campus architecture.
* Maintain existing QoS policies from Phase 05.
* Use IP SLA for WAN health monitoring.
* Use PBR for business-intent forwarding.
* Use ACLs for segmentation enforcement.
* Implement guest Internet breakout through ISP2.
* Keep the design aligned with ENSLD and ENSDWI enterprise WAN design principles.

---

## Skills Developed

This phase provides hands-on experience with:

* Enterprise WAN architecture
* WAN resiliency
* Policy-Based Routing
* Application-aware forwarding
* QoS integration
* IP SLA
* Traffic segmentation
* Direct Internet Access
* Local Internet Breakout
* SD-WAN underlay concepts

These skills form the foundation for the next phase involving more advanced SD-WAN and automation topics.
