# Token Bucket Fundamentals (CIR, PIR, Bc, Be)

A **token bucket** is a mechanism used by traffic policers and shapers to regulate traffic rates while permitting controlled bursts.

Rather than enforcing a strict instantaneous transmission rate, a token bucket allows traffic to exceed a configured rate temporarily, provided sufficient tokens are available.

The fundamental components are:
- **CIR (Committed Information Rate):** The committed rate at which tokens are replenished.
- **PIR (Peak Information Rate):** An additional rate used by two-rate policing.
- **Bc (Committed Burst):** The maximum committed burst allowance.
- **Be (Excess Burst):** An additional burst allowance, whose behavior depends on the policing or shaping algorithm.
- **Tc (Committed Time Interval):** A time interval used to relate the committed rate to a burst size, particularly for shaping.

Token buckets are used by both **traffic policing** and **traffic shaping**, but the actions taken when insufficient tokens are available differ.

---

## Token Bucket Operation

A token bucket can be visualized as a container that accumulates tokens over time.

Each token represents permission to process a certain amount of traffic.

For example, consider a policer with:
- CIR = 1 Mbps.
- Bc = 12,500 bytes.

```text
               CIR = 1 Mbps
              (125,000 B/s)
                     |
                     v
              +-------------+
              |             |
              |   Tokens    | Bc = 12,500 bytes
              |             |
              +-------------+
                     |
                     v
              Packet arrives
                     |
              Enough tokens?
                /         \
              Yes          No
               |            |
               v            v
            Conform       Exceed
```

The process is:

1. Tokens are replenished at the configured CIR.
2. The bucket can accumulate tokens only up to Bc.
3. When a packet arrives, its accounted size is compared to the available tokens.
4. If enough tokens exist, the packet conforms and consumes tokens.
5. If insufficient tokens exist, the packet exceeds the committed allowance.

For a basic single-rate, two-color policer, exceeding packets do not consume tokens.

Importantly, **tokens accumulate even when no packets are arriving**, up to the bucket's maximum capacity.

Tokens are logically replenished continuously at the CIR. In Cisco's class-based policing implementation, the bucket is actually updated when a packet arrives, based on the time elapsed since the previous packet.

### Token Replenishment

The amount of tokens replenished over a particular period is calculated as:

```text
Tokens (bits) = CIR × elapsed time (seconds)
```

When tokens represent bytes:

```text
Tokens (bytes) = CIR × elapsed time / 8
```

For a CIR of 1 Mbps:

| Elapsed time | Tokens replenished |
|---|---:|
| 1 ms | 125 bytes |
| 10 ms | 1,250 bytes |
| 20 ms | 2,500 bytes |
| 100 ms | 12,500 bytes |
| 1 second | 125,000 bytes |

These values assume there is sufficient room in the bucket.

When the bucket is full, additional tokens cannot be stored. They are discarded rather than accumulating indefinitely.

---

## Committed Information Rate (CIR)

**CIR** specifies the committed token replenishment rate.

For policing, it establishes the rate at which the policer receives additional committed tokens.

For shaping, it normally represents the configured average transmission rate.

CIR is expressed in **bits per second (bps)**.

For example:

```text
R1(config-pmap-c)# police cir 10000000
```

Configures a CIR of 10 Mbps.

Similarly:

```text
R1(config-pmap-c)# shape average 10000000
```

Configures average shaping to 10 Mbps.

### CIR Is Not an Instantaneous Rate Limit

A common misconception is that packets exceeding the CIR are always dropped or delayed.

Consider a token bucket with:
- CIR = 10 Mbps.
- Bc = 125,000 bytes.

If the bucket is full, the device can initially process a burst of 125,000 bytes without waiting for additional tokens.

The burst may be transmitted at a much higher instantaneous rate than 10 Mbps.

However, sustained traffic cannot continue consuming tokens faster than they are replenished without eventually exhausting the bucket.

**CIR controls the long-term committed rate, while Bc determines how much burstiness is tolerated.**

---

## Committed Burst (Bc)

**Bc** specifies the committed burst allowance.

In a basic token bucket, Bc determines the maximum number of tokens that can accumulate in the committed bucket.

A larger Bc permits larger short-term bursts without exceeding the committed traffic profile.

A smaller Bc makes the traffic regulator less tolerant of bursts.

Consider two policers with identical CIR values:

| Parameter | Policer A | Policer B |
|---|---:|---:|
| CIR | 10 Mbps | 10 Mbps |
| Bc | 12,500 bytes | 125,000 bytes |
| Maximum committed tokens | 12,500 | 125,000 |

Both policers replenish tokens at the same rate.

However, assuming their buckets are full:
- Policer A permits approximately 12.5 KB of immediately arriving traffic to conform.
- Policer B permits approximately 125 KB to conform.

Policer B therefore allows larger bursts, even though both have the same CIR.

### Packet-by-Packet Example

Consider a single-rate, two-color policer with:
- CIR = 1 Mbps.
- Bc = 12,500 bytes.
- Initial tokens = 12,500 bytes.
- Packet size = 1,500 bytes.

Assume all packets arrive simultaneously, so no meaningful replenishment occurs between them.

| Packet | Tokens before | Result | Tokens after |
|---|---:|---|---:|
| 1 | 12,500 | Conform | 11,000 |
| 2 | 11,000 | Conform | 9,500 |
| 3 | 9,500 | Conform | 8,000 |
| 4 | 8,000 | Conform | 6,500 |
| 5 | 6,500 | Conform | 5,000 |
| 6 | 5,000 | Conform | 3,500 |
| 7 | 3,500 | Conform | 2,000 |
| 8 | 2,000 | Conform | 500 |
| 9 | 500 | Exceed | 500 |

The first eight packets consume 12,000 bytes of tokens.

The ninth packet requires 1,500 bytes, but only 500 remain.

Therefore, the ninth packet exceeds the committed allowance.

If the policer is configured to drop exceeding packets, packet 9 is dropped.

### Token Recovery

After packet 9, the bucket contains 500 bytes.

To accept another 1,500-byte packet, it needs 1,000 additional tokens.

At 1 Mbps:

```text
1 Mbps = 125,000 bytes/second

Time = 1,000 / 125,000
     = 0.008 seconds
     = 8 ms
```

After approximately 8 ms without further token consumption, sufficient tokens are available for another 1,500-byte packet.

This example assumes the stated packet sizes are the sizes accounted for by the policer. Actual accounting can include different protocol overheads depending on the platform and configuration.

---

## Committed Time Interval (Tc)

**Tc** represents the time interval associated with the committed burst.

It is especially important when studying traffic shaping.

The fundamental relationship is:

```text
Bc = CIR × Tc
```

This assumes Bc is expressed in bits.

Rearranging:

```text
Tc = Bc / CIR

CIR = Bc / Tc
```

If Bc is expressed in bytes:

```text
Bc (bytes) = CIR × Tc / 8

Tc = (Bc × 8) / CIR

CIR = (Bc × 8) / Tc
```

### Example: Calculate Bc

Given:
- CIR = 10 Mbps.
- Tc = 20 ms.

Calculate Bc:

```text
Bc = CIR × Tc
   = 10,000,000 × 0.020
   = 200,000 bits
   = 25,000 bytes
```

### Example: Calculate Tc

Given:
- CIR = 8 Mbps.
- Bc = 20,000 bytes.

Calculate Tc:

```text
Tc = (Bc × 8) / CIR
   = (20,000 × 8) / 8,000,000
   = 0.020 seconds
   = 20 ms
```

### Example: Calculate CIR

Given:
- Bc = 50,000 bytes.
- Tc = 40 ms.

Calculate CIR:

```text
CIR = (Bc × 8) / Tc
    = (50,000 × 8) / 0.040
    = 10,000,000 bps
    = 10 Mbps
```

### Tc and Shaping Behavior

In classic interval-based shaping, the shaper obtains a transmission allowance corresponding to Bc during each Tc.

For example:

```text
CIR = 10 Mbps
Tc  = 20 ms
Bc  = 200,000 bits
```

The shaper can schedule approximately 200,000 bits during each 20 ms interval, subject to the implementation's scheduling and burst rules.

A shorter Tc generally produces smaller transmission bursts and smoother output.

A longer Tc permits larger bursts, but can also increase delay variation.

| Tc | Bc at 10 Mbps |
|---|---:|
| 10 ms | 100,000 bits |
| 20 ms | 200,000 bits |
| 25 ms | 250,000 bits |
| 50 ms | 500,000 bits |
| 100 ms | 1,000,000 bits |

Modern hardware shaping implementations may use continuously updated credits or hardware scheduling rather than literally transmitting once per Tc.

**Tc is useful for understanding burst calculations, but it does not necessarily represent a timer that fires once for every transmission interval.**

---

## Excess Burst (Be)

**Be** provides an additional burst allowance beyond Bc.

Its precise meaning depends on the algorithm.

| Mechanism | Role of Be |
|---|---|
| Single-rate three-color policing | Capacity of an excess token bucket, replenished when the committed bucket is full |
| Two-rate three-color policing | Capacity of the peak-rate token bucket, replenished independently at PIR |
| Classic traffic shaping | Additional burst capacity that can permit greater bursts after unused transmission opportunities |

Therefore, **Be does not have identical behavior in all token bucket implementations**.

In particular, Be should not automatically be interpreted as the number of bytes permitted above CIR during every time interval.

### Bc vs. Be

For single-rate three-color policing:
- Bc represents the committed token bucket capacity.
- Be represents the additional excess token bucket capacity.
- Bc is replenished at the CIR, with excess tokens entering the Be bucket when the Bc bucket is full.

For two-rate three-color policing:
- Bc represents the committed token bucket capacity, associated with CIR.
- Be represents the peak token bucket capacity, associated with PIR.
- Each bucket is replenished independently.

These distinctions determine whether packets conform, exceed, or violate.

---

## Single-Rate, Two-Color Token Bucket (1R2C)

The simplest policing implementation uses:
- One rate: CIR.
- One bucket: Bc.
- Two outcomes: conform and exceed.

```text
                   CIR
                    |
                    v
              +-----------+
              |    Bc     |
              |   Bucket  |
              +-----------+
                    |
                    v
             Packet arrives
                    |
              Enough tokens?
                 /      \
               Yes       No
                |         |
                v         v
             Conform    Exceed
```

When a packet arrives:

1. The policer replenishes tokens according to elapsed time, subject to Bc.
2. If enough tokens exist, the packet conforms and tokens are consumed.
3. Otherwise, the packet exceeds and the bucket is not decremented.

The policer then applies the appropriate conform or exceed action.

For example:

```text
R1(config-pmap-c)# police cir 10000000 bc 125000 conform-action transmit exceed-action drop
```

This configures:
- CIR = 10 Mbps.
- Bc = 125,000 bytes.
- Conform action = transmit.
- Exceed action = drop.

The configured Bc represents 100 ms of traffic at the CIR.

**1R2C uses only Bc. It does not require Be or PIR.**

---

## Single-Rate, Three-Color Token Bucket (1R3C)

A **single-rate, three-color policer** uses two token buckets:
- Committed bucket, with capacity Bc.
- Excess bucket, with capacity Be.

However, both buckets share a single token replenishment rate: CIR.

```text
                       CIR
                        |
                        v
                 +-------------+
                 | Committed   |
                 | Bucket (Bc) |
                 +-------------+
                        |
                   If full,
                   overflow
                        |
                        v
                 +-------------+
                 | Excess      |
                 | Bucket (Be) |
                 +-------------+
```

Tokens are added according to the CIR:
1. The committed bucket is replenished first.
2. If the committed bucket is full, new tokens can be added to the excess bucket.
3. If both buckets are full, additional tokens are discarded.

When a packet arrives:

| Condition | Outcome | Tokens consumed |
|---|---|---|
| Committed bucket has enough tokens | Conform | Committed bucket |
| Committed bucket lacks tokens, but excess bucket has enough | Exceed | Excess bucket |
| Neither bucket has enough tokens | Violate | Neither bucket |

### Important Behavior

The excess bucket does **not** have an independent replenishment rate.

It receives additional tokens only when the committed bucket is full.

Therefore, a sustained traffic stream exceeding CIR can eventually exhaust the excess bucket.

Afterward, excess traffic can be classified as violating rather than exceeding, even though the traffic rate has not changed.

This is an important distinction from two-rate policing.

See [Single-Rate and Two-Rate Policing](single-two-rate.md) for detailed packet-by-packet operation.

---

## Peak Information Rate (PIR)

**PIR** is an additional rate parameter used in two-rate policing.

It establishes a peak traffic profile alongside the committed traffic profile.

In a two-rate, three-color policer:
- CIR determines committed token replenishment.
- PIR determines peak token replenishment.

Normally, PIR is configured higher than CIR.

For example:
- CIR = 5 Mbps.
- PIR = 8 Mbps.

For sustained traffic, the intended classifications are:

| Traffic rate | General outcome |
|---|---|
| Below 5 Mbps | Conform |
| Between 5 and 8 Mbps | Mix of conform and exceed |
| Above 8 Mbps | Mix of outcomes, including violate |

These are long-term interpretations, not unconditional per-packet rules.

A short burst above PIR can still conform or exceed if tokens are available.

Likewise, a packet can violate when the peak bucket lacks sufficient tokens, even if a short-term rate calculation alone suggests otherwise.

---

## Two-Rate, Three-Color Token Bucket (2R3C)

A **two-rate, three-color policer** uses:
- CIR and Bc for the committed bucket.
- PIR and Be for the peak bucket.

Unlike 1R3C, the two buckets are replenished **independently**.

```text
       CIR                         PIR
        |                           |
        v                           v
  +-------------+             +-------------+
  | Committed   |             | Peak        |
  | Bucket (Bc) |             | Bucket (Be) |
  +-------------+             +-------------+
        |                           |
        +-------------+-------------+
                      |
                      v
                 Packet test
                      |
               Conform / Exceed
                  / Violate
```

For color-blind policing, the packet evaluation is:

1. Check the peak bucket.
2. If the peak bucket has insufficient tokens, the packet violates.
3. Otherwise, check the committed bucket.
4. If the committed bucket has insufficient tokens, the packet exceeds and consumes peak tokens only.
5. If both buckets have enough tokens, the packet conforms and consumes tokens from both buckets.

| Outcome | Committed tokens sufficient? | Peak tokens sufficient? | Tokens consumed |
|---|---|---|---|
| Conform | Yes | Yes | Both buckets |
| Exceed | No | Yes | Peak bucket only |
| Violate | Either