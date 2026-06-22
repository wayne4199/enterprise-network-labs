# Phase 06 — SD-WAN-Ready Underlay Enhancement Troubleshooting

## 1. Purpose

This document records troubleshooting cases and operational lessons from:

```text
Phase 06 — SD-WAN-Ready Underlay Enhancement
```

Phase 06 focused on improving the existing dual-ISP WAN underlay before moving into SD-WAN Overlay.

Major troubleshooting areas:

```text
NAT/PAT failover
IP SLA / Object Tracking
Track-aware default routing
Policy-Based Routing
Application-aware path selection simulation
QoS and WAN integration
Failure and recovery behavior
```

---

## 2. Troubleshooting Methodology

The troubleshooting approach used in this phase followed this order:

```text
1. Confirm physical/logical interface status.
2. Confirm routing decision.
3. Confirm IP SLA and Track state.
4. Confirm NAT/PAT mapping.
5. Confirm PBR route-map match.
6. Confirm QoS marking and WAN egress class match.
7. Confirm host reachability.
8. Restore failed interfaces after each test.
```

Primary EDGE1 commands:

```cisco
show ip interface brief
show ip route 0.0.0.0
show ip route
show ip sla statistics
show track

show ip nat translations
show ip nat statistics
show running-config | include ip nat
show running-config | section route-map NAT

show ip policy
show route-map WAN-BUSINESS-INTENT

show policy-map interface GigabitEthernet0/0
show policy-map interface GigabitEthernet0/1
show policy-map interface GigabitEthernet0/2
show policy-map interface GigabitEthernet0/3
```

---

## 3. Issue 1 — Routing Failover Worked but NAT/PAT Failed

### Symptom

During the initial ISP1 failure test:

```text
EDGE1 Gi0/0 was shutdown.
Track 1 went Down.
Default route moved to ISP2.
MGMT-SRV1 ping to 203.0.113.10 failed.
```

Observed ping failure:

```text
From 172.16.0.1 icmp_seq=1 Destination Host Unreachable
```

Observed NAT status:

```text
Total active translations: 0
```

Existing NAT/PAT configuration:

```cisco
ip nat inside source list ENTERPRISE-NAT interface GigabitEthernet0/0 overload
```

### Cause

The routing table correctly failed over to ISP2, but NAT/PAT was still tied to the ISP1-facing interface:

```text
EDGE1 Gi0/0 = ISP1
EDGE1 Gi0/1 = ISP2
```

When ISP1 failed, traffic tried to use ISP2, but the only PAT rule was still bound to Gi0/0.

### Lesson

```text
Default route failover alone does not complete Internet failover.
NAT/PAT must also follow the actual WAN egress path.
```

### Fix

Replace single-interface PAT with route-map based dual-ISP NAT/PAT.

```cisco
no ip nat inside source list ENTERPRISE-NAT interface GigabitEthernet0/0 overload

route-map NAT-ISP1 permit 10
 match ip address ENTERPRISE-NAT
 match interface GigabitEthernet0/0

route-map NAT-ISP2 permit 10
 match ip address ENTERPRISE-NAT
 match interface GigabitEthernet0/1

ip nat inside source route-map NAT-ISP1 interface GigabitEthernet0/0 overload
ip nat inside source route-map NAT-ISP2 interface GigabitEthernet0/1 overload
```

### Verification

ISP1 normal state:

```text
10.10.40.10 → 100.64.1.2
NAT-ISP1 refcount 1
NAT-ISP2 refcount 0
```

ISP1 failure state:

```text
10.10.40.10 → 100.64.2.2
NAT-ISP1 refcount 0
NAT-ISP2 refcount 1
```

Result:

```text
PASS
```

---

## 4. Issue 2 — `no ip sla 2` Displayed `Operation not found`

### Symptom

When preparing to configure IP SLA 2:

```cisco
no ip sla 2
```

The router returned:

```text
Operation not found
```

### Cause

IP SLA operation 2 did not exist yet.

This message simply means there was nothing to remove.

### Fix

No fix was required.

Continue with the IP SLA 2 configuration:

```cisco
ip sla 2
 icmp-echo 100.64.2.1 source-interface GigabitEthernet0/1
 threshold 1000
 timeout 1000
 frequency 5

ip sla schedule 2 life forever start-time now
```

### Lesson

```text
"Operation not found" after "no ip sla 2" is not an error if IP SLA 2 was never configured.
```

Result:

```text
PASS
```

---

## 5. Issue 3 — Backup Default Route Needed Track 2

### Symptom

Before Phase 06 enhancement, the backup default route was configured as a simple floating static route:

```cisco
ip route 0.0.0.0 0.0.0.0 100.64.2.1 10
```

### Cause

This route did not verify whether ISP2 was actually reachable.

If ISP1 failed and ISP2 was also unavailable, the router could still try to install or use an unusable backup route.

### Fix

Add IP SLA 2 and Track 2 for ISP2, then tie the backup default route to Track 2.

```cisco
ip sla 2
 icmp-echo 100.64.2.1 source-interface GigabitEthernet0/1
 threshold 1000
 timeout 1000
 frequency 5

ip sla schedule 2 life forever start-time now

track 2 ip sla 2 reachability
 delay down 10 up 5

no ip route 0.0.0.0 0.0.0.0 100.64.2.1 10
ip route 0.0.0.0 0.0.0.0 100.64.2.1 10 track 2
```

### Verification

ISP2 normal:

```text
IP SLA 2 return code: OK
Track 2: Up
```

ISP2 failure:

```text
IP SLA 2 return code: Timeout
Track 2: Down
Default route remains via ISP1 if Track 1 is still Up.
```

Result:

```text
PASS
```

---

## 6. Issue 4 — `show ip route 0.0.0.0` Shows Only ISP1 Route During Normal State

### Symptom

After configuring both default routes:

```cisco
ip route 0.0.0.0 0.0.0.0 100.64.1.1 track 1
ip route 0.0.0.0 0.0.0.0 100.64.2.1 10 track 2
```

`show ip route 0.0.0.0` showed only:

```text
0.0.0.0/0 via 100.64.1.1
```

### Cause

This is normal.

The ISP2 default route has administrative distance 10.
It is a floating backup route and will not appear in the routing table while the primary route through ISP1 is available.

### Verification

Confirm the configured backup route exists:

```cisco
show running-config | include ^ip route
```

Expected configuration:

```cisco
ip route 0.0.0.0 0.0.0.0 100.64.1.1 track 1
ip route 0.0.0.0 0.0.0.0 100.64.2.1 10 track 2
```

### Lesson

```text
A floating static route may exist in the running configuration but not appear in the routing table until the primary route is removed.
```

Result:

```text
NORMAL BEHAVIOR
```

---

## 7. Issue 5 — Mistyped Route Verification Command

### Symptom

The following command was entered:

```cisco
show ip 0.0.0.0
```

The router returned an invalid input error.

### Cause

The command syntax was incorrect.

### Correct Command

Use:

```cisco
show ip route 0.0.0.0
```

### Lesson

```text
Use "show ip route 0.0.0.0" to verify the active default route.
```

Result:

```text
RESOLVED
```

---

## 8. Issue 6 — PBR Route-Map Match Counters Stayed at 0

### Symptom

After applying PBR:

```cisco
interface GigabitEthernet0/2
 ip policy route-map WAN-BUSINESS-INTENT

interface GigabitEthernet0/3
 ip policy route-map WAN-BUSINESS-INTENT
```

`show route-map WAN-BUSINESS-INTENT` initially showed:

```text
Policy routing matches: 0 packets, 0 bytes
```

### Cause

No matching test traffic had been generated yet.

PBR counters increase only when packets matching the ACLs enter the interface where the policy is applied.

### Fix

Generate matching traffic from the correct host.

Examples:

```bash
ping -c 4 203.0.113.10
```

from:

```text
EXEC
SALES
IT-OPS
MGMT-SRV1
ROGUE-PC
```

### Verification

Expected PBR matches:

```text
VOICE-like  → route-map sequence 5
SCAVENGER   → route-map sequence 10
MGMT        → route-map sequence 20
VIDEO-like  → route-map sequence 30
BUSINESS    → route-map sequence 40
```

### Lesson

```text
PBR counters remaining at 0 is normal until matching traffic is generated.
```

Result:

```text
PASS
```

---

## 9. Issue 7 — PBR Must Be Applied on Inside-Facing Interfaces

### Symptom

PBR would not influence outbound ISP path if applied in the wrong direction or on the wrong interface.

### Correct Placement

PBR must be applied inbound on EDGE1 campus-facing interfaces:

```cisco
interface GigabitEthernet0/2
 ip policy route-map WAN-BUSINESS-INTENT

interface GigabitEthernet0/3
 ip policy route-map WAN-BUSINESS-INTENT
```

### Reason

Campus traffic enters EDGE1 from CORE1 or CORE2.

```text
Campus Host
→ CORE1 / CORE2
→ EDGE1 Gi0/2 or Gi0/3 inbound
→ PBR decision
→ NAT/PAT
→ WAN egress
```

### Verification

```cisco
show ip policy
```

Expected output:

```text
Interface        Route map
Gi0/2            WAN-BUSINESS-INTENT
Gi0/3            WAN-BUSINESS-INTENT
```

### Lesson

```text
PBR should be applied where the original inside traffic enters the WAN edge.
```

Result:

```text
PASS
```

---

## 10. Issue 8 — PBR Could Force Traffic to a Failed ISP Without Tracking

### Symptom

A normal PBR statement can force traffic to a next-hop even if that next-hop is down.

Example of risky design:

```cisco
set ip next-hop 100.64.2.1
```

### Cause

Traditional PBR without reachability verification does not automatically know whether the configured next-hop is still valid.

### Fix

Use track-aware PBR:

```cisco
set ip next-hop verify-availability 100.64.2.1 10 track 2
```

or:

```cisco
set ip next-hop verify-availability 100.64.1.1 10 track 1
```

### Verification

When the preferred ISP is healthy:

```text
set ip next-hop verify-availability 100.64.x.1 10 track x [up]
```

When the preferred ISP is failed:

```text
set ip next-hop verify-availability 100.64.x.1 10 track x [down]
```

### Lesson

```text
PBR should express preferred path intent, not force traffic into a failed WAN path.
```

Result:

```text
PASS
```

---

## 11. Issue 9 — ROGUE-PC Preferred ISP2 but Needed ISP1 Fallback

### Symptom

ROGUE-PC was designed to prefer ISP2:

```text
ROGUE-PC 10.10.20.126 → ISP2
```

During ISP2 failure, ROGUE-PC still needed Internet reachability.

### Verification Scenario

ISP2 failure:

```cisco
interface GigabitEthernet0/1
 shutdown
```

Expected state:

```text
Track 2 Down
Track 1 Up
Default route remains via ISP1
PBR next-hop 100.64.2.1 track 2 [down]
```

Expected NAT/PAT result:

```text
10.10.20.126 → 100.64.1.2
```

### Result

ROGUE-PC fell back to ISP1 through normal routing.

```text
NAT-ISP1 refcount 1
NAT-ISP2 refcount 0
```

### Lesson

```text
Track-aware PBR allows preferred path steering while preserving fallback behavior.
```

Result:

```text
PASS
```

---

## 12. Issue 10 — MGMT-SRV1 Preferred ISP1 but Needed ISP2 Fallback

### Symptom

MGMT-SRV1 was designed to prefer ISP1:

```text
MGMT-SRV1 10.10.40.10 → ISP1
```

During ISP1 failure, MGMT-SRV1 still needed Internet reachability.

### Verification Scenario

ISP1 failure:

```cisco
interface GigabitEthernet0/0
 shutdown
```

Expected state:

```text
Track 1 Down
Track 2 Up
Default route moves to ISP2
PBR next-hop 100.64.1.1 track 1 [down]
```

Expected NAT/PAT result:

```text
10.10.40.10 → 100.64.2.2
```

### Result

MGMT-SRV1 fell back to ISP2 through normal routing.

```text
NAT-ISP1 refcount 0
NAT-ISP2 refcount 1
```

QoS on ISP2:

```text
Gi0/1 CM-WAN-MGMT matched
```

### Lesson

```text
Management traffic can prefer ISP1 while still maintaining ISP2 fallback during ISP1 failure.
```

Result:

```text
PASS
```

---

## 13. Issue 11 — VIDEO Preferred ISP2 but Needed ISP1 Fallback

### Symptom

SALES / VIDEO-like traffic was designed to prefer ISP2:

```text
SALES 10.10.20.197 → ISP2
```

During ISP2 failure, VIDEO traffic still needed Internet reachability.

### Verification Scenario

ISP2 failure:

```cisco
interface GigabitEthernet0/1
 shutdown
```

Expected state:

```text
Track 2 Down
Track 1 Up
Default route remains via ISP1
PBR next-hop 100.64.2.1 track 2 [down]
```

Expected NAT/PAT result:

```text
10.10.20.197 → 100.64.1.2
```

### Result

SALES / VIDEO-like traffic fell back to ISP1.

QoS on ISP1:

```text
Gi0/0 CM-WAN-VIDEO matched
```

### Lesson

```text
Application-aware path preference must include fallback verification.
```

Result:

```text
PASS
```

---

## 14. Issue 12 — NAT Translation Did Not Always Show the Expected Host Immediately

### Symptom

After a ping test, `show ip nat translations` sometimes showed MGMT-SRV1 NTP/UDP traffic instead of the expected ICMP host.

Example:

```text
udp 100.64.1.2:49235 10.10.40.10:49235 185.125.190.57:123
```

### Cause

MGMT-SRV1 generated background NTP traffic.
NAT translations are dynamic and may include background system traffic, not only the test ping.

### Fix

Use a more specific command:

```cisco
show ip nat translations | include 10.10.10.117
show ip nat translations | include 10.10.20.197
show ip nat translations | include 10.10.30.140
show ip nat translations | include 10.10.40.10
show ip nat translations | include 10.10.20.126
```

Also clear translations before each test:

```cisco
clear ip nat translation *
```

### Lesson

```text
Use host-specific NAT filtering when validating a particular traffic flow.
```

Result:

```text
RESOLVED
```

---

## 15. Issue 13 — QoS Counters Were Not Always New or Clean

### Symptom

QoS class counters already had previous packet counts from earlier tests.

Example:

```text
CM-WAN-VOICE already had previous packets before the current test.
```

### Cause

QoS counters accumulate over time unless the interface counters or policy counters are cleared.

### Fix

Use directional reasoning:

```text
Check whether the correct class counter increases after generating traffic.
```

Useful commands:

```cisco
show policy-map interface GigabitEthernet0/0
show policy-map interface GigabitEthernet0/1
show policy-map interface GigabitEthernet0/2
show policy-map interface GigabitEthernet0/3
```

Optional cleanup command:

```cisco
clear counters
```

### Lesson

```text
QoS validation should focus on correct class matching and counter increase, not just whether a counter is non-zero.
```

Result:

```text
NORMAL BEHAVIOR
```

---

## 16. Issue 14 — QoS Must Mark Before NAT

### Symptom

From Phase 05, classifying traffic by original inside IP on WAN output caused custom QoS class counters to remain at 0.

### Cause

By the time traffic reaches WAN egress, NAT/PAT may have changed the original inside source IP.

### Final Design

Mark traffic before NAT on EDGE1 campus-facing interfaces:

```text
Gi0/2 input → CAMPUS-QOS-MARK-IN
Gi0/3 input → CAMPUS-QOS-MARK-IN
```

Then queue traffic after NAT on WAN egress using DSCP:

```text
Gi0/0 output → WAN-QOS-EGRESS
Gi0/1 output → WAN-QOS-EGRESS
```

### Lesson

```text
Classify and mark based on original inside identity before NAT.
Queue on WAN egress using DSCP after NAT.
```

Result:

```text
PASS
```

---

## 17. Issue 15 — Preferred Path and Active Default Route May Differ

### Symptom

The active default route was ISP1:

```text
0.0.0.0/0 via 100.64.1.1
```

But some traffic, such as VIDEO or SCAVENGER, used ISP2.

### Cause

PBR overrides the normal routing decision for matching traffic.

Example:

```text
SALES / VIDEO-like → ISP2
ROGUE-PC / SCAVENGER → ISP2
```

Even when the global default route points to ISP1, PBR can send selected traffic to ISP2.

### Verification

Check PBR route-map counters:

```cisco
show route-map WAN-BUSINESS-INTENT
```

Check NAT/PAT translation:

```cisco
show ip nat translations | include 10.10.20.197
show ip nat translations | include 10.10.20.126
```

Expected:

```text
SALES 10.10.20.197 → 100.64.2.2
ROGUE-PC 10.10.20.126 → 100.64.2.2
```

### Lesson

```text
Default route shows the normal path.
PBR can intentionally override that path for selected traffic.
```

Result:

```text
EXPECTED BEHAVIOR
```

---

## 18. Issue 16 — Interfaces Left Shutdown After Failure Testing

### Symptom

After failure testing, the network may remain in a degraded state if the test interface is not restored.

Common test shutdowns:

```cisco
interface GigabitEthernet0/0
 shutdown
```

or:

```cisco
interface GigabitEthernet0/1
 shutdown
```

### Fix

Always restore the interface after testing:

```cisco
configure terminal
interface GigabitEthernet0/0
 no shutdown
end
```

or:

```cisco
configure terminal
interface GigabitEthernet0/1
 no shutdown
end
```

### Verification

```cisco
show ip interface brief
show track
show ip route 0.0.0.0
```

Expected normal state:

```text
Gi0/0 up/up
Gi0/1 up/up
Track 1 Up
Track 2 Up
Default route via 100.64.1.1
```

### Lesson

```text
Every failure test must include a recovery check.
```

Result:

```text
PASS
```

---

## 19. Issue 17 — DHCP Host IPs Can Change After Restart

### Symptom

PBR and QoS ACLs rely on specific host IP addresses:

```text
EXEC        10.10.10.117
SALES       10.10.20.197
IT-OPS      10.10.30.140
ROGUE-PC    10.10.20.126
```

If DHCP assigns different IP addresses after restart, ACL matching may fail.

### Cause

Some hosts use DHCP.

### Fix Options

Option 1:

```text
Verify host IP addresses before testing.
```

Option 2:

```text
Update PBR and QoS ACLs with the new IP addresses.
```

Option 3:

```text
Create DHCP reservations for stable test host addresses.
```

### Verification

On Linux hosts:

```bash
ip addr
ip route
```

On EDGE1:

```cisco
show access-lists
show route-map WAN-BUSINESS-INTENT
show policy-map interface GigabitEthernet0/2
show policy-map interface GigabitEthernet0/3
```

### Lesson

```text
Host-based lab policies are simple, but they depend on stable test IP addresses.
```

Result:

```text
KNOWN LIMITATION
```

---

## 20. Issue 18 — Track Delay Requires Waiting Before Verification

### Symptom

After shutting or restoring an interface, Track state did not change instantly.

### Cause

Track delay was configured:

```cisco
delay down 10 up 5
```

### Expected Behavior

After failure:

```text
Wait around 10 seconds for Track Down.
```

After recovery:

```text
Wait around 5 seconds for Track Up.
```

### Lesson

```text
Track delay is intentional and prevents unnecessary flapping.
Do not verify too quickly after failure or recovery.
```

Result:

```text
EXPECTED BEHAVIOR
```

---

## 21. Issue 19 — IP SLA Failure Count Remained High After Recovery

### Symptom

`show ip sla statistics` showed a high number of failures even after the SLA was currently OK.

Example:

```text
Number of failures: 105
Latest operation return code: OK
```

### Cause

The failure counter is cumulative.

It includes previous intentional failure tests.

### Correct Interpretation

Focus on:

```text
Latest operation return code
Latest RTT
Track state
```

Current healthy state:

```text
Latest operation return code: OK
Track state: Up
```

### Lesson

```text
A high historical failure count is not a current problem if the latest return code is OK and Track is Up.
```

Result:

```text
NORMAL BEHAVIOR
```

---

## 22. Issue 20 — NAT/PAT and PBR Must Be Verified Together

### Symptom

A ping may succeed, but the traffic may not use the intended ISP.

### Cause

Reachability alone does not prove the correct WAN path was used.

### Correct Verification

For each test, verify all of the following:

```text
1. Ping success
2. PBR sequence match
3. NAT inside global address
4. Correct WAN QoS interface/class
```

Example for SALES / VIDEO-like normal path:

```text
Ping success
PBR sequence 30 match
NAT: 10.10.20.197 → 100.64.2.2
Gi0/1 CM-WAN-VIDEO match
```

Example for IT-OPS / BUSINESS-CRITICAL normal path:

```text
Ping success
PBR sequence 40 match
NAT: 10.10.30.140 → 100.64.1.2
Gi0/0 CM-WAN-BUSINESS-CRITICAL match
```

### Lesson

```text
End-to-end ping is only the first check.
Path, NAT, PBR, and QoS must be validated together.
```

Result:

```text
PASS
```

---

## 23. Final Troubleshooting Checklist

Use this checklist when Phase 06 behavior does not match expectations.

### 23.1 Interface Status

```cisco
show ip interface brief
```

Expected normal state:

```text
Gi0/0 up/up
Gi0/1 up/up
Gi0/2 up/up
Gi0/3 up/up
```

---

### 23.2 Track State

```cisco
show track
show ip sla statistics
```

Expected normal state:

```text
Track 1 Up
Track 2 Up
IP SLA 1 OK
IP SLA 2 OK
```

---

### 23.3 Default Route

```cisco
show ip route 0.0.0.0
```

Expected normal state:

```text
0.0.0.0/0 via 100.64.1.1
```

Expected ISP1 failure state:

```text
0.0.0.0/0 via 100.64.2.1
```

---

### 23.4 NAT/PAT

```cisco
show ip nat translations
show ip nat statistics
```

Expected normal mappings:

```text
ISP1 traffic → 100.64.1.2
ISP2 traffic → 100.64.2.2
```

---

### 23.5 PBR

```cisco
show ip policy
show route-map WAN-BUSINESS-INTENT
```

Expected policy placement:

```text
Gi0/2 → WAN-BUSINESS-INTENT
Gi0/3 → WAN-BUSINESS-INTENT
```

Expected PBR sequences:

```text
5   VOICE-like        → ISP1
10  SCAVENGER         → ISP2
20  MGMT              → ISP1
30  VIDEO-like        → ISP2
40  BUSINESS-CRITICAL → ISP1
100 DEFAULT           → normal routing
```

---

### 23.6 QoS

```cisco
show policy-map interface GigabitEthernet0/0
show policy-map interface GigabitEthernet0/1
show policy-map interface GigabitEthernet0/2
show policy-map interface GigabitEthernet0/3
```

Expected policy placement:

```text
Gi0/2 input  → CAMPUS-QOS-MARK-IN
Gi0/3 input  → CAMPUS-QOS-MARK-IN
Gi0/0 output → WAN-QOS-EGRESS
Gi0/1 output → WAN-QOS-EGRESS
```

---

## 24. Final Troubleshooting Result

Final Phase 06 troubleshooting status:

```text
Routing failover troubleshooting:       Completed
NAT/PAT failover troubleshooting:       Completed
IP SLA / Track troubleshooting:         Completed
PBR troubleshooting:                    Completed
Application-aware path troubleshooting: Completed
QoS + WAN integration troubleshooting:  Completed
Failure and recovery troubleshooting:   Completed
```

Final result:

```text
Phase 06 — SD-WAN-Ready Underlay Enhancement
Troubleshooting Status: Completed
Result: PASS
```
