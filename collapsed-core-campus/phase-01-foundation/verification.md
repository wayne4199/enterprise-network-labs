# Phase 01 - Foundation Verification

# Verification Flow

1. VLAN Verification
2. Trunk Verification
3. STP Verification
4. Inter-VLAN Routing Verification
5. HSRP Verification
6. End-to-End Connectivity Verification
7. Failover Verification

---

# STEP 1 — VLAN Verification

## Command - show vlan brief

Expected Result
- VLAN 10 USERS exists
- VLAN 20 SERVERS exists
- VLAN 40 MGMT exists
- VLAN 999 NATIVE-BLACKHOLE exists

# STEP 2 — Trunk Verification

## Command - show interfaces trunk

Expected Result
- All uplinks operate as trunk ports
- Native VLAN is 999
- VLANs 10,20,40 allowed on trunks

# STEP 3 — STP Verification

## Command - show spanning-tree vlan X / show spanning-tree root

CORE1 Expected Result
- VLAN 10
  - Root Primary
  - All uplinks = Desg FWD

- VLAN 20
  - Root Secondary

CORE2 Expected Result
- VLAN 10
  - Root Secondary

- VLAN 20
  - Root Primary
  - All uplinks = Desg FWD

ACC1 Expected Result
- VLAN 10
  - Gi0/0 Root FWD
  - Gi0/1 Altn BLK
  - Gi0/2 Desg FWD
  - Gi0/3 Desg FWD

- VLAN 20
  - Gi0/0 Altn BLK
  - Gi0/1 Root FWD

ACC2 Expected Result
- VLAN 10
  - Gi0/0 Root FWD
  - Gi0/1 Altn BLK
 
- VLAN 20
  - Gi0/0 Altn BLK
  - Gi0/1 Root FWD
  - Gi0/2 Desg FWD

ACC3 Expected Result
- VLAN 10
  - Gi0/0 Root FWD
  - Gi0/1 Altn BLK
 
- VLAN 20
  - Gi0/0 Altn BLK
  - Gi0/1 Root FWD
    

# STEP 4 — Inter-VLAN Routing Verification

## Command - show ip interface brief / show ip route

Expected Result
- VLAN interfaces are UP/UP
- Connected routes exist for VLAN networks
- Inter-VLAN ping succeeds


# STEP 5 — HSRP Verification

## Command - show standby brief

CORE1 Expected Result
- VLAN 10 = Active
- VLAN 20 = Standby
- VLAN 40 = Active

CORE2 Expected Result
- VLAN 10 = Standby
- VLAN 20 = Active
- VLAN 40 = Standby

# STEP 6 — End-to-End Connectivity Verification

## Tests

Expected Result
- All inter-VLAN communication succeeds

USER1
- ping 10.10.10.12
- ping 10.20.20.11
- ping 10.40.40.11

SERVER1
- ping 10.10.10.11

MGMT1
- ping 10.20.20.11

# STEP 7 — Failover Verification

## STP Failover Test

Shutdown forwarding uplink on ACC1:

interface g0/0

shutdown

Expected Result
- STP reconverges successfully
- Connectivity remains operational
- Gi0/0 Root FWD --> Down
- Gi0/1 Altn BLK --> Root FWD

## HSRP Failover Test

Shutdown active VLAN SVI on CORE1:

interface vlan 10

shutdown

Expected Result
- CORE2 becomes Active HSRP gateway
- Gateway redundancy operates successfully
- End host connectivity remains operational

