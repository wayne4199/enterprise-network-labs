# PHASE 01 — troubleshooting.md

## Overview

This document records troubleshooting scenarios performed during PHASE 01.

The objective is to understand how Layer 2 and First-Hop Redundancy issues appear, how to identify them, and how to recover from them.

---

# Scenario 1 — Missing VLAN

## Fault Injection

ACC2

```cisco
no vlan 21
```

---

## Symptoms

```cisco
show vlan brief
```

Result:

```text
VLAN 21 missing
```

MKT hosts cannot be assigned to the correct VLAN.

---

## Root Cause

VLAN 21 does not exist in the VLAN database.

---

## Recovery

```cisco
vlan 21
 name MKT
```

---

## Verification

```cisco
show vlan brief
```

Confirm VLAN 21 is present.

---

# Scenario 2 — Incorrect Access VLAN Assignment

## Fault Injection

ACC3

```cisco
interface g0/3
 switchport access vlan 30
```

---

## Symptoms

IT-SEC device is placed into VLAN 30 instead of VLAN 31.

---

## Verification

```cisco
show vlan brief
show running-config interface g0/3
```

---

## Root Cause

Incorrect VLAN assignment on the access interface.

---

## Recovery

```cisco
interface g0/3
 switchport access vlan 31
```

---

## Verification

```cisco
show vlan brief
```

Confirm the interface is now associated with VLAN 31.

---

# Scenario 3 — Native VLAN Mismatch

## Fault Injection

ACC2

```cisco
interface g1/0
 switchport trunk native vlan 999
```

SRV-ACC1

```cisco
interface g0/0
 switchport trunk native vlan 1
```

---

## Symptoms

Possible issues:

* Native VLAN mismatch warnings
* Unexpected Layer 2 behavior
* Potential VLAN leakage
* CDP mismatch messages

---

## Verification

```cisco
show interfaces trunk
```

```cisco
show spanning-tree
```

---

## Root Cause

Native VLAN values do not match on both sides of the trunk.

---

## Recovery

SRV-ACC1

```cisco
interface g0/0
 switchport trunk native vlan 999
```

---

## Verification

```cisco
show interfaces trunk
```

Confirm both sides use VLAN 999.

---

# Scenario 4 — Incorrect STP Root

## Fault Injection

CORE2

```cisco
spanning-tree vlan 10 root primary
```

---

## Symptoms

Unexpected Root Bridge change.

Traffic forwarding path changes.

---

## Verification

```cisco
show spanning-tree vlan 10
```

Result:

```text
CORE2 becomes Root Bridge
```

---

## Root Cause

Incorrect Root Bridge configuration.

---

## Recovery

CORE2

```cisco
no spanning-tree vlan 10 root primary

spanning-tree vlan 10 root secondary
```

CORE1

```cisco
spanning-tree vlan 10 root primary
```

---

## Verification

```cisco
show spanning-tree vlan 10
```

Confirm CORE1 is Root Bridge.

---

# Scenario 5 — SVI Shutdown

## Fault Injection

CORE1

```cisco
interface vlan 31
 shutdown
```

---

## Symptoms

VLAN 31 loses gateway connectivity.

Inter-VLAN communication fails.

---

## Verification

```cisco
show ip interface brief
```

Result:

```text
Vlan31 administratively down
```

---

## Root Cause

SVI was manually shut down.

---

## Recovery

```cisco
interface vlan 31
 no shutdown
```

---

## Verification

```cisco
show ip interface brief
```

Confirm:

```text
Vlan31 up/up
```

---

# Scenario 6 — HSRP Priority Misconfiguration

## Fault Injection

CORE2

```cisco
interface vlan20
 no standby 20 priority
```

---

## Symptoms

Unexpected Active Gateway ownership.

Traffic forwarding path changes.

---

## Verification

```cisco
show standby brief
```

Result:

```text
CORE1 becomes Active for VLAN20
```

---

## Root Cause

HSRP priority removed.

Default priority of 100 becomes active decision factor.

---

## Recovery

```cisco
interface vlan20
 standby 20 priority 110
 standby 20 preempt
```

---

## Verification

```cisco
show standby brief
```

Confirm CORE2 is Active again.

---

# Scenario 7 — Missing Host IP Configuration

## Fault Injection

Linux Desktop node boots without IP configuration.

---

## Symptoms

```bash
ping 10.10.10.254
```

Result:

```text
Network unreachable
```

---

## Verification

```bash
ip addr
ip route
```

Result:

```text
No IPv4 address
No default route
```

---

## Root Cause

Host addressing was never configured.

This is different from a VLAN or routing issue.

The endpoint itself has no network configuration.

---

## Recovery

Example:

```bash
sudo ip addr add 10.10.10.10/24 dev eth0

sudo ip route add default via 10.10.10.254
```

---

## Verification

```bash
ip addr
ip route
```

Confirm:

```text
10.10.10.10/24
default via 10.10.10.254
```

---

# Scenario 8 — Linux Desktop vs VPCS Configuration Confusion

## Symptoms

Attempting to configure Linux Desktop using VPCS commands.

Example:

```text
ip 10.10.10.10/24 10.10.10.254
```

does not work.

---

## Root Cause

The endpoint is a Linux Desktop node, not a VPCS node.

---

## Verification

```bash
hostname
ip addr
```

Linux shell prompt appears.

---

## Recovery

Use Linux networking commands:

```bash
sudo ip addr add 10.10.10.10/24 dev eth0

sudo ip route add default via 10.10.10.254
```

---

# Lessons Learned

PHASE 01 troubleshooting focused on:

* VLAN database issues
* Access VLAN mistakes
* Trunk mismatches
* STP root placement
* SVI availability
* HSRP ownership
* Endpoint addressing
* Linux Desktop operational differences

These troubleshooting exercises establish the foundation required for Enterprise Services, Routing Resiliency, Security, QoS, and SD-WAN phases.
