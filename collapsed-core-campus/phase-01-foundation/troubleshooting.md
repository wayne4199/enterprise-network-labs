# Phase 01 - Foundation Troubleshooting


# Troubleshooting Methodology

1. Identify Failure Domain
2. Verify Physical Status
3. Verify VLAN/Trunk Status
4. Verify STP State
5. Verify Gateway Status
6. Verify End-to-End Connectivity
7. Validate Failover Recovery

---

# Scenario 1 — Trunk Failure

## Symptom

- End host connectivity loss
- VLAN communication interruption

## Verification Commands - show interfaces trunk / show interfaces status

Expected Findings
- Trunk interface is down
- VLAN forwarding path affected

# Scenario 2 — STP Failure

## Test Performed

Shutdown forwarding uplink on ACC1:

interface g0/0

shutdown

## Verification Command - show spanning-tree vlan 10

## Expected Result

Before failure:
- Gi0/0 Root FWD
- Gi0/1 Altn BLK

After failure:
- Gi0/0 Down
- Gi0/1 Root FWD

## Analysis

Rapid-PVST successfully reconverged and promoted the alternate uplink to forwarding state.


# Scenario 3 — HSRP Failover

## Test Performed

Shutdown active SVI on CORE1:

interface vlan 10

shutdown

## Verification Command - show standby brief

## Expected Result

Before failure:
- CORE1 = Active
- CORE2 = Standby

After failure:
- CORE1 = Down
- CORE2 = Active

## Analysis

HSRP failover successfully maintained default gateway availability for VLAN 10 hosts.


# Scenario 4 — Inter-VLAN Routing Failure

## Symptom

- Hosts within same VLAN communicate successfully
- Inter-VLAN communication fails

## Verification Command - show ip route / show ip interface brief

## Posible Causes

- Missing SVI
- SVI administratively down
- Missing ip routing
- Incorrect default gateway


# Scenario 5 — Host Connectivity Failure

## Symptom

- Host cannot reach gateway

## Verification Command

On Host - ip addr / ip route / ping <gateway>

On Switch - show vlan brief / show interfaces status

## Posible Causes

- Incorrect VLAN assignment
- Missing default route
- Access port shutdown
- STP convergence event


# Key Lessons Learned

- STP provides Layer 2 loop prevention and failover
- HSRP provides gateway redundancy
- VLAN separation creates distinct broadcast domains
- Inter-VLAN routing enables Layer 3 communication
- Dual-homed access design improves resiliency



