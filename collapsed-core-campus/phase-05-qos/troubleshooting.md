# Phase 05 — Enterprise QoS
# troubleshooting.md

# 1. Voice Traffic Not Entering VOICE Class

## Symptoms

```cisco
show policy-map interface g0/0
```

Output:

```text
Class-map: VOICE
  0 packets
```

Voice traffic was not entering the LLQ.

---

## Cause

Wrong DSCP classification was configured:

```cisco
class-map match-any VOICE
 match dscp af31
```

However, actual traffic used:
- DSCP EF (46)

This created a classification mismatch.

---

## Impact

Voice traffic fell into:
- `class-default`
- best-effort queue

Effects:
- no LLQ protection
- increased drops
- increased latency and jitter

---

## Resolution

Restore correct DSCP classification:

```cisco
conf t

class-map match-any VOICE
 no match dscp af31
 match dscp ef
```

---

## Verification

```cisco
show policy-map interface g0/0
```

Expected result:

```text
Class-map: VOICE
 packets increasing
```

---

# 2. VIDEO Class Missing from Policy

## Symptoms

```cisco
show policy-map
```

Output showed:
- VOICE class
- SCAVENGER class
- class-default

But VIDEO class was missing.

---

## Cause

The policy was unintentionally overwritten during policy modification.

This simulated a real operational QoS regression issue.

---

## Impact

AF41 video traffic lost:
- bandwidth reservation
- CBWFQ protection

Video traffic reverted to:
- best-effort handling

---

## Resolution

Recreate VIDEO class:

```cisco
conf t

class-map match-any VIDEO
 match dscp af41
```

Restore VIDEO policy section:

```cisco
policy-map CAMPUS-QOS

 class VIDEO
  bandwidth percent 20
```

---

## Verification

```cisco
show policy-map
```

Expected result:

```text
Class VIDEO
 bandwidth percent 20
```

---

# 3. HQoS Parent Policy Removed

## Symptoms

```cisco
show run interface g0/0
```

Output:

```text
service-policy output CAMPUS-QOS
```

Instead of:

```text
service-policy output WAN-SHAPE
```

---

## Cause

The child policy was attached directly to the interface, removing:
- shaping
- HQoS hierarchy

---

## Impact

The following features were lost:
- parent shaping
- bandwidth inheritance
- HQoS hierarchy
- WAN congestion simulation

---

## Resolution

Restore HQoS attachment:

```cisco
conf t

interface g0/0
 no service-policy output CAMPUS-QOS
 service-policy output WAN-SHAPE
```

---

## Verification

```cisco
show policy-map interface g0/0
```

Expected output:

```text
Service-policy output: WAN-SHAPE
 Service-policy : CAMPUS-QOS
```

---

# 4. Traffic Shaping Applied in Wrong Direction

## Symptoms

Attempting:

```cisco
service-policy input WAN-SHAPE
```

Produced:

```text
Traffic Shaping feature not supported in input policy.
```

---

## Cause

Traffic shaping is an egress-only feature.

Shaping requires:
- queueing
- buffering
- controlled packet transmission

which are output operations.

---

## Impact

HQoS and shaping could not operate.

---

## Resolution

Apply shaping on output direction:

```cisco
interface g0/0
 service-policy output WAN-SHAPE
```

---

## Verification

```cisco
show run interface g0/0
```

Expected result:

```text
service-policy output WAN-SHAPE
```

---

# 5. SCAVENGER Class Not Matching Traffic

## Symptoms

```cisco
show policy-map interface g0/0
```

Output:

```text
Class-map: SCAVENGER
 0 packets
```

---

## Cause

ACL used for classification did not exist.

The class-map referenced:

```cisco
match access-group name ICMP-VOICE
```

But the ACL was missing.

---

## Impact

Traffic classification failed.
DSCP re-marking did not occur.

---

## Resolution

Create ACL:

```cisco
conf t

ip access-list extended ICMP-VOICE
 permit icmp any any
```

---

## Verification

```cisco
show policy-map interface g0/0
```

Expected result:

```text
Class-map: SCAVENGER
 packets increasing
```

And:

```text
Packets marked increasing
```

---

# 6. Excessive LLQ Exceed Drops

## Symptoms

```cisco
show policy-map interface g0/0
```

Output:

```text
b/w exceed drops increasing
```

---

## Cause

LLQ bandwidth reservation was too small.

The parent shaper was configured at:
- 50 Kbps

Resulting LLQ bandwidth:
- 5 Kbps

Voice traffic exceeded reserved bandwidth.

---

## Impact

Voice traffic experienced:
- LLQ policing
- packet drops
- potential jitter and quality degradation

---

## Resolution

Increase parent shaping rate:

```cisco
policy-map WAN-SHAPE
 class class-default
  shape average 1000000
```

---

## Verification

```cisco
show policy-map interface g0/0
```

Expected output:

```text
Priority: 10% (100 kbps)
```

And reduced exceed drops.

---

# 7. Excessive WAN Congestion

## Symptoms

```cisco
show policy-map interface g0/0
```

Output:

```text
drop rate increasing
```

and:

```text
queue drops increasing
```

---

## Cause

Artificially aggressive WAN shaping:
- 50 Kbps shaping rate
- high-frequency traffic generation

created severe congestion.

---

## Impact

Observed:
- shaping drops
- CBWFQ queue drops
- LLQ congestion
- fair-queue drops

---

## Resolution

Restore realistic WAN shaping rate:

```cisco
policy-map WAN-SHAPE
 class class-default
  shape average 1000000
```

---

## Verification

Expected:
- reduced drop rate
- stable queues
- improved WAN stability

---

# 8. QoS Configuration Lost After Reload

## Symptoms

QoS class-maps and policy-maps were missing after restart.

Examples:
- VIDEO class missing
- HQoS missing
- policy attachment removed

---

## Cause

Configuration changes were not saved before system interruption.

---

## Impact

QoS baseline partially disappeared.

---

## Resolution

Restore:
- class-maps
- policy-maps
- interface service-policy attachments

Recommended operational procedure:

```cisco
copy running-config startup-config
```

---

# 9. Traffic Falling into class-default

## Symptoms

```cisco
show policy-map interface g0/0
```

Output:

```text
class-default
 offered rate increasing
```

while expected traffic classes showed:
- low counters
- no matches

---

## Cause

Traffic classification failure caused fallback into:
- best-effort queue

Possible reasons:
- wrong DSCP
- ACL mismatch
- class-map error

---

## Impact

Traffic lost:
- priority handling
- bandwidth reservation
- QoS protection

---

## Resolution

Verify:
- DSCP marking
- ACLs
- class-map matching
- trust boundary behavior

---

## Verification Commands

```cisco
show class-map
show policy-map
show policy-map interface g0/0
```

---

# 10. Operational Troubleshooting Methodology

The following troubleshooting sequence was used throughout the phase:

| Step | Verification |
|---|---|
| 1 | Verify service-policy attachment |
| 2 | Verify policy direction |
| 3 | Verify classification |
| 4 | Verify queue counters |
| 5 | Verify LLQ behavior |
| 6 | Verify shaping drops |
| 7 | Verify DSCP marking |
| 8 | Verify HQoS hierarchy |

Primary operational command:

```cisco
show policy-map interface g0/0
```

This command was used to analyze:
- queue behavior
- congestion
- shaping
- LLQ
- packet drops
- DSCP re-marking
- classification
- HQoS hierarchy
