# Phase 02 — Enterprise Services Ver.2 Configs

## 1. Purpose

This document records the main configuration used in **Phase 02 — Enterprise Services Ver.2** of the Collapsed Core Campus lab.

Phase 02 builds centralized enterprise services using **MGMT-SRV1**.

MGMT-SRV1 is not a simple standalone server.
MGMT-SRV1 functions as the **Enterprise Management Server** for the campus network.

It centralizes:

* DHCP
* DNS Resolver
* NTP
* Syslog
* SNMP Monitoring Tools
* Config Backup Storage

This document includes the practical configuration applied during the lab.

---

## 2. Important Notes

### Password Handling

Do not commit real passwords to GitHub.

Any password shown in this document should be treated as a placeholder.

```text
<ADMIN_PASSWORD>
```

### Device Scope

Primary infrastructure devices configured in this phase:

```text
CORE1
CORE2
EDGE1
ISP1
ISP2
MGMT-SRV1
SERVER1
```

MGMT-SRV1 provides the centralized enterprise services.

---

## 3. IP Address Summary

| Device / Interface           |     IP Address | Purpose                              |
| ---------------------------- | -------------: | ------------------------------------ |
| MGMT-SRV1 ens2               | 10.10.40.10/24 | Enterprise Management Server         |
| MGMT-SRV1 gateway            |   10.10.40.254 | Management VLAN gateway              |
| SERVER1 ens2                 | 10.10.70.10/24 | Server VLAN test node                |
| CORE1 VLAN40                 |  10.10.40.1/24 | Core management SVI                  |
| CORE2 VLAN40                 |  10.10.40.2/24 | Core management SVI                  |
| CORE1 VLAN10                 |  10.10.10.1/24 | VLAN10 gateway member                |
| CORE2 VLAN10                 |  10.10.10.2/24 | VLAN10 gateway member                |
| VLAN10 HSRP VIP              |   10.10.10.254 | Client gateway                       |
| EDGE1 G0/0                   |  100.64.1.2/30 | To ISP1                              |
| EDGE1 G0/1                   |  100.64.2.2/30 | To ISP2                              |
| EDGE1 G0/2                   |  172.16.0.1/30 | To CORE1                             |
| EDGE1 G0/3                   |  172.16.0.5/30 | To CORE2                             |
| ISP1 G0/1                    |  100.64.1.1/30 | To EDGE1                             |
| ISP2 G0/1                    |  100.64.2.1/30 | To EDGE1                             |
| ISP1 / ISP2 Internet segment | 203.0.113.0/24 | Simulated Internet segment           |
| internet-srv.campus.lab      |   203.0.113.10 | Simulated Internet server DNS record |

---

## 4. MGMT-SRV1 — Base Network Configuration

MGMT-SRV1 uses Ubuntu Server.

Interface used:

```text
ens2
```

Expected IP configuration:

```bash
ip -br addr
ip route
```

Expected result:

```text
ens2 UP 10.10.40.10/24
default via 10.10.40.254 dev ens2
```

If manual configuration is needed, use the Ubuntu network configuration method available in the lab image.

The final expected state:

```text
MGMT-SRV1 IP      : 10.10.40.10/24
Default Gateway  : 10.10.40.254
DNS Server       : 10.10.40.10
Domain           : campus.lab
```

---

## 5. MGMT-SRV1 — Required Packages

Install the required tools and services.

```bash
sudo NEEDRESTART_MODE=a apt update
sudo NEEDRESTART_MODE=a apt install -y dnsutils traceroute curl tcpdump
sudo NEEDRESTART_MODE=a apt install -y dnsmasq
sudo NEEDRESTART_MODE=a apt install -y chrony
sudo NEEDRESTART_MODE=a apt install -y snmp
```

Verify installed tools:

```bash
which dig
which curl
which traceroute
which tcpdump
which dnsmasq
which chronyd
which snmpwalk
which snmpget
```

Expected examples:

```text
/usr/bin/dig
/usr/bin/curl
/usr/sbin/traceroute
/usr/bin/tcpdump
/usr/sbin/dnsmasq
/usr/sbin/chronyd
/usr/bin/snmpwalk
/usr/bin/snmpget
```

---

## 6. MGMT-SRV1 — dnsmasq DHCP/DNS Configuration

Configuration file:

```text
/etc/dnsmasq.d/campus-dns.conf
```

Create or edit the file:

```bash
sudo nano /etc/dnsmasq.d/campus-dns.conf
```

Final configuration:

```conf
# Collapsed Core Campus Ver.2 - Phase 02 DNS Resolver
# MGMT-SRV1: 10.10.40.10

interface=ens2
listen-address=10.10.40.10
bind-interfaces

no-resolv
server=8.8.8.8
server=1.1.1.1
filter-AAAA

domain=campus.lab
local=/campus.lab/

cache-size=1000

# ------------------------------------------------------------
# DNS Static Records
# ------------------------------------------------------------

address=/mgmt-srv1.campus.lab/10.10.40.10
address=/server1.campus.lab/10.10.70.10
address=/internet-srv.campus.lab/203.0.113.10

address=/dns.campus.lab/10.10.40.10
address=/ntp.campus.lab/10.10.40.10
address=/syslog.campus.lab/10.10.40.10

# ------------------------------------------------------------
# DHCP Scope - VLAN10
# ------------------------------------------------------------

dhcp-range=set:vlan10,10.10.10.100,10.10.10.199,255.255.255.0,12h
dhcp-option=tag:vlan10,option:router,10.10.10.254
dhcp-option=tag:vlan10,option:dns-server,10.10.40.10
dhcp-option=tag:vlan10,option:domain-name,campus.lab

# ------------------------------------------------------------
# DHCP Scope - VLAN11
# ------------------------------------------------------------

dhcp-range=set:vlan11,10.10.11.100,10.10.11.199,255.255.255.0,12h
dhcp-option=tag:vlan11,option:router,10.10.11.254
dhcp-option=tag:vlan11,option:dns-server,10.10.40.10
dhcp-option=tag:vlan11,option:domain-name,campus.lab

# ------------------------------------------------------------
# DHCP Scope - VLAN20
# ------------------------------------------------------------

dhcp-range=set:vlan20,10.10.20.100,10.10.20.199,255.255.255.0,12h
dhcp-option=tag:vlan20,option:router,10.10.20.254
dhcp-option=tag:vlan20,option:dns-server,10.10.40.10
dhcp-option=tag:vlan20,option:domain-name,campus.lab

# ------------------------------------------------------------
# DHCP Scope - VLAN21
# ------------------------------------------------------------

dhcp-range=set:vlan21,10.10.21.100,10.10.21.199,255.255.255.0,12h
dhcp-option=tag:vlan21,option:router,10.10.21.254
dhcp-option=tag:vlan21,option:dns-server,10.10.40.10
dhcp-option=tag:vlan21,option:domain-name,campus.lab

# ------------------------------------------------------------
# DHCP Scope - VLAN30
# ------------------------------------------------------------

dhcp-range=set:vlan30,10.10.30.100,10.10.30.199,255.255.255.0,12h
dhcp-option=tag:vlan30,option:router,10.10.30.254
dhcp-option=tag:vlan30,option:dns-server,10.10.40.10
dhcp-option=tag:vlan30,option:domain-name,campus.lab

# ------------------------------------------------------------
# DHCP Scope - VLAN31
# ------------------------------------------------------------

dhcp-range=set:vlan31,10.10.31.100,10.10.31.199,255.255.255.0,12h
dhcp-option=tag:vlan31,option:router,10.10.31.254
dhcp-option=tag:vlan31,option:dns-server,10.10.40.10
dhcp-option=tag:vlan31,option:domain-name,campus.lab

# ------------------------------------------------------------
# DHCP Scope - VLAN50
# ------------------------------------------------------------

dhcp-range=set:vlan50,10.10.50.100,10.10.50.199,255.255.255.0,12h
dhcp-option=tag:vlan50,option:router,10.10.50.254
dhcp-option=tag:vlan50,option:dns-server,10.10.40.10
dhcp-option=tag:vlan50,option:domain-name,campus.lab

# ------------------------------------------------------------
# DHCP Scope - VLAN60
# ------------------------------------------------------------

dhcp-range=set:vlan60,10.10.60.100,10.10.60.199,255.255.255.0,12h
dhcp-option=tag:vlan60,option:router,10.10.60.254
dhcp-option=tag:vlan60,option:dns-server,10.10.40.10
dhcp-option=tag:vlan60,option:domain-name,campus.lab

# ------------------------------------------------------------
# DHCP Reservations
# ------------------------------------------------------------

# EXEC - VLAN10 DHCP Reservation
dhcp-host=52:54:00:71:87:ac,10.10.10.136,exec,12h
```

Test and restart dnsmasq:

```bash
sudo dnsmasq --test
sudo systemctl restart dnsmasq
systemctl status dnsmasq --no-pager
```

Expected result:

```text
dnsmasq: syntax check OK.
Active: active (running)
```

Check listening ports:

```bash
sudo ss -lnup | grep ':53'
sudo ss -lnup | grep ':67'
```

Expected:

```text
10.10.40.10:53
0.0.0.0%ens2:67
```

---

## 7. MGMT-SRV1 — dnsmasq Lease Verification

View active leases:

```bash
cat /var/lib/misc/dnsmasq.leases
```

Example observed lease:

```text
52:54:00:71:87:ac 10.10.10.136
```

DHCP reservation verification from client:

```bash
sudo ip addr flush dev eth0
sudo udhcpc -i eth0
ip addr
ip route
```

Expected result:

```text
lease of 10.10.10.136 obtained from 10.10.40.10
10.10.10.136/24
default via 10.10.10.254
```

---

## 8. MGMT-SRV1 — DNS Verification Commands

External DNS forwarding:

```bash
dig @10.10.40.10 google.com A
```

Internal static records:

```bash
dig @10.10.40.10 server1.campus.lab A
dig @10.10.40.10 mgmt-srv1.campus.lab A
dig @10.10.40.10 internet-srv.campus.lab A
```

Expected records:

```text
server1.campus.lab      A 10.10.70.10
mgmt-srv1.campus.lab    A 10.10.40.10
internet-srv.campus.lab A 203.0.113.10
```

---

## 9. CORE1 — DHCP Relay and SVI Configuration

CORE1 DHCP relay is configured on VLAN interfaces using MGMT-SRV1 as the DHCP server.

```cisco
interface Vlan10
 ip address 10.10.10.1 255.255.255.0
 ip helper-address 10.10.40.10
 standby 10 ip 10.10.10.254
 standby 10 priority 110
 standby 10 preempt

interface Vlan11
 ip address 10.10.11.1 255.255.255.0
 ip helper-address 10.10.40.10
 standby 11 ip 10.10.11.254
 standby 11 priority 110
 standby 11 preempt

interface Vlan20
 ip address 10.10.20.1 255.255.255.0
 ip helper-address 10.10.40.10
 standby 20 ip 10.10.20.254

interface Vlan21
 ip address 10.10.21.1 255.255.255.0
 ip helper-address 10.10.40.10
 standby 21 ip 10.10.21.254

interface Vlan30
 ip address 10.10.30.1 255.255.255.0
 ip helper-address 10.10.40.10
 standby 30 ip 10.10.30.254
 standby 30 priority 110
 standby 30 preempt

interface Vlan31
 ip address 10.10.31.1 255.255.255.0
 ip helper-address 10.10.40.10
 standby 31 ip 10.10.31.254

interface Vlan40
 ip address 10.10.40.1 255.255.255.0

interface Vlan50
 ip address 10.10.50.1 255.255.255.0
 ip helper-address 10.10.40.10
 standby 50 ip 10.10.50.254

interface Vlan60
 ip address 10.10.60.1 255.255.255.0
 ip helper-address 10.10.40.10
 standby 60 ip 10.10.60.254
 standby 60 priority 110
 standby 60 preempt

interface Vlan70
 ip address 10.10.70.1 255.255.255.0
```

Verification:

```cisco
show running-config interface vlan10
show running-config interface vlan11
show running-config interface vlan20
show running-config interface vlan21
show running-config interface vlan30
show running-config interface vlan31
show running-config interface vlan50
show running-config interface vlan60
```

---

## 10. CORE2 — DHCP Relay and SVI Configuration

```cisco
interface Vlan10
 ip address 10.10.10.2 255.255.255.0
 ip helper-address 10.10.40.10
 standby 10 ip 10.10.10.254

interface Vlan11
 ip address 10.10.11.2 255.255.255.0
 ip helper-address 10.10.40.10
 standby 11 ip 10.10.11.254

interface Vlan20
 ip address 10.10.20.2 255.255.255.0
 ip helper-address 10.10.40.10
 standby 20 ip 10.10.20.254
 standby 20 priority 110
 standby 20 preempt

interface Vlan21
 ip address 10.10.21.2 255.255.255.0
 ip helper-address 10.10.40.10
 standby 21 ip 10.10.21.254
 standby 21 priority 110
 standby 21 preempt

interface Vlan30
 ip address 10.10.30.2 255.255.255.0
 ip helper-address 10.10.40.10
 standby 30 ip 10.10.30.254

interface Vlan31
 ip address 10.10.31.2 255.255.255.0
 ip helper-address 10.10.40.10
 standby 31 ip 10.10.31.254
 standby 31 priority 110
 standby 31 preempt

interface Vlan40
 ip address 10.10.40.2 255.255.255.0

interface Vlan50
 ip address 10.10.50.2 255.255.255.0
 ip helper-address 10.10.40.10
 standby 50 ip 10.10.50.254
 standby 50 priority 110
 standby 50 preempt

interface Vlan60
 ip address 10.10.60.2 255.255.255.0
 ip helper-address 10.10.40.10
 standby 60 ip 10.10.60.254

interface Vlan70
 ip address 10.10.70.2 255.255.255.0
```

Verification:

```cisco
show running-config interface vlan10
show running-config interface vlan11
show running-config interface vlan20
show running-config interface vlan21
show running-config interface vlan30
show running-config interface vlan31
show running-config interface vlan50
show running-config interface vlan60
```

---

## 11. EDGE1 — NAT/PAT Configuration

EDGE1 standard NAT ACL:

```cisco
ip access-list standard ENTERPRISE-NAT
 permit 10.10.0.0 0.0.255.255
```

EDGE1 interface NAT roles:

```cisco
interface GigabitEthernet0/0
 description TO-ISP1
 ip address 100.64.1.2 255.255.255.252
 ip nat outside
 ip virtual-reassembly in
 duplex auto
 speed auto
 media-type rj45

interface GigabitEthernet0/1
 description TO-ISP2
 ip address 100.64.2.2 255.255.255.252
 ip nat outside
 ip virtual-reassembly in
 duplex auto
 speed auto
 media-type rj45

interface GigabitEthernet0/2
 description TO-CORE1
 ip address 172.16.0.1 255.255.255.252
 ip nat inside
 ip virtual-reassembly in
 duplex auto
 speed auto
 media-type rj45

interface GigabitEthernet0/3
 description TO-CORE2
 ip address 172.16.0.5 255.255.255.252
 ip nat inside
 ip virtual-reassembly in
 duplex auto
 speed auto
 media-type rj45
```

PAT statement:

```cisco
ip nat inside source list ENTERPRISE-NAT interface GigabitEthernet0/0 overload
```

Default route:

```cisco
ip route 0.0.0.0 0.0.0.0 100.64.1.1
```

Verification:

```cisco
show running-config | section ip access-list standard ENTERPRISE-NAT
show running-config interface g0/0
show running-config interface g0/1
show running-config interface g0/2
show running-config interface g0/3
show running-config | include ip nat inside source
show ip route 0.0.0.0
show ip nat statistics
show ip nat translations
```

---

## 12. ISP1 — NAT and Internet Segment Configuration

ISP1 standard NAT ACL:

```cisco
ip access-list standard ISP1-LAB-NAT
 permit 100.64.1.0 0.0.0.3
 permit 203.0.113.0 0.0.0.255
 permit 172.16.0.0 0.0.0.3
 permit 172.16.0.4 0.0.0.3
 permit 10.10.0.0 0.0.255.255
```

ISP1 interfaces:

```cisco
interface GigabitEthernet0/0
 ip address 203.0.113.1 255.255.255.0
 ip nat inside
 ip virtual-reassembly in
 duplex auto
 speed auto
 media-type rj45

interface GigabitEthernet0/1
 description TO-EDGE1
 ip address 100.64.1.1 255.255.255.252
 ip nat inside
 ip virtual-reassembly in
 duplex auto
 speed auto
 media-type rj45

interface GigabitEthernet0/2
 description TO-EXT-CONN-1
 ip address dhcp
 ip nat outside
 ip virtual-reassembly in
 duplex auto
 speed auto
 media-type rj45
```

ISP1 NAT overload:

```cisco
ip nat inside source list ISP1-LAB-NAT interface GigabitEthernet0/2 overload
```

ISP1 default route:

```cisco
ip route 0.0.0.0 0.0.0.0 GigabitEthernet0/2 dhcp 254
```

Verification:

```cisco
show running-config | section ip access-list standard ISP1-LAB-NAT
show running-config interface g0/0
show running-config interface g0/1
show running-config interface g0/2
show running-config | include ip nat inside source
show ip route 0.0.0.0
show ip nat statistics
show ip nat translations
```

---

## 13. ISP2 — NAT and Internet Segment Configuration

ISP2 standard NAT ACL:

```cisco
ip access-list standard ISP2-LAB-NAT
 permit 100.64.2.0 0.0.0.3
 permit 203.0.113.0 0.0.0.255
 permit 172.16.0.0 0.0.0.3
 permit 172.16.0.4 0.0.0.3
 permit 10.10.0.0 0.0.255.255
```

ISP2 interfaces:

```cisco
interface GigabitEthernet0/0
 ip address 203.0.113.2 255.255.255.0
 ip nat inside
 ip virtual-reassembly in
 duplex auto
 speed auto
 media-type rj45

interface GigabitEthernet0/1
 ip address 100.64.2.1 255.255.255.252
 ip nat inside
 ip virtual-reassembly in
 duplex auto
 speed auto
 media-type rj45

interface GigabitEthernet0/2
 description TO-EXT-CONN-2
 ip address dhcp
 ip nat outside
 ip virtual-reassembly in
 duplex auto
 speed auto
 media-type rj45
```

ISP2 NAT overload:

```cisco
ip nat inside source list ISP2-LAB-NAT interface GigabitEthernet0/2 overload
```

ISP2 default route:

```cisco
ip route 0.0.0.0 0.0.0.0 GigabitEthernet0/2 dhcp 254
```

Verification:

```cisco
show running-config | section ip access-list standard ISP2-LAB-NAT
show running-config interface g0/0
show running-config interface g0/1
show running-config interface g0/2
show running-config | include ip nat inside source
show ip route 0.0.0.0
show ip nat statistics
show ip nat translations
```

---

## 14. MGMT-SRV1 — chrony NTP Configuration

Install chrony:

```bash
sudo NEEDRESTART_MODE=a apt install -y chrony
```

Edit configuration:

```bash
sudo nano /etc/chrony/chrony.conf
```

Recommended Phase 02 additions:

```conf
# Collapsed Core Campus Ver.2 - Phase 02 NTP
# MGMT-SRV1 acts as internal NTP server.

allow 10.10.0.0/16
local stratum 10
```

Restart and enable chrony:

```bash
sudo systemctl restart chrony
sudo systemctl enable chrony
systemctl status chrony --no-pager
```

Verify NTP sources:

```bash
chronyc sources -v
chronyc tracking
sudo ss -lnup | grep ':123'
```

Expected:

```text
chrony active (running)
0.0.0.0:123
```

---

## 15. Cisco Devices — NTP Client Configuration

Apply to CORE1, CORE2, and EDGE1:

```cisco
conf t
ntp server 10.10.40.10
end
```

Verification:

```cisco
show ntp associations
show ntp status
show clock
```

Observed devices:

```text
CORE1
CORE2
EDGE1
```

---

## 16. MGMT-SRV1 — rsyslog Syslog Server Configuration

Install or verify rsyslog:

```bash
which rsyslogd
systemctl status rsyslog --no-pager
```

Create rsyslog configuration:

```bash
sudo nano /etc/rsyslog.d/10-campus-network.conf
```

Configuration:

```conf
# Collapsed Core Campus Ver.2 - Phase 02 Syslog Server
# MGMT-SRV1 receives Cisco device logs.

module(load="imudp")
input(type="imudp" port="514")

$template CampusNetworkLogs,"/var/log/campus-network/%FROMHOST-IP%.log"
*.* ?CampusNetworkLogs
& stop
```

Create log directory:

```bash
sudo mkdir -p /var/log/campus-network
sudo chown syslog:adm /var/log/campus-network
sudo chmod 750 /var/log/campus-network
```

Validate and restart:

```bash
sudo rsyslogd -N1
sudo systemctl restart rsyslog
systemctl status rsyslog --no-pager
sudo ss -lnup | grep ':514'
```

Expected:

```text
rsyslog active (running)
0.0.0.0:514
[::]:514
```

---

## 17. Cisco Devices — Syslog Configuration

CORE1:

```cisco
conf t
service timestamps log datetime msec
logging host 10.10.40.10
logging trap informational
logging source-interface Vlan40
no logging console
end
```

CORE2:

```cisco
conf t
service timestamps log datetime msec
logging host 10.10.40.10
logging trap informational
logging source-interface Vlan40
no logging console
end
```

EDGE1:

```cisco
conf t
service timestamps log datetime msec
logging host 10.10.40.10
logging trap informational
logging source-interface GigabitEthernet0/2
no logging console
end
```

Verification on Cisco devices:

```cisco
show running-config | include logging
show logging
```

Verification on MGMT-SRV1:

```bash
sudo ls -l /var/log/campus-network/
sudo tail -n 30 /var/log/campus-network/*
```

Observed log examples:

```text
%SYS-6-LOGGINGHOST_STARTSTOP
%SYS-5-CONFIG_I
```

---

## 18. Cisco Devices — SSH Management Configuration

Apply to CORE1, CORE2, and EDGE1.

```cisco
conf t
ip domain-name campus.lab
username admin privilege 15 secret <ADMIN_PASSWORD>
ip ssh version 2
ip ssh time-out 60
ip ssh authentication-retries 3
line vty 0 4
 login local
 transport input ssh
end
```

Generate RSA keys if required:

```cisco
conf t
crypto key generate rsa modulus 2048
end
```

EDGE1 required RSA key generation during the lab.

Verification:

```cisco
show ip ssh
show running-config | include hostname|ip domain-name|username|ip ssh
show running-config | section line vty
```

Expected:

```text
SSH Enabled - version 2.0
transport input ssh
```

MGMT-SRV1 SSH test command:

```bash
ssh -oKexAlgorithms=+diffie-hellman-group14-sha1 \
-oHostKeyAlgorithms=+ssh-rsa \
-oPubkeyAcceptedAlgorithms=+ssh-rsa \
admin@10.10.40.1
```

CORE2:

```bash
ssh -oKexAlgorithms=+diffie-hellman-group14-sha1 \
-oHostKeyAlgorithms=+ssh-rsa \
-oPubkeyAcceptedAlgorithms=+ssh-rsa \
admin@10.10.40.2
```

EDGE1:

```bash
ssh -oKexAlgorithms=+diffie-hellman-group14-sha1 \
-oHostKeyAlgorithms=+ssh-rsa \
-oPubkeyAcceptedAlgorithms=+ssh-rsa \
admin@172.16.0.1
```

Successful prompts:

```text
CORE1#
CORE2#
EDGE1#
```

---

## 19. Cisco Devices — SNMPv2c Monitoring Configuration

Apply to CORE1, CORE2, and EDGE1.

```cisco
conf t
ip access-list standard SNMP-MGMT
 permit 10.10.40.10
 deny any
exit

snmp-server community NMS-RO RO SNMP-MGMT
snmp-server location Collapsed-Core-Campus-Ver2
snmp-server contact NetOps-Lab
end
```

Verification on Cisco devices:

```cisco
show access-lists SNMP-MGMT
show running-config | include snmp-server
```

Expected ACL:

```text
Standard IP access list SNMP-MGMT
    10 permit 10.10.40.10
    20 deny any
```

MGMT-SRV1 SNMP verification:

```bash
snmpwalk -v2c -c NMS-RO 10.10.40.1 1.3.6.1.2.1.1.5.0
snmpwalk -v2c -c NMS-RO 10.10.40.2 1.3.6.1.2.1.1.5.0
snmpwalk -v2c -c NMS-RO 172.16.0.1 1.3.6.1.2.1.1.5.0
```

Expected:

```text
CORE1.campus.lab
CORE2.campus.lab
EDGE1.campus.lab
```

Interface description OID:

```bash
snmpwalk -v2c -c NMS-RO 10.10.40.1 1.3.6.1.2.1.2.2.1.2
snmpwalk -v2c -c NMS-RO 10.10.40.2 1.3.6.1.2.1.2.2.1.2
snmpwalk -v2c -c NMS-RO 172.16.0.1 1.3.6.1.2.1.2.2.1.2
```

---

## 20. Cisco Devices — AAA Local and Management ACL

Apply to CORE1, CORE2, and EDGE1.

```cisco
conf t

aaa new-model
aaa authentication login default local
aaa authorization exec default local

ip access-list standard MGMT-ACCESS
 permit 10.10.40.10
 deny any
exit

line vty 0 4
 access-class MGMT-ACCESS in
 transport input ssh
exit

end
```

Verification:

```cisco
show running-config | include aaa
show running-config | section ip access-list standard MGMT-ACCESS
show running-config | section line vty
```

Expected:

```text
aaa new-model
aaa authentication login default local
aaa authorization exec default local
aaa session-id common

ip access-list standard MGMT-ACCESS
 permit 10.10.40.10
 deny any

line vty 0 4
 access-class MGMT-ACCESS in
 transport input ssh
```

SSH validation from MGMT-SRV1:

```bash
ssh -oKexAlgorithms=+diffie-hellman-group14-sha1 \
-oHostKeyAlgorithms=+ssh-rsa \
-oPubkeyAcceptedAlgorithms=+ssh-rsa \
admin@10.10.40.1

ssh -oKexAlgorithms=+diffie-hellman-group14-sha1 \
-oHostKeyAlgorithms=+ssh-rsa \
-oPubkeyAcceptedAlgorithms=+ssh-rsa \
admin@10.10.40.2

ssh -oKexAlgorithms=+diffie-hellman-group14-sha1 \
-oHostKeyAlgorithms=+ssh-rsa \
-oPubkeyAcceptedAlgorithms=+ssh-rsa \
admin@172.16.0.1
```

Telnet is blocked by configuration:

```cisco
transport input ssh
```

---

## 21. MGMT-SRV1 — Config Backup Directory

Create backup directories:

```bash
sudo mkdir -p /srv/config-backups/{CORE1,CORE2,EDGE1}
sudo chown -R cisco:cisco /srv/config-backups
chmod -R 750 /srv/config-backups
```

Verify:

```bash
ls -ld /srv/config-backups
ls -l /srv/config-backups
```

Expected:

```text
CORE1
CORE2
EDGE1
```

---

## 22. MGMT-SRV1 — Cisco Running Config Backup Commands

CORE1:

```bash
printf "terminal length 0\nshow running-config\nexit\n" | \
ssh -tt -oKexAlgorithms=+diffie-hellman-group14-sha1 \
-oHostKeyAlgorithms=+ssh-rsa \
-oPubkeyAcceptedAlgorithms=+ssh-rsa \
admin@10.10.40.1 > /srv/config-backups/CORE1/CORE1-running-config-$(date +%F).txt
```

CORE2:

```bash
printf "terminal length 0\nshow running-config\nexit\n" | \
ssh -tt -oKexAlgorithms=+diffie-hellman-group14-sha1 \
-oHostKeyAlgorithms=+ssh-rsa \
-oPubkeyAcceptedAlgorithms=+ssh-rsa \
admin@10.10.40.2 > /srv/config-backups/CORE2/CORE2-running-config-$(date +%F).txt
```

EDGE1:

```bash
printf "terminal length 0\nshow running-config\nexit\n" | \
ssh -tt -oKexAlgorithms=+diffie-hellman-group14-sha1 \
-oHostKeyAlgorithms=+ssh-rsa \
-oPubkeyAcceptedAlgorithms=+ssh-rsa \
admin@172.16.0.1 > /srv/config-backups/EDGE1/EDGE1-running-config-$(date +%F).txt
```

Verify backup files:

```bash
find /srv/config-backups -type f -name "*running-config-$(date +%F).txt" -ls
grep -H "^hostname" /srv/config-backups/*/*running-config-$(date +%F).txt
```

Expected:

```text
/srv/config-backups/CORE1/CORE1-running-config-YYYY-MM-DD.txt:hostname CORE1
/srv/config-backups/CORE2/CORE2-running-config-YYYY-MM-DD.txt:hostname CORE2
/srv/config-backups/EDGE1/EDGE1-running-config-YYYY-MM-DD.txt:hostname EDGE1
```

---

## 23. MGMT-SRV1 — Enterprise Management Server Backup

This backup protects MGMT-SRV1 itself.

It is different from Cisco Config Backup.

Cisco Config Backup:

```text
MGMT-SRV1 backs up Cisco device running-configs.
```

MGMT-SRV1 Backup:

```text
MGMT-SRV1 backs up its own DHCP/DNS/NTP/Syslog/Config Backup service files.
```

Create archive:

```bash
sudo tar -czvf ~/phase02-mgmt-srv1-enterprise-services-$(date +%F).tar.gz \
/etc/dnsmasq.d/campus-dns.conf \
/etc/chrony/chrony.conf \
/etc/rsyslog.d/10-campus-network.conf \
/srv/config-backups
```

Verify archive:

```bash
ls -lh ~/phase02-mgmt-srv1-enterprise-services-$(date +%F).tar.gz
tar -tzvf ~/phase02-mgmt-srv1-enterprise-services-$(date +%F).tar.gz
```

Observed archive:

```text
/home/cisco/phase02-mgmt-srv1-enterprise-services-2026-06-06.tar.gz
```

Expected included files:

```text
/etc/dnsmasq.d/campus-dns.conf
/etc/chrony/chrony.conf
/etc/rsyslog.d/10-campus-network.conf
/srv/config-backups/CORE1/CORE1-running-config-2026-06-06.txt
/srv/config-backups/CORE2/CORE2-running-config-2026-06-06.txt
/srv/config-backups/EDGE1/EDGE1-running-config-2026-06-06.txt
```

Normal tar message:

```text
tar: Removing leading '/' from member names
```

This is expected and safe.

---

## 24. SERVER1 Configuration

SERVER1 is a static server node.

Expected interface:

```text
ens2
```

Expected IP:

```text
10.10.70.10/24
```

Verification:

```bash
ip -br addr
```

Observed:

```text
ens2 UP 10.10.70.10/24
```

SERVER1 is registered in DNS:

```text
server1.campus.lab → 10.10.70.10
```

SERVER1 is not the DNS/DHCP server.
DNS and DHCP services are provided by MGMT-SRV1.

---

## 25. Desktop / Tiny Core / Alpine Client DHCP Test

On the DHCP client:

```bash
sudo ip addr flush dev eth0
sudo udhcpc -i eth0
ip addr
ip route
```

Expected reservation test result:

```text
lease of 10.10.10.136 obtained from 10.10.40.10
```

Expected network state:

```text
IP address      : 10.10.10.136/24
Default gateway : 10.10.10.254
```

DNS direct test:

```bash
nslookup google.com 10.10.40.10
nslookup server1.campus.lab 10.10.40.10
nslookup mgmt-srv1.campus.lab 10.10.40.10
```

Expected:

```text
google.com resolved successfully
server1.campus.lab   → 10.10.70.10
mgmt-srv1.campus.lab → 10.10.40.10
```

---

## 26. Final Verification Commands

MGMT-SRV1 network:

```bash
ip -br addr
ip route
```

MGMT-SRV1 services:

```bash
systemctl status dnsmasq --no-pager
systemctl status chrony --no-pager
systemctl status rsyslog --no-pager
which snmpwalk
which snmpget
```

DNS:

```bash
dig @10.10.40.10 google.com A
dig @10.10.40.10 server1.campus.lab A
dig @10.10.40.10 mgmt-srv1.campus.lab A
dig @10.10.40.10 internet-srv.campus.lab A
```

SNMP:

```bash
snmpwalk -v2c -c NMS-RO 10.10.40.1 1.3.6.1.2.1.1.5.0
snmpwalk -v2c -c NMS-RO 10.10.40.2 1.3.6.1.2.1.1.5.0
snmpwalk -v2c -c NMS-RO 172.16.0.1 1.3.6.1.2.1.1.5.0
```

Config backup:

```bash
find /srv/config-backups -type f -name "*running-config-$(date +%F).txt" -ls
grep -H "^hostname" /srv/config-backups/*/*running-config-$(date +%F).txt
```

Cisco save:

```cisco
write memory
```

Recommended devices to save:

```text
CORE1
CORE2
EDGE1
ISP1
ISP2
```

---

## 27. Final State Summary

At the end of Phase 02:

```text
MGMT-SRV1 = Enterprise Management Server

Services:
- DHCP active
- DNS resolver active
- NTP active
- Syslog active
- SNMP tools installed
- Config backup directory created
- Cisco running-configs backed up
- MGMT-SRV1 service archive created
```

Cisco infrastructure:

```text
CORE1 / CORE2 / EDGE1:
- SSH enabled
- AAA Local enabled
- Management ACL applied
- SNMPv2c RO enabled
- SNMP restricted to MGMT-SRV1
- Syslog forwarding configured
- NTP client configured
- Config backup completed
```

This completes the Phase 02 Enterprise Services Ver.2 configuration set.
