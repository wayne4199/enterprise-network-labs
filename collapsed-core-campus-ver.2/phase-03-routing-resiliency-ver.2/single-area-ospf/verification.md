# Phase 03 — Routing Resiliency Ver.2 Verification

## 1. Purpose

This document records verification steps and expected results for **Phase 03 Routing Resiliency Ver.2 — Case 1: Single Area OSPF**.

Phase 03 is divided into two OSPF cases:

```text
Case 1 — Single Area OSPF
Case 2 — Multi-Area OSPF
```

This verification document currently covers **Case 1 — Single Area OSPF** only.

Multi-Area OSPF verification will be added after that case is implemented and validated.

---

## 2. Verification Scope

The Single Area OSPF case verifies the following items:

```text
1. Phase 02 baseline service continuity
2. OSPF Single Area 0
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
13. HSRP failover validation
14. NAT/PAT failover limitation
15. Failure and recovery behavior
```

The following items are intentionally not verified as completed implementations in this Single Area case:

```text
Route Summarization:
- Not implemented in Single Area OSPF
- Deferred to Multi-Area OSPF

NAT/PAT Failover:
- Limitation identified only
- Full implementation deferred to Phase 06 SD-WAN Underlay Enhancement
```

---

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

Important note:

```text
Direct dnsmasq queries using @10.10.40.10 were successful.
If normal dig queries use 127.0.0.53 and fail, check systemd-resolved behavior.
```

---

## 3.4 Phase 03 Pre-Check Backup

Backup path:

```text
/srv/config-backups/phase03-precheck/
```

Expected files:

```text
CORE1-phase03-precheck.cfg
CORE2-phase03-precheck.cfg
EDGE1-phase03-precheck.cfg
```

Verification command:

```bash
ls -l /srv/config-backups/phase03-precheck/
```

Result:

```text
Verified successfully.
```

---

# 4. EDGE1 Pre-OSPF Baseline Verification

## 4.1 EDGE1 Interface Status

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

## 4.2 EDGE1 Static Route Baseline

Run on EDGE1:

```cisco
show ip route
show running-config | include ^ip route
```

Original baseline included:

```text
Default route to ISP1
Floating default route to ISP2
Internal static summary route to CORE1
Backup internal static summary route to CORE2
```

Observed original behavior:

```text
Primary default route:
0.0.0.0/0 → 100.64.1.1

Backup default route:
0.0.0.0/0 → 100.64.2.1 with higher AD

Internal route:
10.10.0.0/16 → 172.16.0.2

Backup internal route:
10.10.0.0/16 → 172.16.0.6 with higher AD
```

Result:

```text
Verified successfully before OSPF migration.
```

---

## 4.3 EDGE1 NAT/PAT Baseline

Run on EDGE1:

```cisco
show running-config | include ip nat
show ip nat translations
show ip nat statistics
```

Expected NAT role:

```text
NAT inside:
G0/2
G0/3

NAT outside:
G0/0
G0/1
```

Expected PAT baseline:

```text
PAT overload uses G0/0 toward ISP1.
```

Result:

```text
Verified successfully.
```

---

# 5. CORE1 / CORE2 Pre-OSPF Baseline Verification

## 5.1 CORE1 Interface and HSRP Status

Run on CORE1:

```cisco
show ip interface brief
show ip route
show running-config | include ^ip route
show standby brief
```

Expected result:

```text
CORE1 G0/0 172.16.0.2 up/up
Default route points to EDGE1 172.16.0.1
HSRP role matches design
```

CORE1 expected HSRP Active VLANs:

```text
VLAN10
VLAN11
VLAN30
VLAN40
VLAN60
```

Result:

```text
Verified successfully.
```

---

## 5.2 CORE2 Interface and HSRP Status

Run on CORE2:

```cisco
show ip interface brief
show ip route
show running-config | include ^ip route
show standby brief
```

Expected result:

```text
CORE2 G0/0 172.16.0.6 up/up
Default route points to EDGE1 172.16.0.5
HSRP role matches design
```

CORE2 expected HSRP Active VLANs:

```text
VLAN20
VLAN21
VLAN31
VLAN50
VLAN70
```

Result:

```text
Verified successfully.
```

---

# 6. End-to-End Baseline Verification

Run on MGMT-SRV1:

```bash
ping -c 4 10.10.40.254
ping -c 4 172.16.0.1
ping -c 4 100.64.1.1
ping -c 4 203.0.113.10
ping -c 4 google.com
```

Expected result:

```text
10.10.40.254    → Reachable
172.16.0.1      → Reachable
100.64.1.1      → Reachable
203.0.113.10    → Reachable
google.com      → Reachable
```

Result:

```text
Verified successfully.
```

---

# 7. OSPF Single Area 0 Verification

## 7.1 EDGE1 OSPF Neighbor Verification

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

## 7.2 CORE1 OSPF Neighbor Verification

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

## 7.3 CORE2 OSPF Neighbor Verification

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

## 7.4 OSPF Passive Interface Verification

Run on CORE1 / CORE2:

```cisco
show ip ospf interface brief
```

Expected result:

```text
G0/0 shows OSPF neighbor count 1/1.
VLAN SVIs show OSPF enabled but no neighbor adjacency.
```

Interpretation:

```text
VLAN SVIs advertise connected networks but do not form neighbors.
```

Result:

```text
Verified successfully.
```

---

# 8. OSPF Internal Route Learning Verification

## 8.1 EDGE1 OSPF Route Verification

Run on EDGE1:

```cisco
show ip route ospf
```

Expected OSPF-learned internal VLAN routes:

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

Initial expected behavior before cost manipulation:

```text
EDGE1 learns VLAN routes through both CORE1 and CORE2 with equal cost.
```

Result:

```text
Verified successfully.
```

---

## 8.2 EDGE1 Internal Static Route Removal Verification

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

# 9. OSPF Default Route Advertisement Verification

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

# 10. Floating Static Route Verification

Run on EDGE1:

```cisco
show running-config | include ^ip route 0.0.0.0
show ip route 0.0.0.0
```

Expected result:

```text
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

# 11. IP SLA and Object Tracking Verification

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

# 12. WAN Resiliency Verification

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

Expected result after return path correction:

```text
100.64.2.1 reachable
203.0.113.2 reachable
203.0.113.10 reachable using G0/1 source
```

Result:

```text
Verified successfully after ISP return path correction.
```

---

## 12.3 Return Path Verification

Return route added on ISP1:

```cisco
ip route 100.64.2.0 255.255.255.252 203.0.113.2
```

Purpose:

```text
Allow return traffic from INTERNET-SRV / ISP1 side back to the ISP2 WAN subnet.
```

Result:

```text
Verified successfully.
```

---

## 12.4 ISP1 Recovery Test

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
ping 203.0.113.10
```

Expected result:

```text
Track 1: Up
Default route returns to 100.64.1.1
INTERNET-SRV reachable through primary path
```

Result:

```text
Verified successfully.
```

---

# 13. OSPF Cost Manipulation Verification

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

# 14. Route Summarization Verification Decision

Route Summarization is not verified as an implemented feature in this Single Area OSPF case.

Reason:

```text
Current OSPF design is Single Area 0.
EDGE1, CORE1, CORE2, and VLAN SVIs are all in Area 0.
CORE1 and CORE2 are not ABRs.
There is no inter-area boundary for OSPF area summarization.
```

Expected Single Area result:

```text
EDGE1 learns individual /24 VLAN routes.
```

Observed routes:

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

Final decision:

```text
Route Summarization will be implemented and verified in the Multi-Area OSPF case.
```

---

# 15. HSRP Failover Verification

## 15.1 Original VLAN40 HSRP State

Run on CORE1 / CORE2:

```cisco
show standby brief
```

Expected original state:

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

## 15.2 VLAN40 HSRP Failure Test

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

Result:

```text
Verified successfully.
```

---

## 15.3 MGMT-SRV1 Reachability During HSRP Failover

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

## 15.4 VLAN40 HSRP Recovery

On CORE1:

```cisco
conf t
interface Vlan40
 no shutdown
end
write memory
```

Then verify on CORE1 / CORE2:

```cisco
show standby brief
```

Expected result:

```text
CORE1 returns to Active for VLAN40 if preempt is configured.
CORE2 returns to Standby.
```

Result:

```text
Verified successfully.
```

---

# 16. CORE1 Uplink Failure Observation

## 16.1 CORE1 Uplink Failure Test

On CORE1:

```cisco
conf t
interface GigabitEthernet0/0
 shutdown
end
```

Observed result:

```text
CORE1 OSPF neighbor with EDGE1 went down.
EDGE1 moved internal routing toward CORE2.
```

This confirms OSPF routed uplink failover behavior.

---

## 16.2 Important HSRP Observation

When CORE1 G0/0 was shut down, HSRP did not automatically fail over if the VLAN SVI remained active.

Observed behavior:

```text
CORE1 could remain HSRP Active for VLAN40.
MGMT-SRV1 could still reach VLAN40 VIP.
MGMT-SRV1 could fail to reach EDGE1 or INTERNET-SRV.
```

Reason:

```text
HSRP does not automatically track routed uplink failure unless tracking is configured.
```

Result:

```text
Observed and documented.
```

Future enhancement:

```text
Consider HSRP interface tracking if routed uplink failure should trigger HSRP role change.
```

---

# 17. NAT/PAT Failover Limitation Verification

## 17.1 Current NAT/PAT Verification

Run on EDGE1:

```cisco
show running-config | include ip nat
show ip nat statistics
show ip nat translations
show running-config | include ^ip route 0.0.0.0
```

Observed NAT/PAT behavior:

```text
G0/0 = NAT outside
G0/1 = NAT outside
G0/2 = NAT inside
G0/3 = NAT inside

PAT overload is tied to G0/0 only.
Inside global address uses 100.64.1.2.
```

Current PAT configuration:

```cisco
ip nat inside source list ENTERPRISE-NAT interface GigabitEthernet0/0 overload
```

---

## 17.2 Limitation

Routing failover to ISP2 works.

However:

```text
PAT overload remains tied to G0/0.
```

Therefore:

```text
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

# 18. Failure and Recovery Test Summary

The following failure and recovery tests were validated or observed during the Single Area OSPF case.

## 18.1 Test 1 — ISP1 Failure and Recovery

Failure:

```text
EDGE1 G0/0 shutdown
```

Expected and observed result:

```text
Track 1 Down
Primary default route removed
Backup route via ISP2 installed
ISP2 path validated after return path correction
```

Recovery:

```text
EDGE1 G0/0 no shutdown
```

Expected and observed recovery:

```text
Track 1 Up
Default route returns to ISP1
Internet reachability restored through primary path
```

Status:

```text
Passed
```

---

## 18.2 Test 2 — CORE1 Uplink Failure and Recovery

Failure:

```text
CORE1 G0/0 shutdown
```

Expected and observed result:

```text
CORE1 OSPF neighbor with EDGE1 goes down
EDGE1 uses CORE2 path for internal VLAN routes
```

Important observation:

```text
HSRP does not automatically fail over unless SVI fails or HSRP tracking is configured.
```

Status:

```text
Passed as routing failover observation.
HSRP tracking enhancement identified.
```

---

## 18.3 Test 3 — HSRP VLAN40 Failure and Recovery

Failure:

```text
CORE1 Vlan40 shutdown
```

Expected and observed result:

```text
CORE2 becomes Active for VLAN40
VLAN40 VIP remains reachable
MGMT-SRV1 maintains connectivity
```

Recovery:

```text
CORE1 Vlan40 no shutdown
```

Expected and observed recovery:

```text
CORE1 returns to Active if preempt is configured
CORE2 returns to Standby
```

Status:

```text
Passed
```

---

# 19. Final Single Area OSPF Verification Checklist

## 19.1 EDGE1 Final Verification

Run on EDGE1:

```cisco
show ip interface brief
show ip ospf neighbor
show ip ospf interface brief
show ip route ospf
show ip route 0.0.0.0
show track 1
show ip sla summary
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

Default route uses 100.64.1.1 when Track 1 is Up.
Backup default route via 100.64.2.1 remains available.

Internal VLAN routes are learned through OSPF.
CORE1 path is preferred due to OSPF cost manipulation.
```

---

## 19.2 CORE1 Final Verification

Run on CORE1:

```cisco
show ip interface brief
show ip ospf neighbor
show ip ospf interface brief
show ip route ospf
show standby brief
show running-config | section router ospf
```

Expected final state:

```text
G0/0 up/up
OSPF neighbor with EDGE1: FULL
VLAN SVIs are passive in OSPF
HSRP role matches design
```

---

## 19.3 CORE2 Final Verification

Run on CORE2:

```cisco
show ip interface brief
show ip ospf neighbor
show ip ospf interface brief
show ip route ospf
show standby brief
show running-config | section router ospf
```

Expected final state:

```text
G0/0 up/up
OSPF neighbor with EDGE1: FULL
VLAN SVIs are passive in OSPF
HSRP role matches design
```

---

## 19.4 MGMT-SRV1 Final Verification

Run on MGMT-SRV1:

```bash
ping -c 4 10.10.40.254
ping -c 4 172.16.0.1
ping -c 4 203.0.113.10
dig @10.10.40.10 server1.campus.lab
dig @10.10.40.10 mgmt-srv1.campus.lab
dig @10.10.40.10 internet-srv.campus.lab
```

Expected final state:

```text
VLAN40 gateway reachable
EDGE1 reachable
INTERNET-SRV reachable
campus.lab DNS static records resolve correctly
```

---

# 20. Final Verification Result

The following items were verified successfully in the Single Area OSPF case:

```text
Phase 02 baseline services remained operational
OSPF Single Area 0 formed correctly
OSPF neighbors reached FULL state
OSPF passive-interface design worked correctly
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
HSRP VLAN40 failover worked
Failure and recovery behavior was validated
NAT/PAT failover limitation was identified
```

The following items remain for later phases or cases:

```text
Route Summarization:
- To be implemented and verified in Multi-Area OSPF case

NAT/PAT Failover:
- To be implemented and verified in Phase 06 SD-WAN Underlay Enhancement
```

Single Area OSPF verification is complete.

Next step:

```text
Proceed to Multi-Area OSPF case.
```
