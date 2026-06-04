# PHASE 01 — Design Decisions

## Overview

This document explains the intentional design decisions made during PHASE 01.

The goal is not only to document what was configured, but also why specific design choices were made.

PHASE 01 Ver.2 extends the original foundation scope by adding routed links, static routing, Internet Simulation reachability, persistent configuration for long-term server nodes, and documentation strategy for CML export/import limitations.

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
* Troubleshooting scenario support

---

# VLAN 50 Reserved for Guest Use

VLAN 50 is created in PHASE 01 even though no guest endpoint is actively used in this phase.

Reasons:

* Preserve the final VLAN structure from the beginning
* Prepare for guest isolation policy in later phases
* Avoid redesigning the VLAN database in later phases
* Support future DIA and Local Internet Breakout validation

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
* Better representation of enterprise collapsed-core design

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
* Clear separation between user access and server access

SRV-ACC1 is connected to ACC2 using a trunk link so that server and management VLANs can be extended to the server access layer.

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
* Preparation for printer access policy validation

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

All endpoint addresses are manually assigned during PHASE 01.

Reasons:

* Focus on VLAN and Layer 2 fundamentals
* Remove DHCP dependencies
* Simplify initial troubleshooting
* Allow deterministic testing
* Validate gateway, SVI, HSRP, and routing behavior without relying on DHCP

This ensures that connectivity issues can be isolated to switching, gateway, and routing configuration rather than address assignment services.

---

# Temporary Endpoint Configuration

Linux Desktop endpoint nodes were configured using temporary Linux commands.

Examples:

```bash
sudo ip addr add ...
sudo ip route add ...
```

These settings are not expected to survive reboot.

Temporary endpoint nodes include:

```text
EXEC
BIZ-ADMIN
SALES
MKT
IT-OPS
IT-SEC
PRINTER1
```

This is intentional because these endpoints will later become DHCP clients during PHASE 02.

---

# Persistent Server Configuration

Unlike the Linux Desktop endpoints, the following server nodes were configured persistently:

```text
SERVER1
MGMT-SRV1
INTERNET-SRV
```

Reasons:

* These nodes are reused across later phases
* They provide or validate infrastructure services
* They should remain reachable after node restart
* They are not intended to become ordinary DHCP clients

---

# Why SERVER1 Uses Persistent Static Addressing

SERVER1 is placed in:

```text
VLAN 70 SERVER
```

with:

```text
IP Address : 10.10.70.10/24
Gateway    : 10.10.70.254
```

Reasons:

* SERVER1 acts as a long-term server-side validation target
* Future phases will use SERVER1 for routing, security, QoS, and application traffic validation
* Server infrastructure should use predictable addressing
* Static server addressing reduces unnecessary troubleshooting noise

SERVER1 uses Ubuntu Netplan for persistent IP/Gateway configuration.

---

# Why MGMT-SRV1 Uses Persistent Static Addressing

MGMT-SRV1 is placed in:

```text
VLAN 40 MGMT
```

with:

```text
IP Address : 10.10.40.10/24
Gateway    : 10.10.40.254
```

Reasons:

* MGMT-SRV1 will be used for enterprise services in PHASE 02
* Planned roles include DHCP, DNS, Syslog, SNMP, and NTP-related validation
* Management servers require stable addressing
* Infrastructure services should not depend on dynamic addressing

MGMT-SRV1 uses Ubuntu Netplan for persistent IP/Gateway configuration.

---

# Why INTERNET-SRV Uses Persistent Static Addressing

INTERNET-SRV is used as the Internet Simulation target.

```text
IP Address : 203.0.113.10/24
Gateway    : 203.0.113.1
```

Reasons:

* It provides a stable external reachability target
* It allows Internet-like testing without depending on real public Internet
* It will be reused in routing, security, QoS, and SD-WAN policy validation
* It supports deterministic troubleshooting

INTERNET-SRV runs Tiny Core Linux.

Persistent configuration is handled through:

```text
/opt/bootlocal.sh
filetool.sh -b
```

---

# Why Endpoint IP Configuration Was Not Persistently Saved

Linux Desktop endpoints were not persistently configured.

Reasons:

* DHCP will be introduced in PHASE 02
* Endpoint addressing will later be handled dynamically
* Persistently storing endpoint addresses now would create duplicate rework
* Temporary addressing is sufficient for PHASE 01 validation
* PHASE 01 focuses on switching, gateway, and routed reachability foundations

This decision applies only to endpoint nodes, not to long-term server nodes.

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
* Clear gateway ownership per VLAN

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
* Align first-hop gateway ownership with STP root placement

---

# Routed Links Added in PHASE 01 Ver.2

Routed links between CORE, EDGE, and ISP devices were added in PHASE 01 Ver.2.

Routed link domains:

```text
CORE1 ↔ EDGE1
CORE2 ↔ EDGE1
EDGE1 ↔ ISP1
EDGE1 ↔ ISP2
ISP1 / ISP2 ↔ INTERNET-SW / INTERNET-SRV
```

Reasons:

* PHASE 01 should not end only at inter-VLAN reachability
* Campus foundation should include a basic routed edge
* Static routing and Internet Simulation reachability are foundational
* Later phases should not need to redesign basic WAN/edge connectivity
* It enables early troubleshooting of routed paths

This extends PHASE 01 from a pure campus switching foundation into a complete campus-to-edge foundation.

---

# Static Routing in PHASE 01

Static routing was selected for PHASE 01.

Reasons:

* Simple and deterministic
* Easy to troubleshoot
* Appropriate before introducing dynamic routing
* Provides a clear baseline for future OSPF and WAN resiliency work
* Helps isolate path and return-route issues

Static routing is used for:

```text
CORE → EDGE default routing
EDGE → ISP default routing
EDGE → Campus return routing
ISP → Campus return routing
```

Dynamic routing is intentionally deferred to later phases.

---

# Internet Simulation Instead of Real Internet Dependency

INTERNET-SRV is used as the Internet Simulation endpoint.

Reasons:

* Lab remains deterministic
* No dependency on public Internet availability
* No dependency on External Connector behavior
* Easier to reproduce on GitHub
* Easier to troubleshoot routing paths
* Useful for future security, QoS, and SD-WAN policy validation

PHASE 01 validates reachability to:

```text
203.0.113.10
```

rather than requiring reachability to public targets such as:

```text
8.8.8.8
```

---

# External Connector Deferred to PHASE 02

External Connector is intentionally not required for PHASE 01 completion.

Reasons:

* PHASE 01 validates routing using INTERNET-SRV
* Real public Internet access is not required for campus foundation validation
* External Connector is mainly needed for Ubuntu package installation
* PHASE 02 requires `apt update` and package installation
* DNS Resolver testing also depends on public DNS reachability

Therefore, External Connector integration is deferred to PHASE 02.

---

# DNS Resolver Deferred to PHASE 02

DNS Resolver configuration is not performed in PHASE 01.

Reasons:

* PHASE 01 uses IP-based reachability validation
* Public DNS testing requires real DNS server reachability
* External Connector is not part of PHASE 01 completion
* PHASE 02 Enterprise Services is the correct phase for DNS Resolver testing
* DNS naturally belongs with DHCP, DNS, NTP, Syslog, and SNMP service validation

DNS Resolver will be configured and validated in PHASE 02 after External Connector is added.

---

# Static Route Troubleshooting Value

During PHASE 01, CORE1 and CORE2 initially failed to ping INTERNET-SRV.

Reason:

* CORE1 and CORE2 used their routed-link source addresses
* ISP routers only had return routes for 10.10.0.0/16
* Return routes for 172.16.0.0/30 and 172.16.0.4/30 were missing

This was an intentional learning opportunity.

Lessons learned:

* End-user traffic and infrastructure-sourced traffic can require different return routes
* Static routing must account for both forward and return paths
* Successful EDGE-to-INTERNET reachability does not automatically guarantee CORE-sourced reachability
* Source IP matters when troubleshooting static routing

---

# CML Extract Configs Strategy

CML `EXTRACT CONFIGS` is used for Cisco network device configuration preservation.

It is reliable for Cisco network device nodes such as:

```text
IOSv
IOSvL2
```

In this lab, the following Cisco nodes should be saved using `write memory` and then preserved using CML `EXTRACT CONFIGS`:

```text
ISP1
ISP2
EDGE1
CORE1
CORE2
ACC1
ACC2
ACC3
SRV-ACC1
```

Reason:

* These nodes store their configuration in Cisco startup-config
* CML can extract their running/startup configuration reliably
* Their lab state can be reconstructed from extracted Cisco configs

---

# CML Extract Configs Limitation

CML `EXTRACT CONFIGS` should not be treated as a full filesystem backup.

It does not fully preserve or export all runtime and OS-level configurations from the following node types:

```text
Linux Desktop
Ubuntu
Tiny Core Linux
Unmanaged Switch
External Connector
```

This means that the following settings may require manual verification or restoration after lab export, download, re-import, or rebuild:

```text
SERVER1 Netplan configuration
MGMT-SRV1 Netplan configuration
INTERNET-SRV /opt/bootlocal.sh configuration
Linux Desktop temporary IP/Gateway settings
```

---

# Why Linux-Based Node Settings Are Documented Separately

Linux-based node settings are documented in GitHub instead of relying only on CML `EXTRACT CONFIGS`.

Reasons:

* Ubuntu and Tiny Core Linux use OS-level configuration files
* Their persistent settings may not be fully captured by CML Extract Configs
* Lab export/import can cause Linux node settings to require verification
* GitHub documentation makes the lab reproducible
* Recovery steps must be available even if CML node state is lost
* Later phases depend on SERVER1, MGMT-SRV1, and INTERNET-SRV being reachable

Therefore, `configs.md` must include:

```text
SERVER1 Netplan configuration
MGMT-SRV1 Netplan configuration
INTERNET-SRV /opt/bootlocal.sh configuration
Tiny Core filetool.sh -b persistence step
```

---

# Lab Export / Import Documentation Strategy

The lab uses the following preservation strategy:

| Node Type          | Preservation Method                                               |
| ------------------ | ----------------------------------------------------------------- |
| IOSv / IOSvL2      | `write memory` + CML `EXTRACT CONFIGS`                            |
| Ubuntu             | Document Netplan configuration in `configs.md`                    |
| Tiny Core Linux    | Document `/opt/bootlocal.sh` and `filetool.sh -b` in `configs.md` |
| Linux Desktop      | Treat IP/Gateway as temporary; reconfigure when needed            |
| Unmanaged Switch   | No CLI configuration required                                     |
| External Connector | Recreate or reconnect as needed in later phases                   |

This strategy ensures that the lab can be rebuilt from GitHub documentation even if Linux-based node configuration is not fully preserved by CML export/import behavior.

---

# Why Linux Desktop Endpoints Are Not Restored from Extracted Configs

Linux Desktop endpoints are validation nodes, not long-term service nodes.

Examples:

```text
EXEC
BIZ-ADMIN
SALES
MKT
IT-OPS
IT-SEC
PRINTER1
```

Their temporary addressing is intentionally not preserved.

Reasons:

* They will become DHCP clients in PHASE 02
* They are used mainly for connectivity validation
* Persistent static addressing would create unnecessary rework
* Their IP/Gateway configuration can be quickly reapplied when needed

---

# Why Server Nodes Must Be Verifiable After Export / Import

The following nodes must be verified after lab export/import or rebuild:

```text
SERVER1
MGMT-SRV1
INTERNET-SRV
```

Reasons:

* They are reused across PHASE 02 through PHASE 06
* They are long-term service or validation nodes
* Later phases depend on their stable addressing
* They are not fully protected by Cisco-style Extract Configs
* Their recovery procedure must be documented in GitHub

Verification after re-import should include:

```text
IP address
Default gateway
Gateway reachability
Reachability to INTERNET-SRV
Reachability between SERVER1 and MGMT-SRV1
```

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

PHASE 01 Ver.2 intentionally includes:

* Campus Layer 2 foundation
* First-hop gateway redundancy
* Inter-VLAN routing
* Routed edge connectivity
* Static routing baseline
* Internet Simulation reachability
* Persistent addressing for long-term service nodes
* CML Extract Configs limitation awareness
* GitHub-based recovery documentation strategy

All future phases build upon the design decisions established in PHASE 01.
