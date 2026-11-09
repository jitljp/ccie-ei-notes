# Single-Rate and Two-Rate Policing

Cisco IOS XE supports several traffic policing algorithms that differ in the number of configured rates, token buckets, and possible packet outcomes.

The three main algorithms are:

| Algorithm | Rates | Token buckets | Outcomes |
|---|---|---|---|
| **Single-Rate, Two-Color (1R2C)** | CIR | Bc | Conform, exceed |
| **Single-Rate, Three-Color (1R3C)** | CIR | Bc, Be | Conform, exceed, violate |
| **Two-Rate, Three-Color (2R3C)** | CIR, PIR | Bc, Be | Conform, exceed, violate |

The outcomes are commonly represented by colors:

| Color | Policing outcome | Meaning |
|---|---|---|
| **Green** | Conform | Traffic satisfies the committed profile |
| **Yellow** | Exceed | Traffic exceeds the committed allowance but satisfies an additional allowance |
| **Red** | Violate | Traffic does not satisfy the permitted traffic profile |

The colors are logical classifications, not actual colors stored in packet headers.

The configured policing actions determine what happens to each packet. For example, yellow packets might be remarked and transmitted, while red packets are dropped.

**A packet's policing outcome depends on available tokens and packet size, not simply on whether its instantaneous arrival rate exceeds CIR or PIR.**

---

## 1. Single-Rate, Two-Color Policing (1R2C)

**Single-Rate, Two-Color (1R2C)** policing is the simplest policing algorithm.

It uses:
- One rate: CIR.
- One token bucket: Bc.
- Two possible outcomes: conform and exceed.

### Token Bucket Operation

The committed bucket receives tokens at CIR, up to a maximum capacity of Bc.

```text
                 CIR
                  |
                  v
           +-------------+
           | Committed   |
           | Bucket (Bc) |
           +-------------+
                  |
                  v
            Packet arrives
                  |
             Enough tokens?
              /        \
            Yes         No
             |           |
             v           v
          Conform       Exceed
          (Green)       (Yellow)
```

When a packet arrives:

1. The policer updates the token count based on elapsed time.
2. If the committed bucket contains sufficient tokens, the packet conforms.
3. The packet's accounted size is deducted from the committed bucket.
4. Otherwise, the packet exceeds, and the bucket is not decremented.

The policer then applies the configured action.

### Configuration

Configure R1 to police traffic to 5 Mbps, with Bc = 125,000 bytes:

```text
R1(config)# policy-map POLICE-1R2C
R1(config-pmap)# class class-default
R1(config-pmap-c)# police cir 5000000 bc 125000 conform-action transmit exceed-action drop
```

This configures:

| Parameter | Value |
|---|---|
| CIR | 5 Mbps |
| Bc | 125,000 bytes |
| Conform action | Transmit |
| Exceed action | Drop |

The committed bucket replenishes at:

```text
5,000,000 / 8 = 625,000 bytes/second
```

A full bucket therefore represents 200 ms of traffic at CIR.

### Packet-by-Packet Example

Suppose:
- Bc = 3,000 bytes.
- The committed bucket is initially full.
- Each packet is 1,000 bytes.
- Five packets arrive simultaneously, with negligible replenishment between them.

| Packet | Committed tokens before | Outcome | Committed tokens after |
|---|---:|---|---:|
| 1 | 3,000 | Conform | 2,000 |
| 2 | 2,000 | Conform | 1,000 |
| 3 | 1,000 | Conform | 0 |
| 4 | 0 | Exceed | 0 |
| 5 | 0 | Exceed | 0 |

The first three packets conform because enough tokens are available.

The remaining two packets exceed because the bucket is empty.

With the default policing actions, packets 1 through 3 are transmitted, while packets 4 and 5 are dropped.

**An exceeding packet does not consume tokens in a 1R2C policer.**

---

## 2. Single-Rate, Three-Color Policing (1R3C)

**Single-Rate, Three-Color (1R3C)** policing extends 1R2C by adding an excess token bucket.

It uses:
- One rate: CIR.
- Committed bucket: Bc.
- Excess bucket: Be.
- Three possible outcomes: conform, exceed, and violate.

The key difference is that 1R3C can distinguish between traffic that exceeds the committed allowance and traffic that exceeds the additional burst allowance.

### Token Replenishment

Both buckets share a single replenishment rate: CIR.

However, tokens are not distributed equally between them.

```text
                 CIR
                  |
                  v
           +-------------+
           | Committed   |
           | Bucket (Bc) |
           +-------------+
                  |
             If Bc is full,
             tokens overflow
                  |
                  v
           +-------------+
           | Excess      |
           | Bucket (Be) |
           +-------------+
                  |
             If Be is full,
             tokens discarded
```

The replenishment process is:

1. Tokens are added to the committed bucket at CIR.
2. When the committed bucket is full, additional tokens overflow into the excess bucket.
3. The excess bucket can accumulate tokens up to Be.
4. When both buckets are full, additional tokens are discarded.

The buckets are initially full in Cisco's documented implementation.

**The excess bucket does not have its own replenishment rate. It receives tokens only when the committed bucket is full.**

### Packet Classification

When a packet arrives:

1. Check the committed bucket.
2. If sufficient committed tokens exist, the packet conforms. Deduct tokens from the committed bucket only.
3. Otherwise, check the excess bucket.
4. If sufficient excess tokens exist, the packet exceeds. Deduct tokens from the excess bucket only.
5. Otherwise, the packet violates. Neither bucket is decremented.

| Outcome | Condition | Tokens consumed |
|---|---|---|
| **Conform** | Committed bucket has enough tokens | Committed only |
| **Exceed** | Committed bucket lacks tokens, excess bucket has enough | Excess only |
| **Violate** | Neither bucket has enough tokens | Neither |

Notice that a conforming packet does **not** consume tokens from both buckets.

This differs from 2R3C policing.

### Configuration

Configure a single-rate, three-color policer:

```text
R1(config)# policy-map POLICE-1R3C
R1(config-pmap)# class class-default
R1(config-pmap-c)# police cir 5000000 bc 125000 be 250000 conform-action transmit exceed-action set-dscp-transmit af32 violate-action drop
```

This configures:

| Parameter | Value |
|---|---|
| CIR | 5 Mbps |
| Bc | 125,000 bytes |
| Be | 250,000 bytes |
| Conform | Transmit |
| Exceed | Remark AF32 and transmit |
| Violate | Drop |

### Packet-by-Packet Example

Suppose:
- Bc = 3,000 bytes.
- Be = 2,000 bytes.
- Both buckets are initially full.
- Six 1,000-byte packets arrive simultaneously.

| Packet | Committed tokens before | Excess tokens before | Outcome | Committed after | Excess after |
|---|---:|---:|---|---:|---:|
| 1 | 3,000 | 2,000 | Conform | 2,000 | 2,000 |
| 2 | 2,000 | 2,000 | Conform | 1,000 | 2,000 |
| 3 | 1,000 | 2,000 | Conform | 0 | 2,000 |
| 4 | 0 | 2,000 | Exceed | 0 | 1,000 |
| 5 | 0 | 1,000 | Exceed | 0 | 0 |
| 6 | 0 | 0 | Violate | 0 | 0 |

The first three packets consume committed tokens.

Packets 4 and 5 consume excess tokens.

Packet 6 violates because neither bucket has enough tokens.

With the example configuration:
- Packets 1 through 3 are transmitted unchanged.
- Packets 4 and 5 are remarked to AF32 and transmitted.
- Packet 6 is dropped.

### Why 1R3C Does Not Have a PIR

Suppose the CIR is 5 Mbps, but traffic is continuously offered at 7 Mbps.

Because traffic consumes committed tokens faster than they are replenished, the committed bucket will frequently be depleted.

The excess bucket cannot replenish unless the committed bucket becomes full.

Consequently, the excess bucket can eventually become empty, and the policer will classify some packets as violating.

This is true even though 7 Mbps is only slightly above the CIR.

**1R3C provides additional burst tolerance, not a separately sustained peak-rate allowance.**

The Be value controls how much excess traffic can be accommodated, but it does not establish a second replenishment rate.

---

## 3. Two-Rate, Three-Color Policing (2R3C)

**Two-Rate, Three-Color (2R3C)** policing introduces a second rate, the Peak Information Rate (PIR).

It uses:
- CIR: Committed Information Rate.
- PIR: Peak Information Rate.
- Bc: Committed bucket capacity.
- Be: Peak bucket capacity.
- Three outcomes: conform, exceed, and violate.

Unlike 1R3C, **the two buckets are replenished independently**.

### Token Replenishment

```text
        CIR                    PIR
         |                      |
         v                      v
   +-------------+        +-------------+
   | Committed   |        | Peak        |
   | Bucket (Bc) |        | Bucket (Be) |
   +-------------+        +-------------+
         |                      |
         +----------+-----------+
                    |
                    v
              Packet arrives
```

The committed bucket receives tokens at CIR, up to Bc.

The peak bucket receives tokens at PIR, up to Be.

There is no token overflow between the buckets.

Also, **the peak bucket replenishes at PIR, not at PIR minus CIR**.

For example, if:
- CIR = 5 Mbps.
- PIR = 10 Mbps.

Then:
- Committed tokens replenish at 625,000 bytes/second.
- Peak tokens replenish at 1,250,000 bytes/second.

### Packet Classification

A 2R3C policer evaluates the **peak bucket first**.

1. Check whether the peak bucket contains enough tokens for the packet.
2. If not, the packet violates. Neither bucket is decremented.
3. If the peak bucket has enough tokens, check the committed bucket.
4. If the committed bucket lacks tokens, the packet exceeds. Deduct tokens from the peak bucket only.
5. If both buckets have enough tokens, the packet conforms. Deduct tokens from both buckets.

| Outcome | Committed bucket | Peak bucket | Tokens consumed |
|---|---|---|---|
| **Conform** | Sufficient | Sufficient | Both buckets |
| **Exceed** | Insufficient | Sufficient | Peak only |
| **Violate** | Any | Insufficient | Neither |

Two important distinctions:
- A conforming packet consumes tokens from **both** buckets.
- A packet violates when peak tokens are insufficient, even if the committed bucket has sufficient tokens.

### Configuration

Configure a two-rate, three-color policer:

```text
R1(config)# policy-map POLICE-2R3C
R1(config-pmap)# class class-default
R1(config-pmap-c)# police cir 5000000 bc 125000 pir 10000000 be 250000 conform-action transmit exceed-action set-dscp-transmit af32 violate-action drop
```

This configures:

| Parameter | Value |
|---|---|
| CIR | 5 Mbps |
| PIR | 10 Mbps |
| Bc | 125,000 bytes |
| Be | 250,000 bytes |
| Conform | Transmit |
| Exceed | Remark AF32 and transmit |
| Violate | Drop |

For Cisco's documented 2R3C implementation, PIR must be greater than CIR.

### Packet-by-Packet Example

Suppose:
- Bc = 3,000 bytes.
- Be = 5,000 bytes.
- Both buckets are initially full.
- Six 1,000-byte packets arrive simultaneously.

| Packet | Committed tokens before | Peak tokens before | Outcome | Committed after | Peak after |
|---|---:|---:|---|---:|---:|
| 1 | 3,000 | 5,000 | Conform | 2,000 | 4,000 |
| 2 | 2,000 | 4,000 | Conform | 1,000 | 3,000 |
| 3 | 1,000 | 3,000 | Conform | 0 | 2,000 |
| 4 | 0 | 2,000 | Exceed | 0 | 1,000 |
| 5 | 0 | 1,000 | Exceed | 0 | 0 |
| 6 | 0 | 0 | Violate | 0 | 0 |

The first three packets consume tokens from both buckets.

Packets 4 and 5 consume only peak tokens.

Packet 6 violates because the peak bucket is empty.

The packet outcomes happen to match the previous 1R3C example, but the token accounting is different.

The difference becomes more apparent when time is allowed for token replenishment.

---

## 4. Comparing Token Replenishment

Consider a 1R3C and 2R3C policer with the following parameters:

| Parameter | 1R3C | 2R3C |
|---|---:|---:|
| CIR | 8 kbps | 8 kbps |
| PIR | Not configured | 16 kbps |
| Bc | 3,000 bytes | 3,000 bytes |
| Be | 5,000 bytes | 5,000 bytes |

Assume both token buckets have been completely emptied by previous traffic.

No packets arrive for one second.

### 1R3C

The committed bucket receives:

```text
8,000 bps × 1 second / 8 = 1,000 bytes
```

Because the committed bucket is not full, the excess bucket receives no tokens.

After one second:

```text
Committed = 1,000 bytes
Excess    = 0 bytes
```

### 2R3C

The committed bucket receives 1,000 bytes, as before.

The peak bucket independently receives:

```text
16,000 bps × 1 second / 8 = 2,000 bytes
```

After one second:

```text
Committed = 1,000 bytes
Peak      = 2,000 bytes
```

### A 1,500-Byte Packet Arrives

| Policer | Committed tokens | Excess/peak tokens | Outcome |
|---|---:|---:|---|
| 1R3C | 1,000 | 0 | Violate |
| 2R3C | 1,000 | 2,000 | Exceed |

The 1R3C policer violates the packet because neither bucket has enough tokens.

The 2R3C policer exceeds the packet because the peak bucket has enough tokens, even though the committed bucket does not.

**This illustrates the fundamental difference between the algorithms: 1R3C replenishes the excess bucket using committed-token overflow, while 2R3C independently replenishes the peak bucket at PIR.**

---

## 5. Sustained Traffic Behavior

The differences between 1R3C and 2R3C are particularly important for sustained traffic.

Consider:
- CIR = 5 Mbps.
- Offered traffic = 7 Mbps.
- PIR = 10 Mbps, when applicable.
- All policing actions are configured to transmit conforming and exceeding packets, but drop violating packets.

Assume a sufficiently long period of steady traffic and burst allowances large enough to avoid unusual packet-size effects.

### 1R3C

For traffic continuously offered at 7 Mbps:
- Committed tokens replenish at 5 Mbps.
- The traffic consumes committed tokens faster than they are replenished.
- The committed bucket frequently empties.
- The excess bucket cannot replenish unless the committed bucket becomes full.
- The excess bucket eventually empties.
- Some traffic is classified as violating and dropped.

The policer therefore cannot sustain the additional 2 Mbps entirely as exceeding traffic.

### 2R3C

For traffic continuously offered at 7 Mbps:
- Committed tokens replenish at 5 Mbps.
- Peak tokens replenish at 10 Mbps.
- Approximately 5 Mbps can conform.
- Approximately 2 Mbps can exceed.
- Under steady-state conditions, traffic can remain within the peak profile without sustained violations.

The key difference is:

| Offered traffic | 1R3C | 2R3C |
|---|---|---|
| Below CIR | Generally conforms | Generally conforms |
| Sustained traffic between CIR and PIR | Excess allowance eventually depletes; violations can occur | Can be classified as conforming and exceeding without sustained violations |
| Sustained traffic above PIR | No separate PIR threshold | Violations occur when peak tokens are insufficient |

These outcomes describe sustained traffic behavior. Individual packets can still receive different classifications depending on token availability.

---

## 6. Color-Blind vs. Color-Aware Policing

Three-color policers can operate in **color-blind** or **color-aware** mode.

The difference is whether an existing packet color influences the new policing decision.

### Color-Blind Policing

A color-blind policer ignores any previous conform, exceed, or violate designation.

Each packet is evaluated entirely according to the policer's current token availability.

For example, suppose an upstream router has remarked traffic:
- AF31 = Conforming traffic.
- AF32 = Exceeding traffic.
- AF33 = Violating traffic.

A downstream color-blind policer does not automatically treat these DSCP values as policing colors.

If the downstream policer has sufficient tokens, an AF32 packet can be classified as conforming.

This means a packet previously designated as exceeding can effectively be promoted to conforming.

### Color-Aware Policing

A **color-aware policer** considers the packet's existing color.

A packet previously classified as exceeding cannot be promoted to conforming.

Likewise, a packet previously classified as violating remains violating.

| Incoming color | Possible outgoing colors |
|---|---|
| Green (conform) | Green, yellow, red |
| Yellow (exceed) | Yellow, red |
| Red (violate) | Red only |

Color-aware policing therefore preserves or worsens the existing traffic-profile designation.

However, the colors must be explicitly identified through supported classification mechanisms. A policer does not inherently know that AF31 represents green or AF32 represents yellow.

### Color-Aware 1R3C

In color-aware 1R3C:

**Incoming green packet:**
1. Check the committed bucket.
2. If sufficient tokens exist, conform and deduct committed tokens.
3. Otherwise, check the excess bucket.
4. If sufficient excess tokens exist, exceed and deduct excess tokens.
5. Otherwise, violate.

**Incoming yellow packet:**
1. Skip the committed bucket.
2. Check the excess bucket.
3. If sufficient tokens exist, remain exceeding and deduct excess tokens.
4. Otherwise, violate.

**Incoming red packet:**
- Immediately violate without consuming tokens.

### Color-Aware 2R3C

In color-aware 2R3C:

**Incoming green packet:**
1. Check the peak bucket.
2. If peak tokens are insufficient, violate.
3. Otherwise, check the committed bucket.
4. If committed tokens are insufficient, exceed and deduct peak tokens.
5. If both buckets have sufficient tokens, conform and deduct from both.

**Incoming yellow packet:**
1. Skip the committed bucket.
2. Check the peak bucket.
3. If sufficient tokens exist, remain exceeding and deduct peak tokens.
4. Otherwise, violate.

**Incoming red packet:**
- Immediately violate without consuming tokens.

The fundamental rule is that **color-aware policing cannot upgrade a packet to a more favorable color**.

### Color-Aware Policing Configuration

On supported IOS XE router platforms, color-aware policing uses the following commands in police configuration mode:

```text
R1(config-pmap-c-police)# conform-color PRE-CONFORM
R1(config-pmap-c-police)# exceed-color PRE-EXCEED
```

These commands reference class maps that identify packets already designated as conforming or exceeding.

For example, on ASR 1000, Cisco documents color-aware class maps based on `match qos-group`, with a child policy used to derive the QoS group from the packet's DSCP value.

Conceptually:

```text
DSCP AF31 --> QoS group 1 --> Green
DSCP AF32 --> QoS group 2 --> Yellow
Other      --> No match    --> Red
```

The color-aware policer then considers those existing designations when evaluating token availability.

These capabilities and the required policy hierarchy are platform-dependent.

**Color-aware policing is distinct from simply matching a DSCP value in a class map and applying an ordinary color-blind policer.**

---

## 7. Policing Actions and DSCP Markdown

Each policing outcome can have a separate action.

A common design uses Assured Forwarding markings to distinguish conforming, exceeding, and violating traffic.

For example:

| Outcome | Action | Result |
|---|---|---|
| Conform | `transmit` | Forward without changing DSCP |
| Exceed | `set-dscp-transmit af32` | Remark AF32 and forward |
| Violate | `drop` | Discard |

If the traffic originally has DSCP AF31:
- Conforming packets retain AF31.
- Exceeding packets are remarked AF32.
- Violating packets are dropped.

A different policy could instead use:

```text
R1(config-pmap-c-police)# conform-action set-dscp-transmit af31
R1(config-pmap-c-police)# exceed-action set-dscp-transmit af32
R1(config-pmap-c-police)# violate-action set-dscp-transmit af33
```

This transmits all three colors, assigning AF31, AF32, or AF33 according to the policing outcome.

The downstream network can then apply WRED to give packets with higher drop precedence a greater probability of being dropped during congestion.

**Conform, exceed, and violate describe metering outcomes. They do not inherently mean transmit, remark, and drop.**

The configured actions determine the packet's treatment.

See [Traffic Policing](policing.md) for more information about policing actions.

---

## 8. Configuring the Three Policing Algorithms

The following examples use the same CIR, permitting direct comparison.

### 1R2C

```text
R1(config)# policy-map POLICE-1R2C
R1(config-pmap)# class class-default
R1(config-pmap-c)# police cir 5000000 bc 125000 conform-action transmit exceed-action drop
```

One bucket, two outcomes.

### 1R3C

```text
R1(config)# policy-map POLICE-1R3C
R1(config-pmap)# class class-default
R1(config-pmap-c)# police cir 5000000 bc 125000 be 250000 conform-action transmit exceed-action set-dscp-transmit af32 violate-action drop
```

Two buckets, one shared replenishment rate, three outcomes.

### 2R3C

```text
R1(config)# policy-map POLICE-2R3C
R1(config-pmap)# class class-default
R1(config-pmap-c)# police cir 5000000 bc 125000 pir 10000000 be 250000 conform-action transmit exceed-action set-dscp-transmit af32 violate-action drop
```

Two independently replenished buckets, two rates, three outcomes.

### Applying a Policing Policy

For example, to apply the 2R3C policy inbound:

```text
R1(config)# interface GigabitEthernet1
R1(config-if)# service-policy input POLICE-2R3C
```

The policy is applied to traffic entering GigabitEthernet1.

Only one of the example policies should be applied at this interface attachment point at a time.

The supported algorithms, actions, and attachment directions depend on the IOS XE platform.

---

## 9. Verification

Use `show policy-map interface` to verify policing configuration and statistics.

For example:

```text
R1# show policy-map interface GigabitEthernet1
```

For a 2R3C policer, the output can include:

```text
police:
    cir 5000000 bps, bc 125000 bytes
    pir 10000000 bps, be 250000 bytes
  conformed 10000 packets, 15000000 bytes; actions:
    transmit
  exceeded 4000 packets, 6000000 bytes; actions:
    set-dscp-transmit af32
  violated 1000 packets, 1500000 bytes; actions:
    drop
```

This illustrative output shows:
- 10,000 packets conformed and were transmitted.
- 4,000 packets exceeded, were remarked AF32, and were transmitted.
- 1,000 packets violated and were dropped.

An exceeded or violated counter does not necessarily represent dropped traffic.

Always inspect the configured action.

### Additional Commands

| Command | Purpose |
|---|---|
| `show policy-map` | Display configured policing parameters |
| `show policy-map interface GigabitEthernet1` | Verify actual policing outcomes and actions |
| `show policy-map interface GigabitEthernet1 input` | Verify ingress policing |
| `show running-config interface GigabitEthernet1` | Confirm the applied policy |
| `show platform hardware qfp active feature qos interface GigabitEthernet1` | Inspect QFP hardware policing information on supported routers |

### Hardware Implementation Differences

On some QFP-based Cisco routers, the forwarding hardware internally represents even a 1R2C policer using three-color machinery.

The hardware can configure the exceed and violate actions identically to implement two-color behavior.

Consequently, hardware counters or internal structures may show three policing outcomes even when the IOS XE policy is configured as 1R2C.

The normal `show policy-map interface` command presents the logical policing behavior.

Hardware implementations may also round configured CIR, PIR, Bc, and Be values according to supported hardware granularity.

---

## 10. Common Misconceptions

| Misconception | Correct behavior |
|---|---|
| A packet above CIR always exceeds | It can conform if sufficient committed tokens exist |
| A packet above PIR always violates | It can conform or exceed if sufficient tokens exist |
| Be is replenished at PIR in every three-color policer | Only 2R3C independently replenishes its peak bucket at PIR |
| In 2R3C, the peak bucket replenishes at PIR minus CIR | The peak bucket independently replenishes at PIR |
| Conforming packets consume tokens from both buckets in 1R3C | They consume committed tokens only |
| Conforming packets consume tokens from only one bucket in 2R3C | They consume both committed and peak tokens |
| A violating packet always drops | The configured violate action determines its treatment |
| An AF32 packet is automatically considered yellow | Color-aware mapping must explicitly identify its existing designation |
| 1R3C guarantees sustained excess traffic below a peak rate | 1R3C has no separate PIR |
| A red packet cannot be transmitted | Red means violation; a transmit action may still be configured |

---

## Key Points

- **1R2C:** One rate, one bucket, two outcomes.
- **1R3C:** One rate, two buckets, three outcomes.
- **2R3C:** Two rates, two independently replenished buckets, three outcomes.
- In 1R2C, conforming packets consume committed tokens; exceeding packets do not consume tokens.
- In 1R3C, committed tokens are replenished first, and only overflow into the excess bucket when the committed bucket is full.
- In 1R3C, conforming packets consume committed tokens only; exceeding packets consume excess tokens only.
- In 2R3C, the committed bucket replenishes at CIR and the peak bucket replenishes independently at PIR.
- In 2R3C, conforming packets consume tokens from both buckets; exceeding packets consume peak tokens only.
- Violating packets do not consume tokens from either bucket in these color-blind algorithms.
- **Color-blind** policing ignores existing packet color.
- **Color-aware** policing preserves or worsens existing packet color, never improves it.
- Policing outcomes do not automatically determine forwarding actions.
- `show policy-map interface` is the primary verification command.

---

## References

- [Cisco IOS XE 17.x: QoS Packet Policing](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/qos/b-quality-of-service/m_qos-policing.html)
- [Cisco IOS XE 17.x: Two-Rate Policer](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/qos/b-quality-of-service/m_qos-plcshp-two-rate-plcr-0.html)
- [RFC 2697: A Single Rate Three Color Marker](https://www.rfc-editor.org/rfc/rfc2697)
- [RFC 2698: A Two Rate Three Color Marker](https://www.rfc-editor.org/rfc/rfc2698)