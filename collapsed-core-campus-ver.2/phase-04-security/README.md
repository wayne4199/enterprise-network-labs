# Phase 04 - Security

## 1. Overview

Phase 04 focuses on campus access-layer and management-plane security hardening for the `collapsed-core-campus-ver.2` lab.

The goal of this phase is to protect access-facing switch ports, validate common Layer 2 security controls, and restrict SSH management access to the dedicated Enterprise Management Server.

This phase builds on the previous phases:

* Phase 01: Campus foundation, VLANs, trunks, STP, SVIs, HSRP, static routing
* Phase 02: Enterprise services, DHCP, DNS, NTP, Syslog, SNMP, SSH, config backup
* Phase 03: Routing resiliency with OSPF, HSRP validation, IP SLA, object tracking, and WAN failover

Phase 04 does not redesign the topology. Instead, it hardens the existing campus network.

---

## 2. Phase Objectives

The main objectives of this phase are:

* Harden access-facing ports
* Prevent unauthorized endpoint MAC changes
* Protect DHCP infrastructure from rogue or abusive DHCP traffic
* Prevent ARP spoofing and invalid ARP traffic
* Validate IP Source Guard behavior
* Prevent rogue switch attachment using BPDU Guard
* Restrict SSH management access to MGMT-SRV1 only
* Document platform-specific limitations observed in CML IOSvL2

---

## 3. Security Features Covered

Phase 04 includes the following security controls:

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

---

## 4. Topology Scope

The existing collapsed-core campus topology was reused.

Primary network devices:

```text
CORE1
CORE2
EDGE1
ACC1
ACC2
ACC3
SRV-ACC1
```

Important hosts used during this phase:

```text
MGMT-SRV1
SERVER1
ROGUE-PC
ROGUE-SW-TEST
```

MGMT-SRV1 continues to function as the Enterprise Management Server.

```text
MGMT-SRV1:
- IP: 10.10.40.10
- Role: DHCP, DNS, NTP, Syslog, SNMP, SSH source, Config Backup, Management Services
```

---

## 5. Management IP Plan

Management access to switches is performed through VLAN 40.

```text
CORE1      10.10.40.1
CORE2      10.10.40.2
ACC1       10.10.40.11
ACC2       10.10.40.12
ACC3       10.10.40.13
SRV-ACC1   10.10.40.14
MGMT-SRV1  10.10.40.10
HSRP VIP   10.10.40.254
```

EDGE1 is managed through its campus-facing routed links.

```text
EDGE1 primary campus-facing IP:   172.16.0.1
EDGE1 secondary campus-facing IP: 172.16.0.5
```

WAN-facing EDGE1 addresses are not used for internal management validation.

---

## 6. Access Port Hardening

Access-facing ports were explicitly configured as access ports.

Common access port hardening items:

```text
switchport mode access
switchport nonegotiate
spanning-tree portfast edge
```

This prevents unintended trunk negotiation and ensures endpoint-facing ports behave as fixed access ports.

---

## 7. Port Security

Port Security was enabled on access-facing ports.

Final design:

```text
Maximum MAC addresses: 1
MAC learning method: sticky
Violation mode: restrict
```

The `restrict` mode was selected because it allows the interface to remain up while dropping violating traffic and incrementing violation counters. This is useful for lab validation and operational visibility.

Port Security was validated using ROGUE-PC by changing the endpoint MAC address and confirming that violation counters and security logs increased.

---

## 8. DHCP Snooping

DHCP Snooping was enabled on access-layer switches for the campus VLANs.

Protected VLANs:

```text
10, 11, 20, 21, 30, 31, 40, 50, 60, 70
```

Design rules:

```text
Trusted ports:
- Uplinks / trunk-facing ports
- DHCP server-facing port on SRV-ACC1

Untrusted ports:
- User-facing access ports
- Test endpoint ports
- Rogue switch test port

Rate limit:
- 5 packets per second on untrusted access-facing ports

Option 82:
- Disabled
```

DHCP Snooping was validated by confirming DHCP binding entries and successful DHCP lease assignment from MGMT-SRV1.

---

## 9. Dynamic ARP Inspection

Dynamic ARP Inspection was enabled on access switches to protect user and server VLANs from invalid ARP traffic.

DAI validation included:

* Normal ARP forwarding from legitimate DHCP clients
* Invalid ARP drop validation on ACC2
* Static server handling on SRV-ACC1
* DAI statistics and logs

DAI validation mode:

```text
ip arp inspection validate src-mac dst-mac ip
```

Uplink ports were trusted. Access-facing ports remained untrusted.

---

## 10. Static Source Binding for Static Servers

SRV-ACC1 has statically addressed servers connected to it.

Static servers:

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

Static source bindings were configured before enabling DAI on VLAN 40 and VLAN 70.

Reason:

```text
Dynamic ARP Inspection normally relies on DHCP Snooping bindings.
Static servers do not automatically create DHCP Snooping dynamic bindings.
Without static source bindings, valid ARP traffic from static servers can be dropped.
```

Static source bindings were selected instead of ARP ACLs because the same IP/MAC/VLAN/interface mapping can also be reused by IP Source Guard. This keeps the security policy consistent across DHCP Snooping, DAI, and IP Source Guard.

---

## 11. IP Source Guard Validation

IP Source Guard was tested on ACC2 Gi1/1 using ROGUE-PC.

Tested modes:

```text
ip verify source
ip verify source port-security
```

Observed valid binding:

```text
Interface: Gi1/1
Filter type: ip-mac
Filter mode: active
IP address: 10.10.20.126
MAC address: 52:54:00:61:B0:C0
VLAN: 20
```

The DHCP Snooping binding, Port Security sticky MAC, and `show ip verify source` output were correct.

However, in the CML IOSvL2 environment, enabling IP Source Guard caused host traffic to be dropped even when the IP/MAC/VLAN binding matched correctly.

Validation result:

```text
IP Source Guard enabled:
- ROGUE-PC traffic failed

IP Source Guard removed:
- ROGUE-PC traffic immediately recovered
```

Final decision:

```text
IP Source Guard was validated but not retained on active access ports in this phase.
```

This was documented as a CML IOSvL2 data-plane limitation or platform-specific behavior rather than a design error.

---

## 12. BPDU Guard

BPDU Guard was enabled on access-facing PortFast edge ports.

Purpose:

```text
Prevent unauthorized switches from being connected to access ports.
Protect the Layer 2 topology from unintended STP changes or loops.
```

ROGUE-SW-TEST was connected to SRV-ACC1 Gi0/3 to validate BPDU Guard.

Validation result:

```text
SRV-ACC1 Gi0/3 received BPDU from ROGUE-SW-TEST.
BPDU Guard placed Gi0/3 into err-disabled state.
Err-disabled reason: bpduguard.
```

Observed logs:

```text
%SPANTREE-2-BLOCK_BPDUGUARD: Received BPDU on port Gi0/3 with BPDU Guard enabled. Disabling port.
%PM-4-ERR_DISABLE: bpduguard error detected on Gi0/3, putting Gi0/3 in err-disable state
```

This confirms that BPDU Guard successfully protected the access layer from rogue switch attachment.

---

## 13. Management ACL / VTY SSH Access Control

SSH management access was restricted using a standard VTY ACL.

Management source allowed:

```text
MGMT-SRV1: 10.10.40.10
```

Common ACL:

```text
VTY-MGMT-ONLY
permit host 10.10.40.10
deny any log
```

Expected behavior:

```text
MGMT-SRV1:
- SSH allowed

All other VLANs:
- SSH denied
```

Validation results:

```text
MGMT-SRV1 10.10.40.10 successfully accessed network devices through SSH.
ROGUE-PC 10.10.20.126 was denied by the VTY-MGMT-ONLY ACL.
ACL permit and deny hit counts increased as expected.
```

Example validation from ACC1:

```text
permit 10.10.40.10 matched MGMT-SRV1 SSH access.
deny any log matched ROGUE-PC SSH access from 10.10.20.126.
```

---

## 14. SSH Client Compatibility Note

Ubuntu OpenSSH rejected older IOS SSH algorithms by default.

The following SSH client options were used from MGMT-SRV1 to validate access to IOSv and IOSvL2 devices:

```bash
ssh -oKexAlgorithms=+diffie-hellman-group14-sha1 -oHostKeyAlgorithms=+ssh-rsa -oPubkeyAcceptedAlgorithms=+ssh-rsa admin@<DEVICE-IP>
```

This confirms that the management ACL allowed access from MGMT-SRV1, while the remaining issue was SSH algorithm compatibility between Ubuntu OpenSSH and IOSv/IOSvL2.

---

## 15. Important Troubleshooting Outcomes

Several important troubleshooting outcomes were captured during this phase.

### 15.1 Access Switch SSH Login Failure

ACCESS switches had VTY lines configured with `login local`, but local usernames were missing.

Fix:

```text
username admin privilege 15 secret <ADMIN_PASSWORD>
```

Reason:

```text
login local requires a local username and password on the device.
```

### 15.2 EDGE1 SSH Disabled

EDGE1 initially showed:

```text
SSH Disabled - version 2.0
Please create RSA keys to enable SSH
```

Fix:

```text
ip domain-name campus.lab
crypto key generate rsa modulus 2048
```

### 15.3 Static Servers and DAI

DAI initially dropped valid ARP traffic from statically addressed servers on SRV-ACC1 because no dynamic DHCP Snooping binding existed.

Fix:

```text
Static source bindings were configured for MGMT-SRV1 and SERVER1.
```

### 15.4 IP Source Guard Platform Behavior

IP Source Guard showed correct bindings but caused traffic loss in the CML IOSvL2 data plane.

Final decision:

```text
Validated but not retained.
```

---

## 16. Final Phase Status

Phase 04 Security was completed.

Completed items:

```text
Access Port Hardening: Completed
Port Security: Completed
DHCP Snooping: Completed
Dynamic ARP Inspection: Completed
Static Source Binding: Completed
IP Source Guard: Validated, not retained due to CML IOSvL2 behavior
BPDU Guard: Completed
Management ACL / VTY Access Control: Completed
```

---

## 17. Repository Files

Recommended files for this phase:

```text
phase-04-security-ver.2/
├── README.md
├── configs.md
├── design-decisions.md
├── troubleshooting.md
├── verification.md
├── phase-04-security-ver.2.drawio
├── phase-04-security-ver.2.png
└── lab.yaml
```

---

## 18. Next Phase

The next phase is:

```text
Phase 05 - QoS
```

Phase 05 will build on this secured campus foundation and introduce QoS classification, marking, queuing, and policy validation for business-critical traffic.
