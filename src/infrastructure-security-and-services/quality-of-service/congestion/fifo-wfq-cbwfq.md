# FIFO, WFQ, and CBWFQ

**First In, First Out (FIFO)**, **Weighted Fair Queuing (WFQ)**, and **Class-Based Weighted Fair Queuing (CBWFQ)** are congestion management mechanisms that determine how packets are scheduled for transmission.

They differ primarily in how they organize packets into queues and allocate transmission opportunities.

| Mechanism | Queuing method | Scheduling behavior |
|---|---|---|
| **FIFO** | Single queue | Transmit packets in arrival order |
| **WFQ** | Separate queues for individual flows | Weighted fair scheduling among flows |
| **CBWFQ** | Separate queues for administrator-defined classes | Bandwidth allocation and weighted scheduling among classes |

FIFO provides no traffic differentiation.

WFQ automatically distinguishes individual traffic flows and provides fairness among them.

CBWFQ allows administrators to explicitly define traffic classes and configure their bandwidth allocations using the Modular QoS CLI (MQC).

---

## First In, First Out (FIFO)

**FIFO** is the simplest queuing mechanism.

Packets are placed into a single queue and transmitted in the order they arrive.

```text
             FIFO Queue
        +----+----+----+----+
        | P4 | P3 | P2 | P1 |----> Output
        +----+----+----+----+
                            ^
                       First packet
```

FIFO does not distinguish between applications, protocols, or DSCP markings.

For example, suppose packets arrive in this order:

1. Data packet 1.
2. Data packet 2.
3. Voice packet 1.
4. Data packet 3.

FIFO transmits them in exactly that order.

The voice packet must wait for both preceding data packets to be transmitted.

### FIFO Characteristics

- All packets share a single queue.
- Packets are transmitted in arrival order.
- No classification or special scheduling configuration is required.
- All traffic receives the same scheduling treatment.
- Packets are normally discarded using tail drop when the queue reaches its limit.

FIFO is computationally simple and appropriate when differentiated scheduling is unnecessary.

However, it provides no mechanism to protect delay-sensitive traffic from competing applications.

### FIFO and Head-of-Line Blocking

A major limitation of FIFO is **head-of-line blocking**.

For example:

```text
         FIFO Queue
     +-------+-------+-------+
     | Voice | Data2 | Data1 |----> Output
     +-------+-------+-------+
```

The voice packet must wait behind the two data packets.

If a large number of data packets are already queued, voice traffic may experience substantial delay and jitter.

Separate QoS queues can prevent voice packets from having to wait behind data packets still buffered in other queues.

However, packets already being serialized by an interface generally cannot be preempted.

### FIFO in Cisco IOS XE

On many Cisco IOS XE platforms, an interface without an explicitly configured queuing policy uses a default FIFO scheduling behavior.

A CBWFQ policy can also contain individual class queues that operate as FIFO queues internally.

**FIFO is not necessarily incompatible with CBWFQ.**

For example, CBWFQ may create three class queues, with each class queue using FIFO internally.

The CBWFQ scheduler determines which class receives service, while FIFO determines packet order within each class.

---

## Weighted Fair Queuing (WFQ)

**Weighted Fair Queuing (WFQ)** is a congestion management mechanism that automatically separates traffic into individual flows and schedules those flows according to assigned weights.

Unlike FIFO, WFQ does not place every packet into a single queue.

Instead, it identifies individual traffic flows, traditionally called **conversations**, and places them into separate logical queues.

For example:

```text
           Incoming Traffic
                  |
                  v
          Flow Classification
                  |
        +---------+---------+
        |         |         |
        v         v         v
     +------+  +------+  +------+
     |Flow A|  |Flow B|  |Flow C|
     |Queue |  |Queue |  |Queue |
     +------+  +------+  +------+
        |         |         |
        +---------+---------+
                  |
                  v
            WFQ Scheduler
                  |
                  v
           Egress Interface
```

The scheduler shares transmission capacity among the individual flows.

This prevents a single high-bandwidth flow from monopolizing the output queue.

### WFQ Flow Identification

Traditional Cisco WFQ identifies flows using packet header fields.

For IPv4 TCP and UDP traffic, the identifying fields typically include:

- Source IP address.
- Destination IP address.
- IP protocol.
- Source transport-layer port.
- Destination transport-layer port.

These fields form the familiar **5-tuple**.

For example:

| Flow | Source | Destination | Protocol | Source port | Destination port |
|---|---|---|---|---|---|
| A | 10.1.1.10 | 10.2.2.10 | TCP | 50000 | 443 |
| B | 10.1.1.11 | 10.2.2.10 | TCP | 50001 | 443 |
| C | 10.1.1.12 | 10.2.2.20 | UDP | 20000 | 20000 |

These are three different flows.

Traditional WFQ can place them into different logical queues.

The precise classification fields depend on the protocol and WFQ implementation.

### Flow Fairness

Consider a 10-Mbps interface with three continuously backlogged flows.

Assume the flows have equal scheduling weights.

| Flow | Offered traffic |
|---|---:|
| A | 20 Mbps |
| B | 20 Mbps |
| C | 20 Mbps |

With equal-weight fair scheduling, each flow receives approximately:

```text
Flow A = 10 / 3 = 3.33 Mbps
Flow B = 10 / 3 = 3.33 Mbps
Flow C = 10 / 3 = 3.33 Mbps
```

Now suppose Flow C offers only 1 Mbps.

The scheduler does not reserve unused bandwidth for Flow C.

Instead, Flow C receives approximately 1 Mbps, and the remaining 9 Mbps is shared between the other two backlogged flows:

```text
Flow A = 4.5 Mbps
Flow B = 4.5 Mbps
Flow C = 1 Mbps
```

This illustrates the work-conserving nature of WFQ.

A flow can consume less than its fair share without preventing other flows from using the available bandwidth.

---

## WFQ and Low-Bandwidth Flows

One of the primary benefits of WFQ is its treatment of low-bandwidth interactive traffic.

Consider two flows sharing a congested interface:

- An FTP transfer continuously sends large packets.
- An interactive application occasionally sends small packets.

With FIFO, the interactive packets may become trapped behind a large backlog of FTP packets.

With WFQ, the two flows are separated into different queues.

```text
       FTP Flow              Interactive Flow
    +----+----+----+           +----+
    | P3 | P2 | P1 |           | P1 |
    +----+----+----+           +----+
           |                      |
           +----------+-----------+
                      |
                      v
                 WFQ Scheduler
                      |
                      v
                    Output
```

The interactive flow does not need to wait for the entire FTP backlog to drain.

Instead, the scheduler provides transmission opportunities to both flows according to their weights.

Because the interactive flow sends relatively little traffic, it often has few packets waiting in its queue.

Consequently, its packets can experience substantially lower delay than they would under FIFO.

**WFQ does not inherently assign strict priority to small packets or low-bandwidth applications.**

The improvement comes from separating flows and providing weighted fairness rather than allowing one flow to occupy a shared FIFO queue.

---

## WFQ Scheduling Weights

Traditional Cisco flow-based WFQ considers **IP Precedence** when assigning scheduling weights.

Flows with higher IP Precedence values receive more favorable scheduling weights than flows with lower values.

For example:

| IP Precedence | Relative treatment |
|---|---|
| 0 | Lowest normal precedence |
| 1 | Higher than 0 |
| 3 | Higher than 1 |
| 5 | Higher than 3 |
| 7 | Highest precedence value |

In the classic Cisco WFQ model, the relative bandwidth weight increases with:

```text
IP Precedence + 1
```

For two continuously backlogged flows with identical packet characteristics and IP Precedence values of 0 and 3:

```text
Flow A (precedence 0) = weight 1
Flow B (precedence 3) = weight 4
```

Their relative bandwidth shares are approximately:

```text
Flow A = 1 / (1 + 4) = 20%
Flow B = 4 / (1 + 4) = 80%
```

This illustrates the precedence-based weighting in classic WFQ, not a universal scheduling formula for all modern IOS XE implementations.

### IP Precedence vs. DSCP

Traditional WFQ predates the modern DiffServ architecture.

It primarily uses the three IP Precedence bits rather than assigning independent scheduling weights to all 64 DSCP values.

For example:

- CS5 has DSCP 40 (`101000`).
- EF has DSCP 46 (`101110`).

Both have IP Precedence 5 because their first three DSCP bits are `101`.

Therefore, traditional precedence-based WFQ does not inherently distinguish CS5 from EF based on their complete DSCP values.

For explicit DSCP-based scheduling differentiation, CBWFQ is more appropriate.

---

## WFQ Sequence Numbers

Traditional WFQ uses a virtual scheduling algorithm to determine which packets should be transmitted.

Conceptually, packets receive **virtual finish times** that reflect:

- Their size.
- Their flow's scheduling weight.
- Their position relative to preceding packets in the same flow.

The scheduler uses these finish times to determine packet service order.

For equal-weight flows, smaller packets require less transmission service and therefore contribute smaller increments to virtual finish time.

Higher-weight flows receive more favorable scheduling treatment.

This allows WFQ to approximate fair sharing of bandwidth while accounting for packet sizes.

### Why Packet Size Matters

Consider two flows:

- Flow A sends 1,500-byte packets.
- Flow B sends 100-byte packets.

A scheduler that simply alternated one packet from each queue would transmit 15 times as many bytes for Flow A as for Flow B.

That would not provide equal bandwidth.

WFQ accounts for packet lengths when scheduling flows.

As a result, a flow sending larger packets does not automatically receive more bandwidth merely because its packets contain more data.

---

## WFQ Flow Queues and Hashing

Traditional WFQ implementations use a configurable number of dynamic conversation queues.

A hashing mechanism maps flows to these queues.

For example:

```text
Flow A ----+
           +----> Hash ----> Queue 1
Flow B ----+                Queue 2
                            Queue 3
Flow C ----+                Queue 4
           +----> Hash
Flow D ----+
```

Different flows can occasionally map to the same queue.

This is called a **hash collision**.

When flows collide, they share a flow queue rather than receiving fully independent scheduling treatment.

Increasing the number of available flow queues can reduce collisions, although the supported queue count and hardware resource requirements vary by platform.

On classic Cisco IOS, a common default was 256 dynamic conversation queues.

This should not be confused with the 16 flow queues created by the `fair-queue` feature on certain modern IOS XE QFP-based platforms.

---

## Traditional WFQ Configuration

On older Cisco IOS platforms that support interface-level WFQ, it can be enabled using:

```text
R1(config)# interface Serial0/0
R1(config-if)# fair-queue
```

Historically, Cisco IOS enabled flow-based WFQ by default on certain serial interfaces with bandwidths of 2.048 Mbps or lower.

Other interface types commonly used FIFO by default.

This historical behavior should not be assumed for modern IOS XE routers.

### WFQ Configuration Parameters

The traditional interface-level command supports the following syntax:

```text
R1(config-if)# fair-queue [CDT [dynamic-queues [reservable-queues]]]
```

| Parameter | Description |
|---|---|
| `CDT` | Congestive Discard Threshold |
| `dynamic-queues` | Number of dynamic conversation queues |
| `reservable-queues` | Number of queues reserved for features such as RSVP |

For example:

```text
R1(config-if)# fair-queue 64 256 0
```

This configures:

- Congestive Discard Threshold: 64 packets.
- Dynamic conversation queues: 256.
- Reservable queues: 0.

The **Congestive Discard Threshold (CDT)** controls when traditional WFQ begins dropping packets from congested conversations.

In classic WFQ, a high-bandwidth conversation that reaches its threshold can experience drops while low-bandwidth conversations continue receiving service.

This helps prevent aggressive flows from consuming excessive buffering resources.

These interface-level WFQ options are legacy mechanisms. Modern IOS XE platforms more commonly use MQC class-based queuing.

---

## Class-Based Weighted Fair Queuing (CBWFQ)

**Class-Based Weighted Fair Queuing (CBWFQ)** extends weighted scheduling by allowing administrators to define traffic classes explicitly.

Unlike traditional WFQ, which automatically separates individual conversations, CBWFQ classifies packets according to user-defined criteria.

For example:

- Voice traffic marked DSCP EF.
- Business applications marked DSCP AF31.
- Bulk traffic marked DSCP AF11.
- All remaining traffic.

Each class can be assigned a separate queue.

```text
          Incoming Traffic
                 |
                 v
           Classification
                 |
       +---------+----------+
       |         |          |
       v         v          v
   +--------+ +--------+ +--------+
   |Business| | Bulk   | |Default |
   | Queue  | | Queue  | | Queue  |
   +--------+ +--------+ +--------+
       |         |          |
       +---------+----------+
                 |
                 v
          CBWFQ Scheduler
                 |
                 v
          Egress Interface
```

CBWFQ allows administrators to control bandwidth allocation at the class level.

### CBWFQ Characteristics

- Uses MQC class maps to classify traffic.
- Provides separate scheduling queues for configured traffic classes.
- Supports minimum bandwidth guarantees.
- Allows classes to use additional bandwidth when available.
- Supports class-specific queue limits.
- Can be combined with WRED for congestion avoidance.
- Can be combined with strict priority scheduling through LLQ.

**CBWFQ is not the same as strict priority queuing.**

CBWFQ uses weighted scheduling. LLQ adds strict priority treatment to selected classes.

See [Low Latency Queuing (LLQ)](llq.md).

---

## CBWFQ Bandwidth Configuration

Cisco IOS XE supports several commands for controlling bandwidth allocation.

| Command | Function |
|---|---|
| `bandwidth 20000` | Minimum bandwidth guarantee of 20,000 Kbps |
| `bandwidth percent 30` | Minimum bandwidth guarantee of 30% |
| `bandwidth remaining percent 30` | Configure percentage-based excess bandwidth sharing |
| `bandwidth remaining ratio 3` | Configure an excess-bandwidth weight of 3 |

The distinction between `bandwidth` and `bandwidth remaining` is particularly important on modern IOS XE routers.

### Minimum Bandwidth with `bandwidth`

The `bandwidth` command configures a minimum service rate for a class.

For example:

```text
R1(config-pmap-c)# bandwidth 20000
```

This provides a minimum bandwidth guarantee of 20 Mbps during congestion, assuming sufficient matching traffic is offered.

Alternatively:

```text
R1(config-pmap-c)# bandwidth percent 30
```

This guarantees 30% of the applicable scheduling bandwidth.

On a 100-Mbps interface, the minimum is 30 Mbps.

**A minimum bandwidth guarantee is not a maximum bandwidth limit.**

If additional bandwidth is available, the class can transmit above its configured minimum.

To enforce a maximum transmission rate, use shaping.

### Excess Bandwidth with `bandwidth remaining`

The `bandwidth remaining` command controls how available bandwidth is shared among competing bandwidth queues.

On Cisco IOS XE platforms using the three-parameter scheduling model, the command configures the **excess weight (Ex)**.

For example:

```text
R1(config-pmap-c)# bandwidth remaining ratio 3
```

This configures a relative excess weight of 3.

A queue with weight 3 receives three times the excess-bandwidth share of a competing queue with weight 1, assuming both remain backlogged and no other constraints apply.

The command does not configure a maximum bandwidth limit.

---

## CBWFQ Example: Minimum Guarantees

Consider a 100-Mbps interface with the following policy:

```text
R1(config)# policy-map WAN-QOS
R1(config-pmap)# class BUSINESS
R1(config-pmap-c)# bandwidth percent 40
R1(config-pmap)# class BULK
R1(config-pmap-c)# bandwidth percent 20
R1(config-pmap)# class class-default
```

The configured minimum guarantees are:

| Class | Minimum bandwidth |
|---|---:|
| BUSINESS | 40 Mbps |
| BULK | 20 Mbps |
| class-default | No explicit minimum |

Assume all three queues are continuously backlogged.

### Three-Parameter IOS XE Scheduler

On Cisco IOS XE platforms using the three-parameter scheduler, minimum guarantees are satisfied before excess bandwidth is distributed.

First, the guaranteed bandwidth is accounted for:

```text
BUSINESS = 40 Mbps
BULK     = 20 Mbps

Total guaranteed = 60 Mbps
```

The remaining bandwidth is:

```text
100 - 60 = 40 Mbps
```

By default, each bandwidth queue has an excess weight of 1.

Therefore, the 40 Mbps of excess capacity is divided equally among the three backlogged queues.

```text
40 / 3 = 13.33 Mbps
```

The approximate final allocations are:

| Class | Minimum | Excess | Total |
|---|---:|---:|---:|
| BUSINESS | 40 Mbps | 13.33 Mbps | 53.33 Mbps |
| BULK | 20 Mbps | 13.33 Mbps | 33.33 Mbps |
| class-default | 0 Mbps | 13.33 Mbps | 13.33 Mbps |
| **Total** | **60 Mbps** | **40 Mbps** | **100 Mbps** |

This example illustrates the operation of Cisco's three-parameter scheduler under sustained congestion.

### Two-Parameter vs. Three-Parameter Scheduling

Traditional Cisco IOS used a different scheduling model in which the configured class bandwidth also influenced its scheduling weight.

On modern IOS XE platforms such as the ASR 1000, **minimum guarantees and excess-bandwidth sharing are separate parameters**.

Therefore, do not assume the excess bandwidth will be distributed in proportion to the configured minimum bandwidth guarantees.

For example, a 40-Mbps minimum and a 20-Mbps minimum do not automatically result in a 2:1 excess-bandwidth ratio on the three-parameter scheduler.

To explicitly control excess sharing, configure `bandwidth remaining`.

---

## CBWFQ Example: Excess Bandwidth Ratios

Consider the following configuration:

```text
R1(config)# policy-map WAN-QOS
R1(config-pmap)# class BUSINESS
R1(config-pmap-c)# bandwidth remaining ratio 3
R1(config-pmap)# class BULK
R1(config-pmap-c)# bandwidth remaining ratio 1
R1(config-pmap)# class class-default
```

The excess weights are:

```text
BUSINESS      = 3
BULK          = 1
class-default = 1
```

The total weight is:

```text
3 + 1 + 1 = 5
```

Assuming all classes are continuously backlogged, the approximate allocations on a 100-Mbps interface are:

```text
BUSINESS      = 3/5 × 100 = 60 Mbps
BULK          = 1/5 × 100 = 20 Mbps
class-default = 1/5 × 100 = 20 Mbps
```

| Class | Excess weight | Approximate bandwidth |
|---|---:|---:|
| BUSINESS | 3 | 60 Mbps |
| BULK | 1 | 20 Mbps |
| class-default | 1 (default) | 20 Mbps |

The `class-default` queue receives a default excess weight of 1 on this scheduling implementation.

If BUSINESS offers only 30 Mbps, it receives the traffic it requires, and the remaining available bandwidth can be redistributed among the other backlogged queues.

**Excess-bandwidth weights determine relative shares during competition, not fixed reservations that remain unused when a class is idle.**

---

## `bandwidth remaining percent`

Cisco IOS XE also supports percentage-based excess-bandwidth allocation.

For example:

```text
R1(config)# policy-map WAN-QOS
R1(config-pmap)# class BUSINESS
R1(config-pmap-c)# bandwidth remaining percent 75
R1(config-pmap)# class class-default
```

On the three-parameter IOS XE scheduler, this assigns:

- BUSINESS: 75% excess weight.
- class-default: Remaining 25% excess weight.

When both queues remain backlogged and no minimum or maximum rates constrain the allocation, the expected shares are approximately 75% and 25%.

### Unassigned Percentages

When using `bandwidth remaining percent`, IOS XE distributes unassigned percentages among bandwidth queues without explicit percentage assignments.

For example:

```text
R1(config)# policy-map WAN-QOS
R1(config-pmap)# class BUSINESS
R1(config-pmap-c)# bandwidth remaining percent 60
R1(config-pmap)# class BULK
R1(config-pmap-c)# shape average 50000000
R1(config-pmap)# class class-default
```

There are three bandwidth queues:

- BUSINESS has an explicit 60% excess weight.
- BULK has no explicit excess percentage.
- class-default has no explicit excess percentage.

The remaining 40% is distributed equally between BULK and class-default.

Therefore:

| Class | Excess weight |
|---|---:|
| BUSINESS | 60% |
| BULK | 20% |
| class-default | 20% |

BULK also has a maximum shaping rate of 50 Mbps.

If the shaper prevents BULK from using its full excess share, that unused bandwidth can become available to other eligible queues.

This example illustrates that **a class with only a shaping command still participates in the bandwidth scheduler**.

---

## CBWFQ and Flow-Based Fair Queuing

An important limitation of ordinary CBWFQ is that **each class normally uses a single FIFO queue internally** on modern IOS XE implementations such as the ASR 1000.

Suppose a BUSINESS class contains:

- A large file transfer.
- An interactive application.

Both flows match the same class map and enter the same class queue.

```text
          BUSINESS Class
       +----------------------+
       | Single FIFO Queue    |
       |                      |
       | File Transfer        |
       | Interactive Traffic  |
       +----------------------+
                  |
                  v
             Scheduler
```

Even if BUSINESS receives a guaranteed 30 Mbps, a high-bandwidth flow can create a large backlog that delays packets from the interactive application.

CBWFQ provides fairness **between classes**, but a single FIFO queue does not necessarily provide fairness **between flows within a class**.

### Enabling Flow-Based Fair Queuing Within a Class

Cisco IOS XE supports the `fair-queue` command within a bandwidth class on supported platforms.

For example:

```text
R1(config)# policy-map WAN-QOS
R1(config-pmap)# class BUSINESS
R1(config-pmap-c)# bandwidth percent 30
R1(config-pmap-c)# fair-queue
```

This enables flow-based fair queuing within the BUSINESS class.

Instead of using one FIFO queue for all matching packets, the router creates multiple flow queues.

```text
             BUSINESS Class
                    |
                    v
             Flow Hashing
                    |
          +---------+---------+
          |         |         |
          v         v         v
       +-----+   +-----+   +-----+
       |Flow1|   |Flow2|   |Flow3|
       +-----+   +-----+   +-----+
          |         |         |
          +---------+---------+
                    |
                    v
            Fair Queue Scheduler
                    |
                    v
             BUSINESS Class
             Bandwidth Share
```

The scheduler first allocates bandwidth to the BUSINESS class according to its CBWFQ configuration.

Within the class, flow-based fair queuing distributes service among the individual flow queues.

This provides fairness at two levels:

1. **Between classes:** CBWFQ bandwidth scheduling.
2. **Within a class:** Flow-based fair queuing.

### Flow Queues on QFP-Based Platforms

Cisco documents that on ASR 1000 QFP-based platforms, the `fair-queue` command creates 16 flow queues within the class.

Packets are assigned to the flow queues using a hash.

Because there are a finite number of queues, multiple flows can hash into the same queue.

Therefore, flow-based fair queuing improves fairness but does not necessarily provide a dedicated queue for every individual flow.

The number of supported flow queues varies by platform and implementation.

### Flow-Based Fair Queuing over Tunnels

When flow-based fair queuing is applied to encapsulated traffic, the scheduler may hash the **outer tunnel headers** rather than the original inner packet headers.

Because packets belonging to different flows can share identical outer source and destination addresses, they may all hash into the same flow queue, defeating per-flow fairness.

On supported IOS XE platforms, this can be addressed using two commands:

```text
R1(config)# interface Tunnel0
R1(config-if)# qos pre-classify
R1(config)# policy-map WAN-QOS
R1(config-pmap)# class class-default
R1(config-pmap-c)# fair-queue pre-classify
```

The commands work together:

- `qos pre-classify` preserves the original packet header information for QoS processing.
- `fair-queue pre-classify` instructs the flow-based scheduler to use the **inner packet headers** when hashing flows into queues.

The policy can be attached to the tunnel interface or underlying physical interface, subject to platform support.

**Important:** `qos pre-classify` alone does not enable inner-header hashing for `fair-queue`. Both commands are required for this behavior.

Cisco introduced `fair-queue pre-classify` support in IOS XE 16.4.

---

## `class-default` and Flow-Based Fair Queuing

The `class-default` class handles traffic that does not match any preceding class in the policy map.

On supported IOS XE platforms, it is possible to enable flow-based fair queuing within `class-default`.

For example:

```text
R1(config)# policy-map WAN-QOS
R1(config-pmap)# class BUSINESS
R1(config-pmap-c)# bandwidth percent 30
R1(config-pmap)# class class-default
R1(config-pmap-c)# fair-queue
```

This configuration provides:

- A dedicated bandwidth queue for BUSINESS.
- Flow-based fair queuing for unmatched traffic.

This is useful when the default class contains many competing flows.

### Is `fair-queue` Enabled by Default?

The answer depends on the platform and QoS implementation.

In traditional Cisco IOS CBWFQ, unmatched traffic commonly received flow-based WFQ treatment by default.

On the ASR 1000 IOS XE three-parameter scheduler, the default bandwidth class queue is FIFO unless `fair-queue` is explicitly enabled.

Therefore, **do not assume that `class-default` automatically uses flow-based fair queuing on every IOS XE router**.

Verify the applied scheduling configuration.

---

## Complete CBWFQ Configuration Example

Consider a router with a 100-Mbps WAN interface.

The objective is to:

- Guarantee 40 Mbps to business applications.
- Guarantee 20 Mbps to bulk traffic.
- Provide flow fairness for all remaining traffic.

### Step 1: Configure Class Maps

```text
R1(config)# class-map match-any BUSINESS
R1(config-cmap)# match dscp af31
R1(config)# class-map match-any BULK
R1(config-cmap)# match dscp af11
```

### Step 2: Configure the Policy Map

```text
R1(config)# policy-map WAN-QOS
R1(config-pmap)# class BUSINESS
R1(config-pmap-c)# bandwidth percent 40
R1(config-pmap)# class BULK
R1(config-pmap-c)# bandwidth percent 20
R1(config-pmap)# class class-default
R1(config-pmap-c)# fair-queue
```

This policy:

- Creates a bandwidth queue for BUSINESS.
- Creates a bandwidth queue for BULK.
- Enables flow-based fair queuing within `class-default`.

The configured 40% and 20% allocations are minimum guarantees, not maximum rates.

### Step 3: Apply the Policy Outbound

```text
R1(config)# interface GigabitEthernet1
R1(config-if)# service-policy output WAN-QOS
```

The router now uses the policy to schedule traffic leaving GigabitEthernet1.

If the physical interface is faster than the actual available WAN bandwidth, a parent shaping policy may be required.

See [4.4.g Hierarchical QoS (HQoS)](../hqos/overview.md).

---

## FIFO vs. WFQ vs. CBWFQ

| Characteristic | FIFO | WFQ | CBWFQ |
|---|---|---|---|
| Traffic classification | None | Automatic flow identification | Administrator-defined classes |
| Number of queues | One | Multiple flow queues | One queue per class by default |
| Scheduling | Arrival order | Weighted fairness among flows | Weighted scheduling among classes |
| Explicit bandwidth guarantees | No | No | Yes |
| IP Precedence awareness | No | Yes, traditionally | Only if used in classification |
| Protects individual flows from one another | No | Yes | Only with appropriate flow queuing |
| Supports strict priority | No | No | Through LLQ |
| Primary configuration | None | `fair-queue` | MQC `bandwidth` commands |

The key distinction is the level at which fairness is implemented:

- **FIFO:** No fairness mechanism beyond arrival order.
- **WFQ:** Fairness among individual flows.
- **CBWFQ:** Fairness and bandwidth allocation among defined traffic classes.

CBWFQ can optionally use flow-based fair queuing within a class.

---

## Verification and Troubleshooting

The primary IOS XE verification command is:

```text
R1# show policy-map interface GigabitEthernet1
```

This displays the applied QoS policy, including configured classes, bandwidth allocations, and available queue statistics.

### Useful Commands

| Command | Purpose |
|---|---|
| `show policy-map` | Display configured policy maps |
| `show policy-map WAN-QOS` | Display the CBWFQ configuration |
| `show policy-map interface GigabitEthernet1` | Display applied policy and per-class counters |
| `show class-map` | Display class maps and matching criteria |
| `show interfaces GigabitEthernet1` | Check interface utilization and output drops |
| `show running-config interface GigabitEthernet1` | Verify the service-policy attachment |
| `show queueing interface Serial0/0` | Display queuing statistics on supported traditional IOS interfaces |
| `show platform hardware qfp active feature qos interface GigabitEthernet1` | Inspect hardware QoS on supported QFP-based platforms |

### Interpreting CBWFQ Statistics

For each class, examine:

- Packets and bytes matched.
- Configured minimum bandwidth.
- Offered traffic rate.
- Transmitted traffic rate.
- Queue depth.
- Queue limit.
- Packet drops.

If a class receives less bandwidth than expected, verify that it has enough traffic to use its allocation and that congestion actually exists at the point where the policy is applied.

If a class receives more bandwidth than its configured `bandwidth` value, that is not necessarily a problem. CBWFQ allows classes to use additional available bandwidth.

### Common Troubleshooting Scenarios

**A single flow dominates a class**
- Verify whether the class uses a single FIFO queue.
- Consider enabling `fair-queue` within the class.
- Check platform support and hashing behavior.

**A class does not receive its expected minimum bandwidth**
- Verify that the traffic matches the intended class.
- Check the interface bandwidth and parent shaping rate.
- Verify that the class has sufficient offered traffic.
- Examine competing priority traffic and platform constraints.

**A class receives unexpected excess bandwidth**
- Check for explicit `bandwidth remaining` commands.
- Remember that bandwidth queues may have a default excess weight of 1.
- Distinguish minimum bandwidth guarantees from excess-bandwidth weights.

**Unexpected queue drops**
- Check queue limits and traffic bursts.
- Determine whether a single flow is filling the queue.
- Examine whether additional bandwidth or flow fairness is required.

---

## Key Points

- **FIFO** uses one queue and transmits packets in arrival order.
- **WFQ** automatically identifies flows and schedules them according to weights.
- Traditional Cisco WFQ considers IP Precedence when assigning scheduling weights.
- WFQ protects low-bandwidth flows from being delayed behind large backlogs belonging to other flows.
- WFQ uses a packet-size-aware virtual scheduling mechanism.
- **CBWFQ** uses MQC to define traffic classes and allocate bandwidth.
- CBWFQ bandwidth guarantees are minimums, not maximum limits.
- On three-parameter IOS XE schedulers, minimum bandwidth and excess-bandwidth sharing are controlled separately.
- `bandwidth` configures a minimum service rate.
- `bandwidth remaining ratio` configures a relative excess-bandwidth weight.
- `bandwidth remaining percent` configures percentage-based excess sharing.
- On supported QFP-based IOS XE platforms, a bandwidth class uses one FIFO queue by default.
- `fair-queue` enables flow-based fairness within a class.
- The ASR 1000 implementation uses 16 flow queues when `fair-queue` is enabled.
- `class-default` does not necessarily use WFQ by default on modern IOS XE.
- CBWFQ provides class-based scheduling; LLQ adds strict priority scheduling.
- `show policy-map interface` is the primary verification command.

---

## References

- [Cisco IOS XE 17.x: QoS Scheduling](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/qos/b-quality-of-service/m_qos-scheduling.html)
- [Cisco IOS XE 17: Congestion Management Overview](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/qos_conmgt/configuration/xe-17/qos-conmgt-xe-17-book/qos-conmgt-oview.html)
- [Cisco IOS XE 17.x: Applying QoS Features Using MQC](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/qos/b-quality-of-service/m_qos-apply.html)
- [Cisco IOS XE: Queue Limits and WRED](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/quality-of-service/quality-of-service/m_qos-queue_management_and_congestion_avoidance.html)
- [Cisco IOS QoS Command Reference: `fair-queue`](https://www.cisco.com/c/en/us/td/docs/ios/qos/command/reference/qos_book/qos_d1.html)