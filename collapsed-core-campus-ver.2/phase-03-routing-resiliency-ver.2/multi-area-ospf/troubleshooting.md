# Phase 03 — Routing Resiliency Ver.2 Troubleshooting

## 1. Purpose

This document records troubleshooting notes for **Phase 03 — Routing Resiliency Ver.2**.

Phase 03 includes two OSPF cases:

```text
Case 1 — Single Area OSPF
Case 2 — Multi-Area OSPF
```

This document focuses on routing resiliency, OSPF behavior, route summarization, WAN failover, HSRP interaction, and failure recovery troubleshooting.

The following recovery items are intentionally excluded from this document:

```text
- VLAN Database loss recovery
- MGMT-SRV1 netplan loss recovery
- INTERNET-SRV IP/Gateway loss recovery
```

---

## 2. Troubleshooting Scope

This document covers troubleshooting for:

```text
1. OSPF pre-check
2. OSPF neighbor formation
3. OSPF passive-interface behavior
4. Single Area OSPF route learning
5. EDGE1 internal static route removal
6. OSPF default route advertisement
7. IP SLA and object tracking
8. Floating static route failover
9. IP SLA host route behavior
10. Multi-Area OSPF ABR validation
11. Inter-area route learning
12. Route summarization
13. OSPF cost manipulation
14. WAN resiliency
15. HSRP failover behavior
16. Summarization and HSRP partial failure limitation
17. CORE1 uplink failure and recovery
18. NAT/PAT failover limitation
```

---

# General Troubleshooting Baseline

## 3. Routing Protocol Pre-Check

Before applying or changing OSPF configuration, always verify whether an existing routing protocol configuration already exists.

Run on CORE1, CORE2, and EDGE1:

```cisco
show ip protocols
show running-config | section router ospf
show running-config | section router eigrp
show running-config | section router bgp
show ip ospf neighbor
show ip route ospf
```

Reason:

```text
Do not assume that the routing protocol configuration is empty.
Existing OSPF passive-interface or network statements can affect new configuration.
```

Expected clean state before reconfiguration:

```text
No unintended OSPF neighbors
No unexpected OSPF routes
No unintended passive-interface exceptions
No old routing protocol configuration that conflicts with the new design
```

---

## 4. OSPF Interface and Link Baseline Check

Before troubleshooting OSPF neighbors, confirm the routed links are up.

### EDGE1

```cisco
show ip interface brief
```

Expected:

```text
G0/0 100.64.1.2  up/up  → ISP1
G0/1 100.64.2.2  up/up  → ISP2
G0/2 172.16.0.1  up/up  → CORE1
G0/3 172.16.0.5  up/up  → CORE2
```

### CORE1

```cisco
show ip interface brief
```

Expected:

```text
G0/0 172.16.0.2 up/up → EDGE1
```

### CORE2

```cisco
show ip interface brief
```

Expected:

```text
G0/0 172.16.0.6 up/up → EDGE1
```

If a routed interface is administratively down, restore it:

```cisco
conf t
interface <interface-id>
 no shutdown
end
```

---

# Case 1 — Single Area OSPF Troubleshooting

## 5. Issue: OSPF Neighbor Does Not Form

### Symptom

EDGE1 shows OSPF-enabled interfaces, but no neighbors.

```cisco
show ip ospf interface brief
show ip ospf neighbor
```

Observed symptom:

```text
EDGE1 G0/2 is in OSPF Area 0.
EDGE1 G0/3 is in OSPF Area 0.
Neighbor count is 0.
```

### Possible Causes

```text
1. CORE1 / CORE2 OSPF is not configured.
2. CORE1 / CORE2 G0/0 is not included in OSPF.
3. CORE1 / CORE2 G0/0 is passive.
4. Area mismatch.
5. Network statement mismatch.
6. Interface is down/down or administratively down.
```

### Troubleshooting Commands

Run on EDGE1:

```cisco
show ip ospf neighbor
show ip ospf interface brief
show running-config | section router ospf
show ip interface brief
```

Run on CORE1 and CORE2:

```cisco
show ip ospf neighbor
show ip ospf interface brief
show running-config | section router ospf
show ip interface brief
```

### Correct Single Area OSPF Passive Interface Design

CORE1:

```cisco
router ospf 1
 passive-interface default
 no passive-interface GigabitEthernet0/0
```

CORE2:

```cisco
router ospf 1
 passive-interface default
 no passive-interface GigabitEthernet0/0
```

EDGE1:

```cisco
router ospf 1
 passive-interface default
 no passive-interface GigabitEthernet0/2
 no passive-interface GigabitEthernet0/3
```

### Fix

CORE1:

```cisco
conf t
router ospf 1
 passive-interface default
 no passive-interface GigabitEthernet0/0
end
write memory
```

CORE2:

```cisco
conf t
router ospf 1
 passive-interface default
 no passive-interface GigabitEthernet0/0
end
write memory
```

EDGE1:

```cisco
conf t
router ospf 1
 passive-interface default
 no passive-interface GigabitEthernet0/2
 no passive-interface GigabitEthernet0/3
end
write memory
```

### Expected Result

On EDGE1:

```text
Neighbor 2.2.2.2 → FULL
Neighbor 3.3.3.3 → FULL
```

---

## 6. Issue: OSPF Neighbors Form on VLAN SVIs

### Symptom

CORE1 and CORE2 form OSPF neighbors over VLAN SVIs instead of only the routed links to EDGE1.

Possible output:

```text
OSPF neighbors appear on VLAN10, VLAN20, VLAN40, or other SVIs.
```

### Cause

VLAN SVIs are not passive in OSPF.

This can happen if `passive-interface default` is missing or if the passive-interface configuration is inconsistent.

### Correct Design

```text
CORE1 / CORE2 G0/0:
- Non-passive
- OSPF neighbor allowed

CORE1 / CORE2 VLAN SVIs:
- Passive
- Advertise connected networks
- Do not form neighbors
```

### Fix

CORE1:

```cisco
conf t
router ospf 1
 passive-interface default
 no passive-interface GigabitEthernet0/0
end
write memory
```

CORE2:

```cisco
conf t
router ospf 1
 passive-interface default
 no passive-interface GigabitEthernet0/0
end
write memory
```

### Verification

```cisco
show ip ospf neighbor
show ip ospf interface brief
```

Expected result:

```text
CORE1 neighbor: EDGE1 only
CORE2 neighbor: EDGE1 only
VLAN SVIs: passive, no neighbor
```

---

## 7. Issue: OSPF Routes Do Not Appear on EDGE1

### Symptom

OSPF neighbors are FULL, but EDGE1 does not show internal VLAN routes.

```cisco
show ip route ospf
```

Expected internal VLAN routes in Single Area OSPF:

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

### Possible Causes

```text
1. CORE1 / CORE2 do not advertise 10.10.0.0/16 into OSPF.
2. VLAN SVIs are not included in OSPF.
3. VLAN SVIs are down.
4. OSPF process has incorrect network statements.
5. Old static routes are confusing verification.
```

### Troubleshooting Commands

Run on CORE1 / CORE2:

```cisco
show running-config | section router ospf
show ip ospf interface brief
show ip interface brief
show ip route connected
```

Run on EDGE1:

```cisco
show ip route ospf
show ip ospf database
show ip protocols
```

### Correct CORE1 Single Area OSPF Network Statement

```cisco
router ospf 1
 network 172.16.0.0 0.0.0.3 area 0
 network 10.10.0.0 0.0.255.255 area 0
```

### Correct CORE2 Single Area OSPF Network Statement

```cisco
router ospf 1
 network 172.16.0.4 0.0.0.3 area 0
 network 10.10.0.0 0.0.255.255 area 0
```

---

## 8. Issue: Static Routes Still Exist After OSPF Migration

### Symptom

EDGE1 learns OSPF routes, but old internal static routes still exist.

Check:

```cisco
show running-config | include ^ip route 10.10
show ip route 10.10.40.0
show ip route ospf
```

### Important Routing Rule

Routers use longest prefix match before administrative distance.

Example:

```text
Destination 10.10.40.10:
- 10.10.40.0/24 OSPF route is more specific
- 10.10.0.0/16 static summary is less specific
```

However, after OSPF is validated, keeping old internal static summary routes can confuse troubleshooting and route behavior.

### Fix

Remove old internal static summary routes from EDGE1:

```cisco
conf t
no ip route 10.10.0.0 255.255.0.0 172.16.0.2
no ip route 10.10.0.0 255.255.0.0 172.16.0.6 10
end
write memory
```

### Verification

```cisco
show running-config | include ^ip route 10.10
show ip route ospf
show ip route 10.10.40.0
```

Expected result:

```text
No 10.10.0.0/16 static route remains on EDGE1.
Internal VLAN routes are learned through OSPF.
```

---

## 9. Issue: OSPF Default Route Advertisement Is Not Installed in the Routing Table

### Symptom

EDGE1 advertises the default route into OSPF, but CORE1/CORE2 routing tables still show static default routes.

Check:

```cisco
show ip ospf database external
show ip route 0.0.0.0
```

Observed behavior:

```text
OSPF external LSA for 0.0.0.0 exists.
Routing table still uses static default route.
```

### Cause

Static default route has lower administrative distance than OSPF external route.

```text
Static AD = 1
OSPF External AD = 110
```

Therefore, static default route remains preferred in the routing table.

### Expected Result

This is expected if static default routes still exist on CORE1/CORE2.

The important validation point is:

```text
CORE1 and CORE2 receive the OSPF Type-5 External LSA for 0.0.0.0/0 from EDGE1.
```

### Verification

On EDGE1:

```cisco
show ip ospf database external
```

On CORE1 / CORE2:

```cisco
show ip ospf database external
show ip route 0.0.0.0
```

Expected OSPF database entry:

```text
Link State ID: 0.0.0.0
Advertising Router: 1.1.1.1
Network Mask: /0
Metric Type: E2
Metric: 1
```

---

## 10. Issue: IP SLA Timeout or Track 1 Down

### Symptom

EDGE1 shows Track 1 Down.

```cisco
show track 1
show ip sla summary
```

Possible output:

```text
Track 1 Down
IP SLA Timeout
Default route uses 100.64.2.1
```

### Possible Causes

```text
1. ISP1 path is actually down.
2. EDGE1 G0/0 is administratively down.
3. IP SLA target 203.0.113.10 is unreachable through ISP1.
4. IP SLA probe follows the backup route after failover.
5. Missing host route for the IP SLA target.
```

### Verification

On EDGE1:

```cisco
show ip interface brief
show ip route 203.0.113.10
show ip route 0.0.0.0
ping 100.64.1.1
ping 203.0.113.1
ping 203.0.113.10 source GigabitEthernet0/0
show track 1
show ip sla summary
```

### Recommended Stabilization

Add a host route for the IP SLA target through ISP1:

```cisco
conf t
ip route 203.0.113.10 255.255.255.255 100.64.1.1
end
write memory
```

### Reason

The IP SLA operation uses:

```text
icmp-echo 203.0.113.10 source-interface GigabitEthernet0/0
```

If the default route has already moved to ISP2, the IP SLA probe can behave unexpectedly.

The host route ensures that the IP SLA target is checked through the ISP1 path.

### Expected Result

```text
Track 1 Up
IP SLA OK
Default route via 100.64.1.1
Ping 203.0.113.10 source G0/0 succeeds
```

---

## 11. Issue: IP SLA Cannot Be Modified After It Starts

### Symptom

When trying to modify an IP SLA operation after it has started, IOS may show:

```text
Entry already running and cannot be modified
only can delete (no) and start over
```

### Cause

An IP SLA operation that is already scheduled and running cannot be directly modified.

### Option 1 — Keep Current IP SLA

Recommended if the current IP SLA is already working.

The primary learning goal is:

```text
IP SLA reachability monitoring
Object tracking
Default route failover
```

### Option 2 — Delete and Recreate IP SLA

```cisco
conf t
no ip sla 1
ip sla 1
 icmp-echo 203.0.113.10 source-interface GigabitEthernet0/0
 frequency 5
 threshold 1000
exit
ip sla schedule 1 life forever start-time now
track 1 ip sla 1 reachability
end
write memory
```

### Verification

```cisco
show ip sla summary
show ip sla statistics 1
show track 1
```

---

## 12. Issue: Primary Default Route Uses Wrong Next-Hop

### Symptom

The primary default route does not install even though Track 1 is Up.

Routing table shows backup path:

```text
Gateway of last resort → 100.64.2.1
```

Running configuration may show a wrong next-hop:

```cisco
ip route 0.0.0.0 0.0.0.0 100.61.1.1 track 1
```

### Cause

Typo in the ISP1 next-hop address.

Wrong:

```text
100.61.1.1
```

Correct:

```text
100.64.1.1
```

### Fix

```cisco
conf t
no ip route 0.0.0.0 0.0.0.0 100.61.1.1 track 1
ip route 0.0.0.0 0.0.0.0 100.64.1.1 track 1
end
write memory
```

### Verification

```cisco
show track 1
show ip route 0.0.0.0
show running-config | include ^ip route 0.0.0.0
```

Expected result:

```text
Track 1: Up
Default route: 100.64.1.1
Backup route: 100.64.2.1 AD 10
```

---

## 13. Issue: ISP1 Failure Switches Default Route to ISP2 but End-to-End Ping Fails

### Symptom

After shutting down EDGE1 G0/0:

```cisco
interface GigabitEthernet0/0
 shutdown
```

EDGE1 shows:

```text
Track 1 Down
Default route via 100.64.2.1
```

But ping to INTERNET-SRV may fail:

```cisco
ping 203.0.113.10
```

### Important Observation

This does not automatically mean IP SLA failover failed.

Routing failover can work while end-to-end reachability fails due to return-path issues.

### Verification

Run on EDGE1 during failure state:

```cisco
ping 100.64.2.1
ping 203.0.113.2
ping 203.0.113.10 source GigabitEthernet0/1
```

Expected result:

```text
EDGE1 → ISP2 next-hop 100.64.2.1: Success
EDGE1 → ISP2 Internet-side 203.0.113.2: Success
EDGE1 → INTERNET-SRV 203.0.113.10 source G0/1: Success
```

### Return Path Requirement

If the Internet segment uses ISP1 as the normal gateway, ISP1 must know how to reach the ISP2 WAN subnet.

Fix on ISP1:

```cisco
conf t
ip route 100.64.2.0 255.255.255.252 203.0.113.2
end
write memory
```

### Verification

On EDGE1 during ISP1 failure:

```cisco
ping 203.0.113.10 source GigabitEthernet0/1
```

Expected result:

```text
Ping succeeds through ISP2 after return path correction.
```

---

# Case 2 — Multi-Area OSPF Troubleshooting

## 14. Issue: Multi-Area OSPF Neighbor Does Not Form

### Symptom

After reconfiguring Multi-Area OSPF, EDGE1 does not form neighbors with CORE1 or CORE2.

```cisco
show ip ospf neighbor
```

Expected neighbors:

```text
2.2.2.2 FULL
3.3.3.3 FULL
```

### Possible Causes

```text
1. Area mismatch on EDGE-to-CORE routed links.
2. Passive interface still applied to EDGE-facing interface.
3. Incorrect network statement.
4. Routed interface down/down.
5. OSPF process not configured on one side.
```

### Correct Area Placement

```text
EDGE1 G0/2 ↔ CORE1 G0/0 = Area 0
EDGE1 G0/3 ↔ CORE2 G0/0 = Area 0
```

### Verification

On EDGE1:

```cisco
show running-config | section router ospf
show ip ospf interface brief
show ip ospf neighbor
```

On CORE1:

```cisco
show running-config | section router ospf
show ip ospf interface brief
show ip ospf neighbor
```

On CORE2:

```cisco
show running-config | section router ospf
show ip ospf interface brief
show ip ospf neighbor
```

### Fix Example

EDGE1:

```cisco
conf t
router ospf 1
 passive-interface default
 no passive-interface GigabitEthernet0/2
 no passive-interface GigabitEthernet0/3
 network 172.16.0.0 0.0.0.3 area 0
 network 172.16.0.4 0.0.0.3 area 0
end
write memory
```

CORE1:

```cisco
conf t
router ospf 1
 passive-interface default
 no passive-interface GigabitEthernet0/0
 network 172.16.0.0 0.0.0.3 area 0
end
write memory
```

CORE2:

```cisco
conf t
router ospf 1
 passive-interface default
 no passive-interface GigabitEthernet0/0
 network 172.16.0.4 0.0.0.3 area 0
end
write memory
```

---

## 15. Issue: CORE1 / CORE2 Are Not ABRs

### Symptom

CORE1 or CORE2 does not show ABR status.

```cisco
show ip ospf
```

Expected output:

```text
It is an area border router
Number of areas in this router is 2
Area BACKBONE(0)
Area 10
```

### Cause

The router must have at least one active OSPF interface in Area 0 and at least one active OSPF interface in Area 10.

Possible causes:

```text
1. VLAN SVI networks are still in Area 0.
2. VLAN SVI networks are not included in OSPF.
3. EDGE-facing routed link is not in Area 0.
4. VLAN SVI interfaces are down.
```

### Verification

On CORE1 / CORE2:

```cisco
show ip ospf
show ip ospf interface brief
show running-config | section router ospf
```

Expected interface placement:

```text
G0/0 = Area 0
VLAN SVIs = Area 10
```

### Fix Example for CORE1

```cisco
conf t
router ospf 1
 network 172.16.0.0 0.0.0.3 area 0
 network 10.10.10.0 0.0.0.255 area 10
 network 10.10.11.0 0.0.0.255 area 10
 network 10.10.20.0 0.0.0.255 area 10
 network 10.10.21.0 0.0.0.255 area 10
 network 10.10.30.0 0.0.0.255 area 10
 network 10.10.31.0 0.0.0.255 area 10
 network 10.10.40.0 0.0.0.255 area 10
 network 10.10.50.0 0.0.0.255 area 10
 network 10.10.60.0 0.0.0.255 area 10
 network 10.10.70.0 0.0.0.255 area 10
end
write memory
```

### Fix Example for CORE2

```cisco
conf t
router ospf 1
 network 172.16.0.4 0.0.0.3 area 0
 network 10.10.10.0 0.0.0.255 area 10
 network 10.10.11.0 0.0.0.255 area 10
 network 10.10.20.0 0.0.0.255 area 10
 network 10.10.21.0 0.0.0.255 area 10
 network 10.10.30.0 0.0.0.255 area 10
 network 10.10.31.0 0.0.0.255 area 10
 network 10.10.40.0 0.0.0.255 area 10
 network 10.10.50.0 0.0.0.255 area 10
 network 10.10.60.0 0.0.0.255 area 10
 network 10.10.70.0 0.0.0.255 area 10
end
write memory
```

---

## 16. Issue: EDGE1 Does Not See O IA Routes Before Summarization

### Symptom

After Multi-Area OSPF configuration, EDGE1 does not show individual `O IA` VLAN routes.

```cisco
show ip route ospf
```

Expected pre-summarization result:

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

### Possible Causes

```text
1. VLAN SVIs are not in Area 10.
2. CORE1 / CORE2 are not ABRs.
3. VLAN SVI network statements are missing.
4. Summarization was already applied.
5. OSPF neighbor relationship is not established.
```

### Verification

On EDGE1:

```cisco
show ip ospf neighbor
show ip route ospf
show ip ospf database summary
```

On CORE1 / CORE2:

```cisco
show ip ospf
show ip ospf interface brief
show running-config | section router ospf
```

### Interpretation

```text
If individual O IA routes appear:
- Multi-Area route learning is working before summarization.

If only O IA 10.10.0.0/16 appears:
- Summarization may already be active.

If no O IA routes appear:
- Check ABR status, Area 10 network statements, and OSPF neighbors.
```

---

## 17. Issue: Route Summarization Does Not Appear on EDGE1

### Symptom

After applying `area 10 range`, EDGE1 still shows individual /24 routes instead of the summary route.

Expected result:

```text
O IA 10.10.0.0/16
```

But EDGE1 still shows:

```text
O IA 10.10.10.0/24
O IA 10.10.11.0/24
...
O IA 10.10.70.0/24
```

### Possible Causes

```text
1. area 10 range was not configured on CORE1/CORE2.
2. The wrong area ID was used.
3. CORE1/CORE2 are not acting as ABRs.
4. The summary does not cover the internal VLAN routes.
5. OSPF process has not converged yet.
```

### Correct Configuration

CORE1:

```cisco
conf t
router ospf 1
 area 10 range 10.10.0.0 255.255.0.0
end
write memory
```

CORE2:

```cisco
conf t
router ospf 1
 area 10 range 10.10.0.0 255.255.0.0
end
write memory
```

### Verification on CORE1 / CORE2

```cisco
show running-config | section router ospf
show ip ospf
```

Expected:

```text
area 10 range 10.10.0.0 255.255.0.0
It is an area border router
```

### Verification on EDGE1

```cisco
show ip route ospf
show ip route 10.10.0.0
show ip ospf database summary
```

Expected:

```text
O IA 10.10.0.0/16
Type-3 Summary LSA for 10.10.0.0/16
Advertising Router: 2.2.2.2 and/or 3.3.3.3
```

---

## 18. Issue: Summary Route Uses ECMP Through CORE1 and CORE2

### Symptom

After summarization, EDGE1 shows two equal paths for the summary route.

```text
O IA 10.10.0.0/16 via 172.16.0.2, G0/2
O IA 10.10.0.0/16 via 172.16.0.6, G0/3
```

### Cause

CORE1 and CORE2 advertise the same summary route with equal cost.

This creates ECMP.

### Design Decision

The lab design prefers deterministic primary/backup behavior instead of ECMP.

```text
CORE1 = Primary internal path
CORE2 = Backup internal path
```

### Fix

Apply OSPF cost on EDGE1 G0/3 toward CORE2:

```cisco
conf t
interface GigabitEthernet0/3
 ip ospf cost 50
end
write memory
```

### Verification

```cisco
show ip ospf interface brief
show ip route 10.10.0.0
show ip route ospf
```

Expected result:

```text
EDGE1 G0/2 cost 1
EDGE1 G0/3 cost 50

O IA 10.10.0.0/16 via 172.16.0.2, GigabitEthernet0/2
Metric 2
```

---

## 19. Issue: CORE1 Uplink Failure Does Not Move Summary Route to CORE2

### Symptom

After shutting down CORE1 G0/0, EDGE1 still tries to use CORE1 or does not install the CORE2 summary route.

CORE1 failure command:

```cisco
conf t
interface GigabitEthernet0/0
 shutdown
end
```

Expected result on EDGE1:

```text
Neighbor 2.2.2.2 disappears.
Neighbor 3.3.3.3 remains FULL.
O IA 10.10.0.0/16 moves to 172.16.0.6 via G0/3.
```

### Possible Causes

```text
1. CORE2 is not advertising the summary route.
2. CORE2 is not an ABR.
3. CORE2 OSPF neighbor with EDGE1 is down.
4. area 10 range is missing on CORE2.
5. EDGE1 G0/3 or CORE2 G0/0 is down.
```

### Verification

On EDGE1:

```cisco
show ip ospf neighbor
show ip route 10.10.0.0
show ip ospf database summary
```

On CORE2:

```cisco
show ip ospf
show ip ospf neighbor
show running-config | section router ospf
```

### Expected Failure State

```text
EDGE1 OSPF neighbor:
- 2.2.2.2 missing
- 3.3.3.3 FULL

EDGE1 route:
O IA 10.10.0.0/16 via 172.16.0.6, GigabitEthernet0/3

Metric:
51
```

### Recovery

On CORE1:

```cisco
conf t
interface GigabitEthernet0/0
 no shutdown
end
write memory
```

Expected recovery on EDGE1:

```text
Neighbor 2.2.2.2 returns to FULL.
Summary route returns to 172.16.0.2 via G0/2.
Metric returns to 2.
```

---

## 20. Issue: HSRP Failover Works but External Reachability Fails

### Symptom

During VLAN40 HSRP failover:

```cisco
CORE1:
interface Vlan40
 shutdown
```

CORE2 becomes Active for VLAN40.

```text
CORE2 Vlan40 = Active
VIP 10.10.40.254 remains available
```

However, MGMT-SRV1 may fail to reach EDGE1 or INTERNET-SRV:

```bash
ping -c 4 10.10.40.254
ping -c 4 172.16.0.1
ping -c 4 203.0.113.10
```

Possible result:

```text
10.10.40.254 reachable
172.16.0.1 unreachable
203.0.113.10 unreachable
```

### Cause

This is not necessarily an HSRP failure.

In Multi-Area OSPF with summarization, EDGE1 only sees:

```text
O IA 10.10.0.0/16
```

If EDGE1 prefers the summary route through CORE1:

```text
O IA 10.10.0.0/16 via 172.16.0.2
```

then return traffic for VLAN40 can still be sent toward CORE1.

If CORE1 Vlan40 is down:

```text
CORE1 cannot forward return traffic into VLAN40.
```

This creates return-path blackholing.

### Traffic Flow Example

```text
MGMT-SRV1 → CORE2 Active HSRP gateway → EDGE1
EDGE1 → 10.10.40.10 return traffic → CORE1 summary path
CORE1 Vlan40 is down
Return traffic is blackholed
```

### Design Lesson

```text
HSRP failover and OSPF route summarization are separate mechanisms.

HSRP can correctly move the default gateway role to CORE2,
while OSPF summarization can still cause upstream return traffic to prefer CORE1.

Route summarization can hide individual VLAN failures from upstream routers.
```

### Immediate Recovery

Restore CORE1 Vlan40:

```cisco
conf t
interface Vlan40
 no shutdown
end
write memory
```

### Verification

On CORE1 / CORE2:

```cisco
show standby brief
```

On EDGE1:

```cisco
show ip route 10.10.0.0
show ip ospf neighbor
```

From MGMT-SRV1:

```bash
ping -c 4 10.10.40.254
ping -c 4 172.16.0.1
ping -c 4 203.0.113.10
```

### Future Design Considerations

```text
- HSRP tracking
- Object tracking
- Conditional route advertisement
- More specific route handling for critical VLANs
- Careful summarization boundary design
- Alignment between gateway state and routing advertisement
```

This is documented as a design limitation, not as a lab failure.

---

## 21. Issue: HSRP Does Not Fail Over When CORE1 G0/0 Is Shut Down

### Symptom

CORE1 G0/0 is shut down:

```cisco
conf t
interface GigabitEthernet0/0
 shutdown
end
```

Observed behavior:

```text
CORE1 loses OSPF adjacency with EDGE1.
EDGE1 moves the summary route toward CORE2.
CORE1 may still remain HSRP Active for VLAN40.
```

### Cause

HSRP does not automatically track CORE1 G0/0 unless interface tracking is configured.

The VLAN40 SVI remains up, so CORE1 can remain Active for VLAN40.

### Important Design Lesson

This is a key difference between:

```text
Routing uplink failure
HSRP gateway failover
```

A routed uplink failure does not automatically trigger HSRP failover.

### Verification

On EDGE1:

```cisco
show ip ospf neighbor
show ip route 10.10.0.0
```

On CORE1 / CORE2:

```cisco
show standby brief
```

### Future Enhancement

If uplink failure must trigger HSRP failover, configure HSRP tracking in a future enhancement.

Example concept:

```text
Track CORE uplink
Reduce HSRP priority when uplink fails
Allow peer core switch to become Active
```

---

## 22. Issue: OSPF Cost Manipulation Does Not Prefer CORE1

### Symptom

After setting OSPF cost on EDGE1 G0/3, EDGE1 still shows ECMP toward CORE1 and CORE2.

Expected:

```text
10.10.0.0/16 should prefer 172.16.0.2 through G0/2.
```

### Possible Causes

```text
1. OSPF cost was configured on the wrong device.
2. OSPF cost was configured on CORE1 instead of EDGE1.
3. OSPF cost was configured on the wrong interface.
4. OSPF did not reconverge yet.
```

### Correct Device and Interface

The cost must be configured on:

```text
EDGE1 GigabitEthernet0/3
```

Correct configuration:

```cisco
conf t
interface GigabitEthernet0/3
 ip ospf cost 50
end
write memory
```

### Verification

```cisco
show ip ospf interface brief
show ip route 10.10.0.0
show running-config interface GigabitEthernet0/3
```

Expected result:

```text
EDGE1 G0/2 cost 1
EDGE1 G0/3 cost 50
O IA 10.10.0.0/16 via 172.16.0.2, G0/2
```

---

## 23. NAT/PAT Failover Limitation

### Symptom

WAN routing fails over to ISP2, but internal user NAT/PAT behavior is not fully redundant.

### Current NAT/PAT Design

EDGE1 has both ISP-facing interfaces as NAT outside:

```text
G0/0 = NAT outside
G0/1 = NAT outside
```

Internal interfaces are NAT inside:

```text
G0/2 = NAT inside
G0/3 = NAT inside
```

However, PAT overload uses only G0/0:

```cisco
ip nat inside source list ENTERPRISE-NAT interface GigabitEthernet0/0 overload
```

### Cause

Default route can move to ISP2, but NAT overload remains tied to ISP1 G0/0.

Therefore:

```text
Routing failover works.
NAT/PAT failover is not complete.
```

### Verification Commands

On EDGE1:

```cisco
show running-config | include ip nat
show ip nat statistics
show ip nat translations
show running-config | include ^ip route 0.0.0.0
```

### Phase 03 Decision

Do not fully implement NAT/PAT failover in Phase 03.

Document the limitation only.

### Carry-over to Phase 06

NAT/PAT failover must be revisited in Phase 06 SD-WAN Underlay Enhancement with:

```text
- Route-map based NAT
- ISP-specific NAT policy
- PBR + IP SLA
- DIA
- Local Internet Breakout
- Business intent path selection
```

---

# Final Recovery Checklist

## 24. EDGE1 Final State

Run on EDGE1:

```cisco
show ip interface brief
show track 1
show ip sla summary
show ip route 0.0.0.0
show ip ospf neighbor
show ip route 10.10.0.0
show ip ospf database summary
show ip ospf database external
```

Expected state:

```text
G0/0 up/up
G0/1 up/up
G0/2 up/up
G0/3 up/up

Track 1 Up
IP SLA OK
Default route via 100.64.1.1

OSPF neighbors with CORE1 and CORE2 are FULL
O IA 10.10.0.0/16 via 172.16.0.2
Default external LSA exists
```

---

## 25. CORE1 Final State

Run on CORE1:

```cisco
show ip interface brief
show ip ospf
show ip ospf neighbor
show ip ospf interface brief
show standby brief
show running-config | section router ospf
```

Expected state:

```text
CORE1 is an ABR.
Area 0 and Area 10 exist.
G0/0 is in Area 0.
VLAN SVIs are in Area 10 and passive.
OSPF neighbor with EDGE1 is FULL.
VLAN40 is Active.
area 10 range 10.10.0.0 255.255.0.0 exists.
```

---

## 26. CORE2 Final State

Run on CORE2:

```cisco
show ip interface brief
show ip ospf
show ip ospf neighbor
show ip ospf interface brief
show standby brief
show running-config | section router ospf
```

Expected state:

```text
CORE2 is an ABR.
Area 0 and Area 10 exist.
G0/0 is in Area 0.
VLAN SVIs are in Area 10 and passive.
OSPF neighbor with EDGE1 is FULL.
VLAN40 is Standby.
area 10 range 10.10.0.0 255.255.0.0 exists.
```

---

## 27. MGMT-SRV1 Final Connectivity Check

Run on MGMT-SRV1:

```bash
ping -c 4 10.10.40.254
ping -c 4 172.16.0.1
ping -c 4 203.0.113.10
dig @10.10.40.10 server1.campus.lab
dig @10.10.40.10 mgmt-srv1.campus.lab
dig @10.10.40.10 internet-srv.campus.lab
```

Expected result:

```text
VLAN40 gateway reachable
EDGE1 reachable
INTERNET-SRV reachable
campus.lab DNS static records resolve correctly
```

---

## 28. Save Configuration After Recovery

After restoring the lab to normal state, save all Cisco device configurations.

CORE1:

```cisco
write memory
```

CORE2:

```cisco
write memory
```

EDGE1:

```cisco
write memory
```

ISP1:

```cisco
write memory
```

ISP2:

```cisco
write memory
```

---

## 29. Summary of Key Troubleshooting Lessons

```text
1. Always verify existing routing protocol state before applying new OSPF configuration.
2. OSPF passive-interface behavior must be checked carefully.
3. VLAN SVIs should be passive in both Single Area and Multi-Area designs.
4. EDGE-facing routed links should be the only OSPF neighbor links on CORE1/CORE2.
5. Old internal static routes should be removed after OSPF route learning is validated.
6. OSPF default route advertisement can exist in the database even if not installed in the routing table.
7. IP SLA operations cannot be modified after they are already running unless deleted and recreated.
8. A typo in a next-hop address can prevent the tracked primary route from working.
9. IP SLA target reachability should be stabilized with a host route through ISP1.
10. WAN failover may work even when end-to-end reachability fails due to return path issues.
11. Multi-Area OSPF requires CORE1/CORE2 to have interfaces in Area 0 and Area 10 to become ABRs.
12. EDGE1 should see O IA routes before summarization.
13. After area range summarization, EDGE1 should see O IA 10.10.0.0/16.
14. OSPF cost manipulation must be applied on EDGE1 G0/3 to prefer CORE1.
15. HSRP failover and routing failover are different events.
16. HSRP does not automatically track routed uplink failure unless tracking is configured.
17. Route summarization can hide individual VLAN failures from upstream routers.
18. Summarization and HSRP partial failure behavior must be considered together.
19. NAT/PAT failover is not the same as routing failover.
20. NAT/PAT failover is deferred to Phase 06 SD-WAN Underlay Enhancement.
```
