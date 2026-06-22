# Phase 06 — SD-WAN-Ready Underlay Enhancement Design Decisions

## 1. Purpose

This document explains the major design decisions made in:

```text
Phase 06 — SD-WAN-Ready Underlay Enhancement
```

Phase 06 was designed to prepare the existing enterprise WAN underlay for future SD-WAN overlay learning.

The goal was not to deploy SD-WAN yet.
The goal was to make the existing dual-ISP WAN edge more resilient, policy-aware, and SD-WAN-ready.

---

## 2. Design Baseline

Phase 06 continues from the final state of:

```text
Phase 05 — QoS
```

The operational baseline includes:

```text
Phase 01 — Foundation
Phase 02 — Enterprise Services
Phase 03 — Multi-Area OSPF Routing Resiliency
Phase 04 — Security
Phase 05 — QoS
```

Phase 06 does not restart from the earlier Single Area OSPF case.

The selected baseline remains:

```text
Phase 03 Case 2 — Multi-Area OSPF
```

Reason:

```text
Multi-Area OSPF better represents a scalable enterprise campus design.
It also preserves ABR behavior, summarization, and realistic routing boundaries.
```

---

## 3. Title Decision

The selected title is:

```text
Phase 06 — SD-WAN-Ready Underlay Enhancement
```

This title was chosen because Phase 06 does not deploy the SD-WAN overlay itself.

The following SD-WAN overlay components are intentionally not configured in this phase:

```text
SD-WAN Manager
SD-WAN Controller
SD-WAN Validator
cEdge onboarding
OMP
TLOC
VPN segmentation
Centralized policy
App-Aware Routing
BFD tunnel health
```

Instead, this phase enhances the traditional WAN underlay so that the next phase can transition naturally into SD-WAN overlay concepts.

---

## 4. Decision to Keep Phase 06 on the Existing Campus Topology

The existing collapsed-core campus topology was preserved for Phase 06.

Reason:

```text
Phase 06 focuses on improving the current dual-ISP WAN edge.
It needs the existing campus hosts, QoS classes, OSPF routing, and security baseline.
```

A separate SD-WAN overlay topology will be used later in Phase 07.

Reason:

```text
Phase 07 requires SD-WAN controllers, SD-WAN edge routers, and a branch site.
Keeping the full Phase 01–06 campus topology and adding all SD-WAN overlay nodes would exceed the CML node budget and make the lab unnecessarily complex.
```

Final decision:

```text
Phase 06:
Use the existing full campus underlay topology.

Phase 07:
Use a reduced HQ campus plus SD-WAN controllers and branch site.
```

---

## 5. Decision to Keep All Major Phase 06 Configuration on EDGE1

Most Phase 06 configuration was applied only on:

```text
EDGE1
```

Reason:

```text
The Phase 06 scope is WAN underlay enhancement.
EDGE1 is the enterprise WAN edge connecting the campus to ISP1 and ISP2.
```

No major new Phase 06 configuration was required on:

```text
CORE1
CORE2
ACC1
ACC2
ACC3
SRV-ACC1
ISP1
ISP2
INTERNET-SRV
```

Reason:

```text
Those devices already provided the required campus, routing, security, and test-server functions from previous phases.
```

---

## 6. Decision to Replace Single-Interface PAT

### Previous Design

Before Phase 06, NAT/PAT was tied to ISP1:

```cisco
ip nat inside source list ENTERPRISE-NAT interface GigabitEthernet0/0 overload
```

This worked only while traffic exited through ISP1.

### Problem

When ISP1 failed, routing moved to ISP2, but NAT/PAT was still tied to Gi0/0.

Result:

```text
Routing failover: PASS
NAT/PAT failover: FAIL
```

### Final Design

The old single-interface PAT design was replaced with route-map based dual-ISP NAT/PAT:

```cisco
ip nat inside source route-map NAT-ISP1 interface GigabitEthernet0/0 overload
ip nat inside source route-map NAT-ISP2 interface GigabitEthernet0/1 overload
```

Route-map logic:

```cisco
route-map NAT-ISP1 permit 10
 match ip address ENTERPRISE-NAT
 match interface GigabitEthernet0/0

route-map NAT-ISP2 permit 10
 match ip address ENTERPRISE-NAT
 match interface GigabitEthernet0/1
```

Reason:

```text
NAT/PAT must follow the actual WAN egress interface.
If traffic exits ISP1, it should be translated to 100.64.1.2.
If traffic exits ISP2, it should be translated to 100.64.2.2.
```

Final decision:

```text
Use route-map based dual-ISP NAT/PAT.
```

---

## 7. Decision to Track Both ISP Paths

### Previous Design

Before Phase 06:

```text
Track 1 monitored ISP1.
ISP2 backup default route existed as a floating static route.
```

The ISP2 backup route was not tied to ISP2 reachability.

### Problem

A floating static route without tracking may still be configured even if the backup ISP path is unusable.

### Final Design

Track both ISP paths:

```text
Track 1 → ISP1
Track 2 → ISP2
```

Primary route:

```cisco
ip route 0.0.0.0 0.0.0.0 100.64.1.1 track 1
```

Backup route:

```cisco
ip route 0.0.0.0 0.0.0.0 100.64.2.1 10 track 2
```

Reason:

```text
The backup route should only be valid when ISP2 is reachable.
```

Final decision:

```text
Use Track 1 for ISP1 and Track 2 for ISP2.
Tie both default routes to tracking objects.
```

---

## 8. Decision to Use Different IP SLA Targets for ISP1 and ISP2

### ISP1 IP SLA Target

ISP1 is monitored using:

```cisco
icmp-echo 203.0.113.10 source-interface GigabitEthernet0/0
```

A host route is used:

```cisco
ip route 203.0.113.10 255.255.255.255 100.64.1.1
```

Reason:

```text
The host route keeps the ISP1 SLA probe tied to ISP1.
Without this, the SLA probe could potentially succeed through ISP2 after failover, which would hide the ISP1 failure.
```

### ISP2 IP SLA Target

ISP2 is monitored using:

```cisco
icmp-echo 100.64.2.1 source-interface GigabitEthernet0/1
```

Reason:

```text
For the backup route, the key question is whether the ISP2 next-hop is reachable.
Tracking the next-hop is sufficient for this lab scope.
```

Final decision:

```text
Use INTERNET-SRV reachability through ISP1 for Track 1.
Use ISP2 next-hop reachability for Track 2.
```

---

## 9. Decision to Add Track Delay

Both tracking objects use delay timers:

```cisco
delay down 10 up 5
```

Reason:

```text
Track delay helps reduce unnecessary route flapping during brief reachability changes.
```

Final decision:

```text
Use delay down 10 seconds and delay up 5 seconds for both Track 1 and Track 2.
```

---

## 10. Decision to Use Track-Aware PBR

Traditional PBR can force traffic to a next-hop even when that next-hop is down.

To avoid this, Phase 06 used:

```cisco
set ip next-hop verify-availability <NEXT-HOP> 10 track <TRACK-ID>
```

Reason:

```text
PBR should express preferred path intent, not force traffic into a failed WAN path.
```

If the preferred ISP is healthy:

```text
PBR uses the preferred next-hop.
```

If the preferred ISP is down:

```text
PBR does not force the failed next-hop.
Traffic falls back to normal routing.
```

Final decision:

```text
Use track-aware PBR with verify-availability.
```

---

## 11. Decision to Apply PBR on EDGE1 Campus-Facing Interfaces

PBR was applied inbound on:

```text
EDGE1 Gi0/2
EDGE1 Gi0/3
```

Configuration:

```cisco
interface GigabitEthernet0/2
 ip policy route-map WAN-BUSINESS-INTENT

interface GigabitEthernet0/3
 ip policy route-map WAN-BUSINESS-INTENT
```

Reason:

```text
Traffic should be policy-routed before it exits the WAN.
Applying PBR inbound on the campus-facing interfaces allows EDGE1 to steer traffic based on original internal source addresses before NAT/PAT occurs.
```

Final decision:

```text
Apply PBR inbound on EDGE1 inside-facing interfaces.
```

---

## 12. Decision to Preserve Phase 05 QoS Design

Phase 05 QoS already established a two-stage QoS design:

```text
Stage 1:
Classify and mark traffic on EDGE1 campus-facing ingress interfaces.

Stage 2:
Queue traffic on EDGE1 WAN-facing egress interfaces based on DSCP.
```

This design was preserved.

Policy placement:

```text
Gi0/2 input  → CAMPUS-QOS-MARK-IN
Gi0/3 input  → CAMPUS-QOS-MARK-IN

Gi0/0 output → WAN-QOS-EGRESS
Gi0/1 output → WAN-QOS-EGRESS
```

Reason:

```text
NAT/PAT changes source addresses on WAN egress.
Therefore, QoS classification based on original inside source IPs should happen before NAT.
WAN egress queuing should then use DSCP values.
```

Final decision:

```text
Do not redesign QoS.
Preserve Phase 05 ingress marking and WAN egress queuing.
```

---

## 13. Decision to Simulate Application-Aware Path Selection with PBR

Phase 06 does not use SD-WAN App-Aware Routing.

Instead, it simulates the same design idea using traditional PBR and tracking.

Final business intent mapping:

| Traffic Type      | Host                    | Preferred ISP |
| ----------------- | ----------------------- | ------------- |
| VOICE-like        | EXEC / 10.10.10.117     | ISP1          |
| VIDEO-like        | SALES / 10.10.20.197    | ISP2          |
| BUSINESS-CRITICAL | IT-OPS / 10.10.30.140   | ISP1          |
| MGMT              | MGMT-SRV1 / 10.10.40.10 | ISP1          |
| SCAVENGER         | ROGUE-PC / 10.10.20.126 | ISP2          |

Reason:

```text
This gives practical experience with business-intent path selection before learning centralized SD-WAN policies.
```

Final decision:

```text
Use PBR + Track to simulate SD-WAN-like application-aware path selection.
```

---

## 14. Decision to Use Host-Based ACLs for PBR

PBR ACLs match specific lab hosts:

```text
EXEC        10.10.10.117
SALES       10.10.20.197
IT-OPS      10.10.30.140
MGMT-SRV1   10.10.40.10
ROGUE-PC    10.10.20.126
```

Reason:

```text
This lab uses ICMP traffic to simulate application classes.
Host-based ACLs make verification simple and deterministic.
```

Known limitation:

```text
Some hosts received DHCP addresses.
If those addresses change after a lab restart, the PBR and QoS ACLs must be updated or DHCP reservations should be created.
```

Final decision:

```text
Use host-based ACLs for controlled lab validation.
Consider DHCP reservations for long-term stability.
```

---

## 15. Decision to Keep INTERNET-SRV as the Common Test Destination

All test traffic used:

```text
203.0.113.10
```

Reason:

```text
Using a common destination makes path, NAT/PAT, PBR, and QoS behavior easier to compare across traffic classes.
```

Final decision:

```text
Use INTERNET-SRV 203.0.113.10 as the common validation destination.
```

---

## 16. Decision to Keep SD-WAN Controllers Out of Phase 06

No SD-WAN controllers were added in this phase.

Reason:

```text
Phase 06 focuses on underlay readiness.
Adding controllers would change the scope from underlay enhancement to overlay deployment.
```

Controllers are reserved for:

```text
Phase 07 — SD-WAN Overlay
```

Final decision:

```text
Do not add SD-WAN Manager, Controller, Validator, or cEdge onboarding in Phase 06.
```

---

## 17. Decision to Avoid Adding New Nodes

The lab environment has a CML node limit.

Phase 06 reused the existing nodes.

Reason:

```text
The existing topology already includes enough hosts to simulate traffic classes:
VOICE, VIDEO, BUSINESS, MGMT, and SCAVENGER.
```

Final decision:

```text
Reuse existing hosts and avoid unnecessary new nodes.
```

---

## 18. Decision to Preserve Existing Security Baseline

Phase 04 security controls were preserved.

No Phase 06 changes were made to:

```text
DHCP Snooping
DAI
Port Security
BPDU Guard
Management ACL
VTY SSH restriction
```

Reason:

```text
Phase 06 focuses on WAN underlay enhancement.
Security baseline should remain stable and not be redesigned unless required.
```

Final decision:

```text
Preserve Phase 04 security configuration.
```

---

## 19. Decision to Preserve Existing Multi-Area OSPF Design

Phase 06 did not redesign internal OSPF.

Reason:

```text
OSPF was already validated in Phase 03 and used as the operational baseline in Phase 04 and Phase 05.
Phase 06 focuses on WAN edge behavior, not internal routing redesign.
```

Known design consideration:

```text
OSPF summarization may hide individual VLAN failures from EDGE1.
This was already documented as a Phase 03 design trade-off.
```

Final decision:

```text
Preserve the Multi-Area OSPF baseline and avoid unnecessary routing redesign.
```

---

## 20. Decision to Test Both Preferred Path and Fallback Behavior

For each major policy, both normal and failure behavior were tested.

Examples:

```text
MGMT normally prefers ISP1.
If ISP1 fails, MGMT falls back to ISP2.

ROGUE normally prefers ISP2.
If ISP2 fails, ROGUE falls back to ISP1.

VIDEO normally prefers ISP2.
If ISP2 fails, VIDEO falls back to ISP1.
```

Reason:

```text
A preferred path design is incomplete unless failure behavior is also validated.
```

Final decision:

```text
Validate preferred path and fallback path behavior.
```

---

## 21. Final Design Result

The final design provides:

```text
Dual-ISP routing failover
Dual-ISP NAT/PAT failover
Track-aware default route control
Track-aware PBR
Application-aware path selection simulation
QoS marking and WAN queuing preservation
WAN path fallback during ISP failure
```

Final result:

```text
Phase 06 — SD-WAN-Ready Underlay Enhancement
Design Status: Completed
Result: PASS
```

---

## 22. Relationship to Phase 07

Phase 06 prepares the concepts needed for Phase 07.

| Phase 06 Design        | Phase 07 SD-WAN Concept          |
| ---------------------- | -------------------------------- |
| IP SLA / Track         | BFD / TLOC health                |
| PBR                    | Centralized data policy          |
| Preferred ISP          | Preferred color / TLOC           |
| Static route failover  | OMP-based routing                |
| Route-map NAT/PAT      | DIA / local breakout behavior    |
| MQC QoS                | SD-WAN QoS and App-Aware Routing |
| Manual business intent | Controller-based business policy |

Phase 06 is therefore the bridge between traditional enterprise WAN design and SD-WAN overlay design.
