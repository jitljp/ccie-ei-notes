# Traffic Shaping

**Traffic shaping** is a QoS mechanism that regulates the rate at which packets are transmitted by buffering excess traffic and delaying its transmission.

Unlike policing, which can immediately drop or remark traffic that exceeds a configured rate, shaping attempts to preserve excess packets by placing them in a queue.

Traffic shaping is primarily used on **outbound (egress) traffic**.

Common applications include:
- Limiting WAN traffic to a service provider's contracted rate.
- Preventing downstream congestion and policing drops.
- Smoothing bursts of traffic.
- Controlling the maximum rate of specific traffic classes.
- Combining aggregate rate limits with class-based queuing through HQoS.

Cisco IOS XE uses the `shape` command within the Modular QoS CLI (MQC) to configure traffic shaping.

---

## Traffic Shaping Operation

A traffic shaper uses two primary components:
- **Token bucket:** Determines when packets are eligible for transmission.
- **Shaping queue:** Buffers packets until they can be transmitted.

```text
               Incoming Traffic
                      |
                      v
               +-------------+
               |  Shaping    |
               |  Queue      |
               +-------------+
                      |
                      v
               +-------------+
               | Token       |
               | Bucket      |
               +-------------+
                      |
                Enough tokens?
                 /         \
               Yes          No
                |            |
                v            v
             Transmit      Remain
                           Queued
```

The process is:

1. Packets arrive at the outbound QoS scheduler.
2. The shaper determines whether sufficient tokens or scheduling credits are available.
3. If sufficient tokens exist, packets are eligible for transmission.
4. Otherwise, the packets remain in the shaping queue.
5. As tokens become available, queued packets are scheduled for transmission.
6. If the queue becomes full, additional packets may be dropped.

**Shaping does not immediately drop packets simply because they exceed the configured rate.** Instead, it delays them.

However, shaping does not guarantee lossless delivery. Excess packets may eventually be dropped if the queue cannot accommodate them.

### Example: 30 Mbps Offered, 20 Mbps Shaped

Suppose R1 receives traffic from a LAN at 30 Mbps but is configured to shape outbound traffic to 20 Mbps.

```text
            LAN
             |
             | 30 Mbps
             v
       +-----------+
       |    R1     |
       |           |
       |  Shaper   |
       |  20 Mbps  |
       +-----------+
             |
             | 20 Mbps
             v
            WAN
```

While traffic is continuously offered at 30 Mbps:
- Approximately 20 Mbps is transmitted.
- Approximately 10 Mbps accumulates in the shaping queue.
- If the offered rate remains above the shaping rate, the queue eventually reaches its limit.
- Additional packets are then dropped.

If the offered rate falls below 20 Mbps, the router can transmit the queued packets and reduce the backlog.

The shaper does not permanently hold excess packets. It delays them until transmission capacity becomes available.

---

## Token Bucket and Shaping

Traffic shaping uses a **token bucket** mechanism to regulate transmission.

The key parameters are:

| Parameter | Description |
|---|---|
| **CIR** | Committed Information Rate, the configured average shaping rate |
| **Bc** | Committed Burst, the transmission allowance associated with one Tc |
| **Be** | Excess Burst, additional burst allowance in classic shaping |
| **Tc** | Committed Time Interval, used to relate CIR and Bc |

In the classic interval-based shaping model, the bucket receives Bc worth of tokens every Tc.

The relationship is:

```text
Bc = CIR × Tc
```

When Bc is expressed in bits, CIR is in bits per second and Tc is in seconds.

Rearranging:

```text
Tc = Bc / CIR

CIR = Bc / Tc
```

### Example: Calculate Bc

Suppose:
- CIR = 10 Mbps.
- Tc = 20 ms.

Calculate Bc:

```text
Bc = CIR × Tc
   = 10,000,000 × 0.020
   = 200,000 bits
   = 25,000 bytes
```

In the classic interval-based model, the shaper receives a transmission allowance equivalent to 200,000 bits every 20 ms.

The packets can be transmitted at the physical interface's line rate, but the shaper regulates when they are eligible for transmission.

This creates an average rate of 10 Mbps over time, assuming sufficient traffic is available.

### Shaping Does Not Necessarily Transmit at a Constant Bit Rate

A 10 Mbps shaper on a 1 Gbps Ethernet interface does not necessarily cause the physical interface to transmit each bit at 10 Mbps.

Instead:
- The interface can transmit packets at its normal physical line rate.
- The shaper controls when packets are released from the queue.
- The resulting average output rate is limited to the configured shaping rate.

For example, a 1,500-byte packet requires approximately 12 microseconds to serialize at 1 Gbps, ignoring Ethernet overhead.

At 10 Mbps, the same 12,000 bits represent 1.2 ms of shaping allowance.

A shaper can therefore transmit the packet quickly and then delay subsequent packets to maintain the configured average rate.

**Shaping regulates packet scheduling, not the physical interface's signaling speed.**

---

## Committed Burst (Bc) and Tc

Bc and Tc affect the burstiness of shaped traffic in classic interval-based implementations.

For a fixed CIR:

| Tc | Bc at 10 Mbps | General behavior |
|---|---:|---|
| 10 ms | 100,000 bits | Smaller, more frequent bursts |
| 20 ms | 200,000 bits | Moderate bursts |
| 50 ms | 500,000 bits | Larger, less frequent bursts |
| 100 ms | 1,000,000 bits | Larger bursts and potentially greater delay variation |

A shorter Tc generally produces smoother traffic because transmission opportunities occur more frequently.

A longer Tc allows larger bursts, but can introduce more delay variation.

### Important IOS XE Platform Difference

The classic Bc/Tc model does not describe the internal scheduling behavior of every modern IOS XE router.

For example, on Cisco ASR 1000 routers, hardware-based scheduling performs frequent scheduling decisions without requiring the same fixed software intervals.

Cisco documents that the optional Bc and Be arguments of `shape average` do not affect the hardware shaper on ASR 1000.

Therefore:
- The Bc/Tc relationship remains important for understanding shaping.
- The effect of manually configuring Bc and Be depends on the platform.
- Do not assume changing Bc will change the actual burst behavior of every IOS XE device.

For modern QFP-based platforms, the configured shaping rate and hardware scheduler behavior are more important than assuming a fixed Tc timer.

See [Token Bucket Fundamentals (CIR, PIR, Bc, Be)](token-bucket.md) for additional details.

---

## Excess Burst (Be)

**Excess Burst (Be)** specifies how much additional transmission allowance is available beyond Bc.

In classic shaping, the maximum accumulated token allowance is:

```text
Maximum accumulated tokens = Bc + Be
```

For example:
- Bc = 100,000 bits.
- Be = 200,000 bits.

The maximum accumulated allowance is:

```text
100,000 + 200,000 = 300,000 bits
```

However, an important distinction is **how the Be allowance becomes available**.

There are two classic shaping modes:

| Mode | How Be becomes available |
|---|---|
| `shape average` | Unused Bc tokens from previous intervals accumulate, up to the Bc + Be limit |
| `shape peak` | The shaper permits Bc + Be bits of transmission during every Tc interval |

Be does not represent a separately replenished token bucket like the peak bucket in two-rate policing.

**The fundamental difference between `shape average` and `shape peak` is whether Be requires previously unused transmission allowance.**

---

## Average Shaping

**Average shaping** limits the long-term transmission rate to the configured CIR.

The command is:

```text
R1(config-pmap-c)# shape average 10000000
```

This configures an average shaping rate of 10 Mbps.

### Token Replenishment with Average Shaping

In classic average shaping:

1. The shaper receives Bc new tokens every Tc.
2. If fewer than Bc tokens are consumed during an interval, unused tokens are carried forward.
3. The accumulated token allowance cannot exceed Bc + Be.
4. The shaper can use these saved tokens to transmit a larger burst later.
5. Once the saved excess tokens are consumed, sustained transmission is limited to CIR.

**Be is not independently replenished every Tc. It represents the maximum additional tokens that can accumulate from unused transmission allowance.**

### Example: Average Shaping with Be

Consider:

```text
R1(config-pmap-c)# shape average 10000000 200000 200000
```

The parameters are:
- CIR = 10 Mbps.
- Bc = 200,000 bits.
- Be = 200,000 bits.
- Tc = 20 ms.

The shaper receives 200,000 new tokens every 20 ms.

Suppose the shaper has been idle long enough to accumulate its maximum allowance:

```text
Accumulated tokens = Bc + Be
                   = 200,000 + 200,000
                   = 400,000 bits
```

Now a continuous stream of traffic arrives, with enough data to consume all available tokens.

| Interval | Tokens available | Maximum transmitted | Tokens remaining |
|---|---:|---:|---:|
| 1 | 400,000 | 400,000 | 0 |
| 2 | 200,000 | 200,000 | 0 |
| 3 | 200,000 | 200,000 | 0 |
| 4 | 200,000 | 200,000 | 0 |

In the first interval, the shaper can transmit 400,000 bits because it has accumulated additional tokens during the idle period.

However, in subsequent intervals, only Bc = 200,000 new tokens become available.

Therefore, the sustained shaping rate is:

```text
200,000 bits / 0.020 seconds = 10 Mbps
```

The initial burst can temporarily exceed CIR, but the long-term rate approaches 10 Mbps.

### When Does Be Accumulate?

Consider the same shaper, but suppose only 50,000 bits are transmitted during an interval.

The unused allowance is:

```text
Unused tokens = 200,000 - 50,000
              = 150,000 bits
```

These unused tokens are retained rather than discarded, provided the maximum capacity of Bc + Be is not exceeded.

During subsequent intervals, additional unused allowance can accumulate until the bucket reaches 400,000 bits.

This saved allowance can be used for larger bursts when traffic increases.

**Average shaping allows Be to accumulate during periods of low traffic, but it does not provide Be worth of additional tokens every Tc.**

---

## Peak Shaping

**Peak shaping** permits a higher sustained shaping rate by making the Be allowance available during every Tc interval.

The command is:

```text
R1(config-pmap-c)# shape peak 10000000
```

In Cisco's classic shaping model, the effective peak shaping rate depends on CIR, Bc, and Be.

### Token Replenishment with Peak Shaping

Unlike average shaping, classic peak shaping does not require a period of low traffic before the shaper can use Be.

During every Tc interval, the shaper can transmit:

```text
Bc + Be
```

For example:
- Bc = 200,000 bits.
- Be = 200,000 bits.
- Tc = 20 ms.

Each interval permits:

```text
200,000 + 200,000 = 400,000 bits
```

Even if all 400,000 bits were transmitted during the previous interval, the same allowance is available in the next interval.

### Example: Peak Shaping with Be

Configure:

```text
R1(config-pmap-c)# shape peak 10000000 200000 200000
```

The parameters are identical to the previous average-shaping example:
- CIR = 10 Mbps.
- Bc = 200,000 bits.
- Be = 200,000 bits.
- Tc = 20 ms.

Assume sufficient traffic is continuously available.

| Interval | Transmission allowance | Maximum transmitted |
|---|---:|---:|
| 1 | 400,000 bits | 400,000 bits |
| 2 | 400,000 bits | 400,000 bits |
| 3 | 400,000 bits | 400,000 bits |
| 4 | 400,000 bits | 400,000 bits |

The shaper can sustain 400,000 bits every 20 ms:

```text
400,000 bits / 0.020 seconds = 20 Mbps
```

Therefore, even though the configured CIR is 10 Mbps, the effective peak shaping rate is 20 Mbps.

### Peak Shaping Formula

The classic peak shaping rate is calculated as:

```text
Peak rate = CIR × (1 + Be / Bc)
```

For the example:

```text
Peak rate = 10,000,000 × (1 + 200,000 / 200,000)
          = 10,000,000 × 2
          = 20 Mbps
```

If Be equals Bc, the calculated peak shaping rate is twice CIR.

If Be is half of Bc, the calculated peak shaping rate is 1.5 times CIR.

If Be is zero, peak shaping provides no additional rate above CIR.

### Average vs. Peak Shaping Comparison

Consider these two commands:

```text
R1(config-pmap-c)# shape average 10000000 200000 200000
R1(config-pmap-c)# shape peak 10000000 200000 200000
```

| Characteristic | `shape average` | `shape peak` |
|---|---|---|
| CIR | 10 Mbps | 10 Mbps |
| Bc | 200,000 bits | 200,000 bits |
| Be | 200,000 bits | 200,000 bits |
| Tc | 20 ms | 20 ms |
| New transmission allowance per Tc | Bc | Bc + Be |
| Be requires previous idle/unused allowance | Yes | No |
| Maximum accumulated burst | Bc + Be | Bc + Be |
| Sustained shaping rate | 10 Mbps | 20 Mbps |

**Average shaping uses Be as saved transmission allowance. Peak shaping makes Bc + Be available during every interval.**

### When Is Peak Shaping Useful?

Peak shaping can be useful when a service provider permits sustained traffic above the contracted CIR.

For example:
- Provider CIR = 10 Mbps.
- Provider permits traffic up to 20 Mbps when capacity is available.

Peak shaping can take advantage of the additional bandwidth.

However, traffic transmitted above the contracted CIR may be discarded when the provider network becomes congested.

If traffic must strictly conform to the provider's committed rate, average shaping is generally preferable.

---

## Peak Shaping on Modern IOS XE

Cisco documents an important difference on ASR 1000 routers.

On these platforms, `shape peak` does not implement the classic Bc + Be transmission allowance as a separate periodic mechanism.

Instead, the router:
1. Reads the configured CIR, Bc, and Be.
2. Calculates the effective target shaping rate.
3. Programs that rate into the hardware scheduler.

Cisco states that `shape peak` provides no additional functionality beyond `shape average` on ASR 1000.

For example, the classic configuration:

```text
R1(config-pmap-c)# shape peak 10000000 200000 200000
```

Calculates an effective shaping rate of 20 Mbps.

On ASR 1000, Cisco recommends configuring that rate directly:

```text
R1(config-pmap-c)# shape average 20000000
```

For CCIE EI:
- Understand the classic distinction between average and peak shaping.
- Know the peak-rate formula.
- Prioritize `shape average` for practical IOS XE configurations.
- Remember that modern hardware schedulers may not implement the classic Bc/Be interval mechanism.

---

## Configuring Class-Based Shaping with MQC

Cisco IOS XE uses class-based shaping to regulate traffic through policy maps.

The basic configuration steps are:
1. Create a policy map.
2. Configure `shape average` under the desired class.
3. Apply the policy outbound.

### Example: Shape All WAN Traffic to 20 Mbps

Suppose R1 has a 100 Mbps WAN interface, but the service provider contract permits only 20 Mbps.

```text
                  R1
              +--------+
   LAN ------ |        | -------- Provider
              +--------+
                        |
                        | Gi2
                        | 100 Mbps physical
                        | 20 Mbps contracted
```

Configure:

```text
R1(config)# policy-map WAN-SHAPE
R1(config-pmap)# class class-default
R1(config-pmap-c)# shape average 20000000
R1(config-pmap-c)# exit
R1(config)# interface GigabitEthernet2
R1(config-if)# service-policy output WAN-SHAPE
```

This configuration:
- Matches all eligible outbound traffic through `class-default`.
- Shapes aggregate traffic to 20 Mbps.
- Buffers packets that cannot immediately be transmitted within the shaping rate.
- Transmits queued packets as shaping capacity becomes available.

The physical interface still operates at 100 Mbps.

The shaping policy limits its scheduled traffic rate to approximately 20 Mbps.

### Shaping Is an Egress Feature

Unlike policing, shaping is fundamentally an outbound scheduling mechanism.

Therefore, the policy is applied using:

```text
R1(config-if)# service-policy output WAN-SHAPE
```

Not:

```text
R1(config-if)# service-policy input WAN-SHAPE
```

Ingress traffic cannot ordinarily be shaped at the receiving interface because shaping requires a queue from which packets can be scheduled for later transmission.

If incoming traffic must be controlled, ingress policing is generally used instead.

---

## Shaping Individual Traffic Classes

Shaping does not have to apply to all traffic on an interface.

A policy can shape a particular class independently of other traffic.

### Example: Shape Backup Traffic

Suppose R1 should limit traffic marked CS1 to 5 Mbps while leaving other traffic unaffected by the shaper.

Configure:

```text
R1(config)# class-map match-all BACKUP
R1(config-cmap)# match dscp cs1
R1(config-cmap)# exit
R1(config)# policy-map SHAPE-BACKUP
R1(config-pmap)# class BACKUP
R1(config-pmap-c)# shape average 5000000
R1(config-pmap-c)# exit
R1(config)# interface GigabitEthernet2
R1(config-if)# service-policy output SHAPE-BACKUP
```

The behavior is:

| Traffic | Shaping treatment |
|---|---|
| CS1 | Shaped to 5 Mbps |
| Other traffic | Not rate-limited by this shaper |

Traffic not matching BACKUP is handled by `class-default`.

Because no shaping action is configured in `class-default`, it is not subject to the 5 Mbps limit.

The BACKUP class has its own shaping rate.

This does not necessarily constrain the aggregate traffic rate across the interface.

For that, an aggregate shaper is needed.

---

## Percentage-Based Shaping

Cisco IOS XE supports configuring shaping as a percentage of the relevant bandwidth.

The syntax is:

```text
R1(config-pmap-c)# shape average percent <percentage>
```

For example:

```text
R1(config-pmap-c)# shape average percent 25
```

This configures average shaping to 25% of the applicable bandwidth reference.

If the reference bandwidth is 100 Mbps:

```text
100 Mbps × 25% = 25 Mbps
```

The resulting shaping rate is 25 Mbps.

### Bandwidth Reference

The bandwidth used for the percentage calculation depends on the policy's attachment point and hierarchy.

For a top-level interface policy, it is typically based on the interface's QoS bandwidth reference.

For a child policy, the relevant reference can instead be derived from a parent shaping rate.

For example:

| Bandwidth reference | Shaping percentage | Result |
|---|---:|---:|
| 100 Mbps | 25% | 25 Mbps |
| 1 Gbps | 25% | 250 Mbps |
| 20 Mbps parent shaper | 25% | 5 Mbps |

The percentage is not necessarily calculated from the physical interface's negotiated speed.

Percentage-based shaping is useful when reusing a QoS policy across interfaces with different bandwidths.

### Percentage-Based Burst Configuration

On supporting platforms, Bc and Be can also be expressed in milliseconds with percentage-based shaping.

For example:

```text
R1(config-pmap-c)# shape average percent 25 bc 20 ms be 20 ms
```

This configures the rate to 25% of the relevant bandwidth and requests burst allowances corresponding to 20 ms at that rate.

Again, the actual handling of these burst parameters depends on the platform.

---

## Shaping and Queuing

Shaping introduces an important QoS design consideration: **the shaping rate creates a logical congestion point**.

Suppose:
- Physical interface speed = 100 Mbps.
- Configured shaping rate = 20 Mbps.
- Offered traffic = 30 Mbps.

Even though the physical interface has 70 Mbps of unused capacity, the shaping queue experiences congestion because the offered rate exceeds the configured 20 Mbps limit.

This is why shaping is commonly combined with congestion management.

The shaper determines **when traffic can be transmitted**.

Queuing and scheduling determine **which traffic is transmitted first**.

### Shaping with CBWFQ and LLQ

Consider a WAN link carrying:
- Voice traffic marked EF.
- Business traffic marked AF31.
- Best-effort traffic.

The objective is to:
1. Shape aggregate outbound traffic to 20 Mbps.
2. Prioritize voice traffic.
3. Allocate bandwidth to business applications.
4. Allow best-effort traffic to use remaining capacity.

A common approach is to configure a hierarchical policy.

### Example: Parent Shaping and Child Queuing

Create the class maps:

```text
R1(config)# class-map match-all VOICE
R1(config-cmap)# match dscp ef
R1(config-cmap)# exit
R1(config)# class-map match-all BUSINESS
R1(config-cmap)# match dscp af31
```

Create the child queuing policy:

```text
R1(config)# policy-map CHILD-QOS
R1(config-pmap)# class VOICE
R1(config-pmap-c)# priority percent 10
R1(config-pmap-c)# exit
R1(config-pmap)# class BUSINESS
R1(config-pmap-c)# bandwidth remaining percent 50
```

Create the parent shaping policy:

```text
R1(config)# policy-map PARENT-SHAPE
R1(config-pmap)# class class-default
R1(config-pmap-c)# shape average 20000000
R1(config-pmap-c)# service-policy CHILD-QOS
```

Apply the parent policy outbound:

```text
R1(config)# interface GigabitEthernet2
R1(config-if)# service-policy output PARENT-SHAPE
```

The resulting policy hierarchy is:

```text
               PARENT-SHAPE
               ------------
             Shape to 20 Mbps
                     |
                     v
                 CHILD-QOS
               /     |      \
              /      |       \
           VOICE   BUSINESS  DEFAULT
          Priority  Bandwidth Remaining
```

The parent policy limits aggregate outbound traffic to 20 Mbps.

The child policy controls how the available shaping capacity is distributed among traffic classes.

For example, the voice priority rate is calculated relative to the parent's applicable bandwidth reference on supporting platforms.

The `bandwidth remaining percent` command distributes the remaining bandwidth according to scheduling weights. It does not necessarily guarantee that BUSINESS always receives exactly 50% of the total shaped rate.

This is a common WAN QoS design, but the exact scheduler behavior and support for hierarchical configurations depend on the platform.

See [4.4.g Hierarchical QoS (HQoS)](../hqos/overview.md) for detailed operation and configuration.

---

## Shaping, Delay, and Packet Loss

Traffic shaping attempts to avoid immediate packet loss by buffering excess packets.

However, buffering introduces delay.

### Queueing Delay

Suppose:
- Shaping rate = 20 Mbps.
- The shaping queue contains 1,000,000 bytes.
- No higher-priority traffic competes with the queue.

The approximate time required to drain the backlog is:

```text
Delay = Queue size (bits) / Shaping rate (bps)

      = (1,000,000 × 8) / 20,000,000
      = 0.4 seconds
      = 400 ms
```

This assumes no additional packets arrive and ignores other scheduling and encapsulation effects.

A large shaping queue can therefore introduce substantial latency.

### Bufferbloat

**Bufferbloat** occurs when excessive buffering produces unnecessarily high packet delays.

For example, a deep shaping queue might prevent immediate packet drops but cause hundreds of milliseconds of additional delay.

This can negatively affect:
- Voice and video.
- Interactive applications.
- Gaming and remote desktop traffic.
- TCP performance.

A shaper therefore should not simply use the largest possible queue.

Queue limits, appropriate class-based scheduling, and congestion-avoidance mechanisms are important.

### Shaping Can Still Drop Packets

If offered traffic exceeds the shaping rate for long enough, the queue eventually fills.

For example:

| Parameter | Value |
|---|---:|
| Offered traffic | 30 Mbps |
| Shaping rate | 20 Mbps |
| Excess arrival rate | 10 Mbps |
| Queue capacity | 2.5 MB |

Assuming an initially empty queue and sustained rates, the queue fills in approximately:

```text
Excess rate = 10,000,000 bps
            = 1,250,000 bytes/second

Time = 2,500,000 / 1,250,000
     = 2 seconds
```

Once the queue is full, further excess traffic must be dropped or otherwise managed.

**Shaping delays excess traffic; it does not create additional bandwidth.**

---

## Shaping and TCP Traffic

Traffic shaping is often preferred over policing for outbound TCP traffic.

With policing:
- Excess packets may be immediately dropped.
- TCP can interpret losses as congestion.
- TCP reduces its sending rate and retransmits packets.

With shaping:
- Excess packets can be buffered rather than dropped.
- TCP may receive fewer packet losses.
- Traffic is transmitted at a smoother rate.

However, shaping can also degrade TCP performance if:
- Queues become excessively deep.
- Queueing delay increases significantly.
- Sustained overload causes drops.
- The shaper's accounted rate exceeds the actual downstream bottleneck capacity.

A well-designed shaper should match the real available bandwidth without introducing unnecessarily large queues.

---

## Choosing the Shaping Rate

A common deployment uses a physical WAN interface whose capacity exceeds the provider's contracted rate.

For example:

```text
Physical interface: 1 Gbps
Provider CIR:       100 Mbps
```

The enterprise router should generally shape traffic close to the contracted rate:

```text
R1(config-pmap-c)# shape average 100000000
```

However, the exact rate required to avoid provider policing depends on how the provider measures traffic.

### Encapsulation Overhead

Consider a provider that enforces its bandwidth limit using Ethernet frames, while the local shaper accounts for a different packet length.

A shaper configured for 100 Mbps might permit more than 100 Mbps of actual on-wire traffic after Ethernet overhead is included.

This is particularly important for small packets.

Some IOS XE platforms support overhead accounting with the `account` keyword.

For example, Cisco documents the following configuration for accounting for additional Ethernet overhead on supported platforms:

```text
R1(config-pmap-c)# shape average 50000000 account user-defined 24
```

This adds 24 bytes to the scheduler's accounted packet length.

The extra bytes represent Ethernet FCS and additional on-wire overhead in Cisco's example.

Such accounting can help match the local shaping rate to a provider's measurement method.

The supported overhead-accounting options depend on the platform.

If exact overhead accounting is unavailable, configuring a shaping rate somewhat below the provider's contracted rate may help avoid unexpected provider drops.

The appropriate margin depends on the encapsulation and traffic characteristics.

---

## Verification

The primary command for verifying traffic shaping is:

```text
R1# show policy-map interface GigabitEthernet2
```

This displays the applied policy, shaping rate, queue statistics, and available traffic-rate information.

### Example Output

An IOS XE router may display information similar to:

```text
R1# show policy-map interface GigabitEthernet2

 GigabitEthernet2

  Service-policy output: WAN-SHAPE

    Class-map: class-default (match-any)
      10000 packets, 15000000 bytes
      Match: any
      Queueing
      queue limit 64 packets
      (queue depth/total drops/no-buffer drops) 10/0/0
      shape (average) cir 20000000, bc 80000, be 80000
      target shape rate 20000000
```

This illustrative output shows:
- Shaping mode = average.
- Configured CIR = 20 Mbps.
- Bc and Be values reported by the platform.
- Target shaping rate = 20 Mbps.
- Current queue depth = 10 packets.
- No queue drops reported.

The exact output and burst values vary by IOS XE release and hardware platform.

### Important Statistics

| Statistic | Meaning |
|---|---|
| Shaping rate | Configured or calculated target transmission rate |
| Offered rate | Traffic rate entering the QoS class |
| Transmitted rate | Traffic rate successfully transmitted |
| Queue depth | Amount of traffic currently buffered |
| Total drops | Packets discarded by the queue |
| Queue limit | Maximum permitted queue occupancy |
| No-buffer drops | Drops caused by unavailable buffering resources, when reported |

If the offered rate consistently exceeds the shaping rate, queue depth and drop counters may increase.

### Useful Commands

| Command | Purpose |
|---|---|
| `show policy-map` | Display configured shaping policies |
| `show policy-map WAN-SHAPE` | Inspect a specific policy |
| `show policy-map interface GigabitEthernet2` | Verify shaping rate and queue statistics |
| `show policy-map interface GigabitEthernet2 output` | Display egress QoS policy |
| `show running-config interface GigabitEthernet2` | Verify policy attachment |
| `show interfaces GigabitEthernet2` | Check interface traffic rates and drops |

On supported QFP-based IOS XE routers, platform-specific commands can provide additional hardware QoS information.

For example:

```text
R1# show platform hardware qfp active feature qos interface GigabitEthernet2
```

Supported syntax and available statistics vary between platforms.

---

## Common Troubleshooting Issues

### Output Rate Exceeds the Shaping Rate

Possible causes include:
- Measurement over a very short interval that includes a permitted burst.
- A percentage-based shaping rate calculated using an unexpected bandwidth reference.
- Differences between Layer 3, Layer 2, and physical on-wire accounting.
- Platform-specific handling of Bc and Be.
- Shaping applied to only one traffic class rather than all outbound traffic.

Check the configured rate and the actual policy attachment before assuming the shaper is malfunctioning.

### Excessive Queueing Delay

Possible causes include:
- The shaping rate is significantly below the offered traffic rate.
- Queue limits are too large.
- Multiple applications share the same shaping queue.
- No child queuing policy prioritizes delay-sensitive traffic.

Consider using hierarchical QoS to separate voice, business, and best-effort traffic.

### Unexpected Packet Drops

Possible causes include:
- Sustained traffic above the shaping rate.
- Queue exhaustion.
- Hardware buffer limitations.
- An additional downstream policer.
- Incorrect overhead accounting.

Examine queue drop counters and compare offered and transmitted rates.

### Shaper Does Not Appear Active

If the offered traffic rate is lower than the configured shaping rate, the shaper may not need to delay packets.

For example:
- Shaping rate = 20 Mbps.
- Offered traffic = 5 Mbps.

The shaper can transmit the traffic normally because the offered traffic does not exceed the configured rate.

This is expected behavior.

---

## Shaping vs. Policing Summary

| Characteristic | Shaping | Policing |
|---|---|---|
| Token bucket | Yes | Yes |
| Queue for excess traffic | Yes | No |
| Excess traffic | Delayed | Immediately transmitted, remarked, or dropped according to policy |
| Added delay | Yes | No intentional buffering delay |
| Drops possible | Yes, when queues fill | Yes, according to policing actions |
| Direction | Egress | Ingress or egress |
| Primary command | `shape` | `police` |
| Common deployment | WAN rate matching | Traffic contract enforcement |
| Minimum bandwidth guarantee | No | No |

**Shaping is generally preferred when excess traffic should be delayed rather than immediately discarded.**

**Policing is generally preferred when an immediate enforcement action is required.**

---

## Key Points

- **Traffic shaping** regulates outbound transmission using a token bucket and a queue.
- Packets that cannot immediately be transmitted are buffered until scheduling capacity becomes available.
- **Shaping is an egress feature**.
- In classic average shaping, Bc tokens are replenished every Tc, while Be represents accumulated unused allowance.
- In classic peak shaping, Bc + Be is available during every Tc, allowing a higher sustained shaping rate.
- The classic peak-rate formula is `CIR × (1 + Be/Bc)`.
- `shape average` limits the long-term average transmission rate to the configured CIR.
- `shape peak` permits a higher rate determined by Bc and Be in the classic shaping model.
- In classic interval-based shaping, `Bc = CIR × Tc`.
- **Bc and Be are expressed in bits** when using the positional `shape average` and `shape peak` burst arguments.
- Modern IOS XE hardware schedulers may ignore manually configured Bc and Be parameters.
- **Shaping does not guarantee minimum bandwidth**. It imposes a maximum scheduling rate.
- Shaping does not alter the physical interface's line rate.
- Shaping can create a logical congestion point even when the physical interface has unused bandwidth.
- Shaping is commonly combined with CBWFQ and LLQ through **hierarchical QoS**.
- Large shaping queues can introduce excessive delay and bufferbloat.
- Shaping queues can still drop packets when sustained offered traffic exceeds the shaping rate.
- Percentage-based shaping uses the relevant bandwidth reference, which is not necessarily the physical interface speed.
- `show policy-map interface` is the primary verification command.

---

## References

- [Cisco IOS XE 17.x: Regulating Packet Flow Using Traffic Shaping](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/qos/b-quality-of-service/m_qos-regpkt-trfc-0.html)
- [Cisco IOS XE 17.x: Class-Based Traffic Shaping](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/qos/b-quality-of-service/m_qos-regpkt-cls-0.html)
- [Cisco IOS XE 17.x: QoS Percentage-Based Shaping](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/qos/b-quality-of-service/m_qos-plcshp-pct-shp.html)
- [Cisco IOS XE 17.x: QoS Scheduling](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/qos/b-quality-of-service/m_qos-scheduling.html)
- [Cisco: Compare Traffic Policing and Traffic Shaping to Limit Bandwidth](https://www.cisco.com/c/en/us/support/docs/quality-of-service-qos/qos-policing/19645-policevsshape.html)