# Phase 03 — Routing Resiliency Ver.2 Verification

## 1. Purpose

This document records verification steps and expected results for **Phase 03 — Routing Resiliency Ver.2**.

Phase 03 includes two OSPF cases:

```text
Case 1 — Single Area OSPF
Case 2 — Multi-Area OSPF
```

This verification document confirms:

```text
- Single Area OSPF operation
- Multi-Area OSPF operation
- ABR behavior
- Inter-area route learning
- Route summarization
- OSPF cost manipulation
- OSPF default route advertisement
- WAN resiliency
- HSRP failover behavior
- Failure and recovery behavior
- NAT/PAT failover limitation
```

The following recovery items are intentionally excluded from this document:

```text
- VLAN Database loss recovery
- MGMT-SRV1 netplan loss recovery
- INTERNET-SRV IP/Gateway loss recovery
```

---

## 2. Verification Scope

Phase 03 verifies the following items:

```text
1. Phase 02 baseline service continuity
2. Single Area OSPF
3. OSPF neighbor formation
4. OSPF passive-interface behavior
5. OSPF internal route learning
6. EDGE1 internal static route removal
7. OSPF default route advertisement
8. Floating static route
9. IP SLA
10. Object tracking
11. WAN failover and recovery
12. OSPF cost manipulation
13. Multi-Area OSPF
14. ABR validation
15. Inter-area route learning
16. Route summarization with area range
17. Summary route verification
18. Default route advertisement validation
19. HSRP failover revalidation
20. CORE1 uplink failure and recovery
21. NAT/PAT failover limitation
```

---

# Baseline Verification

## 3. Phase 02 Baseline Verification

Before validating Phase 03 routing resiliency, verify that the Phase 02 Enterprise Services baseline is still operational.

## 3.1 MGMT-SRV1 IP and Default Route

Run on MGMT-SRV1:

```bash
ip addr show ens2
ip route
```

Expected result:

```text
MGMT-SRV1 IP: 10.10.40.10/24
Default gateway: 10.10.40.254
```

Result:

```text
Verified successfully.
```

---

## 3.2 MGMT-SRV1 Service Status

Run on MGMT-SRV1:

```bash
systemctl status dnsmasq --no-pager
systemctl status chrony --no-pager
systemctl status rsyslog --no-pager
```

Expected result:

```text
dnsmasq: active
chrony: active
rsyslog: active
```

Result:

```text
Verified successfully.
```

---

## 3.3 campus.lab DNS Static Records

Run on MGMT-SRV1:

```bash
dig @10.10.40.10 server1.campus.lab
dig @10.10.40.10 mgmt-srv1.campus.lab
dig @10.10.40.10 internet-srv.campus.lab
```

Expected result:

```text
server1.campus.lab      → 10.10.70.10
mgmt-srv1.campus.lab    → 10.10.40.10
internet-srv.campus.lab → 203.0.113.10
```

Result:

```text
Verified successfully.
```

---

## 3.4 End-to-End Baseline Reachability

Run on MGMT-SRV1:

```bash
ping -c 4 10.10.40.254
ping -c 4 172.16.0.1
ping -c 4 203.0.113.10
```

Expected result:

```text
10.10.40.254 reachable
172.16.0.1 reachable
203.0.113.10 reachable
```

Result:

```text
Verified successfully.
```

---

# Case 1 — Single Area OSPF Verification

## 4. EDGE1 Interface Baseline

Run on EDGE1:

```cisco
show ip interface brief
```

Expected result:

```text
G0/0 100.64.1.2  up/up  → ISP1
G0/1 100.64.2.2  up/up  → ISP2
G0/2 172.16.0.1  up/up  → CORE1
G0/3 172.16.0.5  up/up  → CORE2
```

Result:

```text
Verified successfully.
```

---

## 5. CORE1 / CORE2 Routed Link Baseline

Run on CORE1:

```cisco
show ip interface brief
```

Expected result:

```text
CORE1 G0/0 172.16.0.2 up/up
```

Run on CORE2:

```cisco
show ip interface brief
```

Expected result:

```text
CORE2 G0/0 172.16.0.6 up/up
```

Result:

```text
Verified successfully.
```

---

## 6. Single Area OSPF Neighbor Verification

## 6.1 EDGE1 OSPF Neighbor Verification

Run on EDGE1:

```cisco
show ip ospf neighbor
show ip ospf interface brief
```

Expected neighbors:

```text
Neighbor 2.2.2.2 → CORE1 → FULL
Neighbor 3.3.3.3 → CORE2 → FULL
```

Expected OSPF interfaces:

```text
G0/2 172.16.0.1 Area 0
G0/3 172.16.0.5 Area 0
```

Result:

```text
Verified successfully.
```

---

## 6.2 CORE1 OSPF Neighbor Verification

Run on CORE1:

```cisco
show ip ospf neighbor
show ip ospf interface brief
show running-config | section router ospf
```

Expected result:

```text
CORE1 forms OSPF neighbor only with EDGE1.
Neighbor ID: 1.1.1.1
State: FULL
Interface: GigabitEthernet0/0
```

Expected passive-interface behavior:

```text
G0/0       → non-passive
VLAN SVIs → passive
```

Result:

```text
Verified successfully.
```

---

## 6.3 CORE2 OSPF Neighbor Verification

Run on CORE2:

```cisco
show ip ospf neighbor
show ip ospf interface brief
show running-config | section router ospf
```

Expected result:

```text
CORE2 forms OSPF neighbor only with EDGE1.
Neighbor ID: 1.1.1.1
State: FULL
Interface: GigabitEthernet0/0
```

Expected passive-interface behavior:

```text
G0/0       → non-passive
VLAN SVIs → passive
```

Result:

```text
Verified successfully.
```

---

## 7. Single Area OSPF Route Learning Verification

Run on EDGE1:

```cisco
show ip route ospf
```

Expected OSPF-learned internal VLAN routes:

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

Result:

```text
Verified successfully.
```

---

## 8. EDGE1 Internal Static Route Removal Verification

Run on EDGE1:

```cisco
show running-config | include ^ip route 10.10
show ip route 10.10.40.0
show ip route ospf
```

Expected result:

```text
No 10.10.0.0/16 static route remains on EDGE1.
10.10.40.0/24 is learned through OSPF.
Other internal VLAN routes are also learned through OSPF.
```

Result:

```text
Verified successfully.
```

---

## 9. Single Area OSPF Default Route Advertisement Verification

## 9.1 EDGE1 Default LSA Verification

Run on EDGE1:

```cisco
show running-config | section router ospf
show ip ospf database external
```

Expected result:

```text
default-information originate is configured.

Type-5 AS External LSA exists:
Link State ID: 0.0.0.0
Advertising Router: 1.1.1.1
Network Mask: /0
Metric Type: E2
Metric: 1
```

Result:

```text
Verified successfully.
```

---

## 9.2 CORE1 / CORE2 Default LSA Verification

Run on CORE1 / CORE2:

```cisco
show ip ospf database external
show ip route 0.0.0.0
```

Expected result:

```text
CORE1 and CORE2 receive Type-5 default LSA from EDGE1.
```

Observed routing table behavior:

```text
Static default route remains preferred if it still exists.
```

Reason:

```text
Static route AD = 1
OSPF external route AD = 110
```

Result:

```text
Verified successfully.
```

---

## 10. Floating Static Route Verification

Run on EDGE1:

```cisco
show running-config | include ^ip route
show ip route 203.0.113.10
show ip route 0.0.0.0
```

Expected result:

```text
Host route:
203.0.113.10/32 → 100.64.1.1

Primary default route:
0.0.0.0/0 → 100.64.1.1 track 1

Backup floating default route:
0.0.0.0/0 → 100.64.2.1 AD 10
```

Normal routing state:

```text
Default route uses 100.64.1.1 when Track 1 is Up.
```

Result:

```text
Verified successfully.
```

---

## 11. IP SLA and Object Tracking Verification

Run on EDGE1:

```cisco
show ip sla summary
show ip sla statistics 1
show track 1
```

Expected result:

```text
IP SLA 1 status: OK
Track 1: Up
```

Normal condition:

```text
EDGE1 uses ISP1 as the primary default path.
```

Result:

```text
Verified successfully.
```

---

## 12. Single Area WAN Resiliency Verification

## 12.1 ISP1 Failure Test

On EDGE1:

```cisco
conf t
interface GigabitEthernet0/0
 shutdown
end
```

Wait 10 to 15 seconds.

Then verify on EDGE1:

```cisco
show track 1
show ip route 0.0.0.0
show ip sla summary
```

Expected result:

```text
Track 1: Down
Primary default route removed
Backup default route via 100.64.2.1 installed
```

Result:

```text
Verified successfully.
```

---

## 12.2 ISP2 Path Verification

During ISP1 failure, run on EDGE1:

```cisco
ping 100.64.2.1
ping 203.0.113.2
ping 203.0.113.10 source GigabitEthernet0/1
```

Expected result:

```text
100.64.2.1 reachable
203.0.113.2 reachable
203.0.113.10 reachable using G0/1 source
```

Result:

```text
Verified successfully.
```

---

## 12.3 ISP1 Recovery Test

On EDGE1:

```cisco
conf t
interface GigabitEthernet0/0
 no shutdown
end
```

Then verify:

```cisco
show track 1
show ip route 0.0.0.0
show ip sla summary
ping 203.0.113.10 source GigabitEthernet0/0
```

Expected result:

```text
Track 1: Up
IP SLA OK
Default route returns to 100.64.1.1
INTERNET-SRV reachable through primary path
```

Result:

```text
Verified successfully.
```

---

## 13. Single Area OSPF Cost Manipulation Verification

Run on EDGE1:

```cisco
show ip ospf interface brief
show ip route ospf
show ip route 10.10.40.0
```

Expected OSPF interface cost:

```text
EDGE1 G0/2 toward CORE1: cost 1
EDGE1 G0/3 toward CORE2: cost 50
```

Expected route result:

```text
10.10.x.0/24 routes prefer 172.16.0.2 through G0/2.
CORE1 path is preferred.
CORE2 path remains available as backup.
```

Result:

```text
Verified successfully.
```

---

## 14. Single Area Route Summarization Verification Decision

Route Summarization is not verified as an implemented feature in the Single Area OSPF case.

Reason:

```text
Current OSPF design is Single Area 0.
EDGE1, CORE1, CORE2, and VLAN SVIs are all in Area 0.
CORE1 and CORE2 are not ABRs.
There is no inter-area boundary for OSPF area summarization.
```

Expected Single Area result:

```text
EDGE1 learns individual intra-area /24 VLAN routes.
```

Final decision:

```text
Route Summarization is implemented and verified in the Multi-Area OSPF case.
```

---

# Case 2 — Multi-Area OSPF Verification

## 15. Single Area OSPF Removal Verification

Before applying Multi-Area OSPF, Single Area OSPF was removed.

Run on CORE1 / CORE2 / EDGE1:

```cisco
show running-config | section router ospf
show ip ospf neighbor
show ip route ospf
```

Expected result before Multi-Area reconfiguration:

```text
router ospf 1 removed.
OSPF neighbors removed.
OSPF routes removed.
```

On EDGE1:

```cisco
show running-config interface GigabitEthernet0/3
```

Expected result:

```text
ip ospf cost 50 removed before Multi-Area reconfiguration.
```

Result:

```text
Verified successfully.
```

---

## 16. Multi-Area OSPF Neighbor Verification

Run on EDGE1:

```cisco
show ip ospf neighbor
show ip ospf interface brief
```

Expected result:

```text
Neighbor 2.2.2.2 → CORE1 → FULL
Neighbor 3.3.3.3 → CORE2 → FULL

EDGE1 G0/2 = Area 0
EDGE1 G0/3 = Area 0
```

Result:

```text
Verified successfully.
```

---

## 17. CORE1 ABR Verification

Run on CORE1:

```cisco
show ip ospf
show ip ospf interface brief
show ip ospf neighbor
show ip ospf border-routers
show running-config | section router ospf
```

Expected ABR result:

```text
It is an area border router
Number of areas in this router is 2
Area BACKBONE(0)
Area 10
```

Expected interface placement:

```text
G0/0    = Area 0
Vlan10  = Area 10
Vlan11  = Area 10
Vlan20  = Area 10
Vlan21  = Area 10
Vlan30  = Area 10
Vlan31  = Area 10
Vlan40  = Area 10
Vlan50  = Area 10
Vlan60  = Area 10
Vlan70  = Area 10
```

Expected neighbor:

```text
CORE1 forms OSPF neighbor with EDGE1 only.
Neighbor ID: 1.1.1.1
State: FULL
```

Result:

```text
Verified successfully.
```

---

## 18. CORE2 ABR Verification

Run on CORE2:

```cisco
show ip ospf
show ip ospf interface brief
show ip ospf neighbor
show ip ospf border-routers
show running-config | section router ospf
```

Expected ABR result:

```text
It is an area border router
Number of areas in this router is 2
Area BACKBONE(0)
Area 10
```

Expected interface placement:

```text
G0/0    = Area 0
Vlan10  = Area 10
Vlan11  = Area 10
Vlan20  = Area 10
Vlan21  = Area 10
Vlan30  = Area 10
Vlan31  = Area 10
Vlan40  = Area 10
Vlan50  = Area 10
Vlan60  = Area 10
Vlan70  = Area 10
```

Expected neighbor:

```text
CORE2 forms OSPF neighbor with EDGE1 only.
Neighbor ID: 1.1.1.1
State: FULL
```

Result:

```text
Verified successfully.
```

---

## 19. Multi-Area Passive Interface Verification

Run on CORE1 / CORE2:

```cisco
show ip ospf interface brief
show running-config | section router ospf
```

Expected result:

```text
CORE1 / CORE2:
- G0/0 is non-passive.
- VLAN SVIs are passive.
- VLAN SVIs are advertised into Area 10.
- VLAN SVIs do not form OSPF neighbors.
```

Run on EDGE1:

```cisco
show ip ospf interface brief
show running-config | section router ospf
```

Expected result:

```text
EDGE1:
- G0/2 is non-passive.
- G0/3 is non-passive.
- G0/0 and G0/1 are not in OSPF.
```

Result:

```text
Verified successfully.
```

---

## 20. Inter-Area Route Learning Verification Before Summarization

Before applying route summarization, run on EDGE1:

```cisco
show ip route ospf
```

Expected result:

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

Interpretation:

```text
VLAN routes are no longer intra-area routes from EDGE1's perspective.
They are inter-area routes learned from Area 10 through CORE1/CORE2 ABRs.
```

Result:

```text
Verified successfully.
```

---

## 21. Route Summarization Configuration Verification

Run on CORE1:

```cisco
show running-config | section router ospf
```

Expected result:

```text
router ospf 1
 area 10 range 10.10.0.0 255.255.0.0
```

Run on CORE2:

```cisco
show running-config | section router ospf
```

Expected result:

```text
router ospf 1
 area 10 range 10.10.0.0 255.255.0.0
```

Result:

```text
Verified successfully.
```

---

## 22. Summary Route Verification on EDGE1

After applying summarization, run on EDGE1:

```cisco
show ip route ospf
show ip route 10.10.0.0
show ip ospf database summary
```

Expected route result:

```text
O IA 10.10.0.0/16
```

Expected behavior:

```text
Individual O IA 10.10.x.0/24 routes are no longer shown on EDGE1.
EDGE1 receives one summarized inter-area route: 10.10.0.0/16.
```

Expected OSPF database result:

```text
Type-3 Summary LSA exists.

Link State ID: 10.10.0.0
Network Mask: /16

Advertising Router:
2.2.2.2
3.3.3.3
```

Result:

```text
Verified successfully.
```

---

## 23. Multi-Area OSPF Cost Manipulation Verification

Run on EDGE1:

```cisco
show ip ospf interface brief
show ip route 10.10.0.0
show ip route ospf
```

Expected OSPF cost:

```text
EDGE1 G0/2 toward CORE1: cost 1
EDGE1 G0/3 toward CORE2: cost 50
```

Expected route result:

```text
O IA 10.10.0.0/16 via 172.16.0.2, GigabitEthernet0/2
Metric 2
```

Interpretation:

```text
CORE1 is the preferred internal path.
CORE2 remains available as the backup internal path.
```

Result:

```text
Verified successfully.
```

---

## 24. Multi-Area Default Route Advertisement Verification

## 24.1 EDGE1 Default External LSA Verification

Run on EDGE1:

```cisco
show running-config | section router ospf
show ip ospf database external
```

Expected result:

```text
default-information originate is configured.

Type-5 External LSA exists:
Link State ID: 0.0.0.0
Advertising Router: 1.1.1.1
Network Mask: /0
Metric Type: E2
Metric: 1
```

Result:

```text
Verified successfully.
```

---

## 24.2 CORE1 / CORE2 Default LSA Verification

Run on CORE1 / CORE2:

```cisco
show ip ospf database external
show ip route 0.0.0.0
```

Expected result:

```text
CORE1 and CORE2 receive 0.0.0.0/0 Type-5 External LSA from EDGE1.
```

Observed routing table behavior:

```text
Static default route remains preferred in the RIB if static default route exists.
```

Reason:

```text
Static default route AD = 1
OSPF external route AD = 110
```

Result:

```text
Verified successfully.
```

---

# WAN Resiliency Verification

## 25. Normal WAN State Verification

Run on EDGE1:

```cisco
show track 1
show ip sla summary
show ip route 0.0.0.0
ping 203.0.113.10 source GigabitEthernet0/0
```

Expected result:

```text
Track 1 Up
IP SLA OK
Default route via 100.64.1.1
Ping 203.0.113.10 source G0/0 succeeds
```

Result:

```text
Verified successfully.
```

---

## 26. ISP1 Failure Verification

On EDGE1:

```cisco
conf t
interface GigabitEthernet0/0
 shutdown
end
```

Wait 10 to 15 seconds.

Then run on EDGE1:

```cisco
show track 1
show ip sla summary
show ip route 0.0.0.0
```

Expected result:

```text
Track 1 Down
IP SLA Timeout
Primary default route via 100.64.1.1 removed
Backup default route via 100.64.2.1 installed
```

Result:

```text
Verified successfully.
```

---

## 27. ISP2 Path Verification

During ISP1 failure, run on EDGE1:

```cisco
ping 100.64.2.1
ping 203.0.113.2
ping 203.0.113.10 source GigabitEthernet0/1
```

Expected result:

```text
100.64.2.1 reachable
203.0.113.2 reachable
203.0.113.10 reachable using G0/1 source
```

Result:

```text
Verified successfully.
```

---

## 28. ISP1 Recovery Verification

On EDGE1:

```cisco
conf t
interface GigabitEthernet0/0
 no shutdown
end
```

Wait 10 to 15 seconds.

Then run on EDGE1:

```cisco
show track 1
show ip sla summary
show ip route 0.0.0.0
ping 203.0.113.10 source GigabitEthernet0/0
```

Expected result:

```text
Track 1 Up
IP SLA OK
Default route returns to 100.64.1.1
Ping 203.0.113.10 source G0/0 succeeds
```

Result:

```text
Verified successfully.
```

---

## 29. OSPF State After WAN Failover and Recovery

After WAN failover and recovery, run on EDGE1:

```cisco
show ip ospf neighbor
show ip route 10.10.0.0
show ip route ospf
```

Expected result:

```text
OSPF neighbor with CORE1: FULL
OSPF neighbor with CORE2: FULL

O IA 10.10.0.0/16 via 172.16.0.2, GigabitEthernet0/2

Multi-Area OSPF state remains stable after WAN failover and recovery.
```

Result:

```text
Verified successfully.
```

---

# HSRP Verification

## 30. VLAN40 HSRP Initial State Verification

Run on CORE1 and CORE2:

```cisco
show standby brief
```

Expected initial state:

```text
VLAN40:
CORE1 = Active
CORE2 = Standby
VIP   = 10.10.40.254
```

Result:

```text
Verified successfully.
```

---

## 31. VLAN40 HSRP Failure Verification

On CORE1:

```cisco
conf t
interface Vlan40
 shutdown
end
```

Then verify on CORE2:

```cisco
show standby brief
```

Expected result:

```text
CORE2 becomes Active for VLAN40.
VLAN40 VIP remains 10.10.40.254.
```

Run from MGMT-SRV1:

```bash
ping -c 4 10.10.40.254
```

Expected result:

```text
VLAN40 VIP remains reachable.
```

Result:

```text
HSRP failover verified successfully.
```

Important observation:

```text
Although HSRP failover itself works, route summarization can hide the individual VLAN40 failure from EDGE1.
EDGE1 may continue to prefer the CORE1 summary route for 10.10.0.0/16.
This can cause return traffic blackholing during partial SVI failure.
```

---

## 32. VLAN40 HSRP Recovery Verification

On CORE1:

```cisco
conf t
interface Vlan40
 no shutdown
end
write memory
```

Then verify on CORE1 and CORE2:

```cisco
show standby brief
```

Expected result:

```text
CORE1 returns to Active for VLAN40 due to preempt.
CORE2 returns to Standby.
```

Run from MGMT-SRV1:

```bash
ping -c 4 10.10.40.254
ping -c 4 172.16.0.1
ping -c 4 203.0.113.10
```

Expected result:

```text
10.10.40.254 reachable
172.16.0.1 reachable
203.0.113.10 reachable
```

Result:

```text
Verified successfully.
```

---

# Failure and Recovery Verification

## 33. CORE1 Uplink Failure Verification

On CORE1:

```cisco
conf t
interface GigabitEthernet0/0
 shutdown
end
```

Wait 10 to 20 seconds.

Run on EDGE1:

```cisco
show ip ospf neighbor
show ip route 10.10.0.0
show ip route ospf
```

Expected result:

```text
EDGE1 loses OSPF neighbor with CORE1.
EDGE1 keeps OSPF neighbor with CORE2.

O IA 10.10.0.0/16 moves to:
via 172.16.0.6, GigabitEthernet0/3

Metric becomes 51.
```

Result:

```text
Verified successfully.
```

---

## 34. CORE1 Uplink Recovery Verification

On CORE1:

```cisco
conf t
interface GigabitEthernet0/0
 no shutdown
end
write memory
```

Wait 10 to 20 seconds.

Run on EDGE1:

```cisco
show ip ospf neighbor
show ip route 10.10.0.0
```

Expected result:

```text
EDGE1 restores OSPF neighbor with CORE1.
EDGE1 keeps OSPF neighbor with CORE2.

O IA 10.10.0.0/16 returns to:
via 172.16.0.2, GigabitEthernet0/2

Metric returns to 2.
```

Result:

```text
Verified successfully.
```

---

# NAT/PAT Limitation Verification

## 35. NAT/PAT Baseline Verification

Run on EDGE1:

```cisco
show running-config | include ip nat
show ip nat statistics
show ip nat translations
show running-config | include ^ip route 0.0.0.0
```

Expected NAT role:

```text
NAT inside:
GigabitEthernet0/2
GigabitEthernet0/3

NAT outside:
GigabitEthernet0/0
GigabitEthernet0/1
```

Expected PAT baseline:

```text
ip nat inside source list ENTERPRISE-NAT interface GigabitEthernet0/0 overload
```

Observed behavior:

```text
PAT overload is tied to G0/0 toward ISP1.
```

Result:

```text
Verified successfully.
```

---

## 36. NAT/PAT Failover Limitation

Routing failover to ISP2 was verified.

However:

```text
PAT overload remains tied to G0/0.
```

Therefore:

```text
Routing failover to ISP2 works.
NAT/PAT failover to ISP2 is not complete.
```

Result:

```text
Limitation identified and documented.
```

Final decision:

```text
Full NAT/PAT failover will be revisited in Phase 06 SD-WAN Underlay Enhancement.
```

---

# Final Verification Checklist

## 37. EDGE1 Final Verification

Run on EDGE1:

```cisco
show ip interface brief
show ip ospf neighbor
show ip ospf interface brief
show ip route ospf
show ip route 10.10.0.0
show ip route 0.0.0.0
show track 1
show ip sla summary
show ip ospf database summary
show ip ospf database external
show running-config | section router ospf
show running-config | include ^ip route
show running-config | include ip nat
```

Expected final state:

```text
G0/0 up/up
G0/1 up/up
G0/2 up/up
G0/3 up/up

OSPF neighbor with CORE1: FULL
OSPF neighbor with CORE2: FULL

O IA 10.10.0.0/16 via 172.16.0.2, GigabitEthernet0/2
Default route uses 100.64.1.1 when Track 1 is Up
Track 1 Up
IP SLA OK

Type-3 Summary LSA for 10.10.0.0/16 exists.
Type-5 External LSA for 0.0.0.0/0 exists.
```

---

## 38. CORE1 Final Verification

Run on CORE1:

```cisco
show ip interface brief
show ip ospf
show ip ospf neighbor
show ip ospf interface brief
show ip ospf border-routers
show standby brief
show running-config | section router ospf
```

Expected final state:

```text
CORE1 is an ABR.
Area BACKBONE(0) exists.
Area 10 exists.
G0/0 is in Area 0.
VLAN SVIs are in Area 10.
VLAN SVIs are passive.
OSPF neighbor with EDGE1 is FULL.
VLAN40 is Active.
area 10 range 10.10.0.0 255.255.0.0 exists.
```

---

## 39. CORE2 Final Verification

Run on CORE2:

```cisco
show ip interface brief
show ip ospf
show ip ospf neighbor
show ip ospf interface brief
show ip ospf border-routers
show standby brief
show running-config | section router ospf
```

Expected final state:

```text
CORE2 is an ABR.
Area BACKBONE(0) exists.
Area 10 exists.
G0/0 is in Area 0.
VLAN SVIs are in Area 10.
VLAN SVIs are passive.
OSPF neighbor with EDGE1 is FULL.
VLAN40 is Standby.
area 10 range 10.10.0.0 255.255.0.0 exists.
```

---

## 40. MGMT-SRV1 Final Verification

Run on MGMT-SRV1:

```bash
ip addr show ens2
ip route
ping -c 4 10.10.40.254
ping -c 4 172.16.0.1
ping -c 4 203.0.113.10
dig @10.10.40.10 server1.campus.lab
dig @10.10.40.10 mgmt-srv1.campus.lab
dig @10.10.40.10 internet-srv.campus.lab
```

Expected final state:

```text
ens2 = 10.10.40.10/24
default via 10.10.40.254
VLAN40 gateway reachable
EDGE1 reachable
INTERNET-SRV reachable
campus.lab DNS static records resolve correctly
```

---

# Final Verification Result

## 41. Verified Successfully

The following items were verified successfully in Phase 03:

```text
Phase 02 baseline services remained operational
Single Area OSPF formed correctly
Single Area OSPF neighbors reached FULL state
Single Area passive-interface design worked correctly
EDGE1 learned internal VLAN routes through OSPF
EDGE1 internal static summary routes were removed
EDGE1 advertised default route into OSPF
CORE1/CORE2 received default route LSA from EDGE1
Floating static default route worked
IP SLA worked
Object tracking worked
ISP1 failure triggered failover to ISP2
ISP1 recovery restored primary path
OSPF cost manipulation made CORE1 the preferred internal path
Single Area summarization was intentionally not applied
Multi-Area OSPF formed correctly
CORE1/CORE2 became ABRs
VLAN SVIs were placed in Area 10
EDGE1 learned individual O IA routes before summarization
Route summarization generated O IA 10.10.0.0/16
EDGE1 received Type-3 Summary LSAs for 10.10.0.0/16
OSPF cost manipulation preferred CORE1 summary path
Default route advertisement remained operational
WAN resiliency remained operational after Multi-Area migration
HSRP VLAN40 failover was revalidated
Summarization and HSRP partial failure limitation was identified
CORE1 uplink failure moved summary route to CORE2
CORE1 uplink recovery returned summary route to CORE1
NAT/PAT failover limitation was identified
```

---

## 42. Deferred Item

The following item remains for a later phase:

```text
NAT/PAT Failover:
- Routing failover was verified.
- PAT overload remains tied to ISP1 G0/0.
- Full NAT/PAT failover is deferred to Phase 06 SD-WAN Underlay Enhancement.
```

---

## 43. Phase 03 Verification Status

```text
Phase 03 — Routing Resiliency Ver.2 verification is complete.
```

Next phase:

```text
Phase 04 — Security
```
