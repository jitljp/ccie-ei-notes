# Queuing and Scheduling Fundamentals

**Queuing** and **scheduling** are the two fundamental components of QoS congestion management.

- **Queuing:** Organizes packets into buffers while they wait to be transmitted.
- **Scheduling:** Determines which queued packet is transmitted next and when it is eligible for transmission.

When an interface is not congested, packets can generally be forwarded without significant queuing delay.

When congestion occurs, packets must compete for the interface's available transmission capacity.

The router uses its queuing and scheduling mechanisms to determine how this capacity is distributed among traffic classes.

Cisco IOS XE supports several scheduling mechanisms, including FIFO, WFQ, CBWFQ, and LLQ.

---

## How Queuing Works

A **queue** is a logical structure that holds packets waiting for transmission.

Queuing is necessary when packets arrive faster than the outgoing interface or an associated shaper can transmit them.

For example, consider a router receiving 200 Mbps of traffic destined for a 100-Mbps interface.

```text
             200 Mbps
                |
                v
          +-----------+
          |    R1     |
          |           |
          |  Output   |      100 Mbps
          |  Queue    |---------------->
          +-----------+
```

The router cannot immediately transmit all the arriving packets.

Instead:

1. Packets are received and processed.
2. The router determines their outgoing interface.
3. Packets requiring buffering are placed into an output queue.
4. The scheduler selects packets for transmission.
5. Packets are removed from the queue as they are scheduled.
6. If the queue reaches its limit, subsequent packets may be dropped.

The process of placing a packet into a queue is called **enqueueing**.

Removing a packet from a queue for transmission is called **dequeueing**.

### Queue Depth

**Queue depth** is the amount of data currently stored in a queue.

Depending on the platform and configuration, it may be expressed in:
- Packets.
- Bytes.
- Time, representing how long the queued data would take to transmit at a specified rate.

Queue depth increases when packets arrive faster than they are dequeued.

Queue depth decreases when packets are dequeued faster than new packets arrive.

### Queue Limit

The **queue limit** defines how much data a queue is permitted to hold.

When a queue reaches its limit, additional packets may be discarded using tail drop.

For example:

```text
                  Queue Limit = 4 Packets

                  +----+----+----+----+
New Packet ----X  | P4 | P3 | P2 | P1 |----> Output
             DROP +----+----+----+----+
```

The queue limit is important because buffering creates delay.

A larger queue can absorb larger traffic bursts but may also introduce more latency.

A smaller queue limits queuing delay but can cause additional packet loss during bursts.

See [Queue Limits and Tail Drop](queue-limits.md).

---

## Packet Buffers and Queue Structures

A logical queue does not necessarily store complete packets in a separate block of memory dedicated to that queue.

On platforms such as the Cisco ASR 1000, packet data is stored in shared packet memory.

The queue contains **packet handles**, which reference the actual packets.

A packet handle includes information such as:
- The packet's location in memory.
- Its length for scheduling purposes.
- Information needed by the queuing mechanism.

For example:

```text
             Logical Queue
        +--------+--------+--------+
        |Handle 3|Handle 2|Handle 1|
        +--------+--------+--------+
             |        |        |
             v        v        v
       +---------------------------+
       |       Packet Memory       |
       |                           |
       |  Packet 3   Packet 2      |
       |             Packet 1      |
       +---------------------------+
```

This distinction matters when interpreting queue limits and drops.

A packet can be dropped because its individual queue has reached its configured limit, even if unused packet memory remains elsewhere in the device.

Conversely, a packet may be dropped because shared buffering resources are exhausted, even when its individual queue has not reached its limit.

The underlying memory architecture varies by platform.

---

## Queuing vs. Scheduling

Although queuing and scheduling work together, they perform different functions.

| Component | Function |
|---|---|
| **Queue** | Holds packets awaiting transmission |
| **Classifier** | Determines which traffic class a packet belongs to |
| **Scheduler** | Selects packets from queues and determines transmission eligibility |
| **Queue limit** | Restricts the amount of buffered data |
| **Drop mechanism** | Determines whether an arriving packet should be discarded |

Consider a router with three output queues.

```text
           Incoming Traffic
                  |
                  v
           +-------------+
           | Classifier  |
           +-------------+
                  |
          +-------+-------+
          |       |       |
          v       v       v
       +-----+ +-----+ +-----+
       |Voice| |Video| |Data |
       |Queue| |Queue| |Queue|
       +-----+ +-----+ +-----+
          |       |       |
          +-------+-------+
                  |
                  v
           +-------------+
           |  Scheduler  |
           +-------------+
                  |
                  v
           Egress Interface
```

The classifier determines where packets are queued.

The scheduler decides how packets from those queues share the outgoing interface.

**Creating multiple queues does not automatically provide priority or bandwidth guarantees.**

The scheduler must be configured to provide the desired treatment.

### Queuing Actions in MQC

On Cisco IOS XE platforms using the class-based scheduling model, an output class generally receives a dedicated queue when a queuing action is configured.

Common queuing actions include:

```text
R1(config-pmap-c)# bandwidth percent 30
R1(config-pmap-c)# bandwidth remaining ratio 3
R1(config-pmap-c)# priority percent 10
R1(config-pmap-c)# shape average 20000000
```

A class map alone does not create a dedicated output queue.

Similarly, configuring only a marking action does not necessarily create a separate queue.

For example:

```text
R1(config)# policy-map QOS
R1(config-pmap)# class BUSINESS
R1(config-pmap-c)# set dscp af31
```

This identifies and marks the traffic but does not configure a dedicated scheduling queue.

On the IOS XE class-based scheduler described by Cisco, traffic belonging to a class with only nonqueuing actions uses the default output queue.

To give that class separate scheduling treatment, configure a queuing action such as `bandwidth`.

---

## Software Queues and the Hardware Transmit Ring

In traditional Cisco IOS router architectures, outbound packet buffering involves two types of queues:

- **Software queues (Layer 3 queues):** Implement QoS scheduling mechanisms such as WFQ, CBWFQ, and LLQ.
- **Hardware transmit ring (Tx ring):** Holds packets that have been handed to the interface hardware for transmission.

The traditional forwarding process can be illustrated as follows:

```text
           Incoming Packets
                   |
                   v
          +------------------+
          | Software Queues  |
          |                  |
          | CBWFQ / LLQ      |
          +------------------+
                   |
                   v
          +------------------+
          | Hardware Tx Ring |
          |                  |
          |       FIFO       |
          +------------------+
                   |
                   v
           Physical Interface
```

### Software Queues

Software queues allow the router to apply QoS scheduling decisions.

For example, an LLQ policy can select voice packets before ordinary data packets.

When the hardware transmit ring is full, the interface driver signals the Layer 3 queuing system to stop dequeueing packets into the ring.

Subsequent packets remain in the software queues, where QoS scheduling determines their transmission order.

### Hardware Transmit Ring

The **Tx ring** is a hardware-level transmit queue that holds packets awaiting transmission by the interface.

In the traditional model, it operates using FIFO.

**Once a packet enters the Tx ring, the software QoS scheduler can no longer reorder it.**

For example, if several large data packets are already waiting in the Tx ring, a newly arriving voice packet cannot bypass them, even if LLQ is configured.

This is why transmit ring size can affect latency.

| Tx ring size | Advantage | Disadvantage |
|---|---|---|
| Larger | Absorbs more bursts and helps keep the interface supplied with packets | Can increase latency and jitter |
| Smaller | Reduces the amount of data waiting beyond the QoS scheduler | May reduce throughput if the interface is not supplied with packets quickly enough |

On certain older Cisco platforms, the `tx-ring-limit` command can control the size of the transmit ring.

For example, Cisco supported this command on specific ATM interfaces.

### Modern IOS XE Considerations

The traditional software queue/Tx ring distinction is useful conceptually, but it does not describe the internal architecture of every modern IOS XE platform.

For example, Cisco ASR 1000 routers use the QFP architecture, where packet buffering, queuing, and scheduling are implemented by dedicated hardware.

Therefore:
- A logical QoS queue is not necessarily a software-managed queue.
- Hardware schedulers can implement CBWFQ and LLQ directly.
- Physical interfaces may still have additional transmit buffering.
- The `tx-ring-limit` command is platform-specific and should not be assumed available on modern IOS XE interfaces.

**For CCIE EI, understand the relationship between QoS scheduling and the final transmit buffers, but focus configuration practice on MQC queuing, scheduling, and queue limits rather than legacy Tx ring tuning.**

---

## Scheduling Fundamentals

A **scheduler** determines which queue should be serviced next.

Different scheduling algorithms produce different forwarding behavior.

For example, suppose three queues contain packets waiting for transmission:

```text
       +-------------------+
       | Voice Queue       |
       +-------------------+

       +-------------------+
       | Business Queue    |
       +-------------------+

       +-------------------+
       | Bulk Data Queue   |
       +-------------------+
                  |
                  v
              Scheduler
                  |
                  v
          Egress Interface
```

The scheduler might use:

- **FIFO:** Transmit packets in their arrival order.
- **Weighted scheduling:** Distribute transmission opportunities among queues using configured weights.
- **Strict priority:** Always select a higher-priority queue before lower-priority queues when eligible packets are waiting.
- **Rate-based scheduling:** Provide minimum service rates or enforce maximum transmission rates.

The scheduling algorithm directly affects bandwidth allocation, latency, and jitter.

### Work-Conserving Scheduling

A **work-conserving scheduler** does not intentionally leave transmission capacity unused when eligible packets are waiting.

For example, consider two queues sharing a 100-Mbps interface.

Queue A has a minimum bandwidth guarantee of 60 Mbps.

Queue B has a minimum bandwidth guarantee of 40 Mbps.

If Queue A has no traffic, Queue B can potentially consume the entire 100 Mbps.

```text
       Queue A = 0 Mbps
       Queue B = 100 Mbps
                |
                v
       Total = 100 Mbps
```

The bandwidth guarantees are not maximum limits.

**Minimum bandwidth guarantees reserve service opportunities during congestion, not exclusive portions of the link that must remain unused when a class is idle.**

### Non-Work-Conserving Scheduling

A **non-work-conserving scheduler** may deliberately delay packets even when the physical interface is available.

Traffic shaping is an example.

Suppose a router has a 1-Gbps physical interface but shapes traffic to 100 Mbps.

Even if the interface could transmit more data, the shaper delays packets to maintain the configured rate.

Therefore:
- Ordinary weighted scheduling is generally work-conserving.
- Shaping introduces rate restrictions that can make transmission intentionally non-work-conserving.

See [Traffic Shaping](../policing-shaping/shaping.md).

---

## Minimum, Maximum, and Excess Bandwidth

Modern IOS XE platforms, including Cisco ASR 1000 routers, use a scheduling model with three important parameters for bandwidth queues:

- **Min:** Minimum guaranteed service rate.
- **Max:** Maximum permitted service rate.
- **Ex:** Relative weight used to distribute excess bandwidth.

These parameters are particularly useful for understanding how different MQC scheduling commands interact.

| Parameter | Meaning | MQC command |
|---|---|---|
| **Min** | Minimum bandwidth guarantee | `bandwidth` |
| **Max** | Maximum transmission rate | `shape` |
| **Ex** | Relative weight for excess bandwidth | `bandwidth remaining` |

### Minimum Bandwidth (Min)

The **minimum bandwidth** determines the transmission capacity guaranteed to a queue under congestion, provided sufficient matching traffic is offered.

For example:

```text
R1(config-pmap-c)# bandwidth percent 30
```

On a 100-Mbps interface, this configures a 30-Mbps minimum bandwidth allocation.

When congestion occurs and the class has at least 30 Mbps of traffic, the scheduler attempts to ensure that the class receives its minimum allocation.

However, the class can receive more than 30 Mbps if additional bandwidth is available.

The `bandwidth` command therefore configures a minimum service rate, not a rate limit.

### Maximum Bandwidth (Max)

The **maximum bandwidth** determines the highest rate at which a class is permitted to transmit over time.

For example:

```text
R1(config-pmap-c)# shape average 20000000
```

This configures a 20-Mbps average shaping rate.

Even if 100 Mbps of physical capacity is available, this class remains constrained by the configured shaping rate, subject to permitted bursts.

Unlike `bandwidth`, the `shape` command does not itself guarantee minimum bandwidth.

It only limits the maximum rate.

### Excess Weight (Ex)

The **excess weight** controls how available bandwidth is distributed after priority traffic and minimum bandwidth guarantees have been serviced.

For example:

```text
R1(config-pmap-c)# bandwidth remaining ratio 3
```

This assigns the class a relative excess-bandwidth weight of 3.

If another class has a weight of 1, the two classes compete for excess bandwidth using a 3:1 ratio.

Assuming both have sufficient traffic and no other constraints, they receive:

```text
Class A = 3 / (3 + 1) = 75%

Class B = 1 / (3 + 1) = 25%
```

The configured weights are relative values.

They are not absolute bandwidth percentages.

On the Cisco IOS XE three-parameter scheduler, a bandwidth queue without an explicitly configured excess weight normally has a default weight of 1.

---

## How the Scheduler Allocates Bandwidth

The scheduler considers priority, minimum bandwidth guarantees, and excess weights when choosing packets.

For the IOS XE scheduling model described above, the simplified service order is:

1. Service eligible strict priority traffic.
2. Service bandwidth queues that have not yet received their minimum guarantees.
3. Distribute remaining capacity among eligible bandwidth queues according to their excess weights.
4. Enforce any configured maximum rates.

These are ongoing scheduling decisions, not separate, permanently reserved time periods.

A queue receives bandwidth only when it has traffic waiting to be transmitted.

### Example: Minimum Guarantees and Excess Bandwidth

Suppose a router has a 100-Mbps interface and the following policy:

| Class | Configuration | Offered traffic |
|---|---|---:|
| Voice | Strict priority, 10 Mbps used | 10 Mbps |
| Business | Minimum 30 Mbps, Ex = 1 | 90 Mbps |
| Bulk | Ex = 3 | 90 Mbps |
| Default | Ex = 1 | 90 Mbps |

Assume the priority traffic is admitted without drops, and all other queues remain backlogged.

**Step 1: Service the priority traffic.**

```text
100 - 10 = 90 Mbps remaining
```

**Step 2: Meet the Business minimum guarantee.**

```text
90 - 30 = 60 Mbps remaining
```

**Step 3: Distribute the excess bandwidth.**

The excess weights are:

```text
Business = 1
Bulk     = 3
Default  = 1

Total    = 5
```

Therefore:

```text
Business excess = 60 × 1/5 = 12 Mbps

Bulk excess     = 60 × 3/5 = 36 Mbps

Default excess  = 60 × 1/5 = 12 Mbps
```

The final transmission rates are:

| Class | Guaranteed/priority rate | Excess bandwidth | Total |
|---|---:|---:|---:|
| Voice | 10 Mbps | 0 Mbps | 10 Mbps |
| Business | 30 Mbps | 12 Mbps | 42 Mbps |
| Bulk | 0 Mbps | 36 Mbps | 36 Mbps |
| Default | 0 Mbps | 12 Mbps | 12 Mbps |
| **Total** | **40 Mbps** | **60 Mbps** | **100 Mbps** |

This illustrates two important principles:

- Minimum bandwidth guarantees are satisfied before excess bandwidth is distributed.
- A queue with a minimum guarantee can also receive excess bandwidth.

If any class becomes idle or reaches a maximum shaping rate, its unused allocation can potentially be redistributed to other eligible queues.

---

## Strict Priority vs. Weighted Scheduling

Two important scheduling approaches are **strict priority scheduling** and **weighted scheduling**.

### Strict Priority Scheduling

With strict priority scheduling, the scheduler selects packets from the highest-priority nonempty eligible queue.

For example:

```text
        +----------------+
        | Priority Queue |----+
        +----------------+    |
                              v
        +----------------+ Scheduler
        | Normal Queue   |    |
        +----------------+    |
                              v
                         Interface
```

As long as eligible priority packets are waiting, the scheduler services them before the normal queue.

This provides low delay for priority traffic.

However, unrestricted priority traffic can prevent lower-priority queues from receiving service.

This is called **starvation**.

Cisco LLQ commonly addresses this problem using a priority bandwidth rate enforced by a conditional policer during congestion.

For example:

```text
R1(config-pmap-c)# priority percent 10
```

The configured rate helps prevent excessive priority traffic from consuming all available bandwidth under congestion.

### Weighted Scheduling

Weighted scheduling distributes transmission capacity among multiple queues according to their configured weights.

For example, suppose two continuously backlogged queues have weights of 3 and 1.

On a 100-Mbps interface, their approximate shares are:

```text
Queue A = 75 Mbps

Queue B = 25 Mbps
```

Weighted scheduling provides controlled sharing without allowing one backlogged queue to consume all available transmission opportunities indefinitely.

The exact packet service sequence depends on the scheduling algorithm.

**Weighted scheduling does not mean packets are transmitted in a fixed, repeating sequence of three packets from Queue A followed by one packet from Queue B.**

Because packets have different sizes, schedulers must account for the amount of data transmitted, not just the number of packets.

See [FIFO, WFQ, and CBWFQ](fifo-wfq-cbwfq.md) and [Low Latency Queuing (LLQ)](llq.md).

---

## Packet Size and Scheduling

Packet size affects scheduling because transmitting a larger packet consumes more link capacity than transmitting a smaller packet.

Consider two queues:

- Queue A contains 1,500-byte packets.
- Queue B contains 100-byte packets.

If a scheduler transmitted one packet from each queue in every round, Queue A would consume much more bandwidth.

```text
Queue A: 1500 bytes per round

Queue B:  100 bytes per round
```

The resulting bandwidth ratio would be approximately 15:1 rather than 1:1.

Fair scheduling algorithms account for packet sizes to provide the intended bandwidth distribution.

### Packet Serialization Delay

**Serialization delay** is the time required to transmit a packet onto a physical link.

The basic formula is:

```text
Serialization delay = Packet size (bits) / Link speed (bps)
```

For example, consider a 1,500-byte packet.

```text
Packet size = 1,500 × 8
            = 12,000 bits
```

| Link speed | Serialization delay |
|---|---:|
| 1 Mbps | 12 ms |
| 10 Mbps | 1.2 ms |
| 100 Mbps | 120 µs |
| 1 Gbps | 12 µs |

These simplified calculations exclude additional Layer 1 and Layer 2 overhead.

### Why Serialization Delay Matters for Priority Queues

Suppose a large data packet has already begun transmission.

A high-priority voice packet arrives immediately afterward.

The voice packet cannot normally interrupt the transmission of the data packet already being serialized.

Instead, it must wait for that transmission to finish.

This is a form of **non-preemptive scheduling**.

Consequently, strict priority queuing cannot eliminate all delay.

On low-speed links, serialization delay can become a significant source of latency and jitter.

Historically, mechanisms such as Multilink PPP fragmentation and interleaving were used to reduce this problem on slow WAN links.

---

## Queuing Delay and Buffer Occupancy

**Queuing delay** is the time a packet spends waiting before it can be transmitted.

The delay depends on:
- The amount of data already queued ahead of the packet.
- The rate at which the scheduler services that queue.
- The scheduling treatment of competing queues.

A simplified approximation is:

```text
Queuing delay = Queued bits / Queue service rate
```

For example, suppose a queue contains 125,000 bytes of data and is being serviced at 10 Mbps.

```text
Queued data = 125,000 × 8
            = 1,000,000 bits

Queuing delay = 1,000,000 / 10,000,000
              = 0.1 seconds
              = 100 ms
```

This means the queue contains approximately 100 ms of transmission work at that service rate.

The calculation assumes a constant service rate and ignores any additional scheduling interruptions.

### Queue Depth and Delay

For a fixed service rate:

- Increasing queue depth generally increases queuing delay.
- Decreasing queue depth generally decreases queuing delay.
- Empty queues introduce little or no queuing delay.

However, a larger buffer may absorb traffic bursts that would otherwise cause packet loss.

This creates a tradeoff between packet loss and latency.

---

## Queue Growth During Congestion

The rate at which a queue grows depends on the difference between its arrival and service rates.

When both rates are constant:

```text
Queue growth rate = Arrival rate - Service rate
```

Suppose:

- Arrival rate = 60 Mbps.
- Service rate = 40 Mbps.

The queue grows at:

```text
60 - 40 = 20 Mbps
```

If the queue has 1 MB of available buffering, the approximate time required to fill it is:

```text
Queue capacity = 1,000,000 × 8
               = 8,000,000 bits

Fill time = 8,000,000 / 20,000,000
          = 0.4 seconds
```

The buffer fills in approximately 400 ms, assuming it starts empty and the traffic rates remain constant.

Once the queue reaches its limit, additional arriving packets may be dropped.

### Temporary vs. Sustained Congestion

**Temporary congestion** occurs when the arrival rate exceeds the service rate for a short period.

The queue may absorb the burst and drain afterward.

**Sustained congestion** occurs when the arrival rate continuously exceeds the service rate.

The queue eventually fills and packet loss occurs unless traffic sources reduce their rates or additional capacity becomes available.

A queue cannot absorb an indefinitely sustained excess traffic rate without dropping packets.

---

## Head-of-Line Blocking

**Head-of-line (HOL) blocking** occurs when a packet at the front of a queue prevents packets behind it from being serviced, even if those later packets would otherwise benefit from different treatment.

Consider a single FIFO output queue:

```text
       +----------+----------+----------+
       |  Voice   |   Data   |   Data   |----> Output
       +----------+----------+----------+
          Newer                 Oldest
```

The voice packet must wait for the older data packets to be transmitted.

Separate class-based queues can reduce this problem.

For example, placing voice traffic into a priority queue allows the scheduler to select voice packets before packets waiting in a normal data queue.

However, HOL blocking can still occur within an individual queue, and an already-transmitting packet generally cannot be preempted.

---

## Default Queues and Class-Based Queues

On Cisco IOS XE, the behavior of the default output queue depends on the configured QoS policy and hardware platform.

When an output queuing policy is applied, packets are classified into appropriate queues.

The `class-default` class handles traffic that does not match any earlier named class.

For example:

```text
R1(config)# class-map match-any BUSINESS
R1(config-cmap)# match dscp af31
R1(config)# policy-map WAN-QOS
R1(config-pmap)# class BUSINESS
R1(config-pmap-c)# bandwidth percent 30
R1(config-pmap)# class class-default
```

On the IOS XE class-based scheduler:

- Packets matching BUSINESS use its dedicated bandwidth queue.
- Unmatched traffic uses the default queue.
- The scheduler distributes bandwidth between these queues according to its configured parameters and defaults.

The default queue is not automatically a strict priority queue.

It is generally serviced using the platform's normal bandwidth scheduling behavior.

### Default Excess Weight

In the IOS XE three-parameter scheduler, the default excess weight for a bandwidth queue is 1.

Consider:

```text
R1(config)# policy-map WAN-QOS
R1(config-pmap)# class BUSINESS
R1(config-pmap-c)# bandwidth remaining ratio 3
R1(config-pmap)# class class-default
```

The effective excess weights are:

```text
BUSINESS      = 3
class-default = 1
```

If both queues remain backlogged and no other constraints apply, their approximate bandwidth shares are:

```text
BUSINESS      = 75%
class-default = 25%
```

There is no requirement to explicitly configure `bandwidth remaining ratio 1` under `class-default` to obtain the default weight on this scheduling implementation.

---

## Percentage-Based Bandwidth Allocation

Cisco IOS XE supports different percentage-based scheduling commands.

Two important commands are:

```text
R1(config-pmap-c)# bandwidth percent 30
R1(config-pmap-c)# bandwidth remaining percent 30
```

Although they look similar, they do not perform the same function.

| Command | Meaning |
|---|---|
| `bandwidth percent 30` | Configures a minimum bandwidth guarantee equal to 30% of the applicable scheduling bandwidth |
| `bandwidth remaining percent 30` | Configures a 30% share of remaining bandwidth among eligible bandwidth queues, using the percentage-based excess-weight model |

For a flat policy on supported IOS XE platforms, `bandwidth percent` uses the interface's nominal bandwidth as the percentage reference.

In hierarchical policies, bandwidth calculations may instead be based on the parent scheduler's available rate.

### Example: Percentage-Based Remaining Bandwidth

Consider a policy with:

```text
R1(config)# policy-map WAN-QOS
R1(config-pmap)# class BUSINESS
R1(config-pmap-c)# bandwidth remaining percent 75
R1(config-pmap)# class class-default
```

On the IOS XE scheduler documented by Cisco, the remaining percentage is assigned to the unconfigured default bandwidth queue.

Therefore:

```text
BUSINESS      = 75%
class-default = 25%
```

These percentages describe relative bandwidth sharing when the queues are competing for transmission capacity.

They do not establish maximum transmission rates.

If BUSINESS has no traffic, `class-default` can consume the available capacity.

---

## Egress Queuing and Congestion Points

Class-based queuing is most commonly applied in the outbound direction.

This is because the router can control the order in which packets are transmitted from an interface.

For example:

```text
             R1
       +-------------+
       |             |
LAN -->|             |---> WAN
       |             |
       | Egress QoS  |
       +-------------+
```

The router classifies packets, places them into appropriate output queues, and schedules their transmission.

Ingress buffering and QoS mechanisms also exist on various platforms, particularly switches, but their operation and configuration can differ from the MQC egress scheduling model.

### Physical vs. Logical Congestion

Congestion does not have to be caused by the physical interface reaching its line rate.

A shaper can create a logical congestion point.

For example:

```text
Physical interface: 1 Gbps
Configured shaper:  100 Mbps
Offered traffic:    200 Mbps
```

Even though the physical interface has additional capacity, the shaper restricts the traffic to an average rate of 100 Mbps.

Packets therefore accumulate in the shaping queue.

This allows a child queuing policy to prioritize traffic within the shaped rate.

Such designs are implemented using **Hierarchical QoS (HQoS)**.

See [4.4.g Hierarchical QoS (HQoS)](../../hqos/overview.md).

---

## IOS XE Hardware Scheduling Considerations

The general queuing concepts are consistent across Cisco platforms, but their implementations are not necessarily identical.

Modern hardware-forwarding routers may use dedicated packet processors and hardware schedulers.

For example, Cisco ASR 1000 routers use the **Quantum Flow Processor (QFP)** architecture for packet processing and QoS scheduling.

On these platforms:
- Packets can be stored in shared packet memory.
- Logical queues contain packet handles referencing those packets.
- Hardware scheduling logic determines packet service.
- Physical interfaces have their own buffering and transmission mechanisms.

Older Cisco IOS explanations often distinguish between a software output queue and a hardware transmit ring.

That model remains useful conceptually, but it should not be assumed to describe the exact internal architecture of all modern IOS XE platforms.

In particular, a physical interface may have additional buffering after a packet has been selected by a QoS scheduler.

The exact queue count, hardware buffer architecture, and supported scheduling behaviors depend on the platform.

### Scheduling Length and Overhead Accounting

A scheduler must account for packet size when distributing bandwidth.

On supported IOS XE platforms, the scheduling length includes the packet and relevant encapsulation overhead.

However, the accounted length may differ from the complete number of bits transmitted on the physical medium.

For example, Ethernet framing and physical-layer overhead can affect the actual on-the-wire rate.

Cisco IOS XE supports overhead accounting adjustments on certain scheduling commands, allowing the administrator to modify the length used in rate calculations.

This is particularly useful when shaping traffic to match a service provider's bandwidth accounting method.

---

## Verification and Troubleshooting

The primary command for verifying output queuing is:

```text
R1# show policy-map interface GigabitEthernet1
```

This displays the configured QoS policy and available traffic statistics.

For classes with queuing actions, the output may include:

```text
Queueing
queue limit 200 packets
(queue depth/total drops/no-buffer drops) 0/0/0
(pkts output/bytes output) 10000/15000000
bandwidth 30% (30000 kbps)
```

This is an illustrative output fragment; actual formatting and counters depend on the platform.

The relevant values are:

| Field | Meaning |
|---|---|
| `queue limit` | Maximum permitted queue depth |
| `queue depth` | Current amount of data buffered |
| `total drops` | Packets dropped by the queue |
| `no-buffer drops` | Drops caused by unavailable buffering resources |
| `pkts output` | Packets transmitted from the queue |
| `bytes output` | Bytes transmitted from the queue |
| `bandwidth` | Configured minimum bandwidth allocation |

### Hardware Queue Verification

On supported QFP-based IOS XE routers, the following command provides additional information about hardware QoS scheduling:

```text
R1# show platform hardware qfp active feature qos interface GigabitEthernet1
```

This can expose details not always visible through `show policy-map interface`, including:
- Hardware queue identifiers.
- Programmed scheduling rates.
- Queue limits.
- Queue depths.
- Tail-drop counters.
- Other hardware-specific statistics.

The precise command syntax and availability vary by platform.

### Interpreting Queue Depth

An important detail is that a queue depth of zero does not necessarily mean the interface has never experienced congestion.

Queues may fill and drain rapidly.

Furthermore, on some IOS XE platforms, `show policy-map interface` statistics are updated periodically rather than continuously.

A short microburst may therefore be missed when examining the current queue depth.

Increasing drop counters, even when queue depth is currently zero, can indicate intermittent congestion.

### Useful Commands

| Command | Purpose |
|---|---|
| `show policy-map` | Display configured QoS policy maps |
| `show policy-map interface GigabitEthernet1` | Display per-class queuing and drop statistics |
| `show policy-map interface GigabitEthernet1 output` | Examine outbound policy statistics |
| `show interfaces GigabitEthernet1` | Check interface utilization and output drops |
| `show running-config interface GigabitEthernet1` | Verify service-policy attachment |
| `show platform hardware qfp active feature qos interface GigabitEthernet1` | Inspect hardware QoS statistics on supported QFP platforms |

### Common Troubleshooting Scenarios

**Queue depth remains at zero**
- Traffic may be below the available service rate.
- Bursts may be too short to appear in periodically updated counters.
- The traffic may not be matching the intended class.
- The actual congestion point may be elsewhere in the network.

**A class receives more bandwidth than configured**
- `bandwidth` provides a minimum, not a maximum.
- Unused bandwidth from other classes may be available.
- Use shaping when a maximum rate is required.

**A class receives less bandwidth than its guarantee**
- Check whether sufficient matching traffic is being offered.
- Verify the applicable interface bandwidth or parent shaping rate.
- Verify that traffic is classified into the correct queue.
- Check for upstream policing, downstream bottlenecks, or platform-specific constraints.

**Packets are dropped despite available physical bandwidth**
- A configured shaper may be restricting the effective service rate.
- A class queue may have reached its queue limit.
- Shared packet buffers may be exhausted.
- Other rate-limiting mechanisms may be active.

---

## Key Points

- **Queuing** buffers packets awaiting transmission.
- **Scheduling** determines which queued packets are transmitted and when.
- **Enqueueing** adds packets to a queue; **dequeueing** removes them for transmission.
- **Queue depth** is the amount of data currently buffered.
- **Queue limits** constrain buffering and therefore help control queuing delay.
- A class map or marking action alone does not necessarily create a dedicated output queue.
- **Strict priority scheduling** always prefers eligible higher-priority traffic and can starve lower-priority queues if left unrestricted.
- **Weighted scheduling** shares available capacity among eligible queues according to scheduling weights.
- **Work-conserving scheduling** uses available capacity when eligible packets are waiting.
- **Shaping** may deliberately delay packets even when the physical interface has spare capacity.
- On the IOS XE three-parameter scheduler, **Min** is configured with `bandwidth`, **Max** with `shape`, and **Ex** with `bandwidth remaining`.
- `bandwidth` defines a minimum service rate, not a maximum.
- `bandwidth remaining ratio` defines a relative excess-bandwidth weight, not an absolute percentage.
- A bandwidth queue's default excess weight is 1 on the IOS XE scheduling model described here.
- Packet size matters when calculating fair bandwidth allocation and serialization delay.
- `show policy-map interface` is the primary IOS XE command for verifying output queuing behavior.

---

## References

- [Cisco IOS XE: QoS Scheduling](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/quality-of-service/quality-of-service/m_qos-scheduling.html)
- [Cisco IOS XE: Queue Limits and WRED](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/quality-of-service/quality-of-service/m_qos-queue_management_and_congestion_avoidance.html)
- [Cisco IOS XE 17.x: QoS Hierarchical Scheduling](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/qos/b-quality-of-service/m_qos-hierarchical_scheduling.html)
- [Cisco IOS XE 17.x: Distribution of Remaining Bandwidth Using Ratio](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/qos/b-quality-of-service/m_qos-plcshp-dist-rem-bw.html)