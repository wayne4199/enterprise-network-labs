# Phase 05 — Enterprise QoS
# configs.md

# 1. QoS Classification ACL

```cisco
ip access-list extended ICMP-VOICE
 permit icmp any any
```

Purpose:
- Simulate scavenger traffic classification
- Demonstrate ACL-based QoS classification
- Support DSCP re-marking scenarios

---

# 2. Voice Class Map

```cisco
class-map match-any VOICE
 match dscp ef
```

Purpose:
- Match voice traffic
- Classify EF-marked packets
- Feed LLQ policy

Traffic Type:
- RTP Voice
- Real-time traffic

DSCP:
- EF (46)

---

# 3. Video Class Map

```cisco
class-map match-any VIDEO
 match dscp af41
```

Purpose:
- Match video traffic
- Separate multimedia traffic from voice

Traffic Type:
- Video conferencing
- Multimedia applications

DSCP:
- AF41 (34)

---

# 4. Scavenger Class Map

```cisco
class-map match-any SCAVENGER
 match access-group name ICMP-VOICE
```

Purpose:
- Match low-priority traffic
- Demonstrate re-marking and scavenger handling

Traffic Type:
- ICMP
- Bulk traffic simulation
- Low-priority traffic

---

# 5. Main QoS Policy

```cisco
policy-map CAMPUS-QOS

 class VOICE
  priority percent 10

 class VIDEO
  bandwidth percent 20

 class SCAVENGER
  set dscp cs1

 class class-default
  fair-queue
```

Purpose:
- Enterprise WAN edge QoS policy
- Multi-class traffic engineering
- WAN congestion management

---

# 6. Voice LLQ Design

```cisco
class VOICE
 priority percent 10
```

Purpose:
- Strict priority queue
- Low latency forwarding
- Jitter reduction
- Real-time voice protection

Behavior:
- LLQ enabled
- Implicit policer enabled
- Exceed traffic may drop

---

# 7. Video CBWFQ Design

```cisco
class VIDEO
 bandwidth percent 20
```

Purpose:
- Guaranteed bandwidth
- Controlled multimedia handling
- Non-priority queueing

Behavior:
- CBWFQ queue
- No strict priority
- Queue drops possible during congestion

---

# 8. Scavenger Re-marking

```cisco
class SCAVENGER
 set dscp cs1
```

Purpose:
- Trust boundary enforcement
- Low-priority traffic handling
- QoS abuse prevention

Behavior:
- Re-mark traffic to CS1
- Less-than-best-effort treatment

---

# 9. Default Queue Handling

```cisco
class class-default
 fair-queue
```

Purpose:
- Fair traffic distribution
- Best-effort traffic handling
- Queue fairness during congestion

---

# 10. Hierarchical QoS Parent Policy

```cisco
policy-map WAN-SHAPE
 class class-default
  shape average 1000000
  service-policy CAMPUS-QOS
```

Purpose:
- WAN shaping
- HQoS implementation
- Parent-child QoS hierarchy

Behavior:
- Parent shaping policy
- Child queueing policy
- Bandwidth inheritance

---

# 11. HQoS Interface Attachment

```cisco
interface GigabitEthernet0/0
 service-policy output WAN-SHAPE
```

Purpose:
- Apply hierarchical QoS
- Enforce WAN shaping
- Apply WAN edge traffic engineering

Direction:
- Output only

Reason:
- Queueing and shaping operate on egress interfaces

---

# 12. WAN Bandwidth Simulation

```cisco
interface GigabitEthernet0/0
 bandwidth 1000
```

Purpose:
- Simulate constrained WAN bandwidth
- Generate congestion conditions
- Support queue behavior analysis

Behavior:
- 1 Mbps reference bandwidth

---

# 13. Policing Demonstration Configuration

```cisco
policy-map CAMPUS-QOS

 class class-default
  police 50000
```

Purpose:
- Demonstrate traffic policing
- Simulate aggressive WAN enforcement
- Observe exceeded drops

Behavior:
- Conformed traffic transmitted
- Exceeded traffic dropped

---

# 14. LLQ Recovery Configuration

```cisco
policy-map CAMPUS-QOS

 class VOICE
  priority percent 10
```

Purpose:
- Restore strict priority queue
- Recover voice protection
- Re-enable LLQ operation

---

# 15. DSCP Re-marking Recovery

```cisco
class-map match-any VOICE
 match dscp ef
```

Purpose:
- Restore EF classification
- Recover voice queue matching
- Restore LLQ traffic handling

---

# 16. Traffic Generation Examples

## Voice Traffic Simulation

```bash
ping -Q 184 -i 0.002 100.64.1.1
```

Purpose:
- Simulate EF voice traffic
- Trigger LLQ activity

---

## Video Traffic Simulation

```bash
ping -Q 136 -i 0.002 100.64.1.1
```

Purpose:
- Simulate AF41 video traffic
- Trigger VIDEO queue activity

---

## Best Effort Traffic Simulation

```bash
ping -i 0.002 100.64.1.1
```

Purpose:
- Generate class-default traffic
- Simulate congestion

---

# 17. Primary Verification Commands

## QoS Policy Verification

```cisco
show policy-map interface g0/0
```

Purpose:
- View queue statistics
- Verify classification
- Observe drops and congestion
- Analyze shaping and LLQ behavior

---

## Policy Verification

```cisco
show policy-map
```

Purpose:
- Verify QoS policy structure
- Confirm LLQ and CBWFQ settings

---

## Class Map Verification

```cisco
show class-map
```

Purpose:
- Verify DSCP classification
- Confirm ACL matching

---

## Interface Verification

```cisco
show run interface g0/0
```

Purpose:
- Verify QoS attachment direction
- Confirm HQoS application

---

# 18. Final Operational Baseline

Final operational state:

```text
WAN-SHAPE
 └─ CAMPUS-QOS
      ├─ VOICE
      │    priority percent 10
      │
      ├─ VIDEO
      │    bandwidth percent 20
      │
      ├─ SCAVENGER
      │    set dscp cs1
      │
      └─ class-default
           fair-queue
```

Purpose:
- Stable Enterprise WAN QoS baseline
- SD-WAN WAN edge preparation
- Realistic WAN traffic engineering model
