# Phase 03 — Routing Resiliency Ver.2

## 1. Overview

This phase focuses on routing resiliency in the Collapsed Core Campus Ver.2 lab.

Phase 03 builds on top of the completed Phase 01 Foundation and Phase 02 Enterprise Services baseline.

The main goal of this phase is to introduce dynamic routing, WAN resiliency, route summarization, and failure recovery validation without redesigning the existing campus topology.

Phase 03 is divided into two OSPF cases:

```text
Case 1 — Single Area OSPF
Case 2 — Multi-Area OSPF
```

Both cases are intentionally included.

Single Area OSPF provides a simple routing baseline.

Multi-Area OSPF introduces ABR behavior, inter-area routing, and route summarization.

---

## 2. Lab Path

```text
enterprise-network-labs/
└── collapsed-core-campus-ver.2/
    ├── phase-01-foundation-ver.2/
    ├── phase-02-enterprise-services-ver.2/
    └── phase-03-routing-resiliency-ver.2/
```

---

## 3. Phase 03 Learning Scope

Phase 03 covers the following routing resiliency topics:

```text
1. Single Area OSPF
2. Multi-Area OSPF
3. OSPF Neighbor Formation
4. Passive Interface Design
5. OSPF Default Route Advertisement
6. Floating Static Route
7. IP SLA
8. Object Tracking
9. WAN Resiliency
10. OSPF Cost Manipulation
11. Inter-Area Route Learning
12. Route Summarization with area range
13. Summary Route Verification
14. HSRP Failover Validation
15. Failure and Recovery Test
16. NAT/PAT Failover Limitation and Phase 06 Carry-over
```

---

## 4. Baseline from Previous Phases

Phase 03 preserves the completed Phase 01 and Phase 02 baseline.

The following items must remain intact:

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

Phase 03 adds routing resiliency on top of this existing enterprise services foundation.

---

## 5. Current Topology

```text
INTERNET-SRV
    |
INTERNET-SW
   /        \
 ISP1      ISP2
   \        /
    \      /
     EDGE1
    /     \
 CORE1   CORE2
  /|\     /|\
ACC1 ACC2 ACC3
      |
   SRV-ACC1
   /      \
SERVER1  MGMT-SRV1
```

No physical topology change was required for Phase 03.

The same topology was used for both Single Area OSPF and Multi-Area OSPF.

---

## 6. WAN and Routed Link Addressing

```text
INTERNET-SRV: 203.0.113.10/24
Gateway     : 203.0.113.1

ISP1 G0/0   : 203.0.113.1/24
ISP2 G0/0   : 203.0.113.2/24

ISP1 G0/1   : 100.64.1.1/30
EDGE1 G0/0  : 100.64.1.2/30

ISP2 G0/1   : 100.64.2.1/30
EDGE1 G0/1  : 100.64.2.2/30

EDGE1 G0/2  : 172.16.0.1/30
CORE1 G0/0  : 172.16.0.2/30

EDGE1 G0/3  : 172.16.0.5/30
CORE2 G0/0  : 172.16.0.6/30
```

---

## 7. VLAN and HSRP Gateway Plan

```text
VLAN10  EXEC       VIP 10.10.10.254
VLAN11  BIZ-ADMIN  VIP 10.10.11.254

VLAN20  SALES      VIP 10.10.20.254
VLAN21  MKT        VIP 10.10.21.254

VLAN30  IT-OPS     VIP 10.10.30.254
VLAN31  IT-SEC     VIP 10.10.31.254

VLAN40  MGMT       VIP 10.10.40.254

VLAN50  GUEST      VIP 10.10.50.254
VLAN60  PRINTER    VIP 10.10.60.254
VLAN70  SERVER     VIP 10.10.70.254

VLAN999 Native Blackhole / Unused
```

---

## 8. HSRP Active Role Design

CORE1 is HSRP Active for:

```text
VLAN10
VLAN11
VLAN30
VLAN40
VLAN60
```

CORE2 is HSRP Active for:

```text
VLAN20
VLAN21
VLAN31
VLAN50
VLAN70
```

The HSRP active role is aligned with the STP root role from Phase 01.

---

# Case 1 — Single Area OSPF

## 9. Single Area OSPF Design

The first routing design uses Single Area OSPF.

```text
OSPF Process ID: 1
OSPF Area      : 0
```

OSPF is enabled only between:

```text
EDGE1 ↔ CORE1
EDGE1 ↔ CORE2
```

The ISP-facing interfaces are not included in OSPF.

```text
EDGE1 G0/0 toward ISP1: Not in OSPF
EDGE1 G0/1 toward ISP2: Not in OSPF
```

The purpose of this step is to migrate internal campus routing from static routes to dynamic OSPF while keeping WAN default routing controlled by floating static routes and IP SLA tracking.

---

## 10. Single Area OSPF Router IDs

```text
EDGE1  router-id 1.1.1.1
CORE1  router-id 2.2.2.2
CORE2  router-id 3.3.3.3
```

Router IDs were manually configured for clarity and deterministic neighbor identification.

---

## 11. Single Area OSPF Network Design

EDGE1 advertises only the routed links toward CORE1 and CORE2:

```text
172.16.0.0/30
172.16.0.4/30
```

CORE1 advertises:

```text
172.16.0.0/30
10.10.0.0/16
```

CORE2 advertises:

```text
172.16.0.4/30
10.10.0.0/16
```

Although CORE1 and CORE2 use the 10.10.0.0/16 network statement, the actual connected VLAN networks are advertised as individual connected /24 routes.

---

## 12. Single Area Passive Interface Design

Only routed links that require OSPF neighbor adjacency are non-passive.

```text
EDGE1:
- G0/2 non-passive
- G0/3 non-passive
- All other interfaces passive

CORE1:
- G0/0 non-passive
- VLAN SVIs passive

CORE2:
- G0/0 non-passive
- VLAN SVIs passive
```

This prevents unnecessary OSPF neighbor formation on VLAN SVI interfaces.

Expected result:

```text
EDGE1 forms OSPF neighbors only with CORE1 and CORE2.
CORE1 forms an OSPF neighbor only with EDGE1.
CORE2 forms an OSPF neighbor only with EDGE1.
```

---

## 13. Single Area OSPF Neighbor Validation

Expected OSPF neighbors on EDGE1:

```text
Neighbor 2.2.2.2 → CORE1
Neighbor 3.3.3.3 → CORE2
State FULL
```

Validated result:

```text
EDGE1 ↔ CORE1 OSPF neighbor: FULL
EDGE1 ↔ CORE2 OSPF neighbor: FULL
```

This confirms that Single Area OSPF Area 0 is operating correctly between EDGE1, CORE1, and CORE2.

---

## 14. Single Area OSPF Route Learning

After OSPF neighbor formation, EDGE1 learned the internal VLAN routes through OSPF.

Observed OSPF routes:

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

Initially, EDGE1 learned the VLAN routes through both CORE1 and CORE2 with equal OSPF cost.

This created an ECMP condition:

```text
EDGE1 → CORE1
EDGE1 → CORE2
```

This was expected before OSPF cost manipulation.

---

## 15. Internal Static Route Removal on EDGE1

Before OSPF was introduced, EDGE1 had static summary routes toward the internal campus network.

Existing internal static routes:

```text
10.10.0.0/16 → CORE1
10.10.0.0/16 → CORE2 with higher administrative distance
```

After OSPF successfully learned the internal VLAN routes, the internal static summary routes were removed from EDGE1.

Result:

```text
EDGE1 now uses OSPF-learned routes for internal VLAN networks.
```

This changed internal campus routing from static routing to dynamic OSPF routing.

---

## 16. OSPF Default Route Advertisement

EDGE1 keeps the WAN default routes toward ISP1 and ISP2.

EDGE1 advertises the default route into OSPF using:

```text
default-information originate
```

This allows CORE1 and CORE2 to receive default route information from EDGE1 through OSPF.

The default route is advertised as an OSPF external route.

Validated result:

```text
CORE1 received 0.0.0.0/0 Type-5 External LSA from EDGE1.
CORE2 received 0.0.0.0/0 Type-5 External LSA from EDGE1.
```

Important observation:

```text
CORE1 and CORE2 still had static default routes during validation.
Therefore, the OSPF default route was visible in the OSPF database,
but the static default route remained preferred in the routing table.
```

This is expected because:

```text
Static route AD = 1
OSPF external route AD = 110
```

---

## 17. Floating Static Route

EDGE1 uses floating static default routes for WAN resiliency.

Primary default route:

```text
0.0.0.0/0 → ISP1 100.64.1.1
```

Backup floating default route:

```text
0.0.0.0/0 → ISP2 100.64.2.1 with higher administrative distance
```

Purpose:

```text
Use ISP1 under normal conditions.
Use ISP2 only when the ISP1 route is removed or becomes unavailable.
```

---

## 18. IP SLA and Object Tracking

IP SLA was configured on EDGE1 to monitor reachability toward INTERNET-SRV.

Target:

```text
203.0.113.10
```

Source interface:

```text
EDGE1 G0/0 toward ISP1
```

Object Tracking was then tied to the IP SLA result.

Purpose:

```text
If ISP1 path is reachable:
- Track 1 remains Up
- Primary default route via ISP1 stays installed

If ISP1 path fails:
- Track 1 goes Down
- Primary default route is removed
- Backup default route via ISP2 is used
```

A host route was also used so that the IP SLA target is checked through the ISP1 path:

```text
203.0.113.10/32 → 100.64.1.1
```

This prevents the IP SLA probe from following the backup default route after failover.

---

## 19. WAN Resiliency Test

ISP1 failure was simulated by shutting down EDGE1 G0/0.

Expected behavior:

```text
EDGE1 G0/0 down
IP SLA fails
Track 1 down
Primary default route removed
Backup default route via ISP2 installed
```

Validated result:

```text
Track 1 Down
Default route changed to 100.64.2.1
```

ISP2 path validation:

```text
EDGE1 → 100.64.2.1: Success
EDGE1 → 203.0.113.2: Success
EDGE1 → 203.0.113.10 source G0/1: Success
```

After EDGE1 G0/0 was restored, Track 1 returned to Up and the default route moved back to ISP1.

Validated recovery:

```text
Track 1 Up
Default route returned to 100.64.1.1
Ping to INTERNET-SRV successful
```

---

## 20. Single Area OSPF Cost Manipulation

After Single Area OSPF was enabled, EDGE1 initially learned all internal VLAN routes through both CORE1 and CORE2 with equal cost.

To make CORE1 the preferred path and CORE2 the backup path, the OSPF cost on EDGE1 G0/3 was increased.

Design:

```text
EDGE1 G0/2 toward CORE1: OSPF cost 1
EDGE1 G0/3 toward CORE2: OSPF cost 50
```

Result:

```text
EDGE1 prefers CORE1 for internal VLAN routes.
CORE2 remains available as a backup path.
```

---

## 21. Route Summarization Decision in Single Area OSPF

Route Summarization was discussed during the Single Area OSPF case, but it was not implemented.

Reason:

```text
Current OSPF design is Single Area 0.
EDGE1, CORE1, and CORE2 are all in Area 0.
The VLAN routes are intra-area OSPF routes.
```

Proper OSPF route summarization is normally performed at:

```text
ABR using area range
ASBR using summary-address
```

In this Single Area design:

```text
CORE1 and CORE2 are not ABRs.
There is no inter-area boundary.
Therefore, OSPF area summarization is not appropriate.
```

Final decision:

```text
Single Area OSPF:
- Learn and validate OSPF fundamentals
- Do not apply route summarization

Multi-Area OSPF:
- Reconfigure CORE1/CORE2 as ABRs
- Place VLAN SVIs into a non-backbone area
- Apply area range summarization
```

---

# Case 2 — Multi-Area OSPF

## 22. Multi-Area OSPF Design

After the Single Area OSPF case, the same physical topology was reused and OSPF was reconfigured into Multi-Area OSPF.

The goal of the Multi-Area OSPF case is to implement the route summarization that was intentionally not applied in the Single Area case.

Planned design:

```text
EDGE1 ↔ CORE1/CORE2 routed links = Area 0
CORE1/CORE2 VLAN SVIs           = Area 10
CORE1/CORE2                     = ABR
```

---

## 23. OSPF Area Reconfiguration

The Single Area OSPF configuration was removed before applying Multi-Area OSPF.

Final Multi-Area area placement:

```text
EDGE1:
- G0/2 to CORE1 = Area 0
- G0/3 to CORE2 = Area 0

CORE1:
- G0/0 to EDGE1 = Area 0
- VLAN SVIs     = Area 10

CORE2:
- G0/0 to EDGE1 = Area 0
- VLAN SVIs     = Area 10
```

The ISP-facing interfaces on EDGE1 remain outside OSPF.

---

## 24. Multi-Area OSPF Neighbor and ABR Validation

OSPF neighbors remain the same as the Single Area case.

Expected neighbors on EDGE1:

```text
EDGE1 ↔ CORE1 = FULL
EDGE1 ↔ CORE2 = FULL
```

Validated result:

```text
EDGE1 neighbor 2.2.2.2 via 172.16.0.2 = FULL
EDGE1 neighbor 3.3.3.3 via 172.16.0.6 = FULL
```

CORE1 and CORE2 were validated as ABRs.

Expected ABR state:

```text
It is an area border router
Number of areas in this router is 2
Area BACKBONE(0)
Area 10
```

Validated result:

```text
CORE1 = ABR
CORE2 = ABR
```

---

## 25. Multi-Area Passive Interface Design

The passive-interface design remains consistent with the Single Area case.

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

This ensures that VLAN SVIs are advertised into OSPF while preventing unnecessary neighbor formation on user VLANs.

---

## 26. Inter-Area Route Learning

Before summarization, EDGE1 learned individual VLAN routes as inter-area OSPF routes.

Expected route type:

```text
O IA
```

Observed routes before summarization:

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

This confirmed that VLAN routes were moved from Area 10 into Area 0 as inter-area routes.

---

## 27. Route Summarization with area range

Route summarization was implemented on CORE1 and CORE2.

Summary configuration concept:

```text
area 10 range 10.10.0.0 255.255.0.0
```

Purpose:

```text
Summarize individual Area 10 VLAN routes into a single 10.10.0.0/16 summary route.
Advertise the summary route from CORE1/CORE2 ABRs into Area 0.
Reduce routing table size on EDGE1.
```

---

## 28. Summary Route Verification

After summarization, EDGE1 no longer learned individual /24 VLAN routes.

Before summarization:

```text
O IA 10.10.10.0/24
O IA 10.10.11.0/24
O IA 10.10.20.0/24
...
O IA 10.10.70.0/24
```

After summarization:

```text
O IA 10.10.0.0/16
```

OSPF database validation showed Type-3 Summary LSAs.

Validated result:

```text
Link State ID: 10.10.0.0
Network Mask: /16

Advertising Router:
2.2.2.2
3.3.3.3
```

This confirms that CORE1 and CORE2 are advertising the summarized 10.10.0.0/16 route into Area 0.

---

## 29. Multi-Area OSPF Cost Manipulation

After summarization, EDGE1 initially had two possible paths for the summary route:

```text
10.10.0.0/16 via CORE1
10.10.0.0/16 via CORE2
```

To keep CORE1 as the preferred internal path, the OSPF cost on EDGE1 G0/3 was increased again.

Final design:

```text
EDGE1 G0/2 toward CORE1: OSPF cost 1
EDGE1 G0/3 toward CORE2: OSPF cost 50
```

Validated result:

```text
O IA 10.10.0.0/16 via 172.16.0.2, GigabitEthernet0/2
Metric 2
```

CORE2 remains the backup path.

---

## 30. Default Route Advertisement Validation in Multi-Area OSPF

EDGE1 continues to advertise default route information into OSPF using:

```text
default-information originate
```

Validated result on EDGE1:

```text
0.0.0.0/0 Type-5 External LSA generated by EDGE1
Advertising Router: 1.1.1.1
```

Validated result on CORE1 and CORE2:

```text
CORE1 received 0.0.0.0/0 Type-5 External LSA from EDGE1.
CORE2 received 0.0.0.0/0 Type-5 External LSA from EDGE1.
```

CORE1 and CORE2 still use static default routes in the routing table.

Reason:

```text
Static default route AD = 1
OSPF external route AD = 110
```

Therefore, the OSPF default LSA is present in the OSPF database, while the static default route remains preferred in the RIB.

---

## 31. WAN Resiliency Validation in Multi-Area OSPF

WAN failover was revalidated after the Multi-Area OSPF migration.

Normal state:

```text
Track 1 Up
IP SLA OK
Default route via 100.64.1.1
EDGE1 → INTERNET-SRV source G0/0: Success
```

ISP1 failure test:

```text
EDGE1 G0/0 shutdown
Track 1 Down
IP SLA Timeout
Default route changed to 100.64.2.1
```

ISP2 validation:

```text
EDGE1 → 100.64.2.1: Success
EDGE1 → 203.0.113.2: Success
EDGE1 → INTERNET-SRV source G0/1: Success
```

ISP1 recovery:

```text
EDGE1 G0/0 no shutdown
Track 1 Up
IP SLA OK
Default route returned to 100.64.1.1
EDGE1 → INTERNET-SRV source G0/0: Success
```

Result:

```text
WAN resiliency remained operational after Multi-Area OSPF and route summarization.
```

---

## 32. HSRP Failover Revalidation

HSRP VLAN40 failover was revalidated in the Multi-Area OSPF environment.

Original VLAN40 state:

```text
CORE1 = Active
CORE2 = Standby
VIP   = 10.10.40.254
```

Failure test:

```text
CORE1 Vlan40 shutdown
```

Validated result:

```text
CORE2 became Active for VLAN40.
VLAN40 VIP 10.10.40.254 remained reachable.
```

Recovery test:

```text
CORE1 Vlan40 no shutdown
```

Validated result:

```text
CORE1 returned to Active for VLAN40 due to preempt.
CORE2 returned to Standby.
```

---

## 33. Important Design Observation: Summarization and HSRP Partial Failure

During the HSRP VLAN40 failover test, an important Multi-Area OSPF design limitation was observed.

HSRP itself worked correctly:

```text
CORE1 Vlan40 down
CORE2 became Active for VLAN40
VIP 10.10.40.254 remained available
```

However, EDGE1 only saw the summarized route:

```text
O IA 10.10.0.0/16 via CORE1
```

Because route summarization hides individual VLAN route state from upstream routers, EDGE1 continued to prefer CORE1 for the entire 10.10.0.0/16 summary route.

This can cause return traffic blackholing during a partial SVI failure.

Example failure condition:

```text
CORE1 Vlan40 is down.
CORE2 is now HSRP Active for VLAN40.
EDGE1 still sends return traffic for 10.10.40.0/24 toward CORE1 due to the 10.10.0.0/16 summary route.
CORE1 cannot forward into VLAN40 because Vlan40 is down.
```

Design lesson:

```text
Route summarization improves scalability, but it can hide more-specific VLAN failures from upstream routers.
HSRP gateway failover and OSPF summarization must be considered together.
```

This is an important operational limitation to document before future Security, QoS, and SD-WAN underlay phases.

---

## 34. Failure and Recovery Test

The following failure and recovery tests were validated in the Multi-Area OSPF case.

### Test 1 — ISP1 Failure and Recovery

Failure:

```text
EDGE1 G0/0 shutdown
```

Observed result:

```text
Track 1 Down
Primary default route removed
Backup default route via ISP2 installed
ISP2 path to INTERNET-SRV worked
```

Recovery:

```text
EDGE1 G0/0 no shutdown
```

Observed recovery:

```text
Track 1 Up
Primary default route via ISP1 restored
INTERNET-SRV reachable through ISP1
```

Status:

```text
Passed
```

---

### Test 2 — HSRP VLAN40 Failure and Recovery

Failure:

```text
CORE1 Vlan40 shutdown
```

Observed result:

```text
CORE2 became Active for VLAN40
VIP 10.10.40.254 remained reachable
```

Important observation:

```text
With route summarization, EDGE1 may continue to prefer CORE1 for 10.10.0.0/16.
This can create return-path blackholing during partial SVI failure.
```

Recovery:

```text
CORE1 Vlan40 no shutdown
```

Observed recovery:

```text
CORE1 returned to Active for VLAN40
CORE2 returned to Standby
```

Status:

```text
HSRP failover passed.
Route summarization limitation observed and documented.
```

---

### Test 3 — CORE1 Uplink Failure and Recovery

Failure:

```text
CORE1 G0/0 shutdown
```

Observed result on EDGE1:

```text
EDGE1 lost OSPF neighbor with CORE1.
EDGE1 kept OSPF neighbor with CORE2.
10.10.0.0/16 summary route moved to CORE2 via 172.16.0.6.
Metric became 51 because EDGE1 G0/3 OSPF cost is 50.
```

Recovery:

```text
CORE1 G0/0 no shutdown
```

Observed recovery on EDGE1:

```text
EDGE1 restored OSPF neighbor with CORE1.
10.10.0.0/16 summary route returned to CORE1 via 172.16.0.2.
Metric returned to 2.
```

Status:

```text
Passed
```

---

## 35. NAT/PAT Failover Limitation and Phase 06 Carry-over

Routing failover was validated in Phase 03.

However, NAT/PAT failover was not fully implemented in this phase.

Current EDGE1 NAT/PAT behavior:

```text
ip nat inside source list ENTERPRISE-NAT interface GigabitEthernet0/0 overload
```

Both ISP-facing interfaces are configured as NAT outside:

```text
G0/0 = NAT outside
G0/1 = NAT outside
```

But only G0/0 toward ISP1 is used for PAT overload.

Therefore:

```text
Routing failover to ISP2 works.
NAT/PAT failover to ISP2 is not complete.
```

This means internal user traffic may not be fully NATed correctly through ISP2 after WAN failover.

Final decision:

```text
Phase 03 validates routing failover.
NAT/PAT failover is intentionally deferred.
```

Carry-over to Phase 06:

```text
NAT/PAT failover must be revisited in Phase 06 SD-WAN Underlay Enhancement.
```

Topics to revisit later:

```text
- Route-map based NAT
- ISP-specific NAT policy
- PBR + IP SLA
- DIA
- Local Internet Breakout
- Business intent based path selection
```

---

## 36. Phase 03 Completion Status

The following items were completed in Phase 03:

```text
Single Area OSPF
OSPF neighbor formation
OSPF passive-interface design
OSPF internal route learning
OSPF default route advertisement
Floating static route
IP SLA
Object tracking
WAN resiliency
OSPF cost manipulation
Multi-Area OSPF
ABR validation
Inter-area route learning
Route summarization with area range
Summary route verification
Default route advertisement validation
WAN resiliency validation
HSRP failover revalidation
Failure and recovery test
```

The following item was intentionally not fully implemented in Phase 03:

```text
NAT/PAT Failover:
- Limitation identified
- Full implementation deferred to Phase 06 SD-WAN Underlay Enhancement
```

---

## 37. Key Lessons Learned

```text
1. Single Area OSPF is simple and useful for small networks, but it does not provide a proper ABR summarization point.
2. Multi-Area OSPF enables ABR behavior and inter-area route summarization.
3. VLAN SVI networks can be placed into a non-backbone area while routed core-edge links remain in Area 0.
4. EDGE1 sees individual O IA routes before summarization.
5. After area range summarization, EDGE1 sees only O IA 10.10.0.0/16.
6. OSPF cost manipulation can still influence the preferred path for a summary route.
7. IP SLA and object tracking provide WAN failover independently from the OSPF internal design.
8. OSPF default route LSAs can exist even when static default routes remain preferred in the routing table.
9. HSRP failover and routing failover are different events.
10. Route summarization can hide individual VLAN failures from upstream routers.
11. Summarization and HSRP partial failure behavior must be considered together.
12. NAT/PAT failover is separate from routing failover and must be handled in a later phase.
```

---

## 38. Final Notes

Phase 03 successfully introduced routing resiliency into the collapsed-core campus design.

The lab validated both:

```text
Case 1 — Single Area OSPF
Case 2 — Multi-Area OSPF
```

The Single Area case established the basic OSPF routing foundation.

The Multi-Area case added ABR behavior, inter-area routing, and route summarization.

Final Multi-Area OSPF state:

```text
EDGE1 ↔ CORE1/CORE2 routed links = Area 0
CORE1/CORE2 VLAN SVIs           = Area 10
CORE1/CORE2                     = ABR
EDGE1 learns                    = O IA 10.10.0.0/16
Preferred internal path          = EDGE1 → CORE1
Backup internal path             = EDGE1 → CORE2
WAN primary path                 = EDGE1 → ISP1
WAN backup path                  = EDGE1 → ISP2
```

Next phase:

```text
Phase 04 — Security
```
