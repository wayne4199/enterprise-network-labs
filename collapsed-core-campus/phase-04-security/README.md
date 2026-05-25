# PHASE 04 — Security

## Overview

Phase-04 focused on practical enterprise campus Layer 2 security features using Cisco IOSvL2 switches inside a Cisco CML-based collapsed-core campus topology.

This phase expanded the previous routing resiliency design by implementing:
- Rogue DHCP protection
- DHCP starvation mitigation
- MAC spoofing protection
- ARP spoofing mitigation
- IP spoofing mitigation
- Rogue switch detection
- Layer 2 attack recovery procedures

The lab emphasized operationally realistic campus security controls commonly used in enterprise networks and Cisco Enterprise certification tracks such as:
- ENCOR
- ENARSI
- ENSLD
- ENSDWI
- ENCC

---

# Topology Scope

The following architecture from previous phases was preserved:

- Collapsed-core campus design
- CORE1 / CORE2 dual-core architecture
- Dual-homed access switches
- HSRP gateway redundancy
- OSPF limited to EDGE and CORE devices
- VLAN999 native blackhole design
- Centralized management services

Phase-04 intentionally avoided redesigning the topology and instead layered security features onto the existing enterprise campus architecture.

---

# Security Features Implemented

## 1. Rogue DHCP Attack Simulation

A dedicated ATTACKER-PC endpoint was used to simulate:
- Rogue DHCP Offers
- Fake Default Gateway advertisements
- Fake DNS server advertisements

The attack simulation demonstrated how unauthorized endpoints may attempt to distribute malicious network configuration information inside enterprise access VLANs.

---

## 2. DHCP Snooping

DHCP Snooping was implemented as the foundational Layer 2 security mechanism.

Features validated:
- Trusted vs untrusted interfaces
- Rogue DHCP detection
- DHCP binding table creation
- DHCP packet filtering
- DHCP statistics monitoring

The lab additionally demonstrated Option 82 interoperability considerations in virtualized environments.

---

## 3. DHCP Snooping Rate Limiting

DHCP rate limiting was applied to access interfaces to protect against:
- DHCP starvation attacks
- Excessive DHCP floods
- Resource exhaustion scenarios

The lab verified automatic err-disable behavior when DHCP request rates exceeded configured thresholds.

---

## 4. Port Security

Port Security was implemented using:
- Sticky MAC learning
- Maximum MAC address limits
- Shutdown violation mode

The lab validated:
- Sticky MAC learning
- MAC spoofing detection
- Unauthorized endpoint blocking
- Err-disable recovery procedures

---

## 5. Dynamic ARP Inspection (DAI)

DAI was configured using DHCP Snooping bindings.

The lab validated:
- ARP packet inspection
- ARP spoofing mitigation
- Gratuitous ARP filtering
- DHCP binding dependency
- DAI statistics monitoring

Additional validation features were also tested:
- Source MAC validation
- Destination MAC validation
- IP address validation

---

## 6. IP Source Guard (IPSG)

IP Source Guard was implemented using DHCP Snooping binding information.

The lab validated:
- Source IP filtering
- DHCP binding dependency
- IP spoofing mitigation concepts

The lab also demonstrated operational limitations occasionally observed in virtualized IOSvL2/CML environments.

---

## 7. BPDU Guard / Rogue Switch Protection

BPDU Guard was enabled on PortFast-enabled access interfaces.

The lab validated:
- Rogue switch detection
- BPDU-based err-disable protection
- STP manipulation prevention
- Interface recovery procedures

---

# Security Validation Scope

The following attack and protection scenarios were successfully tested:

| Feature | Validation |
|---|---|
| Rogue DHCP | Successful detection and mitigation |
| DHCP Flood | Successful rate limiting and err-disable |
| MAC Spoofing | Successful Port Security violation |
| ARP Spoofing | Successful DAI packet drops |
| IP Spoofing | IPSG source filtering validation |
| Rogue Switch | Successful BPDU Guard err-disable |

---

# Recovery and Operational Procedures

Phase-04 intentionally included operational recovery workflows such as:
- Err-disable recovery
- DHCP lease restoration
- Interface reset procedures
- MAC restoration
- DHCP renewal operations

This aligned the lab with real enterprise operational troubleshooting practices rather than purely theoretical attack demonstrations.

---

# ATTACKER-PC Usage

A dedicated ATTACKER-PC endpoint was used throughout the phase to simulate adversarial behavior including:
- Rogue DHCP attacks
- DHCP flooding
- MAC spoofing
- ARP spoofing
- IP spoofing

Using a dedicated adversarial endpoint simplified:
- attack isolation
- troubleshooting
- recovery operations
- operational consistency

---

# CML Configuration Persistence Lessons Learned

During Phase-04, multiple configuration persistence issues were encountered inside CML.

The following workflow was validated as the most reliable backup method:

```text
1. Save device configurations (wr)
2. NODES → Select All Nodes → Extract Configs
3. LAB → Download Lab
4. Re-import verification test
5. GitHub backup
```

This process ensured:
- topology persistence
- startup-config synchronization
- reliable lab restoration

---

# Files Included

```text
phase-04-security-qos/
├── README.md
├── topology.drawio
├── topology.png
├── lab-export.yaml
├── configs.md
├── troubleshooting.md
├── verification.md
└── design-decisions.md
```

---

# Key Lessons Learned

Phase-04 demonstrated that enterprise campus security is built using multiple complementary Layer 2 protection mechanisms working together.

Key operational lessons included:
- DHCP Snooping dependency relationships
- Trusted boundary design
- Access-layer attack surfaces
- Err-disable operational recovery
- Dynamic security binding behavior
- Virtual lab interoperability considerations
- Importance of configuration persistence verification

The phase also reinforced the operational principle that:
- recovery procedures are as important as attack detection
- GitHub configuration archives should be treated as the authoritative source of truth
- lab exports alone are insufficient without configuration extraction verification

---

# Next Phase

QoS topics were intentionally deferred to the next phase in order to maintain:
- cleaner documentation boundaries
- simpler GitHub organization
- more focused learning objectives

The next phase will focus on:
- QoS classification
- DSCP marking
- MQC
- Queuing
- Voice trust boundaries
- Enterprise campus QoS design
