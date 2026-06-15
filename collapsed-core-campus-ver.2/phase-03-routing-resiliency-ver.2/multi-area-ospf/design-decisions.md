# Phase 03 — Routing Resiliency Ver.2 Design Decisions

## 1. Purpose

This document records the key design decisions made during **Phase 03 — Routing Resiliency Ver.2**.

Phase 03 builds on the completed Phase 01 Foundation and Phase 02 Enterprise Services baseline.

The purpose of this phase is to introduce routing resiliency, OSPF-based dynamic routing, WAN failover, route summarization, and failure recovery validation without redesigning the existing collapsed-core campus topology.

---

## 2. Phase 03 Design Direction

Phase 03 is divided into two OSPF cases.

```text
Case 1 — Single Area OSPF
Case 2 — Multi-Area OSPF
```

This design approach was chosen because both models are used in real enterprise networks.

Single Area OSPF provides a simple routing baseline.

Multi-Area OSPF provides a scalable hierarchical design and enables proper route summarization.

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
- STP root and HSRP active role alignment
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

## 4. No Physical Topology Change

### Decision

No new nodes were added and no existing nodes were removed for Phase 03.

The same topology was used for both Single Area OSPF and Multi-Area OSPF.

### Reason

The existing topology already contains the required elements for routing resiliency:

```text
- Dual ISP paths
- EDGE1 WAN edge router
- Dual routed uplinks from EDGE1 to CORE1 and CORE2
- CORE1 / CORE2 collapsed-core pair
- Existing HSRP gateway redundancy
- Existing enterprise services server
```

Therefore, Phase 03 only requires routing configuration changes, not physical topology changes.

---

# Case 1 — Single Area OSPF Design Decisions

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

## 6. Do Not Include ISP Links in OSPF

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

## 7. Use OSPF Only Between EDGE1 and CORE1/CORE2

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

## 8. Use Manual OSPF Router IDs

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

## 9. Use Passive Interface Default

### Decision

OSPF is configured with `passive-interface default`.

Only required routed links are manually set as non-passive.

### Single Area Design

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

---

## 10. Advertise Internal VLANs into OSPF from CORE1 and CORE2

### Decision

CORE1 and CORE2 advertise the internal campus VLAN networks into OSPF.

The Single Area case uses the broader `10.10.0.0/16` network statement.

### Reason

All internal VLAN subnets are under the `10.10.0.0/16` address plan.

This allows EDGE1 to learn the internal campus routes dynamically from CORE1 and CORE2.

### Result

EDGE1 learned the following internal VLAN routes through OSPF:

```text
O 10.10.10.0/24
O 10.10.11.0/24
O 10.10.20.0/24
O 10.10.21.0/24
O 10.10.30.0/24
O 10.10.31.0/24
O 10.10.40.0/24
O 10.10.50.0/24
O 10.10.60.0/24
O 10.10.70.0/24
```

---

## 11. Remove EDGE1 Internal Static Summary Routes After OSPF Validation

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

This also allows OSPF resiliency between EDGE1 and CORE1/CORE2 to be properly tested.

### Final Result

```text
EDGE1 uses OSPF-learned routes for internal VLAN networks.
```

---

## 12. Keep WAN Default Routes as Static/Floating Static

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

## 13. Use IP SLA to Track ISP1 Path

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

A host route is used for the IP SLA target:

```text
203.0.113.10/32 → 100.64.1.1
```

### Reason

A simple interface-up state is not enough to confirm WAN reachability.

IP SLA provides active path validation.

The host route ensures that the IP SLA probe continues to test the ISP1 path directly instead of following the backup default route after failover.

---

## 14. Tie Primary Default Route to Object Tracking

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

## 15. Use OSPF Cost Manipulation to Prefer CORE1

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

## 16. Do Not Implement Route Summarization in Single Area OSPF

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

# Case 2 — Multi-Area OSPF Design Decisions

## 17. Reconfigure from Single Area to Multi-Area OSPF

### Decision

After Single Area OSPF was validated, the OSPF domain was reconfigured into Multi-Area OSPF.

### Design

```text
EDGE1 ↔ CORE1/CORE2 routed links = Area 0
CORE1/CORE2 VLAN SVIs           = Area 10
CORE1/CORE2                     = ABR
```

### Reason

Single Area OSPF is useful for basic routing resiliency validation, but it does not provide an ABR summarization point.

Multi-Area OSPF allows the lab to validate:

```text
- ABR behavior
- Inter-area route learning
- Type-3 Summary LSA behavior
- Route summarization using area range
- Impact of summarization on upstream routing
```

---

## 18. Keep the Same Physical Topology for Multi-Area OSPF

### Decision

The physical topology is unchanged for Multi-Area OSPF.

Only the logical OSPF area design changes.

### Reason

The existing routed links and VLAN design already support a hierarchical OSPF model.

CORE1 and CORE2 can act as ABRs because they connect both:

```text
Area 0   → EDGE-facing routed links
Area 10  → Internal campus VLAN SVIs
```

No additional router is required.

---

## 19. Place EDGE-to-CORE Routed Links in Area 0

### Decision

The routed links between EDGE1 and CORE1/CORE2 remain in OSPF Area 0.

```text
EDGE1 G0/2 ↔ CORE1 G0/0 = Area 0
EDGE1 G0/3 ↔ CORE2 G0/0 = Area 0
```

### Reason

Area 0 is the OSPF backbone area.

Keeping the EDGE-to-CORE routed links in Area 0 provides the backbone path through which inter-area routes are exchanged.

---

## 20. Place VLAN SVIs in Area 10

### Decision

The internal VLAN SVI networks on CORE1 and CORE2 are placed into Area 10.

```text
VLAN10  EXEC       = Area 10
VLAN11  BIZ-ADMIN  = Area 10
VLAN20  SALES      = Area 10
VLAN21  MKT        = Area 10
VLAN30  IT-OPS     = Area 10
VLAN31  IT-SEC     = Area 10
VLAN40  MGMT       = Area 10
VLAN50  GUEST      = Area 10
VLAN60  PRINTER    = Area 10
VLAN70  SERVER     = Area 10
```

### Reason

This creates a clear hierarchical design:

```text
Area 0  = routed backbone links
Area 10 = internal campus VLAN networks
```

This also allows CORE1 and CORE2 to summarize Area 10 routes into Area 0.

---

## 21. CORE1 and CORE2 Act as ABRs

### Decision

CORE1 and CORE2 operate as Area Border Routers.

### Reason

Each core switch has interfaces in two OSPF areas:

```text
Area 0:
- Routed uplink to EDGE1

Area 10:
- Internal VLAN SVIs
```

Validated ABR state:

```text
It is an area border router
Number of areas in this router is 2
Area BACKBONE(0)
Area 10
```

### Result

CORE1 and CORE2 can generate Type-3 Summary LSAs between Area 10 and Area 0.

---

## 22. Keep Passive Interface Design in Multi-Area OSPF

### Decision

The same passive-interface design is kept in Multi-Area OSPF.

```text
EDGE1:
- G0/2 non-passive
- G0/3 non-passive
- ISP links not in OSPF

CORE1:
- G0/0 non-passive
- VLAN SVIs passive

CORE2:
- G0/0 non-passive
- VLAN SVIs passive
```

### Reason

VLAN SVIs should participate in OSPF route advertisement but should not form OSPF neighbors with end-user segments.

This keeps the OSPF domain clean and prevents unnecessary hello packets on access VLANs.

---

## 23. Validate Inter-Area Routes Before Summarization

### Decision

Before applying route summarization, individual VLAN routes are first validated as inter-area routes on EDGE1.

### Expected Route Type

```text
O IA
```

### Reason

This confirms that:

```text
- VLAN SVIs are correctly placed in Area 10.
- CORE1/CORE2 are functioning as ABRs.
- EDGE1 is receiving Area 10 routes through Area 0.
```

### Expected Pre-Summarization Routes

```text
O IA 10.10.10.0/24
O IA 10.10.11.0/24
O IA 10.10.20.0/24
O IA 10.10.21.0/24
O IA 10.10.30.0/24
O IA 10.10.31.0/24
O IA 10.10.40.0/24
O IA 10.10.50.0/24
O IA 10.10.60.0/24
O IA 10.10.70.0/24
```

---

## 24. Implement Route Summarization with area range

### Decision

Route summarization is implemented on CORE1 and CORE2 using `area 10 range`.

### Summary Route

```text
10.10.0.0/16
```

### Configuration Concept

```text
router ospf 1
 area 10 range 10.10.0.0 255.255.0.0
```

### Reason

All internal VLAN subnets are part of the 10.10.0.0/16 campus addressing block.

Summarizing the Area 10 VLAN routes into a single 10.10.0.0/16 route:

```text
- Reduces the routing table size on EDGE1
- Simplifies upstream route visibility
- Demonstrates practical ABR summarization behavior
- Prepares a more scalable routing design for later phases
```

---

## 25. Validate Summary Route on EDGE1

### Decision

After applying summarization, EDGE1 should receive only the summarized inter-area route.

Expected result:

```text
O IA 10.10.0.0/16
```

The individual /24 VLAN routes should no longer appear on EDGE1.

### Reason

This validates that CORE1 and CORE2 are correctly summarizing Area 10 routes into Area 0.

### OSPF Database Result

EDGE1 should see Type-3 Summary LSAs for:

```text
Link State ID: 10.10.0.0
Network Mask: /16
Advertising Router: 2.2.2.2
Advertising Router: 3.3.3.3
```

---

## 26. Keep CORE1 as Preferred Path After Summarization

### Decision

After summarization, EDGE1 still uses CORE1 as the preferred internal path.

### Design

```text
EDGE1 G0/2 toward CORE1 = OSPF cost 1
EDGE1 G0/3 toward CORE2 = OSPF cost 50
```

### Expected Result

```text
O IA 10.10.0.0/16 via 172.16.0.2, GigabitEthernet0/2
```

### Reason

The design intentionally keeps deterministic path preference:

```text
CORE1 = primary internal path
CORE2 = backup internal path
```

This makes failure and recovery behavior easier to validate.

---

## 27. Keep Default Route Advertisement from EDGE1

### Decision

EDGE1 continues to advertise default route information into OSPF using:

```text
default-information originate
```

### Reason

EDGE1 is the WAN edge router.

It has the external default path through ISP1 and ISP2.

CORE1 and CORE2 should continue to receive default route information from EDGE1 through OSPF.

### Important Note

CORE1 and CORE2 still use static default routes in the routing table if those static routes exist.

Reason:

```text
Static default route AD = 1
OSPF external route AD = 110
```

Therefore, the OSPF default LSA can be present in the OSPF database while the static default route remains preferred in the RIB.

---

## 28. Revalidate WAN Resiliency After Multi-Area OSPF

### Decision

WAN failover is revalidated after changing the OSPF design to Multi-Area.

### Reason

Changing the internal routing design should not break WAN resiliency.

The following must remain operational:

```text
- IP SLA
- Track 1
- Primary default route through ISP1
- Backup floating default route through ISP2
- ISP1 failure detection
- ISP2 failover
- ISP1 recovery
```

### Validated Behavior

```text
Normal:
- Track 1 Up
- IP SLA OK
- Default route via 100.64.1.1

Failure:
- EDGE1 G0/0 shutdown
- Track 1 Down
- Default route via 100.64.2.1

Recovery:
- EDGE1 G0/0 no shutdown
- Track 1 Up
- Default route returns to 100.64.1.1
```

---

## 29. Revalidate HSRP Failover After Multi-Area OSPF

### Decision

HSRP failover is revalidated after Multi-Area OSPF and route summarization are implemented.

### Reason

HSRP gateway redundancy must remain operational even after the routing design changes.

VLAN40 was used as the validation VLAN.

### Validated Behavior

```text
Initial:
- CORE1 Active for VLAN40
- CORE2 Standby for VLAN40
- VIP 10.10.40.254

Failure:
- CORE1 Vlan40 shutdown
- CORE2 becomes Active for VLAN40
- VIP remains 10.10.40.254

Recovery:
- CORE1 Vlan40 no shutdown
- CORE1 returns to Active due to preempt
- CORE2 returns to Standby
```

---

## 30. Document Summarization and HSRP Partial Failure Limitation

### Decision

The interaction between route summarization and HSRP partial SVI failure must be documented.

### Observed Behavior

During the VLAN40 HSRP test:

```text
CORE1 Vlan40 was shut down.
CORE2 became HSRP Active for VLAN40.
VIP 10.10.40.254 remained available.
```

However, EDGE1 only had the summarized route:

```text
O IA 10.10.0.0/16 via CORE1
```

Because EDGE1 preferred the CORE1 summary path, return traffic for VLAN40 could still be sent toward CORE1.

Since CORE1 Vlan40 was down, this could create return traffic blackholing.

### Design Lesson

Route summarization improves scalability, but it can hide more-specific VLAN failures from upstream routers.

HSRP failover and OSPF summarization must be considered together.

### Future Consideration

Future phases may consider additional mechanisms such as:

```text
- More specific route handling for critical VLANs
- Conditional route advertisement
- HSRP tracking
- Object tracking
- Better alignment between gateway state and routing advertisement
```

This was documented as a design limitation, not as a lab failure.

---

## 31. Validate CORE1 Uplink Failure and Summary Route Failover

### Decision

CORE1 uplink failure is tested after summarization.

### Reason

This confirms that the summarized route can fail over from CORE1 to CORE2 when the CORE1 Area 0 adjacency fails.

### Failure Test

```text
CORE1 G0/0 shutdown
```

### Expected Result on EDGE1

```text
- OSPF neighbor with CORE1 goes down.
- OSPF neighbor with CORE2 remains FULL.
- 10.10.0.0/16 summary route moves to CORE2 via 172.16.0.6.
```

### Observed Result

```text
O IA 10.10.0.0/16 via 172.16.0.6, GigabitEthernet0/3
Metric 51
```

### Recovery Test

```text
CORE1 G0/0 no shutdown
```

### Observed Recovery

```text
- OSPF neighbor with CORE1 returns to FULL.
- 10.10.0.0/16 summary route returns to CORE1 via 172.16.0.2.
- Metric returns to 2.
```

---

## 32. Do Not Fully Implement NAT/PAT Failover in Phase 03

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

## 33. Keep Access Layer Out of OSPF

### Decision

Access layer switches remain outside the OSPF routing domain.

### Reason

In this collapsed-core campus design, routing is handled by CORE1 and CORE2.

The access layer remains Layer 2.

This keeps the design simple and aligned with the existing campus architecture.

---

## 34. Keep Phase 03 Focused on Routing Resiliency

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

## 35. Final Design Summary

Phase 03 establishes both a simple and scalable routing resiliency baseline.

### Single Area OSPF Outcome

```text
- EDGE1, CORE1, and CORE2 run OSPF Area 0.
- OSPF neighbors form only on intended routed links.
- VLAN SVI interfaces are passive in OSPF.
- EDGE1 learns individual internal VLAN routes through OSPF.
- Internal static routes on EDGE1 are removed.
- EDGE1 advertises default route information to CORE1 and CORE2.
- WAN default routing uses floating static route with IP SLA tracking.
- OSPF cost makes CORE1 the preferred internal path.
- Route summarization is not applied in Single Area OSPF.
```

### Multi-Area OSPF Outcome

```text
- EDGE-to-CORE routed links are in Area 0.
- CORE1/CORE2 VLAN SVIs are in Area 10.
- CORE1 and CORE2 act as ABRs.
- EDGE1 learns individual O IA routes before summarization.
- CORE1 and CORE2 summarize Area 10 routes into 10.10.0.0/16.
- EDGE1 learns O IA 10.10.0.0/16 after summarization.
- OSPF cost keeps CORE1 as the preferred internal path.
- CORE2 remains the backup internal path.
- WAN resiliency remains operational.
- HSRP failover remains operational.
- Summarization and HSRP partial failure interaction is documented.
```

### Deferred Item

```text
NAT/PAT Failover:
- Limitation identified.
- Full implementation deferred to Phase 06 SD-WAN Underlay Enhancement.
```

---

## 36. Next Phase

The next phase is:

```text
Phase 04 — Security
```

Phase 04 will build on this routing resiliency foundation.
