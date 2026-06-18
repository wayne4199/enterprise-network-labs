# Phase 04 Security - Configs

## 1. Scope

This document records the final security-related configurations for Phase 04.

Phase 04 covered:

- Access port hardening
- Port Security
- DHCP Snooping
- Dynamic ARP Inspection
- Static Source Binding for statically addressed servers
- IP Source Guard validation
- BPDU Guard
- Management ACL / VTY SSH Access Control

> Note:
> IP Source Guard was validated on ACC2 Gi1/1, but it was not retained on active access ports because the CML IOSvL2 data plane dropped host traffic even when the DHCP Snooping binding and IP/MAC validation output were correct.

---

## 2. Common Security Notes

### 2.1 Password Handling

Do not store real passwords in GitHub.

Use placeholders in documentation:

```cisco
username admin privilege 15 secret <ADMIN_PASSWORD>
```

### 2.2 Management Source

Only the Enterprise Management Server is allowed to access network devices through SSH.

```text
MGMT-SRV1: 10.10.40.10
```

### 2.3 VTY Management ACL

Common ACL used across all network devices:

```cisco
ip access-list standard VTY-MGMT-ONLY
 permit host 10.10.40.10
 deny any log
```

---

## 3. CORE1

CORE1 uses AAA local authentication for SSH management.

```cisco
! ============================================================
! CORE1 - Management ACL / SSH / AAA Local
! ============================================================

conf t
!
username admin privilege 15 secret <ADMIN_PASSWORD>
!
aaa new-model
aaa authentication login default local
aaa authorization exec default local
aaa session-id common
!
ip ssh version 2
ip ssh time-out 60
ip ssh authentication-retries 3
!
ip access-list standard VTY-MGMT-ONLY
 permit host 10.10.40.10
 deny any log
!
line vty 0 4
 access-class VTY-MGMT-ONLY in
 exec-timeout 0 0
 transport input ssh
!
line vty 5 15
 access-class VTY-MGMT-ONLY in
 transport input ssh
!
end
wr
```

Verification commands:

```cisco
show run | include ^aaa|^username|ip ssh|ip domain-name
show access-lists VTY-MGMT-ONLY
show run | section line vty
show ip ssh
```

---

## 4. CORE2

CORE2 uses AAA local authentication for SSH management.

```cisco
! ============================================================
! CORE2 - Management ACL / SSH / AAA Local
! ============================================================

conf t
!
username admin privilege 15 secret <ADMIN_PASSWORD>
!
aaa new-model
aaa authentication login default local
aaa authorization exec default local
aaa session-id common
!
ip ssh version 2
ip ssh time-out 60
ip ssh authentication-retries 3
!
ip access-list standard VTY-MGMT-ONLY
 permit host 10.10.40.10
 deny any log
!
line vty 0 4
 access-class VTY-MGMT-ONLY in
 exec-timeout 0 0
 transport input ssh
!
line vty 5 15
 access-class VTY-MGMT-ONLY in
 transport input ssh
!
end
wr
```

Verification commands:

```cisco
show run | include ^aaa|^username|ip ssh|ip domain-name
show access-lists VTY-MGMT-ONLY
show run | section line vty
show ip ssh
```

---

## 5. EDGE1

EDGE1 required RSA key generation before SSH could be enabled.

```cisco
! ============================================================
! EDGE1 - Management ACL / SSH / AAA Local
! ============================================================

conf t
!
username admin privilege 15 secret <ADMIN_PASSWORD>
!
aaa new-model
aaa authentication login default local
aaa authorization exec default local
aaa session-id common
!
ip domain-name campus.lab
!
crypto key generate rsa modulus 2048
!
ip ssh version 2
ip ssh time-out 60
ip ssh authentication-retries 3
!
ip access-list standard VTY-MGMT-ONLY
 permit host 10.10.40.10
 deny any log
!
line vty 0 4
 access-class VTY-MGMT-ONLY in
 exec-timeout 0 0
 transport input ssh
!
line vty 5 15
 access-class VTY-MGMT-ONLY in
 transport input ssh
!
end
wr
```

Verification commands:

```cisco
show ip ssh
show crypto key mypubkey rsa
show run | include ^aaa|^username|ip ssh|ip domain-name
show access-lists VTY-MGMT-ONLY
show run | section line vty
```

Management access IPs used for EDGE1:

```text
Primary campus-facing IP: 172.16.0.1
Secondary campus-facing IP: 172.16.0.5
```

WAN-facing IPs were not used for management validation:

```text
100.64.1.2
100.64.2.2
```

---

## 6. ACC1

### 6.1 Management SVI and SSH Access

```cisco
! ============================================================
! ACC1 - Management SVI / SSH / VTY ACL
! ============================================================

conf t
!
username admin privilege 15 secret <ADMIN_PASSWORD>
!
interface vlan 40
 description MGMT-SVI
 ip address 10.10.40.11 255.255.255.0
 no shutdown
!
ip default-gateway 10.10.40.254
!
ip access-list standard VTY-MGMT-ONLY
 permit host 10.10.40.10
 deny any log
!
line vty 0 4
 access-class VTY-MGMT-ONLY in
 exec-timeout 0 0
 login local
 transport input ssh
!
line vty 5 15
 access-class VTY-MGMT-ONLY in
 login local
 transport input ssh
!
end
wr
```

### 6.2 DHCP Snooping

```cisco
! ============================================================
! ACC1 - DHCP Snooping
! ============================================================

conf t
!
ip dhcp snooping vlan 10-11,20-21,30-31,40,50,60,70
no ip dhcp snooping information option
ip dhcp snooping
!
interface range g0/0-1
 ip dhcp snooping trust
!
interface range g0/2-3, g1/0
 ip dhcp snooping limit rate 5
!
end
wr
```

### 6.3 Dynamic ARP Inspection

```cisco
! ============================================================
! ACC1 - Dynamic ARP Inspection
! ============================================================

conf t
!
ip arp inspection vlan 10,11,60
ip arp inspection validate src-mac dst-mac ip
!
interface range g0/0-1
 ip arp inspection trust
!
interface range g0/2-3, g1/0
 ip arp inspection limit rate 15
!
end
wr
```

### 6.4 Access Port Security and BPDU Guard

```cisco
! ============================================================
! ACC1 - Access Port Security / BPDU Guard
! ============================================================

conf t
!
interface g0/2
 description EXEC
 switchport access vlan 10
 switchport mode access
 switchport nonegotiate
 switchport port-security violation restrict
 switchport port-security mac-address sticky
 switchport port-security mac-address sticky 5254.001a.91b4
 switchport port-security
 spanning-tree portfast edge
 spanning-tree bpduguard enable
 ip dhcp snooping limit rate 5
 ip arp inspection limit rate 15
!
interface g0/3
 description BIZ-ADMIN
 switchport access vlan 11
 switchport mode access
 switchport nonegotiate
 switchport port-security violation restrict
 switchport port-security mac-address sticky
 switchport port-security mac-address sticky 5254.00d7.4a2c
 switchport port-security
 spanning-tree portfast edge
 spanning-tree bpduguard enable
 ip dhcp snooping limit rate 5
 ip arp inspection limit rate 15
!
interface g1/0
 description PRINTER
 switchport access vlan 60
 switchport mode access
 switchport nonegotiate
 switchport port-security violation restrict
 switchport port-security mac-address sticky
 switchport port-security mac-address sticky 5254.00a6.378b
 switchport port-security
 spanning-tree portfast edge
 spanning-tree bpduguard enable
 ip dhcp snooping limit rate 5
 ip arp inspection limit rate 15
!
end
wr
```

---

## 7. ACC2

### 7.1 Management SVI and SSH Access

```cisco
! ============================================================
! ACC2 - Management SVI / SSH / VTY ACL
! ============================================================

conf t
!
username admin privilege 15 secret <ADMIN_PASSWORD>
!
interface vlan 40
 description MGMT-SVI
 ip address 10.10.40.12 255.255.255.0
 no shutdown
!
ip default-gateway 10.10.40.254
!
ip access-list standard VTY-MGMT-ONLY
 permit host 10.10.40.10
 deny any log
!
line vty 0 4
 access-class VTY-MGMT-ONLY in
 exec-timeout 0 0
 login local
 transport input ssh
!
line vty 5 15
 access-class VTY-MGMT-ONLY in
 login local
 transport input ssh
!
end
wr
```

### 7.2 DHCP Snooping

```cisco
! ============================================================
! ACC2 - DHCP Snooping
! ============================================================

conf t
!
ip dhcp snooping vlan 10-11,20-21,30-31,40,50,60,70
no ip dhcp snooping information option
ip dhcp snooping
!
interface range g0/0-1, g1/0
 ip dhcp snooping trust
!
interface range g0/2-3, g1/1
 ip dhcp snooping limit rate 5
!
end
wr
```

### 7.3 Dynamic ARP Inspection

```cisco
! ============================================================
! ACC2 - Dynamic ARP Inspection
! ============================================================

conf t
!
ip arp inspection vlan 20,21
ip arp inspection validate src-mac dst-mac ip
!
interface range g0/0-1, g1/0
 ip arp inspection trust
!
interface range g0/2-3, g1/1
 ip arp inspection limit rate 15
!
end
wr
```

### 7.4 Access Port Security and BPDU Guard

```cisco
! ============================================================
! ACC2 - Access Port Security / BPDU Guard
! ============================================================

conf t
!
interface g0/2
 description SALES
 switchport access vlan 20
 switchport mode access
 switchport nonegotiate
 switchport port-security violation restrict
 switchport port-security mac-address sticky
 switchport port-security mac-address sticky 5254.0040.dc2c
 switchport port-security
 spanning-tree portfast edge
 spanning-tree bpduguard enable
 ip dhcp snooping limit rate 5
 ip arp inspection limit rate 15
!
interface g0/3
 description MKT
 switchport access vlan 21
 switchport mode access
 switchport nonegotiate
 switchport port-security violation restrict
 switchport port-security mac-address sticky
 switchport port-security mac-address sticky 5254.0035.d673
 switchport port-security
 spanning-tree portfast edge
 spanning-tree bpduguard enable
 ip dhcp snooping limit rate 5
 ip arp inspection limit rate 15
!
interface g1/1
 description ROGUE-PC-TEST
 switchport access vlan 20
 switchport mode access
 switchport nonegotiate
 switchport port-security violation restrict
 switchport port-security mac-address sticky
 switchport port-security mac-address sticky 5254.0061.b0c0
 switchport port-security
 spanning-tree portfast edge
 spanning-tree bpduguard enable
 ip dhcp snooping limit rate 5
 ip arp inspection limit rate 15
!
end
wr
```

### 7.5 IP Source Guard Validation Only

IP Source Guard was tested on ACC2 Gi1/1, but it was not retained.

Tested commands:

```cisco
conf t
!
interface g1/1
 ip verify source
!
end
```

Then IP/MAC mode with Port Security was tested:

```cisco
conf t
!
interface g1/1
 no ip verify source
 ip verify source port-security
!
end
```

Observed output:

```text
Interface  Filter-type  Filter-mode  IP-address    Mac-address           Vlan
Gi1/1      ip-mac       active       10.10.20.126  52:54:00:61:B0:C0    20
```

However, host traffic was dropped in the CML IOSvL2 data plane even though the binding was correct.

Final retained configuration:

```cisco
conf t
!
interface g1/1
 no ip verify source
!
end
wr
```

Reason:

```text
IP Source Guard was validated but not retained on active access ports because enabling it caused traffic loss in the CML IOSvL2 environment despite correct DHCP Snooping and Port Security bindings.
```

---

## 8. ACC3

### 8.1 Management SVI and SSH Access

```cisco
! ============================================================
! ACC3 - Management SVI / SSH / VTY ACL
! ============================================================

conf t
!
username admin privilege 15 secret <ADMIN_PASSWORD>
!
interface vlan 40
 description MGMT-SVI
 ip address 10.10.40.13 255.255.255.0
 no shutdown
!
ip default-gateway 10.10.40.254
!
ip access-list standard VTY-MGMT-ONLY
 permit host 10.10.40.10
 deny any log
!
line vty 0 4
 access-class VTY-MGMT-ONLY in
 exec-timeout 0 0
 login local
 transport input ssh
!
line vty 5 15
 access-class VTY-MGMT-ONLY in
 login local
 transport input ssh
!
end
wr
```

### 8.2 DHCP Snooping

```cisco
! ============================================================
! ACC3 - DHCP Snooping
! ============================================================

conf t
!
ip dhcp snooping vlan 10-11,20-21,30-31,40,50,60,70
no ip dhcp snooping information option
ip dhcp snooping
!
interface range g0/0-1
 ip dhcp snooping trust
!
interface range g0/2-3
 ip dhcp snooping limit rate 5
!
end
wr
```

### 8.3 Dynamic ARP Inspection

```cisco
! ============================================================
! ACC3 - Dynamic ARP Inspection
! ============================================================

conf t
!
ip arp inspection vlan 30,31
ip arp inspection validate src-mac dst-mac ip
!
interface range g0/0-1
 ip arp inspection trust
!
interface range g0/2-3
 ip arp inspection limit rate 15
!
end
wr
```

### 8.4 Access Port Security and BPDU Guard

```cisco
! ============================================================
! ACC3 - Access Port Security / BPDU Guard
! ============================================================

conf t
!
interface g0/2
 description IT-OPS
 switchport access vlan 30
 switchport mode access
 switchport nonegotiate
 switchport port-security violation restrict
 switchport port-security mac-address sticky
 switchport port-security mac-address sticky 5254.00be.16c1
 switchport port-security
 spanning-tree portfast edge
 spanning-tree bpduguard enable
 ip dhcp snooping limit rate 5
 ip arp inspection limit rate 15
!
interface g0/3
 description IT-SEC
 switchport access vlan 31
 switchport mode access
 switchport nonegotiate
 switchport port-security violation restrict
 switchport port-security mac-address sticky
 switchport port-security mac-address sticky 5254.00ba.87dc
 switchport port-security
 spanning-tree portfast edge
 spanning-tree bpduguard enable
 ip dhcp snooping limit rate 5
 ip arp inspection limit rate 15
!
end
wr
```

---

## 9. SRV-ACC1

### 9.1 Management SVI and SSH Access

```cisco
! ============================================================
! SRV-ACC1 - Management SVI / SSH / VTY ACL
! ============================================================

conf t
!
username admin privilege 15 secret <ADMIN_PASSWORD>
!
interface vlan 40
 description MGMT-SVI
 ip address 10.10.40.14 255.255.255.0
 no shutdown
!
ip default-gateway 10.10.40.254
!
ip access-list standard VTY-MGMT-ONLY
 permit host 10.10.40.10
 deny any log
!
line vty 0 4
 access-class VTY-MGMT-ONLY in
 exec-timeout 0 0
 login local
 transport input ssh
!
line vty 5 15
 access-class VTY-MGMT-ONLY in
 login local
 transport input ssh
!
end
wr
```

### 9.2 DHCP Snooping

SRV-ACC1 has a DHCP server connected on Gi0/2. Therefore, Gi0/2 is trusted for DHCP Snooping.

```cisco
! ============================================================
! SRV-ACC1 - DHCP Snooping
! ============================================================

conf t
!
ip dhcp snooping vlan 10-11,20-21,30-31,40,50,60,70
no ip dhcp snooping information option
ip dhcp snooping
!
interface g0/0
 description TRUNK-TO-ACC2
 ip dhcp snooping trust
!
interface g0/1
 description SERVER1
 ip dhcp snooping limit rate 5
!
interface g0/2
 description MGMT-SRV1
 ip dhcp snooping trust
!
interface g0/3
 description ROGUE-SW-TEST
 ip dhcp snooping limit rate 5
!
end
wr
```

### 9.3 Static Source Binding

SRV-ACC1 has statically addressed servers. Static source bindings were configured before enabling DAI on VLAN 40 and VLAN 70.

```cisco
! ============================================================
! SRV-ACC1 - Static Source Binding
! ============================================================

conf t
!
ip source binding 5254.000d.6061 vlan 40 10.10.40.10 interface Gi0/2
ip source binding 5254.00be.8bee vlan 70 10.10.70.10 interface Gi0/1
!
end
wr
```

Final static bindings:

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

### 9.4 Dynamic ARP Inspection

Gi0/0 is the uplink/trunk and is trusted for DAI.  
Server-facing ports remain untrusted and are validated using static source bindings.

```cisco
! ============================================================
! SRV-ACC1 - Dynamic ARP Inspection
! ============================================================

conf t
!
ip arp inspection vlan 40,70
ip arp inspection validate src-mac dst-mac ip
!
interface g0/0
 ip arp inspection trust
!
interface range g0/1-3
 ip arp inspection limit rate 15
!
end
wr
```

### 9.5 Access Port Security and BPDU Guard

```cisco
! ============================================================
! SRV-ACC1 - Access Port Security / BPDU Guard
! ============================================================

conf t
!
interface g0/1
 description SERVER1
 switchport access vlan 70
 switchport mode access
 switchport nonegotiate
 switchport port-security violation restrict
 switchport port-security mac-address sticky
 switchport port-security mac-address sticky 5254.00be.8bee
 switchport port-security
 spanning-tree portfast edge
 spanning-tree bpduguard enable
 ip dhcp snooping limit rate 5
 ip arp inspection limit rate 15
!
interface g0/2
 description MGMT-SRV1
 switchport access vlan 40
 switchport mode access
 switchport nonegotiate
 switchport port-security violation restrict
 switchport port-security mac-address sticky
 switchport port-security mac-address sticky 5254.000d.6061
 switchport port-security
 spanning-tree portfast edge
 spanning-tree bpduguard enable
 ip dhcp snooping trust
 ip arp inspection limit rate 15
!
interface g0/3
 description ROGUE-SW-TEST
 switchport mode access
 switchport nonegotiate
 switchport port-security violation restrict
 switchport port-security mac-address sticky
 switchport port-security mac-address sticky 5254.00b2.282d
 switchport port-security
 spanning-tree portfast edge
 spanning-tree bpduguard enable
 ip dhcp snooping limit rate 5
 ip arp inspection limit rate 15
!
end
wr
```

Note:

```text
SRV-ACC1 Gi0/3 was intentionally used for the BPDU Guard test.
When ROGUE-SW-TEST sent BPDUs, Gi0/3 was placed into err-disabled state with reason bpduguard.
```

---

## 10. SERVER1 Netplan Configuration

SERVER1 is statically addressed in VLAN 70.

File:

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

Verify:

```bash
ip addr show ens2
ip route
ping -c 4 10.10.70.254
ping -c 4 10.10.40.10
```

Expected result:

```text
SERVER1:
- IP: 10.10.70.10/24
- Default gateway: 10.10.70.254
- DNS server: 10.10.40.10
- Ping to gateway succeeds
- Ping to MGMT-SRV1 succeeds
```

---

## 11. SSH Client Command Used from MGMT-SRV1

Ubuntu OpenSSH rejected older IOS SSH algorithms by default.  
The following options were used to validate SSH access to IOSv/IOSvL2 devices.

```bash
ssh -oKexAlgorithms=+diffie-hellman-group14-sha1 -oHostKeyAlgorithms=+ssh-rsa -oPubkeyAcceptedAlgorithms=+ssh-rsa admin@<DEVICE-IP>
```

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

---

## 12. Final Verification Commands

### 12.1 Access Switches

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

### 12.2 ACC2 Additional Validation

```cisco
show port-security interface g1/1
show ip verify source
show logging | include PSECURE|PORT_SECURITY|DAI|ARP|DHCP|SNOOP|VERIFY|SOURCE
```

Expected IP Source Guard final state:

```text
No active IP Source Guard entry is retained on Gi1/1.
```

### 12.3 SRV-ACC1 Additional Validation

```cisco
show ip source binding
show ip arp inspection statistics
show interface status err-disabled
show logging | include BPDU|SPANTREE|ERR|err
```

Expected results:

```text
Static source bindings exist for MGMT-SRV1 and SERVER1.
VLAN 40 and VLAN 70 DAI are active.
Gi0/3 may be err-disabled due to intentional BPDU Guard validation.
```

### 12.4 CORE1 / CORE2 / EDGE1

```cisco
show run | include ^aaa|^username|ip ssh|ip domain-name
show access-lists VTY-MGMT-ONLY
show run | section line vty
show ip ssh
```

EDGE1 additional verification:

```cisco
show crypto key mypubkey rsa
```

---

## 13. Final Security State

```text
Port Security:
- Enabled on access-facing ports.
- Sticky MAC learning used.
- Violation mode: restrict.

DHCP Snooping:
- Enabled on access-layer switches.
- Uplinks and DHCP server-facing port trusted.
- User-facing ports untrusted with rate limit.

Dynamic ARP Inspection:
- Enabled on user VLANs at access switches.
- Enabled on server VLANs at SRV-ACC1 using static source bindings.
- Uplinks trusted.
- Host/server ports untrusted and validated.

Static Source Binding:
- Used for statically addressed servers on SRV-ACC1.
- Required for safe DAI and future IP Source Guard consistency.

IP Source Guard:
- Validated on ACC2 Gi1/1.
- Not retained due to CML IOSvL2 forwarding limitation.

BPDU Guard:
- Enabled on access-facing PortFast edge ports.
- ROGUE-SW-TEST successfully triggered BPDU Guard on SRV-ACC1 Gi0/3.

Management ACL:
- SSH access allowed only from MGMT-SRV1 10.10.40.10.
- ROGUE-PC from VLAN 20 was denied by VTY-MGMT-ONLY ACL.
```
