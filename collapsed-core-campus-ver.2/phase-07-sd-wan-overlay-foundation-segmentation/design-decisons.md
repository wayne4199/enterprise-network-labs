# Phase 07 — Design Decisions

## 1. Design Purpose

Phase 07 was designed to build the first working Cisco Catalyst SD-WAN overlay fabric on top of the SD-WAN-ready underlay prepared in Phase 06.

The main design goal was not advanced policy.  
The goal was to prove the SD-WAN overlay foundation:

- Controller bring-up
- WAN Edge onboarding
- Certificate-based trust
- Control connections
- OMP peering
- TLOC advertisement
- BFD overlay tunnels
- Service VPN routing
- Multi-VPN segmentation

This phase intentionally focused on the minimum stable SD-WAN fabric required before moving to centralized policy, application-aware routing, DIA, service chaining, or operations monitoring.

---

## 2. Controller Design Decision

### Decision

Use three dedicated SD-WAN controller nodes:

| Device | Role |
|---|---|
| SDWAN-MGR | vManage / Manager |
| SDWAN-CTRL | vSmart / Controller |
| SDWAN-VALID | vBond / Validator |

### Reason

Cisco Catalyst SD-WAN separates the management plane, control plane, and orchestration function.

Using separate controller nodes allowed the lab to clearly validate each role:

- SDWAN-MGR for GUI management, device list, bootstrap generation, and certificate operations
- SDWAN-CTRL for OMP route exchange and TLOC control-plane learning
- SDWAN-VALID for WAN Edge authentication and orchestration

### Result

This design made it possible to independently verify:

```text
show sdwan control connections
show omp peers
show omp routes vpn 10
show omp routes vpn 20
show orchestrator connections
show orchestrator valid-vedges
