# PHASE 02 — Enterprise Services Design Decisions

This document explains the architectural and operational design decisions made during the Phase 02 Enterprise Services implementation.

---

# 1. External Connector Migration

## Decision

The original ISP node was removed and replaced with a direct connection between EDGE1 and the CML External Connector.

---

## Reason

The original ISP node was an unconfigured IOSv router that did not provide:

- DHCP
- NAT
- Internet transit
- Default gateway services

As a result:

- EDGE1 could not obtain an IP address
- No default route was installed
- MGMT-SRV1 could not reach the Internet
- Ubuntu package installation failed

Replacing the ISP node with the External Connector provided:

- Direct DHCP-based WAN connectivity
- Automatic default route installation
- Simplified Internet access design

---

## Result

EDGE1 successfully obtained a DHCP address on G0/0 and provided NAT/PAT services for internal VLANs.

---

# 2. Centralized Management Server Design

## Decision

The dedicated dnsmasq and syslog nodes from Phase 01 were removed.

Instead, a single Ubuntu-based MGMT-SRV1 node was used as a centralized management server.

---

## Reason

The original design used separate nodes for:

- dnsmasq
- Syslog
- Management utilities

However, real-world enterprise environments commonly consolidate management services onto a centralized Linux server.

The centralized Ubuntu server model provides:

- Reduced topology complexity
- Fewer nodes to manage
- Easier operational visibility
- More realistic infrastructure management experience

---

## Services Consolidated onto MGMT-SRV1

MGMT-SRV1 was configured to provide:

- DHCP services (dnsmasq)
- DNS forwarding
- Syslog collection
- SNMP monitoring tools
- Linux management utilities

---

## Operational Benefits

This approach also provided hands-on troubleshooting experience involving:

- DNS resolver recovery
- Linux routing issues
- Service restart operations
- Package installation failures
- Interface binding problems
- Internet reachability troubleshooting

---

# 3. Routed Core-to-Edge Design

## Decision

CORE1 and CORE2 uplinks to EDGE1 were converted from Layer 2 switchports to Layer 3 routed ports.

---

## Reason

Default routes on CORE1 and CORE2 could not be installed while uplinks remained Layer 2 interfaces.

Converting uplinks to routed ports provided:

- Point-to-point Layer 3 connectivity
- Simplified routing behavior
- Direct next-hop reachability to EDGE1
- Better enterprise core-edge separation

---

## Implemented Routed Links

| Link | Subnet |
|---|---|
| CORE1 ↔ EDGE1 | 10.255.255.0/30 |
| CORE2 ↔ EDGE1 | 10.255.255.4/30 |

---

# 4. Centralized NTP Hierarchy

## Decision

CORE1 was configured as the internal NTP master server.

All other network devices were configured as NTP clients.

---

## Reason

Consistent timestamps are critical for:

- Syslog correlation
- Troubleshooting
- Event sequencing
- Operational visibility

Using a centralized NTP hierarchy ensures all devices share the same time reference.

---

## NTP Design

| Role | Device |
|---|---|
| NTP Master | CORE1 |
| NTP Clients | CORE2, EDGE1, ACC Switches |

---

# 5. SSH-Only Remote Management

## Decision

Telnet access was disabled and SSHv2-only remote management was implemented.

---

## Reason

Telnet transmits credentials in plaintext and is considered insecure.

SSH provides:

- Encrypted management sessions
- Secure administrator authentication
- Enterprise-standard remote access

---

## Security Enhancements

The following controls were implemented:

- SSH Version 2
- Local username authentication
- RSA key generation
- Password encryption
- SSH-only VTY transport

---

# 6. SNMP Access Restriction

## Decision

SNMP access was restricted using an ACL.

Only the management subnet was permitted to perform SNMP polling.

---

## Reason

SNMPv2c uses plaintext community strings.

Restricting access reduced exposure and simulated enterprise monitoring design practices.

---

## Allowed Management Subnet

10.20.20.0/24

---

# 7. Centralized Syslog Architecture

## Decision

All network devices were configured to send Syslog messages to MGMT-SRV1.

---

## Reason

Centralized logging improves:

- Troubleshooting
- Operational visibility
- Event tracking
- Network monitoring

This design also simulated enterprise SIEM-style centralized logging architecture.

---

## Syslog Server

| Device | Role |
|---|---|
| MGMT-SRV1 | Central Syslog Server |

---

# 8. Dedicated Management VLAN

## Decision

A dedicated Management VLAN (VLAN 40) was implemented.

---

## Reason

Separating management traffic from user/server traffic improves:

- Operational organization
- Device management consistency
- Security segmentation
- Troubleshooting simplicity

---

## Management Subnet

10.40.40.0/24

---

# 9. DHCP Relay Architecture

## Decision

CORE1 and CORE2 were configured as DHCP relay agents using ip helper-address.

MGMT-SRV1 hosted the centralized DHCP service.

---

## Reason

DHCP broadcasts cannot cross VLAN boundaries.

Using relay agents allowed centralized DHCP management while supporting multiple VLANs.

This approach reflects real enterprise campus designs where DHCP services are centralized rather than deployed separately inside every VLAN.

---

## DHCP Flow

Client VLAN → CORE SVI (DHCP Relay) → MGMT-SRV1 DHCP Server

---

# DHCP Server

| Device | Role |
|---|---|
| MGMT-SRV1 | Central DHCP Server |

---

# DHCP Relay Devices

| Device | Function |
|---|---|
| CORE1 | DHCP Relay Agent |
| CORE2 | DHCP Relay Agent |

---

# Operational Benefits

This design provided experience with:

- Centralized DHCP services
- DHCP relay forwarding
- Inter-VLAN DHCP operations
- Linux-based DHCP server deployment
- Enterprise-style IP address management

---

# 10. Intentional Design Constraints

## Decision

The following technologies were intentionally excluded:

- Layer 2 EtherChannel
- Layer 3 ECMP

---

## Reason

The purpose of this lab series is to strengthen understanding of:

- STP behavior
- HSRP failover
- Static routing
- Routed uplinks
- Layer 2 convergence
- Enterprise redundancy mechanisms

Avoiding EtherChannel and ECMP forces deeper visibility into failover and forwarding behavior.





