# PHASE 04 — Security Design Decisions

---

# 1. Maintain Collapsed-Core Campus Architecture

Phase-04 Security implementation intentionally preserved the existing collapsed-core campus design from previous phases.

The following architectural principles remained unchanged:

- CORE1 / CORE2 collapsed-core design
- Dual-homed access switches
- HSRP gateway redundancy
- OSPF limited to EDGE and CORE devices
- Access layer excluded from OSPF participation
- VLAN999 native blackhole design
- Centralized management services

Security features were added without redesigning the campus topology.

---

# 2. Security Features Focused on Practical Campus Layer 2 Protection

Phase-04 intentionally focused on practical enterprise campus Layer 2 security controls commonly used in real production environments and Cisco Enterprise certifications.

Primary security objectives:

- Rogue DHCP mitigation
- Unauthorized switch detection
- MAC spoofing prevention
- ARP spoofing mitigation
- DHCP starvation protection
- IP spoofing protection

Advanced technologies such as:
- 802.1X
- CoPP
- TrustSec
- SD-Access
- Advanced NAC architectures

were intentionally excluded to maintain practical operational focus.

---

# 3. DHCP Snooping as the Security Foundation

DHCP Snooping was intentionally implemented before:
- DAI
- IPSG

because both features depend on the DHCP Snooping binding database.

This reflects real enterprise campus design dependency order:

```text
DHCP Snooping
    ↓
DAI
    ↓
IP Source Guard
```

---

# 4. DHCP Snooping Trust Boundary Design

Only uplink interfaces toward trusted infrastructure devices were configured as trusted ports.

Trusted interfaces:
- ACC uplinks toward CORE switches

Untrusted interfaces:
- End-host access ports
- ATTACKER-PC access interfaces

This design ensures rogue DHCP servers connected at the access layer cannot distribute unauthorized DHCP offers.

---

# 5. DHCP Option 82 Disabled

The following command was intentionally configured:

```cisco
no ip dhcp snooping information option
```

Reason:

The lab environment used:
- dnsmasq
- Alpine Linux
- CML virtual networking

which demonstrated DHCP processing issues when DHCP Option 82 insertion remained enabled.

Disabling Option 82 improved lab stability and DHCP interoperability.

---

# 6. DHCP Snooping Rate Limiting Applied Only on Access Ports

DHCP rate limiting was intentionally configured only on untrusted access ports.

Example:

```cisco
ip dhcp snooping limit rate 5
```

Reason:

- Protect against DHCP starvation attacks
- Protect against excessive DHCP floods
- Avoid impacting legitimate infrastructure DHCP traffic

Rate limiting was not applied to trusted uplinks.

---

# 7. Port Security Using Sticky MAC Learning

Port Security used:
- sticky MAC learning
- maximum MAC count of 1
- shutdown violation mode

Example:

```cisco
switchport port-security
switchport port-security maximum 1
switchport port-security mac-address sticky
switchport port-security violation shutdown
```

Reason:

Sticky learning allows operational simplicity while still protecting against:
- unauthorized endpoints
- MAC spoofing
- unmanaged device connections

Shutdown violation mode was intentionally selected to provide strong visibility during violations.

---

# 8. Dynamic ARP Inspection Based on DHCP Snooping Bindings

DAI was intentionally configured using DHCP Snooping bindings rather than static ARP ACLs.

Reason:

- Better scalability
- Dynamic host learning
- Simplified operational management
- More realistic enterprise deployment model

This allowed the switch to validate:
- IP-to-MAC relationships
- ARP packet legitimacy

using dynamically learned DHCP information.

---

# 9. DAI Validation Features Enabled for Additional Security

Additional DAI validation features were tested:

```cisco
ip arp inspection validate src-mac dst-mac ip
```

Reason:

Enable deeper ARP packet inspection including:
- Ethernet source MAC validation
- Ethernet destination MAC validation
- ARP IP field validation

The lab also demonstrated that aggressive validation can occasionally create interoperability issues in virtualized lab environments.

---

# 10. IP Source Guard Implemented for IP Spoofing Protection

IP Source Guard was enabled on access interfaces using DHCP Snooping bindings.

Example:

```cisco
ip verify source
```

Reason:

Prevent hosts from:
- spoofing source IP addresses
- bypassing IP-based access restrictions
- impersonating other endpoints

The lab additionally demonstrated IOSvL2/CML behavioral limitations where IPSG may become overly restrictive in virtual environments.

---

# 11. BPDU Guard Applied to Access Ports

BPDU Guard was enabled on access ports configured for PortFast operation.

Example:

```cisco
spanning-tree portfast edge
spanning-tree bpduguard enable
```

Reason:

Protect the campus topology against:
- unauthorized switch insertion
- accidental Layer 2 loops
- STP manipulation attempts

This follows common enterprise campus best practices.

---

# 12. ATTACKER-PC Used as a Dedicated Adversarial Endpoint

A dedicated ATTACKER-PC node was intentionally introduced rather than repeatedly repurposing production endpoints.

Reason:

- Cleaner attack simulation
- Easier recovery procedures
- More realistic security testing
- Better separation between legitimate and malicious traffic

The ATTACKER-PC performed:
- Rogue DHCP attacks
- DHCP flooding
- MAC spoofing
- ARP spoofing
- IP spoofing tests

throughout the phase.

---

# 13. Security Recovery Procedures Included in Lab Scope

Phase-04 intentionally included:
- err-disable recovery
- DHCP lease restoration
- interface recovery
- MAC restoration
- DHCP renewal procedures

rather than only demonstrating attack detection.

Reason:

Operational recovery procedures are critical in real enterprise environments and align closely with:
- ENARSI operational workflows
- enterprise troubleshooting practices
- real-world change management

---

# 14. GitHub Configurations Treated as Source of Truth

Lab YAML export alone was determined insufficient for reliable configuration persistence.

The following workflow became the official backup procedure:

```text
1. Save device configurations
2. Extract Configs
3. Download Lab
4. Re-import verification
5. GitHub upload
```

Reason:

CML topology exports may not reliably preserve startup-config synchronization unless configurations are explicitly extracted into the topology database.

GitHub markdown/config archives therefore became the authoritative source of truth for lab recovery.

---

# 15. Security Scope Intentionally Separated from QoS Scope

Although the original Phase-04 plan included:
- Security
- QoS

the implementation intentionally concluded after the Security section.

QoS topics were deferred to a future dedicated phase to:
- simplify documentation
- maintain cleaner learning boundaries
- improve GitHub organization
- prevent excessive phase complexity

Phase-04 therefore became a dedicated Security-focused phase.
