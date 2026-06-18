# Phase 04 Security - Troubleshooting

## 1. Overview

This document records troubleshooting cases and operational lessons from Phase 04 Security.

Phase 04 focused on:

```text
Access Port Hardening
Port Security
DHCP Snooping
Dynamic ARP Inspection
Static Source Binding
IP Source Guard Validation
BPDU Guard
Management ACL / VTY SSH Access Control
```

Phase 04 was built on the final Phase 03 **Multi-Area OSPF** operational baseline.

---

## 2. Troubleshooting Summary

| Issue                                      | Symptom                            | Root Cause                                  | Resolution                                              |
| ------------------------------------------ | ---------------------------------- | ------------------------------------------- | ------------------------------------------------------- |
| CORE2 VLAN11 HSRP VIP mismatch             | HSRP DIFFVIP log                   | VLAN11 VIP typo                             | Corrected VIP to 10.10.11.254                           |
| Port Security violation                    | Violation counter increased        | ROGUE-PC MAC changed                        | Restored MAC and verified Secure-up                     |
| DHCP client failed to get IP               | DHCP Discover sent but no lease    | DHCP path/service required validation       | Verified snooping trust, relay, server reachability     |
| DAI dropped static server ARP              | MGMT-SRV1 / SERVER1 ARP dropped    | Static IP hosts had no dynamic DHCP binding | Added static source bindings                            |
| IP Source Guard blocked valid host traffic | Binding correct but ping failed    | CML IOSvL2 data-plane behavior              | Validated but did not retain IPSG                       |
| BPDU Guard err-disabled Gi0/3              | ROGUE-SW caused port shutdown      | BPDU received on PortFast edge port         | Expected behavior; remove rogue switch and recover port |
| ACCESS switch SSH login failed             | SSH reachable but login failed     | `login local` used without local username   | Added local admin user                                  |
| EDGE1 SSH disabled                         | SSH not enabled despite VTY config | RSA key missing                             | Generated RSA key after setting domain name             |
| Ubuntu SSH algorithm mismatch              | SSH negotiation failed             | IOS uses legacy SSH algorithms              | Used explicit OpenSSH options                           |
| VTY ACL deny validation                    | ROGUE-PC SSH blocked               | VTY-MGMT-ONLY ACL worked                    | Confirmed deny hit count and log                        |

---

## 3. Baseline Issue - CORE2 VLAN11 HSRP VIP Mismatch

### 3.1 Symptom

During Phase 04 baseline validation, CORE2 generated an HSRP DIFFVIP message for VLAN11.

Observed log concept:

```text
%HSRP-4-DIFFVIP1:
Vlan11 virtual IP 10.10.11.254 is different to locally configured address 10.10.11.54
```

### 3.2 Cause

CORE2 VLAN11 had an incorrect HSRP virtual IP configured.

Incorrect value:

```text
10.10.11.54
```

Correct value:

```text
10.10.11.254
```

### 3.3 Resolution

CORE2:

```cisco
conf t
interface vlan11
 no standby 11 ip 10.10.11.54
 standby 11 ip 10.10.11.254
end
wr
```

### 3.4 Verification

```cisco
show standby vlan 11
show standby brief
show logging | include DIFFVIP
```

Expected result:

```text
VLAN11 VIP = 10.10.11.254
CORE2 VLAN11 = Standby
No new DIFFVIP log after correction
```

### 3.5 Lesson Learned

Baseline validation must be completed before applying new security features.
Security troubleshooting should not start until the routing, HSRP, VLAN, and management baseline is confirmed.

---

## 4. Port Security Troubleshooting

## 4.1 Symptom

ROGUE-PC MAC address was changed during Port Security testing.

Original MAC:

```text
52:54:00:61:B0:C0
```

Test MAC:

```text
02:00:00:11:11:11
```

Observed result on ACC2 Gi1/1:

```text
Port Status: Secure-up
Violation Mode: Restrict
Security Violation Count increased
```

### 4.2 Cause

Port Security was configured with:

```text
Maximum MAC Addresses: 1
Sticky MAC: enabled
Violation mode: restrict
```

When the ROGUE-PC MAC was changed, ACC2 Gi1/1 detected traffic from a MAC address different from the sticky learned MAC.

### 4.3 Expected Behavior

Because violation mode was `restrict`, the port did not become err-disabled.

Expected behavior:

```text
Violating traffic is dropped.
Violation counter increases.
Log messages are generated.
The interface remains Secure-up.
```

### 4.4 Verification Commands

```cisco
show port-security interface g1/1
show port-security
show logging | include PSECURE|PORT_SECURITY
```

### 4.5 Recovery

ROGUE-PC:

```bash
sudo ip link set eth0 down
sudo ip link set eth0 address 52:54:00:61:b0:c0
sudo ip link set eth0 up
```

Then request DHCP again if needed:

```bash
sudo ip addr flush dev eth0
sudo udhcpc -i eth0 -n
```

### 4.6 Lesson Learned

`restrict` mode is useful for validation because it allows the port to remain up while recording violation counters and logs.

---

## 5. DHCP Snooping Troubleshooting

## 5.1 Symptom

A DHCP client may fail to receive an address after DHCP Snooping is enabled.

Client-side symptom:

```text
Sending discover...
No lease, failing
```

### 5.2 Common Causes

Check the following items:

```text
1. Client access port VLAN assignment
2. DHCP Snooping enabled VLAN list
3. Trusted uplink/trunk ports
4. DHCP server-facing trusted port
5. DHCP relay configuration on CORE SVIs
6. Reachability from CORE SVIs to MGMT-SRV1
7. DHCP service availability on MGMT-SRV1
8. DHCP Snooping binding table
9. DHCP Snooping statistics and drop counters
```

### 5.3 Final Design Note

In the final Phase 04 design, DHCP Snooping enforcement is applied at the access layer.

Final enforcement devices:

```text
ACC1
ACC2
ACC3
SRV-ACC1
```

CORE1 and CORE2 remain DHCP relay devices using `ip helper-address`.

They are not retained as DHCP Snooping enforcement points in this phase.

### 5.4 Access Switch Verification

On the access switch:

```cisco
show ip dhcp snooping
show ip dhcp snooping binding
show ip dhcp snooping statistics
show logging | include DHCP|SNOOP|DROP
```

Expected result:

```text
DHCP Snooping enabled
Correct VLANs active
Uplinks trusted
Access ports untrusted
No DHCP Snooping drops for valid clients
Binding table created after DHCP success
```

### 5.5 CORE Relay Verification

On CORE1 and CORE2:

```cisco
show run interface vlan20
show run interface vlan40
ping 10.10.40.10 source vlan20
ping 10.10.40.10 source vlan40
```

Expected VLAN20 relay configuration:

```cisco
interface Vlan20
 ip helper-address 10.10.40.10
```

### 5.6 MGMT-SRV1 Packet Verification

When DHCP troubleshooting requires packet-level validation, use tcpdump:

```bash
sudo tcpdump -ni ens2 port 67 or port 68
```

Then request DHCP again from the client:

```bash
sudo ip addr flush dev eth0
sudo udhcpc -i eth0 -n
```

Interpretation:

```text
DHCPDISCOVER appears on MGMT-SRV1:
- DHCP relay path is working.
- Check DHCP service response.

DHCPDISCOVER does not appear on MGMT-SRV1:
- Check access switch VLAN, trunk, trust, and CORE relay path.
```

### 5.7 Successful Validation Example

ACC2 DHCP Snooping binding after successful validation:

```text
ROGUE-PC:
- MAC: 52:54:00:61:B0:C0
- IP: 10.10.20.126
- VLAN: 20
- Interface: Gi1/1
```

### 5.8 Lesson Learned

DHCP Snooping troubleshooting should follow the packet path:

```text
Client
→ Access port
→ Access switch snooping policy
→ Trunk/uplink trust
→ CORE SVI relay
→ MGMT-SRV1 DHCP service
→ Return path
→ DHCP Snooping binding table
```

---

## 6. Dynamic ARP Inspection Troubleshooting

## 6.1 Symptom

DAI can drop ARP traffic if the ARP sender does not match the DHCP Snooping binding table.

Observed on ACC2 during ARP spoofing test:

```text
Dropped increased
DHCP Drops increased
DAI log generated
```

### 6.2 Test Scenario

Valid DHCP binding:

```text
MAC: 52:54:00:61:B0:C0
IP: 10.10.20.126
VLAN: 20
Interface: Gi1/1
```

ROGUE-PC was manually changed to:

```text
IP: 10.10.20.200
MAC: 52:54:00:61:B0:C0
```

### 6.3 Expected DAI Behavior

DAI drops ARP because the IP/MAC pair does not match the DHCP Snooping binding.

Expected log concept:

```text
Invalid ARPs on Gi1/1, vlan 20
source MAC: 5254.0061.b0c0
spoofed IP: 10.10.20.200
```

### 6.4 Verification Commands

```cisco
show ip arp inspection
show ip arp inspection interfaces
show ip arp inspection statistics
show ip dhcp snooping binding
show logging | include ARP|DAI|INSPECTION
```

### 6.5 Recovery

Restore the client to DHCP:

```bash
sudo ip addr flush dev eth0
sudo udhcpc -i eth0 -n
```

Then verify:

```bash
ip addr show eth0
ip route
ping -c 4 10.10.20.254
ping -c 4 10.10.40.10
```

### 6.6 Lesson Learned

DAI depends on DHCP Snooping bindings.
If a host uses an IP address different from the binding table, DAI correctly blocks the ARP.

---

## 7. DAI and Static IP Server Troubleshooting

## 7.1 Symptom

When DAI was enabled on SRV-ACC1 VLAN40 and VLAN70, valid ARP traffic from statically addressed servers could be dropped.

Observed condition:

```text
VLAN40 DAI drops increased
DHCP Drops increased
```

### 7.2 Cause

SRV-ACC1 connects static IP hosts:

```text
MGMT-SRV1 - 10.10.40.10
SERVER1   - 10.10.70.10
```

Because these hosts use static IP addresses, they do not automatically create dynamic DHCP Snooping bindings.

Without a binding, DAI may treat their ARP traffic as invalid.

### 7.3 Resolution

Static source bindings were configured on SRV-ACC1.

```cisco
ip source binding 5254.000d.6061 vlan 40 10.10.40.10 interface Gi0/2
ip source binding 5254.00be.8bee vlan 70 10.10.70.10 interface Gi0/1
```

### 7.4 Verification

```cisco
show ip source binding
show ip arp inspection
show ip arp inspection statistics
show logging | include ARP|DAI|INSPECTION
```

Expected static bindings:

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

### 7.5 Functional Test

MGMT-SRV1:

```bash
ping -c 4 10.10.40.254
ping -c 4 10.10.70.10
```

SERVER1:

```bash
ping -c 4 10.10.70.254
ping -c 4 10.10.40.10
```

Expected result:

```text
Pings succeed.
DAI Forwarded counters increase.
Dropped counters remain 0 after counter reset.
No new DAI deny logs appear.
```

### 7.6 Lesson Learned

For static IP hosts, DAI should not be enabled blindly.
Use static source bindings or ARP ACLs before enabling DAI on static server VLANs.

Static source bindings were selected because they can also support future IP Source Guard consistency.

---

## 8. IP Source Guard Troubleshooting

## 8.1 Symptom

IP Source Guard was tested on ACC2 Gi1/1.

The switch showed a correct binding, but host traffic failed.

Observed valid output:

```text
Interface: Gi1/1
Filter type: ip-mac
Filter mode: active
IP address: 10.10.20.126
MAC address: 52:54:00:61:B0:C0
VLAN: 20
```

However:

```text
ROGUE-PC → 10.10.20.254 failed
ROGUE-PC → 10.10.40.10 failed
```

### 8.2 Tested Modes

IP-only mode:

```cisco
interface g1/1
 ip verify source
```

IP/MAC mode with Port Security:

```cisco
interface g1/1
 ip verify source port-security
```

### 8.3 Validation Steps

Checked the following:

```cisco
show ip verify source
show ip dhcp snooping binding
show port-security interface g1/1
show ip arp inspection statistics
show logging | include ARP|DAI|INSPECTION|VERIFY|SOURCE|PSECURE|PORT_SECURITY
```

Confirmed:

```text
DHCP Snooping binding was correct.
Port Security sticky MAC was correct.
DAI drops did not increase.
Port Security violations did not increase.
STP forwarding state was normal.
```

### 8.4 Isolation Test

Removed IP Source Guard:

```cisco
conf t
interface g1/1
 no ip verify source
end
wr
```

After removal:

```text
ROGUE-PC ping immediately recovered.
```

### 8.5 Final Decision

IP Source Guard was validated but not retained on active access ports in Phase 04.

Reason:

```text
CML IOSvL2 showed platform-specific data-plane behavior where traffic was dropped even though the IP/MAC/VLAN binding was correct.
```

This was treated as a lab platform limitation, not a security design error.

### 8.6 Lesson Learned

When control-plane validation output is correct but traffic still fails, isolate by removing the feature and testing data-plane recovery.

For this lab:

```text
IP Source Guard enabled  → traffic failed
IP Source Guard removed  → traffic recovered
```

---

## 9. BPDU Guard Troubleshooting

## 9.1 Symptom

SRV-ACC1 Gi0/3 went into err-disabled state.

Observed status:

```text
Gi0/3 err-disabled
Reason: bpduguard
```

### 9.2 Cause

ROGUE-SW-TEST was connected to an access-facing PortFast edge port.

Because BPDU Guard was enabled, receiving a BPDU caused the port to shut down.

Observed log concept:

```text
%SPANTREE-2-BLOCK_BPDUGUARD: Received BPDU on port Gi0/3 with BPDU Guard enabled. Disabling port.
%PM-4-ERR_DISABLE: bpduguard error detected on Gi0/3, putting Gi0/3 in err-disable state
```

### 9.3 Expected Behavior

This is expected and correct behavior.

BPDU Guard protects the access layer by disabling a PortFast edge port that receives BPDUs.

### 9.4 Verification Commands

```cisco
show interface status err-disabled
show logging | include BPDU|SPANTREE|ERR|err
show spanning-tree summary
show run interface g0/3
```

### 9.5 Recovery

First remove or power off the rogue switch.

Then recover the port:

```cisco
conf t
interface g0/3
 shutdown
 no shutdown
end
```

Verify:

```cisco
show interface status err-disabled
show interface status
```

### 9.6 Important Note

If ROGUE-SW-TEST remains connected and continues sending BPDUs, the port will immediately return to err-disabled after recovery.

### 9.7 Lesson Learned

BPDU Guard is effective for preventing unauthorized switch attachment on access-facing ports.

---

## 10. Management ACL / VTY SSH Troubleshooting

## 10.1 Symptom - ACCESS Switch SSH Login Failed

ACCESS switches had VTY lines configured with:

```cisco
login local
transport input ssh
```

However, SSH authentication failed.

### 10.2 Cause

`login local` requires a local username in the device local database.

The local admin account was missing on ACCESS switches.

### 10.3 Resolution

Configured local admin account on ACC1, ACC2, ACC3, and SRV-ACC1:

```cisco
conf t
username admin privilege 15 secret <ADMIN_PASSWORD>
end
wr
```

### 10.4 Verification

```cisco
show run | include ^username
show run | section line vty
show ip ssh
```

Expected result:

```text
username admin privilege 15 secret ...
line vty uses login local
SSH enabled
```

### 10.5 Lesson Learned

VTY ACL controls who can reach the VTY line.
`login local` controls how the user authenticates.
Both are required for successful SSH login.

---

## 11. EDGE1 SSH Troubleshooting

## 11.1 Symptom

EDGE1 showed SSH disabled.

Observed concept:

```text
SSH Disabled - version 2.0
Please create RSA keys to enable SSH
```

### 11.2 Cause

RSA keys had not been generated on EDGE1.

IOS requires a domain name and RSA key pair before SSH can operate.

### 11.3 Resolution

EDGE1:

```cisco
conf t
ip domain-name campus.lab
crypto key generate rsa modulus 2048
ip ssh version 2
ip ssh time-out 60
ip ssh authentication-retries 3
end
wr
```

### 11.4 Verification

```cisco
show ip ssh
show crypto key mypubkey rsa
show run | include ip domain-name|ip ssh
```

Expected result:

```text
SSH Enabled - version 2.0
RSA key exists
```

### 11.5 Lesson Learned

VTY configuration alone is not enough for SSH.
IOS SSH also requires a domain name and RSA keys.

---

## 12. Ubuntu OpenSSH Algorithm Compatibility

## 12.1 Symptom

MGMT-SRV1 attempted SSH to IOS devices but received a negotiation error.

Observed concept:

```text
Unable to negotiate with <DEVICE-IP> port 22:
no matching key exchange method found
```

### 12.2 Cause

Ubuntu OpenSSH disabled older SSH algorithms by default.
IOSv and IOSvL2 may still use legacy SSH algorithms.

### 12.3 Resolution

Use explicit OpenSSH options from MGMT-SRV1:

```bash
ssh -oKexAlgorithms=+diffie-hellman-group14-sha1 -oHostKeyAlgorithms=+ssh-rsa -oPubkeyAcceptedAlgorithms=+ssh-rsa admin@<DEVICE-IP>
```

Example:

```bash
ssh -oKexAlgorithms=+diffie-hellman-group14-sha1 -oHostKeyAlgorithms=+ssh-rsa -oPubkeyAcceptedAlgorithms=+ssh-rsa admin@10.10.40.1
```

### 12.4 Verification

Successful login from MGMT-SRV1 confirms:

```text
TCP/22 reachability works.
VTY-MGMT-ONLY ACL permits 10.10.40.10.
AAA/local authentication works.
SSH algorithm mismatch was a client/server compatibility issue.
```

### 12.5 Lesson Learned

Do not misinterpret SSH algorithm negotiation failure as a VTY ACL failure.
If the SSH server responds with algorithm negotiation errors, the management path is already reaching TCP/22.

---

## 13. VTY Management ACL Troubleshooting

## 13.1 Goal

Only MGMT-SRV1 should be allowed to SSH into network devices.

Allowed source:

```text
10.10.40.10
```

Denied source example:

```text
ROGUE-PC: 10.10.20.126
```

### 13.2 ACL

```cisco
ip access-list standard VTY-MGMT-ONLY
 permit host 10.10.40.10
 deny any log
```

Applied under VTY lines:

```cisco
line vty 0 4
 access-class VTY-MGMT-ONLY in
 transport input ssh
```

### 13.3 Successful Permit Validation

MGMT-SRV1 successfully SSHed into network devices.

Expected ACL result:

```text
permit host 10.10.40.10 hit count increases
```

### 13.4 Successful Deny Validation

ROGUE-PC attempted SSH from VLAN20.

Expected result:

```text
SSH denied
deny any log hit count increases
IP access log generated
```

Observed example on ACC1:

```text
permit 10.10.40.10 matched MGMT-SRV1 SSH access
deny any log matched ROGUE-PC SSH access from 10.10.20.126
```

Example log concept:

```text
%SEC-6-IPACCESSLOGNP:
list VTY-MGMT-ONLY denied 0 10.10.20.126 -> 0.0.0.0
```

### 13.5 Verification Commands

```cisco
show access-lists VTY-MGMT-ONLY
show logging | include VTY|SSH|SEC|denied|Deny|ACL
show run | section line vty
```

### 13.6 Lesson Learned

VTY ACL validation requires both positive and negative tests:

```text
Positive test:
- MGMT-SRV1 SSH succeeds.

Negative test:
- ROGUE-PC SSH fails.
- deny any log counter increases.
```

---

## 14. Access Switch Management SVI Troubleshooting

## 14.1 Symptom

ACCESS switches could not be SSHed using physical interface addresses.

### 14.2 Cause

ACCESS switch physical ports are Layer 2 switchports and do not have IP addresses.

A management SVI is required.

### 14.3 Resolution

Configured VLAN40 management SVIs.

Management IP plan:

```text
ACC1      10.10.40.11
ACC2      10.10.40.12
ACC3      10.10.40.13
SRV-ACC1  10.10.40.14
```

Example ACC1:

```cisco
conf t
interface vlan 40
 description MGMT-SVI
 ip address 10.10.40.11 255.255.255.0
 no shutdown
!
ip default-gateway 10.10.40.254
end
wr
```

### 14.4 Verification

```cisco
show ip interface brief | include Vlan40
show run interface vlan40
show run | include ip default-gateway
show vlan brief | include 40
show interfaces trunk
```

Expected result:

```text
Vlan40 is up/up
VLAN40 exists
VLAN40 is allowed on trunks
ip default-gateway is 10.10.40.254
```

### 14.5 Lesson Learned

Layer 2 access switches require:

```text
Management SVI
Default gateway
Reachable management VLAN
SSH and VTY configuration
Local username
VTY ACL
```

---

## 15. SERVER1 Static IP Persistence

## 15.1 Symptom

SERVER1 did not have an IPv4 address.

Observed condition:

```text
ens2 had MAC address but no IPv4 address.
No default route existed.
```

### 15.2 Resolution

SERVER1 was configured with a static IP in VLAN70.

Netplan file:

```bash
/etc/netplan/50-cloud-init.yaml
```

Configuration:

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

Apply:

```bash
sudo netplan apply
```

### 15.3 Verification

```bash
ip addr show ens2
ip route
ping -c 4 10.10.70.254
ping -c 4 10.10.40.10
```

Expected result:

```text
SERVER1 IP = 10.10.70.10/24
Default gateway = 10.10.70.254
Ping to gateway succeeds
Ping to MGMT-SRV1 succeeds
```

### 15.4 Lesson Learned

Static server endpoints must be verified before applying DAI or static source bindings.

---

## 16. Recommended Troubleshooting Order

Use the following order when troubleshooting Phase 04 security issues.

### 16.1 Port Security

```text
1. Check current MAC on endpoint.
2. Check sticky MAC on switch.
3. Check violation counter.
4. Check log messages.
5. Restore endpoint MAC or clear/relearn sticky MAC.
```

Commands:

```cisco
show port-security
show port-security interface <interface>
show port-security address
show logging | include PSECURE|PORT_SECURITY
```

---

### 16.2 DHCP Snooping

```text
1. Verify VLAN assignment.
2. Verify DHCP Snooping VLAN list.
3. Verify trusted uplinks.
4. Verify DHCP server-facing trusted port.
5. Verify CORE ip helper-address.
6. Verify DHCP packet arrival on MGMT-SRV1.
7. Verify binding table.
```

Commands:

```cisco
show ip dhcp snooping
show ip dhcp snooping binding
show ip dhcp snooping statistics
show logging | include DHCP|SNOOP|DROP
```

MGMT-SRV1:

```bash
sudo tcpdump -ni ens2 port 67 or port 68
```

---

### 16.3 Dynamic ARP Inspection

```text
1. Confirm DHCP Snooping binding exists.
2. Confirm DAI VLAN is enabled.
3. Confirm uplink is trusted.
4. Confirm access port is untrusted.
5. Check DAI statistics.
6. Check DAI logs.
```

Commands:

```cisco
show ip arp inspection
show ip arp inspection interfaces
show ip arp inspection statistics
show ip dhcp snooping binding
show logging | include ARP|DAI|INSPECTION
```

---

### 16.4 Static Server DAI

```text
1. Confirm static server IP.
2. Confirm static server MAC.
3. Configure static source binding.
4. Enable DAI.
5. Clear counters.
6. Test normal server communication.
7. Confirm no new drops.
```

Commands:

```cisco
show ip source binding
show ip arp inspection statistics
show logging | include ARP|DAI|INSPECTION
```

---

### 16.5 BPDU Guard

```text
1. Confirm access port has PortFast.
2. Confirm BPDU Guard is enabled.
3. Connect rogue switch for validation.
4. Confirm err-disabled reason.
5. Remove rogue switch before recovery.
6. Recover port with shutdown/no shutdown.
```

Commands:

```cisco
show interface status err-disabled
show logging | include BPDU|SPANTREE|ERR|err
show run interface <interface>
```

---

### 16.6 Management ACL

```text
1. Confirm management SVI/IP reachability.
2. Confirm local username exists.
3. Confirm SSH is enabled.
4. Confirm VTY ACL is applied.
5. Test SSH from MGMT-SRV1.
6. Test SSH from ROGUE-PC.
7. Check ACL hit counts and logs.
```

Commands:

```cisco
show access-lists VTY-MGMT-ONLY
show run | section line vty
show ip ssh
show logging | include VTY|SSH|SEC|denied|Deny|ACL
```

---

## 17. Final Troubleshooting Notes

Phase 04 completed the following validated troubleshooting outcomes:

```text
Port Security violation detected and logged.
DHCP Snooping binding validated.
DAI spoofed ARP drop validated.
Static source bindings allowed DAI for static servers.
IP Source Guard was validated but not retained due to CML IOSvL2 behavior.
BPDU Guard successfully err-disabled rogue switch port.
Management ACL allowed MGMT-SRV1 and denied ROGUE-PC.
ACCESS switch SSH login issue fixed by adding local admin user.
EDGE1 SSH issue fixed by generating RSA keys.
Ubuntu OpenSSH compatibility handled with explicit SSH options.
```

---

## 18. What Not to Misinterpret

Do not misinterpret the following:

```text
1. Port Security violation counters may remain after testing.
   - This does not mean the port is currently broken.

2. BPDU Guard err-disabled state on SRV-ACC1 Gi0/3 is expected after rogue switch validation.
   - It proves the control worked.

3. SSH algorithm negotiation failure is not the same as ACL failure.
   - It means the SSH session reached the device but failed during algorithm negotiation.

4. DAI drops on static server VLANs are expected if static bindings are missing.
   - Add static source bindings before enforcing DAI.

5. IP Source Guard correct output does not guarantee correct CML IOSvL2 data-plane forwarding.
   - Validate with real traffic.
```

---

## 19. Final Status

Phase 04 Security troubleshooting is complete.

Final result:

```text
All required Phase 04 security controls were configured, tested, or validated.

IP Source Guard was validated but not retained because of CML IOSvL2 behavior.

All other controls were retained in the final Phase 04 configuration.
```
