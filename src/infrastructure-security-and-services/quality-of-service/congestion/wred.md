
# Weighted Random Early Detection (WRED)

**Weighted Random Early Detection (WRED)** is a congestion avoidance mechanism that selectively drops packets before an output queue becomes full.

WRED extends **Random Early Detection (RED)** by allowing different traffic markings to receive different dropping probabilities.

Cisco IOS XE supports WRED based on:
- IP Precedence.
- Differentiated Services Code Point (DSCP).
- Internal discard class, on supported platforms.

WRED is commonly combined with Class-Based Weighted Fair Queuing (CBWFQ) to provide differentiated congestion avoidance within individual traffic classes.

Unlike tail drop, which uses instantaneous queue depth, WRED uses an **average queue depth** to determine when packets should be dropped.

---

## WRED Operation

WRED attempts to prevent sustained queue buildup by providing congestion signals to traffic sources before a queue becomes full.

Consider several TCP flows sharing a congested WAN connection.

```text
        TCP Flow 1 ----+
                       |
        TCP Flow 2 ----+        R1
                       |   +-------------+
        TCP Flow 3 ----+-->| Output Queue|----> WAN
                       |   +-------------+
        TCP Flow 4 ----+          |
                                  |
                                WRED
```

When packets arrive faster than the outgoing interface can transmit them, the queue begins to fill.

Without WRED, packets continue accumulating until the queue reaches its limit.

With WRED, the router can begin randomly dropping packets before that limit is reached.

The general process is:

1. A packet arrives at an output queue.
2. WRED calculates the average queue depth.
3. WRED identifies the packet's drop profile using its IP Precedence, DSCP, or discard class.
4. The average queue depth is compared against the configured thresholds for that profile.
5. WRED determines whether to enqueue or drop the packet.
6. If a TCP sender detects packet loss, it can reduce its transmission rate.

WRED relies on traffic sources responding to congestion signals.

Consequently, it is particularly useful for **elastic traffic**, such as TCP flows.

It is less effective for traffic that does not reduce its transmission rate in response to packet loss.

### WRED vs. Tail Drop

| Characteristic | Tail drop | WRED |
|---|---|---|
| Primary purpose | Enforce queue limits | Avoid persistent congestion |
| Queue measurement | Instantaneous depth | Average depth |
| When dropping begins | Queue reaches its limit | Before queue becomes full |
| Packet selection | Arriving packets when queue is full | Probabilistic |
| Differentiated drop treatment | Not inherently | Yes |
| TCP global synchronization | Can contribute to synchronization | Attempts to reduce synchronization |

WRED does not eliminate tail drop.

If the instantaneous queue depth reaches the queue limit, packets can still be discarded regardless of their WRED drop profile.

See [Queue Limits and Tail Drop](queue-limits.md).

---

## WRED Thresholds

WRED uses three main parameters to control dropping behavior.

| Parameter | Description |
|---|---|
| **Minimum threshold** | Average queue depth at which probabilistic early dropping begins |
| **Maximum threshold** | Average queue depth beyond which arriving packets for that profile are dropped |
| **Mark probability denominator (MPD)** | Controls the maximum probability of a random early drop |

These parameters define a **WRED drop curve**.

### Minimum and Maximum Thresholds

Consider this configuration:

```text
R1(config-pmap-c)# random-detect dscp af31 50 100 10
```

The parameters are:

| Parameter | Value |
|---|---:|
| DSCP | AF31 |
| Minimum threshold | 50 packets |
| Maximum threshold | 100 packets |
| Mark probability denominator | 10 |

WRED evaluates the average queue depth against these thresholds.

| Average queue depth | WRED behavior |
|---|---|
| Below 50 packets | No WRED early drops |
| Between 50 and 100 packets | Randomly drop packets with increasing probability |
| At or above 100 packets | Drop all arriving AF31 packets according to conventional WRED behavior |

The configured maximum random-drop probability is 1/10, or 10%, immediately below the maximum threshold.

At the maximum threshold, conventional WRED transitions to dropping all arriving packets for that profile.

```text
Drop Probability
     ^
100% |                            +----------
     |                            |
     |                            |
     |                            |
 10% |                         ...|
     |                      ...
     |                   ...
     |                ...
  0% +---------------+------------------------>
                     50          100
                          Average Queue Depth
```

The probability increases as the average queue depth approaches the maximum threshold.

**The maximum WRED threshold is not the same as the queue limit.**

WRED can drop all packets belonging to a particular drop profile while the queue still has available buffer space.

### Mark Probability Denominator

The **mark probability denominator** controls the configured maximum random-drop probability.

The relationship is:

```text
Maximum random-drop probability = 1 / MPD
```

Examples:

| MPD | Maximum random-drop probability |
|---|---:|
| 1 | 100% |
| 2 | 50% |
| 5 | 20% |
| 10 | 10% |
| 20 | 5% |
| 100 | 1% |

The commonly documented default denominator is **10**.

For example:

```text
R1(config-pmap-c)# random-detect dscp af31 50 100 20
```

Configures a maximum random-drop probability of 5% for AF31.

This does not mean exactly one out of every 20 packets will be dropped.

The actual dropping behavior is probabilistic and depends on the average queue depth.

**A larger denominator means a lower random-drop probability.**

### WRED Drop Probability

In the basic RED model, the drop probability increases approximately linearly between the minimum and maximum thresholds.

The nominal probability is calculated as:

```text
P = (Average - Min) / (Max - Min) × (1 / MPD)
```

This is the basic probability curve. Actual implementations may adjust the probability according to additional packet-counting or hardware-specific behavior.

For example, assume:

```text
Minimum threshold = 50 packets
Maximum threshold = 100 packets
MPD               = 10
Average depth     = 75 packets
```

The nominal probability is:

```text
P = (75 - 50) / (100 - 50) × (1 / 10)
  = 25 / 50 × 0.1
  = 0.05
  = 5%
```

At an average depth of 75 packets, the nominal random-drop probability is 5%.

As the average approaches 100 packets, the probability approaches 10%.

At the maximum threshold, the conventional algorithm transitions to dropping all arriving packets for that profile.

---

## Average Queue Depth

WRED uses an **Exponentially Weighted Moving Average (EWMA)** to calculate average queue depth.

This prevents short traffic bursts from immediately triggering aggressive packet dropping.

The calculation is:

```text
NewAvg = OldAvg + (Instantaneous - OldAvg) / 2^n
```

Where:
- `NewAvg`: Updated average queue depth.
- `OldAvg`: Previous average queue depth.
- `Instantaneous`: Current queue depth.
- `n`: Exponential weighting constant.

### Example

Suppose:

```text
OldAvg        = 12 packets
Instantaneous = 44 packets
n             = 6
```

The calculation is:

```text
NewAvg = 12 + (44 - 12) / 2^6
       = 12 + 32 / 64
       = 12.5 packets
```

Although the instantaneous queue contains 44 packets, the updated average is only 12.5 packets.

This dampening prevents WRED from reacting too aggressively to short bursts.

### Exponential Weighting Constant

The exponential weighting constant determines how quickly the calculated average responds to queue-depth changes.

| Weighting constant | Effect |
|---|---|
| Smaller value | Average responds more quickly to queue changes |
| Larger value | Average responds more slowly to queue changes |

For example:

```text
R1(config-pmap-c)# random-detect exponential-weighting-constant 6
```

Configures an exponential weighting constant of 6.

This means that each update incorporates 1/64 of the difference between the instantaneous and previous average queue depths.

A smaller constant makes WRED more responsive to short-term congestion.

A larger constant makes WRED less sensitive to temporary queue fluctuations but slower to react to sustained congestion.

Cisco's ASR 1000 documentation describes defaults of 6 for packet-mode WRED and 9 for byte-mode WRED. Defaults and supported values can differ by platform and implementation.

**In most configurations, the default exponential weighting constant should be retained unless there is a specific reason to tune it.**

---

## WRED Classification Modes

Cisco IOS XE supports several methods of selecting a WRED drop profile.

| Mode | Selection criterion | Configuration |
|---|---|---|
| Precedence-based | 3-bit IP Precedence | `random-detect` |
| DSCP-based | 6-bit DSCP | `random-detect dscp-based` |
| Discard-class-based | Internal discard class | `random-detect discard-class-based` |

### IP Precedence-Based WRED

Precedence-based WRED uses the three IP Precedence bits to select the drop profile.

There are eight possible precedence values, 0 through 7.

Enable precedence-based WRED:

```text
R1(config-pmap-c)# random-detect
```

Or explicitly:

```text
R1(config-pmap-c)# random-detect precedence-based
```

Precedence-based WRED is the default mode on the documented IOS XE router implementation.

Individual thresholds can be configured:

```text
R1(config-pmap-c)# random-detect precedence 0 40 100 10
R1(config-pmap-c)# random-detect precedence 3 60 100 10
R1(config-pmap-c)# random-detect precedence 5 80 100 10
```

The configuration makes precedence 0 traffic eligible for early dropping first.

Precedence 5 receives the most protection of the three configured values because its minimum threshold is highest.

On the documented QFP implementation, default minimum thresholds generally increase with precedence, while default maximum thresholds and mark probabilities are shared.

### DSCP-Based WRED

**DSCP-based WRED** uses the six-bit DSCP field to select a drop profile.

There are 64 possible DSCP values, allowing more granular differentiation than IP Precedence.

Enable DSCP-based WRED:

```text
R1(config-pmap-c)# random-detect dscp-based
```

Specific DSCP thresholds can then be configured:

```text
R1(config-pmap-c)# random-detect dscp af31 100 160 20
R1(config-pmap-c)# random-detect dscp af32 80 160 10
R1(config-pmap-c)# random-detect dscp af33 60 160 5
```

The configuration uses different drop curves for AF31, AF32, and AF33.

DSCP-based WRED can also use numerical DSCP values:

```text
R1(config-pmap-c)# random-detect dscp 26 100 160 20
```

This is equivalent to specifying `af31`.

DSCP-based WRED does not create a separate output queue for every DSCP value.

**Packets with different DSCP values can share the same physical queue while receiving different WRED dropping treatment.**

### Discard-Class-Based WRED

On supported IOS XE platforms, WRED can use an internal **discard class** rather than the DSCP or IP Precedence field.

A discard class is an internal QoS attribute that can be assigned to traffic and used by subsequent QoS processing.

Enable discard-class-based WRED:

```text
R1(config-pmap-c)# random-detect discard-class-based
```

On the documented QFP implementation, eight discard-class values are available, numbered 0 through 7.

For example:

```text
R1(config-pmap-c)# random-detect discard-class 1 60 150 10
R1(config-pmap-c)# random-detect discard-class 2 100 150 10
```

Here, discard class 1 is more likely to be dropped during congestion than discard class 2.

This can be useful when packets have identical external markings but should receive different internal dropping treatment.

For example, MPLS traffic may have the same Traffic Class (TC) marking while retaining different internal discard-class values.

Discard class is not a field transmitted in the IPv4 or IPv6 header.

---

## DSCP-Based WRED and Assured Forwarding

WRED is particularly useful for implementing the drop precedence behavior associated with **Assured Forwarding (AF)**.

Within an AF class, the three drop precedences are:

| DSCP | Decimal value | Drop precedence |
|---|---:|---|
| AF31 | 26 | Low |
| AF32 | 28 | Medium |
| AF33 | 30 | High |

AF31 should receive the lowest dropping probability, while AF33 should receive the highest.

However, DSCP markings alone do not guarantee this behavior.

The QoS policy must provide the corresponding dropping treatment.

### Example: AF3 Drop Precedence

Suppose all AF3 packets share one output queue.

Configure the following WRED thresholds:

| DSCP | Minimum threshold | Maximum threshold | MPD |
|---|---:|---:|---:|
| AF31 | 100 | 160 | 20 |
| AF32 | 80 | 160 | 10 |
| AF33 | 60 | 160 | 5 |

The corresponding commands are:

```text
R1(config-pmap-c)# random-detect dscp af31 100 160 20
R1(config-pmap-c)# random-detect dscp af32 80 160 10
R1(config-pmap-c)# random-detect dscp af33 60 160 5
```

Consider an average queue depth of 90 packets.

| DSCP | WRED behavior |
|---|---|
| AF31 | No early drops because average depth is below 100 |
| AF32 | Random early dropping because average depth exceeds 80 |
| AF33 | More aggressive random dropping because average depth exceeds 60 and its denominator is smaller |

This implements the intended AF drop-precedence relationship.

**WRED does not reorder AF packets according to their drop precedence.**

Packets that are accepted into the same FIFO-style class queue retain their normal queuing order. The different DSCP values affect their probability of being admitted or dropped, not their scheduling priority.

See [Per-Hop Behaviors (BE, EF, AF, CS)](../diffserv/phb.md).

---

## WRED Configuration with MQC

On supported IOS XE routers, WRED is configured within a policy-map class.

WRED is normally used with a queueing action such as `bandwidth`, `bandwidth remaining`, or `shape`.

The `class-default` class can also support WRED through its implicit bandwidth scheduling behavior.

### Example: Configure WRED for AF3 Traffic

Configure R1 to:
- Match AF31, AF32, and AF33 traffic.
- Allocate a minimum of 30% bandwidth to this class.
- Limit the class queue to 200 packets.
- Enable DSCP-based WRED.
- Give AF31 the greatest protection from early dropping.
- Give AF33 the least protection.

First, create the class map:

```text
R1(config)# class-map match-any AF3-DATA
R1(config-cmap)# match dscp af31 af32 af33
```

Next, configure the policy:

```text
R1(config)# policy-map WAN-QOS
R1(config-pmap)# class AF3-DATA
R1(config-pmap-c)# bandwidth percent 30
R1(config-pmap-c)# queue-limit 200 packets
R1(config-pmap-c)# random-detect dscp-based
R1(config-pmap-c)# random-detect dscp af31 100 160 20
R1(config-pmap-c)# random-detect dscp af32 80 160 10
R1(config-pmap-c)# random-detect dscp af33 60 160 5
```

Finally, apply the policy:

```text
R1(config)# interface GigabitEthernet2
R1(config-if)# service-policy output WAN-QOS
```

The configuration creates one AF3 traffic class with a shared queue.

The class receives a minimum bandwidth allocation of 30% during congestion.

The queue can hold up to 200 packets, subject to platform buffering behavior.

WRED begins probabilistic dropping according to the DSCP-specific thresholds.

For example, AF33 packets can be dropped when the average queue depth exceeds 60 packets, even though the queue has not reached its 200-packet limit.

### WRED on class-default

WRED can also be enabled for traffic that does not match any preceding class:

```text
R1(config)# policy-map WAN-QOS
R1(config-pmap)# class class-default
R1(config-pmap-c)# random-detect dscp-based
```

This enables DSCP-based WRED for the `class-default` queue.

The exact default thresholds depend on the platform, queue limit, and applicable scheduling bandwidth.

---

## Configuring WRED Threshold Units

On supported IOS XE routers, WRED thresholds can be configured using packets, bytes, or milliseconds.

### Packet-Based Thresholds

Packet-based thresholds are commonly the default.

For example:

```text
R1(config-pmap-c)# queue-limit 200 packets
R1(config-pmap-c)# random-detect dscp-based
R1(config-pmap-c)# random-detect dscp af31 50 150 10
```

This configures:
- Queue limit: 200 packets.
- Minimum WRED threshold: 50 packets.
- Maximum WRED threshold: 150 packets.

### Byte-Based Thresholds

On supported platforms:

```text
R1(config-pmap-c)# queue-limit 400000 bytes
R1(config-pmap-c)# random-detect dscp-based
R1(config-pmap-c)# random-detect dscp af31 150000 bytes 300000 bytes 10
```

This configures:
- Queue limit: 400,000 bytes.
- Minimum WRED threshold: 150,000 bytes.
- Maximum WRED threshold: 300,000 bytes.

### Time-Based Thresholds

On supported platforms, thresholds can also be configured in milliseconds:

```text
R1(config-pmap-c)# queue-limit 50 ms
R1(config-pmap-c)# random-detect dscp-based
R1(config-pmap-c)# random-detect dscp af31 20 ms 40 ms 10
```

The router converts time-based thresholds into buffer quantities using the relevant calculated queue service rate.

For example, at 20 Mbps:

```text
20 ms = 20,000,000 × 0.020 / 8
      = 50,000 bytes

40 ms = 20,000,000 × 0.040 / 8
      = 100,000 bytes
```

These time values represent buffering thresholds, not guaranteed packet delays.

### Important Restrictions

On the documented QFP implementation:
- The WRED thresholds and queue limit must use compatible units.
- Packet-based and byte-based thresholds cannot be mixed within the same queue.
- The queue limit should exceed the maximum WRED threshold.
- The queue's unit mode may not be changeable while the service policy remains attached.

For example, configuring byte-based WRED thresholds while retaining a packet-based queue limit may be rejected.

See [Queue Limits and Tail Drop](queue-limits.md).

---

## WRED and TCP Global Synchronization

WRED was designed partly to reduce **TCP global synchronization**.

Global synchronization can occur when several TCP flows share a queue using tail drop.

When the queue fills, packets from several flows may be dropped during the same congestion episode.

The senders then reduce their sending rates around the same time.

After the queue drains, they increase their rates again.

This creates repeated cycles of queue buildup and packet loss.

WRED attempts to avoid this by dropping packets probabilistically before the queue fills.

Different TCP flows may therefore receive congestion signals at different times.

As individual senders reduce their rates, the queue may stabilize without requiring a large burst of packet losses.

However, WRED does not guarantee that global synchronization will be eliminated.

Its effectiveness depends on the traffic mix, queue configuration, and congestion-control behavior of the endpoints.

---

## WRED and ECN

WRED can also work with **Explicit Congestion Notification (ECN)**.

Without ECN, WRED signals congestion by dropping selected packets.

With ECN, eligible packets can be marked with the Congestion Experienced (CE) codepoint instead of being dropped.

Enable ECN with WRED:

```text
R1(config-pmap-c)# random-detect dscp-based
R1(config-pmap-c)# random-detect ecn
```

When WRED selects a packet for early congestion signaling:
- If the packet is ECN-capable, the router can mark CE instead of dropping it.
- If the packet is not ECN-capable, ordinary WRED dropping applies.

ECN does not prevent drops caused by exhausted queue capacity or other resource constraints.

For more information, see [Explicit Congestion Notification (ECN)](ecn.md).

---

## WRED Limitations and Design Considerations

### WRED Is Not a Scheduling Mechanism

WRED controls packet dropping, not the order in which queues receive transmission opportunities.

For example:

```text
R1(config-pmap-c)# bandwidth percent 30
R1(config-pmap-c)# random-detect dscp-based
```

The `bandwidth` command provides the minimum bandwidth allocation.

The `random-detect` command determines which packets may be dropped as congestion develops.

### WRED Is Generally Not Used for Voice Priority Queues

Voice traffic is typically placed into a strict priority queue using LLQ.

WRED is generally more appropriate for elastic data traffic.

Real-time voice traffic normally does not benefit from intentionally introducing additional packet loss to signal congestion.

Platform-specific restrictions also determine whether WRED can be combined with priority queueing.

### WRED Does Not Prevent Every Tail Drop

WRED bases its early dropping decisions on average queue depth.

A sufficiently large microburst can fill the actual queue before the calculated average reaches a WRED threshold.

The router may therefore perform queue-limit tail drops even when WRED is enabled.

### WRED Cannot Solve Sustained Oversubscription Alone

If traffic sources continuously send more traffic than an interface can transmit, WRED cannot create additional capacity.

It may reduce offered load when senders respond to congestion signals, but unresponsive traffic may continue filling the queue.

### Hierarchical QoS Restrictions

On ASR 1000 QFP-based routers, WRED is applied to actual queueing classes.

A parent class containing a child queuing policy does not itself have an output packet queue and therefore cannot use WRED at that level.

WRED should be configured in the appropriate child queueing class.

The restrictions differ between IOS XE platforms, particularly between router and Catalyst switching architectures.

---

## WRED Verification

The primary verification command is:

```text
R1# show policy-map interface GigabitEthernet2
```

This displays the output service policy and its per-class statistics.

For classes using WRED, the output can include:
- Queue limit.
- Current queue depth.
- Total packet drops.
- Exponential weighting constant.
- Average queue depth.
- Transmitted packets and bytes.
- Random-drop counters.
- Tail-drop counters.
- Minimum and maximum thresholds.
- Mark probability.

### Interpreting WRED Statistics

Important fields include:

| Field | Meaning |
|---|---|
| `queue depth` | Current instantaneous queue occupancy |
| `Mean queue depth` | Average queue depth used by WRED |
| `Exp-weight-constant` | Smoothing factor for average queue depth |
| `Random drop` | Packets discarded by probabilistic WRED decisions |
| `Tail drop` | Packets discarded when the applicable WRED maximum threshold is exceeded |
| `Minimum thresh` | Threshold at which early dropping begins |
| `Maximum thresh` | Threshold above which WRED drops all applicable arriving packets |
| `Mark prob` | Configured maximum random-drop probability |

The `Tail drop` field in WRED's per-profile statistics can represent drops caused by exceeding that profile's maximum average-queue threshold.

This is distinct from a drop caused by reaching the queue's instantaneous `queue-limit`.

The overall queue drop counters can include multiple dropping causes.

### Example Interpretation

Suppose the output reports:

```text
Queue limit:      200 packets
Mean queue depth: 90 packets

DSCP   Min   Max   Mark prob
AF31   100   160   1/20
AF32    80   160   1/10
AF33    60   160   1/5
```

At the current average depth:
- AF31 is below its minimum threshold and is not subject to WRED early dropping.
- AF32 is in its random-drop region.
- AF33 is also in its random-drop region, with a more aggressive configured drop profile.

These are the current conditions. Cumulative counters may also include drops from earlier periods when queue depth was different.

### Additional Commands

| Command | Purpose |
|---|---|
| `show policy-map WAN-QOS` | Display policy configuration |
| `show policy-map interface GigabitEthernet2` | Verify WRED thresholds, queue depths, and drops |
| `show interfaces GigabitEthernet2` | Check interface utilization and output drops |
| `show queueing random-detect` | Display WRED configuration on supported IOS implementations |
| `show platform hardware qfp active bqs 0 packet-buffer utilization` | Examine packet-buffer usage on supported QFP platforms |

### Troubleshooting WRED

**No random drops are observed**
- Verify that WRED is enabled in the correct class.
- Check the average queue depth against the minimum thresholds.
- Confirm that the class receives enough traffic to become congested.
- Verify that the policy is attached in the correct direction.

**Unexpectedly high packet loss**
- Inspect minimum and maximum thresholds.
- Check whether the mark probability denominator is too small.
- Determine whether sustained congestion is exceeding the maximum WRED threshold.
- Examine queue-limit drops and other resource-related drops.

**WRED configuration is rejected**
- Verify platform support.
- Check the queueing actions configured in the class.
- Verify that queue limits and WRED thresholds use compatible units.
- Check whether the queue's hardware mode must be changed with the service policy detached.

**Different DSCP values receive unexpected treatment**
- Verify the packets' actual markings.
- Check the configured drop curves.
- Confirm that the traffic matches the intended MQC class.
- Remember that the drop probability depends on the average queue depth, not simply the DSCP value.

---

## Key Points

- **WRED** is a congestion avoidance mechanism that probabilistically drops packets before queues become full.
- WRED uses average queue depth, whereas queue-limit tail drop uses instantaneous queue depth.
- WRED is designed to work particularly well with elastic traffic such as TCP.
- WRED can reduce TCP global synchronization.
- The three main WRED parameters are minimum threshold, maximum threshold, and mark probability denominator.
- Below the minimum threshold, no WRED early drops occur.
- Between the minimum and maximum thresholds, packets are randomly dropped with increasing probability.
- At or above the maximum threshold, conventional WRED drops all arriving packets for the affected drop profile.
- The default mark probability denominator is commonly 10.
- **A larger mark probability denominator produces a lower random-drop probability.**
- The exponential weighting constant controls how quickly average queue depth responds to changes.
- `random-detect` enables precedence-based WRED by default on documented IOS XE router implementations.
- `random-detect dscp-based` enables DSCP-based WRED.
- `random-detect discard-class-based` enables internal discard-class-based WRED.
- AF31, AF32, and AF33 can share one queue while receiving different dropping treatment.
- WRED does not determine packet scheduling order.
- On supported IOS XE platforms, WRED thresholds can be configured in packets, bytes, or milliseconds.
- WRED can be combined with ECN to mark eligible packets instead of dropping them.
- `show policy-map interface` is the primary WRED verification command.

---

## References

- [Cisco IOS XE: Queue Limits and WRED](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/quality-of-service/quality-of-service/m_qos-queue_management_and_congestion_avoidance.html)
- [Cisco IOS XE 17.x: Byte-Based Weighted Random Early Detection](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/qos/b-quality-of-service/m_qos-conavd-bb-wred.html)
- [Cisco IOS XE: QoS Time-Based Thresholds for WRED and Queue Limit](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/qos_conavd/configuration/xe-3s/qos-conavd-xe-3s-book/qos-conavd-time-wred.html)
- [Cisco IOS Quality of Service Solutions Command Reference: Q through R](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/qos/command/qos-cr-book/qos-q1.html)
