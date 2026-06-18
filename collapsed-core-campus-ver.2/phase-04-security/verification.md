# Phase 04 Security - Verification

## 1. Overview

This document records the verification results for Phase 04 Security.

Phase 04 was built on the final **Phase 03 Case 2 - Multi-Area OSPF** operational baseline.

The purpose of this verification is to confirm that the security features were correctly applied and that the campus network still operates normally after security hardening.

Validated items:

```text
Access Port Hardening
Port Security
DHCP Snooping
Dynamic ARP Inspection
Static Source Binding
IP Source Guard Validation
BPDU Guard
Management ACL / VTY SSH Access Control
Management SVI Reachability
SSH Access from MGMT-SRV1
SSH Deny from ROGUE-PC
```

---

## 2. Baseline Verification

Before applying security features, the Phase 03 Multi-Area OSPF baseline was verified.

### 2.1 EDGE1 OSPF Baseline

Verification commands:

```cisco
show ip ospf neighbor
show ip route ospf
show ip route 0.0.0.0
show ip sla statistics
show track
```

Expected results:

```text
EDGE1 has OSPF FULL neighbors with CORE1 and CORE2.
EDGE1 learns the summarized campus route 10.10.0.0/16.
Primary default route uses ISP1 while Track 1 is Up.
Backup floating default route is available through ISP2.
```

### 2.2 CORE1 / CORE2 Multi-Area OSPF Baseline

Verification commands:

```cisco
show ip ospf neighbor
show ip ospf interface brief
show ip protocols
show ip route ospf
```

Expected results:

```text
CORE1 and CORE2 operate as ABRs.
Area 0 and Area 10 are present.
OSPF summarization for 10.10.0.0/16 is active.
```

### 2.3 HSRP Baseline

Verification command:

```cisco
show standby brief
```

Expected HSRP roles:

```text
CORE1 Active:
- VLAN10
- VLAN11
- VLAN30
- VLAN40
- VLAN60

CORE2 Active:
- VLAN20
- VLAN21
- VLAN31
- VLAN50
- VLAN70
```

### 2.4 Baseline Correction

During baseline verification, CORE2 VLAN11 had an incorrect HSRP virtual IP.

Incorrect value:

```text
10.10.11.54
```

Corrected value:

```text
10.10.11.254
```

Verification after correction:

```cisco
show standby vlan 11
show standby brief
show logging | include DIFFVIP
```

Expected result:

```text
VLAN11 virtual IP is 10.10.11.254.
CORE2 is Standby for VLAN11.
No new DIFFVIP logs appear after correction.
```

---

## 3. Access Port Hardening Verification

Access-facing ports were configured as static access ports with DTP disabled and PortFast enabled.

### 3.1 Verification Commands

Run on ACC1, ACC2, ACC3, and SRV-ACC1:

```cisco
show interfaces status
show interfaces switchport
show spanning-tree interface <interface> detail
```

### 3.2 Expected Results

```text
Access-facing ports are configured as static access ports.
Negotiation of Trunking is Off.
PortFast Edge is enabled on endpoint-facing ports.
Trunk/uplink ports are not treated as access-facing endpoint ports.
```

### 3.3 Verified Access Ports

```text
ACC1:
- Gi0/2 EXEC VLAN10
- Gi0/3 BIZ-ADMIN VLAN11
- Gi1/0 PRINTER VLAN60

ACC2:
- Gi0/2 SALES VLAN20
- Gi0/3 MKT VLAN21
- Gi1/1 ROGUE-PC-TEST VLAN20

ACC3:
- Gi0/2 IT-OPS VLAN30
- Gi0/3 IT-SEC VLAN31

SRV-ACC1:
- Gi0/1 SERVER1 VLAN70
- Gi0/2 MGMT-SRV1 VLAN40
- Gi0/3 ROGUE-SW-TEST
```

Verification result:

```text
Access Port Hardening: Passed
```

---

## 4. Port Security Verification

Port Security was configured on access-facing ports.

Final Port Security design:

```text
Maximum MAC addresses: 1
Sticky MAC learning: enabled
Violation mode: restrict
```

### 4.1 Verification Commands

Run on access switches:

```cisco
show port-security
show port-security interface <interface>
show port-security address
```

### 4.2 Expected Results

```text
Port Security is Enabled.
Port Status is Secure-up on normal endpoint ports.
Violation Mode is Restrict.
Maximum MAC Addresses is 1.
Sticky MAC address is learned.
Security Violation Count is 0 before violation testing.
```

### 4.3 ACC2 ROGUE-PC Port Security Test

Test interface:

```text
ACC2 Gi1/1 - ROGUE-PC-TEST
```

Original ROGUE-PC MAC:

```text
52:54:00:61:B0:C0
```

Test MAC:

```text
02:00:00:11:11:11
```

ROGUE-PC test commands:

```bash
sudo ip link set eth0 down
sudo ip link set eth0 address 02:00:00:11:11:11
sudo ip link set eth0 up
ping -c 4 10.10.20.254
```

ACC2 verification commands:

```cisco
show port-security interface g1/1
show port-security
show logging | include PSECURE|PORT_SECURITY
```

Expected result:

```text
Security Violation Count increases.
The interface remains Secure-up.
Violating traffic is dropped.
Port Security log messages are generated.
```

Recovery:

```bash
sudo ip link set eth0 down
sudo ip link set eth0 address 52:54:00:61:b0:c0
sudo ip link set eth0 up
sudo ip addr flush dev eth0
sudo udhcpc -i eth0 -n
```

Verification result:

```text
Port Security: Passed
```

---

## 5. DHCP Snooping Verification

DHCP Snooping was enabled on access-layer switches.

Protected VLANs:

```text
10, 11, 20, 21, 30, 31, 40, 50, 60, 70
```

Final DHCP Snooping trust model:

```text
Trusted:
- Uplinks / trunks
- DHCP server-facing port on SRV-ACC1 Gi0/2

Untrusted:
- User-facing ports
- ROGUE-PC port
- ROGUE-SW test port
```

### 5.1 Verification Commands

Run on ACC1, ACC2, ACC3, and SRV-ACC1:

```cisco
show ip dhcp snooping
show ip dhcp snooping binding
show ip dhcp snooping statistics
show logging | include DHCP|SNOOP|DROP
```

### 5.2 Expected Results

```text
DHCP Snooping is enabled.
Configured VLANs are operational.
Uplink/trunk ports are trusted.
Access-facing ports are untrusted with rate limit 5.
Option 82 is disabled.
Valid DHCP clients can receive IP addresses.
DHCP Snooping binding table is populated after DHCP success.
No DHCP Snooping drops occur for valid clients.
```

### 5.3 ROGUE-PC DHCP Validation

ROGUE-PC command:

```bash
sudo ip addr flush dev eth0
sudo udhcpc -i eth0 -n
ip addr show eth0
ip route
```

Expected result:

```text
ROGUE-PC receives a VLAN20 DHCP address.
Example IP: 10.10.20.126/24
DHCP server: 10.10.40.10
```

ACC2 binding verification:

```cisco
show ip dhcp snooping binding
```

Expected binding:

```text
MAC: 52:54:00:61:B0:C0
IP: 10.10.20.126
VLAN: 20
Interface: Gi1/1
Type: dhcp-snooping
```

Verification result:

```text
DHCP Snooping: Passed
```

---

## 6. Dynamic ARP Inspection Verification

DAI was enabled after DHCP Snooping was validated.

DAI validation mode:

```text
Validate source MAC, destination MAC, and IP address.
```

### 6.1 ACC2 DAI Baseline Verification

Commands:

```cisco
show ip arp inspection
show ip arp inspection interfaces
show ip arp inspection statistics
show ip dhcp snooping binding
```

Expected results:

```text
VLAN20 and VLAN21 DAI are Enabled and Active.
Gi0/0, Gi0/1, and Gi1/0 are trusted.
Gi0/2, Gi0/3, and Gi1/1 are untrusted with rate limit 15.
DHCP Snooping binding exists for ROGUE-PC.
```

### 6.2 Normal ARP / Ping Verification

ROGUE-PC:

```bash
ping -c 4 10.10.20.254
ping -c 4 10.10.40.10
```

ACC2:

```cisco
show ip arp inspection statistics
show logging | include ARP|DAI|INSPECTION
```

Expected result:

```text
Ping succeeds.
DAI Forwarded counter increases.
DAI Dropped counter remains 0.
No DAI deny log appears for valid ARP traffic.
```

### 6.3 DAI Violation Test

ROGUE-PC was manually assigned an IP address different from its DHCP Snooping binding.

Valid DHCP Snooping binding:

```text
52:54:00:61:B0:C0 / 10.10.20.126 / VLAN20 / Gi1/1
```

Spoofed IP used for test:

```text
10.10.20.200
```

ROGUE-PC commands:

```bash
sudo ip addr flush dev eth0
sudo ip addr add 10.10.20.200/24 dev eth0
sudo ip link set eth0 up
ping -c 4 10.10.20.254
```

ACC2 verification:

```cisco
show ip arp inspection statistics
show logging | include ARP|DAI|INSPECTION
show ip dhcp snooping binding
```

Expected result:

```text
DAI Dropped counter increases.
DHCP Drops counter increases.
DAI log identifies invalid ARP on Gi1/1 VLAN20.
```

Expected log concept:

```text
Invalid ARPs on Gi1/1, vlan 20
source MAC: 5254.0061.b0c0
spoofed IP: 10.10.20.200
target IP: 10.10.20.254
```

Recovery:

```bash
sudo ip addr flush dev eth0
sudo udhcpc -i eth0 -n
ip addr show eth0
ip route
```

Verification result:

```text
Dynamic ARP Inspection: Passed
```

---

## 7. DAI Final Deployment Verification

DAI was applied to the access switches.

### 7.1 ACC1

DAI VLANs:

```text
10, 11, 60
```

Verification commands:

```cisco
show ip arp inspection
show ip arp inspection interfaces
show ip arp inspection statistics
```

Expected result:

```text
VLAN10, VLAN11, and VLAN60 are Enabled and Active.
Uplinks are trusted.
Access ports are untrusted with rate limit 15.
Dropped counter remains 0 for valid traffic.
```

### 7.2 ACC2

DAI VLANs:

```text
20, 21
```

Verification commands:

```cisco
show ip arp inspection
show ip arp inspection interfaces
show ip arp inspection statistics
```

Expected result:

```text
VLAN20 and VLAN21 are Enabled and Active.
Uplinks and trunk to SRV-ACC1 are trusted.
Access ports are untrusted with rate limit 15.
DAI violation test on Gi1/1 was successful.
```

### 7.3 ACC3

DAI VLANs:

```text
30, 31
```

Verification commands:

```cisco
show ip arp inspection
show ip arp inspection interfaces
show ip arp inspection statistics
```

Expected result:

```text
VLAN30 and VLAN31 are Enabled and Active.
Uplinks are trusted.
Access ports are untrusted with rate limit 15.
Dropped counter remains 0 for valid traffic.
```

Verification result:

```text
DAI on user access switches: Passed
```

---

## 8. Static Source Binding and Server VLAN DAI Verification

SRV-ACC1 connects statically addressed servers.

Static hosts:

```text
MGMT-SRV1 - 10.10.40.10
SERVER1   - 10.10.70.10
```

Static source bindings were configured before enabling DAI on VLAN40 and VLAN70.

### 8.1 Static Source Binding Verification

SRV-ACC1 command:

```cisco
show ip source binding
```

Expected result:

```text
MGMT-SRV1:
- MAC: 52:54:00:0D:60:61
- IP: 10.10.40.10
- VLAN: 40
- Interface: Gi0/2

SERVER1:
- MAC: 52:54:00:BE:8B:EE
- IP: 10.10.70.10
- VLAN: 70
- Interface: Gi0/1
```

### 8.2 SRV-ACC1 DAI Verification

Commands:

```cisco
show ip arp inspection
show ip arp inspection interfaces
show ip arp inspection statistics
show logging | include ARP|DAI|INSPECTION
```

Expected result:

```text
VLAN40 and VLAN70 are Enabled and Active.
Gi0/0 is trusted.
Gi0/1, Gi0/2, and Gi0/3 are untrusted with rate limit 15.
No new DAI deny logs appear for valid server traffic.
Dropped counter remains 0 after counter reset and normal traffic test.
```

### 8.3 MGMT-SRV1 Functional Verification

MGMT-SRV1:

```bash
ping -c 4 10.10.40.254
ping -c 4 10.10.70.10
```

Expected result:

```text
MGMT-SRV1 can reach VLAN40 HSRP gateway.
MGMT-SRV1 can reach SERVER1.
```

### 8.4 SERVER1 Functional Verification

SERVER1:

```bash
ping -c 4 10.10.70.254
ping -c 4 10.10.40.10
```

Expected result:

```text
SERVER1 can reach VLAN70 HSRP gateway.
SERVER1 can reach MGMT-SRV1.
```

Verification result:

```text
Static Source Binding + DAI for static server VLANs: Passed
```

---

## 9. IP Source Guard Verification

IP Source Guard was tested on ACC2 Gi1/1.

Test host:

```text
ROGUE-PC
Interface: ACC2 Gi1/1
VLAN: 20
IP: 10.10.20.126
MAC: 52:54:00:61:B0:C0
```

### 9.1 IP-Only Mode Test

Configuration tested:

```cisco
interface g1/1
 ip verify source
```

Verification:

```cisco
show ip verify source
show ip dhcp snooping binding
show port-security interface g1/1
```

Observed result:

```text
Binding output appeared correct.
Host traffic failed.
```

### 9.2 IP/MAC Mode with Port Security Test

Configuration tested:

```cisco
interface g1/1
 no ip verify source
 ip verify source port-security
```

Verification command:

```cisco
show ip verify source
```

Observed output concept:

```text
Interface: Gi1/1
Filter type: ip-mac
Filter mode: active
IP address: 10.10.20.126
MAC address: 52:54:00:61:B0:C0
VLAN: 20
```

### 9.3 Traffic Validation

ROGUE-PC:

```bash
ping -c 4 10.10.20.254
ping -c 4 10.10.40.10
```

Observed result:

```text
Traffic failed while IP Source Guard was enabled.
```

### 9.4 Isolation Test

ACC2:

```cisco
conf t
interface g1/1
 no ip verify source
end
wr
```

ROGUE-PC:

```bash
ping -c 4 10.10.20.254
ping -c 4 10.10.40.10
```

Observed result:

```text
Traffic immediately recovered after removing IP Source Guard.
```

### 9.5 Final IP Source Guard Decision

```text
IP Source Guard was validated but not retained on active access ports.
```

Reason:

```text
CML IOSvL2 showed platform-specific data-plane behavior where host traffic was dropped even when the DHCP Snooping binding, Port Security sticky MAC, and IP Source Guard output were correct.
```

Verification result:

```text
IP Source Guard: Validated, not retained
```

---

## 10. BPDU Guard Verification

BPDU Guard was enabled on access-facing PortFast edge ports.

### 10.1 Verification Commands

Run on access switches:

```cisco
show spanning-tree summary
show run | include bpduguard
show interface status err-disabled
show logging | include BPDU|SPANTREE|ERR|err
```

### 10.2 Expected Results for Normal Access Ports

```text
BPDU Guard is enabled on access-facing ports.
Normal endpoint ports are not err-disabled.
No BPDU Guard violation appears on normal endpoint ports.
```

### 10.3 ROGUE-SW-TEST Validation

Test interface:

```text
SRV-ACC1 Gi0/3
```

Test condition:

```text
ROGUE-SW-TEST was connected to SRV-ACC1 Gi0/3.
Gi0/3 was configured as a PortFast edge port with BPDU Guard enabled.
```

Expected result:

```text
Gi0/3 receives a BPDU.
BPDU Guard disables the port.
Gi0/3 enters err-disabled state.
Err-disable reason is bpduguard.
```

Verification commands:

```cisco
show interface status err-disabled
show logging | include BPDU|SPANTREE|ERR|err
show run interface g0/3
```

Expected log concept:

```text
%SPANTREE-2-BLOCK_BPDUGUARD: Received BPDU on port Gi0/3 with BPDU Guard enabled. Disabling port.
%PM-4-ERR_DISABLE: bpduguard error detected on Gi0/3, putting Gi0/3 in err-disable state
```

Verification result:

```text
BPDU Guard: Passed
```

Note:

```text
SRV-ACC1 Gi0/3 may remain err-disabled as intentional evidence of BPDU Guard validation.
If the port must be restored, remove or power off ROGUE-SW-TEST first, then use shutdown/no shutdown.
```

---

## 11. Management SVI Verification for Access Switches

Access switches were assigned VLAN40 management SVIs.

Management IPs:

```text
ACC1      10.10.40.11
ACC2      10.10.40.12
ACC3      10.10.40.13
SRV-ACC1  10.10.40.14
```

Default gateway:

```text
10.10.40.254
```

### 11.1 Verification Commands

Run on each access switch:

```cisco
show ip interface brief | include Vlan40
show run interface vlan40
show run | include ip default-gateway
show vlan brief | include 40
show interfaces trunk
```

### 11.2 Expected Results

```text
Vlan40 is up/up.
Correct management IP is assigned.
Default gateway is 10.10.40.254.
VLAN40 exists.
VLAN40 is allowed on trunks.
```

### 11.3 MGMT-SRV1 Reachability Test

MGMT-SRV1:

```bash
ping -c 4 10.10.40.11
ping -c 4 10.10.40.12
ping -c 4 10.10.40.13
ping -c 4 10.10.40.14
```

Expected result:

```text
MGMT-SRV1 can ping all access switch management SVIs.
```

Verification result:

```text
Access Switch Management SVI: Passed
```

---

## 12. SSH Management Access Verification

SSH access should be allowed only from MGMT-SRV1.

Allowed source:

```text
MGMT-SRV1: 10.10.40.10
```

Denied test source:

```text
ROGUE-PC: 10.10.20.126
```

### 12.1 Common VTY ACL

```cisco
ip access-list standard VTY-MGMT-ONLY
 permit host 10.10.40.10
 deny any log
```

### 12.2 SSH Client Compatibility Command

Ubuntu OpenSSH required explicit legacy algorithm options for IOSv / IOSvL2 SSH testing.

MGMT-SRV1 SSH command format:

```bash
ssh -oKexAlgorithms=+diffie-hellman-group14-sha1 -oHostKeyAlgorithms=+ssh-rsa -oPubkeyAcceptedAlgorithms=+ssh-rsa admin@<DEVICE-IP>
```

### 12.3 MGMT-SRV1 SSH Permit Test

MGMT-SRV1 successfully SSHed into network devices.

Examples:

```bash
ssh -oKexAlgorithms=+diffie-hellman-group14-sha1 -oHostKeyAlgorithms=+ssh-rsa -oPubkeyAcceptedAlgorithms=+ssh-rsa admin@10.10.40.1
ssh -oKexAlgorithms=+diffie-hellman-group14-sha1 -oHostKeyAlgorithms=+ssh-rsa -oPubkeyAcceptedAlgorithms=+ssh-rsa admin@10.10.40.2
ssh -oKexAlgorithms=+diffie-hellman-group14-sha1 -oHostKeyAlgorithms=+ssh-rsa -oPubkeyAcceptedAlgorithms=+ssh-rsa admin@10.10.40.11
ssh -oKexAlgorithms=+diffie-hellman-group14-sha1 -oHostKeyAlgorithms=+ssh-rsa -oPubkeyAcceptedAlgorithms=+ssh-rsa admin@10.10.40.12
ssh -oKexAlgorithms=+diffie-hellman-group14-sha1 -oHostKeyAlgorithms=+ssh-rsa -oPubkeyAcceptedAlgorithms=+ssh-rsa admin@10.10.40.13
ssh -oKexAlgorithms=+diffie-hellman-group14-sha1 -oHostKeyAlgorithms=+ssh-rsa -oPubkeyAcceptedAlgorithms=+ssh-rsa admin@10.10.40.14
ssh -oKexAlgorithms=+diffie-hellman-group14-sha1 -oHostKeyAlgorithms=+ssh-rsa -oPubkeyAcceptedAlgorithms=+ssh-rsa admin@172.16.0.1
```

Expected result:

```text
SSH login succeeds from MGMT-SRV1.
VTY-MGMT-ONLY permit counter increases.
```

### 12.4 ROGUE-PC SSH Deny Test

ROGUE-PC attempted SSH to network devices.

Example:

```bash
ssh -oKexAlgorithms=+diffie-hellman-group14-sha1 -oHostKeyAlgorithms=+ssh-rsa -oPubkeyAcceptedAlgorithms=+ssh-rsa admin@10.10.40.11
```

Expected result:

```text
SSH login is not allowed.
No successful login prompt should be reached.
VTY-MGMT-ONLY deny counter increases.
```

### 12.5 Device-Side Verification

Run on tested devices:

```cisco
show access-lists VTY-MGMT-ONLY
show logging | include VTY|SSH|SEC|denied|Deny|ACL
show run | section line vty
```

Expected result:

```text
permit host 10.10.40.10 hit count increases for MGMT-SRV1.
deny any log hit count increases for ROGUE-PC.
ACL log entry appears for denied source.
```

Example ACC1 validation concept:

```text
permit 10.10.40.10 matched MGMT-SRV1 SSH access.
deny any log matched ROGUE-PC SSH access from 10.10.20.126.
```

Verification result:

```text
Management ACL / VTY SSH Access Control: Passed
```

---

## 13. EDGE1 SSH Verification

EDGE1 initially required RSA key generation.

### 13.1 Verification Commands

EDGE1:

```cisco
show ip ssh
show crypto key mypubkey rsa
show run | include ^aaa|^username|ip ssh|ip domain-name
show access-lists VTY-MGMT-ONLY
show run | section line vty
```

### 13.2 Expected Results

```text
SSH Enabled - version 2.0
RSA key exists
ip domain-name campus.lab exists
AAA local authentication exists
VTY-MGMT-ONLY ACL exists
VTY lines allow SSH only
```

### 13.3 Management Access IP

EDGE1 was managed through campus-facing routed links.

```text
Primary: 172.16.0.1
Backup:  172.16.0.5
```

Expected result:

```text
MGMT-SRV1 can SSH to EDGE1 using a campus-facing IP.
WAN-facing IPs are not used for management validation.
```

Verification result:

```text
EDGE1 SSH Management: Passed
```

---

## 14. SERVER1 Static IP Verification

SERVER1 was configured with a static IP address in VLAN70.

### 14.1 SERVER1 Netplan

File:

```bash
/etc/netplan/50-cloud-init.yaml
```

Expected configuration:

```yaml
network:
  version: 2
  ethernets:
    ens2:
      addresses:
        - 10.10.70.10/24
      routes:
        - to: default
          via: 10.10.70.254
      nameservers:
        addresses:
          - 10.10.40.10
```

### 14.2 Verification Commands

SERVER1:

```bash
ip addr show ens2
ip route
ping -c 4 10.10.70.254
ping -c 4 10.10.40.10
```

Expected result:

```text
ens2 has 10.10.70.10/24.
Default route points to 10.10.70.254.
Ping to VLAN70 HSRP gateway succeeds.
Ping to MGMT-SRV1 succeeds.
```

Verification result:

```text
SERVER1 Static IP Persistence: Passed
```

---

## 15. Final Device Verification Commands

### 15.1 Access Switches

Run on ACC1, ACC2, ACC3, and SRV-ACC1:

```cisco
show port-security
show ip dhcp snooping
show ip dhcp snooping binding
show ip arp inspection
show ip arp inspection interfaces
show ip arp inspection statistics
show spanning-tree summary
show run | include bpduguard
show interface status err-disabled
show access-lists VTY-MGMT-ONLY
show run | section line vty
show ip ssh
```

### 15.2 ACC2 Additional Verification

```cisco
show port-security interface g1/1
show ip verify source
show logging | include PSECURE|PORT_SECURITY|DAI|ARP|DHCP|SNOOP|VERIFY|SOURCE
```

Expected final IP Source Guard state:

```text
No retained IP Source Guard configuration on Gi1/1.
ROGUE-PC connectivity works after no ip verify source.
```

### 15.3 SRV-ACC1 Additional Verification

```cisco
show ip source binding
show ip arp inspection statistics
show interface status err-disabled
show logging | include BPDU|SPANTREE|ERR|err
```

Expected results:

```text
Static source bindings exist for MGMT-SRV1 and SERVER1.
DAI is active on VLAN40 and VLAN70.
Valid server traffic is not dropped.
Gi0/3 may remain err-disabled due to intentional BPDU Guard validation.
```

### 15.4 CORE1 / CORE2 / EDGE1

```cisco
show run | include ^aaa|^username|ip ssh|ip domain-name
show access-lists VTY-MGMT-ONLY
show run | section line vty
show ip ssh
```

EDGE1 additional command:

```cisco
show crypto key mypubkey rsa
```

---

## 16. Final Verification Matrix

| Feature               | Validation Method                       | Result                  |
| --------------------- | --------------------------------------- | ----------------------- |
| Access Port Hardening | Checked access mode, no DTP, PortFast   | Passed                  |
| Port Security         | Sticky MAC and violation restrict test  | Passed                  |
| DHCP Snooping         | Binding table and DHCP lease validation | Passed                  |
| DAI                   | Normal ARP and spoofed ARP test         | Passed                  |
| Static Source Binding | MGMT-SRV1 / SERVER1 DAI validation      | Passed                  |
| IP Source Guard       | Binding verified, traffic tested        | Validated, not retained |
| BPDU Guard            | ROGUE-SW triggered err-disabled         | Passed                  |
| Management SVI        | VLAN40 SVI reachability                 | Passed                  |
| Management ACL        | MGMT-SRV1 allowed, ROGUE-PC denied      | Passed                  |
| EDGE1 SSH             | RSA key and SSHv2 validation            | Passed                  |
| SERVER1 Static IP     | Netplan and ping verification           | Passed                  |

---

## 17. Final Phase 04 Verification Status

Final result:

```text
Phase 04 Security verification completed successfully.
```

Summary:

```text
Access-layer security controls were configured and validated.
DHCP Snooping and DAI operated correctly.
Static source bindings allowed secure DAI enforcement for static servers.
BPDU Guard successfully blocked rogue switch attachment.
Management-plane SSH access was restricted to MGMT-SRV1.
ROGUE-PC SSH access was denied by VTY-MGMT-ONLY ACL.
IP Source Guard was validated but not retained due to CML IOSvL2 behavior.
```

Final status:

```text
Phase 04 Security: Completed
```
