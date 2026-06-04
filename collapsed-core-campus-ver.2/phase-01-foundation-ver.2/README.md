# PHASE 01 — Foundation

## Overview

This phase establishes the Layer 2 and Layer 3 foundation of the Collapsed Core Campus Ver.2 topology.

The objective is to build the baseline campus infrastructure that will be expanded throughout later phases.

PHASE 01 Ver.2 covers not only the campus switching foundation, but also the basic routed edge foundation and Internet Simulation reachability.

The following technologies are implemented:

* VLAN Segmentation
* Access Port Assignment
* 802.1Q Trunking
* Rapid-PVST
* STP Root Alignment
* Switch Virtual Interfaces (SVI)
* Inter-VLAN Routing
* HSRP Gateway Redundancy
* Routed Links
* Static Routing
* Internet Simulation Reachability
* Persistent IP/Gateway Configuration for Key Server Nodes

DNS Resolver configuration is intentionally deferred to PHASE 02 because actual DNS testing and `apt update` require External Connector connectivity.

---

# Topology

## Core Components

### WAN

* ISP1
* ISP2
* EDGE1

### Core Layer

* CORE1
* CORE2

### Access Layer

* ACC1
* ACC2
* ACC3
* SRV-ACC1

### Servers

* SERVER1
* MGMT-SRV1

### Endpoints

* EXEC
* BIZ-ADMIN
* SALES
* MKT
* IT-OPS
* IT-SEC
* PRINTER1

### Internet Simulation

* INTERNET-SRV
* INTERNET-SW

---

# VLAN Design

| VLAN | Name             |
| ---- | ---------------- |
| 10   | EXEC             |
| 11   | BIZ-ADMIN        |
| 20   | SALES            |
| 21   | MKT              |
| 30   | IT-OPS           |
| 31   | IT-SEC           |
| 40   | MGMT             |
| 50   | GUEST            |
| 60   | PRINTER          |
| 70   | SERVER           |
| 999  | NATIVE-BLACKHOLE |

VLAN 50 is reserved for future guest-related policy validation.

VLAN 999 is used as the native blackhole VLAN and should not carry production user traffic.

---

# Access Port Assignment

## ACC1

| Interface | Device    | VLAN    |
| --------- | --------- | ------- |
| G0/2      | EXEC      | VLAN 10 |
| G0/3      | BIZ-ADMIN | VLAN 11 |
| G1/0      | PRINTER1  | VLAN 60 |

## ACC2

| Interface | Device   | VLAN    |
| --------- | -------- | ------- |
| G0/2      | SALES    | VLAN 20 |
| G0/3      | MKT      | VLAN 21 |
| G1/0      | SRV-ACC1 | Trunk   |

## ACC3

| Interface | Device | VLAN    |
| --------- | ------ | ------- |
| G0/2      | IT-OPS | VLAN 30 |
| G0/3      | IT-SEC | VLAN 31 |

## SRV-ACC1

| Interface | Device    | VLAN    |
| --------- | --------- | ------- |
| G0/0      | ACC2      | Trunk   |
| G0/1      | SERVER1   | VLAN 70 |
| G0/2      | MGMT-SRV1 | VLAN 40 |

---

# Trunk Design

All switch-to-switch links use:

* IEEE 802.1Q
* Native VLAN 999
* Explicit VLAN Allow List

Allowed VLANs:

```text
10,11,20,21,30,31,40,50,60,70,999
```

Trunk links include:

```text
CORE1 ↔ CORE2
CORE1 ↔ ACC1
CORE1 ↔ ACC2
CORE1 ↔ ACC3
CORE2 ↔ ACC1
CORE2 ↔ ACC2
CORE2 ↔ ACC3
ACC2  ↔ SRV-ACC1
```

---

# STP Design

Rapid-PVST is enabled across all Layer 2 switches.

## CORE1 Primary Root

* VLAN 10
* VLAN 11
* VLAN 30
* VLAN 40
* VLAN 60

## CORE2 Primary Root

* VLAN 20
* VLAN 21
* VLAN 31
* VLAN 50
* VLAN 70

This design aligns STP forwarding paths with HSRP Active Gateway ownership.

---

# SVI Design

Inter-VLAN routing is performed on CORE1 and CORE2 using SVIs.

## CORE1

```text
VLAN10  10.10.10.1
VLAN11  10.10.11.1
VLAN20  10.10.20.1
VLAN21  10.10.21.1
VLAN30  10.10.30.1
VLAN31  10.10.31.1
VLAN40  10.10.40.1
VLAN50  10.10.50.1
VLAN60  10.10.60.1
VLAN70  10.10.70.1
```

## CORE2

```text
VLAN10  10.10.10.2
VLAN11  10.10.11.2
VLAN20  10.10.20.2
VLAN21  10.10.21.2
VLAN30  10.10.30.2
VLAN31  10.10.31.2
VLAN40  10.10.40.2
VLAN50  10.10.50.2
VLAN60  10.10.60.2
VLAN70  10.10.70.2
```

---

# HSRP Design

HSRP is used to provide default gateway redundancy for all VLANs.

## CORE1 Active

```text
VLAN10
VLAN11
VLAN30
VLAN40
VLAN60
```

## CORE2 Active

```text
VLAN20
VLAN21
VLAN31
VLAN50
VLAN70
```

HSRP VIP format:

```text
10.10.X.254
```

Example:

```text
VLAN10 VIP = 10.10.10.254
VLAN70 VIP = 10.10.70.254
```

---

# Endpoint Addressing

| Device       | IP Address   | Gateway      |
| ------------ | ------------ | ------------ |
| EXEC         | 10.10.10.10  | 10.10.10.254 |
| BIZ-ADMIN    | 10.10.11.10  | 10.10.11.254 |
| SALES        | 10.10.20.10  | 10.10.20.254 |
| MKT          | 10.10.21.10  | 10.10.21.254 |
| IT-OPS       | 10.10.30.10  | 10.10.30.254 |
| IT-SEC       | 10.10.31.10  | 10.10.31.254 |
| PRINTER1     | 10.10.60.10  | 10.10.60.254 |
| MGMT-SRV1    | 10.10.40.10  | 10.10.40.254 |
| SERVER1      | 10.10.70.10  | 10.10.70.254 |
| INTERNET-SRV | 203.0.113.10 | 203.0.113.1  |

---

# Routed Link Design

PHASE 01 Ver.2 adds routed links between the collapsed core, edge, and ISP routers.

## CORE ↔ EDGE Routed Links

| Link          | Device | Interface | IP Address    |
| ------------- | ------ | --------- | ------------- |
| CORE1 ↔ EDGE1 | EDGE1  | G0/2      | 172.16.0.1/30 |
| CORE1 ↔ EDGE1 | CORE1  | G0/0      | 172.16.0.2/30 |
| CORE2 ↔ EDGE1 | EDGE1  | G0/3      | 172.16.0.5/30 |
| CORE2 ↔ EDGE1 | CORE2  | G0/0      | 172.16.0.6/30 |

## EDGE ↔ ISP Routed Links

| Link         | Device | Interface | IP Address    |
| ------------ | ------ | --------- | ------------- |
| EDGE1 ↔ ISP1 | ISP1   | G0/1      | 100.64.1.1/30 |
| EDGE1 ↔ ISP1 | EDGE1  | G0/0      | 100.64.1.2/30 |
| EDGE1 ↔ ISP2 | ISP2   | G0/1      | 100.64.2.1/30 |
| EDGE1 ↔ ISP2 | EDGE1  | G0/1      | 100.64.2.2/30 |

## Internet Simulation Segment

| Device       | Interface | IP Address      |
| ------------ | --------- | --------------- |
| ISP1         | G0/0      | 203.0.113.1/24  |
| ISP2         | G0/0      | 203.0.113.2/24  |
| INTERNET-SRV | eth0      | 203.0.113.10/24 |

The Internet Simulation Segment is connected through INTERNET-SW.

---

# Static Routing Design

Static routing is used in PHASE 01 to provide basic routed reachability.

## CORE1

```text
Default route → EDGE1
0.0.0.0/0 via 172.16.0.1
```

## CORE2

```text
Default route → EDGE1
0.0.0.0/0 via 172.16.0.5
```

## EDGE1

```text
Internal summary route:
10.10.0.0/16 via CORE1
10.10.0.0/16 via CORE2 as backup

Default route:
0.0.0.0/0 via ISP1
0.0.0.0/0 via ISP2 as backup
```

## ISP1 / ISP2

```text
Return route to campus:
10.10.0.0/16 via EDGE1
```

Additional routes to 172.16.0.0/30 and 172.16.0.4/30 were added during troubleshooting so CORE1 and CORE2 could successfully ping INTERNET-SRV using their routed-link source addresses.

---

# Internet Reachability

PHASE 01 Ver.2 validates Internet Simulation reachability using INTERNET-SRV.

The following path was successfully validated:

```text
EXEC
↓
HSRP VIP
↓
CORE1 / CORE2
↓
EDGE1
↓
ISP1 / ISP2
↓
INTERNET-SRV
```

Validation target:

```text
INTERNET-SRV = 203.0.113.10
```

This verifies:

* VLAN forwarding
* Trunk forwarding
* STP stability
* HSRP gateway reachability
* Inter-VLAN routing
* Routed links
* Static routing
* Internet Simulation reachability

Actual public Internet reachability such as `8.8.8.8` is intentionally not required in PHASE 01.

External Connector will be introduced in PHASE 02 when `apt update` and DNS Resolver testing are required.

---

# Important Note — Linux Desktop vs VPCS

Some CML examples and labs use VPCS nodes.

VPCS addressing is configured with:

```text
ip <address>/<mask> <gateway>
```

Example:

```text
ip 10.10.10.10/24 10.10.10.254
```

This lab intentionally uses Linux Desktop nodes instead of VPCS.

Linux Desktop nodes require Linux networking commands:

```bash
sudo ip addr add 10.10.10.10/24 dev eth0
sudo ip route add default via 10.10.10.254
```

Ubuntu server nodes use Linux networking commands or persistent Netplan configuration.

Temporary Ubuntu example:

```bash
sudo ip addr add 10.10.70.10/24 dev ens2
sudo ip route add default via 10.10.70.254
```

Understanding the difference between VPCS and Linux Desktop nodes is important when following CML-based networking labs.

---

# Persistent IP/Gateway Configuration

PHASE 01 uses two different endpoint configuration approaches.

## Temporary Endpoint Configuration

The following Linux Desktop endpoint nodes use temporary IP/Gateway configuration:

```text
EXEC
BIZ-ADMIN
SALES
MKT
IT-OPS
IT-SEC
PRINTER1
```

These nodes are validation endpoints and will later move toward DHCP-based addressing in PHASE 02.

Their IP/Gateway settings are not persistently saved in PHASE 01.

---

## Persistent Server Configuration

The following long-term server nodes are persistently configured:

```text
SERVER1
MGMT-SRV1
INTERNET-SRV
```

These nodes are reused across later phases and therefore require IP/Gateway settings to survive reboot or lab restart.

---

## SERVER1 Persistent Configuration

SERVER1 uses Ubuntu Netplan.

```text
IP Address : 10.10.70.10/24
Gateway    : 10.10.70.254
Interface  : ens2
```

Netplan file:

```text
/etc/netplan/50-cloud-init.yaml
```

---

## MGMT-SRV1 Persistent Configuration

MGMT-SRV1 uses Ubuntu Netplan.

```text
IP Address : 10.10.40.10/24
Gateway    : 10.10.40.254
Interface  : ens2
```

Netplan file:

```text
/etc/netplan/50-cloud-init.yaml
```

---

## INTERNET-SRV Persistent Configuration

INTERNET-SRV uses Tiny Core Linux.

```text
IP Address : 203.0.113.10/24
Gateway    : 203.0.113.1
Interface  : eth0
```

Persistent startup file:

```text
/opt/bootlocal.sh
```

The following lines were added:

```bash
ifconfig eth0 203.0.113.10 netmask 255.255.255.0 up
route add default gw 203.0.113.1
```

Tiny Core Linux persistence was saved using:

```bash
filetool.sh -b
```

---

# CML Extract Configs and Lab Export Notes

CML `EXTRACT CONFIGS` is useful for preserving Cisco network device startup configurations.

It reliably applies to Cisco network device nodes such as:

```text
IOSv
IOSvL2
```

In this lab, the following Cisco nodes should be saved with `write memory` and then preserved with CML `EXTRACT CONFIGS`:

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

However, CML `EXTRACT CONFIGS` does not fully preserve all runtime or OS-level configurations from Linux-based and non-Cisco node types.

Examples include:

```text
Linux Desktop
Ubuntu
Tiny Core Linux
Unmanaged Switch
External Connector
```

Therefore, this lab uses the following documentation strategy:

* Cisco IOSv and IOSvL2 configurations are preserved using `write memory` and CML `EXTRACT CONFIGS`.
* Ubuntu server configurations are documented in `configs.md` using Netplan examples.
* Tiny Core Linux configuration is documented in `configs.md` using `/opt/bootlocal.sh` and `filetool.sh -b`.
* Linux Desktop endpoint IP/Gateway settings are treated as temporary and are not persistently saved.
* Unmanaged Switch and External Connector nodes do not require CLI configuration.

When exporting, downloading, or re-importing the CML lab, Linux-based node settings must be verified and restored manually if needed by following the documented procedures in `configs.md`.

This is especially important for:

```text
SERVER1
MGMT-SRV1
INTERNET-SRV
```

---

# DNS Resolver Deferral

DNS Resolver configuration is not performed in PHASE 01.

Reason:

* PHASE 01 validates Internet Simulation reachability using INTERNET-SRV.
* Public DNS testing requires reachability to real DNS servers such as 8.8.8.8 or 1.1.1.1.
* Actual Internet access requires External Connector integration.
* External Connector is more appropriate in PHASE 02 because PHASE 02 requires `apt update` and package installation.

Therefore, DNS Resolver configuration and verification will be handled in PHASE 02.

---

# Verification Summary

The following validations were completed:

* Hostname verification
* VLAN verification
* Access Port verification
* Trunk verification
* Rapid-PVST verification
* STP Root verification
* SVI verification
* HSRP verification
* HSRP VIP reachability
* Inter-VLAN communication verification
* Routed Link verification
* Static Route verification
* Internet Simulation reachability verification
* SERVER1 persistent IP/Gateway verification
* MGMT-SRV1 persistent IP/Gateway verification
* INTERNET-SRV persistent IP/Gateway verification
* End-to-End connectivity verification

---

# PHASE 01 Completion Criteria

PHASE 01 is considered complete when all of the following are true:

* VLANs are created on all required Layer 2 switches.
* Access ports are assigned to the correct VLANs.
* Trunk links are operational.
* Native VLAN 999 is configured on trunk links.
* Rapid-PVST is enabled.
* STP Root placement aligns with HSRP Active Gateway ownership.
* SVIs are operational on CORE1 and CORE2.
* HSRP is operational for all VLAN gateways.
* Inter-VLAN routing is successful.
* Routed links between CORE, EDGE, and ISP routers are operational.
* Static routes provide campus-to-Internet-Simulation reachability.
* INTERNET-SRV is reachable from campus endpoints.
* SERVER1, MGMT-SRV1, and INTERNET-SRV have persistent IP/Gateway configuration.
* CML Extract Configs limitations are documented.
* Linux-based node recovery procedures are documented in `configs.md`.
* DNS Resolver is intentionally deferred to PHASE 02.

Successful completion establishes the baseline campus infrastructure for PHASE 02 Enterprise Services and later phases.
