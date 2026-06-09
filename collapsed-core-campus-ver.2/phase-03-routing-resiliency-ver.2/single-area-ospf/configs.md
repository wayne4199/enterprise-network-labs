# Phase 03 — Routing Resiliency Ver.2 Configs

## 1. Purpose

This document records the configuration changes applied during **Phase 03 Routing Resiliency Ver.2 — Case 1: Single Area OSPF**.

Phase 03 is divided into two OSPF cases:

```text
Case 1 — Single Area OSPF
Case 2 — Multi-Area OSPF
```

This file currently documents **Case 1 — Single Area OSPF** only.

Multi-Area OSPF configuration will be added later after the Single Area OSPF case is completed and saved.

---

## 2. Configuration Scope

The Single Area OSPF case includes the following configuration areas:

```text
1. OSPF Single Area 0
2. OSPF Neighbor / Passive Interface
3. OSPF Default Route Advertisement
4. Internal Static Route Removal on EDGE1
5. Floating Static Route
6. IP SLA
7. Object Tracking
8. WAN Resiliency
9. OSPF Cost Manipulation
10. HSRP Failover Validation
11. NAT/PAT Failover Limitation
```

The following items were **not implemented** in this Single Area OSPF case:

```text
Route Summarization:
- Not implemented in Single Area OSPF
- Deferred to Multi-Area OSPF

NAT/PAT Failover:
- Limitation identified only
- Full implementation deferred to Phase 06 SD-WAN Underlay Enhancement
```

---

## 3. Important Baseline Preservation

Phase 03 must preserve the Phase 01 and Phase 02 baseline.

Do not remove or redesign the following:

```text
- VLAN design
- HSRP VIPs
- Rapid-PVST
- STP root and HSRP active role alignment
- VLAN999 native blackhole
- MGMT-SRV1 Enterprise Management Server
- DHCP relay to 10.10.40.10
- campus.lab DNS records
- SSH-only management
- AAA local authentication
- MGMT-ACCESS ACL
- SNMP-MGMT ACL
- Syslog / NTP / SNMP / Config Backup settings
- Existing NAT/PAT baseline unless specifically noted
```

---

# 4. Addressing Reference

## 4.1 WAN / Internet Segment

```text
INTERNET-SRV: 203.0.113.10/24
Gateway     : 203.0.113.1

ISP1 G0/0   : 203.0.113.1/24
ISP2 G0/0   : 203.0.113.2/24

ISP1 G0/1   : 100.64.1.1/30
EDGE1 G0/0  : 100.64.1.2/30

ISP2 G0/1   : 100.64.2.1/30
EDGE1 G0/1  : 100.64.2.2/30
```

## 4.2 EDGE to CORE Routed Links

```text
EDGE1 G0/2  : 172.16.0.1/30
CORE1 G0/0  : 172.16.0.2/30

EDGE1 G0/3  : 172.16.0.5/30
CORE2 G0/0  : 172.16.0.6/30
```

---

# 5. CORE1 Configuration

## 5.1 OSPF Single Area 0

```cisco
conf t
!
router ospf 1
 router-id 2.2.2.2
 passive-interface default
 no passive-interface GigabitEthernet0/0
 network 172.16.0.0 0.0.0.3 area 0
 network 10.10.0.0 0.0.255.255 area 0
end
write memory
```

## 5.2 Expected CORE1 OSPF Behavior

CORE1 should form an OSPF neighbor only with EDGE1 over G0/0.

```text
CORE1 G0/0 → non-passive
CORE1 VLAN SVIs → passive
```

Expected neighbor:

```text
Neighbor ID: 1.1.1.1
State      : FULL
Interface  : GigabitEthernet0/0
```

## 5.3 CORE1 Verification Commands

```cisco
show ip ospf neighbor
show ip ospf interface brief
show running-config | section router ospf
show ip route ospf
show standby brief
```

---

# 6. CORE2 Configuration

## 6.1 OSPF Single Area 0

```cisco
conf t
!
router ospf 1
 router-id 3.3.3.3
 passive-interface default
 no passive-interface GigabitEthernet0/0
 network 172.16.0.4 0.0.0.3 area 0
 network 10.10.0.0 0.0.255.255 area 0
end
write memory
```

## 6.2 Expected CORE2 OSPF Behavior

CORE2 should form an OSPF neighbor only with EDGE1 over G0/0.

```text
CORE2 G0/0 → non-passive
CORE2 VLAN SVIs → passive
```

Expected neighbor:

```text
Neighbor ID: 1.1.1.1
State      : FULL
Interface  : GigabitEthernet0/0
```

## 6.3 CORE2 Verification Commands

```cisco
show ip ospf neighbor
show ip ospf interface brief
show running-config | section router ospf
show ip route ospf
show standby brief
```

---

# 7. EDGE1 Configuration

## 7.1 OSPF Single Area 0

```cisco
conf t
!
router ospf 1
 router-id 1.1.1.1
 passive-interface default
 no passive-interface GigabitEthernet0/2
 no passive-interface GigabitEthernet0/3
 network 172.16.0.0 0.0.0.3 area 0
 network 172.16.0.4 0.0.0.3 area 0
end
write memory
```

## 7.2 Expected EDGE1 OSPF Behavior

EDGE1 should form two OSPF neighbors:

```text
EDGE1 G0/2 ↔ CORE1 G0/0
EDGE1 G0/3 ↔ CORE2 G0/0
```

Expected neighbors:

```text
Neighbor 2.2.2.2 → CORE1 → FULL
Neighbor 3.3.3.3 → CORE2 → FULL
```

## 7.3 EDGE1 OSPF Verification Commands

```cisco
show ip ospf neighbor
show ip ospf interface brief
show ip route ospf
show ip protocols
show running-config | section router ospf
```

---

# 8. Internal Static Route Removal on EDGE1

## 8.1 Original Internal Static Routes

Before OSPF was introduced, EDGE1 had static summary routes toward the internal campus network.

```cisco
ip route 10.10.0.0 255.255.0.0 172.16.0.2
ip route 10.10.0.0 255.255.0.0 172.16.0.6 10
```

After OSPF successfully learned the internal VLAN routes, these internal static routes were removed.

## 8.2 Remove Internal Static Routes

```cisco
conf t
no ip route 10.10.0.0 255.255.0.0 172.16.0.2
no ip route 10.10.0.0 255.255.0.0 172.16.0.6 10
end
write memory
```

## 8.3 Verification

```cisco
show running-config | include ^ip route 10.10
show ip route ospf
show ip route 10.10.40.0
```

Expected result:

```text
No static route for 10.10.0.0/16 should remain on EDGE1.
Internal VLAN routes should be learned through OSPF.
```

Expected OSPF-learned routes:

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

---

# 9. OSPF Default Route Advertisement on EDGE1

## 9.1 Purpose

EDGE1 advertises default route information into OSPF so CORE1 and CORE2 can receive default route information through OSPF.

## 9.2 Configuration

```cisco
conf t
router ospf 1
 default-information originate
end
write memory
```

## 9.3 Verification on EDGE1

```cisco
show running-config | section router ospf
show ip ospf database external
```

Expected result:

```text
Type-5 AS External LSA
Link State ID: 0.0.0.0
Advertising Router: 1.1.1.1
Network Mask: /0
Metric Type: 2
Metric: 1
```

## 9.4 Verification on CORE1 / CORE2

```cisco
show ip ospf database external
show ip route ospf
show ip route 0.0.0.0
```

Expected result:

```text
CORE1 and CORE2 should see the default route LSA from EDGE1.
```

Important note:

```text
If static default routes still exist on CORE1/CORE2,
the OSPF default route may appear in the OSPF database but not be installed in the routing table.

Static route AD = 1
OSPF external route AD = 110
```

---

# 10. Floating Static Default Route on EDGE1

## 10.1 Purpose

EDGE1 uses ISP1 as the primary default path and ISP2 as the backup default path.

## 10.2 Final Default Route Configuration

```cisco
conf t
ip route 0.0.0.0 0.0.0.0 100.64.1.1 track 1
ip route 0.0.0.0 0.0.0.0 100.64.2.1 10
end
write memory
```

## 10.3 Meaning

```text
Primary default route:
0.0.0.0/0 → 100.64.1.1 track 1

Backup floating default route:
0.0.0.0/0 → 100.64.2.1 AD 10
```

## 10.4 Verification

```cisco
show running-config | include ^ip route 0.0.0.0
show ip route 0.0.0.0
```

Expected normal result:

```text
Gateway of last resort is 100.64.1.1
Known via static
Track 1 Up
```

---

# 11. IP SLA Configuration on EDGE1

## 11.1 Purpose

IP SLA monitors the primary ISP path from EDGE1 toward INTERNET-SRV.

Target:

```text
203.0.113.10
```

Source interface:

```text
GigabitEthernet0/0
```

## 11.2 Configuration

```cisco
conf t
!
ip sla 1
 icmp-echo 203.0.113.10 source-interface GigabitEthernet0/0
 frequency 5
 threshold 1000
exit
!
ip sla schedule 1 life forever start-time now
!
track 1 ip sla 1 reachability
!
end
write memory
```

Important note:

```text
During the lab, timeout tuning was not modified after the IP SLA operation started.
The IP SLA entry was kept running because the primary goal was IP SLA + tracking behavior, not timer tuning.
```

## 11.3 Verification

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

---

# 12. OSPF Cost Manipulation on EDGE1

## 12.1 Purpose

After OSPF was enabled, EDGE1 initially learned internal VLAN routes through both CORE1 and CORE2 with equal cost.

To make CORE1 the preferred internal path and CORE2 the backup path, the OSPF cost toward CORE2 was increased.

## 12.2 Configuration

```cisco
conf t
interface GigabitEthernet0/3
 ip ospf cost 50
end
write memory
```

## 12.3 Final OSPF Cost Design

```text
EDGE1 G0/2 toward CORE1: OSPF cost 1
EDGE1 G0/3 toward CORE2: OSPF cost 50
```

## 12.4 Verification

```cisco
show ip ospf interface brief
show ip route ospf
show ip route 10.10.40.0
```

Expected result:

```text
EDGE1 prefers 172.16.0.2 through G0/2 for internal VLAN routes.
CORE2 through 172.16.0.6 remains available as a backup path.
```

---

# 13. ISP Return Path Route

## 13.1 Purpose

During ISP2 failover validation, EDGE1 could reach ISP2 but initially could not complete reachability to INTERNET-SRV.

The issue was the return path.

INTERNET-SRV uses ISP1 as its default gateway:

```text
203.0.113.1
```

Therefore, ISP1 needed a route back to the ISP2 WAN subnet.

## 13.2 ISP1 Configuration

```cisco
conf t
ip route 100.64.2.0 255.255.255.252 203.0.113.2
end
write memory
```

## 13.3 Verification from EDGE1 During ISP1 Failure

```cisco
ping 100.64.2.1
ping 203.0.113.2
ping 203.0.113.10 source GigabitEthernet0/1
```

Expected result:

```text
EDGE1 reaches ISP2 next-hop.
EDGE1 reaches ISP2 Internet-side address.
EDGE1 reaches INTERNET-SRV through ISP2 after return path correction.
```

---

# 14. NAT/PAT Baseline and Limitation

## 14.1 Current NAT/PAT Configuration on EDGE1

Current NAT/PAT uses ISP1 G0/0 for overload.

```cisco
ip nat inside source list ENTERPRISE-NAT interface GigabitEthernet0/0 overload
```

NAT inside interfaces:

```text
GigabitEthernet0/2
GigabitEthernet0/3
```

NAT outside interfaces:

```text
GigabitEthernet0/0
GigabitEthernet0/1
```

## 14.2 Verification Commands

```cisco
show running-config | include ip nat
show ip nat statistics
show ip nat translations
show running-config | include ^ip route 0.0.0.0
```

Observed behavior:

```text
Inside global address uses 100.64.1.2.
PAT overload is tied to G0/0 toward ISP1.
```

## 14.3 Limitation

Routing failover to ISP2 works, but NAT/PAT failover is not complete.

Reason:

```text
Default route can move to ISP2.
PAT overload remains tied to G0/0.
```

## 14.4 Decision

Do not fully implement NAT/PAT failover in Phase 03.

This topic is carried over to Phase 06 SD-WAN Underlay Enhancement.

Future Phase 06 topics:

```text
- Route-map based NAT
- ISP-specific NAT policy
- PBR + IP SLA
- DIA
- Local Internet Breakout
- Business intent path selection
```

---

# 15. HSRP Failover Validation Commands

## 15.1 VLAN40 HSRP Failure Test

VLAN40 was used to validate HSRP failover.

Original state:

```text
CORE1 = Active
CORE2 = Standby
VIP   = 10.10.40.254
```

## 15.2 Simulate VLAN40 Failure on CORE1

```cisco
conf t
interface Vlan40
 shutdown
end
```

## 15.3 Verify on CORE2

```cisco
show standby brief
```

Expected result:

```text
CORE2 becomes Active for VLAN40.
```

## 15.4 Verify from MGMT-SRV1

```bash
ping -c 4 10.10.40.254
ping -c 4 172.16.0.1
ping -c 4 203.0.113.10
```

Expected result:

```text
MGMT-SRV1 can still reach VLAN40 VIP.
MGMT-SRV1 can still reach EDGE1.
MGMT-SRV1 can still reach INTERNET-SRV.
```

## 15.5 Restore CORE1 VLAN40

```cisco
conf t
interface Vlan40
 no shutdown
end
write memory
```

## 15.6 Verify Recovery

```cisco
show standby brief
```

Expected result:

```text
CORE1 returns as Active for VLAN40 if preempt is configured.
CORE2 returns to Standby.
```

---

# 16. Failure and Recovery Test Commands

## 16.1 Test 1 — ISP1 Failure and Recovery

### Failure

On EDGE1:

```cisco
conf t
interface GigabitEthernet0/0
 shutdown
end
```

### Verify Failover

```cisco
show track 1
show ip route 0.0.0.0
show ip sla summary
ping 203.0.113.10 source GigabitEthernet0/1
```

Expected result:

```text
Track 1 Down
Default route changes to 100.64.2.1
Ping to INTERNET-SRV succeeds through ISP2 after return path correction
```

### Recovery

```cisco
conf t
interface GigabitEthernet0/0
 no shutdown
end
```

### Verify Recovery

```cisco
show track 1
show ip route 0.0.0.0
ping 203.0.113.10
```

Expected result:

```text
Track 1 Up
Default route returns to 100.64.1.1
Ping to INTERNET-SRV succeeds through ISP1
```

---

## 16.2 Test 2 — CORE1 Uplink Failure and Recovery

### Failure

On CORE1:

```cisco
conf t
interface GigabitEthernet0/0
 shutdown
end
```

### Verify on EDGE1

```cisco
show ip ospf neighbor
show ip route 10.10.40.0
show ip route ospf
```

Expected result:

```text
CORE1 OSPF neighbor goes down.
EDGE1 uses CORE2 path for internal VLAN routes.
```

Important observation:

```text
HSRP does not automatically fail over if the VLAN SVI remains active.
This can create traffic blackholing if the HSRP Active core loses its routed uplink.
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

### Verify Recovery

```cisco
show ip ospf neighbor
show ip route ospf
```

---

## 16.3 Test 3 — HSRP VLAN40 Failure and Recovery

### Failure

On CORE1:

```cisco
conf t
interface Vlan40
 shutdown
end
```

### Verify on CORE2

```cisco
show standby brief
```

Expected result:

```text
CORE2 becomes Active for VLAN40.
```

### Verify from MGMT-SRV1

```bash
ping -c 4 10.10.40.254
ping -c 4 172.16.0.1
ping -c 4 203.0.113.10
```

Expected result:

```text
Management gateway remains reachable.
Internet reachability remains available.
```

### Recovery

On CORE1:

```cisco
conf t
interface Vlan40
 no shutdown
end
write memory
```

### Verify Recovery

```cisco
show standby brief
```

---

# 17. Final Save Commands

After completing Single Area OSPF validation, save the configuration on all Cisco devices.

## 17.1 CORE1

```cisco
write memory
```

## 17.2 CORE2

```cisco
write memory
```

## 17.3 EDGE1

```cisco
write memory
```

## 17.4 ISP1

```cisco
write memory
```

## 17.5 ISP2

```cisco
write memory
```

---

# 18. Final Verification Commands

## 18.1 EDGE1

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

## 18.2 CORE1

```cisco
show ip interface brief
show ip ospf neighbor
show ip ospf interface brief
show ip route ospf
show standby brief
show running-config | section router ospf
```

## 18.3 CORE2

```cisco
show ip interface brief
show ip ospf neighbor
show ip ospf interface brief
show ip route ospf
show standby brief
show running-config | section router ospf
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

---

# 19. Single Area OSPF Config Completion Status

The following configurations were applied and validated:

```text
OSPF Single Area 0
OSPF passive-interface design
OSPF neighbor formation
OSPF internal route learning
EDGE1 internal static route removal
OSPF default route advertisement
Floating static default route
IP SLA
Object tracking
WAN failover and recovery
OSPF cost manipulation
HSRP VLAN40 failover validation
ISP2 return path correction
```

The following configurations were intentionally not implemented in this case:

```text
Route Summarization:
- Not implemented in Single Area OSPF
- Deferred to Multi-Area OSPF

NAT/PAT Failover:
- Limitation identified
- Full implementation deferred to Phase 06 SD-WAN Underlay Enhancement
```

Next configuration step:

```text
Reconfigure OSPF from Single Area to Multi-Area and implement route summarization.
```
