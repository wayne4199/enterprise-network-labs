# Phase 03 — Routing Resiliency Ver.2 Design Decisions

## 1. Purpose

This document records the key design decisions made during Phase 03 Routing Resiliency Ver.2.

Phase 03 is built on top of the completed Phase 01 Foundation and Phase 02 Enterprise Services baseline.

The purpose of this phase is to add routing resiliency without redesigning or breaking the existing campus services.

---

## 2. Phase 03 Design Direction

Phase 03 is intentionally divided into two OSPF cases.

```text
Case 1 — Single Area OSPF
Case 2 — Multi-Area OSPF
```

This document currently records the design decisions for **Case 1 — Single Area OSPF**.

The Multi-Area OSPF design decisions will be added after that case is implemented and validated.

---

## 3. Preserve Phase 01 and Phase 02 Baseline

### Decision

Phase 03 must preserve the completed Phase 01 and Phase 02 baseline.

### Preserved Items

```text
- VLAN design
- CORE1 / CORE2 collapsed-core structure
- Access layer topology
- Rapid-PVST
- STP root and HSRP active alignment
- HSRP default gateway redundancy
- VLAN999 native blackhole
- MGMT-SRV1 Enterprise Management Server
- DHCP relay to MGMT-SRV1
- campus.lab DNS records
- DNS / DHCP / NTP / Syslog / SNMP / Config Backup
- SSH-only management
- AAA local authentication
- MGMT-ACCESS ACL
- SNMP-MGMT ACL
- EDGE1 NAT/PAT baseline
```

### Reason

Phase 03 is not a redesign phase.

It adds routing resiliency on top of the existing enterprise campus foundation.

The goal is to improve routing behavior while keeping Phase 02 Enterprise Services stable.

---

## 4. No Physical Topology Change for Single Area OSPF

### Decision

No new nodes were added or removed for the Single Area OSPF case.

### Reason

The existing topology already contains the required elements for routing resiliency:

```text
- Dual ISP paths
- EDGE1 WAN edge router
- Dual routed uplinks from EDGE1 to CORE1 and CORE2
- CORE1 / CORE2 collapsed-core pair
- Existing HSRP gateway redundancy
```

Therefore, the Single Area OSPF case only requires configuration changes, not physical topology changes.

---

## 5. Use Single Area OSPF First

### Decision

The first OSPF implementation uses Single Area OSPF Area 0.

```text
OSPF Process ID: 1
OSPF Area      : 0
```

### OSPF Routers

```text
EDGE1
CORE1
CORE2
```

### Reason

Single Area OSPF is the simplest and most appropriate first step for introducing dynamic routing into the campus core.

It allows the lab to focus on:

```text
- OSPF neighbor formation
- Passive interface behavior
- OSPF route learning
- Default route advertisement
- Static-to-OSPF internal route migration
- OSPF cost manipulation
```

This provides a stable routing foundation before moving to Multi-Area OSPF.

---

## 6. Use Multi-Area OSPF Later

### Decision

Multi-Area OSPF will be implemented after the Single Area OSPF case is completed and documented.

### Planned Multi-Area Design

```text
EDGE1 ↔ CORE1/CORE2 routed links = Area 0
CORE1/CORE2 VLAN SVIs = Area 10
CORE1/CORE2 = ABR
```

### Reason

Both Single Area and Multi-Area OSPF are used in real environments.

```text
Small or simple enterprise networks:
- Single Area OSPF may be sufficient

Larger or more structured enterprise networks:
- Multi-Area OSPF is more scalable
- Route summarization becomes meaningful
```

The lab intentionally includes both cases to avoid learning only the easier design.

This also provides a stronger baseline for future phases:

```text
Phase 04 — Security
Phase 05 — QoS
Phase 06 — SD-WAN Underlay Enhancement
```

---

## 7. Do Not Include ISP Links in OSPF

### Decision

The ISP-facing interfaces on EDGE1 are not included in OSPF.

Excluded interfaces:

```text
EDGE1 G0/0 → ISP1
EDGE1 G0/1 → ISP2
```

### Reason

The ISP side is treated as an external WAN or Internet-facing segment.

OSPF is used only inside the enterprise campus routing domain.

WAN default routing is controlled separately through:

```text
- Static default route
- Floating static route
- IP SLA
- Object tracking
```

This keeps the enterprise IGP separate from the ISP-facing network.

---

## 8. Use OSPF Only Between EDGE1 and CORE1/CORE2

### Decision

OSPF neighbor relationships are formed only on the routed links between EDGE1 and CORE1/CORE2.

```text
EDGE1 G0/2 172.16.0.1/30 ↔ CORE1 G0/0 172.16.0.2/30
EDGE1 G0/3 172.16.0.5/30 ↔ CORE2 G0/0 172.16.0.6/30
```

### Reason

These are the only routed links that require OSPF adjacency in the Single Area case.

Access switches remain Layer 2 and are not part of OSPF.

This keeps the OSPF domain clean and aligned with the collapsed-core design.

---

## 9. Use Manual OSPF Router IDs

### Decision

Manual router IDs are used.

```text
EDGE1  1.1.1.1
CORE1  2.2.2.2
CORE2  3.3.3.3
```

### Reason

Manual router IDs make OSPF neighbor validation easier.

They also make troubleshooting outputs clearer and deterministic.

---

## 10. Use Passive Interface Default

### Decision

OSPF is configured with `passive-interface default`.

Only required routed links are manually set as non-passive.

### Design

```text
EDGE1:
- G0/2 non-passive
- G0/3 non-passive

CORE1:
- G0/0 non-passive
- VLAN SVIs passive

CORE2:
- G0/0 non-passive
- VLAN SVIs passive
```

### Reason

This prevents unnecessary OSPF hello packets and neighbor formation on VLAN SVI interfaces.

It also reduces risk and keeps neighbor relationships limited to intended routed links.

### Important Lab Observation

During the lab, an unexpected passive-interface state was observed and corrected.

This showed that routing protocol pre-checks are important before applying new OSPF configuration.

Future pre-checks should include:

```cisco
show ip protocols
show running-config | section router ospf
show running-config | section router eigrp
show running-config | section router bgp
```

---

## 11. Advertise Internal VLANs into OSPF from CORE1 and CORE2

### Decision

CORE1 and CORE2 advertise the internal campus VLAN networks into OSPF.

The network statement uses the broader 10.10.0.0/16 range.

### Reason

All internal VLAN subnets are under the 10.10.0.0/16 address plan.

This allows EDGE1 to learn the internal campus routes dynamically from CORE1 and CORE2.

### Result

EDGE1 learned the following internal VLAN routes through OSPF:

```text
10.10.10.0/24
10.10.11.0/24
10.10.20.0/24
10.10.21.0/24
10.10.30.0/24
10.10.31.0/24
10.10.40.0/24
10.10.50.0/24
10.10.60.0/24
10.10.70.0/24
```

---

## 12. Remove EDGE1 Internal Static Summary Routes After OSPF Validation

### Decision

After OSPF route learning was validated, the internal static summary routes on EDGE1 were removed.

Removed routes:

```text
10.10.0.0/16 → CORE1
10.10.0.0/16 → CORE2 with higher AD
```

### Reason

Once OSPF was validated, EDGE1 no longer needed static routes for internal VLAN networks.

The internal campus routing path should be learned dynamically through OSPF.

This also helps test OSPF resiliency between EDGE1 and CORE1/CORE2.

### Final Result

```text
EDGE1 uses OSPF-learned routes for internal VLAN networks.
```

---

## 13. Keep WAN Default Routes as Static/Floating Static

### Decision

WAN default routing remains static on EDGE1.

Primary route:

```text
0.0.0.0/0 → ISP1 100.64.1.1
```

Backup floating route:

```text
0.0.0.0/0 → ISP2 100.64.2.1 with higher AD
```

### Reason

The ISP-facing side is not part of OSPF.

Therefore, default route control is handled using static routing, floating static routing, IP SLA, and object tracking.

This separates internal dynamic routing from external WAN path selection.

---

## 14. Use IP SLA to Track ISP1 Path

### Decision

EDGE1 uses IP SLA to monitor reachability to INTERNET-SRV through ISP1.

Target:

```text
203.0.113.10
```

Source interface:

```text
EDGE1 G0/0 toward ISP1
```

### Reason

A simple interface-up state is not enough to confirm WAN reachability.

IP SLA provides active path validation.

If ISP1 becomes unreachable, object tracking can remove the primary default route and allow the backup route through ISP2 to take over.

---

## 15. Tie Primary Default Route to Object Tracking

### Decision

The primary default route to ISP1 is tied to Track 1.

```text
0.0.0.0/0 → 100.64.1.1 track 1
```

The backup default route remains a floating static route.

```text
0.0.0.0/0 → 100.64.2.1 with higher AD
```

### Reason

This allows automatic failover from ISP1 to ISP2.

If Track 1 is Up:

```text
EDGE1 uses ISP1.
```

If Track 1 is Down:

```text
The ISP1 default route is removed.
EDGE1 uses ISP2.
```

---

## 16. Add Return Path for ISP2 Validation

### Decision

A return route was added on ISP1 for the ISP2 WAN subnet.

```text
100.64.2.0/30 → 203.0.113.2
```

### Reason

During ISP1 failure testing, EDGE1 successfully failed over to ISP2, but ping to INTERNET-SRV initially failed.

The issue was not EDGE1 routing failover.

The issue was the return path from the Internet segment.

INTERNET-SRV used ISP1 as its default gateway:

```text
203.0.113.1
```

Therefore, the return path needed to know how to reach the ISP2 WAN subnet.

This route allowed proper return traffic during ISP2 path validation.

---

## 17. Use OSPF Cost Manipulation to Prefer CORE1

### Decision

EDGE1 initially learned internal VLAN routes through both CORE1 and CORE2 with equal cost.

To make CORE1 the preferred path, the OSPF cost on EDGE1 G0/3 toward CORE2 was increased.

### Design

```text
EDGE1 G0/2 toward CORE1: cost 1
EDGE1 G0/3 toward CORE2: cost 50
```

### Reason

The lab intentionally avoids relying on ECMP at this phase.

The goal is to create a clear primary/backup routing behavior:

```text
CORE1 = preferred internal path
CORE2 = backup internal path
```

This also makes failover testing easier to observe.

---

## 18. Do Not Implement Route Summarization in Single Area OSPF

### Decision

Route Summarization is not implemented in the Single Area OSPF case.

### Reason

The current OSPF design is Single Area 0.

```text
EDGE1 = Area 0
CORE1 = Area 0
CORE2 = Area 0
VLAN SVIs = Area 0
```

There is no ABR in this design.

OSPF area summarization is normally performed on ABRs using:

```text
area <area-id> range
```

Because CORE1 and CORE2 are not ABRs in Single Area OSPF, there is no proper inter-area boundary for summarization.

### Final Decision

```text
Single Area OSPF:
- Route summarization is not applied.
- This is intentional.

Multi-Area OSPF:
- CORE1/CORE2 will become ABRs.
- VLAN SVIs will move into a non-backbone area.
- Route summarization will be implemented with area range.
```

---

## 19. Validate HSRP Failover Separately from OSPF Uplink Failure

### Decision

HSRP failover is validated separately from OSPF routed uplink failure.

### Reason

A routed uplink failure does not automatically mean HSRP will fail over.

During the lab, CORE1 G0/0 was shut down.

Observed behavior:

```text
CORE1 lost OSPF adjacency with EDGE1.
EDGE1 moved internal routes toward CORE2.
CORE1 still remained HSRP Active for VLAN40 because the VLAN40 SVI stayed up.
```

This is an important design observation.

### Implication

If a core switch remains HSRP Active while its routed uplink is down, traffic can be blackholed.

### Future Enhancement

HSRP interface tracking should be considered if uplink failure must trigger HSRP role change.

Example future design idea:

```text
Track CORE uplink state
Reduce HSRP priority when uplink fails
Allow peer core switch to become HSRP Active
```

This was not implemented in this Single Area OSPF case, but the behavior was observed and documented.

---

## 20. Use VLAN40 for HSRP Failover Validation

### Decision

VLAN40 was used to validate HSRP failover.

### Reason

VLAN40 is the management VLAN and directly affects MGMT-SRV1.

Original VLAN40 role:

```text
CORE1 = Active
CORE2 = Standby
VIP   = 10.10.40.254
```

A VLAN40 SVI failure on CORE1 allowed validation that CORE2 could become Active and preserve the VLAN40 gateway VIP.

### Result

```text
CORE2 became Active for VLAN40.
MGMT-SRV1 retained reachability to the VLAN40 VIP.
MGMT-SRV1 retained reachability to EDGE1 and INTERNET-SRV.
```

---

## 21. Do Not Fully Implement NAT/PAT Failover in Phase 03

### Decision

NAT/PAT failover is not fully implemented in Phase 03.

Only the limitation is identified and documented.

### Current NAT/PAT Behavior

EDGE1 currently uses PAT overload on ISP1 G0/0 only:

```text
ip nat inside source list ENTERPRISE-NAT interface GigabitEthernet0/0 overload
```

Both WAN interfaces are configured as NAT outside:

```text
G0/0 = NAT outside
G0/1 = NAT outside
```

However, only G0/0 is used for PAT overload.

### Limitation

When routing fails over to ISP2:

```text
Default route changes to ISP2.
PAT overload remains tied to G0/0.
```

Therefore, full NAT/PAT failover is incomplete.

### Reason for Deferral

NAT/PAT failover becomes more meaningful when combined with:

```text
- PBR
- IP SLA
- Route-map based NAT
- ISP-specific NAT policy
- DIA
- Local Internet Breakout
- Business intent path selection
```

These topics belong more naturally to Phase 06 SD-WAN Underlay Enhancement.

### Carry-over

NAT/PAT failover must be revisited in Phase 06.

---

## 22. Failure and Recovery Test Scope

### Decision

The Single Area OSPF case validates three major failure and recovery scenarios.

```text
Test 1 — ISP1 failure and recovery
Test 2 — CORE1 uplink failure and recovery
Test 3 — HSRP VLAN40 failure and recovery
```

### Reason

These tests cover the most important routing resiliency behaviors in the current topology.

They validate:

```text
- WAN failover
- IP SLA and object tracking
- Floating static route behavior
- OSPF path failover
- HSRP gateway failover
- Difference between routing failover and gateway failover
```

---

## 23. Keep Access Layer Out of OSPF

### Decision

Access layer switches remain outside the OSPF routing domain.

### Reason

In this collapsed-core campus design, routing is handled by CORE1 and CORE2.

The access layer remains Layer 2.

This keeps the design simple and aligned with the existing campus architecture.

---

## 24. Keep Phase 03 Focused on Routing Resiliency

### Decision

Phase 03 does not redesign Security, QoS, or SD-WAN policy.

### Reason

Those topics are handled in later phases.

```text
Phase 04 — Security
Phase 05 — QoS
Phase 06 — SD-WAN Underlay Enhancement
```

Phase 03 only prepares the routing foundation required for those later phases.

---

## 25. Single Area OSPF Case Completion Decision

### Decision

The Single Area OSPF case is considered complete after validating the following:

```text
- OSPF Single Area 0
- OSPF neighbor formation
- Passive interface behavior
- OSPF route learning
- Internal static route removal
- OSPF default route advertisement
- Floating static default route
- IP SLA
- Object tracking
- WAN resiliency
- OSPF cost manipulation
- HSRP failover validation
- Failure and recovery behavior
```

### Not Implemented in Single Area OSPF

The following items are intentionally not implemented in this case:

```text
Route Summarization:
- Not appropriate in Single Area OSPF
- Deferred to Multi-Area OSPF

NAT/PAT Failover:
- Limitation identified
- Full implementation deferred to Phase 06
```

---

## 26. Final Design Summary

The Single Area OSPF design provides a stable routing resiliency baseline.

Completed design outcomes:

```text
- EDGE1, CORE1, and CORE2 run OSPF Area 0.
- OSPF neighbors form only on intended routed links.
- VLAN SVI interfaces are passive in OSPF.
- EDGE1 learns internal VLAN routes through OSPF.
- Internal static routes on EDGE1 are removed.
- EDGE1 advertises default route information to CORE1 and CORE2.
- WAN default routing uses floating static route with IP SLA tracking.
- ISP1 is primary and ISP2 is backup.
- OSPF cost makes CORE1 the preferred internal path.
- HSRP failover is validated separately from routed uplink failure.
- Route summarization is deferred to Multi-Area OSPF.
- NAT/PAT failover is deferred to Phase 06.
```

Next design step:

```text
Reconfigure the OSPF domain into Multi-Area OSPF and implement route summarization.
```
