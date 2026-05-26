# Design Decisions — Phase 05 Enterprise QoS

# 1. QoS Focused on EDGE1

QoS policies were intentionally implemented primarily on EDGE1.

This design reflects real Enterprise WAN architecture where:
- WAN edge routers are common congestion points
- shaping and queueing are most meaningful at WAN egress interfaces
- WAN edge devices enforce traffic engineering policies

The campus core was intentionally left mostly untouched because:
- core links are high-speed
- congestion probability is low
- preserving forwarding performance is preferred

---

# 2. Enterprise WAN Edge Design Philosophy

The topology intentionally follows this QoS model:

| Layer | QoS Responsibility |
|---|---|
| Access Layer | Classification / Trust Boundary |
| Core Layer | Preserve DSCP |
| WAN Edge | Queueing / Shaping / Congestion Handling |

This design mirrors:
- Enterprise WAN
- MPLS WAN
- Internet WAN
- SD-WAN WAN Edge architectures

---

# 3. MQC-Based QoS Design

MQC (Modular QoS CLI) was intentionally selected because it:
- aligns with modern Cisco QoS architecture
- supports modular traffic engineering
- scales into Enterprise WAN and SD-WAN environments
- integrates naturally with HQoS

The lab intentionally standardized around:
- `class-map`
- `policy-map`
- `service-policy`

rather than legacy QoS models.

---

# 4. DSCP-Based Traffic Classification

DSCP-based classification was intentionally used instead of:
- application inspection
- NBAR
- DPI

because this phase focused on:
- packet-level QoS behavior
- WAN queue mechanics
- transport-level traffic engineering

This also simplified:
- traffic generation
- verification
- operational troubleshooting

---

# 5. Voice Traffic Design

Voice traffic was intentionally mapped to:
- DSCP EF (46)
- LLQ using `priority percent`

This reflects real Enterprise WAN QoS design where:
- voice requires minimal latency
- voice requires jitter reduction
- voice traffic must receive strict priority forwarding

The design intentionally demonstrated:
- LLQ protection
- priority queue policing
- exceed drops
- congestion behavior

---

# 6. Video Traffic Design

Video traffic was intentionally mapped to:
- DSCP AF41
- CBWFQ bandwidth reservation

Video was intentionally not placed into LLQ because:
- video traffic consumes significantly more bandwidth
- unrestricted priority queues can starve other traffic classes
- CBWFQ better reflects practical Enterprise WAN design

---

# 7. Scavenger Traffic Design

Scavenger traffic handling was intentionally implemented using:
- DSCP CS1
- re-marking policies

This reflects real Enterprise WAN policies where:
- backup traffic
- bulk synchronization
- low-priority applications
- undesirable traffic

are intentionally deprioritized.

The lab intentionally demonstrated:
- trust boundary enforcement
- DSCP rewriting
- low-priority queue treatment

---

# 8. Hierarchical QoS (HQoS)

Hierarchical QoS was intentionally implemented using:
- parent shaping policy
- child queueing policy

Structure:

```text
WAN-SHAPE
 └─ CAMPUS-QOS
