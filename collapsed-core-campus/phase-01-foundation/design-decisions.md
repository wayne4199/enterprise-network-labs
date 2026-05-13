# Phase 01 - Foundation Design Decisions

## Key Decisions

- Use Access + Collapsed Core architecture
- Use dual core switches
- Use dual-homed access switches
- Use Rapid-PVST
- Align STP root placement with HSRP active gateway
- Use VLAN 999 as native blackhole VLAN

---

## Why Collapsed Core Architecture?

Collapsed core architecture provides:

- Simpler operations
- Lower management overhead
- Reduced failure domains
- Suitable design for small to medium enterprise campuses

---

## Why Dual Core Switches?

Using dual core switches provides:

- Gateway redundancy
- STP resiliency
- High availability
- Traffic distribution

---

## Why Dual-Homed Access Switches?

Each access switch connects to both core switches to provide:

- Redundant uplinks
- Faster failover
- Reduced outage impact

---

## Why Rapid-PVST?

Rapid-PVST provides:

- Faster convergence
- VLAN-based STP control
- Better campus resiliency

---

## Why Align STP Root with HSRP Active Gateway?

Aligning STP Root Bridge and HSRP Active Gateway:

- Optimizes forwarding path
- Reduces asymmetric traffic flow
- Improves campus efficiency

---

## Why VLAN 999 as Native VLAN?

Using an unused native VLAN improves security by:

- Reducing VLAN hopping risk
- Preventing untagged traffic leakage
- Isolating unused traffic


