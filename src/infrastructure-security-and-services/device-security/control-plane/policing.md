# CoPP Policing Rates and Actions

**Control Plane Policing (CoPP)** uses traffic policers to limit the amount of traffic that can reach a device's control plane.

After a packet matches a CoPP class, the configured **policer** determines whether the packet is within the permitted traffic rate and what action should be taken.

CoPP commonly uses three actions:

- **Conform:** Traffic is within the configured rate and burst allowance.
- **Exceed:** Traffic exceeds the committed allowance.
- **Violate:** Traffic exceeds the additional allowance provided by a three-color policer.

For example, a policy might allow SSH traffic at a specified rate and drop packets exceeding that rate.

```text
Incoming SSH Traffic
        |
        v
  CoPP SSH Class
        |
        v
      Policer
        |
        +---- Conform ----> Transmit to CPU
        |
        +---- Exceed -----> Drop
```

Policing protects the CPU by limiting the amount of traffic admitted for processing.

---

## Policing vs. Shaping

Policing and shaping both regulate traffic rates, but they operate differently.

| Feature | Policing | Shaping |
|---|---|---|
| Traffic exceeding the configured allowance | Can be dropped or remarked | Buffered and transmitted later |
| Uses buffering to delay excess traffic | No | Yes |
| Can immediately drop excessive traffic | Yes | Not the normal behavior |
| Used for CoPP | Yes | No |

**CoPP uses policing, not shaping.**

Buffering excessive control-plane traffic could allow large backlogs to accumulate, defeating the purpose of protecting the CPU from excessive processing demands.

Policing can discard excessive packets immediately.

---

## Token Bucket Policing

Traffic policing commonly uses a **token bucket** mechanism to enforce an average traffic rate while permitting limited bursts.

A token bucket has two important properties:

1. **Token replenishment rate:** Determines the sustained traffic rate.

2. **Bucket capacity:** Determines how much traffic can be sent in a burst.

For a byte-based policer, tokens represent bytes of traffic.

```text
              Tokens added at CIR
                      |
                      v
              +---------------+
              |               |
              | Token Bucket  |
              |               |
              +---------------+
                      |
                      v
               Incoming Packet
                      |
               Enough tokens?
                 /         \\
               Yes          No
                |            |
                v            v
             Conform       Exceed
```

When a packet arrives:

1. The policer checks whether enough tokens are available in the bucket to cover the packet.

2. If there are enough tokens, the packet **conforms** and the required tokens are removed.

3. If aren't enough tokens, the packet **exceeds** the committed allowance and receives the configured exceed treatment.

Tokens are replenished over time, up to the configured bucket capacity.

This allows brief traffic bursts without requiring the sustained traffic rate to exceed the configured limit.

> A policing rate is not necessarily a strict instantaneous maximum. Token bucket policing permits traffic bursts according to the configured burst parameters.

---

## Committed Information Rate (CIR)

The **Committed Information Rate (CIR)** specifies the sustained rate permitted by a policer.

CIR is commonly configured in **bits per second (bps)**.

For example:

```text
CIR = 1,000,000 bps
```

This represents a sustained rate of 1 Mbps.

On supported IOS XE router platforms, a basic single-rate policer can be configured using:

```text
R1(config)# policy-map COPP-IN
R1(config-pmap)# class COPP-SSH
R1(config-pmap-c)# police cir 1000000 conform-action transmit exceed-action drop
```

This instructs the policer to:

- Transmit conforming packets.
- Drop packets exceeding the committed allowance.

The class map `COPP-SSH` must already exist.

An alternative supported syntax uses the rate directly:

```text
R1(config-pmap-c)# police 1000000 conform-action transmit exceed-action drop
```

The exact syntax and supported rate ranges depend on the IOS XE platform.

### Default Policer Actions

For a basic two-color policer, the normal default actions are:

| Condition | Default Action |
|---|---|
| Conform | Transmit |
| Exceed | Drop |

Therefore, on platforms supporting the simplified syntax:

```text
R1(config-pmap-c)# police 1000000
```

is equivalent in basic behavior to:

```text
R1(config-pmap-c)# police 1000000 conform-action transmit exceed-action drop
```

The explicit form makes the intended behavior easier to understand.

---

## Committed Burst (Bc)

The **Committed Burst (Bc)** defines the amount of traffic that can accumulate in the committed token bucket.

For byte-based policing, Bc is measured in **bytes**.

A larger Bc permits larger short-term traffic bursts.

A smaller Bc restricts bursts more aggressively.

For example:

```text
R1(config-pmap-c)# police cir 1000000 bc 12500 conform-action transmit exceed-action drop
```

This configures:

| Parameter | Value |
|---|---|
| CIR | 1,000,000 bps |
| Bc | 12,500 bytes |
| Conform Action | Transmit |
| Exceed Action | Drop |

At a CIR of 1 Mbps, a committed burst of 12,500 bytes represents approximately 100 milliseconds of tokens.

The relationship is:

```text
Burst duration (seconds) = Bc (bytes) × 8 / CIR (bps)
```

For this example:

```text
12,500 × 8 / 1,000,000 = 0.1 seconds
```

This means the bucket can accumulate approximately 100 milliseconds' worth of traffic allowance when it is not being consumed.

It does not mean that the router measures traffic only once every 100 milliseconds.

### Effect of Burst Size

Consider two policers using the same CIR:

| Parameter | Policer A | Policer B |
|---|---|---|
| CIR | 1 Mbps | 1 Mbps |
| Bc | 5,000 bytes | 50,000 bytes |
| Short traffic bursts | More restrictive | More tolerant |
| Sustained allowed rate | 1 Mbps | 1 Mbps |

Both policers enforce the same sustained rate, but Policer B can accommodate larger bursts.

**Increasing Bc does not increase CIR.**

It increases the amount of short-term traffic that can conform.

> Burst sizes are particularly important for control-plane traffic because protocols such as routing protocols can generate bursts during neighbor establishment, topology changes, and reconvergence.

---

## Excess Burst (Be)

The **Excess Burst (Be)** provides an additional burst allowance in policing models that support three traffic colors.

This allows the policer to distinguish among:

1. **Conforming traffic:** Within the committed allowance.

2. **Exceeding traffic:** Outside the committed allowance but within an additional allowance.

3. **Violating traffic:** Outside both allowances.

For a single-rate, three-color policer, tokens replenish the committed bucket at the CIR. Only tokens that overflow from the full committed bucket can replenish the additional excess bucket.

An example is:

```text
R1(config-pmap-c)# police cir 1000000 bc 12500 be 25000 conform-action transmit exceed-action transmit violate-action drop
```

The parameters are:

| Parameter | Value |
|---|---|
| CIR | 1 Mbps |
| Bc | 12,500 bytes |
| Be | 25,000 bytes |
| Conform Action | Transmit |
| Exceed Action | Transmit |
| Violate Action | Drop |

The policer permits some traffic beyond the committed burst allowance before dropping packets that violate both allowances.

The additional allowance does not change the sustained CIR.

> The availability and syntax of Bc, Be, and three-color policing depend on the platform and policing mode. Not every IOS XE CoPP implementation supports all policing variants.

---

## Two-Color vs. Three-Color Policing

Cisco IOS XE supports three main token-bucket policing models. The diagrams below illustrate the **color-blind** versions, in which classification depends on the token counters rather than an incoming packet color.

In all three diagrams, **B** is the size of the arriving packet (in bytes). A packet consumes tokens only when the selected bucket or buckets have at least B tokens available.

### Single-Rate, Two-Color Policing (1R2C)

A **single-rate, two-color (1R2C)** policer uses one token bucket and one replenishment rate, the **CIR**.

```text
                 Tokens added at CIR
                          |
                          v
                   +-------------+
                   |  Committed  |
                   |  bucket (C) |    Capacity: Bc
                   +-------------+
                          |
                Packet arrives (B)
                          |
                     C >= B ?
                     /      \
                   Yes       No
                    |         |
                    v         v
                CONFORM     EXCEED
               Remove B    No tokens
                from C      removed
```

- The committed bucket refills at the **CIR**, up to **Bc**. Additional tokens are discarded when it is full.
- If the bucket contains at least B tokens, the packet **conforms** and B tokens are removed.
- Otherwise, the packet **exceeds** and no tokens are removed.

| Condition | Meaning |
|---|---|
| Conform | Sufficient tokens in the committed bucket |
| Exceed | Insufficient tokens in the committed bucket |

Example:

```text
R1(config-pmap-c)# police cir 1000000 bc 12500 conform-action transmit exceed-action drop
```

This is a common, straightforward CoPP policing model.

### Single-Rate, Three-Color Policing (1R3C)

A **single-rate, three-color (1R3C)** policer uses one rate (**CIR**) and two token buckets: a committed bucket (**Bc**) and an excess bucket (**Be**).

The buckets share a single token source. **Tokens overflow into the excess bucket only when the committed bucket is full.**

```text
                     Tokens added at CIR
                              |
                              v
                    +--------------------+
                    | Committed bucket C |   Capacity: Bc
                    +--------------------+
                              |
                     Overflow ONLY when
                      C is already full
                              |
                              v
                    +--------------------+
                    | Excess bucket E    |   Capacity: Be
                    +--------------------+
                              |
                  Further overflow discarded

Packet arrives (B)
       |
       v
   C >= B ? -------- Yes -------> CONFORM
       |                         Remove B from C
       No
       |
       v
   E >= B ? -------- Yes -------> EXCEED
       |                         Remove B from E
       No
       |
       v
    VIOLATE
 No tokens removed
```

- The committed bucket refills at **CIR** up to **Bc**.
- **Only overflow from a full committed bucket** replenishes the excess bucket; Be does not have an independent refill rate.
- If C has sufficient tokens, the packet **conforms** (only C is decremented).
- Otherwise, if E has sufficient tokens, it **exceeds** (only E is decremented).
- Otherwise, it **violates** (neither bucket is decremented).

| Condition | Meaning |
|---|---|
| Conform | Committed bucket has enough tokens |
| Exceed | Committed bucket lacks tokens, but excess bucket has enough |
| Violate | Neither bucket has enough tokens |

Example:

```text
R1(config-pmap-c)# police cir 1000000 bc 12500 be 25000 conform-action transmit exceed-action transmit violate-action drop
```

**Important:** A sustained stream above CIR generally drains the excess bucket, because it is replenished only when the committed bucket becomes full. Over time, packets tend to alternate between **conform** and **violate**, rather than continuously exceeding.

### Two-Rate, Three-Color Policing (2R3C)

A **two-rate, three-color (2R3C)** policer uses two token buckets that **refill independently**:

- **Committed bucket (C):** Refilled at CIR, with capacity Bc.
- **Peak bucket (P):** Refilled at PIR, with capacity Be (the peak burst allowance in Cisco IOS XE syntax).

There is **no token overflow between the buckets**. PIR must be greater than CIR for this IOS XE policing model.

```text
        Tokens at CIR                  Tokens at PIR
              |                              |
              v                              v
     +------------------+           +------------------+
     | Committed bucket |           |    Peak bucket   |
     |        (C)       |           |        (P)       |
     +------------------+           +------------------+
        Capacity: Bc                   Capacity: Be

                   No overflow between buckets

Packet arrives (B)
       |
       v
   P >= B ? -------- No --------> VIOLATE
       |                          No tokens removed
      Yes
       |
       v
   C >= B ? -------- No --------> EXCEED
       |                          Remove B from P only
      Yes
       |
       v
    CONFORM
 Remove B from BOTH C and P
```

- The **peak bucket is checked first**. If it has insufficient tokens, the packet **violates**.
- If the peak bucket has enough tokens but the committed bucket does not, the packet **exceeds**. Tokens are deducted **only from P**.
- If both buckets have enough tokens, the packet **conforms**. Tokens are deducted **from both C and P**.
- If the packet violates, neither bucket is decremented.

| Condition | Meaning |
|---|---|
| Conform | Both committed and peak buckets have enough tokens |
| Exceed | Peak bucket has enough tokens, committed bucket does not |
| Violate | Peak bucket does not have enough tokens |

Example, on platforms supporting this model:

```text
R1(config-pmap-c)# police cir 500000 bc 10000 pir 1000000 be 10000 conform-action transmit exceed-action transmit violate-action drop
```

| Parameter | Value |
|---|---|
| CIR | 500 kbps |
| PIR | 1 Mbps |
| Bc | 10,000 bytes |
| Be (peak burst) | 10,000 bytes |

Unlike 1R3C, **2R3C can continually classify traffic between CIR and PIR as exceeding**, because the peak bucket has its own refill rate. Actual per-packet results also depend on the available tokens and burst allowances.

> These algorithms describe QoS policing behavior on supporting IOS XE platforms. CoPP commonly uses simpler two-color policing, and support for three-color/two-rate policing in control-plane policies is platform-dependent.

---

## Bits per Second vs. Packets per Second

CoPP policing rates can be expressed using different units, depending on the platform.

Two important units are:

| Unit | Meaning |
|---|---|
| **bps** | Bits per second |
| **pps** | Packets per second |

### Bits per Second

A bps policer limits traffic based on the number of bits transmitted.

On supported IOS XE platforms, bps policing can be configured using either `police cir` or `police rate`.

```text
R1(config-pmap-c)# police cir 1000000 bc 12500
```

Or:

```text
R1(config-pmap-c)# police rate 1000000 bps burst 12500 bytes
```

Both examples configure a sustained policing rate of **1 Mbps** with a committed burst allowance of **12,500 bytes**.

Larger packets consume more tokens than smaller packets.

### Packets per Second

A pps policer limits the packet arrival rate rather than the total bandwidth.

For example, on supported IOS XE platforms:

```text
R1(config-pmap-c)# police rate 100 pps
```

This configures a committed rate of **100 packets per second**.

Packet-based policing is useful for protecting the CPU because processing many small packets can consume significant CPU resources even when the total bandwidth is low.

> `police cir` and `police rate ... bps` both configure bit-rate policing. The main differences are their syntax and platform-supported options. Use `police rate ... pps` for packet-rate policing where supported.

### Comparison

Consider traffic consisting of 100-byte packets and 1,000-byte packets.

| Traffic | 100 pps | Approximate IP packet bandwidth |
|---|---|---|
| 100-byte packets | 100 packets/s | 80 kbps |
| 1,000-byte packets | 100 packets/s | 800 kbps |

The same pps rate can represent significantly different bandwidth consumption.

Conversely, the same bps rate can represent different packet rates.

**A bps policer limits traffic volume, while a pps policer limits packet rate.**

Packet-size accounting details can vary by platform.

---

## Packet-Based Burst Configuration

On platforms supporting packet-rate policing, burst sizes can also be expressed in packets.

For example:

```text
R1(config-pmap-c)# police rate 100 pps burst 25 packets
```

This configures:

| Parameter | Value |
|---|---|
| Committed Rate | 100 pps |
| Burst | 25 packets |

The policer replenishes tokens at a rate corresponding to 100 packets per second, up to a capacity of 25 packet tokens.

This allows brief bursts above 100 pps, provided sufficient tokens have accumulated.

Without sufficient tokens, packets receive the configured exceed treatment.

If no burst size is specified, the platform supplies a default.

> Cisco IOS XE supports different packet-based policing syntax depending on the platform. The `police rate ... pps` syntax is particularly important for Catalyst 9000 System CoPP.

---

## Policer Actions

The policing action determines what happens to packets after they have been classified by the policer.

The most important actions for CoPP are **transmit** and **drop**.

| Action | Behavior |
|---|---|
| `transmit` | Allows the packet to continue toward the control plane |
| `drop` | Discards the packet |
| `set-dscp-transmit` | Remarks DSCP and transmits the packet, where supported |
| `set-prec-transmit` | Remarks IP precedence and transmits the packet, where supported |

For example:

```text
R1(config-pmap-c)# police cir 1000000 conform-action transmit exceed-action drop
```

Packets within the committed allowance are transmitted.

Packets exceeding that allowance are dropped.

### conform-action

The `conform-action` parameter specifies what happens to conforming traffic.

For example:

```text
conform-action transmit
```

This allows packets to continue toward CPU processing.

### exceed-action

The `exceed-action` parameter specifies what happens to traffic exceeding the committed allowance.

For example:

```text
exceed-action drop
```

This causes excessive packets to be discarded.

Alternatively:

```text
exceed-action transmit
```

allows packets to continue even when they exceed the committed allowance.

This can be useful when initially evaluating traffic classification and policing rates.

### violate-action

The `violate-action` parameter applies to traffic classified as violating the policer's additional allowance.

For example:

```text
violate-action drop
```

It is used by policing models that distinguish a third traffic condition.

A simple two-color policer does not require a separate violate action.

---

## Common CoPP Policing Behaviors

The following examples illustrate common policy designs.

### 1. Limit Traffic to a Specific Rate

```text
R1(config)# policy-map COPP-IN
R1(config-pmap)# class COPP-SSH
R1(config-pmap-c)# police cir 128000 bc 8000 conform-action transmit exceed-action drop
```

SSH traffic matching the class is permitted within the policer's allowance.

Excessive traffic is dropped.

### 2. Permit All Traffic in a Class

```text
R1(config)# policy-map COPP-IN
R1(config-pmap)# class COPP-ROUTING
```

If no policing or dropping action is configured, matching routing traffic is not restricted by this class.

Alternatively, a policer can be configured with both actions set to transmit:

```text
R1(config-pmap-c)# police 1000000 conform-action transmit exceed-action transmit
```

This still classifies and meters traffic but does not drop packets because of the configured rate.

### 3. Drop All Matching Traffic

On supported platforms, the following policer drops matching traffic regardless of the committed rate:

```text
R1(config)# policy-map COPP-IN
R1(config-pmap)# class COPP-UNWANTED
R1(config-pmap-c)# police 8000 conform-action drop exceed-action drop
```

Some IOS XE platforms also support a direct `drop` action at the class level.

However, support for that command varies by platform and release.

**To drop an entire class, both conforming and exceeding traffic must be dropped**, or an equivalent supported class-level drop action must be used.

### 4. Monitor Before Enforcing Drops

An initial CoPP deployment can be configured to transmit both conforming and exceeding packets:

```text
R1(config)# policy-map COPP-IN
R1(config-pmap)# class COPP-MGMT
R1(config-pmap-c)# police 128000 conform-action transmit exceed-action transmit
```

After observing traffic rates and confirming that the classification is correct, the exceed action can be changed:

```text
R1(config-pmap-c)# police 128000 conform-action transmit exceed-action drop
```

This approach reduces the risk of disrupting legitimate control-plane traffic while determining suitable policing values.

---

## Protecting Critical Routing Traffic

One of the most important CoPP design considerations is avoiding excessive policing of critical control-plane protocols.

For example, OSPF traffic is normally relatively low-volume during stable operation.

However, significant topology changes can temporarily increase the amount of OSPF control traffic.

If the CoPP rate is too restrictive, important packets may be dropped.

This can interfere with:

- Neighbor establishment.
- Routing protocol Hello exchanges.
- Database synchronization.
- Route reconvergence.
- Session stability.

A policy that appears sufficient during normal operation might be inadequate during a network failure.

Therefore, CoPP policing rates should account for **peak legitimate control-plane traffic**, not just average traffic.

### Example: Routing and Management Classes

```text
R1(config)# policy-map COPP-IN
R1(config-pmap)# class COPP-ROUTING
R1(config-pmap-c)# police 1000000 conform-action transmit exceed-action drop
R1(config-pmap-c)# exit
R1(config-pmap)# class COPP-MGMT
R1(config-pmap-c)# police 128000 conform-action transmit exceed-action drop
```

This example gives routing traffic a higher committed rate than management traffic.

The rates are illustrative, not universal recommendations.

The correct values depend on the network size, protocols, platform, and actual control-plane traffic patterns.

---

## class-default and CoPP Safety

The special `class-default` class matches CPU-bound traffic that does not match any preceding class.

Aggressively policing this class can be dangerous because it may include legitimate traffic that has not been explicitly classified.

For example:

```text
R1(config)# policy-map COPP-IN
R1(config-pmap)# class class-default
R1(config-pmap-c)# police 8000 conform-action drop exceed-action drop
```

This configuration would drop all CPU-bound traffic matching `class-default`.

Depending on the classification rules and platform, that could include essential control-plane traffic.

Cisco recommends avoiding aggressive policing of `class-default` unless all necessary traffic has been accounted for.

A safer initial approach is to leave `class-default` without a dropping action while validating the traffic classes.

---

## Catalyst 9000 System CoPP Policing Rates

Modern Catalyst 9000 switches use predefined **System CoPP** classes and policers.

Unlike router platforms supporting user-defined CoPP policies, Catalyst 9000 System CoPP generally permits administrators to adjust policing rates only for the predefined classes.

The policing rates are commonly specified in **packets per second (pps)**.

For example, on a supported Catalyst 9300:

```text
SW1(config)# policy-map system-cpp-policy
SW1(config-pmap)# class system-cpp-police-protocol-snooping
SW1(config-pmap-c)# police rate 2000 pps
```

This sets the policing rate for the predefined protocol-snooping class to 2,000 pps.

The switch automatically uses a platform-defined burst size unless otherwise supported and configured.

Verify with:

```text
SW1# show policy-map control-plane
```

Example output:

```text
Class-map: system-cpp-police-protocol-snooping (match-any)
  police:
    rate 2000 pps, burst 488 packets
  conformed 0 bytes; actions:
    transmit
  exceeded 0 bytes; actions:
    drop
```

This indicates:

- Packets conforming to the policer are transmitted.
- Packets exceeding the policer are dropped.
- The configured rate is 2,000 pps.
- The reported burst allowance is 488 packets.

The exact default policing rates, burst values, and available classes depend on the Catalyst model and IOS XE release.

> Changing System CoPP policing rates can affect routing, switching, and management functions. Avoid using arbitrary policing values without understanding the traffic handled by each predefined class.

---

## IOS XE Verification

The main command for verifying a CoPP policy is:

```text
R1# show policy-map control-plane
```

On supported platforms, the output displays the applied policy and traffic statistics.

For example:

```text
Control Plane
 Service-policy input: COPP-IN
  Class-map: COPP-MGMT (match-all)
    1000 packets, 800000 bytes
    5 minute offered rate 64000 bps, drop rate 8000 bps
    Match: access-group name COPP-MGMT-ACL
    police:
      cir 128000 bps, bc 8000 bytes
      conformed 900 packets, 720000 bytes; actions:
        transmit
      exceeded 100 packets, 80000 bytes; actions:
        drop
```

**Illustrative output; actual formatting varies by platform.**

### Interpreting the Output

| Field | Meaning |
|---|---|
| `offered rate` | Measured rate of traffic entering the class |
| `drop rate` | Measured rate of traffic being dropped |
| `cir` | Configured committed rate |
| `bc` | Configured committed burst size |
| `conformed` | Packets classified as conforming |
| `exceeded` | Packets classified as exceeding |
| `violated` | Packets classified as violating, when supported |
| `actions` | Action applied to each traffic condition |

A nonzero exceed count does not necessarily indicate a problem.

However, persistent drops in a critical routing protocol class may indicate that the policing rate or burst allowance is insufficient.

Similarly, a class showing no matched packets might indicate incorrect classification or that the expected traffic is not reaching the control plane.

### Check the Configured Policy

```text
R1# show policy-map
```

Displays configured QoS policy maps.

```text
R1# show running-config | section policy-map
```

Displays configured policy-map commands.

```text
R1# show policy-map control-plane input
```

Displays statistics for the applied input control-plane policy on supported platforms.

---

## Key Points

- **CoPP uses policing, not shaping**, to regulate CPU-bound traffic.
- Policing typically uses a **token bucket** mechanism.
- **CIR** determines the sustained permitted rate.
- **Bc** determines the committed burst allowance.
- **Be** provides an additional burst allowance in policing models that support it.
- Increasing Bc allows larger bursts but **does not increase CIR**.
- A **two-color policer** distinguishes conforming and exceeding traffic.
- A **three-color policer** distinguishes conforming, exceeding, and violating traffic.
- **`conform-action transmit`** allows conforming packets to proceed.
- **`exceed-action drop`** discards packets exceeding the committed allowance.
- **`violate-action drop`** discards packets classified as violating the additional allowance.
- A basic two-color policer normally transmits conforming traffic and drops exceeding traffic by default.
- CoPP can use **bps or pps** policing, depending on platform support.
- **bps policing** limits traffic volume, while **pps policing** limits packet rate.
- A configured policing rate is an average rate with a burst allowance, not necessarily a strict instantaneous maximum.
- Routing protocol classes need sufficient rates and burst allowances for normal operation and reconvergence.
- Avoid aggressively policing **`class-default`** without understanding the traffic it includes.
- Catalyst 9000 System CoPP commonly uses predefined **pps-based policers**.
- Use **`show policy-map control-plane`** to verify policing rates, conform/exceed counters, and drops.

---

## References

- [Cisco IOS XE 17.x - Control Plane Policing](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/qos/b-quality-of-service/m_qos-plcshp-ctrl-pln-plc-0.html)
- [Cisco IOS XE 17.x - QoS Packet Policing](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/qos/b-quality-of-service/m_qos-policing.html)
- [Cisco IOS XE 17.x - Two-Rate Policer](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/qos/b-quality-of-service/m_qos-plcshp-two-rate-plcr-0.html)
- [Cisco - Control Plane Policing Implementation Best Practices](https://www.cisco.com/c/en/us/about/security-center/copp-best-practices.html)
- [Cisco Catalyst 9300 IOS XE 17.17.x - Configuring Control Plane Policing](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9300/software/release/17-17/configuration_guide/sec/b_1717_sec_9300_cg/configuring_control_plane_policing.html)
- [RFC 2697 - A Single Rate Three Color Marker](https://www.rfc-editor.org/rfc/rfc2697)
- [RFC 2698 - A Two Rate Three Color Marker](https://www.rfc-editor.org/rfc/rfc2698)
