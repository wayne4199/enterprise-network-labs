# PHASE 04 — Security Verification

---

# SECTION 1 — Rogue DHCP Attack Baseline

## Verify Rogue DHCP Server Operation

ATTACKER-PC:

```bash
sudo dnsmasq -d
```

Expected result:

```text
DHCPOFFER(eth0)
DHCPDISCOVER(eth0)
```

---

## Verify Legitimate DHCP Server Still Wins

USER1:

```bash
sudo udhcpc -i eth0
```

Expected result:

```text
lease of 10.10.10.x obtained from 10.20.20.31
```

---

# SECTION 2 — DHCP Snooping

## Verify DHCP Snooping Status

ACC1 / ACC2:

```cisco
show ip dhcp snooping
```

Expected result:

```text
Switch DHCP snooping is enabled
DHCP snooping is configured on following VLANs
```

---

## Verify Trusted Interfaces

```cisco
show run interface g0/0
show run interface g0/1
```

Expected result:

```text
ip dhcp snooping trust
```

---

## Verify DHCP Snooping Statistics

```cisco
show ip dhcp snooping statistics
```

Expected result:

```text
Packets Dropped
```

counter increments when rogue DHCP traffic appears.

---

## Verify DHCP Snooping Binding

```cisco
show ip dhcp snooping binding
```

Expected result:

```text
MAC Address
IP Address
Lease(sec)
Type
Interface
```

binding entries exist.

---

# SECTION 3 — DHCP Snooping Rate Limit

## Verify Rate Limit Configuration

```cisco
show ip dhcp snooping
```

Expected result:

```text
Gi0/3 rate limit 5
```

---

## Verify DHCP Flood Protection

ATTACKER-PC:

```bash
sudo pkill udhcpc

while true; do
 sudo udhcpc -n -q -i eth0
done
```

---

## Verify Err-Disable State

```cisco
show interface status err-disabled
```

Expected result:

```text
Reason: dhcp-rate-limit
```

---

## Verify Interface Recovery

```cisco
show interface status err-disabled
```

Expected result:

```text
No err-disabled interfaces
```

---

# SECTION 4 — Port Security

## Verify Port Security Status

```cisco
show port-security interface g0/3
```

Expected result:

```text
Port Security              : Enabled
Port Status                : Secure-up
Violation Mode             : Shutdown
Maximum MAC Addresses      : 1
```

---

## Verify Sticky MAC Learning

```cisco
show port-security address
```

Expected result:

```text
SecureSticky
```

MAC entry appears.

---

## Verify Security Violation

After changing ATTACKER-PC MAC:

```bash
ping 10.10.10.254
```

---

## Verify Err-Disable

```cisco
show interface status err-disabled
```

Expected result:

```text
Reason: psecure-violation
```

---

## Verify Recovery

```cisco
show interface status err-disabled
```

Expected result:

```text
No err-disabled interfaces
```

---

# SECTION 5 — Dynamic ARP Inspection (DAI)

## Verify DAI Status

```cisco
show ip arp inspection
show ip arp inspection vlan 10
```

Expected result:

```text
Vlan     Configuration    Operation   ACL Match   Static ACL
10       Enabled          Active
```

---

## Verify DHCP Snooping Binding Dependency

```cisco
show ip dhcp snooping binding
```

Expected result:

```text
Binding entry exists for ATTACKER-PC
```

---

## Verify ARP Spoofing Protection

ATTACKER-PC:

```bash
sudo arping -I eth0 -S 10.10.10.254 10.10.10.1
```

Expected result:

```text
Timeout
```

---

## Verify DAI Drop Counters

```cisco
show ip arp inspection statistics
```

Expected result:

```text
Dropped packets increment
```

---

# SECTION 6 — DAI Validation

## Verify Validation Status

```cisco
show ip arp inspection
show ip arp inspection vlan 10
```

Expected result:

```text
Source MAC Validation      : Enabled
Destination MAC Validation : Enabled
IP Address Validation      : Enabled
```

---

## Verify Connectivity Recovery

ATTACKER-PC:

```bash
sudo pkill udhcpc
sudo ip addr flush dev eth0
sudo udhcpc -i eth0
```

---

## Verify DHCP Recovery

```bash
ip addr
ip route
ping 10.10.10.254
```

Expected result:

```text
DHCP address restored
Default route restored
Successful ping replies
```

---

# SECTION 7 — IP Source Guard (IPSG)

## Verify IPSG Status

```cisco
show ip verify source
show ip verify source interface g0/3
```

Expected result:

```text
Filter-type: ip
Filter-mode: active
```

---

## Verify DHCP Snooping Binding

```cisco
show ip dhcp snooping binding
```

Expected result:

```text
Binding entry exists for ATTACKER-PC
```

---

## Verify IPSG Removal

```cisco
show run interface g0/3
```

Expected result:

```text
No ip verify source configuration present
```

---

# SECTION 8 — BPDU Guard / Rogue Switch Protection

## Verify BPDU Guard Status

```cisco
show spanning-tree interface g0/3 detail
```

Expected result:

```text
PortFast enabled
BPDU Guard enabled
```

---

## Verify Rogue Switch Protection

Connect ROGUE-SW1 to ACC3 G0/3.

---

## Verify Err-Disable

```cisco
show interface status err-disabled
```

Expected result:

```text
Reason: bpduguard
```

---

## Verify Interface Recovery

```cisco
show interface status err-disabled
```

Expected result:

```text
No err-disabled interfaces
```

---

# LAB BACKUP VERIFICATION

## Verify Device Configurations Saved

On all Cisco devices:

```cisco
show startup-config
```

Expected result:

```text
Latest Phase-04 security configuration exists
```

---

## Verify Config Extraction

CML GUI:

```text
NODES
→ Select All Nodes
→ Extract Configs
```

Expected result:

```text
Configs successfully extracted into topology database
```

---

## Verify Exported Lab

CML GUI:

```text
LAB
→ Download Lab
```

Expected result:

```text
Lab YAML file successfully downloaded
```

---

## Verify Re-Import Persistence

Re-import exported lab.

Verify:
- VLAN configuration
- DHCP Snooping
- DAI
- IPSG
- Port Security
- BPDU Guard
- interface configurations
- startup-config persistence

Expected result:

```text
All Phase-04 security configurations preserved after import
```
