# Bidirectional PIM

**Bidirectional PIM (Bidir-PIM)** is a multicast routing protocol designed for **many-to-many multicast applications**, where numerous sources send traffic to the same multicast group.

Like traditional PIM Sparse Mode (PIM-SM), Bidir-PIM uses a shared multicast distribution tree rooted at a **Rendezvous Point (RP)**.

However, unlike traditional PIM-SM, **multicast traffic can flow both upstream toward the RP and downstream toward receivers along the same shared tree**.

This eliminates the need for source-specific distribution trees and allows routers to maintain only `(*,G)` multicast forwarding state.

> Bidir-PIM is **not** a CCIE-EI topic, so this is just a high-level overview.

## Bidirectional Shared Tree

Traditional PIM-SM uses a shared tree to deliver traffic from the RP toward receivers. Traffic from a new source is initially encapsulated in PIM Register messages and sent to the RP.

Bidir-PIM eliminates this registration process.

Instead, multicast sources send traffic natively along the bidirectional shared tree toward the RP.

Consider the following topology:

```text
                    RP
                 1.1.1.1
                    |
                    R3
                   /  \
                  /    \
                 R1    R4
                /  \   /  \
               S1  H1 S2  H2
```

- S1 and S2 are multicast sources.
- H1 and H2 are receivers for the same group, `239.1.1.1`.
- The RP is `1.1.1.1`.

Suppose S1 begins sending multicast traffic.

The traffic flows upstream toward the RP:

```text
S1 → R1 → R3 → RP
```

As the traffic travels toward the RP, routers also forward copies onto downstream branches that contain interested receivers.

For example, R3 forwards S1's traffic toward R4 so that H2 can receive it.

If S2 begins sending, traffic can travel in the opposite direction along the same distribution tree:

```text
S2 → R4 → R3 → R1 → H1
```

**The same shared tree supports traffic from multiple sources without building a separate source tree for each source.**

Traffic does not necessarily have to travel all the way to the RP before reaching receivers. Routers can replicate traffic onto interested downstream branches along the way.

## (*,G) State Only

One of Bidir-PIM's most important characteristics is that it maintains **`(*,G)` forwarding state rather than `(S,G)` state**.

For example, suppose 100 sources are sending to `239.1.1.1`.

Traditional PIM-SM may maintain individual `(S,G)` entries for those sources:

```text
(10.1.1.1, 239.1.1.1)
(10.1.1.2, 239.1.1.1)
(10.1.1.3, 239.1.1.1)
...
```

Bidir-PIM instead maintains shared-tree state:

```text
(*, 239.1.1.1)
```

This significantly reduces multicast forwarding state when numerous sources transmit to the same group.

Because Bidir-PIM uses only the shared tree:

- There is **no PIM Register or Register-Stop process**.
- There is **no SPT switchover**.
- There are **no source-specific `(S,G)` distribution trees**.
- An RP is still required to identify the root of the shared tree.

## Designated Forwarder (DF) Election

Bidir-PIM uses a special mechanism called the **Designated Forwarder (DF) election**.

The DF election is essential because multicast packets can travel upstream along the shared tree. Without controls, multiple routers could forward the same packets toward the RP, causing duplicates or forwarding loops.

On a shared LAN, the DF is the router responsible for forwarding multicast traffic **from that LAN toward the RP**.

### Example: DF Election

Consider two routers connected to a multicast source through a shared Ethernet LAN:

```text
                   Toward RP
                  /         \
                 R1         R2
                  \         /
                   \       /
                      SW1
                       |
                    Source
```

Both R1 and R2 have paths toward the RP.

Suppose their routing information is:

| Attribute | R1 | R2 |
|---|---|---|
| Metric Preference to RP | 110 | 110 |
| Route Metric to RP | 20 | 10 |
| LAN IP Address | 10.0.12.1 | 10.0.12.2 |

R2 wins the DF election because it has the lower route metric toward the RP.

```text
R1: Not DF
R2: DF
```

When the source sends multicast traffic onto SW1:

- R2 accepts the multicast traffic and forwards it upstream toward the RP.
- R1 does not forward that traffic upstream from SW1.

This ensures that only one router forwards the source's traffic toward the RP from the shared LAN.

### DF Election Rules

The DF election uses routing information toward the RP:

1. **Lowest Metric Preference** wins.
2. If equal, the **lowest Route Metric** wins.
3. If still equal, the **highest IP address** wins.

These metric comparisons resemble those used by PIM Assert, but the mechanisms are different.

Bidir-PIM uses dedicated DF election messages:

| Message | Purpose |
|---|---|
| Offer | Advertise candidacy and the router's metric toward the RP |
| Winner | Announce the elected DF |
| Backoff | Delay an election while a better candidate is considered |
| Pass | Transfer DF responsibility to another router |

A separate DF election is maintained for each RP address on a link, rather than for every individual multicast source and group.

The DF election can be repeated when routing metrics change, a new neighbor appears, or the current DF becomes unavailable.

## DF vs. DR vs. Assert Winner

The Bidir-PIM DF is different from both the ordinary PIM Designated Router (DR) and the PIM Assert winner.

| Role | Purpose |
|---|---|
| PIM DR | Performs source/receiver responsibilities in traditional PIM-SM |
| PIM Assert Winner | Resolves competing multicast forwarders for particular multicast state |
| Bidir-PIM DF | Controls upstream forwarding toward the RP on a link |

In Bidir-PIM, the DF also assumes the DR's responsibility for handling local receiver interest.

For example, when a host sends an IGMP Membership Report, the elected DF originates the necessary `(*,G)` Join toward the RP.

**Bidir-PIM does not use ordinary PIM Assert elections to select multicast forwarders.** Its DF election prevents the competing upstream forwarding paths that could otherwise create loops.

## Bidir-PIM vs. Traditional PIM-SM

| Feature | PIM-SM (ASM) | Bidir-PIM |
|---|---|---|
| RP required | Yes | Yes |
| Shared Tree | Yes | Yes |
| Source Tree | Yes | No |
| Multicast state | `(*,G)` and `(S,G)` | `(*,G)` only |
| Traffic travels upstream on shared tree | No | Yes |
| PIM Register messages | Yes | No |
| SPT switchover | Yes | No |
| PIM Assert | Yes | Replaced by DF election |
| Ideal application | Typically one-to-many | Many-to-many |

Bidir-PIM trades the potentially shorter forwarding paths of source-specific trees for simpler multicast forwarding state and improved scalability with many sources.

## Basic IOS XE Configuration

On supported IOS XE platforms, enable Bidir-PIM globally:

```text
R1(config)# ip multicast-routing
R1(config)# ip pim bidir-enable
```

Enable PIM Sparse Mode on the multicast interfaces:

```text
R1(config)# interface Ethernet0/0
R1(config-if)# ip pim sparse-mode
R1(config-if)# exit
R1(config)# interface Ethernet0/1
R1(config-if)# ip pim sparse-mode
```

Configure an RP for bidirectional multicast groups:

```text
R1(config)# ip access-list standard BIDIR-GROUPS
R1(config-std-nacl)# permit 239.1.1.0 0.0.0.255
R1(config-std-nacl)# exit
R1(config)# ip pim rp-address 1.1.1.1 BIDIR-GROUPS bidir
```

The `bidir` keyword identifies the RP mapping as a bidirectional group mapping.

All participating routers must support Bidir-PIM and have consistent RP/group mapping information.

The RP address must be reachable through the routing infrastructure.

> Note: Unlike traditional PIM-SM, Bidir-PIM does not require the RP address to be assigned to an actual router. The RP address can be a **Phantom RP**, an unassigned but routable IP address used to determine the direction of the bidirectional shared tree. Multiple routers can advertise routes covering this address to provide redundancy through normal routing convergence.

### Verification

Useful IOS XE commands include:

```text
R1# show ip pim rp mapping
R1# show ip pim interface Ethernet0/0 df
R1# show ip mroute
```

The `show ip pim interface ... df` command displays DF election information, including the elected DF and its routing metric toward the RP.

The multicast routing table can be used to verify that the bidirectional groups maintain `(*,G)` state without individual `(S,G)` forwarding entries.

> **Note:** Bidir-PIM support and verification commands vary by IOS XE platform and software release.

## Key Points

- **Bidir-PIM uses a bidirectional shared tree**, allowing multicast traffic to travel toward and away from the RP.
- It is designed for **many-to-many multicast applications** with numerous sources.
- Routers maintain **`(*,G)` state only**, regardless of the number of sources.
- Bidir-PIM does not use PIM Register messages, source-specific trees, or SPT switchover.
- The **Designated Forwarder (DF)** determines which router forwards traffic upstream toward the RP on a shared LAN.
- The DF election selects the router with the most preferred routing metric toward the RP.
- DF election is performed per RP address on a link, rather than per multicast source.
- Unlike traditional PIM-SM, Bidir-PIM uses DF elections instead of ordinary PIM Assert elections to prevent competing forwarding paths.
- On IOS XE, `ip pim bidir-enable` enables IPv4 Bidir-PIM, and the `bidir` keyword identifies a bidirectional RP mapping.