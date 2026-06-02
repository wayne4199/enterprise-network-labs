# PHASE 01 — Foundation

## Overview

This phase establishes the Layer 2 and Layer 3 foundation of the Collapsed Core Campus topology.

The objective is to build the baseline campus infrastructure that will be expanded throughout later phases.

The following technologies are implemented:

* VLAN Segmentation
* Access Port Assignment
* 802.1Q Trunking
* Rapid-PVST
* STP Root Alignment
* Switch Virtual Interfaces (SVI)
* Inter-VLAN Routing
* HSRP Gateway Redundancy

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

---

# Access Port Assignment

## ACC1

| Interface | Device    | VLAN    |
| --------- | --------- | ------- |
| G0/2      | EXEC      | VLAN 10 |
| G0/3      | BIZ-ADMIN | VLAN 11 |
| G1/0      | PRINTER1  | VLAN 60 |

## ACC2

| Interface | Device | VLAN    |
| --------- | ------ | ------- |
| G0/2      | SALES  | VLAN 20 |
| G0/3      | MKT    | VLAN 21 |

## ACC3

| Interface | Device | VLAN    |
| --------- | ------ | ------- |
| G0/2      | IT-OPS | VLAN 30 |
| G0/3      | IT-SEC | VLAN 31 |

## SRV-ACC1

| Interface | Device    | VLAN    |
| --------- | --------- | ------- |
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

This design aligns STP forwarding paths with gateway ownership.

---

# SVI Design

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

HSRP VIP:

```text
10.10.X.254
```

for each VLAN.

---

# Endpoint Addressing

| Device    | IP Address  | Gateway      |
| --------- | ----------- | ------------ |
| EXEC      | 10.10.10.10 | 10.10.10.254 |
| BIZ-ADMIN | 10.10.11.10 | 10.10.11.254 |
| SALES     | 10.10.20.10 | 10.10.20.254 |
| MKT       | 10.10.21.10 | 10.10.21.254 |
| IT-OPS    | 10.10.30.10 | 10.10.30.254 |
| IT-SEC    | 10.10.31.10 | 10.10.31.254 |
| PRINTER1  | 10.10.60.10 | 10.10.60.254 |
| MGMT-SRV1 | 10.10.40.10 | 10.10.40.254 |
| SERVER1   | 10.10.70.10 | 10.10.70.254 |

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

Ubuntu server nodes use:

```bash
sudo ip addr add <ip-address> dev ens2
sudo ip route add default via <gateway>
```

Understanding the difference between VPCS and Linux Desktop nodes is important when following CML-based networking labs.

---

# Verification Summary

The following validations were completed:

* VLAN verification
* Access Port verification
* Trunk verification
* Rapid-PVST verification
* STP Root verification
* SVI verification
* HSRP verification
* HSRP VIP reachability
* Inter-VLAN communication verification
* End-to-End connectivity verification

PHASE 01 is considered complete after successful end-to-end connectivity testing between all VLANs.
