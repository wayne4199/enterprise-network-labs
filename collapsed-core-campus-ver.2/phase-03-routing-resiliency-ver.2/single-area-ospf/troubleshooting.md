# Phase 03 — Routing Resiliency Ver.2 Troubleshooting

## 1. Purpose

This document records troubleshooting notes for **Phase 03 Routing Resiliency Ver.2 — Case 1: Single Area OSPF**.

Phase 03 is divided into two OSPF cases:

```text
Case 1 — Single Area OSPF
Case 2 — Multi-Area OSPF
```

This troubleshooting document currently covers **Case 1 — Single Area OSPF** only.

Multi-Area OSPF troubleshooting notes will be added after that case is implemented and validated.

---

## 2. Troubleshooting Scope

This document covers troubleshooting for:

```text
1. Phase 03 pre-check issues
2. OSPF neighbor formation
3. OSPF passive-interface behavior
4. OSPF route learning
5. OSPF default route advertisement
6. Floating static route behavior
7. IP SLA and object tracking
8. WAN failover
9. ISP return path
10. OSPF cost manipulation
11. HSRP failover behavior
12. NAT/PAT failover limitation
13. Failure and recovery observations
```

The following topics are not implemented in this Single Area OSPF case:

```text
Route Summarization:
- Not implemented in Single Area OSPF
- Deferred to Multi-Area OSPF

NAT/PAT Failover:
- Limitation identified only
- Full implementation deferred to Phase 06 SD-WAN Underlay Enhancement
```

---

## 3. General Troubleshooting Baseline

Before troubleshooting any Phase 03 routing issue, verify the Phase 02 baseline is still stable.

## 3.1 MGMT-SRV1 Service Check

Run on MGMT-SRV1:

```bash
ip addr show ens2
ip route
systemctl status dnsmasq --no-pager
systemctl status chrony --no-pager
systemctl status rsyslog --no-pager
```

Expected baseline:

```text
MGMT-SRV1 IP: 10.10.40.10/24
Default gateway: 10.10.40.254
dnsmasq: active
chrony: active
rsyslog: active
```

## 3.2 DNS Check

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

## 3.3 Important DNS Observation

During pre-check, direct dnsmasq queries worked correctly:

```bash
dig @10.10.40.10 server1.campus.lab
```

However, normal queries using the system resolver could fail if Ubuntu used `127.0.0.53` instead of dnsmasq.

Example symptom:

```text
server1.campus.lab → NXDOMAIN
```

Cause:

```text
systemd-resolved was not using dnsmasq as the default resolver.
dnsmasq itself was working correctly.
```

Troubleshooting command:

```bash
resolvectl status
```

Operational workaround:

```bash
dig @10.10.40.10 server1.campus.lab
dig @10.10.40.10 mgmt-srv1.campus.lab
dig @10.10.40.10 internet-srv.campus.lab
```

---

## 4. OSPF Pre-Check

Before applying OSPF configuration, check whether old routing protocol configuration already exists.

Run on CORE1, CORE2, and EDGE1:

```cisco
show ip protocols
show running-config | section router ospf
show running-config | section router eigrp
show running-config | section router bgp
```

Reason:

```text
During the lab, an unexpected OSPF passive-interface state existed.
This affected OSPF neighbor formation.
```

Lesson learned:

```text
Do not assume the routing protocol configuration is empty.
Always verify existing routing protocol state before starting a routing phase.
```

---

## 5. Issue: OSPF Neighbor Does Not Form

## 5.1 Symptom

EDGE1 shows OSPF-enabled interfaces, but no neighbors.

```cisco
show ip ospf interface brief
show ip ospf neighbor
```

Observed symptom:

```text
EDGE1 Gi0/2 → Area 0
EDGE1 Gi0/3 → Area 0
Neighbor count → 0
```

## 5.2 Possible Causes

```text
1. CORE1 / CORE2 OSPF is not configured.
2. CORE1 / CORE2 G0/0 is not included in OSPF.
3. CORE1 / CORE2 G0/0 is passive.
4. Area mismatch.
5. Network statement mismatch.
6. Interface is down/down or administratively down.
```

## 5.3 Troubleshooting Commands

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

## 5.4 Correct OSPF Passive Interface Design

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

## 5.5 Fix

If G0/0 on CORE1 or CORE2 is passive, remove passive state for that interface.

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

Expected result on EDGE1:

```text
Neighbor 2.2.2.2 → FULL
Neighbor 3.3.3.3 → FULL
```

---

## 6. Issue: OSPF Neighbors Form on VLAN SVIs

## 6.1 Symptom

CORE1 and CORE2 form OSPF neighbors over VLAN SVIs instead of only the routed link to EDGE1.

Possible output:

```text
OSPF neighbors appear on VLAN10, VLAN20, VLAN40, or other SVIs.
```

## 6.2 Cause

VLAN SVIs are not passive in OSPF.

This can happen if the configuration uses specific passive-interface commands incorrectly or if `passive-interface default` was not applied.

## 6.3 Correct Design

```text
CORE1 / CORE2 G0/0:
- Non-passive
- OSPF neighbor allowed

CORE1 / CORE2 VLAN SVIs:
- Passive
- Advertise connected networks
- Do not form neighbors
```

## 6.4 Fix

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

Verify:

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

## 7.1 Symptom

OSPF neighbors are FULL, but EDGE1 does not show internal VLAN routes.

```cisco
show ip route ospf
```

Expected internal VLAN routes:

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

## 7.2 Possible Causes

```text
1. CORE1 / CORE2 do not advertise 10.10.0.0/16 into OSPF.
2. VLAN SVIs are down/down.
3. OSPF process does not include the VLAN interfaces.
4. Static routes with more preferred attributes are hiding expected route behavior.
```

## 7.3 Troubleshooting Commands

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

## 7.4 Correct CORE1 OSPF Network Statement

```cisco
router ospf 1
 network 172.16.0.0 0.0.0.3 area 0
 network 10.10.0.0 0.0.255.255 area 0
```

## 7.5 Correct CORE2 OSPF Network Statement

```cisco
router ospf 1
 network 172.16.0.4 0.0.0.3 area 0
 network 10.10.0.0 0.0.255.255 area 0
```

---

## 8. Issue: Static Routes Still Preferred Over OSPF

## 8.1 Symptom

EDGE1 has OSPF-learned internal routes, but old static routes still exist.

Check:

```cisco
show running-config | include ^ip route 10.10
show ip route 10.10.40.0
show ip route ospf
```

## 8.2 Important Routing Rule

Routers use longest prefix match before administrative distance.

Example:

```text
10.10.40.10 destination:
- 10.10.40.0/24 OSPF route is more specific
- 10.10.0.0/16 static summary is less specific
```

However, after OSPF is validated, keeping old internal static summary routes can confuse troubleshooting and route behavior.

## 8.3 Fix

Remove old internal static summary routes from EDGE1:

```cisco
conf t
no ip route 10.10.0.0 255.255.0.0 172.16.0.2
no ip route 10.10.0.0 255.255.0.0 172.16.0.6 10
end
write memory
```

Verify:

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

## 9. Issue: OSPF Default Route Advertisement Is Not Installed on CORE1/CORE2

## 9.1 Symptom

EDGE1 advertises default route into OSPF, but CORE1/CORE2 routing table still shows static default route.

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

## 9.2 Cause

Static default route has lower administrative distance than OSPF external route.

```text
Static AD = 1
OSPF External AD = 110
```

Therefore, static default route remains preferred.

## 9.3 Expected Result

This is expected if static default routes still exist on CORE1/CORE2.

The important validation point is:

```text
CORE1 and CORE2 receive the OSPF Type-5 External LSA for 0.0.0.0/0 from EDGE1.
```

## 9.4 Verification

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

## 10. Issue: IP SLA Timeout Cannot Be Modified

## 10.1 Symptom

When trying to modify IP SLA after it has started, IOS shows:

```text
Entry already running and cannot be modified
only can delete (no) and start over
```

## 10.2 Cause

An IP SLA operation that is already scheduled and running cannot be directly modified.

## 10.3 Options

Option 1 — Keep current IP SLA:

```text
Recommended during the lab.
The goal is to validate IP SLA + tracking behavior, not timer tuning.
```

Option 2 — Delete and recreate IP SLA:

```cisco
conf t
no ip sla 1
ip sla 1
 icmp-echo 203.0.113.10 source-interface GigabitEthernet0/0
 frequency 5
 threshold 1000
 timeout 1000
exit
ip sla schedule 1 life forever start-time now
track 1 ip sla 1 reachability
end
write memory
```

## 10.4 Lab Decision

During this lab, the IP SLA was kept running without deleting it.

The primary learning goal was:

```text
IP SLA reachability monitoring
Object tracking
Default route failover
```

---

## 11. Issue: Primary Default Route Uses Wrong Next-Hop

## 11.1 Symptom

The primary default route does not install even though Track 1 is Up.

Routing table shows backup path:

```text
Gateway of last resort → 100.64.2.1
```

Running-config shows a wrong next-hop:

```cisco
ip route 0.0.0.0 0.0.0.0 100.61.1.1 track 1
```

## 11.2 Cause

Typo in the ISP1 next-hop address.

Wrong:

```text
100.61.1.1
```

Correct:

```text
100.64.1.1
```

## 11.3 Fix

```cisco
conf t
no ip route 0.0.0.0 0.0.0.0 100.61.1.1 track 1
ip route 0.0.0.0 0.0.0.0 100.64.1.1 track 1
end
write memory
```

## 11.4 Verification

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

## 12. Issue: ISP1 Failure Switches Route to ISP2 but INTERNET-SRV Ping Fails

## 12.1 Symptom

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

But ping to INTERNET-SRV fails:

```cisco
ping 203.0.113.10
```

## 12.2 Important Observation

This does not mean IP SLA failover failed.

Routing failover worked correctly.

The issue is the return path from the Internet segment.

## 12.3 Verification

Run on EDGE1 during failure state:

```cisco
ping 100.64.2.1
ping 203.0.113.2
ping 203.0.113.10 source GigabitEthernet0/1
```

Observed result:

```text
EDGE1 → ISP2 next-hop 100.64.2.1: Success
EDGE1 → ISP2 Internet-side 203.0.113.2: Success
EDGE1 → INTERNET-SRV 203.0.113.10: Fail before return path correction
```

## 12.4 Cause

INTERNET-SRV uses ISP1 as its default gateway.

```text
INTERNET-SRV default gateway: 203.0.113.1
```

When traffic leaves EDGE1 through ISP2, the return traffic goes back toward ISP1.

ISP1 must know how to reach the ISP2 WAN subnet.

## 12.5 Fix

Add return route on ISP1:

```cisco
conf t
ip route 100.64.2.0 255.255.255.252 203.0.113.2
end
write memory
```

## 12.6 Verification

On EDGE1 during ISP1 failure:

```cisco
ping 203.0.113.10 source GigabitEthernet0/1
```

Expected result:

```text
Ping succeeds through ISP2 after return path correction.
```

---

## 13. Issue: HSRP Does Not Fail Over When CORE1 G0/0 Is Shut Down

## 13.1 Symptom

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
EDGE1 can move internal routes toward CORE2.
CORE1 may still remain HSRP Active for VLAN40.
```

MGMT-SRV1 may still reach:

```text
10.10.40.254
```

But fail to reach:

```text
172.16.0.1
203.0.113.10
```

## 13.2 Cause

HSRP does not automatically track CORE1 G0/0 unless interface tracking is configured.

The VLAN40 SVI remains up, so CORE1 can remain Active for VLAN40.

## 13.3 Important Design Lesson

This is a key difference between:

```text
Routing uplink failure
HSRP gateway failover
```

A routed uplink failure does not automatically trigger HSRP failover.

## 13.4 Immediate Fix

Restore CORE1 G0/0:

```cisco
conf t
interface GigabitEthernet0/0
 no shutdown
end
write memory
```

Verify:

```cisco
show ip ospf neighbor
show ip route ospf
```

## 13.5 Future Enhancement

If uplink failure must trigger HSRP failover, configure HSRP tracking in a future enhancement.

Example concept:

```text
Track CORE uplink
Reduce HSRP priority when uplink fails
Allow peer core switch to become Active
```

---

## 14. Issue: HSRP Failover Does Not Occur for VLAN40

## 14.1 Symptom

CORE1 VLAN40 is shut down, but CORE2 does not become Active.

Test command:

```cisco
conf t
interface Vlan40
 shutdown
end
```

## 14.2 Troubleshooting Commands

On CORE1 / CORE2:

```cisco
show standby brief
show ip interface brief | include Vlan40
show running-config interface Vlan40
```

## 14.3 Possible Causes

```text
1. CORE2 VLAN40 SVI is down.
2. CORE2 VLAN40 HSRP is not configured correctly.
3. VLAN40 is not allowed on trunks.
4. HSRP group number mismatch.
5. HSRP virtual IP mismatch.
6. Preempt or priority behavior is not as expected.
```

## 14.4 Expected Result

When CORE1 VLAN40 goes down:

```text
CORE2 VLAN40 becomes Active.
VLAN40 VIP 10.10.40.254 remains reachable.
MGMT-SRV1 maintains connectivity.
```

## 14.5 Restore CORE1 VLAN40

```cisco
conf t
interface Vlan40
 no shutdown
end
write memory
```

Verify:

```cisco
show standby brief
```

Expected recovery:

```text
CORE1 returns as Active for VLAN40 if preempt is configured.
CORE2 returns to Standby.
```

---

## 15. Issue: OSPF Cost Manipulation Does Not Prefer CORE1

## 15.1 Symptom

After setting OSPF cost on EDGE1 G0/3, EDGE1 still shows ECMP toward CORE1 and CORE2.

Expected:

```text
10.10.x.0/24 routes should prefer 172.16.0.2 through G0/2.
```

## 15.2 Troubleshooting Commands

On EDGE1:

```cisco
show ip ospf interface brief
show ip route ospf
show ip route 10.10.40.0
show running-config interface GigabitEthernet0/3
```

## 15.3 Correct Configuration

```cisco
conf t
interface GigabitEthernet0/3
 ip ospf cost 50
end
write memory
```

## 15.4 Expected Result

```text
EDGE1 G0/2 cost 1  → CORE1
EDGE1 G0/3 cost 50 → CORE2

EDGE1 prefers CORE1 for internal VLAN routes.
```

---

## 16. NAT/PAT Failover Limitation

## 16.1 Symptom

WAN routing fails over to ISP2, but internal user NAT/PAT behavior is not fully redundant.

## 16.2 Current NAT/PAT Design

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

## 16.3 Cause

Default route can move to ISP2, but NAT overload remains tied to ISP1 G0/0.

Therefore:

```text
Routing failover works.
NAT/PAT failover is not complete.
```

## 16.4 Verification Commands

On EDGE1:

```cisco
show running-config | include ip nat
show ip nat statistics
show ip nat translations
show running-config | include ^ip route 0.0.0.0
```

## 16.5 Phase 03 Decision

Do not fully implement NAT/PAT failover in Phase 03.

Document the limitation only.

## 16.6 Carry-over to Phase 06

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

## 17. Route Summarization Troubleshooting Note

## 17.1 Symptom

Route summarization is expected, but EDGE1 still learns individual /24 VLAN routes.

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

## 17.2 Cause

This is expected in Single Area OSPF.

Current design:

```text
EDGE1, CORE1, CORE2, and VLAN SVIs are all in Area 0.
```

There is no ABR and no inter-area boundary.

## 17.3 Correct Interpretation

This is not a failure.

Route summarization is not implemented in the Single Area OSPF case.

## 17.4 Future Fix

Move to Multi-Area OSPF.

Planned future design:

```text
EDGE1 ↔ CORE1/CORE2 routed links = Area 0
CORE1/CORE2 VLAN SVIs = Area 10
CORE1/CORE2 = ABR
```

Then apply summarization on CORE1/CORE2:

```cisco
router ospf 1
 area 10 range 10.10.0.0 255.255.0.0
```

Expected future result on EDGE1:

```text
10.10.0.0/16 summary route
```

---

## 18. Final Recovery Checklist

After any failure test, return the lab to normal state.

## 18.1 EDGE1

```cisco
show ip interface brief
show track 1
show ip route 0.0.0.0
show ip ospf neighbor
show ip route ospf
```

Expected state:

```text
G0/0 up/up
G0/1 up/up
G0/2 up/up
G0/3 up/up
Track 1 Up
Default route via 100.64.1.1
OSPF neighbors with CORE1 and CORE2 are FULL
```

## 18.2 CORE1

```cisco
show ip interface brief
show ip ospf neighbor
show standby brief
```

Expected state:

```text
G0/0 up/up
Vlan40 up/up
OSPF neighbor with EDGE1 is FULL
VLAN40 should return to CORE1 Active if preempt is configured
```

## 18.3 CORE2

```cisco
show ip interface brief
show ip ospf neighbor
show standby brief
```

Expected state:

```text
G0/0 up/up
OSPF neighbor with EDGE1 is FULL
VLAN40 should return to CORE2 Standby after CORE1 recovery if preempt is configured
```

## 18.4 MGMT-SRV1

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
campus.lab DNS records resolve correctly
```

---

## 19. Save Configuration After Recovery

After restoring the lab to normal state, save all device configurations.

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

## 20. Summary of Key Troubleshooting Lessons

```text
1. Always verify existing routing protocol state before applying new OSPF configuration.
2. OSPF passive-interface behavior must be checked carefully.
3. VLAN SVIs should be passive in this Single Area design.
4. EDGE-facing routed links should be the only OSPF neighbor links on CORE1/CORE2.
5. Static routes can hide or override OSPF route behavior.
6. OSPF default route advertisement can exist in the database even if not installed in the routing table.
7. IP SLA operations cannot be modified after they are already running unless deleted and recreated.
8. A typo in next-hop address can make a tracked primary route fail silently.
9. WAN failover may work even when end-to-end Internet reachability fails due to return path issues.
10. HSRP failover and routing uplink failover are different events.
11. HSRP does not automatically track uplink failure unless tracking is configured.
12. NAT/PAT failover is not the same as routing failover.
13. Route summarization is not appropriate in Single Area OSPF and must be implemented in Multi-Area OSPF.
```
