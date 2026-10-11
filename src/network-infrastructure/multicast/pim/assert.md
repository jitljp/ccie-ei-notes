# PIM Assert

**PIM Assert** is a mechanism used to prevent duplicate multicast forwarding onto a shared multiaccess network, such as an Ethernet LAN.

When multiple PIM routers forward the same multicast traffic onto a shared segment, an **Assert election** determines which router is responsible for forwarding that traffic.

The winning router continues forwarding, while the losing routers stop forwarding the affected multicast traffic onto that segment.

PIM Assert is used by both **PIM Sparse Mode (PIM-SM)** and **PIM Dense Mode (PIM-DM)**.

> **Important:** PIM Assert does not prevent the initial duplicate packets. The duplication triggers the Assert process, which then elects a single forwarder.

## Why PIM Assert Is Necessary

Consider the following topology:

```text
             Multicast Source
               10.10.10.10
                    |
                 Network
                 /     \
                /       \
               R1       R2
                \       /
                 \     /
                   SW1
                    |
                    R3
                    |
                 Receiver
                239.1.1.1
```

R1 and R2 both have multicast forwarding state for:

```text
(10.10.10.10, 239.1.1.1)
```

Both routers have their interfaces toward SW1 in their Outgoing Interface Lists (OILs).

Consequently, both forward multicast packets onto the shared LAN.

```text
R1 → SW1 → R3
R2 → SW1 → R3
```

R3 may receive duplicate copies of the same multicast packet.

Furthermore, because R1 and R2 share the Ethernet segment, each can receive the multicast packets forwarded by the other.

PIM Assert resolves this forwarding conflict by electing one router to continue forwarding the traffic onto the LAN.

For example:

```text
Before PIM Assert:

R1 → SW1
R2 → SW1

Both forward multicast traffic.


After PIM Assert:

R1 → SW1    (Assert Winner)
R2   SW1    (Assert Loser, stops forwarding)
```

The Assert election is specific to the affected multicast forwarding state and interface. A router can be the Assert winner for one multicast stream and the loser for another.

## How PIM Assert Is Triggered

The typical trigger for a PIM Assert election is a router receiving a multicast packet on an interface through which it would normally **forward that same multicast traffic**.

In other words, the packet arrives on an interface in the router's OIL rather than its expected incoming RPF interface.

For example:

```text
R1:

RPF Interface: Ethernet0/0
OIL:           Ethernet0/1 (toward SW1)
```

R1 is forwarding multicast packets from Ethernet0/0 out Ethernet0/1.

However, R2 is also forwarding the same traffic onto SW1.

When R1 receives a multicast packet from R2 on Ethernet0/1, it detects the forwarding conflict.

```text
Packet received on Ethernet0/1
           |
           v
Ethernet0/1 is in OIL
           |
           v
Potential duplicate forwarding
           |
           v
Send PIM Assert
```

R2 can detect the same condition and send its own Assert.

The routers then compare their advertised Assert metrics to determine the winner.

**An RPF failure and a PIM Assert are not the same thing.** A multicast packet arriving on a non-RPF interface may trigger special PIM Assert processing when that interface is an outgoing interface for the affected traffic.

PIM Assert messages can also be transmitted in response to Assert messages received from other routers.

## PIM Assert Message

PIM Assert messages use the following protocol information:

| Field | IPv4 | IPv6 |
|---|---|---|
| Destination IP | `224.0.0.13` | `FF02::D` |
| IP Protocol / Next Header | 103 | 103 |
| TTL / Hop Limit | 1 | 1 |
| PIM Version | 2 | 2 |
| Message Type | 5 (Assert) | 5 (Assert) |

An Assert message includes:

- **Multicast Group Address (G)**
- **Source Address (S)**
- **RPT-bit**
- **Metric Preference**
- **Route Metric**

The Assert message format is:

```text
  0                   1                   2                   3
  0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 | Ver   |  Type |   Reserved    |           Checksum            |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 |                  Group Address (encoded)                      |
 |                             ...                               |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 |                  Source Address (encoded)                     |
 |                             ...                               |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 |R|                     Metric Preference                       |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 |                            Metric                             |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

The Group and Source Address fields use variable-length PIM address encodings.

The **RPT-bit (R)** occupies the most significant bit of the 32-bit Metric Preference field, leaving 31 bits for the metric preference itself.

## PIM Assert Election

The Assert winner is selected using the following criteria, in order:

1. **RPT-bit:** Prefer an Assert with RPT-bit `0` over one with RPT-bit `1`.
2. **Metric Preference:** Prefer the lowest value.
3. **Route Metric:** Prefer the lowest value.
4. **IP Address:** If everything else is equal, prefer the highest IP address.

The RPT-bit has the highest priority in PIM-SM Assert elections.

### RPT-bit

The RPT-bit indicates whether the router's Assert is based on source-tree or shared-tree forwarding state.

| RPT-bit | Assert Type | Preference |
|---|---|---|
| 0 | `(S,G)` / SPT | Higher |
| 1 | `(*,G)` / RPT | Lower |

Therefore, an Assert originating from active SPT forwarding state wins over an Assert originating from shared-tree forwarding state, **regardless of the remaining metric values**.

### Metric Preference

The Metric Preference is the routing preference associated with the RPF route.

This corresponds to the route's **administrative distance (AD)**.

For example:

```text
R1: OSPF, AD 110
R2: EIGRP internal, AD 90
```

Assuming both routers advertise the same RPT-bit:

```text
R2 wins (lower Metric Preference).
```

The Metric Preference is compared before the actual route metric.

### Route Metric

If the RPT-bits and Metric Preferences are equal, the routers compare their RPF route metrics.

For example:

```text
R1: OSPF, AD 110, metric 20
R2: OSPF, AD 110, metric 10
```

R2 wins because it has the lower metric.

For `(S,G)` Asserts, the metric is calculated toward the multicast source. For `(*,G)` Asserts, it is calculated toward the RP.

### IP Address Tie-Breaker

If all previous criteria are equal, the router with the highest IP address on the shared interface wins.

For example:

```text
R1: 10.0.12.1
R2: 10.0.12.2
```

If the Assert metrics are otherwise identical, R2 wins.

## Example 1: Assert Winner Selected by Route Metric

Consider the following topology:

```text
               Source
            10.10.10.10
                  |
                 R4
                /  \
               /    \
              R1    R2
               \    /
                \  /
                 SW1
                  |
                  R3
                  |
               Receiver
              239.1.1.1
```

R1 and R2 both forward traffic for `(10.10.10.10, 239.1.1.1)` onto SW1 using active `(S,G)` forwarding state.

Suppose their RPF information is:

| Attribute | R1 | R2 |
|---|---|---|
| RPT-bit | 0 | 0 |
| Routing Protocol | OSPF | OSPF |
| Metric Preference | 110 | 110 |
| Route Metric | 20 | 30 |
| LAN IP Address | 10.0.12.1 | 10.0.12.2 |

The comparison is:

1. RPT-bit: Equal (`0`).
2. Metric Preference: Equal (`110`).
3. Route Metric: R1 wins (`20` is lower than `30`).
4. IP Address: Not considered.

The result is:

```text
R1: Assert Winner
R2: Assert Loser
```

R1 continues forwarding traffic onto SW1.

R2 suppresses forwarding for the affected `(S,G)` state on the shared segment.

Notice that R2 has a higher LAN IP address, but that does not matter because the election was already decided by the route metric.

## Example 2: SPT vs. RPT Assert

An especially important situation occurs when multicast traffic reaches a shared LAN through both an SPT and an RPT.

Consider:

```text
       Source (S)                RP
      10.10.10.10             1.1.1.1
           |                     |
           R1                    R2
            \                    /
             \                  /
                    SW1
                     |
                     R3
                     |
                  Receiver
                 239.1.1.1
```

Suppose:

- R1 forwards traffic using `(S,G)` state on the source tree.
- R2 forwards traffic using `(*,G)` state on the shared tree.
- Both have SW1 in their OILs.

Both routers begin forwarding multicast packets onto the shared segment, triggering an Assert election.

Their advertised metrics are:

| Attribute | R1 (SPT) | R2 (RPT) |
|---|---|---|
| RPT-bit | 0 | 1 |
| Metric Preference | 110 | 90 |
| Route Metric | 50 | 10 |
| LAN IP | 10.0.12.1 | 10.0.12.2 |

**R1 wins because its RPT-bit is 0.**

Even though R2 has a better Metric Preference and Route Metric, these values are not considered because the RPT-bit already determines the winner.

```text
R1: SPT forwarding, RPT-bit 0  <-- WINNER
R2: RPT forwarding, RPT-bit 1
```

This ensures that source-tree forwarding is preferred over shared-tree forwarding when the two compete to forward the same stream onto a LAN.

> **Important:** The RPT-bit in a PIM Assert message should not be confused with the RP-bit (`R` flag) displayed for certain `(S,G)` multicast routing entries by `show ip mroute`. They relate to different aspects of PIM operation.

## PIM Assert and RPF Neighbor Selection

PIM Assert can also influence the upstream neighbor selected by a downstream router.

Consider the previous topology with R1 and R2 connected to R3 through SW1.

Suppose R3's routing information initially determines:

```text
RPF Interface: Ethernet0/0
RPF Neighbor:  R2
```

However, R1 wins the Assert election on the shared segment.

R3 can use the Assert winner as its upstream PIM neighbor for the affected multicast tree.

Conceptually:

```text
Before Assert:

RPF Interface: Ethernet0/0
RPF Neighbor:  R2

After R1 wins Assert:

RPF Interface: Ethernet0/0
Upstream PIM Neighbor: R1
```

The physical RPF interface remains the same, but the selected upstream PIM neighbor changes.

This is an important distinction:

**PIM Assert can override the upstream neighbor used for a multicast tree without changing the underlying unicast routing table or RPF route.**

RFC 7761 represents this Assert-aware upstream neighbor selection using `RPF'(S,G)` and `RPF'(*,G)`.

Downstream PIM Join/Prune messages can consequently be directed toward the Assert winner.

## PIM Assert vs. PIM Designated Router

The **PIM DR** and **PIM Assert winner** perform different roles.

| Feature | PIM DR | PIM Assert Winner |
|---|---|---|
| Purpose | Coordinates source/receiver PIM responsibilities | Prevents duplicate forwarding |
| Election | DR priority, then IP address | RPT-bit, metric preference, route metric, IP |
| Scope | PIM interface/LAN | Individual multicast forwarding state |
| Trigger | Neighbor/election information | Forwarding conflict or Assert processing |

For example:

```text
R1:
DR Priority 100
OSPF Metric to Source 30

R2:
DR Priority 1
OSPF Metric to Source 10
```

R1 might win the DR election because it has the higher DR priority.

However, if both routers compete in an `(S,G)` Assert election with equal Metric Preferences, R2 wins because it has the lower route metric.

```text
PIM DR:        R1
Assert Winner: R2
```

The two roles are independent.

The Assert winner takes responsibility for forwarding the affected multicast traffic onto the shared LAN. 

## Assert State and Timers

PIM maintains separate Assert state for the relevant multicast tree and interface.

The three basic Assert states are:

| State | Meaning |
|---|---|
| NoInfo | No active Assert winner information |
| Winner | Local router won the Assert election |
| Loser | Another router won the Assert election |

The default PIM-SM Assert timers defined by RFC 7761 are:

| Timer | Default | Purpose |
|---|---|---|
| Assert Time | 180 seconds | How long a losing router retains Assert winner information without receiving a refresh |
| Assert Override Interval | 3 seconds | How long before the losers' Assert state expires that the winner refreshes its Assert message |

### Assert Override Interval

The **Assert Override Interval** ensures that the Assert winner refreshes its election result before the losing routers forget it.

When an Assert election completes:

- The **Assert winner** starts its Assert Timer at `Assert Time - Assert Override Interval` (177 seconds by default).
- The **Assert losers** start their Assert Timers at the full Assert Time (180 seconds by default).

When the winner's timer expires, it sends another Assert message, refreshing the losers' timers.

For example:

```text
Time 0:
  R1 wins Assert election.
  R1 timer = 177 seconds
  R2 timer = 180 seconds

Time 177:
  R1's timer expires.
  R1 sends another Assert.
  R2 receives it and resets its timer to 180 seconds.
  R1 resets its timer to 177 seconds.

Time 354:
  R1 sends another Assert.
  R2 refreshes its timer again.
```

This prevents the losers from unnecessarily resuming multicast forwarding and triggering another election.

If the winner stops sending Asserts, the losers eventually time out their Assert state and return to **NoInfo**, allowing normal PIM forwarding procedures to resume.

> **Note:** The Assert Override Interval is different from the Join/Prune Override Interval. The former refreshes Assert winner information, whereas the latter allows routers to override PIM Prune messages.

Assert state can also be cleared or reevaluated when the winner becomes unavailable, its Generation ID changes, routing metrics change, or multicast forwarding state changes.

### Assert Cancel

A router can transmit an **Assert Cancel** when it is no longer eligible to remain the Assert winner.

This informs other routers that the previously advertised winner information should no longer be relied upon.

An Assert Cancel uses an infinite Assert metric, allowing other routers to discard the previous winner information and resume normal PIM processing.
## IOS XE Configuration

PIM Assert operates automatically when the necessary PIM forwarding conditions exist.

**There is no separate command required to enable ordinary PIM Assert operation.**

For example:

```text
R1(config)# ip multicast-routing
R1(config)# interface Ethernet0/0
R1(config-if)# ip pim sparse-mode
```

The router can participate in Assert elections on the interface when a forwarding conflict occurs.

PIM Assert is not configured by assigning an Assert priority. Instead, the election uses RPF routing information, unlike the separate `ip pim dr-priority` mechanism.

Changing routing metrics or RPF selection can therefore affect which router wins an Assert election.

## Verification

### `show ip mroute`

The primary IOS XE command for examining Assert-related forwarding state is:

```text
R1# show ip mroute 239.1.1.1
```

Suppose R1 has won an Assert election for `(10.10.10.10, 239.1.1.1)` on Ethernet0/1.

Example output:

```text
R1# show ip mroute 239.1.1.1

(10.10.10.10, 239.1.1.1), 00:05:00/00:02:59, flags: T
  Incoming interface: Ethernet0/0, RPF nbr 10.0.14.4
  Outgoing interface list:
    Ethernet0/1, Forward/Sparse, 00:05:00/00:02:59, A
```

Notice the `A` flag:

```text
Ethernet0/1, Forward/Sparse, ..., A
```

In the **Outgoing Interface List**, `A` identifies the interface as an **Assert winner** for that multicast entry.

This must not be confused with the `A` flag in the main multicast routing entry's flag list, which can indicate candidacy for MSDP advertisement.

### Checking RPF Metrics

To examine the RPF information used in an Assert election:

```text
R1# show ip rpf 10.10.10.10 metric
```

For example, the relevant fields might be:

```text
Metric preference: 110
Metric: 20
```

These correspond to the metric preference and route metric used for an `(S,G)` Assert.

For `(*,G)` Asserts, examine the RPF route toward the RP instead.

For example:

```text
R1# show ip rpf 1.1.1.1 metric
```

### PIM Debugging

For deeper troubleshooting, IOS XE provides:

```text
R1# debug ip pim
```

PIM debugging can reveal Assert messages and other control-plane events, depending on the platform and debug options.

Disable debugging when finished:

```text
R1# undebug all
```

Use debugging cautiously on production networks because PIM events can generate significant output.

### Additional Commands

| Command | Purpose |
|---|---|
| `show ip mroute <group>` | Examine OILs and Assert winner flags |
| `show ip rpf <source> metric` | Check source-based RPF metrics |
| `show ip rpf <RP> metric` | Check RP-based RPF metrics |
| `show ip pim neighbor` | Verify PIM neighbors on the shared LAN |
| `show ip pim interface` | Verify PIM-enabled interfaces and DR |
| `show ipv6 mroute` | Examine IPv6 multicast forwarding state |
| `show ipv6 rpf <address>` | Check IPv6 RPF selection |
| `debug ip pim` | Troubleshoot PIM control-plane events |

## IPv6 PIM Assert

IPv6 PIM uses the same fundamental Assert mechanism.

When multiple IPv6 PIM routers attempt to forward the same multicast traffic onto a shared link, an Assert election selects the preferred forwarder.

The same RPT-bit, metric preference, route metric, and IP address tie-breaking principles apply, using IPv6 routing information and addresses.

IPv6 PIM neighbors normally use link-local addresses, which are also relevant to upstream neighbor selection.

## PIM Assert vs. Bidirectional PIM

PIM Assert is important in traditional PIM-SM and PIM-DM because multiple routers can attempt to forward the same multicast traffic onto a shared network.

**Bidirectional PIM uses a different mechanism.**

Instead of relying on the ordinary Assert process to resolve these forwarding conflicts, Bidirectional PIM uses a **Designated Forwarder (DF) election**.

The DF mechanism determines which router is responsible for forwarding traffic for a particular RP on a link, helping prevent forwarding loops and duplicate traffic in the bidirectional shared-tree topology.

This is covered in [Bidirectional PIM](bidirectional-pim.md).

## Key Points

- **PIM Assert prevents ongoing duplicate multicast forwarding** onto shared multiaccess networks.
- The initial duplicate traffic triggers the Assert process; the election does not prevent the first duplicate packets.
- An Assert can be triggered when a router receives multicast traffic on an interface in its **OIL**.
- PIM Assert messages are Type **5**, sent to `224.0.0.13` or `FF02::D`.
- The election order is: **RPT-bit (0 preferred) > Lowest Metric Preference > Lowest Route Metric > Highest IP Address**.
- **SPT forwarding is preferred over RPT forwarding** in PIM-SM Assert elections.
- The winner continues forwarding onto the LAN; losing routers suppress forwarding for the affected state.
- PIM Assert can influence the upstream PIM neighbor independently of the underlying unicast routing table.
- The **PIM DR and Assert winner are different roles** and need not be the same router.
- The default PIM-SM Assert Time is **180 seconds**, with an Assert Override Interval of **3 seconds**.
- Assert operation is automatic when PIM is enabled.
- Use `show ip mroute` to identify Assert winner flags on outgoing interfaces.