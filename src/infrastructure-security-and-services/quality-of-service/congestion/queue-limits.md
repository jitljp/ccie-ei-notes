# Queue Limits and Tail Drop

**Queue limits** determine the maximum amount of data that can be buffered in an output queue.

When a queue reaches its configured limit, additional arriving packets may be discarded using **tail drop**.

Queue limits serve two primary purposes:
- **Limit queuing delay:** Prevent excessive buffering from introducing unacceptable latency.
- **Protect buffer resources:** Prevent individual queues from consuming excessive amounts of shared packet memory.

Cisco IOS XE supports configurable queue limits through the Modular QoS CLI (MQC).

On supported QFP-based routers, queue limits can be configured in packets, bytes, or milliseconds.

---

## Queue Limits

A **queue limit** defines the maximum amount of data that a queue can hold while waiting for transmission.

Consider a router receiving more traffic than its outgoing interface can transmit.

```text
        Incoming Traffic
               |
               v
        +-------------+
        | Output Queue|
        |             |
        | P4 P3 P2 P1 |----> Interface
        +-------------+
               ^
               |
          Queue Limit
```

When packets arrive:

1. The router determines the appropriate output queue.
2. It checks the queue's current depth against its configured limit.
3. If sufficient space is available, the packet is enqueued.
4. The scheduler removes packets as transmission opportunities become available.
5. If the queue has reached its limit, additional packets may be dropped.

The amount of data currently buffered is called the **queue depth**.

Queue depth increases when packets arrive faster than they are transmitted.

Queue depth decreases when the scheduler transmits packets faster than new packets arrive.

### Why Queue Limits Matter

Without an appropriate queue limit, packets could accumulate for an extended period during congestion.

For example, consider a queue containing 1,000 packets, each 1,500 bytes.

The total buffered data is:

```text
1,000 × 1,500 × 8 = 12,000,000 bits
```

The time required to drain that queue depends on the service rate.

| Queue service rate | Time to transmit 1,000 packets |
|---|---:|
| 1 Gbps | 12 ms |
| 100 Mbps | 120 ms |
| 10 Mbps | 1.2 seconds |
| 1 Mbps | 12 seconds |

These calculations assume a constant queue service rate and exclude additional framing overhead.

The same queue depth can therefore introduce very different amounts of delay depending on the available bandwidth.

**Queue limits should be considered in relation to the queue's service rate, not simply the number of packets it can hold.**

---

## Tail Drop

**Tail drop** is a queue management mechanism that discards arriving packets when a queue reaches its configured limit.

For example, suppose a FIFO queue has a limit of four packets.

```text
           Queue Limit = 4 Packets

           +----+----+----+----+
New P5 ---X| P4 | P3 | P2 | P1 |----> Output
         | +----+----+----+----+
         |
        DROP
```

When P5 arrives:
- The queue already contains four packets.
- The queue limit has been reached.
- P5 is discarded.
- The existing queued packets remain in the queue.

The term *tail drop* refers to dropping packets arriving at the tail of a full queue.

### Tail Drop Uses Instantaneous Queue Depth

Tail-drop decisions are based on the **instantaneous queue depth**.

For example, if the queue limit is 200 packets:

| Current queue depth | New packet |
|---|---|
| 100 packets | Enqueued |
| 199 packets | Enqueued, assuming one packet of available capacity |
| 200 packets | Dropped |

For byte-based queue limits, the size of the arriving packet must also be considered.

For example, if only 500 bytes of capacity remain, a new 1,500-byte packet cannot be enqueued.

Tail drop does not consider the average queue depth over time.

This distinguishes it from WRED, which uses an average queue-depth calculation to make early drop decisions.

### Tail Drop Is Not Proactive Congestion Avoidance

Tail drop reacts when a queue reaches its limit.

It does not intentionally provide early congestion signals before that limit is reached.

For example:

```text
Queue Depth
     |
 100 |                       XXXXX
     |                       XXXXX
  75 |                  XXXXX
     |             XXXXX
  50 |        XXXXX
     |   XXXXX
   0 +-------------------------------->
              Time

X = Queued packets

Tail drops begin at the queue limit.
```

By contrast, WRED can begin dropping packets before the queue becomes full.

See [Weighted Random Early Detection (WRED)](wred.md).

---

## Problems with Tail Drop

Tail drop is simple, but it can cause several performance problems.

### TCP Global Synchronization

When multiple TCP flows share a congested queue, the queue may become full and drop packets from several flows within a short period.

The affected TCP senders detect congestion and reduce their sending rates.

As congestion subsides, the senders may increase their rates again.

If they increase together, the queue may fill again, producing another wave of packet loss.

This behavior is called **TCP global synchronization**.

```text
TCP Sending Rate
     ^
     |      /\        /\        /\
     |     /  \      /  \      /  \
     |    /    \    /    \    /    \
     |___/      \__/      \__/      \___
     |
     +------------------------------------> Time
```

The resulting synchronized increases and decreases can lead to inefficient bandwidth utilization.

WRED attempts to reduce this behavior by randomly dropping packets before the queue reaches its limit.

### Excessive Queuing Delay

Large queue limits can permit excessive amounts of data to accumulate.

For example, a queue limit of 1,000 packets on a 10-Mbps link could represent approximately 1.2 seconds of buffering for 1,500-byte packets.

Even if no packets are dropped, applications may experience unacceptable latency.

This phenomenon is commonly associated with **bufferbloat**.

### Burst Loss

When a queue becomes full, several consecutive arriving packets may be dropped.

For TCP applications, this can trigger retransmissions and congestion control.

For real-time applications such as voice, consecutive packet losses can significantly degrade audio quality.

---

## Queue Limit Configuration

On supported IOS XE routers, queue limits are configured within an MQC policy-map class.

The basic command is:

```text
R1(config-pmap-c)# queue-limit 200
```

This configures a queue limit of 200 packets.

The command can also explicitly specify the unit:

```text
R1(config-pmap-c)# queue-limit 200 packets
```

Cisco IOS XE supports three queue-limit units on applicable platforms:

| Unit | Example | Meaning |
|---|---|---|
| Packets | `queue-limit 200 packets` | Maximum of 200 packets |
| Bytes | `queue-limit 300000 bytes` | Maximum of 300,000 bytes |
| Time | `queue-limit 50 ms` | Buffering equivalent to 50 ms at the queue's calculated service rate |

The default unit for a queue created through MQC on the documented ASR 1000 QFP scheduler is packets.

### Queueing Action Requirement

On the documented QFP scheduler, `queue-limit` can be configured in a class with a queuing action such as:

- `bandwidth`
- `bandwidth remaining`
- `priority`
- `shape`

For example:

```text
R1(config)# policy-map WAN-QOS
R1(config-pmap)# class BUSINESS
R1(config-pmap-c)# bandwidth percent 30
R1(config-pmap-c)# queue-limit 200
```

This creates a bandwidth queue for BUSINESS with a maximum queue depth of 200 packets.

The queue limit does not change the class's bandwidth allocation.

It only controls how much traffic can be buffered.

---

## Packet-Based Queue Limits

With a **packet-based queue limit**, the queue can hold a specified number of packets.

For example:

```text
R1(config-pmap-c)# queue-limit 100 packets
```

The queue can hold up to 100 packets, regardless of their individual sizes.

This is easy to configure but does not directly control the amount of buffered data.

Consider two queues, each with a limit of 100 packets.

| Queue | Packet size | Maximum buffered data |
|---|---:|---:|
| A | 100 bytes | 10,000 bytes |
| B | 1,500 bytes | 150,000 bytes |

Both queues have identical packet limits.

However, Queue B can buffer 15 times as much data.

Consequently, packet-based limits do not guarantee consistent queuing delay when packet sizes differ.

---

## Byte-Based Queue Limits

A **byte-based queue limit** restricts the total amount of buffered data rather than the number of packets.

For example:

```text
R1(config-pmap-c)# queue-limit 150000 bytes
```

This limits the queue to approximately 150,000 bytes of buffered packet data, according to the platform's accounting rules.

The number of packets that can fit depends on their sizes.

For example:

| Average packet size | Approximate packets fitting in 150,000 bytes |
|---|---:|
| 100 bytes | 1,500 |
| 500 bytes | 300 |
| 1,500 bytes | 100 |

Byte-based limits provide more direct control over memory consumption.

They are particularly useful when packet sizes vary significantly.

**Important:** Byte-based queue limits and packet-based queue limits are not interchangeable without considering packet sizes.

### Changing Queue Limit Units

On QFP-based platforms, the queue's hardware mode may be determined when the queue is created.

The hardware can operate in packet mode or byte mode.

If a policy is already attached to an interface, changing its queue limit from packets to bytes may be rejected.

For example:

```text
R1(config-pmap-c)# queue-limit 5000 bytes
```

The router may report:

```text
Runtime changing queue-limit unit is not supported,
please remove service-policy first
```

To change the mode:

1. Remove the service policy from the interface.
2. Modify the queue-limit units.
3. Reattach the service policy.

For example:

```text
R1(config)# interface GigabitEthernet1
R1(config-if)# no service-policy output WAN-QOS
R1(config-if)# exit
R1(config)# policy-map WAN-QOS
R1(config-pmap)# class BUSINESS
R1(config-pmap-c)# queue-limit 150000 bytes
R1(config-pmap-c)# exit
R1(config-pmap)# exit
R1(config)# interface GigabitEthernet1
R1(config-if)# service-policy output WAN-QOS
```

This restriction is platform-specific, but it is important when troubleshooting policy changes on QFP-based routers.

---

## Time-Based Queue Limits

A **time-based queue limit** defines the amount of buffering in terms of transmission time.

For example:

```text
R1(config-pmap-c)# queue-limit 50 ms
```

This configures a queue limit equivalent to 50 milliseconds of traffic at the queue's calculated service rate.

On the QFP implementation, the hardware does not actually store queue depth in milliseconds.

Instead, IOS XE converts the configured time into an equivalent number of bytes.

The formula is:

```text
Queue limit (bytes) = Rate (bps) × Time (seconds) / 8
```

### Example: 20-Mbps Queue

Suppose a queue is shaped to 20 Mbps:

```text
R1(config)# policy-map WAN-QOS
R1(config-pmap)# class BULK
R1(config-pmap-c)# shape average 20000000
R1(config-pmap-c)# queue-limit 50 ms
```

The byte limit is calculated as:

```text
Queue limit = 20,000,000 × 0.050 / 8
            = 125,000 bytes
```

Therefore, the queue can hold approximately 125,000 bytes.

If the shaping rate were increased to 40 Mbps, the same 50-ms limit would correspond to:

```text
Queue limit = 40,000,000 × 0.050 / 8
            = 250,000 bytes
```

This illustrates the benefit of time-based queue limits.

They allow buffering to scale with the configured service rate.

### Important: Time-Based Limits Are Not Delay Guarantees

A queue limit of 50 ms does not guarantee that every packet experiences less than 50 ms of queuing delay.

The configured time is converted using the queue's calculated service rate.

Actual delay can be affected by:
- Other queues competing for bandwidth.
- Changes in available scheduling capacity.
- Parent shaping policies.
- Packet sizes and scheduling behavior.

The 50-ms value therefore represents a buffering allowance, not an absolute end-to-end delay guarantee.

---

## Default Queue Limits on QFP-Based Routers

Cisco ASR 1000 routers calculate default queue limits based on the type of queue and its configured bandwidth.

The following behavior describes Cisco's documented QFP implementation.

### Default Bandwidth Queue Limit

For ordinary bandwidth queues, Cisco targets approximately **50 ms of buffering**.

The default packet-based limit is calculated using:

```text
Queue limit (packets) =
    Visible bandwidth (bps) × 0.050
    --------------------------------
             8 × MTU
```

The **visible bandwidth** is the rate IOS XE derives from the queue's configuration.

It is used to determine an appropriate queue limit.

It is not necessarily the queue's actual transmission rate.

| Queue configuration | Visible bandwidth |
|---|---|
| `shape` | Shaping rate |
| `bandwidth` | Configured minimum bandwidth |
| `shape` and `bandwidth` | Configured minimum bandwidth |
| `bandwidth remaining` | Rate inherited from the parent |

For a flat policy, `bandwidth remaining` generally inherits the physical interface speed.

For a child policy beneath a parent shaper, it inherits the parent shaping rate.

### Example: 100-Mbps Queue

Assume:
- Visible bandwidth = 100 Mbps.
- MTU = 1,500 bytes.
- Default buffering target = 50 ms.

Calculate:

```text
Queue limit = 100,000,000 × 0.050
              -------------------
                   8 × 1500

            = 416.67 packets
```

The documented example uses a default of approximately 416 packets.

### Minimum Default Queue Limit

On the ASR 1000 scheduler, the calculated default limit for bandwidth queues has a minimum of 64 packets.

For example, suppose a queue has a visible bandwidth of 5 Mbps:

```text
Queue limit = 5,000,000 × 0.050
              -----------------
                  8 × 1500

            = 20.83 packets
```

Because this is below 64, the queue receives the minimum default limit of 64 packets.

This creates an important consequence:

**Low-bandwidth queues can have substantially more than 50 ms of buffering when the minimum packet limit applies.**

For example, 64 packets of 1,500 bytes at 5 Mbps represent:

```text
Delay = 64 × 1500 × 8 / 5,000,000
      = 0.1536 seconds
      = 153.6 ms
```

Therefore, manually reducing the queue limit may be useful when low latency is important.

### Default Priority Queue Limit

On the documented ASR 1000 scheduler, priority queues have a default limit of **512 packets**.

This is intentionally relatively large.

Cisco expects properly controlled priority queues to remain almost empty because they receive strict priority scheduling and typically use priority admission control.

However, a large priority queue limit should not be interpreted as a recommendation to buffer hundreds of voice packets.

If a priority queue regularly accumulates large amounts of data, investigate its offered rate, burst characteristics, and policing behavior.

### Newer Platform Differences

Do not assume all IOS XE routers use the same default formulas.

For example, Cisco documents different queue-limit minimums for certain ISR 1100 and Catalyst 8200/8300/8500L platforms.

For CCIE EI, understand the 50-ms calculation model, but verify the actual default on the platform being used.

---

## Queue Limits with Flow-Based Fair Queuing

On supported QFP-based routers, the MQC `fair-queue` command enables multiple flow queues within a class.

For example:

```text
R1(config)# policy-map WAN-QOS
R1(config-pmap)# class BUSINESS
R1(config-pmap-c)# bandwidth percent 30
R1(config-pmap-c)# fair-queue
```

On the ASR 1000 implementation:
- The class uses 16 flow queues.
- Each flow queue has its own queue limit.
- The default limit of each flow queue is one-quarter of the ordinary class queue's calculated limit.

### Example

Suppose a class would normally have a default queue limit of 416 packets.

With `fair-queue` enabled:

```text
Default class queue limit = 416 packets

Per-flow queue limit = 416 / 4
                     = 104 packets
```

Each of the 16 flow queues receives a default limit of 104 packets.

**The limit applies to each flow queue, not collectively to all 16 queues.**

This helps prevent individual flows from building excessively large backlogs.

It also means that a flow can experience tail drops even while other flow queues in the same class have unused capacity.

See [FIFO, WFQ, and CBWFQ](fifo-wfq-cbwfq.md).

---

## Tail Drop vs. Out-of-Resources Drops

On QFP-based routers, packet memory is generally shared among multiple interfaces and queues.

A queue limit constrains an individual queue's memory consumption, but it does not necessarily reserve that amount of memory exclusively for the queue.

Two distinct drop conditions are therefore possible.

### Tail Drop

A tail drop occurs when an individual queue reaches its configured queue limit.

For example:

```text
Queue limit: 200 packets
Queue depth: 200 packets

Shared memory: Available

Result: Tail drop
```

The packet is dropped even though the router still has unused packet memory.

### Out-of-Resources (OOR) Drop

An **Out-of-Resources (OOR)** drop occurs when shared buffering resources become unavailable.

For example:

```text
Queue limit: 200 packets
Queue depth: 100 packets

Shared memory: Exhausted

Result: No-buffer / OOR drop
```

The packet is dropped even though its individual queue has not reached its limit.

| Condition | Tail drop | OOR drop |
|---|---|---|
| Individual queue full | Yes | Not necessarily |
| Shared packet memory exhausted | Not required | Yes, or a protective resource threshold has been reached |
| Primary cause | Queue limit | Shared resource pressure |
| Typical counter | Queue drops | No-buffer drops |

### QFP Memory Protection

Cisco's ASR 1000 QFP architecture reserves some packet-memory capacity for important traffic.

The documented thresholds include:
- **85%:** Normal nonpriority traffic may be dropped to preserve memory for priority traffic.
- **98%:** A vital threshold may cause user traffic, including priority traffic, to be dropped to preserve resources for internal control traffic.

These thresholds are specific to the documented QFP implementation.

An OOR condition therefore does not necessarily mean every byte of physical packet memory has been consumed.

### Verifying Packet Buffer Utilization

On supported QFP routers:

```text
R1# show platform hardware qfp active bqs 0 packet-buffer utilization
```

This displays information such as:
- Total packet-buffer memory.
- Used and free packet-buffer memory.
- Current utilization.
- OOR thresholds.
- Vital thresholds.

If OOR drops occur frequently, consider whether individual queues have excessively large limits.

Increasing queue limits to resolve tail drops can make OOR conditions worse.

---

## Queue Limits and Traffic Shaping

Queue limits are particularly important when traffic shaping is configured.

Shaping intentionally delays packets to restrict their transmission rate.

For example:

```text
R1(config)# policy-map WAN-QOS
R1(config-pmap)# class BULK
R1(config-pmap-c)# shape average 20000000
R1(config-pmap-c)# queue-limit 200 packets
```

This configuration:
- Shapes BULK traffic to an average of 20 Mbps.
- Allows up to 200 packets to accumulate in the queue.
- Drops additional packets when the queue reaches its limit.

### Example: Sustained Congestion

Assume:
- Offered traffic = 30 Mbps.
- Shaping rate = 20 Mbps.
- Queue capacity = 200 packets of 1,500 bytes.

The queue grows at approximately:

```text
30 Mbps - 20 Mbps = 10 Mbps
```

The queue capacity is:

```text
200 × 1500 × 8 = 2,400,000 bits
```

Assuming the queue starts empty and both rates remain constant:

```text
Time to fill = 2,400,000 / 10,000,000
             = 0.24 seconds
             = 240 ms
```

After approximately 240 ms, the queue becomes full.

Additional packets are then dropped unless the offered rate decreases.

**Increasing the queue limit delays the onset of drops, but it does not resolve sustained oversubscription.**

If traffic continuously arrives faster than the configured shaping rate, any finite queue will eventually fill.

---

## Choosing an Appropriate Queue Limit

Queue-limit tuning involves balancing three requirements:

- Absorbing temporary traffic bursts.
- Limiting queuing delay.
- Avoiding excessive consumption of shared buffer memory.

### Larger Queue Limits

Advantages:
- Can absorb larger temporary bursts.
- May reduce packet loss caused by short-term congestion.

Disadvantages:
- Permit longer queuing delays.
- Can contribute to bufferbloat.
- Increase potential shared memory consumption.

### Smaller Queue Limits

Advantages:
- Reduce maximum queue occupancy.
- Can reduce queuing delay.
- Limit the amount of shared memory consumed by a queue.

Disadvantages:
- May cause more drops during microbursts.
- Can reduce performance when legitimate traffic bursts exceed the available buffering.

### Distinguishing Bursts from Sustained Congestion

Suppose a class has an average offered rate below its guaranteed bandwidth but still experiences tail drops.

This may indicate that the traffic arrives in short bursts exceeding the queue's buffering capacity.

Increasing the queue limit may help.

However, if the offered rate continuously exceeds the available service rate, increasing the queue limit will not eliminate drops.

A useful troubleshooting approach is:

1. Verify the offered and transmitted rates.
2. Check the queue limit and drop counters.
3. Determine whether drops occur during bursts or sustained congestion.
4. Temporarily increase the queue limit if burst loss is suspected.
5. Evaluate the resulting packet loss and queuing delay.
6. Reduce the limit again if the additional buffering is unnecessary.

Cisco suggests experimenting with an increase of approximately twice the existing queue limit when investigating suspected burst-related tail drops, then reducing it to the smallest value that accommodates the required bursts.

The objective is not to maximize buffering.

It is to provide sufficient buffering without introducing excessive latency.

---

## Queue Limits and WRED

**Weighted Random Early Detection (WRED)** is a congestion avoidance mechanism that can begin dropping packets before a queue reaches its maximum limit.

The important distinction is:

| Mechanism | Queue-depth measurement | Drop behavior |
|---|---|---|
| Tail drop | Instantaneous | Drop arriving packets when the queue reaches its limit |
| WRED | Average | Randomly drop selected packets as congestion increases |

For example, suppose:
- Queue limit = 200 packets.
- WRED minimum threshold = 100 packets.
- WRED maximum threshold = 150 packets.

WRED can begin randomly dropping packets when the average queue depth exceeds 100 packets.

Once its average queue depth reaches the maximum WRED threshold, eligible packets are subject to the configured maximum-threshold dropping behavior.

Independently, packets are tail dropped if the instantaneous queue depth reaches 200 packets.

### WRED Threshold Units

When configuring WRED on supported QFP-based routers, its thresholds must use a compatible unit mode with the queue limit.

For example:

```text
R1(config-pmap-c)# bandwidth percent 30
R1(config-pmap-c)# queue-limit 200 packets
R1(config-pmap-c)# random-detect dscp-based
```

This creates a packet-based queue limit and enables DSCP-based WRED.

For byte-based WRED, configure the queue in byte mode before configuring byte-based thresholds.

For example:

```text
R1(config-pmap-c)# queue-limit 300000 bytes
R1(config-pmap-c)# random-detect dscp-based
```

The queue-limit and WRED threshold configurations must be consistent with the platform's supported units.

See [Weighted Random Early Detection (WRED)](wred.md).

---

## Queue Limits on Catalyst 9000 Switches

Many Catalyst 9000 switches use UADP switching ASICs rather than the QFP architecture.

On these switches, queue limits are closely related to hardware buffer allocation and **Weighted Tail Drop (WTD)**.

WTD allows different traffic markings within an output queue to have different drop thresholds.

For example, a queue may contain packets marked with several DSCP values.

Rather than using one identical drop threshold for all packets, the switch can apply different thresholds based on their markings.

### Weighted Tail Drop (WTD)

WTD differs from WRED.

| Feature | WTD | WRED |
|---|---|---|
| Primary mechanism | Threshold-based tail drop | Random early detection |
| Drop decision | Queue occupancy compared with assigned threshold | Average queue depth and drop probability |
| Traffic differentiation | Different drop thresholds | Different drop probability curves |
| Common implementation | Catalyst switching ASICs | IOS XE routers and supported switching platforms |

On supported Catalyst 9300 switches, egress queues can have multiple programmable WTD thresholds.

Packets are assigned to thresholds based on DSCP, CoS, or other supported classification fields.

### Example: DSCP-Based Queue Limits

On supported Catalyst 9300 platforms:

```text
SW1(config)# policy-map EGRESS-QOS
SW1(config-pmap)# class BUSINESS
SW1(config-pmap-c)# bandwidth remaining ratio 3
SW1(config-pmap-c)# queue-limit dscp 26 percent 80
```

This assigns packets marked DSCP 26 (AF31) a WTD threshold corresponding to 80% of the applicable queue buffer allocation.

The precise relationship between thresholds, reserved buffers, and shared-buffer consumption depends on the switch ASIC.

### Queue Buffer Allocation

Catalyst 9300 switches also support configuring relative buffer allocations using:

```text
SW1(config-pmap-c)# queue-buffers ratio 20
```

This controls the relative buffer allocation associated with the class queue.

It is separate from the queue-limit threshold.

**`queue-buffers` determines buffer allocation, while `queue-limit` controls the threshold at which packets are dropped.**

Catalyst 9300 switches can use a combination of reserved buffers and dynamically shared buffers.

Some platforms support queue-limit percentages above 100% because a queue can borrow additional capacity from the shared buffer pool.

These values should not be interpreted as percentages of total physical switch memory.

For CCIE EI, understand WTD and the relationship between queue limits and buffer allocation, but verify the exact commands against the switch platform.

---

## Verification and Troubleshooting

The primary command for verifying MQC queue limits is:

```text
R1# show policy-map interface GigabitEthernet1
```

On supported IOS XE routers, the output may include:

```text
Queueing
queue limit 200 packets
(queue depth/total drops/no-buffer drops) 25/150/0
(pkts output/bytes output) 10000/15000000
bandwidth 30% (30000 kbps)
```

This is an illustrative output fragment.

### Understanding the Counters

| Field | Meaning |
|---|---|
| `queue limit` | Configured maximum queue depth |
| `queue depth` | Current buffered data |
| `total drops` | Packets dropped in the class |
| `no-buffer drops` | Drops caused by buffer resource exhaustion |
| `pkts output` | Packets transmitted from the queue |
| `bytes output` | Bytes transmitted from the queue |

For the example above:

```text
queue depth    = 25 packets
total drops    = 150 packets
no-buffer drops = 0 packets
```

The class has experienced 150 drops but has no reported no-buffer drops.

If WRED is not configured and no other dropping actions contribute to the class counters, these drops are consistent with tail drop.

A current queue depth of only 25 packets does not mean congestion never occurred.

The queue may have filled earlier and subsequently drained.

### Useful Commands

| Command | Purpose |
|---|---|
| `show policy-map` | Display configured queue limits |
| `show policy-map interface GigabitEthernet1` | Verify queue depths and drops |
| `show policy-map interface GigabitEthernet1 output` | Verify outbound queuing statistics |
| `show interfaces GigabitEthernet1` | Check interface utilization and output drops |
| `show platform hardware qfp active feature qos interface GigabitEthernet1` | Inspect hardware QoS on supported QFP routers |
| `show platform hardware qfp active bqs 0 packet-buffer utilization` | Inspect shared packet-buffer utilization on supported QFP platforms |

For UADP-based Catalyst 9000 switches, additional commands include:

```text
SW1# show platform hardware fed switch active qos queue stats interface GigabitEthernet1/0/1
SW1# show platform hardware fed switch active qos queue config interface GigabitEthernet1/0/1
```

These provide hardware queue statistics and configuration information on supported platforms.

### Troubleshooting Tail Drops

**Unexpected tail drops**
- Check whether the queue limit is too small for normal bursts.
- Compare offered traffic with the queue's service rate.
- Determine whether congestion is temporary or sustained.
- Consider increasing the limit if bursts are the problem.

**High latency without packet loss**
- Check whether queue limits are excessively large.
- Examine whether a parent shaper is limiting the service rate.
- Consider reducing queue limits to constrain buffering delay.

**Frequent no-buffer drops**
- Inspect shared packet-memory utilization.
- Determine whether individual queues have excessively large limits.
- Consider reducing queue limits to prevent memory exhaustion.

**Changing queue-limit units fails**
- Determine whether the policy is currently attached to an interface.
- Remove the service policy before changing between packet and byte modes when required by the platform.

**Queue depth is zero despite recorded drops**
- Queue depth is an instantaneous measurement.
- Bursts may have occurred between command executions.
- Examine cumulative drop counters rather than relying only on current queue depth.

---

## Key Points

- **Queue limits** determine the maximum amount of data that can be buffered in a queue.
- **Tail drop** discards arriving packets when the queue reaches its configured limit.
- Tail drop uses instantaneous queue depth, not average queue depth.
- Tail drop can occur even when shared packet memory is available.
- **OOR drops** occur when shared buffer resources become unavailable or a protective threshold is reached.
- On supported QFP routers, `queue-limit` supports packets, bytes, and milliseconds.
- Packet-based limits control the number of packets but not their total byte count.
- Byte-based limits provide more direct control over buffered data.
- Time-based limits are converted into byte limits based on the calculated queue service rate.
- On the documented ASR 1000 scheduler, ordinary bandwidth queues target approximately 50 ms of buffering.
- The documented ASR 1000 minimum default bandwidth queue limit is 64 packets.
- The documented ASR 1000 default priority queue limit is 512 packets.
- With `fair-queue`, the documented ASR 1000 implementation uses 16 flow queues, each with its own queue limit.
- Larger queues can absorb bursts but increase potential queuing delay.
- Increasing queue limits cannot eliminate drops caused by sustained oversubscription.
- **WRED** uses average queue depth to perform early dropping.
- **WTD** provides different tail-drop thresholds for different traffic markings on supported Catalyst switches.
- `show policy-map interface` is the primary command for verifying queue limits and drops.

---

## References

- [Cisco IOS XE: Queue Limits and WRED](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/quality-of-service/quality-of-service/m_qos-queue_management_and_congestion_avoidance.html)
- [Cisco IOS XE 17.x: Byte-Based Weighted Random Early Detection](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/qos/b-quality-of-service/m_qos-conavd-bb-wred.html)
- [Cisco IOS XE: QoS Scheduling](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/quality-of-service/quality-of-service/m_qos-scheduling.html)
- [Cisco IOS XE: QoS Time-Based Thresholds for WRED and Queue Limit](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/qos_conavd/configuration/xe-3s/qos-conavd-xe-3s-book/qos-conavd-time-wred.html)
- [Cisco Catalyst 9300 Series: Configuring QoS](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9300/software/release/17-12/configuration_guide/qos/b_1712_qos_9300_cg/configuring_qos.html)