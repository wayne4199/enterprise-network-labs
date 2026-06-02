# PHASE 01 — verification.md

## Overview

This document records all verification procedures performed during PHASE 01.

The objective is to confirm that:

* VLANs are operational
* Access ports are assigned correctly
* Trunks are forwarding VLANs correctly
* Rapid-PVST is operational
* STP Root placement is correct
* SVIs are operational
* HSRP is functioning correctly
* Inter-VLAN routing is operational
* End-to-end communication is successful

---

# Verification 1 — Hostname

## Command

```cisco
show running-config | include hostname
```

## Expected Result

Examples:

```text
hostname ISP1
hostname ISP2
hostname EDGE1
hostname CORE1
hostname CORE2
hostname ACC1
hostname ACC2
hostname ACC3
hostname SRV-ACC1
```

---

# Verification 2 — VLAN Database

## Command

```cisco
show vlan brief
```

## Expected Result

```text
10 EXEC
11 BIZ-ADMIN

20 SALES
21 MKT

30 IT-OPS
31 IT-SEC

40 MGMT
50 GUEST

60 PRINTER
70 SERVER

999 NATIVE-BLACKHOLE
```

All VLANs should exist on:

* CORE1
* CORE2
* ACC1
* ACC2
* ACC3
* SRV-ACC1

---

# Verification 3 — Access Port Assignment

## Command

```cisco
show vlan brief
```

## Expected Result

### ACC1

```text
VLAN10  Gi0/2
VLAN11  Gi0/3
VLAN60  Gi1/0
```

### ACC2

```text
VLAN20  Gi0/2
VLAN21  Gi0/3
```

### ACC3

```text
VLAN30  Gi0/2
VLAN31  Gi0/3
```

### SRV-ACC1

```text
VLAN70  Gi0/1
VLAN40  Gi0/2
```

---

# Verification 4 — Trunk Status

## Command

```cisco
show interfaces trunk
```

## Expected Result

All switch-to-switch links appear as trunks.

Native VLAN:

```text
999
```

Allowed VLANs:

```text
10,11,20,21,30,31,40,50,60,70,999
```

---

# Verification 5 — Rapid-PVST

## Command

```cisco
show spanning-tree summary
```

## Expected Result

```text
Switch is in rapid-pvst mode
```

for all Layer 2 switches.

---

# Verification 6 — STP Root Alignment

## VLAN 10

### CORE1

```cisco
show spanning-tree vlan 10
```

Expected:

```text
This bridge is the root
```

---

## VLAN 20

### CORE2

```cisco
show spanning-tree vlan 20
```

Expected:

```text
This bridge is the root
```

---

## Root Placement Summary

### CORE1 Root

```text
VLAN10
VLAN11
VLAN30
VLAN40
VLAN60
```

### CORE2 Root

```text
VLAN20
VLAN21
VLAN31
VLAN50
VLAN70
```

---

# Verification 7 — SVI Status

## Command

```cisco
show ip interface brief
```

## Expected Result

### CORE1

```text
Vlan10 up/up
Vlan11 up/up
Vlan20 up/up
Vlan21 up/up
Vlan30 up/up
Vlan31 up/up
Vlan40 up/up
Vlan50 up/up
Vlan60 up/up
Vlan70 up/up
```

### CORE2

```text
Vlan10 up/up
Vlan11 up/up
Vlan20 up/up
Vlan21 up/up
Vlan30 up/up
Vlan31 up/up
Vlan40 up/up
Vlan50 up/up
Vlan60 up/up
Vlan70 up/up
```

---

# Verification 8 — HSRP Status

## Command

```cisco
show standby brief
```

## Expected Result

### CORE1 Active

```text
Vlan10
Vlan11
Vlan30
Vlan40
Vlan60
```

### CORE2 Active

```text
Vlan20
Vlan21
Vlan31
Vlan50
Vlan70
```

---

## HSRP Virtual IP

Expected:

```text
10.10.X.254
```

for every VLAN.

---

# Verification 9 — Linux Desktop Addressing

## Command

```bash
ip addr
```

## Expected Result

Example:

```text
inet 10.10.10.10/24
```

---

## Command

```bash
ip route
```

## Expected Result

Example:

```text
default via 10.10.10.254
```

---

# Verification 10 — Ubuntu Addressing

## Command

```bash
ip addr
```

Expected:

```text
10.10.40.10/24
10.10.70.10/24
```

depending on the host.

---

## Command

```bash
ip route
```

Expected:

```text
default via 10.10.40.254
```

or

```text
default via 10.10.70.254
```

---

# Verification 11 — Gateway Reachability

## EXEC

```bash
ping 10.10.10.254
```

Expected:

```text
Success
```

---

## SALES

```bash
ping 10.10.20.254
```

Expected:

```text
Success
```

---

## SERVER1

```bash
ping 10.10.70.254
```

Expected:

```text
Success
```

---

# Verification 12 — Inter-VLAN Connectivity

## EXEC → BIZ-ADMIN

```bash
ping 10.10.11.10
```

Expected:

```text
Success
```

---

## EXEC → SALES

```bash
ping 10.10.20.10
```

Expected:

```text
Success
```

---

## EXEC → IT-OPS

```bash
ping 10.10.30.10
```

Expected:

```text
Success
```

---

## EXEC → MGMT-SRV1

```bash
ping 10.10.40.10
```

Expected:

```text
Success
```

---

## EXEC → SERVER1

```bash
ping 10.10.70.10
```

Expected:

```text
Success
```

---

# Verification 13 — End-to-End Validation

The following must be confirmed:

* VLAN forwarding works
* Trunk forwarding works
* STP is stable
* HSRP is operational
* Gateway redundancy is established
* Inter-VLAN routing is operational
* Endpoints can communicate across VLAN boundaries

---

# PHASE 01 Completion Criteria

PHASE 01 is considered complete when all of the following are true:

* VLAN verification passes
* Trunk verification passes
* Rapid-PVST verification passes
* STP Root verification passes
* SVI verification passes
* HSRP verification passes
* Gateway reachability passes
* Inter-VLAN connectivity passes
* End-to-end communication passes

Successful completion establishes the baseline campus infrastructure for all future phases.
