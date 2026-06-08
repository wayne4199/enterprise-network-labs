# Phase 02 — Enterprise Services Ver.2 Design Decisions

## 1. Purpose of This Document

This document records the major design decisions made during **Phase 02 — Enterprise Services Ver.2** of the Collapsed Core Campus lab.

The purpose is to explain not only what was configured, but also why each design choice was made.

Phase 02 focuses on building a practical enterprise-style services and management foundation before moving into later phases such as security, QoS, routing resiliency, WAN, automation, and operations.

---

## 2. MGMT-SRV1 as the Enterprise Management Server

### Decision

MGMT-SRV1 is designed as the centralized **Enterprise Management Server** for the campus network.

MGMT-SRV1 is not a simple standalone server.

It provides and centralizes the following services:

* DHCP Server
* DNS Resolver
* NTP Server
* Syslog Server
* SNMP Monitoring Station
* Config Backup Server

### Reason

Enterprise networks usually centralize operational services instead of spreading them randomly across network devices.

By using MGMT-SRV1 as the Enterprise Management Server, the lab creates a realistic operational management point for:

* IP address assignment
* Name resolution
* Time synchronization
* Log collection
* Device monitoring
* Configuration recovery

This design makes the lab closer to a real campus network operations model.

### Result

MGMT-SRV1 became the central operational services node for Phase 02.

---

## 3. Centralized DHCP Using dnsmasq

### Decision

Use `dnsmasq` on MGMT-SRV1 as the centralized DHCP server.

### Reason

`dnsmasq` is lightweight and suitable for lab environments.
It can provide both DHCP and DNS resolver functions from the same server.

This keeps the service model simple while still allowing realistic enterprise concepts such as:

* DHCP scopes
* DHCP relay
* Default gateway assignment
* DNS server assignment
* Domain name option
* DHCP reservation

### Result

MGMT-SRV1 provides DHCP services for multiple VLANs.

The collapsed core switches relay DHCP requests to MGMT-SRV1 using:

```cisco
ip helper-address 10.10.40.10
```

---

## 4. DHCP Relay on CORE1 and CORE2

### Decision

Configure DHCP relay on CORE1 and CORE2 SVI interfaces instead of placing DHCP directly on the core switches.

### Reason

In enterprise designs, DHCP is usually centralized on a server rather than configured locally on Layer 3 gateways.

Using DHCP relay provides a more realistic design:

```text
Client VLAN → CORE SVI → DHCP Relay → MGMT-SRV1
```

This also separates Layer 3 gateway functions from enterprise service functions.

### Result

CORE1 and CORE2 provide the gateway and relay functions, while MGMT-SRV1 provides DHCP service.

---

## 5. DNS Resolver and Static Records on MGMT-SRV1

### Decision

Use MGMT-SRV1 as the internal DNS resolver for the campus network.

### Reason

Enterprise networks usually need both:

* External name resolution
* Internal service name resolution

MGMT-SRV1 forwards external queries to public DNS servers and answers internal campus records locally.

### Internal Domain

```text
campus.lab
```

### Static Records

```text
mgmt-srv1.campus.lab     → 10.10.40.10
server1.campus.lab       → 10.10.70.10
internet-srv.campus.lab  → 203.0.113.10
dns.campus.lab           → 10.10.40.10
ntp.campus.lab           → 10.10.40.10
syslog.campus.lab        → 10.10.40.10
```

### Result

Clients and servers can resolve both internal and external names using MGMT-SRV1.

---

## 6. Use `campus.lab` Instead of `campus.local`

### Decision

Use `campus.lab` as the internal lab DNS domain.

### Reason

The `.local` domain is commonly associated with mDNS behavior.
During testing, `.local` caused warning messages related to multicast DNS behavior.

To avoid confusion and keep the lab DNS behavior cleaner, the internal domain was changed to:

```text
campus.lab
```

### Result

DNS records were updated from `campus.local` to `campus.lab`.

---

## 7. Disable AAAA Responses with dnsmasq Filter

### Decision

Use dnsmasq filtering for AAAA records.

### Reason

The lab is IPv4-focused.
Some clients generated additional IPv6-related lookups, which could make DNS troubleshooting output confusing.

The design keeps the current Phase 02 DNS validation focused on IPv4 A records.

### Result

A record validation became easier and more predictable.

---

## 8. SERVER1 Uses Static IP Instead of DHCP Reservation

### Decision

SERVER1 uses a static IP address:

```text
10.10.70.10/24
```

and is registered in DNS as:

```text
server1.campus.lab
```

### Reason

SERVER1 represents a server VLAN resource.
Important infrastructure servers are commonly addressed statically or managed through carefully controlled reservation policies.

During verification, SERVER1 was confirmed as a static server node rather than a DHCP client.

### Result

SERVER1 was documented as a static server and DNS static record, not as the DHCP reservation example.

---

## 9. DHCP Reservation Used for Client Example

### Decision

Use a DHCP client in VLAN10 as the DHCP reservation example.

Reservation example:

```text
MAC: 52:54:00:71:87:ac
IP : 10.10.10.136
Name: exec
```

dnsmasq entry:

```conf
dhcp-host=52:54:00:71:87:ac,10.10.10.136,exec,12h
```

### Reason

A DHCP reservation should be tested with an actual DHCP client.

The selected client already existed in the lease database and successfully renewed the same IP address after the reservation was added.

### Result

The DHCP reservation was verified successfully.

---

## 10. NAT/PAT on EDGE1

### Decision

Use EDGE1 as the enterprise NAT/PAT boundary.

### Reason

EDGE1 is the logical edge device between the campus network and the ISP/external network.

NAT/PAT belongs at the edge because internal private networks need controlled outbound access to external networks.

### NAT Role

```text
EDGE1 internal links → ip nat inside
EDGE1 ISP-facing links → ip nat outside
```

### Result

Internal networks reached external destinations through EDGE1 NAT/PAT.

---

## 11. Simple Dynamic PAT Instead of Advanced NAT Policy

### Decision

Use simple dynamic PAT for Phase 02.

### Reason

Phase 02 is focused on enterprise services, not advanced WAN policy or SD-WAN traffic steering.

The purpose of this phase is to provide basic external reachability for service validation, such as:

* DNS forwarding
* Package installation
* NTP synchronization
* HTTP/HTTPS testing

More advanced WAN policy will be handled in later phases.

### Result

NAT/PAT was kept practical and simple.

---

## 12. NTP Centralized on MGMT-SRV1

### Decision

Use MGMT-SRV1 as the NTP server for the campus network.

### Reason

Accurate time is required for:

* Syslog timestamps
* Troubleshooting
* Event correlation
* Security auditing
* Configuration backup tracking

Instead of letting every device depend independently on the Internet, MGMT-SRV1 provides a centralized internal time source.

### Result

CORE1, CORE2, and EDGE1 were configured as NTP clients of MGMT-SRV1.

---

## 13. Use chrony for NTP

### Decision

Use `chrony` on MGMT-SRV1.

### Reason

Chrony is commonly used on modern Linux systems and handles time synchronization well in virtualized environments.

### Result

MGMT-SRV1 provided NTP service and listened on UDP/123.

---

## 14. Syslog Centralized on MGMT-SRV1

### Decision

Use MGMT-SRV1 as the centralized Syslog server.

### Reason

Centralized logging is a core enterprise operations requirement.

It allows device messages to be collected even when an individual device console or buffer is no longer available.

### Result

Cisco device logs were collected under:

```text
/var/log/campus-network/
```

---

## 15. Use rsyslog for Syslog Collection

### Decision

Use `rsyslog` on MGMT-SRV1.

### Reason

Rsyslog is already common on Linux and is suitable for receiving UDP/514 syslog messages from Cisco devices.

It is lightweight enough for the lab while still reflecting real operational practices.

### Result

MGMT-SRV1 listened on UDP/514 and stored logs from Cisco devices.

---

## 16. Syslog Source Interface Selection

### Decision

Use stable internal or management-facing source interfaces for Cisco Syslog.

### Source Interfaces

| Device | Source Interface   |
| ------ | ------------------ |
| CORE1  | Vlan40             |
| CORE2  | Vlan40             |
| EDGE1  | GigabitEthernet0/2 |

### Reason

Using a consistent source interface makes log identification easier and avoids unpredictable source IP behavior.

CORE1 and CORE2 have VLAN40 management SVIs.
EDGE1 does not have a VLAN40 SVI, so an internal routed interface was used.

### Result

Syslog records were received from predictable source IP addresses.

---

## 17. SSH-Only Management

### Decision

Allow SSH only for VTY access.

```cisco
transport input ssh
```

### Reason

Telnet sends credentials in clear text and should not be used for management access.

Even in a lab, SSH-only access builds the correct operational habit.

### Result

CORE1, CORE2, and EDGE1 use SSH-only management access.

---

## 18. Local Admin User for Phase 02

### Decision

Use a local privilege 15 admin account.

```cisco
username admin privilege 15 secret <configured-password>
```

### Reason

Phase 02 is focused on enterprise service foundations.

External AAA systems such as TACACS+ or RADIUS are intentionally deferred to a later phase.

A local admin account is sufficient for this phase and supports SSH access testing.

### Result

Local authentication was used successfully for SSH access.

---

## 19. AAA Local Instead of TACACS+ or RADIUS

### Decision

Use local AAA.

```cisco
aaa new-model
aaa authentication login default local
aaa authorization exec default local
```

### Reason

The goal of Phase 02 is to introduce structured management authentication without adding external AAA complexity.

TACACS+ and RADIUS require additional infrastructure and are better handled in a later operations/security phase.

### Result

Local AAA was configured and verified.

---

## 20. Management Access ACL

### Decision

Restrict VTY management access to MGMT-SRV1 only.

```cisco
ip access-list standard MGMT-ACCESS
 permit 10.10.40.10
 deny any
```

Applied under VTY:

```cisco
access-class MGMT-ACCESS in
```

### Reason

Management access should not be reachable from arbitrary user or guest networks.

Only the enterprise management server should initiate management sessions to infrastructure devices.

### Result

SSH access from MGMT-SRV1 was verified successfully after applying the ACL.

---

## 21. SNMPv2c Read-Only for Phase 02

### Decision

Use SNMPv2c read-only community for this phase.

```cisco
snmp-server community NMS-RO RO SNMP-MGMT
```

### Reason

SNMPv2c is simple and useful for learning monitoring basics.

The goal in this phase is to understand:

* Managed devices
* Monitoring station
* Community strings
* Read-only monitoring
* SNMP ACL restriction
* OID queries

SNMPv3 is more secure but adds additional complexity and is better placed in a later operations/security phase.

### Result

MGMT-SRV1 successfully queried CORE1, CORE2, and EDGE1 using SNMPv2c.

---

## 22. SNMP Access Restricted to MGMT-SRV1

### Decision

Use an SNMP ACL to restrict SNMP queries to MGMT-SRV1.

```cisco
ip access-list standard SNMP-MGMT
 permit 10.10.40.10
 deny any
```

### Reason

SNMP community strings should not be usable from arbitrary hosts.

Even though SNMPv2c does not provide strong security, access restriction improves the management plane design.

### Result

SNMPv2c read-only access was limited to MGMT-SRV1.

---

## 23. Numeric OID Queries for SNMP Validation

### Decision

Use numeric OIDs during SNMP validation.

Examples:

```text
1.3.6.1.2.1.1.5.0
1.3.6.1.2.1.2.2.1.2
```

### Reason

Net-SNMP on MGMT-SRV1 did not resolve MIB names such as `sysName.0`.

Numeric OIDs avoid dependency on local MIB name loading and validate SNMP communication directly.

### Result

SNMP validation succeeded using numeric OIDs.

---

## 24. Config Backup to MGMT-SRV1

### Decision

Use MGMT-SRV1 as the centralized backup destination for Cisco running-config files.

Backup path:

```text
/srv/config-backups/
```

### Reason

Network devices should not be the only place where their own configurations exist.

If a device is reset, corrupted, or misconfigured, a saved configuration backup is required for recovery.

Centralizing backups on MGMT-SRV1 supports:

* Recovery
* Comparison
* Documentation
* GitHub preparation
* Operational discipline

### Result

CORE1, CORE2, and EDGE1 running-configs were backed up to MGMT-SRV1.

---

## 25. Separate Cisco Config Backup and MGMT-SRV1 Backup

### Decision

Keep two backup concepts separate:

| Backup Type         | Target                  | Purpose                                       |
| ------------------- | ----------------------- | --------------------------------------------- |
| Cisco Config Backup | CORE1 / CORE2 / EDGE1   | Recover network device configurations         |
| MGMT-SRV1 Backup    | MGMT-SRV1 service files | Recover Enterprise Management Server services |

### Reason

Section 11 Config Backup protects Cisco network devices.

The later MGMT-SRV1 backup protects the management server itself, including:

* DHCP/DNS configuration
* NTP configuration
* Syslog configuration
* Stored Cisco backups

### Result

Both backup layers were completed.

---

## 26. MGMT-SRV1 Service Backup

### Decision

Create a tar backup archive of MGMT-SRV1 service configuration.

Backup archive:

```text
/home/cisco/phase02-mgmt-srv1-enterprise-services-2026-06-06.tar.gz
```

Included items:

```text
/etc/dnsmasq.d/campus-dns.conf
/etc/chrony/chrony.conf
/etc/rsyslog.d/10-campus-network.conf
/srv/config-backups/
```

### Reason

If MGMT-SRV1 is lost or rebuilt, the enterprise service configuration must also be recoverable.

### Result

MGMT-SRV1 Enterprise Management Server configuration was backed up successfully.

---

## 27. Access Switches Deferred for Some Management Services

### Decision

Focus management service validation primarily on CORE1, CORE2, and EDGE1.

### Reason

During this phase, the core and edge devices were the most important infrastructure devices for enterprise service validation.

Some access-layer management paths may require additional cleanup and verification in later phases.

### Result

CORE1, CORE2, and EDGE1 were fully integrated with:

* NTP
* Syslog
* SSH
* SNMP
* AAA Local
* Management ACL
* Config Backup

---

## 28. Keep Phase 02 Practical, Not Overly Advanced

### Decision

Do not add TACACS+, RADIUS, PKI, NetFlow, Telemetry, or advanced monitoring in Phase 02.

### Reason

Those topics are important, but they would expand the scope too much.

Phase 02 is intended to build foundational enterprise services.

Advanced operations and security services should be introduced later after the base network services are stable.

### Result

Phase 02 remained focused and practical.

---

## 29. Preserve Phase 01 Foundation

### Decision

Do not redesign the Phase 01 switching and collapsed-core foundation.

### Reason

Phase 02 is an extension phase.

The purpose is to add enterprise services on top of the existing collapsed-core campus design.

### Result

The lab preserved the Phase 01 topology and added centralized services without unnecessary redesign.

---

## 30. Use Documentation-Friendly Validation

### Decision

Use validation commands that can be easily documented in GitHub.

Examples:

```bash
dig @10.10.40.10 server1.campus.lab A
snmpwalk -v2c -c NMS-RO 10.10.40.1 1.3.6.1.2.1.1.5.0
grep -H "^hostname" /srv/config-backups/*/*running-config-$(date +%F).txt
```

### Reason

The lab is not only for configuration, but also for repeatable documentation and troubleshooting practice.

### Result

The final state can be clearly recorded in:

* README.md
* configs.md
* verification.md
* troubleshooting.md
* design-decisions.md

---

## 31. Final Design Summary

Phase 02 establishes MGMT-SRV1 as the central Enterprise Management Server.

The design centralizes operational services while keeping network devices focused on routing, switching, gateway, and management-plane access.

Final service placement:

| Function                | Device        |
| ----------------------- | ------------- |
| DHCP                    | MGMT-SRV1     |
| DNS Resolver            | MGMT-SRV1     |
| NTP                     | MGMT-SRV1     |
| Syslog                  | MGMT-SRV1     |
| SNMP Tools              | MGMT-SRV1     |
| Config Backup Storage   | MGMT-SRV1     |
| Layer 3 Gateways        | CORE1 / CORE2 |
| NAT/PAT                 | EDGE1         |
| External connectivity   | ISP1 / ISP2   |
| Static server test node | SERVER1       |

This creates a practical enterprise services foundation for later lab phases.
