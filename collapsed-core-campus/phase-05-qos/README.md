# PHASE 05 — Enterprise QoS

## Overview

Phase 05 focuses on Enterprise WAN QoS design, implementation, verification, troubleshooting, and operational recovery.

The lab extends the existing collapsed-core campus topology from previous phases and introduces practical WAN QoS concepts commonly used in Enterprise WAN, MPLS WAN, Internet Edge, and SD-WAN environments.

The primary goal of this phase is not simply configuring QoS commands, but understanding:

- traffic classification
- DSCP marking
- queue behavior
- congestion handling
- WAN bandwidth management
- operational troubleshooting
- QoS lifecycle management

This phase intentionally emphasizes operational analysis and WAN edge behavior in preparation for future SD-WAN underlay and overlay studies.

---

# Objectives

The goals of this phase are:

- Understand MQC (Modular QoS CLI)
- Implement DSCP-based traffic classification
- Configure LLQ for voice traffic
- Configure CBWFQ for video traffic
- Implement hierarchical QoS (HQoS)
- Understand policing vs shaping
- Implement DSCP re-marking and trust boundaries
- Configure scavenger traffic handling
- Simulate WAN congestion and queue contention
- Perform operational QoS troubleshooting
- Analyze queue statistics and drop behavior
- Prepare for SD-WAN SLA and application-aware routing concepts

---

# Topology Focus

QoS was intentionally implemented primarily on EDGE1.

This reflects real Enterprise WAN design principles:

| Device Role | QoS Responsibility |
|---|---|
| Access Layer | Classification / Trust Boundary |
| Core Layer | Preserve DSCP |
| WAN Edge | Queueing / Shaping / Congestion Handling |

EDGE1 acts as the Enterprise WAN Edge router and represents:
- Internet edge
- MPLS WAN edge
- SD-WAN WAN transport edge

---

# Technologies Implemented

## MQC Components

The following MQC components were implemented:

- `class-map`
- `policy-map`
- `service-policy`

---

## DSCP Classification

Traffic was classified using DSCP values:

| Traffic Type | DSCP |
|---|---|
| Voice | EF (46) |
| Video | AF41 (34) |
| Scavenger | CS1 |

---

## LLQ (Low Latency Queue)

Voice traffic was protected using:

```cisco
priority percent 10
