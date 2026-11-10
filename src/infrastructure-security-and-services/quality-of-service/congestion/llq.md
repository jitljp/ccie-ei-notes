# Low Latency Queuing (LLQ)

**Low Latency Queuing (LLQ)** is a congestion management mechanism that combines **Class-Based Weighted Fair Queuing (CBWFQ)** with **strict priority queuing (PQ)**.

CBWFQ allocates bandwidth among administrator-defined traffic classes using weighted scheduling.

LLQ extends CBWFQ by allowing selected traffic classes to receive strict priority treatment.

Strict priority traffic is scheduled before ordinary bandwidth queues, reducing queuing delay and jitter.

LLQ is particularly useful for delay-sensitive applications such as:
- Voice over IP (VoIP).
- Interactive video.
- Other real-time traffic requiring low latency and jitter.

Cisco IOS XE configures LLQ using the `priority` command within an MQC policy map.

---

## LLQ Operation

With ordinary CBWFQ, the scheduler distributes transmission opportunities among bandwidth queues according to their configured minimum rates and excess weights.

With LLQ, one or more classes are assigned to strict priority queues.

```text
                Incoming Traffic
                       |
                       v
                Classification
                       |
           +-----------+-----------+
           |           |           |
           v           v           v
      +----------+ +--------+ +---------+
      |  Voice   | |Business| | Default |
      | Priority | | Queue  | |  Queue  |
      +----------+ +--------+ +---------+
           |           |           |
           +-----------+-----------+
                       |
                       v
                   Scheduler
                       |
                       v
                Egress Interface
```

The scheduler services the priority queue before ordinary bandwidth queues whenever eligible priority traffic is waiting.

For example, suppose:
- The voice queue contains 5 packets.
- The business queue contains 20 packets.
- The default queue contains 100 packets.

The scheduler selects packets from the voice queue before the ordinary queues.

Once there are no eligible priority packets waiting, it services the bandwidth queues according to their scheduling weights.

If another voice packet arrives while the scheduler is servicing a bandwidth queue, the priority queue receives preferential treatment at the next appropriate scheduling opportunity.

**Strict priority scheduling is not packet preemption.** A voice packet cannot normally interrupt a data packet that has already begun transmission.

### LLQ vs. CBWFQ

| Feature | CBWFQ | LLQ |
|---|---|---|
| Scheduling | Weighted | Strict priority plus weighted |
| Minimum bandwidth guarantees | Yes | Yes, for ordinary bandwidth classes |
| Strict priority treatment | No | Yes |
| Suitable for voice | Less ideal | Yes |
| Risk of starvation | Controlled by weighted scheduling | Possible without priority admission control |
| Primary MQC command | `bandwidth` | `priority` |

LLQ does not replace CBWFQ.

It adds strict priority scheduling to a class-based queuing policy.

See [FIFO, WFQ, and CBWFQ](fifo-wfq-cbwfq.md).

---

## Strict Priority and Starvation

An important limitation of strict priority scheduling is that it can cause **starvation**.

Starvation occurs when lower-priority queues do not receive sufficient transmission opportunities because higher-priority traffic consumes the available capacity.

Consider a 100-Mbps interface with:
- Voice traffic offered at 100 Mbps.
- Business traffic offered at 50 Mbps.

If voice uses an unrestricted strict priority queue, it can consume the entire 100-Mbps interface capacity.

```text
           Voice: 100 Mbps
                  |
                  v
          +----------------+
          | Priority Queue |
          +----------------+
                  |
                  v
             100 Mbps
                Link
                  ^
                  |
          +----------------+
          | Business Queue |
          +----------------+
                  ^
                  |
           Business: 50 Mbps
```

The business queue may receive no bandwidth.

This is why priority queues should normally have some form of **queue admission control**.

Queue admission control limits the amount of traffic that can enter the priority queue, protecting the bandwidth available to other traffic classes.

Cisco IOS XE supports several approaches:
- Unconstrained priority queuing.
- Conditional priority policing.
- Always-on priority policing.

---

## The `priority` Command

Cisco IOS XE supports several forms of the `priority` command.

| Command | Function |
|---|---|
| `priority` | Configure an unconstrained strict priority queue |
| `priority 10000` | Strict priority with a conditional policer at 10,000 Kbps |
| `priority percent 10` | Strict priority with a percentage-based conditional policer |
| `priority level 1` | Unconstrained priority queue at level 1 |
| `priority level 2` | Unconstrained priority queue at level 2 |
| `priority level 1 percent 10` | Level 1 priority with a conditional percentage-based policer |
| `priority level 2 20000` | Level 2 priority with a conditional policer at 20,000 Kbps |

The supported syntax and policing behavior vary by platform.

The following sections primarily describe the QFP-based Cisco IOS XE scheduling model used on platforms such as the ASR 1000.

---

## Unconstrained Priority Queuing

An **unconstrained priority queue** has strict priority but no built-in rate limit.

It is configured using:

```text
R1(config)# policy-map WAN-QOS
R1(config-pmap)# class VOICE
R1(config-pmap-c)# priority
```

The voice class receives strict priority scheduling.

However, nothing in this configuration restricts the amount of bandwidth voice traffic can consume.

If voice traffic continuously offers enough packets to saturate the interface, other queues can be starved.

### Important Restriction

On the Cisco IOS XE QFP scheduler, unconstrained priority queues cannot be combined with minimum bandwidth guarantees configured using `bandwidth`.

For example, the following configuration is not supported:

```text
R1(config)# policy-map WAN-QOS
R1(config-pmap)# class VOICE
R1(config-pmap-c)# priority
R1(config-pmap)# class BUSINESS
R1(config-pmap-c)# bandwidth percent 30
```

The router cannot guarantee 30% of the interface bandwidth to BUSINESS while permitting VOICE to consume an unlimited amount of bandwidth.

However, unconstrained priority queues can be combined with excess-weight scheduling using `bandwidth remaining` on supported platforms.

In practice, unconstrained priority queues should generally be avoided unless another mechanism reliably limits the offered priority traffic.

---

## Conditional Priority Policing

A **conditional priority policer** limits priority traffic only when congestion exists.

This is the usual behavior of rate-based `priority` commands on QFP-based IOS XE routers.

For example:

```text
R1(config-pmap-c)# priority percent 10
```

This command:
1. Configures a strict priority queue.
2. Creates an implicit conditional policer with a rate equal to 10% of the applicable scheduling bandwidth.

On a 100-Mbps interface, the configured rate is:

```text
100 Mbps × 10% = 10 Mbps
```

However, **10 Mbps is not an unconditional maximum rate**.

The priority queue can exceed 10 Mbps when sufficient bandwidth is available and the scheduler is not congested.

When congestion occurs, the policer enforces the configured rate.

### Example: No Congestion

Consider a 100-Mbps interface.

The offered traffic is:

```text
Voice:    30 Mbps
Business: 20 Mbps

Total:    50 Mbps
```

The policy contains:

```text
R1(config-pmap-c)# priority percent 10
```

Although the configured priority rate is 10 Mbps, the interface has sufficient capacity to transmit all 50 Mbps.

Therefore:
- Voice can transmit at 30 Mbps.
- Business can transmit at 20 Mbps.
- No packets are dropped by the conditional priority policer solely because voice exceeds 10 Mbps.

The conditional policer monitors the priority traffic but suppresses its drop action when the scheduler is not congested.

### Example: With Congestion

Now suppose the offered traffic increases:

```text
Voice:    30 Mbps
Business: 90 Mbps

Total:   120 Mbps
```

The interface can transmit only 100 Mbps.

Congestion occurs.

The conditional priority policer now enforces the configured 10-Mbps rate.

Ignoring permitted bursts:

```text
Voice transmitted:  10 Mbps
Voice dropped:      20 Mbps

Business capacity:  90 Mbps
```

The priority policer prevents voice traffic from consuming bandwidth needed by the business queue.

The important distinction is:

| Condition | `priority percent 10` behavior |
|---|---|
| No congestion | Priority traffic may exceed 10% |
| Congestion | Priority traffic is policed to 10% |
| Priority traffic below 10% | Transmitted with strict priority |

The 10% value is therefore a **conditional policing rate**, not a permanent maximum transmission rate.

### How the Conditional Policer Works

On the QFP scheduler, the conditional policer operates before packets are admitted into the priority queue.

The scheduler provides congestion feedback to the policer.

If congestion is absent, packets that would otherwise exceed the configured rate can still be admitted.

If congestion exists, packets exceeding the allowed rate are dropped.

This protects other queues from starvation while allowing the priority class to use otherwise idle bandwidth.

---

## Always-On Priority Policing

An **always-on priority policer** enforces its configured rate regardless of whether congestion exists.

On supported QFP-based IOS XE routers, it can be configured using an unconstrained priority queue combined with an explicit policer.

For example:

```text
R1(config)# policy-map WAN-QOS
R1(config-pmap)# class VOICE
R1(config-pmap-c)# priority
R1(config-pmap-c)# police cir 10000000
```

This configuration:
- Assigns strict priority scheduling to VOICE.
- Applies an explicit policer with a CIR of 10 Mbps.
- Enforces the 10-Mbps rate regardless of interface congestion.

### Example: No Congestion

Suppose:

```text
Interface capacity: 100 Mbps
Voice offered:       30 Mbps
Business offered:    20 Mbps
```

With:

```text
R1(config-pmap-c)# priority
R1(config-pmap-c)# police cir 10000000
```

The policer enforces the 10-Mbps limit even though the interface has spare capacity.

Ignoring permitted bursts:

```text
Voice transmitted: 10 Mbps
Voice dropped:     20 Mbps
Business:          20 Mbps
```

The interface transmits approximately 30 Mbps and has unused capacity.

The priority traffic cannot consume the unused bandwidth because the explicit policer is always active.

### Conditional vs. Always-On Policing

| Characteristic | Conditional policer | Always-on policer |
|---|---|---|
| Configuration | `priority percent 10` | `priority` + `police cir` |
| Strict priority scheduling | Yes | Yes |
| Policer active during congestion | Yes | Yes |
| Policer drops excess traffic without congestion | No | Yes |
| Can exceed configured rate without congestion | Yes | No, except permitted bursts |
| Protects other queues | Yes | Yes |

Always-on policing provides more predictable admission control because its behavior does not depend on congestion state.

Conditional policing offers greater flexibility by allowing the priority class to use additional available bandwidth.

**Neither mechanism guarantees lossless voice delivery.** If the priority policer is configured below the application's actual requirements, packets may be dropped.

---

## Priority Queue Burst Size

Priority policers use token bucket mechanisms that permit a configurable amount of burst traffic.

Allowing some burst traffic is necessary because real applications do not transmit packets at perfectly constant rates.

However, excessive bursts can cause queue buildup.

Even though a priority queue is serviced before ordinary bandwidth queues, packets within the priority queue must still wait behind earlier priority packets.

This increases delay and jitter.

### Calculating a Burst Size

A useful starting point is to calculate the amount of data transmitted at the policing rate during a desired burst interval.

The formula is:

```text
Burst (bytes) = Rate (bps) × Time (seconds) / 8
```

For example, suppose:
- Priority rate = 10 Mbps.
- Desired burst tolerance = 5 ms.

Calculate:

```text
Burst = 10,000,000 × 0.005 / 8
      = 6,250 bytes
```

On supported QFP-based platforms, the burst can be included in the `priority` command.

```text
R1(config-pmap-c)# priority 10000 6250
```

This configures:
- Priority policing rate: 10,000 Kbps.
- Burst allowance: 6,250 bytes.

The second numeric argument is expressed in bytes.

### Burst Size and Jitter

A larger burst allowance permits more traffic to enter the priority queue in a short period.

This can increase the amount of data waiting for transmission and therefore increase jitter.

A smaller burst allowance can reduce queue buildup, but may cause legitimate traffic bursts to be dropped.

Cisco's QFP scheduling documentation suggests that a burst allowance corresponding to approximately 5 to 10 ms of traffic can be a useful design guideline for low-jitter priority applications.

The appropriate value depends on the application, packet sizes, codec behavior, and platform.

### Default Burst Allowances

Cisco documents the following defaults for the IOS XE scheduling implementation discussed here:

| Policer type | Documented default burst time |
|---|---|
| Conditional priority policer | 200 ms |
| Explicit always-on policer | 250 ms |

These are platform-specific defaults, not universal values for all Cisco IOS XE devices.

Because the default burst allowances may be relatively large for delay-sensitive traffic, explicitly configuring an appropriate burst size can be useful.

---

## Multiple Priority Queues (MPQ)

**Multi-Level Priority Queuing (MPQ)** allows multiple strict priority levels within the same policy.

On supported QFP-based IOS XE routers, two priority levels are available:

- **Priority Level 1 (P1):** Highest priority.
- **Priority Level 2 (P2):** Lower priority than P1, but higher than ordinary bandwidth queues.

For example:

```text
                  Scheduler
                      |
             +--------+--------+
             |        |        |
             v        v        v
          +------+ +------+ +------+
          |  P1  | |  P2  | | CBWFQ|
          |Voice | |Video | | Data |
          +------+ +------+ +------+
             |        |        |
             +--------+--------+
                      |
                      v
                   Output
```

The scheduler services queues in the following order:

1. Priority Level 1.
2. Priority Level 2.
3. Ordinary CBWFQ bandwidth queues.

If P1 has eligible packets waiting, they are serviced before P2.

If P1 has no eligible packets but P2 does, P2 is serviced before the ordinary bandwidth queues.

### MPQ Configuration

For example:

```text
R1(config)# policy-map WAN-QOS
R1(config-pmap)# class VOICE
R1(config-pmap-c)# priority level 1 percent 10
R1(config-pmap)# class VIDEO
R1(config-pmap-c)# priority level 2 percent 20
R1(config-pmap)# class BUSINESS
R1(config-pmap-c)# bandwidth percent 30
```

This configures:

| Class | Scheduling | Conditional rate |
|---|---|---|
| VOICE | Priority Level 1 | 10% |
| VIDEO | Priority Level 2 | 20% |
| BUSINESS | CBWFQ | Minimum 30% |

VOICE receives strict priority over VIDEO.

VIDEO receives strict priority over BUSINESS.

The configured conditional rates help prevent excessive priority traffic from starving lower-priority queues.

### Explicit Priority Levels Are Required

When configuring multiple priority levels on the QFP-based IOS XE scheduler, specify the levels explicitly.

For example:

```text
R1(config-pmap-c)# priority level 1 percent 10
```

and:

```text
R1(config-pmap-c)# priority level 2 percent 20
```

Although a standalone `priority` command uses P1 by default, it should not be mixed with an explicit P2 configuration.

On the documented QFP implementation, the following combination is rejected:

```text
class VOICE
 priority
 police cir 10000000
class VIDEO
 priority level 2
 police cir 20000000
```

Instead, explicitly configure `priority level 1` for VOICE.

### Multiple Classes at the Same Priority Level

On supported IOS XE platforms, multiple traffic classes can be associated with the same priority level.

For example, separate classes may identify different categories of real-time traffic but ultimately share the same strict priority queue.

Their traffic does not receive separate scheduling priority within that level merely because they belong to different class maps.

Specific restrictions on assigning multiple classes to the same priority level vary by platform.

---

## LLQ and CBWFQ Bandwidth Allocation

LLQ combines priority queues with ordinary bandwidth queues.

Consider a 100-Mbps interface with:

```text
class VOICE
 priority percent 10
class BUSINESS
 bandwidth percent 40
class BULK
 bandwidth percent 20
class class-default
```

Assume all classes have sufficient traffic to remain backlogged.

During congestion:
- VOICE receives strict priority, with a conditional policing rate of 10 Mbps.
- BUSINESS has a minimum bandwidth guarantee of 40 Mbps.
- BULK has a minimum bandwidth guarantee of 20 Mbps.
- Remaining bandwidth is distributed among eligible bandwidth queues according to their excess weights.

If VOICE continuously offers at least 10 Mbps, the conceptual allocation is:

```text
Interface capacity = 100 Mbps

VOICE = 10 Mbps

Remaining = 90 Mbps

BUSINESS minimum = 40 Mbps
BULK minimum     = 20 Mbps

Excess = 30 Mbps
```

On the QFP three-parameter scheduler, assuming BUSINESS, BULK, and `class-default` each have the default excess weight of 1:

```text
30 Mbps / 3 = 10 Mbps per queue
```

The resulting rates are:

| Class | Base allocation | Excess | Total |
|---|---:|---:|---:|
| VOICE | 10 Mbps | 0 Mbps | 10 Mbps |
| BUSINESS | 40 Mbps | 10 Mbps | 50 Mbps |
| BULK | 20 Mbps | 10 Mbps | 30 Mbps |
| class-default | 0 Mbps | 10 Mbps | 10 Mbps |
| **Total** | **70 Mbps** | **30 Mbps** | **100 Mbps** |

These values illustrate sustained congestion with a fully utilized 10-Mbps priority allowance.

Actual behavior depends on offered traffic, bursts, configured scheduling weights, and platform implementation.

If VOICE offers only 5 Mbps, more bandwidth becomes available to the ordinary queues.

If the interface is not congested, a conditional priority queue may also exceed its configured rate.

---

## LLQ Configuration Restrictions

On QFP-based IOS XE routers, several restrictions are important.

### `priority` vs. `bandwidth`

A class cannot normally be configured with both strict priority and a minimum bandwidth guarantee.

For example:

```text
R1(config-pmap-c)# priority percent 10
R1(config-pmap-c)# bandwidth percent 20
```

These commands conflict because the class is being configured as both a priority queue and an ordinary bandwidth queue.

Use `priority` for strict priority scheduling.

Use `bandwidth` for weighted scheduling with a minimum bandwidth guarantee.

### `priority` vs. `shape`

Likewise, the QFP scheduling model does not support configuring a priority queue and a shaper in the same class.

For example:

```text
R1(config-pmap-c)# priority percent 10
R1(config-pmap-c)# shape average 10000000
```

These commands are not valid together in the same class on the documented QFP scheduler.

However, a parent shaping policy can control the aggregate rate while a child policy provides strict priority scheduling.

### `class-default`

The default class cannot be assigned strict priority treatment on the QFP scheduling implementation.

For example:

```text
R1(config-pmap)# class class-default
R1(config-pmap-c)# priority percent 10
```

This is not supported.

Priority treatment should be assigned to explicitly defined traffic classes.

### WRED and Priority Queues

WRED is generally used with ordinary bandwidth queues to provide congestion avoidance for responsive traffic.

It is not normally necessary for a well-designed voice priority queue.

Voice traffic is typically not responsive to packet drops in the same way as TCP traffic.

On some platforms, WRED or queue-limit configuration is supported for certain priority queue arrangements, but restrictions vary.

See [Weighted Random Early Detection (WRED)](wred.md).

---

## LLQ with Hierarchical QoS

LLQ is commonly combined with traffic shaping in a hierarchical QoS policy.

Consider:
- Physical interface: 1 Gbps.
- WAN service contract: 100 Mbps.
- Voice traffic requiring strict priority treatment.

Applying only an LLQ policy to the 1-Gbps physical interface may not be sufficient.

The router can transmit packets at up to 1 Gbps, but the service provider may police traffic above 100 Mbps.

Consequently, the actual congestion point may be at the provider's policer rather than within the router's QoS scheduler.

A hierarchical policy solves this problem.

### Example

Create the child policy:

```text
R1(config)# policy-map WAN-CHILD
R1(config-pmap)# class VOICE
R1(config-pmap-c)# priority percent 10
R1(config-pmap)# class BUSINESS
R1(config-pmap-c)# bandwidth percent 40
R1(config-pmap)# class class-default
```

Create the parent policy:

```text
R1(config)# policy-map WAN-PARENT
R1(config-pmap)# class class-default
R1(config-pmap-c)# shape average 100000000
R1(config-pmap-c)# service-policy WAN-CHILD
```

Apply the parent policy:

```text
R1(config)# interface GigabitEthernet1
R1(config-if)# service-policy output WAN-PARENT
```

The parent policy shapes aggregate traffic to 100 Mbps.

The child policy schedules traffic within the shaped capacity.

On supported hierarchical schedulers, priority traffic can receive strict priority throughout the scheduling hierarchy without requiring a separate priority command in the parent.

**The parent shaper creates a local congestion point where LLQ can control packet scheduling before traffic reaches the provider's policer.**

See [4.4.g Hierarchical QoS (HQoS)](../hqos/overview.md).

---

## Complete LLQ Configuration Example

Consider R1 with a 100-Mbps WAN interface.

The requirements are:

| Traffic | DSCP | Treatment |
|---|---|---|
| Voice | EF | Priority Level 1, 10% |
| Interactive video | AF41 | Priority Level 2, 20% |
| Business applications | AF31 | Minimum bandwidth 30% |
| Other traffic | Any unmatched DSCP | Default bandwidth scheduling |

### Step 1: Configure Class Maps

```text
R1(config)# class-map match-any VOICE
R1(config-cmap)# match dscp ef
R1(config)# class-map match-any VIDEO
R1(config-cmap)# match dscp af41
R1(config)# class-map match-any BUSINESS
R1(config-cmap)# match dscp af31
```

### Step 2: Configure the Policy Map

```text
R1(config)# policy-map WAN-QOS
R1(config-pmap)# class VOICE
R1(config-pmap-c)# priority level 1 percent 10
R1(config-pmap)# class VIDEO
R1(config-pmap-c)# priority level 2 percent 20
R1(config-pmap)# class BUSINESS
R1(config-pmap-c)# bandwidth percent 30
R1(config-pmap)# class class-default
```

On a supported QFP router with a 100-Mbps scheduling rate:

- VOICE receives strict P1 treatment and a 10-Mbps conditional policing rate.
- VIDEO receives strict P2 treatment and a 20-Mbps conditional policing rate.
- BUSINESS receives a 30-Mbps minimum bandwidth guarantee during congestion.
- Unmatched traffic uses the default bandwidth queue.

### Step 3: Apply the Policy Outbound

```text
R1(config)# interface GigabitEthernet1
R1(config-if)# service-policy output WAN-QOS
```

The policy now controls packets transmitted from GigabitEthernet1.

If the WAN bandwidth is lower than the physical interface's capacity, use hierarchical shaping as described previously.

---

## LLQ on Catalyst 9000 Switches

Catalyst 9000 switches use hardware switching ASICs rather than the QFP architecture.

For example, many Catalyst 9000 platforms use Cisco UADP ASICs.

These ASICs provide hardware egress queues and scheduling mechanisms.

On UADP-based Catalyst 9000 switches, the scheduler commonly supports:
- Strict priority queues.
- Weighted scheduling for ordinary queues.
- Platform-specific queue buffers and dropping thresholds.

Cisco documents support for two strict priority levels, P1 and P2, on several UADP-based Catalyst 9000 platforms.

However, **do not assume that the QFP conditional policing behavior applies to Catalyst switches**.

For example, Catalyst 9300 documentation describes priority rate limits that can be enforced independently of congestion.

The supported number of queues, priority levels, policing behavior, and command combinations depend on the switch model and software release.

For CCIE EI, the essential concepts remain the same:
- Priority queues are serviced before weighted queues.
- P1 is serviced before P2.
- Excessive priority traffic can starve other queues.
- Appropriate rate limiting protects lower-priority traffic.

---

## Verification and Troubleshooting

The primary IOS XE verification command is:

```text
R1# show policy-map interface GigabitEthernet1
```

For LLQ, examine:
- The class matching criteria.
- The configured priority level.
- The configured priority policing rate.
- Conformed and exceeded packet counters.
- Priority bandwidth-exceed drops.
- Queue depth and queue drops.
- Actual offered and transmitted traffic rates.

### Priority Queue Statistics

On supported IOS XE routers, output may include information similar to:

```text
Priority: Strict, b/w exceed drops: 0
Priority Level: 1
```

This indicates:
- The class receives strict priority scheduling.
- The class is associated with priority level 1.
- No priority bandwidth-exceed drops have been recorded.

When a conditional policer is configured, the output may also display its CIR and burst size.

For example, if the policy uses:

```text
R1(config-pmap-c)# priority level 1 percent 10
```

the policy statistics should identify the 10% policing rate and any available exceeded-packet counters.

### Useful Commands

| Command | Purpose |
|---|---|
| `show policy-map` | Display configured QoS policies |
| `show policy-map WAN-QOS` | Verify configured priority levels and rates |
| `show policy-map interface GigabitEthernet1` | Display LLQ statistics |
| `show policy-map interface GigabitEthernet1 output` | Verify outbound QoS behavior |
| `show interfaces GigabitEthernet1` | Check utilization and output drops |
| `show running-config interface GigabitEthernet1` | Verify the attached service policy |
| `show platform hardware qfp active feature qos interface GigabitEthernet1` | Inspect hardware QoS statistics on supported QFP platforms |

### Common Troubleshooting Scenarios

**Voice traffic experiences unexpected drops**
- Check the priority policer's exceed counters.
- Verify the configured rate and burst allowance.
- Confirm that only intended traffic matches the voice class.
- Check whether the actual voice traffic exceeds the configured rate.

**Voice works without congestion but fails under congestion**
- Check whether a conditional priority policer is dropping excess packets.
- Verify that the configured priority rate is sufficient.
- Determine whether packet bursts exceed the allowed burst size.

**Business traffic receives very little bandwidth**
- Check whether the priority queue is unconstrained.
- Verify the configured priority rates.
- Confirm that lower-priority classes have appropriate scheduling allocations.

**Voice experiences excessive jitter**
- Check for excessive bursts within the priority queue.
- Consider whether the priority policer's burst allowance is too large.
- Examine physical serialization delay and downstream buffering.
- Verify that the actual congestion point is controlled by the QoS policy.

**Priority policing does not appear to work**
- Determine whether the policer is conditional or always-on.
- Conditional policers do not necessarily drop excess traffic when congestion is absent.
- Verify that the policy is applied to the correct interface and direction.
- Confirm platform-specific priority scheduling and policing behavior.

---

## Key Points

- **LLQ** combines CBWFQ with strict priority queuing.
- Strict priority queues are serviced before ordinary bandwidth queues.
- LLQ is particularly useful for voice and other delay-sensitive traffic.
- Strict priority queuing does not preempt packets already being transmitted.
- Unrestricted priority traffic can starve other queues.
- The `priority` command without a rate configures an unconstrained priority queue on the documented QFP scheduler.
- `priority percent 10` configures strict priority with a 10% conditional policing rate.
- **Conditional priority policing enforces its configured rate only during congestion** on the QFP implementation.
- An explicit `police` command provides always-on rate enforcement when combined with unconstrained priority queuing.
- Priority policer burst allowances affect packet loss, delay, and jitter.
- **MPQ** supports multiple strict priority levels.
- On supported QFP-based IOS XE routers, P1 has higher priority than P2.
- P1 and P2 both have higher priority than ordinary CBWFQ queues.
- When using multiple priority levels, the levels must be explicitly configured on the QFP scheduler.
- On the QFP scheduler, `priority` and `bandwidth` cannot be configured within the same class.
- LLQ is commonly combined with parent shaping in hierarchical QoS policies.
- Catalyst 9000 priority queuing implementations may differ from QFP-based IOS XE routers.
- `show policy-map interface` is the primary command for verifying LLQ operation.

---

## References

- [Cisco IOS XE: QoS Scheduling](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/quality-of-service/quality-of-service/m_qos-scheduling.html)
- [Cisco IOS XE 17.x: QoS Hierarchical Scheduling](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/qos/b-quality-of-service/m_qos-hierarchical_scheduling.html)
- [Cisco IOS QoS Command Reference: `priority`](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/qos/command/qos-cr-book/qos-n1.html)
- [Cisco Catalyst 9000 Switching Platforms: QoS and Queuing White Paper](https://www.cisco.com/c/en/us/products/collateral/switches/catalyst-9000/white-paper-c11-742388.html)
- [Cisco Catalyst 9300 Series: Configuring QoS](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9300/software/release/26-x/configuration_guide/qos/b_26x_qos_9300_cg/configuring_qos.html)