# 4.4.e Policing and Shaping

**Traffic policing** and **traffic shaping** are QoS mechanisms used to regulate the rate at which traffic enters or leaves a network.

Both use **token bucket algorithms** to measure traffic against configured rate and burst parameters.

However, they handle traffic exceeding the configured rate differently:
- **Policing** immediately applies an action, typically dropping or remarking excess packets.
- **Shaping** buffers excess packets and delays their transmission until they can be sent within the configured rate.

These mechanisms are commonly deployed at network boundaries and WAN interfaces to enforce bandwidth limits and prevent downstream congestion.

---

## Policing vs. Shaping

| Characteristic | Policing | Shaping |
|---|---|---|
| Primary purpose | Enforce a traffic rate | Smooth traffic to a specified rate |
| Excess traffic | Dropped, remarked, or transmitted according to policy | Buffered and transmitted later |
| Buffering | No | Yes |
| Additional delay | Does not intentionally delay packets | Introduces queuing delay |
| Packet loss | Can immediately drop nonconforming packets | Drops packets if the shaping queue becomes full |
| Traffic direction | Ingress or egress, depending on platform | Primarily egress |
| Typical deployment | Network edge, traffic contracts, rate enforcement | WAN egress, provider rate matching |
| Rate measurement | Token bucket | Token bucket |

For example, suppose a router connects to a service provider:

```text
       Enterprise                    Service Provider
          LAN                             WAN
           |                               |
           |  1 Gbps             100 Mbps  |
           |                               |
          SW1 -------- R1 ---------------- SP
                         |
                         | CIR = 50 Mbps
                         |
                    Shape outbound
                    traffic to 50 Mbps
```

Although R1's WAN interface operates at 100 Mbps, the service provider only permits 50 Mbps under the service contract.

R1 can use shaping to limit outbound traffic to approximately 50 Mbps over time.

Without shaping, R1 might transmit at a rate approaching 100 Mbps, causing congestion or policing drops within the provider network.

Alternatively, the provider might configure an ingress policer that immediately drops traffic exceeding the contracted rate, subject to its configured burst allowance.

**Shaping on the enterprise router can reduce the likelihood of provider policing drops by controlling traffic before it reaches the provider.**

---

## Token Bucket Fundamentals

Both policing and shaping use a **token bucket** mechanism to determine how much traffic may be transmitted or accepted.

A token bucket is a logical mechanism that maintains tokens representing permission to process a certain quantity of traffic.

The basic process is:

1. Tokens accumulate at a configured rate.
2. The bucket can hold only a limited number of tokens.
3. Processing a packet consumes tokens proportional to its size.
4. When sufficient tokens are available, the packet can conform to the configured traffic profile.
5. When insufficient tokens are available, the policer or shaper takes its respective action.

The bucket's capacity permits short bursts above the configured rate without necessarily treating those bursts as nonconforming.

### Token Bucket Parameters

| Parameter | Name | Description |
|---|---|---|
| **CIR** | Committed Information Rate | The rate at which committed tokens are replenished |
| **PIR** | Peak Information Rate | An additional rate used by two-rate policers |
| **Bc** | Committed Burst | The capacity associated with committed traffic |
| **Be** | Excess Burst | Additional burst allowance or peak-bucket capacity, depending on the algorithm |
| **Tc** | Committed Time Interval | A time interval commonly used to describe shaping behavior and burst calculations |

The rate parameters are specified in **bits per second (bps)**.

Burst parameters have different configuration units depending on the feature and platform. For example, IOS XE policers commonly express `bc` and `be` in bytes, while classic shaping calculations express burst sizes in bits.

### Committed Burst and Time Interval

The relationship between CIR, Bc, and Tc is:

`Bc (bits) = CIR (bps) × Tc (seconds)`

Or, when Bc is expressed in bytes:

`Bc (bytes) = CIR × Tc / 8`

For example, with:
- CIR = 10 Mbps
- Tc = 20 ms

The committed burst is:

```text
Bc = (10,000,000 × 0.020) / 8
   = 25,000 bytes
```

This corresponds to 20 milliseconds of traffic at the configured CIR.

A larger burst allowance permits more traffic to arrive or be transmitted in a short period without exhausting the available tokens.

**The CIR does not prohibit all transmission above that rate.** A token bucket can permit short bursts above the CIR when sufficient tokens have accumulated.

For detailed token bucket operation, see [Token Bucket Fundamentals (CIR, PIR, Bc, Be)](token-bucket.md).

---

## Traffic Policing

**Traffic policing** measures traffic against a configured traffic profile and immediately applies actions to packets according to whether they conform to that profile.

A policer does not buffer packets to delay their transmission.

For example:

```text
                  Traffic Policer
                         |
                         v
                  +--------------+
 Incoming ------> | Token Bucket |
 Traffic          +--------------+
                         |
                 +-------+-------+
                 |               |
              Conform          Exceed
                 |               |
                 v               v
              Transmit          Drop
```

If enough tokens are available to cover the packet's size, the packet conforms and the configured conform action is performed.

Otherwise, the packet is classified as exceeding or violating, depending on the policing algorithm.

The policer can perform several actions:

| Action | Behavior |
|---|---|
| `transmit` | Forward the packet |
| `drop` | Discard the packet |
| `set-dscp-transmit` | Change the DSCP value and forward the packet |
| `set-prec-transmit` | Change the IP precedence and forward the packet |

Available actions depend on the platform and policing configuration.

### Policing with MQC

Cisco IOS XE uses the `police` command within a policy-map class.

For example, configure R1 to police AF31 traffic to 5 Mbps on ingress:

```text
R1(config)# class-map match-all BUSINESS
R1(config-cmap)# match dscp af31

R1(config)# policy-map INGRESS-POLICE
R1(config-pmap)# class BUSINESS
R1(config-pmap-c)# police cir 5000000 conform-action transmit exceed-action drop

R1(config)# interface GigabitEthernet1
R1(config-if)# service-policy input INGRESS-POLICE
```

The policy:
- Classifies packets marked AF31.
- Uses a CIR of 5 Mbps.
- Transmits conforming packets.
- Drops exceeding packets.

The policer permits bursts according to its token bucket capacity, so individual packets are not automatically dropped simply because the instantaneous traffic rate exceeds 5 Mbps.

The actual burst allowance depends on the configured or platform-derived burst parameters.

Policing can also **remark excess traffic** instead of dropping it.

For example:

```text
R1(config-pmap-c)# police cir 5000000 conform-action transmit exceed-action set-dscp-transmit af32
```

Conforming AF31 traffic retains its marking, while exceeding traffic is forwarded with DSCP AF32.

A downstream device implementing AF PHBs can then treat AF32 packets with a higher drop precedence than AF31 packets during congestion.

See [Traffic Policing](policing.md) for detailed operation, configuration options, and policing actions.

---

## Single-Rate and Two-Rate Policing

Cisco IOS XE supports several policing algorithms that differ in their rates, token buckets, and packet classification outcomes.

### Single-Rate, Two-Color (1R2C)

A **single-rate, two-color policer** uses:
- One rate: CIR.
- One token bucket with capacity Bc.
- Two packet outcomes: conform and exceed.

| Outcome | Condition |
|---|---|
| Conform | Sufficient committed tokens exist for the packet |
| Exceed | Insufficient committed tokens exist |

This is the simplest policing model.

### Single-Rate, Three-Color (1R3C)

A **single-rate, three-color policer** uses:
- One rate: CIR.
- Two token buckets: Bc and Be.
- Three packet outcomes: conform, exceed, and violate.

The excess bucket provides additional tolerance for bursts beyond the committed allowance.

In Cisco's single-rate three-color model, unused committed tokens can replenish the excess bucket when the committed bucket is full.

### Two-Rate, Three-Color (2R3C)

A **two-rate, three-color policer** uses:
- Two rates: CIR and PIR.
- Two token buckets, replenished independently.
- Three packet outcomes: conform, exceed, and violate.

| Outcome | General interpretation |
|---|---|
| Conform | Traffic is within the committed profile |
| Exceed | Traffic exceeds the committed profile but remains within the peak profile |
| Violate | Traffic exceeds the peak profile |

For sustained traffic, CIR and PIR establish the approximate boundaries between the three outcomes.

However, **the actual packet classification depends on available tokens and packet size**, not solely on an instantaneous rate comparison.

For example, a two-rate policer might use:

```text
R1(config-pmap-c)# police cir 5000000 bc 156250 pir 8000000 be 250000 conform-action transmit exceed-action set-dscp-transmit af32 violate-action drop
```

This configures:
- CIR = 5 Mbps.
- PIR = 8 Mbps.
- Bc = 156,250 bytes.
- Be = 250,000 bytes.
- Conforming traffic is transmitted.
- Exceeding traffic is remarked to AF32 and transmitted.
- Violating traffic is dropped.

The burst capacities permit short-term traffic variations around the configured rates.

See [Single-Rate and Two-Rate Policing](single-two-rate.md) for detailed token bucket behavior, algorithm comparisons, and configuration.

---

## Traffic Shaping

**Traffic shaping** controls the rate at which traffic is transmitted by buffering packets that cannot immediately be sent within the configured traffic profile.

Unlike policing, shaping attempts to preserve excess packets by delaying them rather than immediately dropping or remarking them.

```text
                   Traffic Shaper
                         |
                         v
 Incoming         +-------------+
 Traffic -------> | Shaping     |
                  | Queue       |
                  +-------------+
                         |
                         v
                  +-------------+
                  | Token Bucket|
                  +-------------+
                         |
                         v
                  Transmitted
                    Traffic
```

When a packet is ready for transmission:

1. The shaper determines whether sufficient tokens are available.
2. If sufficient tokens exist, the packet can be transmitted.
3. Otherwise, the packet remains queued until sufficient tokens become available.
4. If the shaping queue reaches its limit, additional packets may be dropped.

Shaping therefore trades **additional delay and potentially jitter** for a smoother transmission rate and fewer immediate rate-enforcement drops.

### Average and Peak Shaping

Cisco IOS XE supports shaping modes such as:

| Mode | Description |
|---|---|
| `shape average` | Limits the long-term transmission rate to the configured shaping rate, with permitted bursts |
| `shape peak` | On supporting implementations, permits additional transmission beyond the committed rate using excess-burst allowances |

Average shaping is the normal choice when matching an enterprise WAN connection to a provider's contracted rate.

The exact behavior and availability of peak shaping and configurable burst parameters depend on the platform. Some IOS XE hardware schedulers do not use manually configured Bc and Be values in the same way as classic software shaping.

### Shaping with MQC

Suppose R1 has a 100 Mbps physical WAN interface, but its service provider contract permits only 20 Mbps.

Configure an outbound shaping policy:

```text
R1(config)# policy-map WAN-SHAPE
R1(config-pmap)# class class-default
R1(config-pmap-c)# shape average 20000000

R1(config)# interface GigabitEthernet1
R1(config-if)# service-policy output WAN-SHAPE
```

The policy shapes all outbound traffic to an average rate of 20 Mbps.

If traffic arrives at the egress scheduler faster than the configured shaping rate, packets can accumulate in the shaping queue.

The shaping rate is not a guaranteed minimum bandwidth allocation. It is an upper bound on the class's long-term scheduled transmission rate.

For more information, see [Traffic Shaping](shaping.md).

---

## Combining Policing and Shaping

Policing and shaping can be deployed together to achieve different objectives.

A common WAN design uses:
- **Ingress policing** to enforce which traffic is permitted into the network at a particular rate.
- **Egress shaping** to match outbound traffic to the provider's contracted rate.
- **Queuing** within the shaped traffic to prioritize delay-sensitive applications.

For example:

```text
        Enterprise                       Provider
           LAN                             WAN
            |                               |
            v                               v
       +----------+                    +----------+
       |    R1    |------------------->|    R2    |
       +----------+                    +----------+
            |                               |
       Egress Shape                 Ingress Police
       (WAN rate)                  (Contract limit)


```

### Shaping with Queuing

Shaping becomes especially useful when combined with congestion management.

Consider two traffic classes sharing a shaped WAN connection:

| Traffic class | Desired treatment |
|---|---|
| Voice | Low-latency priority queue |
| Data | Bandwidth allocation |
| Total traffic | Shaped to WAN contract rate |

A hierarchical QoS policy can implement this design using:
- A **parent policy** to shape the aggregate traffic rate.
- A **child policy** to allocate bandwidth and prioritize individual classes.

This allows traffic to be scheduled intelligently while the aggregate output remains constrained by the parent shaper.

Detailed configurations are covered in [4.4.g Hierarchical QoS (HQoS)](../hqos/overview.md).

---

## Verification and Troubleshooting

The primary command for verifying policing and shaping policies is:

```text
R1# show policy-map interface GigabitEthernet1
```

This displays the policies attached to the interface, including their configured rates and available statistics.

For policing, examine:
- Configured CIR and PIR.
- Configured Bc and Be.
- Conformed packet and byte counters.
- Exceeded packet and byte counters.
- Violated packet and byte counters, when applicable.
- Actions assigned to each category.
- Packet drop statistics.

For shaping, examine:
- Configured shaping rate.
- Offered and transmitted rates, when reported.
- Queue depth and packet counts.
- Queue drops and other available statistics.

### Useful Commands

| Command | Purpose |
|---|---|
| `show policy-map` | Display configured QoS policy maps |
| `show policy-map interface GigabitEthernet1` | Display applied QoS policies and statistics |
| `show policy-map interface GigabitEthernet1 input` | Verify ingress policies |
| `show policy-map interface GigabitEthernet1 output` | Verify egress policies |
| `show running-config interface GigabitEthernet1` | Verify service-policy attachments |
| `show interfaces GigabitEthernet1` | Check interface bandwidth, traffic rates, and drops |

The displayed counters vary by hardware platform and IOS XE release.

### Common Troubleshooting Considerations

**Unexpected policing drops**
- Verify CIR, PIR, Bc, and Be.
- Check whether the traffic profile allows enough burst tolerance.
- Verify the traffic matching the policer.
- Consider differences between measured traffic and the traffic size accounted for by the policer.

**Unexpected shaping drops**
- Check whether the shaping queue is reaching its limit.
- Verify the shaping rate against the actual available bandwidth.
- Examine traffic bursts and application behavior.
- Consider whether additional queuing or bandwidth allocation is needed.

**Shaping does not appear to work**
- Verify that the policy is applied to the correct egress interface.
- Confirm that traffic matches the expected class.
- Check whether sufficient traffic is being offered to activate shaping.
- Verify platform support and the available hardware QoS statistics.

See [Policing and Shaping Configuration](configuration.md) for additional configuration and verification examples.

---

## Key Points

- **Policing** enforces traffic profiles by immediately dropping, remarking, or transmitting packets according to configured actions.
- **Shaping** regulates transmission by buffering and delaying excess packets.
- Both mechanisms use **token buckets** to control rates and permit bounded bursts.
- **CIR** determines the committed token replenishment rate.
- **PIR** provides an additional rate threshold for two-rate policing.
- **Bc** and **Be** determine burst allowances, with their precise roles depending on the algorithm.
- **Single-rate, two-color policing** distinguishes conforming and exceeding packets.
- **Single-rate, three-color policing** uses committed and excess buckets with one rate.
- **Two-rate, three-color policing** uses independently replenished buckets associated with CIR and PIR.
- **Shaping primarily operates on egress traffic**, while policing can operate on ingress or egress depending on platform support.
- Shaping does not eliminate all packet drops. Packets can still be discarded when the shaping queue becomes full.
- Policing and shaping are commonly combined with queuing and scheduling through **hierarchical QoS**.
- `show policy-map interface` is the primary IOS XE verification command.

---

## References

- [Cisco IOS XE 17.x: Policing and Shaping Overview](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/qos/b-quality-of-service/m_qos-plcshp-oview-0.html)
- [Cisco IOS XE 17.x: QoS Packet Policing](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/qos/b-quality-of-service/m_qos-policing.html)
- [Cisco IOS XE 17.x: Regulating Packet Flow Using Traffic Shaping](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/qos/b-quality-of-service/m_qos-regpkt-trfc-0.html)
- [Cisco IOS XE 17.x: QoS Percentage-Based Policing](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/qos/b-quality-of-service/m_qos-plcshp-pct-plc-0.html)
- [Cisco IOS XE 17.x: QoS Percentage-Based Shaping](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/qos/b-quality-of-service/m_qos-plcshp-pct-shp.html)