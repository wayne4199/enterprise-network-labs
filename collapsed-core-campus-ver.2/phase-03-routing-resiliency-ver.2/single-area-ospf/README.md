# Phase 03 — Routing Resiliency Ver.2

## 1. Overview

This phase focuses on routing resiliency in the Collapsed Core Campus Ver.2 lab.

The goal is to build a resilient routing foundation on top of the completed Phase 01 Foundation and Phase 02 Enterprise Services.

Phase 03 is intentionally divided into two practical OSPF cases:

```text
Case 1 — Single Area OSPF
Case 2 — Multi-Area OSPF
```

This README currently documents **Case 1 — Single Area OSPF**.

The Multi-Area OSPF case will be added after the Single Area OSPF case is fully documented and validated.

---

## 2. Lab Path

```text
enterprise-network-labs/
└── collapsed-core-campus/
    ├── phase-01-foundation-ver.2/
    ├── phase-02-enterprise-services-ver.2/
    └── phase-03-routing-resiliency-ver.2/
```

---

## 3. Phase 03 Learning Scope

Phase 03 covers the following routing resiliency topics:

```text
1. OSPF Single Area 0
2. OSPF Neighbor / Passive Interface
3. OSPF Default Route Advertisement
4. Floating Static Route
5. IP SLA
6. Object Tracking
7. WAN Resiliency
8. OSPF Cost Manipulation
9. Route Summarization Design Decision
10. HSRP Failover Validation
11. NAT/PAT Failover Limitation and Phase 06 Carry-over
12. Failure and Recovery Test
```

Important distinction:

```text
Route Summarization was not implemented in the Single Area OSPF case.
It will be implemented in the Multi-Area OSPF case.

NAT/PAT failover was not fully implemented in Phase 03.
The limitation was identified and intentionally carried over to Phase 06.
```

---

## 4. Existing Baseline from Previous Phases

Phase 03 must preserve the completed Phase 01 and Phase 02 baseline.

The following items must not be removed or redesigned during Phase 03:

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

## 10. OSPF Router IDs

```text
EDGE1  router-id 1.1.1.1
CORE1  router-id 2.2.2.2
CORE2  router-id 3.3.3.3
```

Router IDs were manually configured for clarity and deterministic neighbor identification.

---

## 11. OSPF Network Design

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

## 12. OSPF Passive Interface Design

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

Important operational note:

During the lab, an unexpected passive-interface state was observed and corrected.

This reinforced the importance of validating existing routing protocol configuration before introducing OSPF.

Future routing pre-checks should include:

```cisco
show ip protocols
show running-config | section router ospf
show running-config | section router eigrp
show running-config | section router bgp
```

---

## 13. OSPF Neighbor Validation

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

## 14. OSPF Route Learning

After OSPF neighbor formation, EDGE1 learned the internal VLAN routes through OSPF.

Observed OSPF routes:

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

Initially, EDGE1 learned the VLAN routes through both CORE1 and CORE2 with equal OSPF cost.

This created an ECMP condition:

```text
EDGE1 → CORE1
EDGE1 → CORE2
```

This was expected before OSPF cost manipulation.

---

## 15. Internal Static Route Removal

Before OSPF was introduced, EDGE1 had static summary routes toward the internal campus network.

Existing internal static routes:

```text
10.10.0.0/16 → CORE1
10.10.0.0/16 → CORE2 with higher AD
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

Validated result:

```text
IP SLA 1: OK
Track 1: Up
Default route: 100.64.1.1
```

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

At first, ping to INTERNET-SRV failed through ISP2 because the return path was incomplete.

The issue was not the EDGE1 failover logic.

The problem was the return path from the Internet segment back to the ISP2 WAN path.

A return route was added on ISP1:

```text
100.64.2.0/30 → 203.0.113.2
```

After this route was added, EDGE1 successfully reached INTERNET-SRV through ISP2.

Validated result:

```text
EDGE1 → ISP2 → INTERNET-SRV: Success
```

After EDGE1 G0/0 was restored, Track 1 returned to Up and the default route moved back to ISP1.

Validated recovery:

```text
Track 1 Up
Default route returned to 100.64.1.1
Ping to INTERNET-SRV successful
```

---

## 20. OSPF Cost Manipulation

After Single Area OSPF was enabled, EDGE1 initially learned all internal VLAN routes through both CORE1 and CORE2 with equal cost.

This created ECMP.

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

Validated result:

```text
10.10.x.0/24 routes were installed via 172.16.0.2 through EDGE1 G0/2.
```

---

## 21. Route Summarization Design Decision

Route Summarization was discussed during the Single Area OSPF case, but it was **not implemented**.

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

Route Summarization will be implemented and validated in the Multi-Area OSPF case.

---

## 22. HSRP Failover Validation

HSRP Failover was validated using VLAN40.

Original VLAN40 state:

```text
CORE1 = Active
CORE2 = Standby
VIP   = 10.10.40.254
```

A VLAN40 SVI failure was simulated on CORE1.

Expected behavior:

```text
CORE1 VLAN40 goes down
CORE2 becomes Active for VLAN40
MGMT-SRV1 continues to use VIP 10.10.40.254
```

Validated result:

```text
CORE2 became Active for VLAN40.
MGMT-SRV1 continued to reach the VLAN40 VIP.
MGMT-SRV1 maintained reachability to EDGE1 and INTERNET-SRV.
```

This confirmed that HSRP gateway redundancy works for the management VLAN.

Important observation:

A separate CORE1 uplink failure test was also observed.

When CORE1 G0/0 was shut down, HSRP did not automatically fail over because the VLAN40 SVI remained active.

This created an important design observation:

```text
Uplink failure alone does not guarantee HSRP role change.
If HSRP tracking is not configured, a core switch can remain Active even when its routed uplink fails.
```

This can cause traffic blackholing for VLANs where that core switch remains the HSRP Active gateway.

This observation should be considered for future HSRP interface tracking enhancement.

---

## 23. NAT/PAT Failover Limitation and Phase 06 Carry-over

Routing failover was validated in Phase 03.

However, NAT/PAT failover was **not fully implemented** in this phase.

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

This means that internal user traffic may not be fully NATed correctly through ISP2 after WAN failover.

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

## 24. Failure and Recovery Test Summary

The following failure and recovery scenarios were validated during the Single Area OSPF case.

### Test 1 — ISP1 Failure and Recovery

Failure:

```text
EDGE1 G0/0 shutdown
```

Observed result:

```text
IP SLA failed
Track 1 went Down
Primary default route was removed
Backup default route via ISP2 was installed
```

After return path correction, EDGE1 successfully reached INTERNET-SRV through ISP2.

Recovery:

```text
EDGE1 G0/0 no shutdown
```

Observed recovery:

```text
Track 1 returned Up
Default route returned to ISP1
Internet reachability restored through primary path
```

---

### Test 2 — CORE1 Uplink Failure and Recovery

Failure:

```text
CORE1 G0/0 shutdown
```

Observed result:

```text
CORE1 OSPF neighbor with EDGE1 went down
EDGE1 used CORE2 path for internal VLAN routes
```

Important observation:

```text
HSRP did not automatically fail over because the VLAN SVI remained active.
```

This test confirmed the difference between:

```text
Routing adjacency failure
HSRP gateway failover
```

Future enhancement:

```text
HSRP interface tracking should be considered if uplink failure must trigger gateway role change.
```

---

### Test 3 — HSRP VLAN40 Failure and Recovery

Failure:

```text
CORE1 Vlan40 shutdown
```

Observed result:

```text
CORE2 became Active for VLAN40
VLAN40 VIP remained reachable
MGMT-SRV1 maintained connectivity
```

Recovery:

```text
CORE1 Vlan40 no shutdown
```

Expected recovery:

```text
CORE1 returns as Active for VLAN40 if preempt is configured.
CORE2 returns to Standby.
```

---

## 25. Single Area OSPF Case Completion Status

The following items were completed in the Single Area OSPF case:

```text
OSPF Single Area 0
OSPF Neighbor / Passive Interface
OSPF Default Route Advertisement
Floating Static Route
IP SLA
Object Tracking
WAN Resiliency
OSPF Cost Manipulation
HSRP Failover Validation
Failure and Recovery Test
```

The following items were intentionally not implemented in the Single Area OSPF case:

```text
Route Summarization:
- Not applied in Single Area OSPF
- Deferred to Multi-Area OSPF

NAT/PAT Failover:
- Limitation identified
- Full implementation deferred to Phase 06 SD-WAN Underlay Enhancement
```

Single Area OSPF Case is complete.

---

# Case 2 — Multi-Area OSPF

## 26. Multi-Area OSPF Case Plan

The next part of Phase 03 will reconfigure the OSPF design from Single Area to Multi-Area.

The physical topology will remain the same.

The OSPF logical design will change.

Planned Multi-Area design:

```text
EDGE1 ↔ CORE1/CORE2 routed links = Area 0
CORE1/CORE2 VLAN SVIs = Area 10
CORE1/CORE2 = ABR
```

Planned learning goals:

```text
1. Multi-Area OSPF design
2. ABR behavior
3. VLAN SVI area separation
4. Inter-area route learning
5. Route summarization with area range
6. Summary route validation on EDGE1
7. Impact on future Security / QoS / SD-WAN Underlay phases
```

Expected summarization result:

```text
EDGE1 learns summary route:
10.10.0.0/16

Instead of individual routes:
10.10.10.0/24
10.10.11.0/24
10.10.20.0/24
...
10.10.70.0/24
```

---

## 27. Why Both Single Area and Multi-Area Are Included

Both Single Area and Multi-Area OSPF are used in real environments.

Single Area OSPF is common in smaller or simpler networks.

Multi-Area OSPF is common in larger, hierarchical, or more structured enterprise networks.

This lab intentionally includes both cases.

Reason:

```text
The goal is not to choose only the easier design.
The goal is to learn both practical routing models and understand their impact on future phases.
```

Future phases depend on this routing foundation:

```text
Phase 04 — Security
Phase 05 — QoS
Phase 06 — SD-WAN Underlay Enhancement
```

Therefore, Phase 03 must provide both:

```text
Simple routing baseline through Single Area OSPF
Practical scalable routing baseline through Multi-Area OSPF
```

---

## 28. Final Notes

Phase 03 Single Area OSPF successfully introduced dynamic routing and WAN resiliency into the collapsed-core campus design.

The lab validated:

```text
- OSPF neighbor formation
- Passive interface design
- Default route advertisement
- Dynamic internal route learning
- Static-to-OSPF internal route migration
- Floating static default route
- IP SLA and object tracking
- ISP failover and recovery
- OSPF cost manipulation
- HSRP failover behavior
- NAT/PAT failover limitation
- Failure and recovery behavior
```

Next step:

```text
Proceed to Multi-Area OSPF Case.
```
