# PHASE 06 — Troubleshooting Guide

## Overview

This document records the major issues encountered during the SD-WAN Underlay Enhancement lab and the troubleshooting process used to resolve them.

---

# Issue 1 — INTERNET-SRV1 Could Not Reach EDGE1 via ISP2

## Symptoms

Guest traffic was intended to use ISP2 through Policy-Based Routing (PBR).

However:

```text
EDGE1# ping 203.0.113.10 source g0/3

Success rate is 0 percent (0/5)
```

Traffic through ISP1 worked correctly.

---

## Investigation

Verified connectivity between:

```text
ISP1 ↔ INTERNET-SRV1
ISP2 ↔ INTERNET-SRV1
```

Both paths were reachable.

Checked routing table on INTERNET-SRV1:

```bash
route -n
```

Only ISP1 return path existed.

---

## Root Cause

INTERNET-SRV1 had no return route toward the ISP2 WAN subnet.

Packets sent through ISP2 reached the server, but reply traffic returned through ISP1.

This created asymmetric routing.

---

## Resolution

Added a static route on INTERNET-SRV1.

```bash
route add -net 100.64.2.0 netmask 255.255.255.252 gw 203.0.113.2
```

Verification:

```bash
ping 100.64.2.2
```

Successful.

---

# Issue 2 — VLAN 50 Gateway Unreachable

## Symptoms

GUEST1 could not ping:

```text
10.50.50.254
```

---

## Investigation

Checked ACC3:

```cisco
show vlan brief
show interfaces status
```

Verified:

```text
Gi0/3 assigned to VLAN 50
```

Checked CORE1 and CORE2:

```cisco
show ip interface brief
```

Output:

```text
Vlan50 administratively down
```

---

## Root Cause

SVI was configured but never enabled.

---

## Resolution

On both CORE1 and CORE2:

```cisco
interface vlan 50
 no shutdown
```

Verification:

```text
show standby brief
```

HSRP became operational.

---

# Issue 3 — Guest Internet Access Failed

## Symptoms

GUEST1 could reach:

```text
10.50.50.254
```

but could not reach:

```text
203.0.113.10
```

---

## Investigation

Verified:

```cisco
show route-map
show ip policy
```

PBR counters increased.

Traffic was matching the route-map.

However:

```cisco
show ip nat translations
```

showed no Guest NAT entries.

---

## Root Cause

ACL 10 was originally used for Phase 03 NAT.

It only contained:

```text
10.10.10.0/24
10.20.20.0/24
10.40.40.0/24
```

Guest subnet:

```text
10.50.50.0/24
```

was missing.

Therefore NAT never occurred.

---

## Resolution

Created dedicated Guest NAT ACL.

```cisco
ip access-list standard NAT-GUEST
 permit 10.50.50.0 0.0.0.255
```

Configured:

```cisco
ip nat inside source list NAT-GUEST interface g0/3 overload
```

Verification:

```cisco
show ip nat translations
```

Output:

```text
Inside local 10.50.50.10
Inside global 100.64.2.2
```

---

# Issue 4 — NAT Continued Using ISP1

## Symptoms

After configuring Guest NAT for ISP2:

```cisco
show ip nat translations
```

still displayed:

```text
Inside global 100.64.1.2
```

---

## Investigation

Old NAT translations remained in memory.

---

## Root Cause

Existing dynamic NAT entries were created before Guest NAT migration.

IOS continued using active translations.

---

## Resolution

Cleared translations.

```cisco
clear ip nat translation *
```

Generated new traffic.

```bash
ping 203.0.113.10
```

Verification:

```cisco
show ip nat translations
```

Now showed:

```text
Inside global 100.64.2.2
```

---

# Issue 5 — IP SLA Track 2 Stayed Down

## Symptoms

Track 2 never recovered.

```cisco
show track
```

Output:

```text
Track 2 Reachability Down
```

---

## Investigation

Checked ISP2:

```cisco
show ip interface brief
```

Output:

```text
GigabitEthernet0/0 administratively down
```

---

## Root Cause

ISP2 WAN-facing interface was intentionally shut during failover testing and not restored.

---

## Resolution

Re-enable interface.

```cisco
interface g0/0
 no shutdown
```

Verification:

```cisco
show track
```

Output:

```text
Track 2 Reachability Up
```

---

# Issue 6 — Guest Segmentation Not Working

## Symptoms

Guest users could still access internal networks.

---

## Investigation

ACL existed:

```cisco
show access-lists GUEST-SEGMENTATION
```

However ACL hit counters remained at zero.

---

## Root Cause

ACL was not applied to VLAN 50 SVI.

---

## Resolution

Applied ACL inbound.

```cisco
interface vlan 50
 ip access-group GUEST-SEGMENTATION in
```

Verification:

```cisco
show access-lists GUEST-SEGMENTATION
```

Deny counters increased.

---

# Issue 7 — PBR Verification

## Symptoms

Need to verify Guest traffic preferred ISP2.

---

## Investigation

Checked route-map statistics.

```cisco
show route-map
```

Output:

```text
Policy routing matches increasing
```

Checked NAT:

```cisco
show ip nat translations
```

Output:

```text
Inside global 100.64.2.2
```

---

## Verification

Guest traffic:

```text
10.50.50.0/24
```

Successfully exited via:

```text
ISP2 (100.64.2.2)
```

When ISP2 failed:

```text
Track 2 Down
```

Traffic automatically moved to:

```text
ISP1 (100.64.1.2)
```

without manual intervention.

---

# Final Validation Checklist

```text
[PASS] Guest VLAN reachable
[PASS] HSRP operational
[PASS] PBR matches increasing
[PASS] NAT translations created
[PASS] Guest traffic exits ISP2
[PASS] Automatic failover to ISP1
[PASS] Guest segmentation ACL enforced
[PASS] IP SLA Track 1 Up
[PASS] IP SLA Track 2 Up
[PASS] Internet server reachable
```
