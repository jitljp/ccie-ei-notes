# 4.4.g Hierarchical QoS (HQoS)

**Hierarchical Quality of Service (HQoS)** allows multiple QoS policies to be organized into a hierarchy, enabling different QoS mechanisms to operate at different levels.

HQoS is configured using Cisco's **Modular QoS CLI (MQC)** by nesting one policy map inside another.

The most common HQoS deployment combines:

- **Parent policy:** Controls the aggregate transmission rate using traffic shaping.
- **Child policy:** Determines how the available bandwidth is distributed among traffic classes using queuing and scheduling.

This is commonly called a **shape-on-parent, queue-on-child** configuration.

HQoS is particularly useful on WAN connections where the physical interface speed exceeds the bandwidth purchased from the service provider.

---

## Why HQoS Is Needed

Consider a router connected to a service provider.

```text
                1 Gbps Interface
                      |
                      |
                     R1
                      |
                      | 1 Gbps Ethernet
                      |
                 ISP Network
                      |
                      | 20 Mbps Service
                      |
                 Remote Site
```

R1 has a 1 Gbps physical interface, but the service provider only allows 20 Mbps of traffic.

If R1 transmits above 20 Mbps, the service provider may police the excess traffic.

This can result in:

- Packet loss.
- Increased latency and jitter.
- Dropped voice packets.
- Unpredictable treatment of important applications.

R1 should therefore shape its traffic to 20 Mbps.

However, **shaping alone does not determine which applications receive priority during congestion**.

For example, if R1 receives 30 Mbps of traffic but can transmit only 20 Mbps, it must determine which packets to transmit first.

The solution is HQoS:

1. The **parent policy** shapes all outbound traffic to 20 Mbps.
2. The **child policy** prioritizes voice traffic and distributes the remaining bandwidth among other applications.

This allows R1 to control both the aggregate transmission rate and the treatment of individual traffic classes.

---

## HQoS Hierarchy

A typical HQoS configuration consists of two levels.

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

### Parent Policy

The parent policy controls the aggregate traffic handled by its classes.

In a typical WAN deployment, the parent policy contains:

```text
R1(config)# policy-map PARENT
R1(config-pmap)# class class-default
R1(config-pmap-c)# shape average 20000000
```

This configures a 20 Mbps average shaping rate.

Because `class-default` matches all traffic not matched by another class, and is the only class in this parent policy, the shaper applies to all traffic handled by the policy.

The parent shaper controls the aggregate transmission rate, including traffic belonging to priority classes in the child policy.

### Child Policy

The child policy determines how packets are handled within the bandwidth available from the parent.

For example:

```text
R1(config)# policy-map CHILD
R1(config-pmap)# class VOICE
R1(config-pmap-c)# priority percent 10
R1(config-pmap-c)# exit
R1(config-pmap)# class BUSINESS
R1(config-pmap-c)# bandwidth remaining percent 60
```

This policy:

- Gives VOICE strict priority scheduling.
- Assigns BUSINESS a weighted share of the remaining bandwidth.
- Handles unmatched traffic through `class-default`.

The child policy does not independently increase the bandwidth available from the parent.

**All child classes collectively operate within the aggregate rate enforced by the parent shaper.**

---

## Parent-Child Policy Relationships

A child policy is attached to a parent class using the `service-policy` command.

For example:

```text
R1(config)# policy-map PARENT
R1(config-pmap)# class class-default
R1(config-pmap-c)# shape average 20000000
R1(config-pmap-c)# service-policy CHILD
```

The command:

```text
R1(config-pmap-c)# service-policy CHILD
```

Nests the CHILD policy under the parent's `class-default`.

The parent policy is then applied to an interface:

```text
R1(config)# interface GigabitEthernet2
R1(config-if)# service-policy output PARENT
```

**Only the parent policy is attached directly to the interface.**

The child policy is applied indirectly through the parent policy.

### Classification at Different Levels

Each level can classify traffic.

For a packet to match a child class, it must first match the parent class containing that child policy.

For example:

```text
Parent Class: class-default
    |
    +-- Child Class: VOICE
    |
    +-- Child Class: BUSINESS
    |
    +-- Child Class: class-default
```

All traffic enters the parent's `class-default`.

The child policy then classifies traffic into VOICE, BUSINESS, or its own `class-default`.

More complex hierarchies can classify traffic by customer, VLAN, or other criteria at one level, then by application at another.

The supported number of hierarchy levels and available classification methods depend on the platform.

See [Hierarchical Policy Maps and Parent-Child Policies](parent-child.md).

---

## Hierarchical Shaping and Queuing

The most common HQoS design uses **shaping at the parent level** and **queuing at the child level**.

These mechanisms perform different functions.

| Mechanism | Typical level | Purpose |
|---|---|---|
| Traffic shaping | Parent | Limits aggregate transmission rate |
| Strict priority queuing (LLQ) | Child | Prioritizes latency-sensitive traffic |
| CBWFQ | Child | Provides bandwidth allocations to traffic classes |
| WRED | Child | Performs congestion avoidance within eligible queues |
| Queue limits | Child | Controls queue capacity |

The parent determines **how fast traffic can be transmitted**.

The child determines **which traffic is transmitted when bandwidth is limited**.

### Hierarchical Scheduling

Consider three child classes:

- VOICE: 5 Mbps offered.
- BUSINESS: 15 Mbps offered.
- DEFAULT: 10 Mbps offered.

Total offered traffic:

```text
VOICE     =  5 Mbps
BUSINESS  = 15 Mbps
DEFAULT   = 10 Mbps
--------------------
TOTAL     = 30 Mbps
```

The parent shapes traffic to 20 Mbps.

Consequently, the child scheduler must distribute the available 20 Mbps among the competing classes.

If VOICE uses strict priority queuing, its eligible packets receive preferential scheduling.

The remaining capacity is distributed among BUSINESS and DEFAULT according to their configured scheduling parameters.

Packets that cannot be transmitted immediately may remain in their queues.

If queues become full, packets may be dropped.

**The parent shaper controls the aggregate rate, while the child scheduler controls how competing classes share that rate.**

---

## Bandwidth Percentages in HQoS

One important HQoS concept is how percentage-based commands are interpreted.

In a typical shape-on-parent, queue-on-child policy, child bandwidth percentages are calculated relative to the **parent shaping rate**, rather than the physical interface speed.

Consider:

```text
Physical interface = 1 Gbps
Parent shape rate  = 20 Mbps
```

### Priority Percent

Suppose the child contains:

```text
R1(config-pmap-c)# priority percent 10
```

The priority rate is calculated as:

```text
20 Mbps × 10% = 2 Mbps
```

Therefore, the configured priority rate is 2 Mbps.

On Cisco IOS XE platforms using a conditional priority policer:

- When there is no congestion, the priority class may exceed 2 Mbps if bandwidth is available.
- During congestion, the priority class is policed to its configured rate, subject to burst allowances.

This differs from an explicit always-on policer, which enforces its rate regardless of congestion.

### Bandwidth Percent

Suppose a child class contains:

```text
R1(config-pmap-c)# bandwidth percent 30
```

The minimum bandwidth allocation is:

```text
20 Mbps × 30% = 6 Mbps
```

This provides a minimum scheduling allocation of 6 Mbps when the class is backlogged and the applicable bandwidth guarantees can be honored.

It is not a maximum rate.

The class may use additional available bandwidth.

### Bandwidth Remaining Percent

Consider:

```text
R1(config-pmap-c)# bandwidth remaining percent 60
```

Unlike `bandwidth percent`, this command configures a relative scheduling weight for distributing excess bandwidth.

For example, if the child has:

- VOICE with priority scheduling.
- BUSINESS with `bandwidth remaining percent 60`.
- DEFAULT with no explicit bandwidth allocation.

On the documented IOS XE scheduler, the remaining percentage is assigned to `class-default`, giving BUSINESS and DEFAULT a 60:40 excess-bandwidth sharing relationship.

If VOICE consumes 2 Mbps of the 20 Mbps parent rate, approximately 18 Mbps remains.

When both nonpriority classes are continuously backlogged, their weighted shares would be approximately:

```text
BUSINESS = 18 Mbps × 60% = 10.8 Mbps
DEFAULT  = 18 Mbps × 40% =  7.2 Mbps
```

These are illustrative congestion-sharing values, not fixed transmission rates.

Actual scheduling depends on offered traffic, other minimum guarantees, shaping constraints, and platform behavior.

**Do not confuse `bandwidth percent` with `bandwidth remaining percent`.**

See [Hierarchical Shaping and Queuing](shaping-queuing.md).

---

## Basic HQoS Configuration

Consider the following requirements:

- Physical WAN interface: GigabitEthernet2.
- Service provider bandwidth: 20 Mbps.
- VOICE: DSCP EF, strict priority with a 10% rate.
- BUSINESS: DSCP AF31, 60% of excess bandwidth.
- DEFAULT: All remaining traffic.

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
```

This defines the queuing and scheduling treatment.

### Step 3: Configure Parent Policy

```text
R1(config)# policy-map PARENT-SHAPE
R1(config-pmap)# class class-default
R1(config-pmap-c)# shape average 20000000
R1(config-pmap-c)# service-policy CHILD-QOS
```

The parent:

1. Shapes aggregate traffic to 20 Mbps.
2. Invokes CHILD-QOS to determine the treatment of individual traffic classes.

### Step 4: Apply the Parent Policy

```text
R1(config)# interface GigabitEthernet2
R1(config-if)# service-policy output PARENT-SHAPE
```

The hierarchy is now attached to GigabitEthernet2.

### Result

| Class | Configuration | Behavior |
|---|---|---|
| Parent | `shape average 20000000` | Aggregate shaping to 20 Mbps |
| VOICE | `priority percent 10` | Strict priority with a 2 Mbps conditional priority rate |
| BUSINESS | `bandwidth remaining percent 60` | Receives a weighted share of excess bandwidth |
| DEFAULT | Implicit `class-default` | Receives remaining weighted bandwidth |

The parent shapes the total traffic, while the child distributes transmission opportunities among the classes.

---

## HQoS Verification

The main verification command is:

```text
R1# show policy-map interface GigabitEthernet2
```

This displays the applied hierarchical policy and available statistics.

The output should identify:

1. The parent service policy.
2. The parent's `class-default`.
3. The configured shaping rate.
4. The nested child policy.
5. The child traffic classes and their scheduling parameters.

Depending on the platform, the output may also include:

- Offered rates.
- Transmitted rates.
- Priority policing counters.
- Queue depths.
- Queue drops.
- Bandwidth allocations.
- Shaping statistics.

### Useful Verification Commands

| Command | Purpose |
|---|---|
| `show policy-map` | Display configured policy maps |
| `show policy-map PARENT-SHAPE` | Examine the parent policy |
| `show policy-map CHILD-QOS` | Examine the child policy |
| `show policy-map interface GigabitEthernet2` | Verify the active hierarchy and statistics |
| `show class-map` | Verify classification criteria |
| `show running-config interface GigabitEthernet2` | Confirm the parent policy is applied |
| `show interfaces GigabitEthernet2` | Check interface utilization and drops |

On supported QFP-based platforms, additional hardware QoS verification commands may be available.

### Troubleshooting Considerations

When HQoS does not behave as expected, check:

- **Parent shaping rate:** Is it configured to the correct service-provider rate?
- **Policy attachment:** Is the parent policy applied to the correct interface and direction?
- **Child attachment:** Is the child policy nested under the intended parent class?
- **Classification:** Are packets matching the expected child classes?
- **Bandwidth reference:** Are percentages calculated using the expected parent rate?
- **Priority policing:** Are priority packets being dropped because the configured rate is too low?
- **Queue drops:** Are queues filling because offered traffic exceeds the shaped rate?
- **Platform support:** Does the platform support the configured hierarchy and QoS actions?

---

## HQoS Design Considerations

### 1. Shape to the Actual Available Bandwidth

The parent shaping rate should reflect the bandwidth available through the service provider.

For example, a 1 Gbps Ethernet interface connected to a 50 Mbps WAN service should generally be shaped according to the 50 Mbps service rate.

This prevents the router from transmitting faster than the downstream connection can handle.

Encapsulation overhead must also be considered when selecting a shaping rate.

### 2. Avoid Oversubscribing Priority Traffic

Strict priority queues are serviced before ordinary bandwidth queues.

Without appropriate limits, excessive priority traffic could starve other applications.

Use conditional priority limits or explicit policing as appropriate for the design.

### 3. Plan Bandwidth Allocations Carefully

Child bandwidth allocations must be consistent with the parent shaping rate.

For example, configuring a 20 Mbps parent shaper does not provide sufficient capacity to satisfy 30 Mbps of simultaneous minimum bandwidth requirements.

### 4. HQoS Support Is Platform-Dependent

Not every Cisco IOS XE platform supports the same HQoS features.

Differences include:

- Supported hierarchy depth.
- Whether HQoS is supported on physical interfaces, subinterfaces, tunnels, or other interfaces.
- Available parent and child QoS actions.
- Supported queuing and scheduling mechanisms.
- Bandwidth calculation and hardware accounting behavior.

Some platforms support only two levels, while others support deeper hierarchies.

Always verify the platform's capabilities and actual installed policy behavior.

---

## HQoS Compared to Flat QoS

| Feature | Flat QoS | HQoS |
|---|---|---|
| Policy structure | Single policy level | Multiple nested levels |
| Aggregate shaping | Possible | Commonly implemented at parent level |
| Per-class queuing | Yes | Commonly implemented at child level |
| Multiple scheduling levels | No | Yes, subject to platform support |
| Typical deployment | Interface-based QoS | WAN shaping with per-class queuing |
| Configuration complexity | Lower | Higher |

Flat QoS is sufficient when only one level of traffic classification and treatment is required.

HQoS is useful when the design requires multiple levels of bandwidth control or scheduling.

---

## Topics Covered in This Section

The following pages examine HQoS in more detail:

1. [Hierarchical Policy Maps and Parent-Child Policies](parent-child.md)
   - Parent and child policy relationships.
   - Classification across hierarchy levels.
   - Policy nesting and attachment.

2. [Hierarchical Shaping and Queuing](shaping-queuing.md)
   - Parent shaping and child scheduling.
   - Bandwidth allocations and percentage calculations.
   - Priority queuing and hierarchical scheduler behavior.

3. [HQoS Configuration](configuration.md)
   - Complete IOS XE configurations.
   - Verification and troubleshooting.
   - Practical WAN HQoS scenarios.

---

## Key Points

- HQoS allows multiple MQC policies to be organized into a hierarchy.
- The most common deployment uses a parent shaper and child queuing policy.
- The parent controls the aggregate transmission rate.
- The child controls how bandwidth is distributed among traffic classes.
- A child policy is nested using `service-policy` under a parent class.
- The parent policy is attached to the interface using `service-policy input` or `service-policy output`, as supported by the platform and configured actions.
- Percentage-based child bandwidth allocations typically use the nearest applicable parent shaping rate as their reference.
- `priority percent` provides strict priority treatment with a rate-based conditional policer on supported platforms.
- `bandwidth percent` configures a minimum bandwidth allocation.
- `bandwidth remaining percent` configures relative sharing of excess bandwidth.
- `show policy-map interface` is the primary HQoS verification command.
- Supported hierarchy depth and QoS actions depend on the platform.

---

## References

- [Cisco IOS XE 17.x: Applying QoS Features Using the MQC](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/qos/b-quality-of-service/m_qos-mqc-0.html)
- [Cisco IOS XE 17.x: QoS Hierarchical Scheduling](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/qos/b-quality-of-service/m_qos-hierarchical_scheduling.html)
- [Cisco IOS XE 17.x: QoS Scheduling](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/qos/b-quality-of-service/m_qos-scheduling.html)
- [Cisco IOS XE 17.x: QoS Packet Policing](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/qos/b-quality-of-service/m_qos-policing.html)