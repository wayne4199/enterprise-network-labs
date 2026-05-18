# Phase 02 — Enterprise Services Troubleshooting

This document summarizes major troubleshooting scenarios encountered during the Phase 02 Enterprise Services lab.

---

# 1. Ubuntu DNS Resolution Failure

## Problem

MGMT-SRV1 could not access Ubuntu repositories.

```bash
sudo apt update
```

Error message:

```bash
Temporary failure resolving 'archive.ubuntu.com'
```

---

## Root Cause

The Ubuntu management server did not have:

- Proper DNS resolver configuration
- Valid IP addressing
- Default gateway routing

The node initially booted without persistent network configuration.

---

## Verification

### DNS Resolution Failure

```bash
ping archive.ubuntu.com
```

Result:

```bash
Temporary failure in name resolution
```

### No Default Route

```bash
ip route
```

Result:

```bash
(no default route present)
```

### Interface Without IPv4 Address

```bash
ip a
```

Result:

```bash
ens2 had no IPv4 address configured
```

---

## Solution

### Configure Management IP Address

```bash
sudo ip addr add 10.20.20.31/24 dev ens2
```

### Bring Interface Up

```bash
sudo ip link set ens2 up
```

### Configure Default Gateway

```bash
sudo ip route add default via 10.20.20.254 dev ens2
```

### Configure DNS Resolvers

```bash
sudo nano /etc/resolv.conf
```

Configured:

```bash
nameserver 8.8.8.8
nameserver 1.1.1.1
```

---

## Verification After Fix

### Verify Routing Table

```bash
ip route
```

Expected:

```bash
default via 10.20.20.254 dev ens2
```

### Verify Internet Reachability

```bash
ping 8.8.8.8
```

### Verify DNS Resolution

```bash
ping archive.ubuntu.com
```

### Verify Package Repository Access

```bash
sudo apt update
```

Successful package downloads confirmed connectivity.

---

# 2. SNMP Walk Object Identifier Error

## Problem

Initial SNMP verification failed.

Command:

```bash
snmpwalk -v2c -c NMS-RO 10.255.255.1 system
```

Error:

```bash
Unknown Object Identifier
```

---

## Root Cause

The Ubuntu SNMP package did not include symbolic MIB translation support for textual object names.

The keyword:

```bash
system
```

could not be resolved into an SNMP OID.

---

## Solution

Use the numeric OID directly.

```bash
snmpwalk -v2c -c NMS-RO 10.255.255.1 1.3.6.1.2.1.1
```

---

## Verification

Successful SNMP responses returned:

- IOS version
- System uptime
- Contact information
- Hostname
- Location

Example:

```bash
iso.3.6.1.2.1.1.5.0 = STRING: "EDGE1.lab.local"
```

---

# 3. SNMP Package Installation Delay

## Problem

During installation of SNMP packages, the terminal appeared frozen.

Command:

```bash
sudo apt install snmp snmp-mibs-downloader -y
```

The console stopped responding during:

```bash
Scanning processes...
```

---

## Root Cause

Ubuntu was processing package triggers and background updates.

This behavior is common in low-resource virtual environments.

---

## Solution

Wait for package processing to complete.

The installation eventually finished successfully.

---

## Verification

```bash
snmpwalk --version
```

Expected:

```bash
NET-SNMP version: 5.9.4.pre2
```

---

# 4. Missing VLAN Visibility on Trunk Ports

## Problem

Some VLANs did not appear under:

```bash
show vlan brief
```

Expected VLAN assignments were not visible on trunk interfaces.

---

## Root Cause

`show vlan brief` only displays access VLAN membership.

Trunk ports are not listed under VLAN membership tables.

---

## Verification

### Incorrect Assumption

```bash
show vlan brief
```

Did not display trunk membership.

### Correct Verification Command

```bash
show interfaces trunk
```

This confirmed:

- 802.1Q trunking
- Native VLAN 999
- Allowed VLANs
- Active VLAN forwarding

---

# 5. SRV-ACC1 VLAN40 Interface Down

## Problem

The management SVI on SRV-ACC1 showed:

```bash
Vlan40 down/down
```

---

## Root Cause

No active port in VLAN40 existed on the switch.

SVI status depends on active Layer 2 VLAN membership.

---

## Verification

```bash
show ip interface brief
```

Result:

```bash
Vlan40 down/down
```

---

## Solution

Assign an active access port into VLAN40.

Example:

```bash
interface GigabitEthernet0/2
 switchport mode access
 switchport access vlan 40
 no shutdown
```

---

## Verification After Fix

```bash
show ip interface brief
```

Expected:

```bash
Vlan40 up/up
```

---

# 6. Invalid ACL Command Syntax

## Problem

Incorrect ACL syntax caused configuration errors.

Incorrect command:

```bash
ip acc standard SNMP-MGMT
```

---

## Root Cause

Typographical error.

Correct IOS syntax uses:

```bash
ip access-list
```

---

## Solution

Corrected command:

```bash
ip access-list standard SNMP-MGMT
 permit 10.20.20.0 0.0.0.255
```

---

# 7. Enterprise Services Validation Summary

The following enterprise services were successfully validated:

- SSH remote management
- Centralized Syslog logging
- NTP synchronization
- SNMP monitoring
- DHCP relay forwarding
- Linux-based DHCP services
- Linux-based DNS services
- VLAN-based management segmentation
- Trunk-based Layer 2 transport

---

# 8. Operational Lessons Learned

## Key Lessons

- Linux management services require proper Layer 3 reachability
- DNS failures are often caused by routing problems
- SNMP verification is easier with numeric OIDs
- Trunk VLANs must be verified using dedicated trunk commands
- SVI operational state depends on active VLAN ports
- Enterprise management services should be centralized
- Ubuntu package installations may appear stalled in virtual labs

---
