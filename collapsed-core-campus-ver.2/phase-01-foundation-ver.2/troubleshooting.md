# PHASE 01 — troubleshooting.md

## Overview

This document records troubleshooting scenarios performed during PHASE 01.

The objective is to understand how Layer 2, First-Hop Redundancy, Routed Link, Static Routing, Internet Simulation, and Linux host configuration issues appear, how to identify them, and how to recover from them.

PHASE 01 Ver.2 troubleshooting covers:

* VLAN database issues
* Access VLAN mistakes
* Trunk and Native VLAN issues
* STP root placement issues
* SVI availability issues
* HSRP ownership issues
* Endpoint IP/Gateway issues
* Routed link addressing issues
* Static routing return-path issues
* Internet Simulation reachability issues
* Tiny Core Linux persistence issues
* Ubuntu Netplan persistence issues

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

# Scenario 4 — Trunk Encapsulation Missing

## Symptoms

When configuring trunk mode on IOSvL2, the following message may appear:

```text
Command rejected: An interface whose trunk encapsulation is "Auto" can not be configured to "trunk" mode.
```

---

## Root Cause

On IOSvL2, trunk encapsulation must be explicitly configured before enabling trunk mode.

---

## Recovery

```cisco
interface <interface-id>
 switchport trunk encapsulation dot1q
 switchport mode trunk
```

---

## Verification

```cisco
show interfaces trunk
```

Expected:

```text
Encapsulation 802.1q
Mode trunk
```

---

# Scenario 5 — Incorrect STP Root

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

# Scenario 6 — SVI Shutdown

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

# Scenario 7 — HSRP Priority Misconfiguration

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

# Scenario 8 — Missing Host IP Configuration

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

The endpoint itself has no IP address or default route.

---

## Recovery

Example for EXEC:

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

# Scenario 9 — Linux Desktop vs VPCS Configuration Confusion

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

Linux Desktop uses Linux shell commands.

VPCS uses VPCS-specific syntax.

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

# Scenario 10 — Routed Link IP Address Mistake

## Symptoms

CORE1 or CORE2 cannot ping EDGE1.

Routed link does not work as expected.

---

## Example Issue

CORE1 G0/0 was incorrectly configured with:

```text
172.16.0.6/30
```

This address belongs to CORE2.

Correct design:

```text
EDGE1 G0/2 = 172.16.0.1/30
CORE1 G0/0 = 172.16.0.2/30

EDGE1 G0/3 = 172.16.0.5/30
CORE2 G0/0 = 172.16.0.6/30
```

---

## Verification

CORE1:

```cisco
show ip interface brief
show running-config interface g0/0
ping 172.16.0.1
```

CORE2:

```cisco
show ip interface brief
show running-config interface g0/0
ping 172.16.0.5
```

---

## Root Cause

Duplicate or incorrect routed-link IP addressing.

---

## Recovery

CORE1:

```cisco
interface g0/0
 ip address 172.16.0.2 255.255.255.252
 no shutdown
```

---

## Verification

```cisco
ping 172.16.0.1
```

Expected:

```text
Success
```

---

# Scenario 11 — Missing Static Default Route on CORE

## Symptoms

CORE1 or CORE2 can reach directly connected routed links, but cannot reach INTERNET-SRV.

---

## Verification

CORE1:

```cisco
show ip route static
show ip route 0.0.0.0
```

CORE2:

```cisco
show ip route static
show ip route 0.0.0.0
```

---

## Root Cause

CORE1 or CORE2 does not have a default route toward EDGE1.

---

## Recovery

CORE1:

```cisco
ip route 0.0.0.0 0.0.0.0 172.16.0.1
```

CORE2:

```cisco
ip route 0.0.0.0 0.0.0.0 172.16.0.5
```

---

## Verification

```cisco
show ip route 0.0.0.0
ping 203.0.113.10
```

---

# Scenario 12 — Missing EDGE1 Default Route Toward ISP

## Symptoms

CORE1 or CORE2 can reach EDGE1, but traffic does not reach ISP1, ISP2, or INTERNET-SRV.

---

## Verification

EDGE1:

```cisco
show ip route static
show ip route 0.0.0.0
```

---

## Root Cause

EDGE1 does not have a default route toward ISP1 or ISP2.

---

## Recovery

EDGE1:

```cisco
ip route 0.0.0.0 0.0.0.0 100.64.1.1
ip route 0.0.0.0 0.0.0.0 100.64.2.1 10
```

---

## Verification

EDGE1:

```cisco
show ip route 0.0.0.0
ping 203.0.113.10
```

---

# Scenario 13 — Missing Return Route to Campus VLANs

## Symptoms

Campus endpoints cannot reach INTERNET-SRV even though ISP1 and ISP2 can ping INTERNET-SRV.

Example:

```bash
ping 203.0.113.10
```

from EXEC fails.

---

## Verification

ISP1:

```cisco
show ip route 10.10.0.0
```

ISP2:

```cisco
show ip route 10.10.0.0
```

---

## Root Cause

ISP routers do not know how to return traffic to campus VLANs.

Campus VLAN summary:

```text
10.10.0.0/16
```

must point back to EDGE1.

---

## Recovery

ISP1:

```cisco
ip route 10.10.0.0 255.255.0.0 100.64.1.2
```

ISP2:

```cisco
ip route 10.10.0.0 255.255.0.0 100.64.2.2
```

---

## Verification

From EXEC:

```bash
ping 203.0.113.10
```

Expected:

```text
Success
```

---

# Scenario 14 — CORE Ping to INTERNET-SRV Fails While EDGE and ISP Ping Succeed

## Symptoms

The following tests succeed:

```text
ISP1  → INTERNET-SRV
ISP2  → INTERNET-SRV
EDGE1 → INTERNET-SRV
```

But the following tests fail:

```text
CORE1 → INTERNET-SRV
CORE2 → INTERNET-SRV
```

---

## Verification

CORE1:

```cisco
ping 203.0.113.10
```

CORE2:

```cisco
ping 203.0.113.10
```

ISP1:

```cisco
show ip route 172.16.0.0
```

ISP2:

```cisco
show ip route 172.16.0.0
```

---

## Root Cause

CORE1 and CORE2 use their routed-link source IP addresses when pinging INTERNET-SRV.

Examples:

```text
CORE1 source = 172.16.0.2
CORE2 source = 172.16.0.6
```

ISP routers originally had return routes only for:

```text
10.10.0.0/16
```

They did not have return routes for:

```text
172.16.0.0/30
172.16.0.4/30
```

As a result, Echo Request reached INTERNET-SRV, but Echo Reply could not return to CORE1 or CORE2.

---

## Recovery

ISP1:

```cisco
ip route 172.16.0.0 255.255.255.252 100.64.1.2
ip route 172.16.0.4 255.255.255.252 100.64.1.2
```

ISP2:

```cisco
ip route 172.16.0.0 255.255.255.252 100.64.2.2
ip route 172.16.0.4 255.255.255.252 100.64.2.2
```

---

## Verification

CORE1:

```cisco
ping 203.0.113.10
```

CORE2:

```cisco
ping 203.0.113.10
```

Expected:

```text
Success
```

---

# Scenario 15 — EXEC Cannot Reach INTERNET-SRV Because IP/Gateway Was Lost

## Symptoms

EXEC previously reached INTERNET-SRV, but later fails with:

```text
ping: sendto: Network unreachable
```

---

## Verification

EXEC:

```bash
ip addr
ip route
```

Expected problem state:

```text
No 10.10.10.10/24 address
No default route
```

---

## Root Cause

Linux Desktop endpoint addressing was temporary.

After restart or session reset, IP/Gateway settings may disappear.

---

## Recovery

EXEC:

```bash
sudo ip addr add 10.10.10.10/24 dev eth0
sudo ip route add default via 10.10.10.254
```

---

## Verification

```bash
ip addr
ip route
ping 203.0.113.10
```

Expected:

```text
Success
```

---

# Scenario 16 — INTERNET-SRV IP/Gateway Lost After Reboot

## Symptoms

ISP1 or ISP2 can no longer ping INTERNET-SRV.

```cisco
ping 203.0.113.10
```

fails.

---

## Verification

INTERNET-SRV:

```bash
ifconfig eth0
route -n
```

Expected problem state:

```text
No 203.0.113.10 address
No default gateway 203.0.113.1
```

---

## Root Cause

INTERNET-SRV runs Tiny Core Linux.

Runtime commands such as:

```bash
ifconfig eth0 203.0.113.10 netmask 255.255.255.0 up
route add default gw 203.0.113.1
```

are not persistent unless saved through Tiny Core persistence.

---

## Recovery

Edit:

```bash
sudo vi /opt/bootlocal.sh
```

Add:

```bash
ifconfig eth0 203.0.113.10 netmask 255.255.255.0 up
route add default gw 203.0.113.1
```

Save Tiny Core persistence:

```bash
filetool.sh -b
```

---

## Verification

```bash
cat /opt/bootlocal.sh
ifconfig eth0
route -n
```

Expected:

```text
203.0.113.10/24
default gateway 203.0.113.1
```

Then test from ISP1:

```cisco
ping 203.0.113.10
```

---

# Scenario 17 — SERVER1 IP/Gateway Missing

## Symptoms

SERVER1 cannot reach its gateway, MGMT-SRV1, or INTERNET-SRV.

---

## Verification

SERVER1:

```bash
ip addr
ip route
```

Problem state:

```text
ens2 is UP
No IPv4 address
No default route
```

---

## Root Cause

SERVER1 was initially using DHCP or had no persistent Netplan static configuration.

---

## Recovery

Overwrite Netplan file:

```bash
sudo tee /etc/netplan/50-cloud-init.yaml > /dev/null << 'EOF'
network:
  version: 2
  ethernets:
    ens2:
      dhcp4: no
      addresses:
        - 10.10.70.10/24
      routes:
        - to: default
          via: 10.10.70.254
      nameservers:
        addresses:
          - 8.8.8.8
          - 1.1.1.1
EOF
```

Apply:

```bash
sudo netplan generate
sudo netplan apply
```

---

## Verification

```bash
ip addr show ens2
ip route
ping 10.10.70.254
ping 10.10.40.10
ping 203.0.113.10
```

Expected:

```text
SERVER1 can reach gateway, MGMT-SRV1, and INTERNET-SRV
```

---

# Scenario 18 — MGMT-SRV1 IP/Gateway Missing

## Symptoms

MGMT-SRV1 cannot reach its gateway, SERVER1, or INTERNET-SRV.

---

## Verification

MGMT-SRV1:

```bash
ip addr
ip route
```

Problem state:

```text
ens2 is UP
No IPv4 address
No default route
```

---

## Root Cause

MGMT-SRV1 was initially using DHCP or had no persistent Netplan static configuration.

---

## Recovery

Overwrite Netplan file:

```bash
sudo tee /etc/netplan/50-cloud-init.yaml > /dev/null << 'EOF'
network:
  version: 2
  ethernets:
    ens2:
      dhcp4: no
      addresses:
        - 10.10.40.10/24
      routes:
        - to: default
          via: 10.10.40.254
      nameservers:
        addresses:
          - 8.8.8.8
          - 1.1.1.1
EOF
```

Apply:

```bash
sudo netplan generate
sudo netplan apply
```

---

## Verification

```bash
ip addr show ens2
ip route
ping 10.10.40.254
ping 10.10.70.10
ping 203.0.113.10
```

Expected:

```text
MGMT-SRV1 can reach gateway, SERVER1, and INTERNET-SRV
```

---

# Scenario 19 — Ubuntu Netplan Editing Difficulty

## Symptoms

Editing `/etc/netplan/50-cloud-init.yaml` with `vi` is confusing or difficult.

Common issues:

* Difficulty deleting existing lines
* Insert mode confusion
* Save and quit errors
* File appears unchanged after edit

---

## Root Cause

The issue is operational, not networking-related.

Manual editing with `vi` can be error-prone during lab work.

---

## Recovery

Use `tee` to overwrite the file directly.

Example:

```bash
sudo tee /etc/netplan/50-cloud-init.yaml > /dev/null << 'EOF'
network:
  version: 2
  ethernets:
    ens2:
      dhcp4: no
      addresses:
        - <IP>/<PREFIX>
      routes:
        - to: default
          via: <GATEWAY>
EOF
```

Then apply:

```bash
sudo netplan generate
sudo netplan apply
```

---

## Verification

```bash
sudo cat /etc/netplan/50-cloud-init.yaml
ip addr
ip route
```

---

# Scenario 20 — DNS Resolver Not Tested in PHASE 01

## Symptoms

Public DNS testing such as:

```bash
ping google.com
```

is not performed in PHASE 01.

---

## Root Cause

PHASE 01 uses INTERNET-SRV as an Internet Simulation target.

Actual public DNS testing requires External Connector reachability to real DNS servers such as:

```text
8.8.8.8
1.1.1.1
```

External Connector is not required for PHASE 01 completion.

---

## Recovery / Design Decision

DNS Resolver testing is deferred to PHASE 02.

PHASE 02 will include:

```text
External Connector
apt update
DNS Resolver
DNS verification
Enterprise Services
```

---

# Lessons Learned

PHASE 01 troubleshooting focused on:

* VLAN database issues
* Access VLAN mistakes
* Trunk encapsulation requirements
* Native VLAN mismatches
* STP root placement
* SVI availability
* HSRP ownership
* Endpoint addressing
* Linux Desktop operational differences
* Routed link addressing
* Static route forward-path and return-path behavior
* Source IP behavior during ping tests
* Internet Simulation reachability
* Tiny Core Linux persistence
* Ubuntu Netplan persistence
* DNS Resolver phase boundary

These troubleshooting exercises establish the foundation required for Enterprise Services, Routing Resiliency, Security, QoS, and SD-WAN phases.
