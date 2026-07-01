# Phase 07 — SD-WAN Overlay Foundation & VPN Segmentation

## 1. Phase Overview

This phase builds the first working Cisco Catalyst SD-WAN overlay fabric on top of the SD-WAN-ready underlay prepared in Phase 06.

The main goal of this phase is not advanced policy yet.  
The goal is to bring up the SD-WAN control plane, onboard HQ and Branch WAN Edges, form secure overlay tunnels, exchange OMP routes, verify BFD data-plane tunnels, and confirm basic multi-VPN segmentation.

This phase validates the transition from a traditional routed underlay to an SD-WAN overlay fabric.

---

## 2. Phase Objectives

The objectives of Phase 07 are:

1. Understand Cisco Catalyst SD-WAN architecture
2. Understand the role of each controller
   - SDWAN-MGR
   - SDWAN-CTRL
   - SDWAN-VALID
3. Build VPN 0 transport connectivity
4. Prepare VPN 512 / management access conceptually
5. Bring up SD-WAN controllers
6. Onboard HQ WAN Edge
7. Onboard Branch WAN Edge
8. Validate SD-WAN control connections
9. Validate OMP peer relationships
10. Understand TLOC and Color
11. Validate BFD overlay tunnels
12. Build Service VPN 10 communication
13. Build VPN 20 segmentation test
14. Confirm multi-VPN segmentation behavior

---

## 3. Final Lab Status

Phase 07 was completed successfully.

The final validated state is:

```text
Controller Bring-up:        PASS
HQ Edge Onboarding:         PASS
Branch Edge Onboarding:     PASS
Control Connections:        PASS
OMP Peering:                PASS
TLOC Advertisement:         PASS
BFD Overlay Tunnels:        PASS
VPN 10 Overlay Routing:     PASS
VPN 20 Segmentation:        PASS
HQ ↔ Branch Reachability:   PASS
Validator Edge Validity:    PASS
vManage GUI Visibility:     PASS
