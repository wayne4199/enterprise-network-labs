# Phase 04 Security - Design Decisions

## 1. Operational Baseline Decision

Phase 04 Security uses the final state of **Phase 03 Case 2 - Multi-Area OSPF** as the operational baseline.

Phase 03 had two learning cases:

```text
Case 1 - Single Area OSPF
Case 2 - Multi-Area OSPF
```

Single Area OSPF was used as a learning and comparison case.
Multi-Area OSPF was selected as the operational baseline for Phase 04 and later phases.

Final operational baseline:

```text
Phase 03 Case 2 - Multi-Area OSPF Final State
```

This means Phase 04 does not split the security design into Single Area and Multi-Area cases. All security features are applied on top of the Multi-Area OSPF design.

---

## 2. Why Multi-Area OSPF Was Selected for Phase 04

Multi-Area OSPF was selected because it better represents the long-term operational campus design.

The final Phase 03 design introduced the following structure:

```text
EDGE1 ↔ CORE1 routed link = OSPF Area 0
EDGE1 ↔ CORE2 routed link = OSPF Area 0

CORE1 VLAN SVIs = OSPF Area 10
CORE2 VLAN SVIs = OSPF Area 10

CORE1 = ABR
CORE2 = ABR
EDGE1 = Area 0 router
```

This design provides a more realistic enterprise routing structure than a flat Single Area OSPF design.

The main reasons for using Multi-Area OSPF as the Phase 04 baseline are:

```text
1. It supports a more scalable routing design.
2. It allows route summarization from Area 10 toward Area 0.
3. It separates the campus access VLAN area from the backbone/WAN-facing area.
4. It provides a better foundation for later phases such as QoS and SD-WAN underlay enhancement.
5. It reflects the final Phase 03 operational design rather than the temporary learning design.
```

---

## 3. Role of Single Area OSPF

Single Area OSPF remains documented as a Phase 03 learning case.

It was useful for:

```text
- Understanding basic OSPF adjacency formation
- Validating passive-interface behavior
- Testing default route advertisement
- Testing OSPF cost manipulation
- Comparing simple routing behavior against a Multi-Area design
```

However, Single Area OSPF was not selected as the Phase 04 baseline because it does not include the route summarization and ABR behavior that were intentionally added and validated in Phase 03 Case 2.

Therefore:

```text
Single Area OSPF = learning comparison case
Multi-Area OSPF = operational baseline
```

---

## 4. Route Summarization Baseline

In the final Multi-Area OSPF state, CORE1 and CORE2 summarize the campus VLAN prefixes toward Area 0.

Summary route:

```cisco
area 10 range 10.10.0.0 255.255.0.0
```

Expected EDGE1 route:

```text
O IA 10.10.0.0/16
```

This keeps the upstream routing table cleaner and provides a more scalable routing model.

Preferred internal path:

```text
EDGE1 → CORE1
Next-hop: 172.16.0.2
```

Backup internal path:

```text
EDGE1 → CORE2
Next-hop: 172.16.0.6
```

OSPF cost manipulation keeps CORE1 as the preferred path under normal conditions.

---

## 5. Known Multi-Area OSPF Design Limitation

The Multi-Area OSPF design has an important limitation that must be documented and understood.

Because CORE1 and CORE2 summarize the campus VLANs into a single `/16` route, upstream devices such as EDGE1 may not see individual VLAN-level failures.

Example:

```text
CORE1 VLAN40 SVI fails.
CORE2 becomes HSRP Active for VLAN40.
The VLAN40 HSRP VIP remains reachable from the campus side.
```

However, EDGE1 may still see only:

```text
O IA 10.10.0.0/16 via CORE1
```

Possible issue:

```text
Return traffic from EDGE1 may continue to use CORE1 because the summary route still exists.
If CORE1 cannot forward traffic for the failed VLAN, return traffic can be blackholed.
```

Design lesson:

```text
Route summarization improves scalability but can hide more-specific VLAN failures from upstream routers.
HSRP gateway failover and OSPF summarization must be considered together.
```

This is not a Phase 04 Security failure. It is a routing design trade-off discovered in Phase 03 and carried forward as an operational consideration.

---

## 6. Security Phase Scope Decision

Phase 04 does not redesign routing.

The following Phase 03 routing components are preserved:

```text
Multi-Area OSPF
OSPF Area 0 / Area 10 structure
OSPF route summarization
HSRP Active/Standby roles
WAN failover using IP SLA and object tracking
EDGE1 primary and backup default routes
Existing VLAN and trunk structure
```

Phase 04 focuses only on security hardening.

Security controls are applied to the existing campus design without changing the routing architecture.

---

## 7. Access-Layer Security Focus

Most Phase 04 features were applied at the access layer because the main security risks are endpoint-facing.

Access-layer security controls include:

```text
Access Port Hardening
Port Security
DHCP Snooping
Dynamic ARP Inspection
BPDU Guard
Management VLAN SVI for switch management
VTY SSH Access Control
```

The access layer is the correct enforcement point because it is where users, servers, printers, test devices, and rogue devices connect to the network.

---

## 8. Access Port Hardening Decision

Access-facing ports were manually forced into access mode.

Design:

```text
switchport mode access
switchport nonegotiate
spanning-tree portfast edge
```

Reason:

```text
Access ports should not negotiate trunking.
Endpoint ports should transition to forwarding quickly.
PortFast is appropriate only on endpoint-facing access ports.
```

This reduces the risk of unintended trunk formation or DTP-based misbehavior.

---

## 9. Port Security Decision

Port Security was enabled on endpoint-facing access ports.

Final design:

```text
Maximum MAC addresses: 1
Sticky MAC learning: enabled
Violation mode: restrict
```

The `restrict` mode was selected instead of `shutdown`.

Reason:

```text
Restrict mode drops violating traffic and increments violation counters.
The port remains up, making it easier to observe and troubleshoot violations during lab validation.
```

This mode is useful for learning and controlled validation.

---

## 10. DHCP Snooping Decision

DHCP Snooping was enabled on access-layer switches.

Trusted ports:

```text
Uplinks and trunk ports
DHCP server-facing port on SRV-ACC1
```

Untrusted ports:

```text
User-facing access ports
ROGUE-PC test port
ROGUE-SW test port
```

Rate limit:

```text
5 packets per second on untrusted access ports
```

Option 82:

```text
Disabled
```

Reason:

```text
The goal was to allow only the authorized DHCP server on MGMT-SRV1 to provide DHCP service while preventing rogue or abusive DHCP behavior from endpoint-facing ports.
```

CORE1 and CORE2 were not kept as DHCP Snooping enforcement points in the final design. They remain DHCP relay devices using `ip helper-address`, while DHCP Snooping enforcement is handled at the access layer.

---

## 11. Dynamic ARP Inspection Decision

Dynamic ARP Inspection was enabled after DHCP Snooping was validated.

Reason:

```text
DAI relies on DHCP Snooping bindings to validate ARP traffic.
DHCP Snooping must be working before DAI is enabled.
```

DAI was applied on user access switches and validated on ACC2 using ROGUE-PC.

Validation result:

```text
Normal ARP from a valid DHCP-bound host was forwarded.
Spoofed ARP using a different IP address was dropped.
```

DAI validation mode:

```cisco
ip arp inspection validate src-mac dst-mac ip
```

---

## 12. Static Source Binding Decision for SRV-ACC1

SRV-ACC1 connects statically addressed servers:

```text
MGMT-SRV1 - 10.10.40.10
SERVER1   - 10.10.70.10
```

These hosts do not create dynamic DHCP Snooping bindings because they use static IP addresses.

If DAI is enabled on VLAN 40 or VLAN 70 without static bindings, legitimate ARP traffic from these servers can be dropped.

Final decision:

```text
Use static source bindings before enabling DAI on SRV-ACC1 VLAN 40 and VLAN 70.
```

Static source bindings:

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

Reason for choosing static source bindings instead of ARP ACLs:

```text
Static source bindings can be reused by both Dynamic ARP Inspection and IP Source Guard.
This keeps the security policy consistent across DHCP Snooping, DAI, and IP Source Guard.
```

---

## 13. IP Source Guard Decision

IP Source Guard was tested on ACC2 Gi1/1.

Tested modes:

```text
ip verify source
ip verify source port-security
```

The switch showed correct binding output:

```text
Interface: Gi1/1
Filter type: ip-mac
Filter mode: active
IP address: 10.10.20.126
MAC address: 52:54:00:61:B0:C0
VLAN: 20
```

However, in the CML IOSvL2 environment, enabling IP Source Guard caused host traffic to fail even though the binding was correct.

Observed behavior:

```text
IP Source Guard enabled:
- ROGUE-PC traffic failed.

IP Source Guard removed:
- ROGUE-PC traffic immediately recovered.
```

Final decision:

```text
IP Source Guard was validated but not retained on active access ports in Phase 04.
```

This was documented as a CML IOSvL2 platform-specific behavior or data-plane limitation, not as a design error.

---

## 14. BPDU Guard Decision

BPDU Guard was enabled on access-facing PortFast edge ports.

Reason:

```text
Access ports should not receive BPDUs.
If a switch is connected to an access port, the access layer should block that port to prevent unintended STP topology changes or Layer 2 loops.
```

ROGUE-SW-TEST was connected to SRV-ACC1 Gi0/3 for validation.

Validation result:

```text
SRV-ACC1 Gi0/3 received BPDU.
BPDU Guard placed the port into err-disabled state.
Err-disable reason: bpduguard.
```

Final decision:

```text
BPDU Guard remains enabled on access-facing PortFast edge ports.
```

---

## 15. Management ACL Decision

Management-plane access was restricted to the Enterprise Management Server.

Allowed management source:

```text
MGMT-SRV1: 10.10.40.10
```

VTY ACL:

```text
VTY-MGMT-ONLY
permit host 10.10.40.10
deny any log
```

Reason:

```text
Only the dedicated management server should be allowed to initiate SSH sessions to network devices.
User VLANs, test hosts, and rogue hosts should not be able to access network device VTY lines.
```

Validation result:

```text
MGMT-SRV1 successfully accessed devices through SSH.
ROGUE-PC from VLAN 20 was denied by the VTY-MGMT-ONLY ACL.
ACL permit and deny hit counts increased as expected.
```

---

## 16. AAA and SSH Design Decision

CORE1, CORE2, and EDGE1 use AAA local authentication.

Design:

```text
aaa new-model
aaa authentication login default local
aaa authorization exec default local
```

ACCESS switches use local login on VTY lines:

```text
login local
```

Because `login local` requires a local user database, the following account was added to access switches:

```cisco
username admin privilege 15 secret <ADMIN_PASSWORD>
```

Real passwords are not stored in GitHub.

---

## 17. EDGE1 SSH RSA Key Decision

EDGE1 initially had SSH configured but SSH was disabled because RSA keys were missing.

Observed condition:

```text
SSH Disabled
Please create RSA keys to enable SSH
```

Fix:

```cisco
ip domain-name campus.lab
crypto key generate rsa modulus 2048
ip ssh version 2
```

Reason:

```text
IOS requires a domain name and RSA key pair before SSH can be enabled.
```

---

## 18. Management SVI Decision for Access Switches

Access switches require a management SVI because their physical switchports are Layer 2 ports.

Management VLAN:

```text
VLAN 40
```

Management IP plan:

```text
ACC1      10.10.40.11
ACC2      10.10.40.12
ACC3      10.10.40.13
SRV-ACC1  10.10.40.14
```

Default gateway:

```text
10.10.40.254
```

Reason:

```text
Access switches are Layer 2 switches.
They need an SVI and default gateway for out-of-band-style in-band management through VLAN 40.
```

---

## 19. Final Phase 04 Security Design Summary

Final security design:

```text
Access Port Hardening:
- Enabled on endpoint-facing ports.

Port Security:
- Enabled with sticky MAC and restrict mode.

DHCP Snooping:
- Enabled at the access layer.
- Uplinks and DHCP server-facing ports trusted.
- Access-facing ports untrusted.

Dynamic ARP Inspection:
- Enabled on user VLANs.
- Enabled on server VLANs using static source bindings.

Static Source Binding:
- Used for MGMT-SRV1 and SERVER1 on SRV-ACC1.

IP Source Guard:
- Validated but not retained due to CML IOSvL2 forwarding behavior.

BPDU Guard:
- Enabled on PortFast edge ports.
- Rogue switch test successfully triggered err-disable.

Management ACL:
- SSH access allowed only from MGMT-SRV1.
- Non-management VLAN SSH attempts denied.
```

---

## 20. Carry-Forward Notes

The following items should be remembered for later phases:

```text
1. Phase 04 uses Phase 03 Multi-Area OSPF as the operational baseline.
2. Single Area OSPF remains only a Phase 03 learning comparison case.
3. Route summarization can hide individual VLAN failures and should be considered in later designs.
4. IP Source Guard was validated but not retained due to CML IOSvL2 behavior.
5. Static source bindings on SRV-ACC1 provide a reusable foundation for future IP/MAC-based security controls.
6. NAT/PAT failover remains a Phase 06 SD-WAN Underlay Enhancement carry-over item from Phase 03.
```
