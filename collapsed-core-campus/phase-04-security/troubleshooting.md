# PHASE 04 — Security Troubleshooting

---

# 1. DHCP Client Unable to Receive IP Address

## Symptoms

```text
udhcpc: no lease, failing
```

---

## Possible Causes

- DHCP Snooping blocking DHCP packets
- Rogue dnsmasq still running on ATTACKER-PC
- DHCP Snooping trust missing on uplink
- Option 82 insertion issue
- DHCP Snooping binding missing
- Interface err-disabled state

---

## Verification

```cisco
show ip dhcp snooping
show ip dhcp snooping statistics
show ip dhcp snooping binding
show interface status err-disabled
```

```bash
ps aux | grep dnsmasq
```

---

## Resolution

### Stop Rogue DHCP Server

```bash
sudo pkill dnsmasq
```

---

### Disable DHCP Option 82 Insertion

```cisco
conf t

no ip dhcp snooping information option

end
```

---

### Verify Trusted Interfaces

```cisco
show run interface g0/0
show run interface g0/1
```

Expected result:

```text
ip dhcp snooping trust
```

---

### Recover Err-Disabled Interface

```cisco
conf t

interface g0/3
 shutdown
 no shutdown

end
```

---

# 2. DHCP Snooping Rate Limit Err-Disable

## Symptoms

```text
Interface enters err-disabled state
Reason: dhcp-rate-limit
```

---

## Cause

Excessive DHCP Discover packets exceeded configured rate limit.

---

## Verification

```cisco
show ip dhcp snooping statistics
show interface status err-disabled
```

---

## Resolution

```cisco
conf t

interface g0/3
 shutdown
 no shutdown

end
```

---

# 3. Port Security Violation

## Symptoms

```text
Interface enters err-disabled state
Reason: psecure-violation
```

---

## Cause

A different MAC address appeared after sticky MAC learning.

---

## Verification

```cisco
show port-security interface g0/3
show port-security address
show interface status err-disabled
```

---

## Resolution

```cisco
conf t

interface g0/3
 shutdown
 no switchport port-security
 no switchport port-security mac-address sticky
 no shutdown

end
```

---

## Restore Original MAC Address

```bash
sudo ip link set eth0 down
sudo ip link set dev eth0 address 52:54:00:6b:3e:46
sudo ip link set eth0 up
```

---

# 4. DAI Blocking ARP Traffic

## Symptoms

```text
ping: sendto: Network unreachable
```

or:

```text
ARP requests fail
```

---

## Possible Causes

- Missing DHCP Snooping binding
- DAI validation enabled
- Linux/CML ARP frame compatibility issue
- DHCP lease temporarily lost

---

## Verification

```cisco
show ip arp inspection
show ip arp inspection vlan 10
show ip arp inspection statistics
show ip dhcp snooping binding
```

---

## Resolution

### Disable Validation

```cisco
conf t

no ip arp inspection validate src-mac dst-mac ip

end
```

---

### Renew DHCP Lease

```bash
sudo pkill udhcpc
sudo ip addr flush dev eth0
sudo udhcpc -i eth0
```

---

### Verification

```bash
ip addr
ip route
ping 10.10.10.254
```

---

# 5. IP Source Guard Blocking Traffic

## Symptoms

```text
Ping traffic fails after enabling IPSG
```

---

## Possible Causes

- DHCP Snooping binding mismatch
- DHCP lease renewal issue
- IOSvL2/CML IPSG behavior limitation

---

## Verification

```cisco
show ip verify source
show ip verify source interface g0/3
show ip dhcp snooping binding
```

---

## Resolution

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

# 6. BPDU Guard Err-Disable

## Symptoms

```text
Interface enters err-disabled state
Reason: bpduguard
```

---

## Cause

A switch or BPDU-generating device was connected to a PortFast-enabled access port.

---

## Verification

```cisco
show interface status err-disabled
show spanning-tree interface g0/3 detail
```

---

## Resolution

```cisco
conf t

interface g0/3
 shutdown
 no shutdown

end
```

---

# 7. DHCP Snooping Binding Missing

## Symptoms

```text
show ip dhcp snooping binding
```

returns no entries.

---

## Possible Causes

- DHCP lease not successfully obtained
- DHCP Snooping disabled
- ATTACKER-PC using static IP
- Rogue DHCP interfering
- DHCP traffic blocked

---

## Verification

```bash
sudo udhcpc -i eth0
```

```cisco
show ip dhcp snooping
show ip dhcp snooping binding
```

---

## Resolution

### Verify DHCP Snooping Enabled

```cisco
show ip dhcp snooping
```

---

### Verify Trusted Interfaces

```cisco
show run interface g0/0
show run interface g0/1
```

---

### Disable Rogue dnsmasq

```bash
sudo pkill dnsmasq
```

---

### Renew DHCP Lease

```bash
sudo pkill udhcpc
sudo ip addr flush dev eth0
sudo udhcpc -i eth0
```

---

# 8. CML Configuration Loss After Export/Import

## Symptoms

- Device configurations disappear after re-importing the lab
- VLANs or security settings missing
- Startup-config not preserved

---

## Cause

Running/startup-config was not synchronized into the CML topology database before export.

---

## Resolution Procedure

### Save Configurations

```cisco
wr
```

---

### Extract Configurations

CML GUI:

```text
NODES
→ Select All Nodes
→ Extract Configs
```

---

### Export Lab

CML GUI:

```text
LAB
→ Download Lab
```

---

## Verification Procedure

Re-import the exported lab and verify:
- topology
- VLANs
- routing
- security configurations
- startup-config persistence

before considering the backup complete.

---

# 9. ATTACKER-PC Network Stack Instability

## Symptoms

- DHCP lease failures
- Interface unable to ping gateway
- Route missing
- ARP resolution failures

---

## Cause

Repeated:
- MAC address changes
- DHCP restarts
- ARP testing
- dnsmasq testing
- Interface reset operations

can destabilize Alpine Linux networking during lab testing.

---

## Resolution

### Reset Network State

```bash
sudo pkill udhcpc
sudo ip addr flush dev eth0
sudo ip neigh flush all
sudo ip link set eth0 down
sudo ip link set eth0 up
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
Valid DHCP address
Default route restored
Successful ping replies
```
