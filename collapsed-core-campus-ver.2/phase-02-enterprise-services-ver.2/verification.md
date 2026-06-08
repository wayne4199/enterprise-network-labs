# Phase 02 — Enterprise Services Ver.2 Verification

## 1. Purpose

This document records the verification commands and successful validation results for **Phase 02 — Enterprise Services Ver.2** of the Collapsed Core Campus lab.

Phase 02 validates the enterprise services built around **MGMT-SRV1**.

MGMT-SRV1 is not a simple standalone server.
MGMT-SRV1 functions as the **Enterprise Management Server** for the campus network.

Centralized services validated in this phase:

* DHCP
* DNS Resolver
* NAT/PAT
* NTP
* Syslog
* SSH
* SNMP
* AAA Local
* Management ACL
* Config Backup
* DHCP Reservation
* DNS Static Record
* MGMT-SRV1 Enterprise Management Server Backup

---

## 2. Verification Summary

| Area                       | Status     |
| -------------------------- | ---------- |
| Phase 01 baseline retained | Successful |
| External Connector         | Successful |
| Internet access            | Successful |
| NAT/PAT                    | Successful |
| DNS Resolver               | Successful |
| DHCP                       | Successful |
| DHCP Relay                 | Successful |
| DHCP Reservation           | Successful |
| DNS Static Records         | Successful |
| NTP                        | Successful |
| Syslog                     | Successful |
| SSH                        | Successful |
| SNMP                       | Successful |
| AAA Local                  | Successful |
| Management ACL             | Successful |
| Config Backup              | Successful |
| MGMT-SRV1 service backup   | Successful |

---

## 3. Phase 01 Baseline Verification

Before starting Phase 02, the Phase 01 foundation was verified.

### Cisco Device Checks

Commands used:

```cisco
show ip interface brief
show running-config | include ip route
show ip route
show vlan brief
show interfaces trunk
show spanning-tree summary
show standby brief
```

Validated devices:

```text
EDGE1
CORE1
CORE2
ISP1
ISP2
ACC1
ACC2
ACC3
SRV-ACC1
```

### Result

Phase 01 baseline was retained.

Confirmed items:

```text
VLANs retained
SVIs retained
HSRP retained
STP retained
Trunks retained
Routed links retained
Static routes retained
Internet simulation reachability retained
```

---

## 4. Linux Node Baseline Verification

### SERVER1

Commands:

```bash
ip -br addr
ip route
ping -c 4 10.10.70.254
ping -c 4 10.10.40.10
ping -c 4 203.0.113.10
```

Expected:

```text
ens2 UP 10.10.70.10/24
default via 10.10.70.254
```

Result:

```text
SERVER1 static IP retained
SERVER1 default gateway retained
SERVER1 reached MGMT-SRV1
SERVER1 reached INTERNET-SRV
```

### MGMT-SRV1

Commands:

```bash
ip -br addr
ip route
ping -c 4 10.10.40.254
ping -c 4 10.10.70.10
ping -c 4 203.0.113.10
```

Expected:

```text
ens2 UP 10.10.40.10/24
default via 10.10.40.254
```

Result:

```text
MGMT-SRV1 static IP retained
MGMT-SRV1 default gateway retained
MGMT-SRV1 reached SERVER1
MGMT-SRV1 reached INTERNET-SRV
```

### INTERNET-SRV

Commands:

```bash
ifconfig eth0
route -n
ping -c 4 203.0.113.1
ping -c 4 203.0.113.2
ping -c 4 10.10.40.10
ping -c 4 10.10.70.10
```

Result:

```text
INTERNET-SRV reached ISP1
INTERNET-SRV reached ISP2
INTERNET-SRV reached MGMT-SRV1
INTERNET-SRV reached SERVER1
```

---

## 5. External Connector Verification

External Connector interfaces were attached to ISP1 and ISP2.

### ISP1

Commands:

```cisco
show ip interface brief
show dhcp lease
show ip route
ping 192.168.255.1
ping 8.8.8.8 source GigabitEthernet0/2
ping 1.1.1.1 source GigabitEthernet0/2
```

Observed:

```text
ISP1 G0/2 received DHCP address from External Connector
ISP1 default route via 192.168.255.1
ISP1 reached 8.8.8.8
ISP1 reached 1.1.1.1
```

### ISP2

Commands:

```cisco
show ip interface brief
show dhcp lease
show ip route
ping 192.168.255.1
ping 8.8.8.8 source GigabitEthernet0/2
ping 1.1.1.1 source GigabitEthernet0/2
```

Observed:

```text
ISP2 G0/2 received DHCP address from External Connector
ISP2 default route via 192.168.255.1
ISP2 reached 8.8.8.8
ISP2 reached 1.1.1.1
```

Result:

```text
External Connector verification successful
```

---

## 6. NAT/PAT Verification

### MGMT-SRV1 Internet IP Reachability

Commands:

```bash
ping -c 4 8.8.8.8
ping -c 4 1.1.1.1
```

Expected:

```text
0% packet loss
```

Result:

```text
MGMT-SRV1 reached real Internet IP addresses
```

### EDGE1 NAT/PAT Verification

Commands:

```cisco
show running-config | section ip access-list standard ENTERPRISE-NAT
show running-config | include ip nat inside source
show ip route 0.0.0.0
show ip nat statistics
show ip nat translations
```

Expected:

```text
ENTERPRISE-NAT permits 10.10.0.0/16
G0/2 and G0/3 are NAT inside
G0/0 and G0/1 are NAT outside
PAT uses G0/0
```

Observed translation pattern:

```text
10.10.40.10 → 100.64.1.2 → Internet
```

Result:

```text
EDGE1 Enterprise NAT/PAT successful
```

### ISP1 Lab NAT Verification

Commands:

```cisco
show running-config | section ip access-list standard ISP1-LAB-NAT
show running-config | include ip nat inside source
show ip route 0.0.0.0
show ip nat statistics
show ip nat translations
```

Observed translation pattern:

```text
100.64.1.2 → 192.168.255.79 → Internet
```

Result:

```text
ISP1 Lab NAT successful
```

### ISP2 Lab NAT Verification

Commands:

```cisco
show running-config | section ip access-list standard ISP2-LAB-NAT
show running-config | include ip nat inside source
show ip route 0.0.0.0
show ip nat statistics
show ip nat translations
```

Expected:

```text
ISP2 NAT configuration present
Translation count may be 0
```

Reason:

```text
EDGE1 primary default route uses ISP1
ISP2 is backup-ready in Phase 02
```

Result:

```text
ISP2 Lab NAT configuration verified
```

---

## 7. apt update and Tool Installation Verification

MGMT-SRV1 commands:

```bash
sudo apt update
sudo NEEDRESTART_MODE=a apt install -y dnsutils curl traceroute tcpdump
```

Verification:

```bash
which dig
which curl
which traceroute
which tcpdump
```

Expected:

```text
/usr/bin/dig
/usr/bin/curl
/usr/sbin/traceroute
/usr/bin/tcpdump
```

Additional external verification:

```bash
dig google.com
curl -I https://www.google.com
traceroute 8.8.8.8
```

Observed:

```text
dig returned DNS answers
curl returned HTTP response
traceroute showed path through CORE/EDGE/ISP/External Connector
```

Result:

```text
MGMT-SRV1 package installation and Internet access verified
```

---

## 8. DNS Resolver Verification

### dnsmasq Service Status

Commands:

```bash
sudo dnsmasq --test
systemctl status dnsmasq --no-pager
sudo ss -lnup | grep ':53'
```

Expected:

```text
dnsmasq: syntax check OK
dnsmasq active (running)
10.10.40.10:53 listening
```

Result:

```text
dnsmasq DNS service active
```

### External DNS Forwarding

Command:

```bash
dig @10.10.40.10 google.com A
```

Expected:

```text
status: NOERROR
ANSWER SECTION present
SERVER: 10.10.40.10#53
```

Result:

```text
External DNS forwarding successful
```

### Internal Static DNS Records

Commands:

```bash
dig @10.10.40.10 server1.campus.lab A
dig @10.10.40.10 mgmt-srv1.campus.lab A
dig @10.10.40.10 internet-srv.campus.lab A
```

Expected:

```text
server1.campus.lab      → 10.10.70.10
mgmt-srv1.campus.lab    → 10.10.40.10
internet-srv.campus.lab → 203.0.113.10
```

Result:

```text
Internal DNS static records verified
```

---

## 9. DHCP Server Verification

### dnsmasq DHCP Service

Commands:

```bash
sudo dnsmasq --test
systemctl status dnsmasq --no-pager
sudo ss -lnup | grep ':67'
```

Expected:

```text
dnsmasq active (running)
UDP 67 listening
```

Result:

```text
MGMT-SRV1 DHCP service active
```

### DHCP Lease File

Command:

```bash
cat /var/lib/misc/dnsmasq.leases
```

Observed leases included:

```text
10.10.10.136
10.10.11.100
10.10.20.117
10.10.21.133
10.10.30.192
10.10.31.167
10.10.60.127
```

Result:

```text
DHCP leases successfully created
```

---

## 10. DHCP Relay Verification

### CORE1 / CORE2

Commands:

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

Expected:

```cisco
ip helper-address 10.10.40.10
```

Result:

```text
DHCP relay configured on CORE1 and CORE2 user/service VLAN SVIs
```

---

## 11. DHCP Client Verification

On the Alpine Desktop client:

```bash
sudo ip addr flush dev eth0
sudo udhcpc -i eth0
ip addr
ip route
```

Expected:

```text
IP address from DHCP pool
Default gateway = VLAN HSRP VIP
```

Observed reservation client:

```text
lease of 10.10.10.136 obtained from 10.10.40.10
IP address: 10.10.10.136/24
Default gateway: 10.10.10.254
```

Result:

```text
DHCP client successfully received IP and gateway from MGMT-SRV1
```

---

## 12. DHCP Reservation Verification

Reservation entry:

```conf
dhcp-host=52:54:00:71:87:ac,10.10.10.136,exec,12h
```

Client renewal:

```bash
sudo ip addr flush dev eth0
sudo udhcpc -i eth0
```

Observed:

```text
lease of 10.10.10.136 obtained from 10.10.40.10
```

Result:

```text
DHCP reservation successful
```

---

## 13. Client DNS Verification

On Alpine Desktop client:

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

Result:

```text
Client DNS resolution through MGMT-SRV1 verified
```

Note:

```text
Alpine / BusyBox nslookup may show additional NXDOMAIN lines.
The A record response is the important validation result.
```

---

## 14. NTP Server Verification on MGMT-SRV1

Commands:

```bash
systemctl status chrony --no-pager
chronyc sources -v
chronyc tracking
sudo ss -lnup | grep ':123'
```

Expected:

```text
chrony active (running)
Leap status: Normal
UDP 123 listening
```

Result:

```text
MGMT-SRV1 chrony NTP service verified
```

---

## 15. Cisco NTP Client Verification

Devices verified:

```text
CORE1
CORE2
EDGE1
```

Commands:

```cisco
show ntp associations
show ntp status
show clock
```

Expected:

```text
Clock is synchronized
reference is 10.10.40.10
```

Observed:

```text
CORE1 synchronized to 10.10.40.10
CORE2 synchronized to 10.10.40.10
EDGE1 synchronized to 10.10.40.10
```

Result:

```text
NTP client synchronization successful for core and edge devices
```

---

## 16. Syslog Server Verification on MGMT-SRV1

Commands:

```bash
systemctl status rsyslog --no-pager
sudo rsyslogd -N1
sudo ss -lnup | grep ':514'
```

Expected:

```text
rsyslog active (running)
End of config validation run. Bye.
0.0.0.0:514
[::]:514
```

Result:

```text
MGMT-SRV1 rsyslog server active and listening on UDP/514
```

---

## 17. Cisco Syslog Client Verification

Devices configured:

```text
CORE1
CORE2
EDGE1
```

Cisco verification:

```cisco
show running-config | include logging
show logging
```

Expected:

```text
logging host 10.10.40.10
logging trap informational
logging source-interface <interface>
```

MGMT-SRV1 log verification:

```bash
sudo ls -l /var/log/campus-network/
sudo tail -n 30 /var/log/campus-network/*
```

Observed example:

```text
/var/log/campus-network/10.10.40.1.log
%SYS-6-LOGGINGHOST_STARTSTOP
%SYS-5-CONFIG_I
```

Result:

```text
Cisco Syslog messages successfully received by MGMT-SRV1
```

---

## 18. SSH Server Verification on Cisco Devices

Devices verified:

```text
CORE1
CORE2
EDGE1
```

Cisco commands:

```cisco
show ip ssh
show running-config | include hostname|ip domain-name|username|ip ssh
show running-config | section line vty
```

Expected:

```text
SSH Enabled - version 2.0
username admin privilege 15 secret ...
transport input ssh
```

Result:

```text
SSH server configuration verified on CORE1, CORE2, and EDGE1
```

---

## 19. SSH Access Verification from MGMT-SRV1

Because Ubuntu 24.04 OpenSSH rejected older Cisco IOSv algorithms by default, compatibility options were used.

### CORE1

```bash
ssh -oKexAlgorithms=+diffie-hellman-group14-sha1 \
-oHostKeyAlgorithms=+ssh-rsa \
-oPubkeyAcceptedAlgorithms=+ssh-rsa \
admin@10.10.40.1
```

Expected:

```text
CORE1#
```

### CORE2

```bash
ssh -oKexAlgorithms=+diffie-hellman-group14-sha1 \
-oHostKeyAlgorithms=+ssh-rsa \
-oPubkeyAcceptedAlgorithms=+ssh-rsa \
admin@10.10.40.2
```

Expected:

```text
CORE2#
```

### EDGE1

```bash
ssh -oKexAlgorithms=+diffie-hellman-group14-sha1 \
-oHostKeyAlgorithms=+ssh-rsa \
-oPubkeyAcceptedAlgorithms=+ssh-rsa \
admin@172.16.0.1
```

Expected:

```text
EDGE1#
```

Result:

```text
MGMT-SRV1 successfully connected to CORE1, CORE2, and EDGE1 using SSH
```

---

## 20. AAA Local Verification

Devices:

```text
CORE1
CORE2
EDGE1
```

Commands:

```cisco
show running-config | include aaa
show running-config | section line vty
```

Expected:

```text
aaa new-model
aaa authentication login default local
aaa authorization exec default local
aaa session-id common
```

Result:

```text
AAA Local configuration verified
```

SSH was re-tested after AAA Local was applied.

Observed:

```text
MGMT-SRV1 → CORE1 SSH successful
MGMT-SRV1 → CORE2 SSH successful
MGMT-SRV1 → EDGE1 SSH successful
```

---

## 21. Management ACL Verification

Devices:

```text
CORE1
CORE2
EDGE1
```

Commands:

```cisco
show running-config | section ip access-list standard MGMT-ACCESS
show running-config | section line vty
```

Expected:

```text
ip access-list standard MGMT-ACCESS
 permit 10.10.40.10
 deny any

line vty 0 4
 access-class MGMT-ACCESS in
 transport input ssh
```

Result:

```text
Management ACL applied to VTY lines
SSH access from MGMT-SRV1 still successful
```

Telnet blocking was verified by configuration:

```text
transport input ssh
```

---

## 22. SNMP Configuration Verification

Devices:

```text
CORE1
CORE2
EDGE1
```

Cisco verification:

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

Expected SNMP configuration:

```text
snmp-server community NMS-RO RO SNMP-MGMT
snmp-server location Collapsed-Core-Campus-Ver2
snmp-server contact NetOps-Lab
```

Result:

```text
SNMP community and SNMP ACL verified
```

---

## 23. SNMP Query Verification from MGMT-SRV1

### sysName OID

Commands:

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

Result:

```text
SNMP sysName query successful
```

### Interface Description OID

Commands:

```bash
snmpwalk -v2c -c NMS-RO 10.10.40.1 1.3.6.1.2.1.2.2.1.2
snmpwalk -v2c -c NMS-RO 10.10.40.2 1.3.6.1.2.1.2.2.1.2
snmpwalk -v2c -c NMS-RO 172.16.0.1 1.3.6.1.2.1.2.2.1.2
```

Expected examples:

```text
GigabitEthernet0/0
GigabitEthernet0/1
GigabitEthernet0/2
GigabitEthernet0/3
Vlan10
Vlan20
Vlan40
Null0
NVI0
```

Result:

```text
SNMP interface description query successful
```

---

## 24. Config Backup Verification

Backup directories:

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

Backup files:

```bash
find /srv/config-backups -type f -name "*running-config-$(date +%F).txt" -ls
```

Hostname verification:

```bash
grep -H "^hostname" /srv/config-backups/*/*running-config-$(date +%F).txt
```

Expected:

```text
/srv/config-backups/CORE1/CORE1-running-config-YYYY-MM-DD.txt:hostname CORE1
/srv/config-backups/CORE2/CORE2-running-config-YYYY-MM-DD.txt:hostname CORE2
/srv/config-backups/EDGE1/EDGE1-running-config-YYYY-MM-DD.txt:hostname EDGE1
```

Observed:

```text
CORE1 backup file contained hostname CORE1
CORE2 backup file contained hostname CORE2
EDGE1 backup file contained hostname EDGE1
```

Result:

```text
Cisco running-config backup verified
```

---

## 25. MGMT-SRV1 Enterprise Management Server Backup Verification

Backup archive:

```text
/home/cisco/phase02-mgmt-srv1-enterprise-services-2026-06-06.tar.gz
```

Commands:

```bash
ls -lh ~/phase02-mgmt-srv1-enterprise-services-$(date +%F).tar.gz
tar -tzvf ~/phase02-mgmt-srv1-enterprise-services-$(date +%F).tar.gz
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

Result:

```text
MGMT-SRV1 Enterprise Management Server backup verified
```

---

## 26. MGMT-SRV1 Reboot Persistence Verification

After MGMT-SRV1 reboot, the following were verified.

Commands:

```bash
ip -br addr
ip route
systemctl status dnsmasq --no-pager
systemctl status chrony --no-pager
systemctl status rsyslog --no-pager
which snmpwalk
which snmpget
```

Expected:

```text
ens2 UP 10.10.40.10/24
default via 10.10.40.254
dnsmasq active (running)
chrony active (running)
rsyslog active (running)
snmpwalk present
snmpget present
```

Observed:

```text
MGMT-SRV1 IP persisted
Default route persisted
dnsmasq remained active
chrony remained active
rsyslog remained active
SNMP tools remained installed
```

Result:

```text
MGMT-SRV1 Enterprise Management Server configuration survived reboot
```

---

## 27. Final Integrated Verification

The final integrated checks confirmed:

### MGMT-SRV1 Network

```bash
ip -br addr
ip route
```

Result:

```text
ens2 10.10.40.10/24
default via 10.10.40.254
```

### Services

```bash
systemctl status dnsmasq --no-pager
systemctl status chrony --no-pager
systemctl status rsyslog --no-pager
which snmpwalk
which snmpget
```

Result:

```text
dnsmasq active
chrony active
rsyslog active
snmpwalk installed
snmpget installed
```

### DNS

```bash
dig @10.10.40.10 google.com A
dig @10.10.40.10 server1.campus.lab A
dig @10.10.40.10 mgmt-srv1.campus.lab A
dig @10.10.40.10 internet-srv.campus.lab A
```

Result:

```text
google.com resolved
server1.campus.lab      → 10.10.70.10
mgmt-srv1.campus.lab    → 10.10.40.10
internet-srv.campus.lab → 203.0.113.10
```

### SNMP

```bash
snmpwalk -v2c -c NMS-RO 10.10.40.1 1.3.6.1.2.1.1.5.0
snmpwalk -v2c -c NMS-RO 10.10.40.2 1.3.6.1.2.1.1.5.0
snmpwalk -v2c -c NMS-RO 172.16.0.1 1.3.6.1.2.1.1.5.0
```

Result:

```text
CORE1.campus.lab
CORE2.campus.lab
EDGE1.campus.lab
```

### Config Backup

```bash
find /srv/config-backups -type f -name "*running-config-$(date +%F).txt" -ls
grep -H "^hostname" /srv/config-backups/*/*running-config-$(date +%F).txt
```

Result:

```text
hostname CORE1
hostname CORE2
hostname EDGE1
```

---

## 28. Final Verification Status

| Feature                    | Verification Result |
| -------------------------- | ------------------- |
| Phase 01 baseline          | Passed              |
| External Connector         | Passed              |
| Real Internet reachability | Passed              |
| NAT/PAT                    | Passed              |
| apt update                 | Passed              |
| DNS resolver               | Passed              |
| DNS static records         | Passed              |
| DHCP service               | Passed              |
| DHCP relay                 | Passed              |
| DHCP client lease          | Passed              |
| DHCP reservation           | Passed              |
| NTP server                 | Passed              |
| NTP clients                | Passed              |
| Syslog server              | Passed              |
| Syslog clients             | Passed              |
| SSH access                 | Passed              |
| AAA Local                  | Passed              |
| Management ACL             | Passed              |
| SNMPv2c                    | Passed              |
| Config Backup              | Passed              |
| MGMT-SRV1 backup           | Passed              |
| Reboot persistence         | Passed              |

---

## 29. Completion Statement

Phase 02 Enterprise Services Ver.2 verification is complete.

MGMT-SRV1 successfully functions as the Enterprise Management Server for the campus network.

Verified centralized services:

```text
DHCP
DNS Resolver
NTP
Syslog
SNMP Monitoring
Config Backup
Enterprise Management Server Backup
```

Verified managed Cisco devices:

```text
CORE1
CORE2
EDGE1
```

The Phase 02 Enterprise Services foundation is ready for the next lab phase.
