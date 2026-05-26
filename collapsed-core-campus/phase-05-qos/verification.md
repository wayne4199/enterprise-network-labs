# Phase 05 — Enterprise QoS
# verification.md

# 1. Verify QoS Policy Attachment

## Command

```cisco
show run interface g0/0
```

---

## Expected Output

```text
service-policy output WAN-SHAPE
```

---

## Purpose

Verify:
- QoS attachment
- correct interface
- correct policy direction
- HQoS parent policy attachment

---

# 2. Verify HQoS Structure

## Command

```cisco
show policy-map interface g0/0
```

---

## Expected Output

```text
Service-policy output: WAN-SHAPE
```

And:

```text
Service-policy : CAMPUS-QOS
```

---

## Purpose

Verify:
- hierarchical QoS
- parent-child policy relationship
- shaping inheritance

---

# 3. Verify Voice Classification

## Traffic Generation

MGMT-SRV1:

```bash
ping -Q 184 -i 0.002 100.64.1.1
```

---

## Verification Command

```cisco
show policy-map interface g0/0
```

---

## Expected Output

```text
Class-map: VOICE
 packets increasing
```

And:

```text
Match: dscp ef (46)
```

---

## Purpose

Verify:
- EF classification
- LLQ operation
- voice queue activity

---

# 4. Verify Video Classification

## Traffic Generation

MGMT-SRV1:

```bash
ping -Q 136 -i 0.002 100.64.1.1
```

---

## Verification Command

```cisco
show policy-map interface g0/0
```

---

## Expected Output

```text
Class-map: VIDEO
 packets increasing
```

And:

```text
Match: dscp af41 (34)
```

---

## Purpose

Verify:
- AF41 classification
- CBWFQ queue activity
- video bandwidth reservation

---

# 5. Verify Scavenger Classification

## Traffic Generation

MGMT-SRV1:

```bash
ping -i 0.002 100.64.1.1
```

---

## Verification Command

```cisco
show policy-map interface g0/0
```

---

## Expected Output

```text
Class-map: SCAVENGER
 packets increasing
```

And:

```text
QoS Set
 dscp cs1
```

---

## Purpose

Verify:
- ACL classification
- DSCP re-marking
- scavenger queue handling
- trust boundary enforcement

---

# 6. Verify LLQ Operation

## Command

```cisco
show policy-map interface g0/0
```

---

## Expected Output

```text
Priority: 10%
```

And:

```text
Priority: 10% (100 kbps)
```

---

## Purpose

Verify:
- LLQ configuration
- strict priority queue
- reserved voice bandwidth

---

# 7. Verify LLQ Exceed Drops

## Traffic Generation

Generate aggressive EF traffic:

```bash
ping -Q 184 -i 0.002 100.64.1.1
```

---

## Verification Command

```cisco
show policy-map interface g0/0
```

---

## Expected Output

```text
b/w exceed drops increasing
```

---

## Purpose

Verify:
- LLQ policing
- priority queue enforcement
- congestion behavior

---

# 8. Verify CBWFQ Operation

## Command

```cisco
show policy-map interface g0/0
```

---

## Expected Output

```text
Class-map: VIDEO
 bandwidth 20%
```

---

## Purpose

Verify:
- CBWFQ operation
- bandwidth reservation
- multimedia queue behavior

---

# 9. Verify Traffic Shaping

## Command

```cisco
show policy-map interface g0/0
```

---

## Expected Output

```text
shape average 1000000
```

And:

```text
target shape rate 1000000
```

---

## Purpose

Verify:
- WAN shaping
- parent shaping policy
- traffic smoothing

---

# 10. Verify Policing

## Temporary Configuration

```cisco
policy-map CAMPUS-QOS

 class class-default
  police 50000
```

---

## Verification Command

```cisco
show policy-map interface g0/0
```

---

## Expected Output

```text
conformed packets increasing
```

And:

```text
exceeded packets increasing
```

---

## Purpose

Verify:
- traffic policing
- policing enforcement
- exceeded traffic handling

---

# 11. Verify Queue Drops

## Traffic Generation

Generate aggressive traffic:

```bash
ping -i 0.002 100.64.1.1
```

---

## Verification Command

```cisco
show policy-map interface g0/0
```

---

## Expected Output

```text
queue drops increasing
```

Or:

```text
flowdrops increasing
```

---

## Purpose

Verify:
- WAN congestion
- queue contention
- fair-queue behavior

---

# 12. Verify HQoS Bandwidth Inheritance

## Command

```cisco
show policy-map interface g0/0
```

---

## Expected Output

```text
Priority: 10% (100 kbps)
```

And:

```text
bandwidth 20% (200 kbps)
```

---

## Purpose

Verify:
- parent shaping inheritance
- child queue bandwidth calculation
- HQoS operation

---

# 13. Verify Re-marking Counters

## Command

```cisco
show policy-map interface g0/0
```

---

## Expected Output

```text
Packets marked increasing
```

---

## Purpose

Verify:
- DSCP rewriting
- trust boundary enforcement
- policy action execution

---

# 14. Verify Wrong Classification Failure

## Temporary Misconfiguration

```cisco
class-map match-any VOICE
 no match dscp ef
 match dscp af31
```

---

## Verification Command

```cisco
show policy-map interface g0/0
```

---

## Expected Output

```text
Class-map: VOICE
 0 packets
```

And:

```text
class-default counters increasing
```

---

## Purpose

Verify:
- classification failure
- DSCP mismatch behavior
- fallback into best-effort queue

---

# 15. Verify Service-Policy Direction

## Command

```cisco
show run interface g0/0
```

---

## Expected Output

```text
service-policy output WAN-SHAPE
```

---

## Purpose

Verify:
- output QoS attachment
- correct WAN queueing direction
- shaping placement

---

# 16. Verify Final Stable WAN Baseline

## Command

```cisco
show policy-map interface g0/0
```

---

## Expected Output

```text
drop rate 0000 bps
```

And:

```text
target shape rate 1000000
```

And:

```text
Priority: 10% (100 kbps)
```

---

## Purpose

Verify:
- operational recovery
- stable WAN QoS baseline
- reduced congestion
- realistic WAN shaping

---

# 17. Primary Operational Verification Command

## Command

```cisco
show policy-map interface g0/0
```

---

## Operational Analysis Areas

This command was used to analyze:

| Area | Verification |
|---|---|
| Classification | DSCP matching |
| LLQ | Priority queue operation |
| CBWFQ | Bandwidth reservation |
| Shaping | Parent shaping behavior |
| Policing | Exceeded traffic |
| Re-marking | Packet marking |
| Queue Drops | Congestion |
| HQoS | Parent-child hierarchy |
| WAN Stability | Drop rate analysis |

---

# 18. Final Operational Verification Summary

Final verified operational state:

| Feature | Status |
|---|---|
| HQoS | Operational |
| LLQ | Operational |
| CBWFQ | Operational |
| DSCP Classification | Operational |
| Re-marking | Operational |
| Traffic Shaping | Operational |
| WAN Congestion Recovery | Operational |
| Trust Boundary | Operational |
| Scavenger Handling | Operational |
| WAN QoS Baseline | Stable |

This final state represents:
- Enterprise WAN QoS foundation
- SD-WAN WAN edge preparation
- operationally stable QoS baseline
