# Hierarchical Shaping and Queuing

**Hierarchical shaping and queuing** combines traffic shaping and class-based queuing within a Hierarchical QoS (HQoS) policy.

The most common design uses:

- **Parent policy:** Shapes traffic to an aggregate transmission rate.
- **Child policy:** Uses queuing and scheduling to distribute the available bandwidth among traffic classes.

This is commonly called **shape-on-parent, queue-on-child**.

It allows a router to enforce an aggregate bandwidth limit while providing differentiated treatment to applications such as voice, video, business traffic, and best-effort data.

---

## Shape-on-Parent, Queue-on-Child

Consider a router with a 1 Gbps physical interface connected to a service provider offering a 20 Mbps WAN service.

               LAN
                |
                | 1 Gbps
                v
               R1
                |
                v
        +------------------+
        |   Child Policy   |
        |                  |
        |   LLQ / CBWFQ    |
        +------------------+
           |     |     |
           v     v     v
         VOICE  DATA  DEFAULT
           |     |     |
           +-----+-----+
                 |
                 v
        +------------------+
        |  Parent Policy   |
        |                  |
        | Shape: 20 Mbps   |
        +------------------+
                 |
                 v
          GigabitEthernet2
          Physical: 1 Gbps
                 |
                 v
             ISP Network

Without HQoS, R1 could transmit at its physical interface speed, potentially exceeding the ISP's contracted bandwidth.

A parent shaper prevents this by controlling the aggregate rate.

However, shaping alone does not provide differentiated scheduling among applications.

The child policy determines which packets receive transmission opportunities when traffic exceeds the shaping rate.

### Basic Configuration

Configure the child policy:

```text
R1(config)# policy-map CHILD-QOS
R1(config-pmap)# class VOICE
R1(config-pmap-c)# priority percent 10
R1(config-pmap-c)# exit
R1(config-pmap)# class BUSINESS
R1(config-pmap-c)# bandwidth remaining percent 60
```

Configure the parent:

```text
R1(config)# policy-map PARENT-SHAPE
R1(config-pmap)# class class-default
R1(config-pmap-c)# shape average 20000000
R1(config-pmap-c)# service-policy CHILD-QOS
```

Apply the parent:

```text
R1(config)# interface GigabitEthernet2
R1(config-if)# service-policy output PARENT-SHAPE
```

This creates two scheduling levels.

The parent limits aggregate transmission to 20 Mbps.

The child distributes that bandwidth among VOICE, BUSINESS, and `class-default`.

**The child policy cannot bypass the parent shaping rate.**

---

## Parent Shaping Behavior

The parent's `shape average` command controls the maximum aggregate transmission rate.

For example:

```text
R1(config-pmap-c)# shape average 20000000
```

This configures an average shaping rate of 20 Mbps.

The parent shaper applies to **all traffic matching its class**, including traffic belonging to the child's priority queue.

### Aggregate Rate Calculation

Suppose the child has:

- VOICE: 5 Mbps.
- BUSINESS: 15 Mbps.
- DEFAULT: 10 Mbps.

Total offered traffic:

```text
VOICE      5 Mbps
BUSINESS  15 Mbps
DEFAULT   10 Mbps
-----------------
TOTAL     30 Mbps
```

However:

```text
Parent shape rate = 20 Mbps
```

Therefore, the aggregate transmission rate is limited to approximately 20 Mbps.

The remaining 10 Mbps of offered traffic cannot be transmitted immediately.

Packets may be buffered in their respective child queues.

If the offered load remains above 20 Mbps, queues may eventually fill and packets may be dropped.

---

## Parent Shaping and Child Scheduling

The parent and child schedulers perform different functions.

| Scheduler | Responsibility |
|---|---|
| Parent | Controls the maximum aggregate transmission rate |
| Child | Determines which traffic classes receive transmission opportunities |
| Physical interface | Transmits packets subject to its physical capacity |

On Cisco IOS XE platforms using hierarchical hardware scheduling, each level participates in the scheduling decision.

The child selects eligible packets according to its queuing configuration.

The parent determines whether those packets can be transmitted within its shaping constraints.

### Example

Suppose:

```text
Parent shaping rate = 20 Mbps

Child:
  VOICE    = Priority
  BUSINESS = Weighted queue
  DEFAULT  = Weighted queue
```

When all three child queues contain packets:

1. The child scheduler gives eligible VOICE packets preferential treatment.
2. The remaining transmission opportunities are distributed between BUSINESS and DEFAULT.
3. All transmitted packets count toward the parent's shaping rate.
4. If the parent cannot transmit additional packets, they remain queued until scheduling capacity becomes available.

The parent does not independently increase the bandwidth available to the child.

**Strict priority traffic is subject to the parent shaping rate, just like ordinary bandwidth traffic.**

---

## Child Bandwidth Calculations

A critical HQoS concept is the bandwidth reference used by percentage-based QoS commands.

In a typical shape-on-parent, queue-on-child policy, the child's percentage-based bandwidth allocations use the **nearest applicable parent shaping rate**.

They do not use the physical interface speed.

For example:

```text
Physical interface = 1 Gbps
Parent shape rate  = 20 Mbps
```

Child percentages are calculated relative to 20 Mbps.

### `priority percent`

Consider:

```text
R1(config-pmap-c)# priority percent 10
```

The configured priority rate is:

```text
20 Mbps × 10% = 2 Mbps
```

Therefore, VOICE receives strict priority scheduling with a configured priority rate of 2 Mbps.

On platforms using conditional priority policing, this rate is enforced during congestion.

### `bandwidth percent`

Consider:

```text
R1(config-pmap-c)# bandwidth percent 30
```

The minimum bandwidth allocation is:

```text
20 Mbps × 30% = 6 Mbps
```

This provides a 6 Mbps minimum scheduling allocation for the class when it is backlogged, subject to the applicable scheduler and available capacity.

It does not limit the class to 6 Mbps.

The class may use more bandwidth when additional capacity is available.

### `bandwidth remaining percent`

Consider:

```text
R1(config-pmap-c)# bandwidth remaining percent 60
```

This command behaves differently.

It configures a **relative weight for sharing excess bandwidth**.

The percentage is not simply 60% of the parent shaping rate.

The amount of bandwidth actually received depends on:

- Priority traffic.
- Minimum bandwidth allocations.
- Other classes' excess-bandwidth weights.
- Which classes are backlogged.

This distinction is important when calculating HQoS scheduling behavior.

---

## Priority Queuing in HQoS

**Low Latency Queuing (LLQ)** is commonly used in the child policy to provide strict priority treatment for delay-sensitive traffic.

For example:

```text
R1(config)# policy-map CHILD-QOS
R1(config-pmap)# class VOICE
R1(config-pmap-c)# priority percent 10
```

The VOICE class is placed into a strict priority queue.

On the documented Cisco IOS XE QFP scheduler, `priority percent 10` also configures a **conditional priority policer**.

### Conditional Priority Policing

A conditional priority policer enforces its configured rate when the relevant scheduler is congested.

Consider:

```text
Parent shape rate = 20 Mbps
VOICE priority    = 10%

Priority rate     = 2 Mbps
```

#### Scenario 1: No Congestion

Suppose:

```text
VOICE     = 5 Mbps
BUSINESS  = 8 Mbps
DEFAULT   = 4 Mbps
-------------------
TOTAL     = 17 Mbps
```

The total offered traffic is below the 20 Mbps parent rate.

There is no sustained congestion.

On the documented conditional priority policing implementation, VOICE can transmit all 5 Mbps even though its configured priority rate is 2 Mbps.

The priority policer does not drop packets merely because the VOICE rate exceeds 2 Mbps.

#### Scenario 2: Congestion

Suppose:

```text
VOICE     = 5 Mbps
BUSINESS  = 15 Mbps
DEFAULT   = 10 Mbps
-------------------
TOTAL     = 30 Mbps
```

The parent is congested because 30 Mbps is offered to a 20 Mbps shaper.

The conditional priority policer now enforces its configured rate.

Approximately:

```text
VOICE transmitted = 2 Mbps
VOICE excess      = 3 Mbps
```

Excess VOICE packets may be dropped by the priority policer, subject to burst allowances.

The remaining approximately 18 Mbps is available for other classes.

### Why Priority Policing Matters

Without appropriate priority traffic limits, strict priority queues can starve ordinary bandwidth queues.

For example:

```text
Parent shape = 20 Mbps

VOICE offered = 25 Mbps
DATA offered  = 15 Mbps
```

If VOICE has unrestricted strict priority scheduling, it may consume the entire 20 Mbps shaping capacity.

DATA might receive no transmission opportunities.

A correctly configured priority policer protects bandwidth for other classes.

**Priority traffic is not exempt from the parent shaper, and unlimited priority traffic can starve lower-priority queues.**

### Always-On Priority Policing

An explicit policer can be used to enforce a maximum rate regardless of congestion.

For example:

```text
R1(config-pmap-c)# priority
R1(config-pmap-c)# police cir 2000000
```

On supported QFP-based platforms, this combines:

- Unconstrained strict priority scheduling.
- An explicit 2 Mbps policer.

The explicit policer is always active.

Therefore, traffic exceeding the policing profile can be dropped even when the parent is not congested.

This differs from `priority percent 10`, which uses conditional policing on the documented QFP implementation.

See [Low Latency Queuing (LLQ)](../congestion/llq.md).

---

## `bandwidth percent` vs. `bandwidth remaining percent`

These two commands have different scheduling semantics on modern IOS XE platforms.

| Command | Purpose |
|---|---|
| `bandwidth percent 30` | Configures a minimum bandwidth allocation |
| `bandwidth remaining percent 30` | Configures a relative excess-bandwidth sharing weight |

### Minimum Bandwidth Allocation

Suppose:

```text
Parent shape rate = 20 Mbps
```

The child contains:

```text
policy-map CHILD
 class BUSINESS
  bandwidth percent 30
```

The minimum allocation is:

```text
20 Mbps × 30% = 6 Mbps
```

When BUSINESS is backlogged, its local scheduler attempts to provide at least 6 Mbps.

If BUSINESS needs more than 6 Mbps, it may receive additional available bandwidth.

The extra bandwidth is distributed according to the scheduler's excess-bandwidth weights.

### Excess Bandwidth Sharing

Now consider:

```text
policy-map CHILD
 class BUSINESS
  bandwidth remaining percent 60
 class class-default
  bandwidth remaining percent 40
```

The two bandwidth queues have excess-bandwidth weights of:

```text
BUSINESS = 60%
DEFAULT  = 40%
```

When both queues are continuously backlogged, they share available excess bandwidth in a 60:40 relationship.

Suppose a parent shaper provides 20 Mbps, and neither class has an additional minimum bandwidth allocation or priority competitor.

The expected rates are:

```text
BUSINESS = 20 Mbps × 60% = 12 Mbps
DEFAULT  = 20 Mbps × 40% =  8 Mbps
```

These are scheduling shares when both queues have sufficient demand.

If DEFAULT only needs 2 Mbps, BUSINESS may use the unused capacity.

The percentages do not impose maximum transmission rates.

### Excess Bandwidth Is Not Always the Entire Parent Rate

Suppose a child also includes:

```text
class VOICE
 priority percent 10
```

If VOICE is consuming 2 Mbps during congestion:

```text
Parent rate  = 20 Mbps
VOICE usage  =  2 Mbps
----------------------
Remaining    = 18 Mbps
```

With a 60:40 sharing relationship:

```text
BUSINESS = 18 Mbps × 60% = 10.8 Mbps
DEFAULT  = 18 Mbps × 40% =  7.2 Mbps
```

These calculations assume:

- Both nonpriority classes are continuously backlogged.
- No additional minimum bandwidth allocations.
- No other shaping restrictions.
- VOICE consumes exactly 2 Mbps.
- No relevant scheduling or accounting overhead.

The actual packet rates may differ because of bursts, packet sizes, and hardware scheduler implementation.

---

## `bandwidth remaining ratio`

Cisco IOS XE also supports:

```text
R1(config-pmap-c)# bandwidth remaining ratio <weight>
```

This assigns a relative excess-bandwidth weight rather than a percentage.

For example:

```text
R1(config)# policy-map CHILD-QOS
R1(config-pmap)# class BUSINESS
R1(config-pmap-c)# bandwidth remaining ratio 4
R1(config-pmap-c)# exit
R1(config-pmap)# class BULK
R1(config-pmap-c)# bandwidth remaining ratio 2
R1(config-pmap-c)# exit
R1(config-pmap)# class class-default
R1(config-pmap-c)# bandwidth remaining ratio 1
```

The scheduling weights are:

```text
BUSINESS = 4
BULK     = 2
DEFAULT  = 1
-------------
TOTAL    = 7
```

The shares are:

```text
BUSINESS = 4/7 = 57.14%
BULK     = 2/7 = 28.57%
DEFAULT  = 1/7 = 14.29%
```

If 14 Mbps is available for these classes:

```text
BUSINESS = 14 Mbps × 4/7 = 8 Mbps
BULK     = 14 Mbps × 2/7 = 4 Mbps
DEFAULT  = 14 Mbps × 1/7 = 2 Mbps
```

These are relative scheduling shares, not configured maximum rates.

### Default Excess Weight

On the documented IOS XE QFP scheduler, every ordinary bandwidth queue has a default excess weight of 1.

Therefore:

```text
policy-map CHILD
 class BUSINESS
  bandwidth remaining ratio 3
```

creates a 3:1 relationship between BUSINESS and the implicit `class-default` queue.

When both queues are backlogged:

```text
BUSINESS = 75%
DEFAULT  = 25%
```

This behavior does not require explicitly configuring a ratio on `class-default`.

### Ratio vs. Percent

Both forms configure excess-bandwidth scheduling weights.

| Command | Behavior |
|---|---|
| `bandwidth remaining ratio 3` | Assigns weight 3 to the class |
| `bandwidth remaining percent 75` | Assigns a 75% excess-bandwidth share |
| `bandwidth percent 75` | Configures a minimum bandwidth allocation |

With `bandwidth remaining percent`, unassigned percentages are distributed among eligible bandwidth queues without explicit percentage allocations.

With `bandwidth remaining ratio`, each queue has an independent weight, which defaults to 1.

**`bandwidth remaining ratio` is useful when relative weights are easier to configure than percentages.**

---

## Hierarchical Scheduling with Minimum Bandwidth Guarantees

Cisco IOS XE QFP-based routers use a scheduler that distinguishes three parameters:

| Parameter | Meaning | Typical command |
|---|---|---|
| Min | Minimum bandwidth allocation | `bandwidth` |
| Max | Maximum scheduling rate | `shape` |
| Ex | Excess-bandwidth sharing weight | `bandwidth remaining` |

These are often called the scheduler's **Min, Max, and Ex** parameters.

### Example

Consider a 20 Mbps parent shaper with:

```text
policy-map CHILD
 class VOICE
  priority percent 10
 class BUSINESS
  bandwidth percent 30
 class class-default
```

The relevant values are:

```text
Parent Max          = 20 Mbps
VOICE priority rate =  2 Mbps
BUSINESS Min        =  6 Mbps
```

During sustained congestion, assuming VOICE consumes its 2 Mbps priority rate:

- VOICE receives priority treatment up to its conditional policing rate.
- BUSINESS has a 6 Mbps minimum allocation within its local scheduler.
- The remaining capacity is distributed among eligible bandwidth queues.
- BUSINESS can receive more than 6 Mbps if additional bandwidth is available.

The remaining 12 Mbps after accounting for VOICE and BUSINESS's minimum is not necessarily assigned exclusively to `class-default`.

Its distribution depends on excess-bandwidth weights and which queues have traffic.

### Important Hierarchical Limitation

On the Cisco IOS XE QFP hierarchical scheduler, minimum bandwidth guarantees have **local significance**.

A minimum bandwidth allocation configured inside a child policy does not automatically become a minimum guarantee at every higher scheduling level.

For example:

```text
Physical Interface = 100 Mbps

Parent A:
  Shape = 80 Mbps
  Child BUSINESS bandwidth = 60 Mbps

Parent B:
  Shape = 80 Mbps
  Child BUSINESS bandwidth = 60 Mbps
```

If both parents compete for the same 100 Mbps physical interface, the router cannot satisfy both 60 Mbps minimum allocations simultaneously.

The parent schedulers must share the available physical bandwidth.

Cisco describes this behavior as **no minimum bandwidth propagation**.

This matters especially when multiple parent shapers are oversubscribed.

---

## Parent Shaping and Priority Propagation

On supported Cisco IOS XE QFP platforms, strict priority treatment can be propagated through the scheduling hierarchy.

This is called **priority propagation**.

Consider:

```text
Physical Interface
       |
       v
   Parent Shaper
       |
       v
   Child Scheduler
       |
    +--+--+
    |     |
    v     v
  VOICE  DATA
  (PQ)   (BQ)
```

The child scheduler distinguishes:

- Priority traffic.
- Ordinary bandwidth traffic.

When traffic is presented to the parent scheduler, the parent can preserve the distinction between priority and ordinary traffic.

Therefore, priority packets can continue receiving preferential treatment across scheduling levels.

### Key Rules

On the documented QFP hierarchical scheduler:

1. Priority traffic counts toward the parent's shaping rate.
2. Priority traffic receives preferential treatment over ordinary bandwidth traffic.
3. Parent excess-bandwidth weights control sharing among bandwidth traffic, not priority traffic.
4. Minimum bandwidth guarantees configured at a child level are not propagated to the parent.
5. Priority traffic may compete across different parent schedulers.

### Multiple Priority Levels

If a child has:

```text
class VOICE
 priority level 1

class VIDEO
 priority level 2
```

Level 1 receives higher priority than level 2 within that child scheduler.

However, when priority traffic propagates to a higher-level scheduler, the documented QFP implementation does not preserve the distinction between P1 and P2 at that higher level.

Both are treated as priority traffic.

**Priority levels are locally significant within their scheduling level.**

This becomes important when designing multi-level HQoS with multiple parent classes or subinterfaces.

See [Low Latency Queuing (LLQ)](../congestion/llq.md).

---

## Multiple Parent Shapers Sharing a Physical Interface

HQoS is also useful when multiple logical interfaces share a physical interface.

Consider:

```text
                  Physical Interface
                       100 Mbps
                           |
                 +---------+---------+
                 |                   |
                 v                   v
             VLAN 100             VLAN 200
             Shape 80M            Shape 80M
                 |                   |
                 v                   v
              CHILD-A             CHILD-B
```

Each parent limits its own traffic to 80 Mbps.

However, the physical interface has only 100 Mbps of capacity.

The configured shaping rates total:

```text
80 Mbps + 80 Mbps = 160 Mbps
```

The physical interface cannot transmit 160 Mbps.

The parent scheduling entries therefore compete for the available physical bandwidth.

### Equal Parent Weights

If both parent schedules have equal excess weights and all competing traffic is ordinary bandwidth traffic, the available capacity can be divided approximately equally:

```text
VLAN 100 = 50 Mbps
VLAN 200 = 50 Mbps
```

Neither receives its full 80 Mbps shaping allowance while both are saturated.

### Different Parent Weights

Suppose the parent schedules use these weights:

```text
VLAN 100 = 1
VLAN 200 = 2
```

The approximate sharing relationship becomes:

```text
VLAN 100 = 1/3
VLAN 200 = 2/3
```

With 100 Mbps available:

```text
VLAN 100 = 33.33 Mbps
VLAN 200 = 66.67 Mbps
```

This assumes:

- Both parents are continuously backlogged.
- There is no competing priority traffic.
- No other scheduler constraints.
- The platform implements the documented QFP excess-weight behavior.

### Configuring Parent Weights

On supported IOS XE platforms, the parent can combine shaping with a remaining-bandwidth weight.

For example:

```text
R1(config)# policy-map PARENT-100
R1(config-pmap)# class class-default
R1(config-pmap-c)# shape average 80000000
R1(config-pmap-c)# bandwidth remaining ratio 1
R1(config-pmap-c)# service-policy CHILD-QOS
```

```text
R1(config)# policy-map PARENT-200
R1(config-pmap)# class class-default
R1(config-pmap-c)# shape average 80000000
R1(config-pmap-c)# bandwidth remaining ratio 2
R1(config-pmap-c)# service-policy CHILD-QOS
```

Apply the policies to separate subinterfaces:

```text
R1(config)# interface GigabitEthernet2.100
R1(config-subif)# encapsulation dot1Q 100
R1(config-subif)# service-policy output PARENT-100
R1(config-subif)# exit
R1(config)# interface GigabitEthernet2.200
R1(config-subif)# encapsulation dot1Q 200
R1(config-subif)# service-policy output PARENT-200
```

The parent `bandwidth remaining ratio` commands control how the two parent schedules compete for ordinary bandwidth traffic.

**The parent shaping rate and parent excess-bandwidth weight serve different purposes.**

- `shape average`: Limits maximum transmission rate.
- `bandwidth remaining ratio`: Influences relative scheduling share when parents compete for capacity.

The physical-interface scheduling behavior may also be affected by priority propagation and other platform-specific constraints.

---

## Queue Limits in Hierarchical QoS

Each child queue can have its own queue limit.

For example:

```text
R1(config)# policy-map CHILD-QOS
R1(config-pmap)# class BUSINESS
R1(config-pmap-c)# bandwidth remaining percent 60
R1(config-pmap-c)# queue-limit 200 packets
```

The BUSINESS queue can buffer up to its configured queue limit, subject to hardware behavior and resource availability.

When its queue fills, additional packets may be dropped.

### Why Queue Limits Matter with Shaping

A parent shaper may create congestion even when the physical interface is underutilized.

Consider:

```text
Physical interface = 1 Gbps
Parent shaper      = 20 Mbps
Offered traffic    = 30 Mbps
```

Although the physical interface can transmit at 1 Gbps, the parent allows an average rate of only 20 Mbps.

Packets accumulate in the child queues.

If the offered load remains at 30 Mbps, the queue backlog grows until traffic rates change or packets are dropped.

### Queue Delay

Suppose a queue contains 250,000 bytes.

At a queue service rate of 10 Mbps:

```text
Queue data = 250,000 × 8
           = 2,000,000 bits

Drain time = 2,000,000 / 10,000,000
           = 0.2 seconds
           = 200 ms
```

The buffered traffic represents approximately 200 ms of transmission time at that service rate.

This is significant for delay-sensitive applications.

The calculation is illustrative and excludes changes in scheduling share, new arrivals, and encapsulation overhead.

**Queue delay depends on the actual queue service rate, not simply the physical interface bandwidth.**

### Time-Based Queue Limits

On supported QFP-based platforms:

```text
R1(config-pmap-c)# queue-limit 50 ms
```

This configures the queue limit using a time-based value.

The router derives a buffering limit from the applicable scheduler rate.

This can be useful with parent shaping because the relevant scheduling rate may be much lower than the physical interface speed.

See [Queue Limits and Tail Drop](../congestion/queue-limits.md).

---

## WRED Within a Child Policy

HQoS can also combine parent shaping with congestion avoidance in child queues.

For example:

```text
R1(config)# policy-map CHILD-QOS
R1(config-pmap)# class BUSINESS
R1(config-pmap-c)# bandwidth remaining percent 60
R1(config-pmap-c)# queue-limit 200 packets
R1(config-pmap-c)# random-detect dscp-based
R1(config-pmap-c)# random-detect dscp af31 60 150 10
```

This configures DSCP-based WRED for the BUSINESS class.

The WRED profile for AF31 uses:

```text
Minimum threshold = 60 packets
Maximum threshold = 150 packets
MPD               = 10
```

When the average queue depth rises above the minimum threshold, WRED can begin probabilistically dropping AF31 packets.

This provides congestion feedback before the queue reaches its limit.

### WRED and Parent Shaping

The parent shaper can cause child queues to build up because it restricts the aggregate transmission rate.

WRED operates within the child queue and responds to that congestion.

The relationship is:

```text
Parent Shaper
     |
     | Rate = 20 Mbps
     |
     v
Child Policy
     |
     +-- VOICE
     |   Priority Queue
     |
     +-- BUSINESS
     |   CBWFQ + WRED
     |
     +-- DEFAULT
         Ordinary Queue
```

WRED does not change the parent shaping rate.

It controls how packets are admitted to an eligible child queue as congestion develops.

Strict priority queues are generally protected using priority policing rather than WRED.

See [Weighted Random Early Detection (WRED)](../congestion/wred.md).

---

## Complete Hierarchical Shaping and Queuing Example

Consider the following requirements:

- Physical interface: GigabitEthernet2.
- Physical interface speed: 1 Gbps.
- WAN service bandwidth: 50 Mbps.
- VOICE: DSCP EF, priority rate of 10%.
- BUSINESS: DSCP AF31, 60% of excess bandwidth.
- DEFAULT: Remaining 40% of excess bandwidth.
- BUSINESS queue limit: 200 packets.
- Enable WRED for BUSINESS.
- DEFAULT queue limit: 200 packets.

### Step 1: Configure Class Maps

```text
R1(config)# class-map match-all VOICE
R1(config-cmap)# match dscp ef
R1(config-cmap)# exit
R1(config)# class-map match-all BUSINESS
R1(config-cmap)# match dscp af31
```

### Step 2: Configure Child Policy

```text
R1(config)# policy-map CHILD-QOS
R1(config-pmap)# class VOICE
R1(config-pmap-c)# priority percent 10
R1(config-pmap-c)# exit
R1(config-pmap)# class BUSINESS
R1(config-pmap-c)# bandwidth remaining percent 60
R1(config-pmap-c)# queue-limit 200 packets
R1(config-pmap-c)# random-detect dscp-based
R1(config-pmap-c)# random-detect dscp af31 60 150 10
R1(config-pmap-c)# exit
R1(config-pmap)# class class-default
R1(config-pmap-c)# bandwidth remaining percent 40
R1(config-pmap-c)# queue-limit 200 packets
```

### Step 3: Configure Parent Policy

```text
R1(config)# policy-map PARENT-SHAPE
R1(config-pmap)# class class-default
R1(config-pmap-c)# shape average 50000000
R1(config-pmap-c)# service-policy CHILD-QOS
```

### Step 4: Apply the Parent

```text
R1(config)# interface GigabitEthernet2
R1(config-if)# service-policy output PARENT-SHAPE
```

### Resulting Hierarchy

```text
                    GigabitEthernet2
                       1 Gbps
                          |
                          v
                 +-----------------+
                 |  PARENT-SHAPE   |
                 |                 |
                 | Shape 50 Mbps   |
                 +-----------------+
                          |
                          v
                 +-----------------+
                 |    CHILD-QOS    |
                 +-----------------+
                          |
             +------------+------------+
             |            |            |
             v            v            v
         +--------+  +----------+  +---------+
         | VOICE  |  | BUSINESS |  | DEFAULT |
         |        |  |          |  |         |
         | LLQ    |  | CBWFQ    |  | CBWFQ   |
         | 10%    |  | 60% Ex   |  | 40% Ex  |
         |        |  | WRED     |  |         |
         +--------+  +----------+  +---------+
             |            |            |
             +------------+------------+
                          |
                          v
                    Transmission
                     <= 50 Mbps
```

### Bandwidth Calculations

The parent shapes traffic to 50 Mbps.

VOICE priority rate:

```text
50 Mbps × 10% = 5 Mbps
```

During congestion, assume VOICE uses its full configured priority rate.

Available bandwidth for BUSINESS and DEFAULT:

```text
50 Mbps - 5 Mbps = 45 Mbps
```

BUSINESS:

```text
45 Mbps × 60% = 27 Mbps
```

DEFAULT:

```text
45 Mbps × 40% = 18 Mbps
```

| Class | Configured treatment | Illustrative congested rate |
|---|---|---:|
| VOICE | Priority, 10% | 5 Mbps |
| BUSINESS | 60% excess weight | 27 Mbps |
| DEFAULT | 40% excess weight | 18 Mbps |
| **Total** | | **50 Mbps** |

These values assume all queues remain backlogged and no additional scheduling constraints apply.

If one class does not require its calculated share, other classes may use the unused bandwidth.

For example, if BUSINESS only needs 10 Mbps and VOICE uses 5 Mbps, DEFAULT may use the remaining 35 Mbps, subject to its offered traffic.

The 40% excess-bandwidth weight is not a maximum transmission limit.

---

## HQoS Verification

The primary verification command is:

```text
R1# show policy-map interface GigabitEthernet2
```

This displays the parent policy and nested child policy.

### Verify Parent Shaping

Look for the parent's shaping configuration:

```text
Service-policy output: PARENT-SHAPE
  Class-map: class-default
    shape (average) cir 50000000
```

Depending on the platform, the output may also display:

- Target shaping rate.
- Offered rate.
- Transmitted rate.
- Queue depth.
- Packet drops.

### Verify Child Scheduling

The same command displays the nested CHILD-QOS policy.

Look for:

```text
Service-policy: CHILD-QOS
```

Under the child, verify:

- VOICE is using priority scheduling.
- The configured priority rate is based on the 50 Mbps parent rate.
- BUSINESS has the intended excess-bandwidth share.
- DEFAULT has the intended excess-bandwidth share.
- WRED is configured and its counters are available.
- Queue limits are installed as expected.

### Useful Commands

| Command | Purpose |
|---|---|
| `show policy-map` | Display configured QoS policies |
| `show policy-map PARENT-SHAPE` | Verify parent shaping configuration |
| `show policy-map CHILD-QOS` | Verify child queuing configuration |
| `show policy-map interface GigabitEthernet2` | Verify the active hierarchy |
| `show policy-map interface GigabitEthernet2 output` | Display outbound QoS statistics |
| `show running-config interface GigabitEthernet2` | Verify parent policy attachment |
| `show interfaces GigabitEthernet2` | Check interface traffic and drops |

On supported QFP-based routers, additional hardware information may be available:

```text
R1# show platform hardware qfp active feature qos interface GigabitEthernet2
```

The exact syntax and available information vary by platform.

---

## Troubleshooting Hierarchical Shaping and Queuing

### Child Rates Are Calculated Using the Wrong Bandwidth

If a child uses:

```text
priority percent 10
```

and the parent shapes to 50 Mbps, the configured priority rate is typically 5 Mbps.

It is not 100 Mbps just because the physical interface operates at 1 Gbps.

Verify the parent shaping rate and the actual installed policy.

### Priority Traffic Experiences Drops

Possible causes:

- The priority rate is too low.
- The conditional priority policer is active during congestion.
- Traffic bursts exceed the allowed burst size.
- The parent shaping rate is too low.
- Other priority traffic competes for limited capacity.

Inspect priority policing and drop counters.

### Nonpriority Queues Are Starved

Possible causes:

- Priority traffic is not properly rate-limited.
- Multiple priority queues collectively consume too much bandwidth.
- Parent shapers are oversubscribed.
- The physical interface is congested.

Ensure that sufficient capacity remains for bandwidth queues after priority traffic is serviced.

### BUSINESS Does Not Receive Its Expected Percentage

Check:

- Whether the class is continuously backlogged.
- Whether the configured command is `bandwidth percent` or `bandwidth remaining percent`.
- Whether other classes have minimum bandwidth allocations.
- Whether priority traffic is consuming bandwidth.
- Whether another scheduler level is constraining the available capacity.

Remember that a `bandwidth remaining` percentage is a scheduling share, not a fixed transmission rate.

### Shaping Causes Excessive Delay

Possible causes:

- Offered traffic exceeds the parent shaping rate.
- Child queue limits are too large.
- Important applications are placed in ordinary queues.
- Traffic is excessively buffered.

Consider adjusting queue limits and checking traffic classification.

### Traffic Is Dropped Even Though the Physical Interface Is Not Full

This can be normal with HQoS.

For example:

```text
Physical interface = 1 Gbps
Parent shaper      = 20 Mbps
Offered traffic    = 30 Mbps
```

The physical interface may show only approximately 2% utilization.

However, the parent shaping rate is fully utilized.

The child queues can therefore experience congestion and packet loss.

**A parent shaper creates a logical congestion point independently of physical interface utilization.**

---

## Key Commands

| Objective | Command |
|---|---|
| Shape parent to 20 Mbps | `shape average 20000000` |
| Configure percentage-based parent shaping | `shape average percent 50` |
| Attach child policy | `service-policy CHILD-QOS` |
| Configure child strict priority | `priority` |
| Configure child conditional priority rate | `priority percent 10` |
| Configure child minimum bandwidth | `bandwidth percent 30` |
| Configure excess-bandwidth percentage | `bandwidth remaining percent 60` |
| Configure excess-bandwidth weight | `bandwidth remaining ratio 3` |
| Configure queue limit | `queue-limit 200 packets` |
| Enable DSCP-based WRED | `random-detect dscp-based` |
| Apply parent policy outbound | `service-policy output PARENT-SHAPE` |
| Verify hierarchy | `show policy-map interface` |

---

## Key Points

- Hierarchical shaping and queuing commonly uses a parent shaper and child queuing policy.
- The parent shaper limits aggregate transmission, including priority traffic.
- The child determines how traffic classes share the available bandwidth.
- A parent shaper establishes a maximum rate, not a minimum guarantee.
- Child percentage-based bandwidth allocations typically reference the nearest applicable parent shaping rate.
- `priority percent` configures strict priority with conditional priority policing on supported IOS XE QFP platforms.
- Conditional priority policing allows a priority class to exceed its configured rate when there is no congestion.
- An explicit policer can enforce a rate regardless of congestion.
- `bandwidth percent` configures a minimum bandwidth allocation.
- `bandwidth remaining percent` configures relative excess-bandwidth sharing.
- `bandwidth remaining ratio` configures an excess-bandwidth weight.
- Ordinary bandwidth queues have a default excess weight of 1 on the documented QFP scheduler.
- On the QFP scheduler, minimum bandwidth guarantees are locally significant and are not automatically propagated to higher scheduling levels.
- Priority propagation preserves priority treatment across hierarchical scheduling levels.
- Parent excess-bandwidth weights can control how multiple logical interfaces share physical bandwidth.
- Parent shaping can cause child-queue congestion even when the physical interface is not saturated.
- Queue limits and WRED can be configured within supported child queues.
- `show policy-map interface` is the primary verification command.

---

## References

- [Cisco IOS XE 17.x: QoS Hierarchical Scheduling](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/qos/b-quality-of-service/m_qos-hierarchical_scheduling.html)
- [Cisco IOS XE 17.x: QoS Scheduling](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/qos/b-quality-of-service/m_qos-scheduling.html)
- [Cisco IOS XE: 3-Level User-Defined Queuing Policy Support](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/quality-of-service/quality-of-service/m-3-level-user-defined-queuing-policy-support.html)
- [Cisco: Understand the Bandwidth and Priority Commands of a QoS Service Policy](https://www.cisco.com/c/en/us/support/docs/quality-of-service-qos/qos-packet-marking/10100-priorityvsbw.html)