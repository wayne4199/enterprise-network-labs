# Phase 03 — Routing Resiliency Ver.2 Configs

## 1. Purpose

This document records the configuration changes applied during **Phase 03 — Routing Resiliency Ver.2**.

Phase 03 covers two OSPF cases:

```text
Case 1 — Single Area OSPF
Case 2 — Multi-Area OSPF
```

This file includes:

```text
- Single Area OSPF configuration
- Multi-Area OSPF configuration
- Route summarization configuration
- OSPF cost manipulation
- Default route advertisement
- Floating static route
- IP SLA and object tracking
- WAN resiliency configuration
- HSRP failover validation commands
- NAT/PAT limitation notes
```

The following recovery items are intentionally excluded from this document:

```text
- VLAN Database loss recovery
- MGMT-SRV1 netplan loss recovery
- INTERNET-SRV IP/Gateway loss recovery
```

---

## 2. Configuration Scope

Phase 03 includes the following configuration areas:

```text
1. Single Area OSPF
2. Multi-Area OSPF
3. Passive Interface Design
4. OSPF Default Route Advertisement
5. Internal Static Route Removal on EDGE1
6. Floating Static Route
7. IP SLA
8. Object Tracking
9. WAN Resiliency
10. OSPF Cost Manipulation
11. Route Summarization with area range
12. HSRP Failover Validation
13. NAT/PAT Failover Limitation
```

---

## 3. Important Baseline Preservation

Phase 03 must preserve the Phase 01 and Phase 02 baseline.

Do not remove or redesign the following:

```text
- VLAN design
- SVI IP addressing
- HSRP VIPs
- Rapid-PVST
- STP root and HSRP active role alignment
- VLAN999 native blackhole
- MGMT-SRV1 Enterprise Management Server role
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

# Case 1 — Single Area OSPF

## 5. CORE1 Single Area OSPF Configuration

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

### Expected CORE1 Behavior

```text
CORE1 G0/0       = OSPF Area 0, non-passive
CORE1 VLAN SVIs  = OSPF Area 0, passive
OSPF neighbor    = EDGE1 only
```

### CORE1 Verification Commands

```cisco
show ip ospf neighbor
show ip ospf interface brief
show running-config | section router ospf
show ip route ospf
show standby brief
```

---

## 6. CORE2 Single Area OSPF Configuration

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

### Expected CORE2 Behavior

```text
CORE2 G0/0       = OSPF Area 0, non-passive
CORE2 VLAN SVIs  = OSPF Area 0, passive
OSPF neighbor    = EDGE1 only
```

### CORE2 Verification Commands

```cisco
show ip ospf neighbor
show ip ospf interface brief
show running-config | section router ospf
show ip route ospf
show standby brief
```

---

## 7. EDGE1 Single Area OSPF Configuration

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

### Expected EDGE1 Behavior

```text
EDGE1 G0/2 = OSPF Area 0, non-passive, neighbor with CORE1
EDGE1 G0/3 = OSPF Area 0, non-passive, neighbor with CORE2
EDGE1 G0/0 = ISP1-facing, not in OSPF
EDGE1 G0/1 = ISP2-facing, not in OSPF
```

### EDGE1 Verification Commands

```cisco
show ip ospf neighbor
show ip ospf interface brief
show ip route ospf
show ip protocols
show running-config | section router ospf
```

---

## 8. EDGE1 Internal Static Route Removal

Before OSPF was introduced, EDGE1 used static summary routes toward the internal campus network.

Original routes:

```cisco
ip route 10.10.0.0 255.255.0.0 172.16.0.2
ip route 10.10.0.0 255.255.0.0 172.16.0.6 10
```

After OSPF successfully learned the internal VLAN routes, these internal static routes were removed.

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
No static route for 10.10.0.0/16 should remain on EDGE1.
Internal VLAN routes should be learned through OSPF.
```

---

## 9. OSPF Default Route Advertisement on EDGE1

EDGE1 advertises default route information into OSPF.

```cisco
conf t
router ospf 1
 default-information originate
end
write memory
```

### Verification on EDGE1

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

### Verification on CORE1 / CORE2

```cisco
show ip ospf database external
show ip route 0.0.0.0
```

Important note:

```text
If static default routes still exist on CORE1/CORE2,
the OSPF default route may appear in the OSPF database but not be installed in the routing table.

Static route AD = 1
OSPF external route AD = 110
```

---

## 10. Floating Static Default Route on EDGE1

EDGE1 uses ISP1 as the primary default path and ISP2 as the backup default path.

Final default route configuration:

```cisco
conf t
ip route 203.0.113.10 255.255.255.255 100.64.1.1
ip route 0.0.0.0 0.0.0.0 100.64.1.1 track 1
ip route 0.0.0.0 0.0.0.0 100.64.2.1 10
end
write memory
```

### Meaning

```text
203.0.113.10/32 → 100.64.1.1
- Forces IP SLA target checking through ISP1

Primary default route:
0.0.0.0/0 → 100.64.1.1 track 1

Backup floating default route:
0.0.0.0/0 → 100.64.2.1 AD 10
```

### Verification

```cisco
show running-config | include ^ip route
show ip route 203.0.113.10
show ip route 0.0.0.0
```

Expected normal result:

```text
Host route to 203.0.113.10 via 100.64.1.1 exists.
Track 1 is Up.
Default route uses 100.64.1.1.
Backup default route via 100.64.2.1 remains available.
```

---

## 11. IP SLA Configuration on EDGE1

IP SLA monitors the primary ISP path from EDGE1 toward INTERNET-SRV.

Target:

```text
203.0.113.10
```

Source interface:

```text
GigabitEthernet0/0
```

Configuration:

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

### Verification

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

## 12. Single Area OSPF Cost Manipulation on EDGE1

To make CORE1 the preferred internal path and CORE2 the backup path, the OSPF cost toward CORE2 was increased.

Configuration:

```cisco
conf t
interface GigabitEthernet0/3
 ip ospf cost 50
end
write memory
```

Final cost design:

```text
EDGE1 G0/2 toward CORE1: OSPF cost 1
EDGE1 G0/3 toward CORE2: OSPF cost 50
```

Verification:

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

## 13. ISP Return Path Route

During ISP2 failover validation, return traffic from the Internet segment must be able to reach the ISP2 WAN subnet.

ISP1 configuration:

```cisco
conf t
ip route 100.64.2.0 255.255.255.252 203.0.113.2
end
write memory
```

### Verification from EDGE1 During ISP1 Failure

```cisco
ping 100.64.2.1
ping 203.0.113.2
ping 203.0.113.10 source GigabitEthernet0/1
```

Expected result:

```text
EDGE1 reaches ISP2 next-hop.
EDGE1 reaches ISP2 Internet-side address.
EDGE1 reaches INTERNET-SRV through ISP2.
```

---

# Case 2 — Multi-Area OSPF

## 14. Remove Single Area OSPF Before Multi-Area Reconfiguration

Before applying Multi-Area OSPF, the Single Area OSPF configuration was removed.

### CORE1

```cisco
conf t
no router ospf 1
end
write memory
```

### CORE2

```cisco
conf t
no router ospf 1
end
write memory
```

### EDGE1

```cisco
conf t
no router ospf 1
interface GigabitEthernet0/3
 no ip ospf cost 50
end
write memory
```

### Verification

```cisco
show running-config | section router ospf
show ip ospf neighbor
show ip route ospf
show running-config interface GigabitEthernet0/3
```

Expected result:

```text
router ospf 1 removed.
OSPF neighbors removed.
OSPF routes removed.
EDGE1 G0/3 ip ospf cost 50 removed before Multi-Area reconfiguration.
```

---

## 15. CORE1 Multi-Area OSPF Configuration

CORE1 acts as an ABR.

Area design:

```text
CORE1 G0/0       = Area 0
CORE1 VLAN SVIs  = Area 10
```

Configuration:

```cisco
conf t
!
router ospf 1
 router-id 2.2.2.2
 passive-interface default
 no passive-interface GigabitEthernet0/0
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

### Verification

```cisco
show ip ospf
show ip ospf neighbor
show ip ospf interface brief
show ip ospf border-routers
show running-config | section router ospf
```

Expected result:

```text
CORE1 is an area border router.
Area BACKBONE(0) exists.
Area 10 exists.
G0/0 is in Area 0.
VLAN SVIs are in Area 10.
VLAN SVIs are passive.
```

---

## 16. CORE2 Multi-Area OSPF Configuration

CORE2 acts as an ABR.

Area design:

```text
CORE2 G0/0       = Area 0
CORE2 VLAN SVIs  = Area 10
```

Configuration:

```cisco
conf t
!
router ospf 1
 router-id 3.3.3.3
 passive-interface default
 no passive-interface GigabitEthernet0/0
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

### Verification

```cisco
show ip ospf
show ip ospf neighbor
show ip ospf interface brief
show ip ospf border-routers
show running-config | section router ospf
```

Expected result:

```text
CORE2 is an area border router.
Area BACKBONE(0) exists.
Area 10 exists.
G0/0 is in Area 0.
VLAN SVIs are in Area 10.
VLAN SVIs are passive.
```

---

## 17. EDGE1 Multi-Area OSPF Configuration

EDGE1 remains only in Area 0.

Area design:

```text
EDGE1 G0/2 to CORE1 = Area 0
EDGE1 G0/3 to CORE2 = Area 0
EDGE1 G0/0 to ISP1  = Not in OSPF
EDGE1 G0/1 to ISP2  = Not in OSPF
```

Configuration:

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
 default-information originate
end
write memory
```

### Verification

```cisco
show ip ospf neighbor
show ip ospf interface brief
show ip route ospf
show ip ospf database summary
show ip ospf database external
show running-config | section router ospf
```

Expected result before summarization:

```text
EDGE1 neighbors:
2.2.2.2 FULL
3.3.3.3 FULL

EDGE1 route table:
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

---

## 18. Route Summarization on CORE1 and CORE2

After validating individual `O IA` routes on EDGE1, route summarization was applied on CORE1 and CORE2.

### CORE1

```cisco
conf t
router ospf 1
 area 10 range 10.10.0.0 255.255.0.0
end
write memory
```

### CORE2

```cisco
conf t
router ospf 1
 area 10 range 10.10.0.0 255.255.0.0
end
write memory
```

### Verification on EDGE1

```cisco
show ip route ospf
show ip route 10.10.0.0
show ip ospf database summary
```

Expected result after summarization:

```text
O IA 10.10.0.0/16
```

Expected OSPF database summary LSA:

```text
Link State ID: 10.10.0.0
Network Mask: /16

Advertising Router:
2.2.2.2
3.3.3.3
```

The individual `O IA 10.10.x.0/24` routes should no longer appear on EDGE1.

---

## 19. Multi-Area OSPF Cost Manipulation on EDGE1

After summarization, EDGE1 initially had two equal summary paths through CORE1 and CORE2.

To keep CORE1 as the preferred internal path, OSPF cost was applied again on EDGE1 G0/3.

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

CORE2 remains available as the backup path.

---

# 20. WAN Resiliency Configuration and Validation Commands

## 20.1 Normal State Verification

On EDGE1:

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

---

## 20.2 ISP1 Failure Test

On EDGE1:

```cisco
conf t
interface GigabitEthernet0/0
 shutdown
end
```

Wait 10 to 15 seconds.

Verify:

```cisco
show track 1
show ip sla summary
show ip route 0.0.0.0
```

Expected result:

```text
Track 1 Down
IP SLA Timeout
Default route via 100.64.2.1
```

---

## 20.3 ISP2 Path Verification

On EDGE1:

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

---

## 20.4 ISP1 Recovery Test

On EDGE1:

```cisco
conf t
interface GigabitEthernet0/0
 no shutdown
end
```

Wait 10 to 15 seconds.

Verify:

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

---

# 21. HSRP Failover Validation Commands

## 21.1 VLAN40 HSRP Initial State

On CORE1 and CORE2:

```cisco
show standby brief
```

Expected result:

```text
VLAN40:
CORE1 = Active
CORE2 = Standby
VIP   = 10.10.40.254
```

---

## 21.2 Simulate VLAN40 Failure on CORE1

On CORE1:

```cisco
conf t
interface Vlan40
 shutdown
end
```

Verify on CORE2:

```cisco
show standby brief
```

Expected result:

```text
CORE2 becomes Active for VLAN40.
VIP 10.10.40.254 remains available.
```

Verify from MGMT-SRV1:

```bash
ping -c 4 10.10.40.254
ping -c 4 172.16.0.1
ping -c 4 203.0.113.10
```

Important note:

```text
HSRP failover itself should occur.
However, with route summarization, upstream return traffic may still prefer CORE1 for 10.10.0.0/16.
If CORE1 Vlan40 is down, return traffic can be blackholed.
This is a summarization and partial SVI failure design limitation.
```

---

## 21.3 Restore CORE1 VLAN40

On CORE1:

```cisco
conf t
interface Vlan40
 no shutdown
end
write memory
```

Verify on CORE1 and CORE2:

```cisco
show standby brief
```

Expected result:

```text
CORE1 returns as Active for VLAN40 if preempt is configured.
CORE2 returns to Standby.
```

---

# 22. CORE1 Uplink Failure and Recovery Test

## 22.1 Failure

On CORE1:

```cisco
conf t
interface GigabitEthernet0/0
 shutdown
end
```

Wait 10 to 20 seconds.

Verify on EDGE1:

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

Metric becomes 51 because EDGE1 G0/3 OSPF cost is 50.
```

---

## 22.2 Recovery

On CORE1:

```cisco
conf t
interface GigabitEthernet0/0
 no shutdown
end
write memory
```

Wait 10 to 20 seconds.

Verify on EDGE1:

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

---

# 23. NAT/PAT Baseline and Limitation

## 23.1 Current NAT/PAT Configuration on EDGE1

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

## 23.2 Verification Commands

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

## 23.3 Limitation

Routing failover to ISP2 works, but NAT/PAT failover is not complete.

Reason:

```text
Default route can move to ISP2.
PAT overload remains tied to G0/0.
```

## 23.4 Decision

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

# 24. Final Save Commands

After completing Phase 03 validation, save the configuration on all Cisco devices.

## CORE1

```cisco
write memory
```

## CORE2

```cisco
write memory
```

## EDGE1

```cisco
write memory
```

## ISP1

```cisco
write memory
```

## ISP2

```cisco
write memory
```

---

# 25. Final Verification Commands

## 25.1 EDGE1

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

Expected final EDGE1 state:

```text
OSPF neighbor with CORE1: FULL
OSPF neighbor with CORE2: FULL
O IA 10.10.0.0/16 via 172.16.0.2
Default route via 100.64.1.1 when Track 1 is Up
Track 1 Up
IP SLA OK
default-information originate configured
```

---

## 25.2 CORE1

```cisco
show ip interface brief
show ip ospf
show ip ospf neighbor
show ip ospf interface brief
show ip ospf border-routers
show standby brief
show running-config | section router ospf
```

Expected final CORE1 state:

```text
CORE1 is an ABR.
Area 0 and Area 10 exist.
G0/0 is in Area 0.
VLAN SVIs are in Area 10.
VLAN SVIs are passive.
OSPF neighbor with EDGE1 is FULL.
VLAN40 is Active.
area 10 range 10.10.0.0 255.255.0.0 is configured.
```

---

## 25.3 CORE2

```cisco
show ip interface brief
show ip ospf
show ip ospf neighbor
show ip ospf interface brief
show ip ospf border-routers
show standby brief
show running-config | section router ospf
```

Expected final CORE2 state:

```text
CORE2 is an ABR.
Area 0 and Area 10 exist.
G0/0 is in Area 0.
VLAN SVIs are in Area 10.
VLAN SVIs are passive.
OSPF neighbor with EDGE1 is FULL.
VLAN40 is Standby.
area 10 range 10.10.0.0 255.255.0.0 is configured.
```

---

## 25.4 MGMT-SRV1

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

Expected final MGMT-SRV1 state:

```text
ens2 = 10.10.40.10/24
default via 10.10.40.254
VLAN40 gateway reachable
EDGE1 reachable
INTERNET-SRV reachable
campus.lab DNS static records resolve correctly
```

---

# 26. Phase 03 Config Completion Status

The following configurations were applied and validated:

```text
Single Area OSPF
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
Multi-Area OSPF
ABR configuration
Area 0 / Area 10 separation
Inter-area route learning
Route summarization with area range
Summary route verification
Multi-Area OSPF cost manipulation
HSRP VLAN40 failover validation
CORE1 uplink failure and recovery validation
```

The following configuration was intentionally not implemented in this phase:

```text
NAT/PAT Failover:
- Limitation identified
- Full implementation deferred to Phase 06 SD-WAN Underlay Enhancement
```

Next phase:

```text
Phase 04 — Security
```
