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
* Routed links are operational
* Static routing is operational
* Internet Simulation reachability is successful
* SERVER1, MGMT-SRV1, and INTERNET-SRV have persistent IP/Gateway configuration
* CML Extract Configs was applied correctly for Cisco IOSv / IOSvL2 nodes
* Linux-based node settings are verified after export/import or rebuild
* End-to-end communication is successful

---

# Verification 1 — Hostname

## Command

```cisco
show running-config | include hostname
```

## Expected Result

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

# Verification 2 — Base Device Configuration

## Command

```cisco
show running-config | include no ip domain-lookup
show running-config | include no logging console
```

## Expected Result

```text
no ip domain-lookup
no logging console
```

---

# Verification 3 — VLAN Database

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

# Verification 4 — Access Port Assignment

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

# Verification 5 — Trunk Status

## Command

```cisco
show interfaces trunk
```

## Expected Result

All switch-to-switch links should appear as trunks.

Native VLAN:

```text
999
```

Allowed VLANs:

```text
10,11,20,21,30,31,40,50,60,70,999
```

Expected trunk links:

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

# Verification 6 — Rapid-PVST

## Command

```cisco
show spanning-tree summary
```

## Expected Result

```text
Switch is in rapid-pvst mode
```

for all Layer 2 switches:

* CORE1
* CORE2
* ACC1
* ACC2
* ACC3
* SRV-ACC1

---

# Verification 7 — STP Root Alignment

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

# Verification 8 — SVI Status

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

# Verification 9 — HSRP Status

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

Expected VIP format:

```text
10.10.X.254
```

Examples:

```text
VLAN10 VIP = 10.10.10.254
VLAN70 VIP = 10.10.70.254
```

---

# Verification 10 — Linux Desktop Addressing

Linux Desktop endpoint nodes use temporary IP/Gateway settings during PHASE 01.

## Command

```bash
ip addr
```

## Expected Result

Example for EXEC:

```text
inet 10.10.10.10/24
```

---

## Command

```bash
ip route
```

## Expected Result

Example for EXEC:

```text
default via 10.10.10.254
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

## MGMT-SRV1

```bash
ping 10.10.40.254
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

# Verification 13 — Routed Link Interface Status

## EDGE1

```cisco
show ip interface brief
```

Expected:

```text
Gi0/0  100.64.1.2   up/up
Gi0/1  100.64.2.2   up/up
Gi0/2  172.16.0.1   up/up
Gi0/3  172.16.0.5   up/up
```

---

## CORE1

```cisco
show ip interface brief
```

Expected:

```text
Gi0/0  172.16.0.2   up/up
```

---

## CORE2

```cisco
show ip interface brief
```

Expected:

```text
Gi0/0  172.16.0.6   up/up
```

---

## ISP1

```cisco
show ip interface brief
```

Expected:

```text
Gi0/0  203.0.113.1  up/up
Gi0/1  100.64.1.1   up/up
```

---

## ISP2

```cisco
show ip interface brief
```

Expected:

```text
Gi0/0  203.0.113.2  up/up
Gi0/1  100.64.2.1   up/up
```

---

# Verification 14 — Routed Link Reachability

## CORE1 → EDGE1

```cisco
ping 172.16.0.1
```

Expected:

```text
Success
```

---

## CORE2 → EDGE1

```cisco
ping 172.16.0.5
```

Expected:

```text
Success
```

---

## EDGE1 → ISP1

```cisco
ping 100.64.1.1
```

Expected:

```text
Success
```

---

## EDGE1 → ISP2

```cisco
ping 100.64.2.1
```

Expected:

```text
Success
```

---

## ISP1 → INTERNET-SRV Segment

```cisco
ping 203.0.113.10
```

Expected:

```text
Success
```

---

## ISP2 → INTERNET-SRV Segment

```cisco
ping 203.0.113.10
```

Expected:

```text
Success
```

---

# Verification 15 — Static Route Verification

## CORE1

```cisco
show ip route static
show ip route 0.0.0.0
```

Expected:

```text
S* 0.0.0.0/0 via 172.16.0.1
```

---

## CORE2

```cisco
show ip route static
show ip route 0.0.0.0
```

Expected:

```text
S* 0.0.0.0/0 via 172.16.0.5
```

---

## EDGE1

```cisco
show ip route static
show ip route 10.10.0.0
show ip route 0.0.0.0
```

Expected:

```text
S  10.10.0.0/16 via 172.16.0.2
S* 0.0.0.0/0 via 100.64.1.1
```

Backup routes may also exist with higher administrative distance.

---

## ISP1

```cisco
show ip route static
show ip route 10.10.0.0
show ip route 172.16.0.0
```

Expected:

```text
S 10.10.0.0/16 via 100.64.1.2
S 172.16.0.0/30 via 100.64.1.2
S 172.16.0.4/30 via 100.64.1.2
```

---

## ISP2

```cisco
show ip route static
show ip route 10.10.0.0
show ip route 172.16.0.0
```

Expected:

```text
S 10.10.0.0/16 via 100.64.2.2
S 172.16.0.0/30 via 100.64.2.2
S 172.16.0.4/30 via 100.64.2.2
```

---

# Verification 16 — Internet Simulation Reachability

## ISP1 → INTERNET-SRV

```cisco
ping 203.0.113.10
```

Expected:

```text
Success
```

---

## ISP2 → INTERNET-SRV

```cisco
ping 203.0.113.10
```

Expected:

```text
Success
```

---

## EDGE1 → INTERNET-SRV

```cisco
ping 203.0.113.10
```

Expected:

```text
Success
```

---

## CORE1 → INTERNET-SRV

```cisco
ping 203.0.113.10
```

Expected:

```text
Success
```

---

## CORE2 → INTERNET-SRV

```cisco
ping 203.0.113.10
```

Expected:

```text
Success
```

---

## EXEC → INTERNET-SRV

```bash
ping 203.0.113.10
```

Expected:

```text
Success
```

This confirms complete end-to-end reachability:

```text
EXEC
→ HSRP VIP
→ CORE
→ EDGE1
→ ISP
→ INTERNET-SRV
```

---

# Verification 17 — SERVER1 Persistent IP/Gateway

## Check Netplan File

```bash
sudo cat /etc/netplan/50-cloud-init.yaml
```

Expected:

```yaml
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
```

---

## Check IP Address

```bash
ip addr show ens2
```

Expected:

```text
inet 10.10.70.10/24
```

---

## Check Default Route

```bash
ip route
```

Expected:

```text
default via 10.10.70.254
```

---

## Connectivity Test

```bash
ping 10.10.70.254
ping 10.10.40.10
ping 203.0.113.10
```

Expected:

```text
Success
```

---

# Verification 18 — MGMT-SRV1 Persistent IP/Gateway

## Check Netplan File

```bash
sudo cat /etc/netplan/50-cloud-init.yaml
```

Expected:

```yaml
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
```

---

## Check IP Address

```bash
ip addr show ens2
```

Expected:

```text
inet 10.10.40.10/24
```

---

## Check Default Route

```bash
ip route
```

Expected:

```text
default via 10.10.40.254
```

---

## Connectivity Test

```bash
ping 10.10.40.254
ping 10.10.70.10
ping 203.0.113.10
```

Expected:

```text
Success
```

---

# Verification 19 — INTERNET-SRV Persistent IP/Gateway

## Check bootlocal.sh

```bash
cat /opt/bootlocal.sh
```

Expected lines:

```bash
ifconfig eth0 203.0.113.10 netmask 255.255.255.0 up
route add default gw 203.0.113.1
```

---

## Check Tiny Core Backup

```bash
filetool.sh -b
```

Expected:

```text
Backing up files to /mnt/vda1/tce/mydata.tgz Done.
```

---

## Check IP Address

```bash
ifconfig eth0
```

Expected:

```text
203.0.113.10
```

---

## Check Default Route

```bash
route -n
```

Expected:

```text
0.0.0.0         203.0.113.1
```

---

## Connectivity Test

```bash
ping 203.0.113.1
```

Expected:

```text
Success
```

---

# Verification 20 — Cisco Config Preservation Before CML Export

Before exporting or downloading the CML lab, all Cisco IOSv / IOSvL2 nodes must save their running configuration.

## Target Nodes

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

## Command

Run on each Cisco node:

```cisco
write memory
```

or:

```cisco
copy running-config startup-config
```

## Expected Result

```text
[OK]
```

or:

```text
Building configuration...
[OK]
```

---

# Verification 21 — CML Extract Configs Result

CML `EXTRACT CONFIGS` should be used for Cisco network devices only.

## Procedure

In CML Workbench:

```text
Select Cisco IOSv / IOSvL2 nodes only
→ EXTRACT CONFIGS
```

## Expected Result

Cisco nodes should retain their startup configuration after lab export/import.

Expected node types:

```text
IOSv
IOSvL2
```

Expected Cisco nodes:

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

---

# Verification 22 — Linux-Based Node Verification After Export / Import

CML `EXTRACT CONFIGS` does not fully preserve Linux OS-level settings.

After exporting, downloading, importing, or rebuilding the lab, verify the Linux-based nodes manually.

## Target Nodes

```text
SERVER1
MGMT-SRV1
INTERNET-SRV
```

---

## SERVER1

```bash
ip addr show ens2
ip route
sudo cat /etc/netplan/50-cloud-init.yaml
ping 10.10.70.254
ping 10.10.40.10
ping 203.0.113.10
```

Expected:

```text
10.10.70.10/24
default via 10.10.70.254
Gateway reachable
MGMT-SRV1 reachable
INTERNET-SRV reachable
```

---

## MGMT-SRV1

```bash
ip addr show ens2
ip route
sudo cat /etc/netplan/50-cloud-init.yaml
ping 10.10.40.254
ping 10.10.70.10
ping 203.0.113.10
```

Expected:

```text
10.10.40.10/24
default via 10.10.40.254
Gateway reachable
SERVER1 reachable
INTERNET-SRV reachable
```

---

## INTERNET-SRV

```bash
ifconfig eth0
route -n
cat /opt/bootlocal.sh
ping 203.0.113.1
```

Expected:

```text
203.0.113.10/24
default via 203.0.113.1
Gateway reachable
```

---

# Verification 23 — Linux Desktop Endpoint Reverification

Linux Desktop endpoint addressing is temporary.

After lab restart, export/import, or rebuild, reapply endpoint IP/Gateway only when testing is required.

## EXEC Example

```bash
sudo ip addr add 10.10.10.10/24 dev eth0
sudo ip route add default via 10.10.10.254
```

## Verification

```bash
ip addr
ip route
ping 10.10.10.254
ping 203.0.113.10
```

Expected:

```text
10.10.10.10/24
default via 10.10.10.254
Gateway reachable
INTERNET-SRV reachable
```

---

# Verification 24 — DNS Resolver Phase Boundary

DNS Resolver verification is intentionally not performed in PHASE 01.

The following tests are deferred to PHASE 02:

```bash
ping 8.8.8.8
ping google.com
apt update
```

Reason:

```text
External Connector is required for real Internet access and public DNS testing.
```

PHASE 01 uses INTERNET-SRV as the Internet Simulation target.

---

# Verification 25 — Final End-to-End Validation

The following must be confirmed:

* VLAN forwarding works
* Access VLAN assignment is correct
* Trunk forwarding works
* Native VLAN 999 is configured
* Rapid-PVST is enabled
* STP Root placement is correct
* HSRP is operational
* HSRP VIP is reachable
* Inter-VLAN routing is operational
* Routed links are up/up
* Static routes are installed
* Internet Simulation reachability works
* SERVER1 persistent IP/Gateway works
* MGMT-SRV1 persistent IP/Gateway works
* INTERNET-SRV persistent IP/Gateway works
* Cisco IOSv / IOSvL2 configurations are saved before export
* CML Extract Configs is applied to Cisco nodes
* Linux-based node settings are verified after export/import
* EXEC can reach INTERNET-SRV

---

# PHASE 01 Completion Criteria

PHASE 01 is considered complete when all of the following are true:

* VLAN verification passes
* Access port verification passes
* Trunk verification passes
* Rapid-PVST verification passes
* STP Root verification passes
* SVI verification passes
* HSRP verification passes
* Gateway reachability passes
* Inter-VLAN connectivity passes
* Routed link verification passes
* Static route verification passes
* Internet Simulation reachability passes
* Persistent server IP/Gateway verification passes
* Cisco IOSv / IOSvL2 configurations are saved with `write memory`
* CML Extract Configs is completed for Cisco nodes
* Linux-based node recovery procedures are documented and verified
* DNS Resolver is intentionally deferred to PHASE 02

Successful completion establishes the baseline campus infrastructure for PHASE 02 Enterprise Services and all later phases.
