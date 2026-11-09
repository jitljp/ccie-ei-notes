# Traffic Policing

**Traffic policing** is a QoS mechanism that enforces a configured traffic rate by immediately applying an action to packets based on whether they conform to a traffic profile.

The most common actions are:
- **Transmit** packets that conform to the configured rate.
- **Drop** packets that exceed the configured rate.
- **Remark and transmit** exceeding packets with a lower QoS marking.

Unlike traffic shaping, **policing does not buffer excess packets for later transmission**.

Policing is commonly used at network boundaries to enforce bandwidth contracts, limit particular traffic classes, and protect network resources from excessive traffic.

---

## Traffic Policing Operation

A policer uses a **token bucket** to determine whether packets conform to the configured traffic profile.

Consider a router receiving traffic at 15 Mbps with a policer configured for 10 Mbps:

```text
        Incoming Traffic
             15 Mbps
                |
                v
        +---------------+
        |    Policer    |
        | CIR = 10 Mbps |
        +---------------+
                |
          +-----+-----+
          |           |
          v           v
       Conform      Exceed
          |           |
          v           v
       Transmit      Drop
```

The policer attempts to enforce an average transmitted rate of approximately 10 Mbps over time, allowing bursts according to its configured token bucket.

It does not necessarily transmit exactly 10 Mbps every second.

### Packet Processing

For a basic single-rate, two-color policer:

1. A packet arrives and matches a traffic class.
2. The policer calculates the tokens accumulated since the previous packet arrival.
3. The token count is updated, subject to the bucket capacity (Bc).
4. The policer compares the packet's accounted size to the available tokens.
5. If sufficient tokens exist, the packet conforms and consumes tokens.
6. Otherwise, the packet exceeds and does not consume tokens.
7. The configured conform or exceed action is immediately applied.

The token replenishment calculation is:

```text
Tokens added (bytes) = CIR × elapsed time / 8
```

Here, CIR is expressed in bits per second and elapsed time in seconds.

**The token count is logically replenished over time, but Cisco's class-based policer calculates the replenishment when packets arrive.** It does not need to update the bucket continuously while no packets are arriving.

See [Token Bucket Fundamentals (CIR, PIR, Bc, Be)](token-bucket.md) for a detailed explanation.

### Example: Packet-by-Packet Policing

Suppose a policer has:
- CIR = 1 Mbps (125,000 bytes/second).
- Bc = 12,500 bytes.
- Packet accounting based on the stated packet sizes.
- Initial bucket = 12,500 bytes.

| Event | Tokens before event | Result | Tokens remaining |
|---|---:|---|---:|
| 10,000-byte packet arrives | 12,500 | Conform | 2,500 |
| 20 ms elapses | 2,500 | Add 2,500 tokens | 5,000 |
| 4,000-byte packet arrives | 5,000 | Conform | 1,000 |
| 10 ms elapses | 1,000 | Add 1,250 tokens | 2,250 |
| 3,000-byte packet arrives | 2,250 | Exceed | 2,250 |

The final packet exceeds because the bucket contains insufficient tokens.

If `exceed-action drop` is configured, it is discarded.

However, a smaller packet arriving at the same instant could conform if its size does not exceed the 2,250 available tokens.

**Policing decisions depend on the available tokens and packet size, not simply on whether the instantaneous traffic rate exceeds CIR.**

---

## Policing Parameters

Cisco IOS XE policers use several parameters to define traffic profiles.

| Parameter | Description | Units |
|---|---|---|
| **CIR** | Committed Information Rate, the committed token replenishment rate | bps |
| **PIR** | Peak Information Rate, used by two-rate policers | bps |
| **Bc** | Committed token bucket capacity | Bytes |
| **Be** | Excess or peak token bucket capacity, depending on the policing algorithm | Bytes |

### Committed Information Rate (CIR)

CIR specifies the long-term rate at which committed tokens are replenished.

For example:

```text
R1(config-pmap-c)# police cir 10000000
```

Configures a policer with CIR = 10 Mbps.

Packets can temporarily exceed 10 Mbps when sufficient tokens have accumulated.

Therefore, CIR is a **committed token replenishment rate**, not a strict instantaneous transmission-speed limit.

### Committed Burst (Bc)

Bc determines the maximum number of committed tokens that can accumulate.

For example:

```text
R1(config-pmap-c)# police cir 10000000 bc 125000
```

Configures:
- CIR = 10 Mbps.
- Bc = 125,000 bytes.

This Bc corresponds to 100 ms of traffic at CIR:

```text
Tc = (Bc × 8) / CIR
   = (125,000 × 8) / 10,000,000
   = 0.1 seconds
   = 100 ms
```

A larger Bc permits larger bursts before packets are classified as exceeding.

A smaller Bc reduces burst tolerance and can result in more frequent drops, particularly with bursty TCP traffic.

> Although the CIR is configured in **bits per second**, in policing Bc is configured in **bytes**.

### Default Burst Size

When Bc is not explicitly configured, Cisco IOS XE calculates a default.

Cisco's IOS XE router policing documentation describes a default Bc corresponding to **250 ms of traffic at CIR**.

For example, at 10 Mbps:

```text
Bc = (10,000,000 × 0.250) / 8
   = 312,500 bytes
```

However, defaults vary between Cisco platforms and QoS implementations.

Some platforms also round burst sizes to values supported by their forwarding hardware.

Use `show policy-map interface` to determine the actual configured burst allowance.

---

## Policing Actions

Cisco IOS XE uses the `police` command to configure the traffic profile and the actions associated with policing outcomes.

A policer can classify traffic into two or three outcomes:

| Outcome | Meaning |
|---|---|
| **Conform** | Packet satisfies the committed traffic profile |
| **Exceed** | Packet exceeds the committed allowance but may satisfy an additional allowance |
| **Violate** | Packet exceeds the allowed profile of a three-color policer |

The exact conditions depend on the policing algorithm.

### Common Actions

| Action | Description |
|---|---|
| `transmit` | Forward the packet without changing its markings |
| `drop` | Discard the packet |
| `set-dscp-transmit` | Change the DSCP value and forward the packet |
| `set-prec-transmit` | Change the IP precedence and forward the packet |
| `set-cos-transmit` | Change the CoS marking and forward the packet, where supported |

Other actions, including MPLS traffic-class marking, may be supported depending on the platform and encapsulation.

The `-transmit` suffix is significant.

For example:

```text
R1(config-pmap-c-police)# exceed-action set-dscp-transmit af32
```

This action:
1. Changes the DSCP value to AF32.
2. Transmits the packet rather than dropping it.

### Default Policing Actions

For a basic single-rate, two-color policer, the default actions are:

| Outcome | Default action |
|---|---|
| Conform | `transmit` |
| Exceed | `drop` |

Therefore, this command:

```text
R1(config-pmap-c)# police cir 10000000
```

is equivalent in behavior to:

```text
R1(config-pmap-c)# police cir 10000000 conform-action transmit exceed-action drop
```

The configured policer drops exceeding packets even though no explicit `exceed-action drop` was entered.

For three-color policers, the default actions and available options should be verified for the specific platform and policing mode.

---

## Configuring Traffic Policing with MQC

Cisco IOS XE uses the **Modular QoS CLI (MQC)** to configure class-based policing.

The process involves:
1. Creating a class map.
2. Creating a policy map containing the policer.
3. Applying the policy to an interface.

### Example: Police Traffic to 10 Mbps

Suppose R1 must limit traffic marked AF31 to 10 Mbps.

**Step 1: Create the class map.**

```text
R1(config)# class-map match-all BUSINESS
R1(config-cmap)# match dscp af31
```

**Step 2: Create the policy map.**

```text
R1(config)# policy-map INGRESS-POLICE
R1(config-pmap)# class BUSINESS
R1(config-pmap-c)# police cir 10000000 conform-action transmit exceed-action drop
```

**Step 3: Apply the policy.**

```text
R1(config)# interface GigabitEthernet1
R1(config-if)# service-policy input INGRESS-POLICE
```

The resulting behavior is:

| Traffic | Result |
|---|---|
| AF31, conforming | Transmitted |
| AF31, exceeding | Dropped |
| Other DSCP values | Not affected by this policer |

Packets that do not match `BUSINESS` are handled by `class-default`.

Because no policing action is configured in `class-default`, those packets are not rate-limited by this policy.

### Policing All Traffic

To police all traffic matching the default class, configure the policer under `class-default`:

```text
R1(config)# policy-map POLICE-ALL
R1(config-pmap)# class class-default
R1(config-pmap-c)# police cir 10000000
```

This applies a single aggregate policer to traffic handled by `class-default`.

If the policy contains no preceding named classes, `class-default` includes all traffic eligible for that policy.

### Policing Multiple Classes

Each class can have its own policer.

For example:

```text
R1(config)# class-map match-all VOICE
R1(config-cmap)# match dscp ef
R1(config-cmap)# exit
R1(config)# class-map match-all BUSINESS
R1(config-cmap)# match dscp af31
R1(config-cmap)# exit
R1(config)# policy-map INGRESS-POLICE
R1(config-pmap)# class VOICE
R1(config-pmap-c)# police cir 2000000
R1(config-pmap-c)# exit
R1(config-pmap)# class BUSINESS
R1(config-pmap-c)# police cir 10000000
```

This creates two independent policers:

| Class | CIR | Shared token bucket? |
|---|---:|---|
| VOICE | 2 Mbps | No |
| BUSINESS | 10 Mbps | No |

The classes have separate token accounting.

Consequently, unused VOICE tokens do not increase the BUSINESS class's allowance.

**Separate class-based policers do not automatically enforce a combined aggregate rate.** An additional aggregate policy is required if the total traffic must also be limited.

---

## Policer-Based Remarking (Markdown)

A policer does not have to drop packets that exceed CIR.

It can instead **remark exceeding packets** so that downstream devices provide them with a less favorable forwarding treatment.

This is particularly useful with **Assured Forwarding (AF)**.

Within an AF class, different DSCP values indicate different drop precedence levels.

For example:

| DSCP | Drop precedence |
|---|---|
| AF31 (26) | Low |
| AF32 (28) | Medium |
| AF33 (30) | High |

### Example: Remark Excess AF31 Traffic

Suppose R1 should:
- Transmit conforming AF31 traffic without changing its marking.
- Remark exceeding AF31 traffic to AF32.
- Continue forwarding the exceeding packets.

Configure:

```text
R1(config)# class-map match-all BUSINESS
R1(config-cmap)# match dscp af31
R1(config-cmap)# exit
R1(config)# policy-map POLICE-BUSINESS
R1(config-pmap)# class BUSINESS
R1(config-pmap-c)# police cir 5000000 conform-action transmit exceed-action set-dscp-transmit af32
R1(config-pmap-c)# exit
R1(config)# interface GigabitEthernet1
R1(config-if)# service-policy input POLICE-BUSINESS
```

The result is:

| Original DSCP | Policer outcome | Final DSCP | Forwarded? |
|---|---|---|---|
| AF31 | Conform | AF31 | Yes |
| AF31 | Exceed | AF32 | Yes |

This allows the network to distinguish **in-contract traffic** from **out-of-contract traffic**.

A downstream router can use WRED to preferentially discard AF32 packets before AF31 packets during congestion.

However, this requires an appropriate downstream QoS policy.

**Remarking a packet to AF32 does not automatically cause it to be dropped or deprioritized.**

Also, notice that the policer is not enforcing a strict maximum forwarded rate in this example. Both conforming and exceeding packets are transmitted.

It is enforcing a **committed-rate marking profile**, not a hard forwarding-rate cap.

---

## Single-Rate and Two-Rate Policing

Cisco IOS XE supports multiple policing algorithms.

| Algorithm | Rates | Buckets | Outcomes |
|---|---|---|---|
| **Single-rate, two-color (1R2C)** | CIR | Bc | Conform, exceed |
| **Single-rate, three-color (1R3C)** | CIR | Bc, Be | Conform, exceed, violate |
| **Two-rate, three-color (2R3C)** | CIR, PIR | Bc, Be | Conform, exceed, violate |

### Single-Rate, Two-Color

This is the simplest form of policing.

```text
R1(config-pmap-c)# police cir 5000000 bc 125000 conform-action transmit exceed-action drop
```

It uses one token bucket and two possible outcomes.

### Single-Rate, Three-Color

A single-rate, three-color policer uses committed and excess token buckets associated with one rate, CIR.

```
R1(config-pmap-c)# police cir 5000000 bc 125000 be 250000 conform-action transmit exceed-action set-dscp-transmit af32 violate-action drop
```

It permits three different actions, such as transmitting conforming traffic, remarking exceeding traffic, and dropping violating traffic.

### Two-Rate, Three-Color

A two-rate policer uses independently replenished token buckets associated with CIR and PIR.

For example:

```text
R1(config-pmap-c)# police cir 5000000 bc 125000 pir 10000000 be 250000 conform-action transmit exceed-action set-dscp-transmit af32 violate-action drop
```

This creates a committed profile of 5 Mbps and a peak profile of 10 Mbps, with burst allowances.

Traffic within the committed profile conforms. Traffic outside the committed profile can exceed, while packets exceeding the peak token allowance violate.

The packet's actual outcome depends on the available tokens, not just its instantaneous arrival rate.

The algorithms and packet-level decisions are covered in [Single-Rate and Two-Rate Policing](single-two-rate.md).

---

## Color-Blind vs. Color-Aware Policing

Three-color policers can also be classified as **color-blind** or **color-aware**.

These modes determine whether the policer considers an existing traffic-profile designation.

| Mode | Behavior |
|---|---|
| **Color-blind** | Determines the packet's outcome without considering a previous conform/exceed/violate designation |
| **Color-aware** | Considers the packet's existing designation and does not promote previously downgraded traffic to a better designation |

For example, suppose one router marks excess AF31 traffic as AF32.

A downstream color-aware policer can recognize the packet as already exceeding its original traffic profile.

It can retain or worsen that designation, but should not promote the packet back to conforming.

On supported IOS XE router platforms, color-aware policing can use `conform-color` and `exceed-color` with class maps identifying the existing designation.

Color-aware policing is not the same thing as ordinary DSCP-based classification followed by a color-blind policer. The former uses the existing designation when making its policing decision.

---

## Percentage-Based Policing

Cisco IOS XE also supports specifying a policing rate as a **percentage of available bandwidth**.

For example:

```text
R1(config-pmap-c)# police cir percent 20
```

On a supported platform, this configures CIR to 20% of the relevant bandwidth reference.

At the top level of a policy applied to an interface, that reference is normally the interface's QoS bandwidth.

For example:

| Bandwidth reference | 20% CIR |
|---|---:|
| 10 Mbps | 2 Mbps |
| 100 Mbps | 20 Mbps |
| 1 Gbps | 200 Mbps |

The bandwidth reference is not necessarily the physical negotiated speed. It depends on the interface's configured or operational QoS bandwidth and platform behavior.

### Burst Size in Milliseconds

Supported IOS XE policers can also specify Bc as a time interval:

```text
R1(config-pmap-c)# police cir percent 20 bc 100 ms
```

This configures:
- CIR = 20% of the relevant bandwidth.
- Bc = 100 ms of traffic at the resulting CIR.

For a resulting CIR of 20 Mbps:

```text
Bc = (20,000,000 × 0.100) / 8
   = 250,000 bytes
```

Percentage-based policing is particularly useful when applying the same policy to interfaces with different bandwidths.

### Hierarchical Percentage-Based Policing

Within a hierarchical policy, the bandwidth reference can change.

For example, a child policer can calculate its rate as a percentage of the parent class's policing or shaping rate, rather than the physical interface bandwidth.

This is important when combining shaping and policing in HQoS designs.

See [4.4.g Hierarchical QoS (HQoS)](../hqos/overview.md).

---

## Ingress vs. Egress Policing

Traffic policing can be applied in either direction, depending on platform support.

### Ingress Policing

```text
R1(config-if)# service-policy input INGRESS-POLICE
```

Ingress policing is commonly used to:
- Enforce traffic contracts at network boundaries.
- Prevent excessive traffic from entering the network.
- Limit specific traffic classes.
- Remark out-of-contract traffic.

Packets dropped by an ingress policer do not continue through normal forwarding.

### Egress Policing

```text
R1(config-if)# service-policy output EGRESS-POLICE
```

Egress policing limits traffic as it leaves the interface.

However, it does not buffer exceeding packets to smooth the output rate.

For this reason, **egress shaping is generally preferred to egress policing when the goal is to match a provider's contracted WAN rate while minimizing unnecessary packet loss**.

Egress policing remains useful when a hard rate limit or specific remarking behavior is required.

Not all Cisco switches and routers support the same policing directions, algorithms, or actions.

---

## Policing and Congestion

An important distinction is that **policing does not require congestion to occur**.

For example, suppose:
- R1 has a 1 Gbps interface.
- A class is policed to 10 Mbps.
- The class receives sustained traffic at 20 Mbps.
- The physical interface is otherwise idle.

The policer can still drop approximately 10 Mbps of the traffic over time, subject to burst allowances.

It does not matter that the physical interface has sufficient unused bandwidth to forward all the traffic.

The configured policer enforces its traffic profile independently of physical interface congestion.

### Policing Is Not Queuing

Policing does not determine which class should transmit first during congestion.

A policer simply evaluates traffic and applies the configured actions.

Traffic that conforms and is transmitted by the policer can still experience queuing delay or congestion drops elsewhere in the forwarding process.

Also, CIR is not a minimum bandwidth guarantee.

A policer configured for 10 Mbps does not reserve 10 Mbps for its traffic class. It only regulates the traffic according to that profile.

For bandwidth guarantees and priority scheduling, Cisco uses mechanisms such as CBWFQ and LLQ.

---

## Policing Overhead Accounting

A policer measures traffic using a platform-defined **accounted packet length**.

This is important because the accounted length may differ from the packet's total physical transmission size.

For example, Cisco's ASR 1000 policing documentation describes Ethernet policing that includes:
- Layer 3 datagram length.
- Layer 2 Ethernet header.

It does not include:
- Ethernet FCS.
- Ethernet preamble and SFD.
- Interframe gap.

Consider a 100-byte IP packet transmitted over Ethernet without VLAN tagging:

| Component | Bytes |
|---|---:|
| IP packet | 100 |
| Ethernet header | 14 |
| Ethernet FCS | 4 |
| Preamble and SFD | 8 |
| Interframe gap | 12 |
| Total on-wire accounting | 138 |

The documented ASR 1000 default policing length for this packet is 114 bytes, not 138 bytes.

Therefore, even a policer enforcing an accounted rate of 100 Mbps could result in more than 100 Mbps of actual on-wire consumption.

This matters when enforcing strict provider bandwidth limits, particularly with small packets.

Some platforms support additional overhead-accounting options to compensate for encapsulation differences.

**Policing overhead accounting is platform-dependent.** Do not assume all devices count the same protocol headers.

---

## Policing and TCP Performance

Policing can significantly affect TCP throughput.

Suppose a TCP flow attempts to send traffic at 20 Mbps through a 10 Mbps policer.

When the token bucket is exhausted, excess packets are dropped.

TCP interprets packet loss as a possible sign of congestion and normally reduces its sending rate.

This can produce a repeating pattern:

1. TCP increases its sending rate.
2. The policer begins dropping packets.
3. TCP reduces its sending rate.
4. Tokens accumulate while the offered traffic rate is lower.
5. TCP increases its sending rate again.

This can reduce average throughput below the configured CIR, especially when:
- Bc is too small for normal traffic bursts.
- Round-trip time is high.
- Multiple TCP flows compete.
- Packet loss causes retransmissions and congestion-window reductions.

Shaping is generally more suitable for controlling outbound TCP traffic when preserving packets is desirable.

However, shaping introduces buffering delay and can still drop packets when its queue fills.

---

## Verification

The primary command for verifying policing is:

```text
R1# show policy-map interface GigabitEthernet1
```

It displays the applied policy, traffic-class matches, configured rate parameters, and available policing statistics.

### Example Output

```text
R1# show policy-map interface GigabitEthernet1
 GigabitEthernet1

  Service-policy input: INGRESS-POLICE

    Class-map: BUSINESS (match-all)
      1000 packets, 1500000 bytes
      Match: dscp af31 (26)
      police:
          cir 10000000 bps, bc 312500 bytes
        conformed 800 packets, 1200000 bytes; actions:
          transmit
        exceeded 200 packets, 300000 bytes; actions:
          drop
```

This illustrative output indicates:
- CIR = 10 Mbps.
- Bc = 312,500 bytes.
- 800 packets conformed.
- 200 packets exceeded and were dropped.

The output format and available statistics depend on the platform.

### Important Counters

| Counter | Meaning |
|---|---|
| Class packets/bytes | Traffic classified into the class |
| Offered rate | Average traffic rate offered to the policy |
| Drop rate | Average rate of traffic dropped |
| Conformed | Packets classified as conforming |
| Exceeded | Packets classified as exceeding |
| Violated | Packets classified as violating, when applicable |

**An exceeded packet is not necessarily dropped.**

For example, if the configured action is `set-dscp-transmit af32`, exceeded packets are remarked and transmitted.

Always examine the action associated with each policing counter.

### Useful Verification Commands

| Command | Purpose |
|---|---|
| `show policy-map` | Display configured policies and policing parameters |
| `show policy-map INGRESS-POLICE` | Inspect a specific policy map |
| `show policy-map interface GigabitEthernet1` | Verify applied policies and policing statistics |
| `show policy-map interface GigabitEthernet1 input` | Verify ingress policing |
| `show policy-map interface GigabitEthernet1 output` | Verify egress policing |
| `show running-config interface GigabitEthernet1` | Confirm policy attachment |
| `show class-map` | Verify classification criteria |

### Hardware Policing Verification

On supported QFP-based IOS XE routers, use:

```text
R1# show platform hardware qfp active feature qos interface GigabitEthernet1
```

This can display the policing rates and burst values programmed into forwarding hardware.

These values may differ slightly from the configured values because of hardware resolution and rounding.

The command is platform-specific and should not be assumed to work on all IOS XE devices.

---

## Common Troubleshooting Issues

### Unexpected Drops Below CIR

If packets are dropped even though the observed average traffic rate is below CIR:
- Check Bc. Short bursts may exceed the token allowance.
- Check packet sizes and overhead accounting.
- Verify whether multiple traffic flows share the same policer.
- Remember that average rates can hide short-term bursts.
- Examine conform and exceed counters rather than relying only on interface utilization.

### Policer Not Dropping Packets

If a policer does not appear to enforce the expected maximum rate:
- Verify that the policy is attached to the correct interface and direction.
- Check that the traffic matches the intended class.
- Verify the configured exceed and violate actions.
- Check the burst allowance.
- Confirm platform support and dataplane enforcement.

A policer using `exceed-action transmit` or `exceed-action set-dscp-transmit` does not enforce a hard forwarding-rate cap.

### Configured CIR vs. Actual Throughput

The observed throughput may differ from CIR because of:
- Token-bucket burst allowance.
- Packet size and accounting overhead.
- TCP congestion control.
- Traffic measurement intervals.
- Hardware rate granularity.

For example, a traffic generator measuring Layer 3 throughput may report a different rate from a switch port measuring physical Ethernet bandwidth.

---

## Key Points

- **Traffic policing** enforces traffic profiles by immediately applying actions to packets.
- Policing uses token buckets and permits bursts when sufficient tokens are available.
- **CIR** controls committed token replenishment.
- **Bc** determines the committed burst allowance.
- Cisco's class-based policing implementation replenishes tokens based on elapsed time when packets arrive.
- A basic **1R2C policer** distinguishes conforming and exceeding traffic.
- Three-color policing adds a violating outcome.
- The default actions of a basic 1R2C policer are **conform = transmit** and **exceed = drop**.
- `set-dscp-transmit` remarks and forwards traffic rather than dropping it.
- Policing can operate on ingress or egress, depending on platform support.
- Separate class policers maintain separate token accounting.
- **Policing does not require interface congestion** to drop traffic.
- **Policing does not provide minimum bandwidth guarantees**.
- Bc defaults and packet overhead accounting depend on the platform.
- `show policy-map interface` is the primary verification command.

---

## References

- [Cisco IOS XE 17.x: QoS Packet Policing](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/qos/b-quality-of-service/m_qos-policing.html)
- [Cisco: Compare Traffic Policing and Traffic Shaping to Limit Bandwidth](https://www.cisco.com/c/en/us/support/docs/quality-of-service-qos/qos-policing/19645-policevsshape.html)
- [Cisco IOS XE 17.x: Class-Based Policing](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/qos/b-quality-of-service/m_qos-plcshp-class-plc-0.html)
- [Cisco IOS XE 17.x: QoS Percentage-Based Policing](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/qos/b-quality-of-service/m_qos-plcshp-pct-plc-0.html)
- [Cisco IOS XE 17.x: Two-Rate Policer](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/qos/b-quality-of-service/m_qos-plcshp-two-rate-plcr-0.html)