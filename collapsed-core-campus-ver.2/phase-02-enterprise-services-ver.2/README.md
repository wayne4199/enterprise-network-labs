# Phase 02 — Enterprise Services Ver.2

## 1. Overview

This phase builds the Enterprise Services foundation for the Collapsed Core Campus Ver.2 lab.

Phase 01 focused on the campus switching and Layer 3 foundation.
Phase 02 extends that foundation by adding centralized management and operational services through **MGMT-SRV1**.

In this phase, **MGMT-SRV1 is not a simple standalone server**.
MGMT-SRV1 functions as the **Enterprise Management Server** for the campus network.

It centralizes the following operational services:

* DHCP Server
* DNS Resolver
* NTP Server
* Syslog Server
* SNMP Monitoring Station
* Config Backup Server

The goal is not only to make the network reachable, but also to build a practical enterprise-style management plane that can support operations, monitoring, troubleshooting, and recovery.

---

## 2. Lab Objectives

The main objectives of this phase are:

* Build centralized DHCP services using MGMT-SRV1
* Configure DHCP relay on the collapsed core
* Configure DNS resolver and internal static DNS records
* Provide NAT/PAT for internal users to reach external networks
* Configure centralized NTP service
* Configure centralized Syslog collection
* Enable SSH-only management access
* Configure SNMPv2c read-only monitoring
* Apply AAA Local authentication
* Restrict management access using ACLs
* Back up Cisco device configurations to MGMT-SRV1
* Create DHCP reservation examples
* Validate DNS static records
* Preserve MGMT-SRV1 Enterprise Management Server configuration

---

## 3. Design Role of MGMT-SRV1

MGMT-SRV1 is the central Enterprise Management Server in this phase.

It provides the operational services that enterprise networks commonly centralize on a management server or management services segment.

MGMT-SRV1 provides:

| Service       | Purpose                                             |
| ------------- | --------------------------------------------------- |
| DHCP          | Provides IP addressing to user and service VLANs    |
| DNS Resolver  | Resolves external names and internal campus records |
| NTP           | Provides time synchronization for network devices   |
| Syslog        | Collects logs from Cisco infrastructure devices     |
| SNMP Tools    | Queries managed devices for monitoring information  |
| Config Backup | Stores running-config backups from Cisco devices    |

This design separates the management service function from normal user traffic and makes MGMT-SRV1 the operational control point for Phase 02.

---

## 4. Phase 02 Scope

This phase includes the following sections:

1. Management Server Baseline
2. DHCP Service
3. DHCP Relay
4. DNS Resolver
5. NAT/PAT
6. NTP
7. Syslog
8. SSH Management Access
9. SNMP Monitoring
10. AAA Local + Management Access Hardening
11. Config Backup
12. DHCP Reservation / DNS Static Record
13. Final Verification
14. MGMT-SRV1 Enterprise Management Server Backup

The following items are intentionally out of scope for Phase 02:

* TACACS+
* RADIUS
* PKI
* NetFlow
* Telemetry
* Full monitoring platform deployment
* Automation framework
* SNMPv3

These will be handled in later phases.

---

## 5. Topology Summary

Phase 02 continues from the Collapsed Core Campus Ver.2 foundation.

Core devices:

* CORE1
* CORE2
* EDGE1
* ISP1
* ISP2

Access and service devices:

* ACC1
* ACC2
* ACC3
* SERVER1
* MGMT-SRV1
* PRINTER1
* Desktop test clients

MGMT-SRV1 is connected to the management VLAN and provides centralized enterprise services.

---

## 6. VLAN and IP Plan

| VLAN     |                         Purpose |  Gateway VIP | Notes                  |
| -------- | ------------------------------: | -----------: | ---------------------- |
| VLAN 10  |             User / EXEC example | 10.10.10.254 | DHCP enabled           |
| VLAN 11  |         Additional user segment | 10.10.11.254 | DHCP enabled           |
| VLAN 20  |                  Server segment | 10.10.20.254 | DHCP enabled           |
| VLAN 21  |       Additional server segment | 10.10.21.254 | DHCP enabled           |
| VLAN 30  |        Voice / endpoint segment | 10.10.30.254 | DHCP enabled           |
| VLAN 31  |     Additional endpoint segment | 10.10.31.254 | DHCP enabled           |
| VLAN 40  |                      Management | 10.10.40.254 | MGMT-SRV1 resides here |
| VLAN 50  |                           Guest | 10.10.50.254 | DHCP enabled           |
| VLAN 60  | Additional user/service segment | 10.10.60.254 | DHCP enabled           |
| VLAN 70  |           Server static segment | 10.10.70.254 | SERVER1 resides here   |
| VLAN 999 |                Native blackhole |          N/A | Reserved native VLAN   |

---

## 7. Important Node IP Addresses

| Node                   |     IP Address | Role                                 |
| ---------------------- | -------------: | ------------------------------------ |
| MGMT-SRV1              | 10.10.40.10/24 | Enterprise Management Server         |
| SERVER1                | 10.10.70.10/24 | Server VLAN test server              |
| CORE1 VLAN40           |  10.10.40.1/24 | Core management SVI                  |
| CORE2 VLAN40           |  10.10.40.2/24 | Core management SVI                  |
| EDGE1 inside link      |     172.16.0.1 | Edge internal routed interface       |
| Internet server record |   203.0.113.10 | Simulated external server DNS record |

---

## 8. DHCP Design

DHCP service is centralized on MGMT-SRV1 using `dnsmasq`.

DHCP relay is configured on CORE1 and CORE2 SVIs using:

```cisco
ip helper-address 10.10.40.10
```

MGMT-SRV1 provides DHCP scopes for multiple VLANs, including:

* VLAN 10
* VLAN 11
* VLAN 20
* VLAN 21
* VLAN 30
* VLAN 31
* VLAN 50
* VLAN 60

Each DHCP scope provides:

* IP address
* Subnet mask
* Default gateway
* DNS server
* Domain name

The DHCP server address provided to clients is:

```text
10.10.40.10
```

The domain name provided to clients is:

```text
campus.lab
```

---

## 9. DNS Resolver Design

MGMT-SRV1 also acts as the DNS resolver for the campus network.

The DNS resolver provides:

* External DNS forwarding
* Internal static DNS records
* Local campus domain resolution

Upstream DNS servers:

```text
8.8.8.8
1.1.1.1
```

Internal domain:

```text
campus.lab
```

Important static DNS records:

| DNS Name                |   IP Address |
| ----------------------- | -----------: |
| mgmt-srv1.campus.lab    |  10.10.40.10 |
| server1.campus.lab      |  10.10.70.10 |
| internet-srv.campus.lab | 203.0.113.10 |
| dns.campus.lab          |  10.10.40.10 |
| ntp.campus.lab          |  10.10.40.10 |
| syslog.campus.lab       |  10.10.40.10 |

DNS validation was completed using `dig` and `nslookup`.

Example successful records:

```text
server1.campus.lab      → 10.10.70.10
mgmt-srv1.campus.lab    → 10.10.40.10
internet-srv.campus.lab → 203.0.113.10
google.com              → external DNS forwarding successful
```

---

## 10. NAT/PAT Design

EDGE1 provides enterprise NAT/PAT for internal networks.

EDGE1 uses inside and outside NAT roles:

| Interface | Role        |
| --------- | ----------- |
| G0/0      | NAT outside |
| G0/1      | NAT outside |
| G0/2      | NAT inside  |
| G0/3      | NAT inside  |

EDGE1 uses dynamic PAT toward the preferred ISP-facing interface.

A key observation from this lab:

```text
Only one dynamic PAT statement using the same ACL can be active at a time when configured in this simple form.
```

When the PAT statement is configured toward G0/0, the dynamic mapping uses G0/0.
When the PAT statement is changed to G0/1, the G0/0 mapping is replaced.

This behavior was observed and recorded as part of the NAT/PAT troubleshooting process.

---

## 11. NTP Design

MGMT-SRV1 provides NTP service using `chrony`.

Chrony was installed and configured on MGMT-SRV1.

MGMT-SRV1 successfully synchronized with external NTP sources and listened on UDP/123.

Cisco infrastructure devices were configured to use MGMT-SRV1 as the NTP server:

```cisco
ntp server 10.10.40.10
```

Validated devices:

* CORE1
* CORE2
* EDGE1

NTP validation commands included:

```cisco
show ntp associations
show ntp status
show clock
```

MGMT-SRV1 validation commands included:

```bash
chronyc sources -v
chronyc tracking
sudo ss -lnup | grep ':123'
```

---

## 12. Syslog Design

MGMT-SRV1 provides centralized Syslog collection using `rsyslog`.

Rsyslog was configured to listen on UDP/514.

MGMT-SRV1 validation:

```bash
sudo ss -lnup | grep ':514'
```

Expected result:

```text
0.0.0.0:514
[::]:514
```

Cisco devices were configured to send Syslog messages to MGMT-SRV1:

```cisco
logging host 10.10.40.10
logging trap informational
```

Syslog source interfaces:

| Device | Source Interface   |
| ------ | ------------------ |
| CORE1  | Vlan40             |
| CORE2  | Vlan40             |
| EDGE1  | GigabitEthernet0/2 |

Syslog messages were stored under:

```text
/var/log/campus-network/
```

CORE1 Syslog validation example:

```text
/var/log/campus-network/10.10.40.1.log
```

Observed messages included:

```text
%SYS-6-LOGGINGHOST_STARTSTOP
%SYS-5-CONFIG_I
```

---

## 13. SSH Management Access

SSH management was configured on:

* CORE1
* CORE2
* EDGE1

Each device was configured with:

```cisco
ip domain-name campus.lab
username admin privilege 15 secret <configured-password>
ip ssh version 2
ip ssh time-out 60
ip ssh authentication-retries 3
```

VTY configuration:

```cisco
line vty 0 4
 login local
 transport input ssh
```

EDGE1 required RSA key generation:

```cisco
crypto key generate rsa modulus 2048
```

SSH validation was completed from MGMT-SRV1 to:

```text
CORE1 → 10.10.40.1
CORE2 → 10.10.40.2
EDGE1 → 172.16.0.1
```

Because Ubuntu 24.04 OpenSSH rejects some older Cisco IOSv SSH algorithms by default, the following compatibility options were required:

```bash
ssh -oKexAlgorithms=+diffie-hellman-group14-sha1 \
-oHostKeyAlgorithms=+ssh-rsa \
-oPubkeyAcceptedAlgorithms=+ssh-rsa \
admin@<device-ip>
```

Successful prompts:

```text
CORE1#
CORE2#
EDGE1#
```

---

## 14. SNMP Monitoring

SNMPv2c read-only monitoring was configured on:

* CORE1
* CORE2
* EDGE1

SNMP community:

```text
NMS-RO
```

SNMP ACL:

```cisco
ip access-list standard SNMP-MGMT
 permit 10.10.40.10
 deny any
```

SNMP community configuration:

```cisco
snmp-server community NMS-RO RO SNMP-MGMT
snmp-server location Collapsed-Core-Campus-Ver2
snmp-server contact NetOps-Lab
```

MGMT-SRV1 was used as the SNMP monitoring station.

SNMP tools installed on MGMT-SRV1:

```text
snmpwalk
snmpget
```

SNMP sysName OID validation:

```bash
snmpwalk -v2c -c NMS-RO 10.10.40.1 1.3.6.1.2.1.1.5.0
snmpwalk -v2c -c NMS-RO 10.10.40.2 1.3.6.1.2.1.1.5.0
snmpwalk -v2c -c NMS-RO 172.16.0.1 1.3.6.1.2.1.1.5.0
```

Successful results:

```text
CORE1.campus.lab
CORE2.campus.lab
EDGE1.campus.lab
```

Interface description OID validation was also completed:

```bash
1.3.6.1.2.1.2.2.1.2
```

This returned interface names such as:

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

A key troubleshooting note:

```text
sysName.0 failed because Net-SNMP did not resolve the MIB name.
Numeric OID queries worked successfully.
```

---

## 15. AAA Local and Management Access Hardening

AAA Local was configured on:

* CORE1
* CORE2
* EDGE1

AAA configuration:

```cisco
aaa new-model
aaa authentication login default local
aaa authorization exec default local
aaa session-id common
```

Management access ACL:

```cisco
ip access-list standard MGMT-ACCESS
 permit 10.10.40.10
 deny any
```

VTY hardening:

```cisco
line vty 0 4
 access-class MGMT-ACCESS in
 transport input ssh
```

This restricts management access to MGMT-SRV1 only.

SSH access was revalidated from MGMT-SRV1 after applying AAA and the management ACL.

Successful validation:

```text
MGMT-SRV1 → CORE1 SSH successful
MGMT-SRV1 → CORE2 SSH successful
MGMT-SRV1 → EDGE1 SSH successful
```

Telnet is blocked by configuration because VTY accepts only SSH:

```cisco
transport input ssh
```

---

## 16. Config Backup

MGMT-SRV1 was used as the centralized Config Backup Server.

Backup directory:

```text
/srv/config-backups/
```

Backup subdirectories:

```text
/srv/config-backups/CORE1/
/srv/config-backups/CORE2/
/srv/config-backups/EDGE1/
```

Backed up files:

```text
CORE1-running-config-2026-06-06.txt
CORE2-running-config-2026-06-06.txt
EDGE1-running-config-2026-06-06.txt
```

Validation command:

```bash
grep -H "^hostname" /srv/config-backups/*/*running-config-$(date +%F).txt
```

Successful result:

```text
hostname CORE1
hostname CORE2
hostname EDGE1
```

Important distinction:

```text
Section 11 Config Backup backs up Cisco network device configurations to MGMT-SRV1.
The later MGMT-SRV1 Enterprise Management Server backup protects MGMT-SRV1's own service configuration.
```

---

## 17. DHCP Reservation and DNS Static Records

A DHCP reservation example was configured using dnsmasq.

Reservation example:

```text
MAC: 52:54:00:71:87:ac
IP : 10.10.10.136
Name: exec
```

dnsmasq reservation entry:

```conf
dhcp-host=52:54:00:71:87:ac,10.10.10.136,exec,12h
```

Validation from the client:

```text
lease of 10.10.10.136 obtained from 10.10.40.10
```

The client also confirmed:

```text
IP address      : 10.10.10.136/24
Default gateway : 10.10.10.254
```

DNS static record validation was completed using:

```bash
dig @10.10.40.10 server1.campus.lab A
dig @10.10.40.10 mgmt-srv1.campus.lab A
dig @10.10.40.10 internet-srv.campus.lab A
```

Successful results:

```text
server1.campus.lab      → 10.10.70.10
mgmt-srv1.campus.lab    → 10.10.40.10
internet-srv.campus.lab → 203.0.113.10
```

---

## 18. MGMT-SRV1 Enterprise Management Server Backup

After completing all Enterprise Services, MGMT-SRV1 itself was backed up.

Backup archive:

```text
/home/cisco/phase02-mgmt-srv1-enterprise-services-2026-06-06.tar.gz
```

Included files and directories:

```text
/etc/dnsmasq.d/campus-dns.conf
/etc/chrony/chrony.conf
/etc/rsyslog.d/10-campus-network.conf
/srv/config-backups/
```

This backup protects the configuration of MGMT-SRV1 itself.

This is different from Cisco Config Backup:

| Backup Type         | Purpose                                                            |
| ------------------- | ------------------------------------------------------------------ |
| Cisco Config Backup | Backs up CORE1/CORE2/EDGE1 running-configs                         |
| MGMT-SRV1 Backup    | Backs up MGMT-SRV1 service configuration and stored device backups |

In short:

```text
Section 11 = MGMT-SRV1 backs up Cisco devices
MGMT-SRV1 backup = MGMT-SRV1 backs up itself
```

Both are required for operational recovery.

---

## 19. Final Verification Summary

The following items were successfully verified:

| Function                       | Result                           |
| ------------------------------ | -------------------------------- |
| MGMT-SRV1 IP and default route | Successful                       |
| dnsmasq DHCP/DNS service       | Active                           |
| DHCP relay                     | Successful                       |
| DHCP lease allocation          | Successful                       |
| DHCP reservation               | Successful                       |
| DNS forwarding                 | Successful                       |
| DNS static records             | Successful                       |
| NAT/PAT                        | Successful                       |
| Chrony NTP service             | Active                           |
| Cisco NTP clients              | Successful for core/edge devices |
| Rsyslog Syslog service         | Active                           |
| Cisco Syslog forwarding        | Successful                       |
| SSH management                 | Successful                       |
| SNMPv2c monitoring             | Successful                       |
| AAA Local                      | Successful                       |
| Management ACL                 | Successful                       |
| Config Backup                  | Successful                       |
| MGMT-SRV1 service backup       | Successful                       |

---

## 20. Key Troubleshooting Notes

Important troubleshooting observations from this phase:

1. Ubuntu package installation may appear stuck because of `needrestart` output.
2. `Ctrl + Q` may be required if the Linux terminal enters XOFF state.
3. Ubuntu 24.04 OpenSSH may reject older Cisco IOSv KEX algorithms.
4. SSH to Cisco IOSv required compatibility options.
5. Net-SNMP MIB names such as `sysName.0` may fail if MIB names are not loaded.
6. Numeric OIDs worked successfully for SNMP validation.
7. `dnsmasq` may show `Failed to set DNS config` while the service remains active and functional.
8. `tar: Removing leading '/' from member names` is a normal tar safety message.
9. SERVER1 is not the DNS/DHCP server; MGMT-SRV1 provides those services.
10. SERVER1 uses static IP `10.10.70.10/24` and is registered in DNS.

---

## 21. Design Decisions

Major design decisions in Phase 02:

* Use MGMT-SRV1 as the centralized Enterprise Management Server.
* Keep DHCP/DNS/NTP/Syslog/SNMP/Config Backup centralized.
* Use `dnsmasq` for lightweight DHCP and DNS resolver functionality.
* Use `chrony` for NTP.
* Use `rsyslog` for centralized logging.
* Use SNMPv2c read-only with ACL restriction for this phase.
* Restrict SNMP access to MGMT-SRV1 only.
* Restrict SSH management access to MGMT-SRV1 only.
* Use AAA Local instead of TACACS+/RADIUS in Phase 02.
* Keep TACACS+, RADIUS, PKI, NetFlow, and Telemetry for later phases.
* Use Config Backup as an operational recovery mechanism.
* Back up MGMT-SRV1 itself after building Enterprise Management Server services.

---

## 22. Completion Status

Phase 02 Enterprise Services Ver.2 is complete.

MGMT-SRV1 now provides centralized enterprise services for the campus network, and CORE1, CORE2, and EDGE1 are integrated into the management plane through SSH, Syslog, SNMP, NTP, and configuration backup.

This phase establishes the operational services foundation required before moving into later security, QoS, resiliency, WAN, automation, and operations phases.
