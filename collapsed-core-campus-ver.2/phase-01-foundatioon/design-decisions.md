# PHASE 01 — Design Decisions

## Overview

This document explains the intentional design decisions made during PHASE 01.

The goal is not only to document what was configured, but also why specific design choices were made.

---

# Collapsed Core Architecture

A Collapsed Core design was selected as the foundation topology.

Benefits:

* Reduced device count
* Suitable for small and medium enterprise campuses
* Easier operational management
* Supports redundancy through dual core switches
* Provides a realistic foundation for future routing, security, QoS, and SD-WAN phases

This architecture is retained throughout PHASE 01 through PHASE 06.

---

# VLAN Segmentation Strategy

Business functions were separated into dedicated VLANs.

| VLAN | Purpose   |
| ---- | --------- |
| 10   | EXEC      |
| 11   | BIZ-ADMIN |
| 20   | SALES     |
| 21   | MKT       |
| 30   | IT-OPS    |
| 31   | IT-SEC    |
| 40   | MGMT      |
| 50   | GUEST     |
| 60   | PRINTER   |
| 70   | SERVER    |

Design objectives:

* Department isolation
* Policy enforcement
* Future ACL implementation
* Security segmentation
* Route summarization preparation

---

# Native VLAN 999

VLAN 999 is used as the Native VLAN on all trunks.

Reasons:

* Avoid using VLAN 1
* Improve operational consistency
* Reduce accidental traffic leakage
* Align with common enterprise design practices

The VLAN is intentionally named:

```text
NATIVE-BLACKHOLE
```

to clearly indicate that it should never carry production user traffic.

---

# Dual Core Design

Two core switches were deployed.

```text
CORE1
CORE2
```

Reasons:

* Gateway redundancy
* STP redundancy
* HSRP redundancy
* Foundation for future routing resiliency

This design allows traffic forwarding responsibilities to be distributed between the two core switches.

---

# STP and HSRP Alignment

STP Root placement is intentionally aligned with HSRP Active Gateway ownership.

Example:

```text
VLAN10
CORE1 = STP Root
CORE1 = HSRP Active
```

Benefits:

* Predictable forwarding paths
* Reduced Layer 2 suboptimal traffic
* Simplified troubleshooting
* Better operational visibility

---

# Dedicated Server Access Switch

A dedicated switch was introduced for server connectivity.

```text
SRV-ACC1
```

Reasons:

* Logical separation of server infrastructure
* Easier policy implementation
* Improved scalability
* Better representation of enterprise access layer design

---

# Printer Separation

PRINTER1 was placed into a dedicated printer VLAN.

```text
VLAN 60
```

Reasons:

* Future security policy implementation
* Reduced broadcast scope
* Better visibility of printer traffic
* Realistic enterprise segmentation

---

# Linux Desktop Selection Instead of VPCS

Many networking labs use VPCS nodes.

This lab intentionally uses Linux Desktop nodes.

Reasons:

* More realistic host behavior
* Linux troubleshooting practice
* ARP inspection visibility
* DHCP client visibility
* Security validation support
* Future automation opportunities

The Linux Desktop nodes behave similarly to actual Linux hosts rather than simulated VPCS devices.

---

# Static Addressing During PHASE 01

All endpoints use manually assigned addresses during PHASE 01.

Reasons:

* Focus on VLAN and Layer 2 fundamentals
* Remove DHCP dependencies
* Simplify initial troubleshooting
* Allow deterministic testing

This ensures that connectivity issues can be isolated to switching and gateway configuration rather than address assignment services.

---

# Why Endpoint IP Configuration Was Not Persistently Saved

Desktop and Ubuntu nodes were configured using temporary Linux commands.

Examples:

```bash
sudo ip addr add ...
sudo ip route add ...
```

These settings will be lost after reboot.

This was an intentional decision.

Reasons:

* DHCP will be introduced in PHASE 02
* Endpoint addressing will change during DHCP implementation
* Avoid unnecessary duplicate configuration work
* Keep PHASE 01 focused on switching fundamentals

Persistently storing endpoint addresses before DHCP deployment would add operational overhead without providing additional learning value.

---

# Inter-VLAN Routing Location

Inter-VLAN routing is performed on:

```text
CORE1
CORE2
```

using Switch Virtual Interfaces (SVIs).

Reasons:

* Consistent with Collapsed Core design
* Centralized gateway location
* Easier HSRP deployment
* Better support for future routing features

---

# HSRP Load Distribution

HSRP Active roles are intentionally distributed.

CORE1 Active:

```text
VLAN10
VLAN11
VLAN30
VLAN40
VLAN60
```

CORE2 Active:

```text
VLAN20
VLAN21
VLAN31
VLAN50
VLAN70
```

Reasons:

* Avoid single-core forwarding concentration
* Simulate real enterprise gateway distribution
* Prepare for failover validation in later phases

---

# Design Philosophy

The objective of PHASE 01 is not simply to build VLANs.

The objective is to establish a stable enterprise campus foundation that supports:

* Enterprise Services
* Routing Resiliency
* Security Controls
* QoS
* SD-WAN Readiness
* Operations and Troubleshooting

All future phases build upon the design decisions established in PHASE 01.
