# PHASE 04 — Security

---

# SECTION 1 — Rogue DHCP Attack Baseline

## Objective

Validate that a rogue endpoint can attempt to distribute malicious DHCP information such as:
- Fake Default Gateway
- Fake DNS Server
- Rogue DHCP Offer

---

## Topology

ATTACKER-PC <--> ACC2 G0/3  
USER1       <--> ACC1  
MGMT-SRV1   = Legitimate DHCP Server

---

## ATTACKER-PC Rogue DHCP Preparation

### Install dnsmasq

```bash
sudo apk update
sudo apk add dnsmasq
```

### Configure Rogue DHCP Server

```bash
sudo vi /etc/dnsmasq.conf
```

Append:

```ini
interface=eth0

dhcp-range=10.10.10.200,10.10.10.220,255.255.255.0,12h

dhcp-option=3,10.10.10.1
dhcp-option=6,6.6.6.6

log-dhcp
```

### Start Rogue DHCP Server

```bash
sudo dnsmasq -d
```

---

## USER1 DHCP Test

```bash
sudo udhcpc -i eth0
```

Expected result:

```text
lease of 10.10.10.x obtained from 10.20.20.31
```

---

# SECTION 2 — DHCP Snooping

## Objective

Protect the campus access layer against Rogue DHCP attacks.

---

## ACC1 Configuration

```cisco
conf t

ip dhcp snooping
ip dhcp snooping vlan 10,20,30,40,50
no ip dhcp snooping information option

interface g0/0
 ip dhcp snooping trust

interface g0/1
 ip dhcp snooping trust

end
```

---

## ACC2 Configuration

```cisco
conf t

ip dhcp snooping
ip dhcp snooping vlan 10
no ip dhcp snooping information option

interface g0/0
 ip dhcp snooping trust

interface g0/1
 ip dhcp snooping trust

end
```

---

## Verification

```cisco
show ip dhcp snooping
show ip dhcp snooping statistics
show ip dhcp snooping binding
```

---

## DHCP Client Verification

```bash
sudo udhcpc -i eth0
```

Expected result:

```text
lease of 10.10.10.x obtained from 10.20.20.31
```

---

# SECTION 3 — DHCP Snooping Rate Limit

## Objective

Protect against DHCP starvation/flood attacks.

---

## Configure DHCP Rate Limit

```cisco
conf t

interface g0/3
 ip dhcp snooping limit rate 5

end
```

---

## Verification

```cisco
show ip dhcp snooping
```

Expected result:

```text
Gi0/3 rate limit 5
```

---

## DHCP Flood Test

```bash
sudo pkill udhcpc

while true; do
 sudo udhcpc -n -q -i eth0
done
```

---

## Verification

```cisco
show ip dhcp snooping statistics
show interface status err-disabled
```

Expected result:

```text
Reason: dhcp-rate-limit
```

---

## Recovery Procedure

```cisco
conf t

interface g0/3
 shutdown
 no shutdown

end
```

---

## Verification

```cisco
show interface status err-disabled
```

Expected result:

```text
No err-disabled interfaces
```

---

# SECTION 4 — Port Security

## Objective

Protect access ports against unauthorized endpoints and MAC spoofing.

---

## Configure Port Security

```cisco
conf t

interface g0/3
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation shutdown

end
```

---

## Verification

```cisco
show port-security interface g0/3
show port-security address
```

---

## MAC Spoofing Test

### Verify Original MAC

```bash
ip link show eth0
```

### Change MAC Address

```bash
sudo ip link set eth0 down
sudo ip link set dev eth0 address 52:54:00:AA:BB:CC
sudo ip link set eth0 up
```

---

## Trigger Violation

```bash
ping 10.10.10.254
```

---

## Verification

```cisco
show interface status err-disabled
show port-security interface g0/3
```

Expected result:

```text
Reason: psecure-violation
```

---

## Recovery Procedure

```cisco
conf t

interface g0/3
 shutdown
 no switchport port-security
 no switchport port-security mac-address sticky
 no shutdown

end
```

### Restore Original MAC

```bash
sudo ip link set eth0 down
sudo ip link set dev eth0 address 52:54:00:6b:3e:46
sudo ip link set eth0 up
```

---

# SECTION 5 — Dynamic ARP Inspection (DAI)

## Objective

Protect against ARP spoofing and man-in-the-middle attacks.

---

## Enable DAI

```cisco
conf t

ip arp inspection vlan 10

end
```

---

## Verification

```cisco
show ip arp inspection
show ip arp inspection vlan 10
show ip arp inspection statistics
```

Expected result:

```text
VLAN 10 Enabled / Active
```

---

## DHCP Snooping Binding Verification

```cisco
show ip dhcp snooping binding
```

Expected result:

```text
Binding entry exists for ATTACKER-PC
```

---

## Install arping

```bash
sudo apk add arping
```

---

## ARP Spoofing Test

```bash
sudo arping -I eth0 -S 10.10.10.254 10.10.10.1
```

---

## Verification

```cisco
show ip arp inspection statistics
```

Expected result:

```text
Dropped packets increment
```

---

# SECTION 6 — DAI Validation

## Objective

Enable additional ARP packet validation checks.

---

## Configure Validation

```cisco
conf t

ip arp inspection validate src-mac dst-mac ip

end
```

---

## Verification

```cisco
show ip arp inspection
show ip arp inspection vlan 10
```

Expected result:

```text
Source MAC Validation : Enabled
Destination MAC Validation : Enabled
IP Address Validation : Enabled
```

---

## Recovery Procedure

Some Linux/CML combinations may temporarily lose DHCP lease or routing information after DAI validation testing.

If connectivity fails:

```bash
sudo pkill udhcpc
sudo ip addr flush dev eth0
sudo udhcpc -i eth0
```

---

## Verification

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

## Objective

Protect against IP spoofing attacks using DHCP Snooping binding information.

---

## Configure IPSG

```cisco
conf t

interface g0/3
 ip verify source

end
```

---

## Verification

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

## DHCP Snooping Binding Verification

```cisco
show ip dhcp snooping binding
```

Expected result:

```text
Binding entry exists for ATTACKER-PC
```

---

## Recovery Procedure

```cisco
conf t

interface g0/3
 no ip verify source

end
```

---

## Verification

```cisco
show run interface g0/3
```

Expected result:

```text
No ip verify source configuration present
```

---

# SECTION 8 — BPDU Guard / Rogue Switch Protection

## Objective

Protect access ports against unauthorized switches and STP manipulation.

---

## Configure BPDU Guard

```cisco
conf t

interface g0/3
 spanning-tree portfast edge
 spanning-tree bpduguard enable

end
```

---

## Verification

```cisco
show spanning-tree interface g0/3 detail
```

Expected result:

```text
PortFast enabled
BPDU Guard enabled
```

---

## Rogue Switch Detection Test

Connect ROGUE-SW1 to ACC3 G0/3.

Expected result:

```text
Interface enters err-disabled state
```

---

## Verification

```cisco
show interface status err-disabled
```

Expected result:

```text
Reason: bpduguard
```

---

## Recovery Procedure

```cisco
conf t

interface g0/3
 shutdown
 no shutdown

end
```

---

## Verification

```cisco
show interface status err-disabled
```

Expected result:

```text
No err-disabled interfaces
```

---

# LAB BACKUP PROCEDURE

## Save Device Configurations

```cisco
wr
```

---

## Extract Configurations into CML Topology

NODES  
→ Select All Nodes  
→ Extract Configs

---

## Export Lab

LAB  
→ Download Lab

---

## Verification Procedure

Re-import the exported lab and verify:
- topology
- VLANs
- routing
- security configurations
- startup-config persistence

before considering the backup complete.
