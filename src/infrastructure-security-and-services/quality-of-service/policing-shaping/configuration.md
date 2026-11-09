# Policing and Shaping Configuration

Cisco IOS XE uses the **Modular QoS CLI (MQC)** to configure traffic policing and shaping.

Both mechanisms regulate traffic rates, but they handle excess traffic differently:
- **Policing:** Immediately transmits, remarks, or drops packets according to the policing outcome.
- **Shaping:** Buffers packets and delays transmission to maintain the configured shaping rate.

This page focuses on practical IOS XE configuration, including policing algorithms, traffic remarking, class-based shaping, percentage-based rates, and verification.

For the underlying mechanisms, see [Traffic Policing](policing.md) and [Traffic Shaping](shaping.md).

---

## MQC Configuration Workflow

Both policing and shaping use three MQC components:

| Component | Purpose |
|---|---|
| `class-map` | Identifies traffic to which the policy applies |
| `policy-map` | Defines the policing or shaping action |
| `service-policy` | Applies the policy to an interface |

The general configuration workflow is:

```text
R1(config)# class-map match-all CLASS
R1(config-cmap)# match ...

R1(config)# policy-map POLICY
R1(config-pmap)# class CLASS
R1(config-pmap-c)# police ...
```

Or, for shaping:

```text
R1(config)# policy-map POLICY
R1(config-pmap)# class CLASS
R1(config-pmap-c)# shape average ...
```

The policy is then applied to an interface:

```text
R1(config)# interface GigabitEthernet1
R1(config-if)# service-policy input POLICY
```

Or:

```text
R1(config-if)# service-policy output POLICY
```

**Policing can operate on ingress or egress, depending on platform support. Shaping is an egress scheduling mechanism.**

The configurations below use IOS XE router-style MQC syntax. Exact supported commands and QoS capabilities vary among IOS XE platforms.

---

## 1. Basic Traffic Policing

### Example: Police All Incoming Traffic to 10 Mbps

Configure R1 to police all incoming traffic on GigabitEthernet1 to 10 Mbps.

Traffic that conforms should be transmitted. Traffic that exceeds should be dropped.

```text
R1(config)# policy-map INGRESS-POLICE
R1(config-pmap)# class class-default
R1(config-pmap-c)# police cir 10000000 conform-action transmit exceed-action drop
R1(config-pmap-c)# exit
R1(config)# interface GigabitEthernet1
R1(config-if)# service-policy input INGRESS-POLICE
```

The configuration:
- Creates a policer with CIR = 10 Mbps.
- Uses the platform's default Bc.
- Transmits conforming packets.
- Drops exceeding packets.
- Applies the policer inbound to GigabitEthernet1.

A policer permits bursts according to the available tokens, so short bursts above CIR can still conform.

### Explicit Burst Configuration

To specify Bc explicitly:

```text
R1(config-pmap-c)# police cir 10000000 bc 125000 conform-action transmit exceed-action drop
```

This configures:
- CIR = 10 Mbps.
- Bc = 125,000 bytes.

The configured Bc represents 100 ms of traffic at CIR:

```text
Tc = (125,000 × 8) / 10,000,000
   = 0.1 seconds
   = 100 ms
```

**For policing, Bc and Be are expressed in bytes when configured as numeric burst sizes.**

---

## 2. Policing Specific Traffic Classes

Policing can be applied to particular classes rather than all traffic.

### Example: Police AF31 Traffic to 5 Mbps

Suppose R1 should:
- Police AF31 traffic to 5 Mbps.
- Drop exceeding AF31 packets.
- Leave other traffic unaffected by this policer.

Create a class map:

```text
R1(config)# class-map match-all BUSINESS
R1(config-cmap)# match dscp af31
```

Create the policy:

```text
R1(config)# policy-map POLICE-BUSINESS
R1(config-pmap)# class BUSINESS
R1(config-pmap-c)# police cir 5000000 bc 125000 conform-action transmit exceed-action drop
```

Apply the policy:

```text
R1(config)# interface GigabitEthernet1
R1(config-if)# service-policy input POLICE-BUSINESS
```

The result is:

| Traffic | Treatment |
|---|---|
| AF31, conforming | Transmit |
| AF31, exceeding | Drop |
| Other DSCP values | Not policed by this policy |

Traffic not matching BUSINESS is handled by `class-default`.

Because no policer is configured for that class, it is not subject to the 5 Mbps limit.

### Multiple Independent Policers

A single policy can contain different policers for different classes.

For example:

```text
R1(config)# class-map match-all VOICE
R1(config-cmap)# match dscp ef
R1(config-cmap)# exit
R1(config)# class-map match-all BUSINESS
R1(config-cmap)# match dscp af31
R1(config-cmap)# exit
R1(config)# policy-map POLICE-CLASSES
R1(config-pmap)# class VOICE
R1(config-pmap-c)# police cir 2000000
R1(config-pmap-c)# exit
R1(config-pmap)# class BUSINESS
R1(config-pmap-c)# police cir 10000000
```

This creates two independent policers:

| Class | CIR |
|---|---:|
| VOICE | 2 Mbps |
| BUSINESS | 10 Mbps |

Each policer maintains separate token accounting.

Unused tokens from one class cannot be used by the other class.

The policy does not impose an aggregate 12 Mbps limit across all traffic. An aggregate limit requires an additional policy mechanism.

---

## 3. Single-Rate, Three-Color Policing (1R3C)

A single-rate, three-color policer uses:
- CIR.
- Bc.
- Be.
- Conform, exceed, and violate actions.

### Example: Transmit, Remark, or Drop

Configure:
- CIR = 5 Mbps.
- Bc = 125,000 bytes.
- Be = 250,000 bytes.
- Conforming packets: Transmit.
- Exceeding packets: Remark AF32 and transmit.
- Violating packets: Drop.

```text
R1(config)# policy-map POLICE-1R3C
R1(config-pmap)# class class-default
R1(config-pmap-c)# police cir 5000000 bc 125000 be 250000 conform-action transmit exceed-action set-dscp-transmit af32 violate-action drop
```

The result is:

| Outcome | Action |
|---|---|
| Conform | Transmit |
| Exceed | Remark AF32, transmit |
| Violate | Drop |

Because no PIR is configured, both buckets share the same committed replenishment rate.

The excess bucket receives additional tokens only when the committed bucket is full.

**Adding Be and a violate action allows the policer to distinguish exceeding traffic from violating traffic.**

---

## 4. Two-Rate, Three-Color Policing (2R3C)

A two-rate, three-color policer uses:
- CIR and Bc.
- PIR and Be.
- Conform, exceed, and violate actions.

### Example: CIR 5 Mbps, PIR 10 Mbps

Configure:

```text
R1(config)# policy-map POLICE-2R3C
R1(config-pmap)# class class-default
R1(config-pmap-c)# police cir 5000000 bc 125000 pir 10000000 be 250000 conform-action transmit exceed-action set-dscp-transmit af32 violate-action drop
```

The parameters are:

| Parameter | Value |
|---|---:|
| CIR | 5 Mbps |
| PIR | 10 Mbps |
| Bc | 125,000 bytes |
| Be | 250,000 bytes |

The actions are:

| Outcome | Action |
|---|---|
| Conform | Transmit |
| Exceed | Remark AF32, transmit |
| Violate | Drop |

In 2R3C policing:
- The committed bucket replenishes at CIR.
- The peak bucket replenishes independently at PIR.
- Conforming packets consume tokens from both buckets.
- Exceeding packets consume peak tokens only.
- Violating packets consume neither.

A packet can exceed PIR instantaneously without necessarily violating if sufficient tokens are available.

For detailed packet-level operation, see [Single-Rate and Two-Rate Policing](single-two-rate.md).

---

## 5. Configuring Policing Actions Separately

Policing actions can also be configured from **police configuration mode**.

For example:

```text
R1(config)# policy-map POLICE-2R3C
R1(config-pmap)# class class-default
R1(config-pmap-c)# police cir 5000000 bc 125000 pir 10000000 be 250000
R1(config-pmap-c-police)# conform-action transmit
R1(config-pmap-c-police)# exceed-action set-dscp-transmit af32
R1(config-pmap-c-police)# violate-action drop
```

This produces the same policing behavior as the previous single-line configuration.

Police configuration mode also allows multiple compatible actions for an individual policing outcome on supported platforms.

For example:

```text
R1(config-pmap-c-police)# conform-action set-dscp-transmit af31
R1(config-pmap-c-police)# exceed-action set-dscp-transmit af32
R1(config-pmap-c-police)# violate-action set-dscp-transmit af33
```

This example remarks all three outcomes rather than dropping violating packets.

The result is:

| Outcome | DSCP | Forwarded? |
|---|---|---|
| Conform | AF31 | Yes |
| Exceed | AF32 | Yes |
| Violate | AF33 | Yes |

This allows downstream WRED policies to treat the packets according to their assigned drop precedence.

**A violate outcome does not automatically mean the packet is dropped. The configured action determines what happens.**

---

## 6. Policing with Percentage-Based Rates

Cisco IOS XE supports specifying CIR as a percentage of the applicable bandwidth.

The syntax is:

```text
R1(config-pmap-c)# police cir percent <percentage>
```

For example:

```text
R1(config-pmap-c)# police cir percent 20
```

This sets CIR to 20% of the relevant bandwidth reference.

If that reference is 100 Mbps:

```text
CIR = 100 Mbps × 20%
    = 20 Mbps
```

### Configuring Bc in Milliseconds

When percentage-based policing is used, burst parameters are specified in milliseconds rather than bytes.

For example:

```text
R1(config-pmap-c)# police cir percent 20 bc 100 ms
```

For a resulting CIR of 20 Mbps:

```text
Bc = (20,000,000 × 0.100) / 8
   = 250,000 bytes
```

### Two-Rate Percentage-Based Policing

On supporting platforms, CIR and PIR can both be expressed as percentages:

```text
R1(config-pmap-c)# police cir percent 20 bc 100 ms pir percent 40 be 100 ms conform-action transmit exceed-action transmit violate-action drop
```

Assuming a bandwidth reference of 100 Mbps:

| Parameter | Result |
|---|---:|
| CIR | 20 Mbps |
| PIR | 40 Mbps |
| Bc | 250,000 bytes |
| Be | 500,000 bytes |

Bc is calculated from CIR, while Be is calculated from PIR in this two-rate configuration.

The actual reference bandwidth depends on the policy attachment and hierarchy.

Percentage-based policing is useful when applying a common policy to interfaces with different bandwidths.

---

## 7. Basic Traffic Shaping

### Example: Shape All Outbound Traffic to 20 Mbps

Suppose R1 has:
- A 100 Mbps physical WAN interface.
- A 20 Mbps provider contract.

Configure an outbound shaper:

```text
R1(config)# policy-map WAN-SHAPE
R1(config-pmap)# class class-default
R1(config-pmap-c)# shape average 20000000
R1(config-pmap-c)# exit
R1(config)# interface GigabitEthernet2
R1(config-if)# service-policy output WAN-SHAPE
```

The policy limits aggregate outbound traffic to an average rate of 20 Mbps.

Packets exceeding the available shaping allowance are queued rather than immediately dropped.

The physical interface still operates at 100 Mbps.

### Verifying the Shaper

```text
R1# show policy-map interface GigabitEthernet2
```

Look for:
- `Service-policy output: WAN-SHAPE`.
- `shape (average)`.
- CIR = 20,000,000 bps.
- Target shaping rate.
- Queue depth and drop statistics.

The shaper becomes active when traffic must be delayed to maintain the configured rate.

If the offered traffic rate is below 20 Mbps, the shaper may not need to buffer packets.

---

## 8. Shaping Individual Traffic Classes

Shaping can limit a particular traffic class without applying the same rate to other classes.

### Example: Shape Backup Traffic to 5 Mbps

Suppose backup traffic is marked CS1.

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

The result is:

| Traffic | Treatment |
|---|---|
| CS1 | Shaped to 5 Mbps |
| Other traffic | Not limited by this shaper |

The 5 Mbps rate applies to the aggregate traffic matching BACKUP.

It does not establish a 5 Mbps limit independently for each backup flow.

### Average vs. Peak Shaping

Cisco IOS XE provides two shaping modes:

```text
R1(config-pmap-c)# shape average 10000000
R1(config-pmap-c)# shape peak 10000000
```

In classic IOS shaping:
- `shape average` limits the long-term rate to CIR.
- `shape peak` uses Bc and Be to calculate a higher sustainable rate.

The classic peak shaping formula is:

```text
Peak rate = CIR × (1 + Be / Bc)
```

For example:

```text
R1(config-pmap-c)# shape peak 10000000 200000 200000
```

Calculates a peak rate of 20 Mbps.

On ASR 1000, Cisco documents that `shape peak` effectively calculates a rate for the hardware scheduler rather than providing the classic periodic excess-burst behavior.

For new configurations on this platform, Cisco recommends configuring the desired effective rate directly with `shape average`.

See [Traffic Shaping](shaping.md) for details.

---

## 9. Percentage-Based Shaping

Cisco IOS XE supports configuring shaping as a percentage of the relevant bandwidth.

For example:

```text
R1(config)# policy-map SHAPE-PERCENT
R1(config-pmap)# class class-default
R1(config-pmap-c)# shape average percent 25
```

If the applicable bandwidth is 100 Mbps:

```text
Shaping rate = 100 Mbps × 25%
             = 25 Mbps
```

### Configuring Bc and Be

On supporting platforms, burst allowances can be expressed in milliseconds:

```text
R1(config-pmap-c)# shape average percent 25 bc 20 ms be 20 ms
```

If the resulting shaping rate is 25 Mbps, Bc corresponds to 20 ms at that rate:

```text
Bc = 25,000,000 × 0.020
   = 500,000 bits
```

For percentage-based shaping, the platform calculates the rate from the applicable bandwidth reference.

**On modern hardware schedulers such as ASR 1000, manually configuring Bc and Be does not necessarily alter the actual scheduling behavior.**

---

## 10. Shaping with Explicit Burst Parameters

On platforms supporting classic burst-based shaping, the command accepts Bc and Be as positional arguments:

```text
R1(config-pmap-c)# shape average <rate-bps> [Bc] [Be]
```

For example:

```text
R1(config-pmap-c)# shape average 10000000 200000 200000
```

This specifies:
- CIR = 10 Mbps.
- Bc = 200,000 bits.
- Be = 200,000 bits.

The corresponding Tc is:

```text
Tc = Bc / CIR
   = 200,000 / 10,000,000
   = 20 ms
```

In classic average shaping, unused tokens can accumulate up to Bc + Be.

### Important Unit Difference

| Feature | Rate unit | Explicit burst unit |
|---|---|---|
| `police cir 10000000 bc 125000` | bps | Bytes |
| `shape average 10000000 200000 200000` | bps | Bits |
| `police cir percent 20 bc 100 ms` | Percentage | Milliseconds |
| `shape average percent 25 bc 20 ms` | Percentage | Milliseconds |

This distinction is important when interpreting configuration values.

---

## 11. Shaping with Overhead Accounting

By default, a shaper may not account for every byte transmitted on the physical medium.

On supported IOS XE platforms, the `account` keyword modifies the packet length used for scheduling calculations.

For example:

```text
R1(config-pmap-c)# shape average 50000000 account user-defined 24
```

This:
- Shapes traffic to an average accounted rate of 50 Mbps.
- Adds 24 bytes to each packet's accounted size.
- Does not modify the packet transmitted on the wire.

The additional 24 bytes can compensate for Ethernet overhead not included in the platform's default accounting.

This is useful when the provider enforces its rate limit using a different packet-length calculation.

### Example

Suppose the shaper's normal accounting measures a packet as 1,000 bytes.

With:

```text
R1(config-pmap-c)# shape average 50000000 account user-defined 24
```

The shaper accounts for:

```text
1,000 + 24 = 1,024 bytes
```

The packet itself remains unchanged.

The larger accounted size consumes more scheduling allowance, reducing the number of packets the shaper can transmit during a given period.

This can help prevent exceeding a provider's contracted on-wire rate.

The supported accounting options and default packet-length calculations are platform-dependent.

---

## 12. Combining Policing and Shaping

Policing and shaping can be used together on the same router.

A common design applies:
- Ingress policing to enforce application traffic limits.
- Egress shaping to match the provider's contracted bandwidth.

Consider:

```text
              Enterprise R1
         +-----------------------+
         |                       |
LAN -----| Gi1               Gi2 |----- WAN
         |                       |
         | Ingress Police        |
         | 5 Mbps AF31           |
         |                       |
         |        Egress Shape   |
         |        20 Mbps total  |
         +-----------------------+
```

### Step 1: Configure Ingress Policing

```text
R1(config)# class-map match-all BUSINESS
R1(config-cmap)# match dscp af31
R1(config-cmap)# exit
R1(config)# policy-map INGRESS-POLICE
R1(config-pmap)# class BUSINESS
R1(config-pmap-c)# police cir 5000000 conform-action transmit exceed-action drop
R1(config-pmap-c)# exit
R1(config)# interface GigabitEthernet1
R1(config-if)# service-policy input INGRESS-POLICE
```

This enforces a 5 Mbps policing profile for AF31 traffic entering GigabitEthernet1.

### Step 2: Configure Egress Shaping

```text
R1(config)# policy-map WAN-SHAPE
R1(config-pmap)# class class-default
R1(config-pmap-c)# shape average 20000000
R1(config-pmap-c)# exit
R1(config)# interface GigabitEthernet2
R1(config-if)# service-policy output WAN-SHAPE
```

This shapes aggregate traffic leaving GigabitEthernet2 to 20 Mbps.

### Result

| Traffic | Ingress treatment | Egress treatment |
|---|---|---|
| AF31, conforming | Transmit | Subject to 20 Mbps aggregate shaper |
| AF31, exceeding | Drop | Never reaches the shaper |
| Other traffic | Not policed by the ingress policy | Subject to 20 Mbps aggregate shaper |

This design uses policing and shaping for separate purposes.

The ingress policer restricts a particular traffic class.

The egress shaper limits aggregate WAN transmission.

---

## 13. Shaping with Hierarchical QoS

A common WAN configuration combines:
- A parent shaper.
- A child queuing policy.

The parent controls the total transmitted rate.

The child determines how bandwidth is allocated among traffic classes.

### Example: Shape to 20 Mbps with Voice Priority

Create the class maps:

```text
R1(config)# class-map match-all VOICE
R1(config-cmap)# match dscp ef
R1(config-cmap)# exit
R1(config)# class-map match-all BUSINESS
R1(config-cmap)# match dscp af31
```

Create the child policy:

```text
R1(config)# policy-map CHILD-QOS
R1(config-pmap)# class VOICE
R1(config-pmap-c)# priority percent 10
R1(config-pmap-c)# exit
R1(config-pmap)# class BUSINESS
R1(config-pmap-c)# bandwidth remaining percent 50
```

Create the parent policy:

```text
R1(config)# policy-map PARENT-SHAPE
R1(config-pmap)# class class-default
R1(config-pmap-c)# shape average 20000000
R1(config-pmap-c)# service-policy CHILD-QOS
```

Apply the parent policy:

```text
R1(config)# interface GigabitEthernet2
R1(config-if)# service-policy output PARENT-SHAPE
```

The resulting hierarchy is:

```text
             PARENT-SHAPE
             Shape 20 Mbps
                   |
                   v
               CHILD-QOS
              /    |     \
             /     |      \
          VOICE  BUSINESS  DEFAULT
          LLQ     CBWFQ    Other
```

On supporting platforms:
- The parent limits aggregate transmission to 20 Mbps.
- The VOICE class receives priority scheduling.
- The BUSINESS class receives a weighted share of remaining bandwidth.
- Unmatched traffic is handled by `class-default`.

The percentages are interpreted according to the platform's hierarchical scheduler and bandwidth reference.

Detailed bandwidth calculations, priority policing, and scheduling behavior are covered in [4.4.g Hierarchical QoS (HQoS)](../hqos/overview.md).

---

## 14. Verification

The primary command for both policing and shaping is:

```text
R1# show policy-map interface
```

This displays applied MQC policies and available statistics.

### Verify Ingress Policing

```text
R1# show policy-map interface GigabitEthernet1 input
```

Example output, simplified:

```text
GigabitEthernet1

 Service-policy input: INGRESS-POLICE

  Class-map: BUSINESS (match-all)
    10000 packets, 15000000 bytes
    Match: dscp af31 (26)
    police:
        cir 5000000 bps, bc 156250 bytes
      conformed 8000 packets, 12000000 bytes; actions:
        transmit
      exceeded 2000 packets, 3000000 bytes; actions:
        drop
```

This indicates:
- CIR = 5 Mbps.
- 8,000 packets conformed.
- 2,000 packets exceeded and were dropped.

The output format and default burst values vary by platform.

### Verify Three-Color Policing

```text
R1# show policy-map interface GigabitEthernet1
```

Example output, simplified:

```text
police:
    cir 5000000 bps, bc 125000 bytes
    pir 10000000 bps, be 250000 bytes
  conformed 8000 packets, 12000000 bytes; actions:
    transmit
  exceeded 1500 packets, 2250000 bytes; actions:
    set-dscp-transmit af32
  violated 500 packets, 750000 bytes; actions:
    drop
```

The output shows separate counters for conform, exceed, and violate.

Remember that the counters indicate **policing outcomes**, not necessarily packet drops.

An exceeding packet with `set-dscp-transmit af32` is forwarded rather than discarded.

### Verify Egress Shaping

```text
R1# show policy-map interface GigabitEthernet2 output
```

Example output, simplified:

```text
GigabitEthernet2

 Service-policy output: WAN-SHAPE

  Class-map: class-default (match-any)
    Match: any
    Queueing
    (queue depth/total drops/no-buffer drops) 12/0/0
    shape (average) cir 20000000
    target shape rate 20000000
```

The important fields are:
- Shaping mode.
- CIR or target shaping rate.
- Current queue depth.
- Queue drops.

Depending on the platform, the output may also include Bc, Be, offered rate, and transmitted rate.

### Useful Verification Commands

| Command | Purpose |
|---|---|
| `show policy-map` | Display configured policies |
| `show policy-map POLICY` | Examine a particular policy |
| `show policy-map interface` | Show applied policies and statistics |
| `show policy-map interface GigabitEthernet1 input` | Verify ingress policing |
| `show policy-map interface GigabitEthernet2 output` | Verify egress shaping |
| `show class-map` | Verify classification criteria |
| `show running-config interface GigabitEthernet2` | Confirm policy attachment |
| `show interfaces GigabitEthernet2` | Check interface rates and drops |

On supporting QFP-based platforms, additional hardware information may be available:

```text
R1# show platform hardware qfp active feature qos interface GigabitEthernet2
```

The exact syntax and displayed information depend on the platform.

---

## 15. Troubleshooting

### Policing Drops Traffic Below CIR

Possible causes:
- Short traffic bursts exhaust Bc.
- Bc is too small for the packet sizes and burst patterns.
- Multiple flows share the same policer.
- The policer accounts for additional encapsulation overhead.
- The measured average rate hides short-term bursts.

Examine conform and exceed counters rather than relying only on average interface utilization.

### A Policer Does Not Enforce the Expected Maximum Rate

Check:
- Whether the policy is applied to the correct interface and direction.
- Whether traffic matches the expected class.
- Whether the exceed or violate action actually drops traffic.
- Whether Bc and Be permit large bursts.
- Whether the observed rate includes different packet overhead from the policer's accounting.

For example:

```text
R1(config-pmap-c)# police cir 5000000 conform-action transmit exceed-action set-dscp-transmit af32
```

Does not enforce a hard 5 Mbps forwarded-rate limit.

Exceeding packets are remarked and transmitted.

### Shaping Does Not Limit Traffic

Check:
- Whether the shaping policy is attached outbound.
- Whether the correct traffic class is matched.
- Whether the shaping rate is greater than the offered rate.
- Whether the policy shapes all traffic or only one class.
- Whether a percentage-based rate uses the expected bandwidth reference.

If the offered traffic is below the shaping rate, the shaper may not need to delay any packets.

### Shaping Causes Excessive Delay

Possible causes:
- Sustained offered traffic exceeds the shaping rate.
- Queue limits are too large.
- Multiple applications share one shaping queue.
- No priority scheduling is configured for delay-sensitive applications.

Consider hierarchical QoS with appropriate queuing and queue limits.

### Shaping Causes Packet Drops

Possible causes:
- The shaping queue becomes full.
- Traffic is offered above the shaping rate for a sustained period.
- Available hardware buffering is exhausted.
- The queue limit is too small for normal traffic bursts.

Shaping attempts to delay excess packets, but it cannot prevent all packet loss.

---

## Key Commands

| Objective | Command |
|---|---|
| Police traffic to 10 Mbps | `police cir 10000000` |
| Set policing Bc | `police cir 10000000 bc 125000` |
| Configure 1R3C | `police cir 5000000 bc 125000 be 250000` |
| Configure 2R3C | `police cir 5000000 bc 125000 pir 10000000 be 250000` |
| Transmit conforming traffic | `conform-action transmit` |
| Drop exceeding traffic | `exceed-action drop` |
| Remark exceeding traffic | `exceed-action set-dscp-transmit af32` |
| Drop violating traffic | `violate-action drop` |
| Percentage-based policing | `police cir percent 20` |
| Police with a burst specified in milliseconds | `police cir percent 20 bc 100 ms` |
| Average shaping | `shape average 20000000` |
| Peak shaping | `shape peak 10000000 200000 200000` |
| Percentage-based shaping | `shape average percent 25` |
| Shaping overhead accounting | `shape average 50000000 account user-defined 24` |
| Apply ingress policy | `service-policy input POLICY` |
| Apply egress policy | `service-policy output POLICY` |
| Attach child QoS policy | `service-policy CHILD-QOS` |
| Verify policy | `show policy-map interface` |

---

## Key Points

- Cisco IOS XE uses MQC to configure policing and shaping.
- `police cir` configures a committed-rate policer.
- Adding `be` allows a single-rate three-color policing configuration.
- Adding `pir` configures two-rate policing.
- `conform-action`, `exceed-action`, and `violate-action` determine the treatment of each policing outcome.
- `set-dscp-transmit` remarks and transmits packets.
- Numeric Bc and Be values in policing are expressed in **bytes**.
- Positional Bc and Be values in classic shaping are expressed in **bits**.
- Percentage-based policing and shaping can express burst allowances in milliseconds.
- `shape average` is the primary command for limiting outbound transmission rates.
- Shaping is applied outbound, while policing can be applied inbound or outbound on supporting platforms.
- `class-default` can be used to police or shape all traffic handled by the policy.
- Separate class policers have independent token buckets.
- A parent shaper can be combined with a child queuing policy to implement hierarchical QoS.
- Shaping overhead accounting adjusts the packet size used in scheduling calculations without changing transmitted packets.
- `show policy-map interface` is the primary verification command.

---

## References

- [Cisco IOS XE 17.x: QoS Packet Policing](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/qos/b-quality-of-service/m_qos-policing.html)
- [Cisco IOS XE 17.x: Two-Rate Policer](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/qos/b-quality-of-service/m_qos-plcshp-two-rate-plcr-0.html)
- [Cisco IOS XE 17.x: QoS Percentage-Based Shaping](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/qos/b-quality-of-service/m_qos-plcshp-pct-shp.html)
- [Cisco IOS XE 17.x: QoS Scheduling](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/qos/b-quality-of-service/m_qos-scheduling.html)
- [Cisco IOS XE 17.x: Policer Enhancement Multiple Actions](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/qos/b-quality-of-service/m_qos-plcshp-plcr-mact-0.html)