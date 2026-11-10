# 4.4.f Congestion Management and Avoidance

**Congestion management** and **congestion avoidance** are Quality of Service (QoS) mechanisms used to control how packets are handled when network resources become congested.

Although closely related, they serve different purposes:

- **Congestion management:** Determines the order in which packets are transmitted using queuing and scheduling mechanisms.
- **Congestion avoidance:** Attempts to prevent excessive queue buildup by selectively dropping or marking packets before queues become full.

These mechanisms are particularly important on WAN links, where traffic arriving from high-speed LAN interfaces may exceed the available transmission capacity.

Cisco IOS XE implements congestion management and avoidance using mechanisms such as **Class-Based Weighted Fair Queuing (CBWFQ)**, **Low Latency Queuing (LLQ)**, **Weighted Random Early Detection (WRED)**, and **Explicit Congestion Notification (ECN)**.

---

## Understanding Network Congestion

**Congestion** occurs when packets arrive at a network resource faster than that resource can process or transmit them.

For example, consider a router receiving traffic from a 1-Gbps LAN and forwarding it over a 100-Mbps WAN connection.

```text
       LAN                         WAN
      1 Gbps                     100 Mbps
        |                            |
        v                            v
   +----------+                 +----------+
   |    R1    |---------------->|    R2    |
   +----------+                 +----------+
        |
        |
   Traffic arriving
   faster than it can
   be transmitted
```

If R1 receives 200 Mbps of traffic destined for the 100-Mbps WAN link, it cannot transmit all the traffic immediately.

Packets must either:

1. Be placed into an output queue and transmitted later.
2. Be discarded when buffering resources are exhausted or a configured dropping mechanism selects them for removal.

When packets accumulate in a queue, they experience **queuing delay**.

If the queue continues growing, packets may eventually be dropped.

### Effects of Congestion

| Effect | Description |
|---|---|
| **Increased delay** | Packets spend more time waiting in queues |
| **Jitter** | Variations in queue occupancy cause variations in packet delay |
| **Packet loss** | Packets are discarded when queues reach their limits or congestion avoidance mechanisms select them for dropping |
| **Reduced application performance** | TCP retransmissions, congestion control, and increased latency may reduce application performance |

Congestion can occur even when an interface's average utilization is relatively low.

For example, a short **microburst** can temporarily overwhelm an output queue even if the average traffic rate remains below the interface's capacity.

This is why both sustained utilization and short-term traffic bursts matter when designing QoS policies.

---

## Congestion Management

**Congestion management** determines how packets are buffered and scheduled for transmission when multiple packets compete for an output interface.

Its two fundamental components are:

- **Queuing:** Organizing packets into one or more queues.
- **Scheduling:** Deciding which queue is serviced next and how much transmission capacity each queue receives.

Consider three types of traffic sharing a congested WAN link:

```text
       Incoming Traffic
               |
               v
        Classification
               |
       +-------+-------+
       |       |       |
       v       v       v
    +-----+ +-----+ +-----+
    |Voice| |Data | |Bulk |
    |Queue| |Queue| |Queue|
    +-----+ +-----+ +-----+
       |       |       |
       +-------+-------+
               |
               v
         Scheduler
               |
               v
       Outgoing Interface
```

Rather than placing all packets into a single queue, a congestion management policy can assign packets to different queues.

The scheduler then determines how those queues share the available bandwidth.

For example:

- Voice traffic may receive strict priority.
- Business applications may receive a minimum bandwidth guarantee.
- Bulk transfers may receive a smaller share of the available bandwidth.

**Congestion management does not increase the capacity of an interface.** It determines how that capacity is distributed among competing traffic classes.

### When Congestion Management Takes Effect

Queuing and scheduling mechanisms become particularly important when there are more packets waiting for transmission than the interface can immediately send.

When an interface is not congested, packets generally pass through its queues without significant waiting.

When congestion occurs, the scheduling algorithm determines which queued packets are transmitted first.

A packet that has already begun transmission generally cannot be interrupted by the arrival of a higher-priority packet. Therefore, strict priority queuing cannot eliminate serialization delay.

See [Queuing and Scheduling Fundamentals](queuing-fundamentals.md) for additional details.

---

## Congestion Management Mechanisms

Cisco supports several queuing and scheduling mechanisms.

| Mechanism | Description | Primary purpose |
|---|---|---|
| **FIFO** | Packets are transmitted in arrival order | Simple best-effort forwarding |
| **WFQ** | Separates traffic into flows and schedules them according to weights | Fairness among competing flows |
| **CBWFQ** | Uses administrator-defined classes with bandwidth allocations | Bandwidth guarantees for traffic classes |
| **LLQ** | Adds strict priority queuing to CBWFQ | Low delay and jitter for real-time traffic |

### First In, First Out (FIFO)

**FIFO** is the simplest queuing mechanism.

All packets enter a single queue and are transmitted in the order they arrive.

```text
             FIFO Queue
        +----+----+----+----+
        | P4 | P3 | P2 | P1 |----> Output
        +----+----+----+----+
                            ^
                      Oldest packet
```

FIFO does not distinguish between traffic classes.

A large file transfer can therefore cause voice packets to wait behind many data packets.

FIFO is simple and has minimal scheduling complexity, but it does not provide differentiated forwarding treatment.

### Weighted Fair Queuing (WFQ)

**Weighted Fair Queuing (WFQ)** provides fairer treatment among competing traffic flows.

Rather than placing all traffic into one FIFO queue, flow-based WFQ identifies individual flows and uses separate queues to schedule them.

The scheduler uses weights to determine how traffic is serviced.

For example, WFQ helps prevent a large file transfer from monopolizing transmission opportunities at the expense of a low-bandwidth interactive application.

Traditional Cisco WFQ is also aware of IP Precedence, allowing different flows to receive different scheduling weights.

WFQ operates automatically without requiring an administrator to manually define individual traffic classes.

However, it offers less explicit control than CBWFQ.

### Class-Based Weighted Fair Queuing (CBWFQ)

**Class-Based Weighted Fair Queuing (CBWFQ)** extends weighted fair scheduling by allowing administrators to define traffic classes.

Each class can receive its own queue and bandwidth allocation.

For example:

| Class | Minimum bandwidth |
|---|---:|
| Business applications | 40% |
| Video | 30% |
| Other traffic | Remaining capacity |

A class with a configured bandwidth guarantee receives at least its allocated bandwidth when sufficient matching traffic is offered during congestion, subject to the configured scheduling policy and platform behavior.

**A CBWFQ bandwidth guarantee is not a maximum bandwidth limit.**

If other classes are not using their allocations, a class can generally consume additional available bandwidth.

For example, configuring:

```text
R1(config-pmap-c)# bandwidth percent 40
```

Allocates a minimum of 40% of the applicable scheduling bandwidth to that class during congestion.

It does not restrict the class to 40% when additional bandwidth is available.

### Low Latency Queuing (LLQ)

**Low Latency Queuing (LLQ)** combines CBWFQ with strict priority queuing.

It allows delay-sensitive traffic, such as voice, to be transmitted before packets waiting in ordinary bandwidth queues.

For example:

```text
       +-------------------+
       | Voice Priority    |----+
       | Queue             |    |
       +-------------------+    |
                                v
       +-------------------+  Scheduler
       | Business Queue    |     |
       +-------------------+     |
                                 v
       +-------------------+   Output
       | Best Effort Queue |  Interface
       +-------------------+
```

In MQC, a priority queue can be configured with:

```text
R1(config-pmap-c)# priority percent 10
```

This provides strict priority treatment to the class.

On supported IOS XE scheduling implementations, specifying a rate with `priority` also creates a conditional priority policer that enforces the configured rate during congestion.

This helps prevent excessive priority traffic from starving other classes.

Some platforms also support multiple priority levels, typically using `priority level 1` and `priority level 2`.

**The key difference between CBWFQ and LLQ is that CBWFQ provides weighted scheduling, while LLQ adds strict priority scheduling.**

See [FIFO, WFQ, and CBWFQ](fifo-wfq-cbwfq.md) and [Low Latency Queuing (LLQ)](llq.md).

---

## Bandwidth Allocation and Scheduling

Cisco IOS XE supports several MQC commands for controlling how queues receive transmission opportunities.

| Command | Purpose |
|---|---|
| `bandwidth 20000` | Configure a minimum bandwidth of 20,000 Kbps |
| `bandwidth percent 30` | Configure a minimum bandwidth allocation of 30% |
| `bandwidth remaining percent 30` | Configure a relative share of bandwidth available to nonpriority queues |
| `bandwidth remaining ratio 3` | Configure a relative excess-bandwidth scheduling weight |
| `priority percent 10` | Configure a strict priority queue with a percentage-based conditional priority rate |
| `queue-limit 200` | Configure a queue limit of 200 packets, on platforms using packet-based queue limits |

An important distinction exists between `bandwidth percent` and `bandwidth remaining percent`.

- `bandwidth percent` specifies a minimum bandwidth allocation.
- `bandwidth remaining percent` influences how available bandwidth is shared among nonpriority queues.

The `bandwidth remaining ratio` command provides relative scheduling weights rather than specifying an absolute percentage.

For example, two queues configured with excess-bandwidth ratios of 3 and 1 have a 3:1 weighting for sharing excess bandwidth, subject to the scheduler and other configured constraints.

The exact implementation and supported combinations of these commands depend on the IOS XE platform.

---

## Congestion Avoidance

**Congestion avoidance** attempts to control queue buildup before queues become completely full.

Traditional congestion avoidance primarily relies on sending early congestion signals to traffic sources.

These signals can take the form of:

- **Packet drops:** Discarding selected packets before the queue becomes full.
- **ECN marking:** Marking eligible packets to indicate congestion without necessarily dropping them.

For TCP traffic, packet loss or ECN feedback can cause senders to reduce their transmission rates.

This can help prevent persistent queue growth and reduce the likelihood of large bursts of packet loss.

Congestion avoidance does not guarantee that congestion will be eliminated. Its effectiveness depends partly on how traffic sources react to congestion signals.

---

## Queue Limits and Tail Drop

Every queue has a finite amount of buffering available to it.

The **queue limit** defines the maximum amount of data that a queue may hold.

Depending on the platform and configuration, queue limits can be expressed in packets, bytes, or time.

When a packet arrives and the queue has already reached its configured limit, the packet may be discarded.

This behavior is known as **tail drop**.

```text
          Queue Limit Reached
       +----+----+----+----+
       | P4 | P3 | P2 | P1 |----> Output
       +----+----+----+----+
         ^
         |
      New packet
       DROPPED
```

Tail drop makes no distinction between packets based on their flow or priority unless other queue classification or drop mechanisms have already separated them.

### Problems with Tail Drop

Tail drop can contribute to several undesirable behaviors:

- **Large bursts of packet loss:** Several packets may be dropped when a full queue receives additional traffic.
- **TCP global synchronization:** Multiple TCP flows may reduce their sending rates around the same time and then increase them together.
- **Excessive queuing delay:** Large buffers may allow packets to accumulate for long periods before being transmitted.

The final problem is commonly associated with **bufferbloat**, where excessive buffering causes unnecessarily high latency.

Increasing queue limits may reduce drops caused by temporary bursts, but it can also increase delay.

Decreasing queue limits may reduce queuing delay but increase packet loss.

Consequently, queue limits should be selected according to the traffic characteristics and available bandwidth.

See [Queue Limits and Tail Drop](queue-limits.md).

---

## Weighted Random Early Detection (WRED)

**Weighted Random Early Detection (WRED)** is a congestion avoidance mechanism that selectively drops packets before a queue becomes full.

WRED extends **Random Early Detection (RED)** by allowing different traffic markings to receive different dropping behavior.

Cisco IOS XE supports WRED based on criteria such as:

- IP Precedence.
- DSCP.
- Internal discard class, on supported platforms.

WRED typically uses an **average queue depth** rather than relying only on the instantaneous queue depth.

It compares that average with two configurable thresholds.

### WRED Thresholds

| Average queue depth | WRED behavior |
|---|---|
| Below minimum threshold | No WRED early drops |
| Between minimum and maximum thresholds | Randomly drop packets with increasing probability |
| At or above maximum threshold | Drop packets according to the WRED maximum-threshold behavior |

The general RED dropping curve can be represented as:

```text
 Drop
 Probability
 100% |                              +----------
      |                              |
      |                              |
      |                             /
      |                           /
      |                        /
      |                     /
   0% +-------------------+
      |
      +---------------------------------------->
                     Average Queue Depth
                         Min      Max
```

The probability rises from zero at the minimum threshold to a configured maximum-marking probability near the maximum threshold.

At the maximum threshold, conventional WRED transitions to dropping all eligible packets, although ECN and platform-specific behavior can affect the action taken.

WRED can use different threshold values for different DSCP markings.

For example, within the AF3 class:

- AF31 can receive the lowest drop probability.
- AF32 can receive a higher drop probability.
- AF33 can receive the highest drop probability.

This implements the relative drop precedence intended by Assured Forwarding.

These relationships depend on the configured thresholds and drop probabilities. WRED does not inherently guarantee that all higher numerical DSCP values receive better treatment.

### WRED and TCP Global Synchronization

Without WRED, multiple TCP flows may fill the same queue and experience packet loss at approximately the same time.

The flows then reduce their transmission rates together.

After congestion subsides, they increase their transmission rates and may cause another wave of congestion.

This repeated behavior is called **TCP global synchronization**.

WRED attempts to reduce synchronization by randomly selecting packets for early dropping.

Because different flows are less likely to receive congestion signals simultaneously, the network can maintain more stable utilization.

WRED is generally most effective for responsive traffic such as TCP.

It is less effective for traffic sources that do not reduce their sending rates in response to packet loss.

### Configuring WRED

WRED can be enabled within an MQC policy class.

For example:

```text
R1(config)# policy-map WAN-QOS
R1(config-pmap)# class BULK
R1(config-pmap-c)# bandwidth percent 20
R1(config-pmap-c)# random-detect dscp-based
```

This enables DSCP-based WRED for the BULK traffic class.

Specific DSCP thresholds can also be configured, and the WRED exponential weighting constant can be adjusted.

See [Weighted Random Early Detection (WRED)](wred.md).

---

## Explicit Congestion Notification (ECN)

**Explicit Congestion Notification (ECN)** allows congestion to be signaled by marking packets rather than dropping them.

ECN uses the lower two bits of the IPv4 Differentiated Services field or the IPv6 Traffic Class field.

```text
       IPv4 DS Field / IPv6 Traffic Class

       +-----------------------+--------+
       |         DSCP          |  ECN   |
       +-----------------------+--------+
                6 bits           2 bits
```

The four ECN codepoints are:

| Binary | Name | Meaning |
|---|---|---|
| `00` | Not-ECT | Not ECN-capable transport |
| `01` | ECT(1) | ECN-capable transport |
| `10` | ECT(0) | ECN-capable transport |
| `11` | CE | Congestion Experienced |

ECN-capable endpoints can transmit packets marked ECT(0) or ECT(1).

When a router detects congestion, it can change the ECN field to `11` (CE) instead of dropping an eligible packet.

The receiving endpoint then provides congestion feedback to the sender through the transport protocol.

For example, TCP can use ECN feedback to adjust its sending rate.

The important distinction is that **ECN communicates congestion without requiring packet loss as the initial congestion signal**.

However, ECN does not eliminate all drops. Packets can still be discarded when queue limits or other dropping conditions are reached.

### ECN with WRED

Cisco IOS XE supports ECN as an extension of WRED.

It can be enabled using:

```text
R1(config-pmap-c)# random-detect dscp-based
R1(config-pmap-c)# random-detect ecn
```

When WRED selects a packet for early congestion signaling:

- An eligible ECN-capable packet can be marked CE instead of being dropped.
- A non-ECN-capable packet is subject to ordinary WRED dropping.

See [Explicit Congestion Notification (ECN)](ecn.md).

---

## Congestion Management vs. Congestion Avoidance

The following table summarizes the distinction between these QoS mechanisms.

| Feature | Congestion management | Congestion avoidance |
|---|---|---|
| Primary objective | Decide transmission order and bandwidth allocation | Control queue buildup |
| Main mechanisms | FIFO, WFQ, CBWFQ, LLQ | WRED, ECN |
| Main operation | Queuing and scheduling | Early dropping or marking |
| Controls delay | Yes, by prioritizing or scheduling packets | Indirectly, by limiting queue growth |
| Controls packet loss | Indirectly, through scheduling | Yes, through controlled congestion signaling |
| Main IOS XE commands | `bandwidth`, `priority`, `fair-queue` | `random-detect`, `random-detect ecn` |

**Tail drop** is a queue overflow mechanism rather than a proactive congestion avoidance mechanism.

These mechanisms can be combined.

For example, CBWFQ can allocate a minimum bandwidth to a business application, while WRED manages congestion within that application's queue.

---

## MQC Configuration Example

Consider a router with a 100-Mbps WAN interface.

The objective is to provide differentiated treatment to three traffic classes.

| Traffic | DSCP | Treatment |
|---|---|---|
| Voice | EF | Strict priority, 10% |
| Business applications | AF31 | Minimum bandwidth, 30% |
| Bulk data | AF11 | Minimum bandwidth, 20%, WRED |
| Other traffic | Any remaining unmatched DSCP | Default forwarding |

Assume the physical interface's effective transmission rate is 100 Mbps, with no separate provider-imposed bandwidth limit.

### Step 1: Configure Class Maps

```text
R1(config)# class-map match-any VOICE
R1(config-cmap)# match dscp ef
R1(config)# class-map match-any BUSINESS
R1(config-cmap)# match dscp af31
R1(config)# class-map match-any BULK
R1(config-cmap)# match dscp af11
```

These classes identify packets using their existing DSCP markings.

### Step 2: Configure the Policy Map

```text
R1(config)# policy-map WAN-QOS
R1(config-pmap)# class VOICE
R1(config-pmap-c)# priority percent 10
R1(config-pmap)# class BUSINESS
R1(config-pmap-c)# bandwidth percent 30
R1(config-pmap)# class BULK
R1(config-pmap-c)# bandwidth percent 20
R1(config-pmap-c)# random-detect dscp-based
R1(config-pmap)# class class-default
```

This policy provides:

- **VOICE:** Strict priority scheduling, with a conditional priority rate of 10 Mbps during congestion on supported implementations.
- **BUSINESS:** A minimum bandwidth allocation of 30 Mbps during congestion.
- **BULK:** A minimum bandwidth allocation of 20 Mbps, with WRED managing congestion within its queue.
- **class-default:** Default treatment for unmatched traffic, which can use available bandwidth subject to the scheduler.

The configured class bandwidth allocations are not hard caps.

Nonpriority classes can generally consume additional available capacity when other classes do not require it.

### Step 3: Apply the Policy Outbound

```text
R1(config)# interface GigabitEthernet1
R1(config-if)# service-policy output WAN-QOS
```

The policy is applied in the output direction because the objective is to control traffic waiting to leave the interface.

**Important:** If the physical interface is 1 Gbps but the service provider limits traffic to 100 Mbps, the router may need a parent shaping policy to create a local congestion point at the contracted rate.

Without shaping, packets might leave the router at 1 Gbps and be discarded by the provider's policer instead of being properly scheduled by the router.

This design is covered in [4.4.g Hierarchical QoS (HQoS)](../hqos/overview.md).

---

## Verification and Troubleshooting

The primary command for verifying congestion management and avoidance policies is:

```text
R1# show policy-map interface GigabitEthernet1
```

This displays the applied policy and the available per-class statistics.

Depending on the platform and configured features, the output can include:

- Packets and bytes matched by each class.
- Configured bandwidth or priority allocations.
- Offered and transmitted traffic rates.
- Current queue depth and queue limits.
- Total packet drops.
- Tail-drop counters.
- WRED random-drop counters.
- WRED minimum and maximum thresholds.
- ECN marking statistics.

### Useful Commands

| Command | Purpose |
|---|---|
| `show policy-map` | Display configured QoS policy maps |
| `show policy-map WAN-QOS` | Display the configured scheduling and avoidance actions |
| `show policy-map interface GigabitEthernet1` | Display applied policies and per-class statistics |
| `show policy-map interface GigabitEthernet1 output` | Display outbound QoS statistics |
| `show class-map` | Display class maps and matching criteria |
| `show interfaces GigabitEthernet1` | Check interface utilization and output drops |
| `show running-config interface GigabitEthernet1` | Verify the attached service policy |

Available counters and their formatting depend on the IOS XE release and hardware platform.

### Common Troubleshooting Considerations

**Voice traffic experiences excessive delay or jitter**
- Verify that packets are correctly classified into the priority class.
- Check for priority policer drops.
- Confirm that the priority allocation is sufficient for the actual voice traffic.
- Check for upstream or downstream congestion.

**Business traffic does not receive its expected bandwidth**
- Verify that packets match the correct class.
- Confirm the policy is applied to the congested egress interface.
- Verify bandwidth allocations and competing traffic.
- Check whether the actual bottleneck is outside the router.

**WRED does not appear to drop packets**
- Check whether the average queue depth reaches the configured minimum threshold.
- Verify that WRED is enabled on the correct class.
- Confirm that the class is experiencing congestion.

**Excessive tail drops**
- Examine queue limits and the instantaneous queue occupancy.
- Determine whether congestion is sustained or caused by microbursts.
- Check for insufficient bandwidth allocations.
- Consider whether queue limits, WRED thresholds, or shaping rates require adjustment.

---

## Key Points

- **Congestion management** controls packet transmission order through queuing and scheduling.
- **Congestion avoidance** attempts to control queue buildup through early dropping or congestion marking.
- **FIFO** transmits packets in arrival order without distinguishing traffic classes.
- **WFQ** provides weighted fairness among individual flows.
- **CBWFQ** provides administrator-defined classes with minimum bandwidth allocations.
- **LLQ** adds strict priority scheduling for delay-sensitive traffic.
- `bandwidth percent` provides a minimum bandwidth allocation, not a maximum transmission rate.
- **Tail drop** discards packets when a queue reaches its configured limit.
- **WRED** selectively drops packets based on average queue depth and configurable thresholds.
- **WRED reduces the likelihood of TCP global synchronization** by providing early, randomized congestion signals.
- **ECN** allows eligible packets to be marked CE instead of being dropped for early congestion signaling.
- Congestion management and avoidance generally operate at output congestion points.
- `show policy-map interface` is the primary IOS XE command for verifying these mechanisms.

---

## References

- [Cisco IOS XE 17.x: QoS Scheduling](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/qos/b-quality-of-service/m_qos-scheduling.html)
- [Cisco IOS XE 17: Congestion Management Overview](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/qos_conmgt/configuration/xe-17/qos-conmgt-xe-17-book/qos-conmgt-oview.html)
- [Cisco IOS XE 17.x: Queue Limits and WRED](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/qos/b-quality-of-service/m_qos-queue_management_and_congestion_avoidance.html)
- [Cisco IOS XE 17.x: Congestion Avoidance Overview](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/qos/b-quality-of-service/m_qos-conavd-oview-0.html)
- [Cisco IOS XE 17.x: Configuring Weighted Random Early Detection](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/qos/b-quality-of-service/m_qos-conavd-cfg-wred-0.html)
- [Cisco IOS XE 17.x: WRED Explicit Congestion Notification](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/qos/b-quality-of-service/m_qos-conavd-wred-ecn-0.html)