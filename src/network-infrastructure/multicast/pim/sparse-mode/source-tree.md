# PIM-SM Source Tree and (S,G) State

In PIM Sparse Mode (PIM-SM), a **Source Tree**, also called a **Shortest-Path Tree (SPT)**, is a multicast distribution tree rooted at a particular multicast source.

Unlike the shared tree (RPT), which is rooted at the Rendezvous Point (RP), an SPT follows the multicast routing path toward the source.

Source-tree forwarding is represented by **`(S,G)` state**, where:

- `S` identifies a specific multicast source.
- `G` identifies a specific multicast group.

For example:

```text
(10.10.10.10, 239.1.1.1)
```

This state identifies multicast traffic sent from `10.10.10.10` to `239.1.1.1`.

**An SPT allows multicast traffic to reach receivers without necessarily passing through the RP**, potentially providing a more direct forwarding path.

## Source-Tree Operation

Source trees are established using **PIM `(S,G)` Join messages**, which travel hop-by-hop toward the multicast source.

Consider the following topology:

```text
             Source (S)
            10.10.10.10
                 |
               R1 (FHR)
                 |
                 R2 -------- R3 (RP)
                  \          /
                   \        /
                    R4 (LHR)
                      |
                   Receiver
                  239.1.1.1
```

Assume:

- R1 is the First-Hop Router (FHR), directly connected to the source.
- R3 is the RP for group `239.1.1.1`.
- R4 is the Last-Hop Router (LHR), directly connected to the receiver.
- R4's RPF path toward the source goes through R2.
- R4's RPF path toward the RP goes directly through R3.

### Shared-Tree Forwarding

Initially, R4 joins the shared tree toward R3.

Multicast traffic from the source reaches the RP and is forwarded along the shared tree:

```text
Source → R1 → R2 → R3 (RP) → R4 → Receiver
```

R4's `(*,G)` state uses the RPF interface toward R3.

### Source-Tree Forwarding

When R4 switches to the SPT, it sends an `(S,G)` Join toward the source.

```text
R4 → R2: Join (10.10.10.10, 239.1.1.1)
```

R2 establishes the necessary `(S,G)` state and propagates the Join toward R1.

```text
R2 → R1: Join (10.10.10.10, 239.1.1.1)
```

The source tree is established:

```text
Source → R1 → R2 → R4 → Receiver
```

Traffic can now reach R4 without traversing R3.

Once source-tree forwarding is active, R4 can prune this source's traffic from the shared tree to prevent duplicate forwarding.

> **Note:** The term *Shortest-Path Tree* does not necessarily mean the mathematically shortest physical path. The tree follows the routes selected by multicast RPF lookups, which can be influenced by unicast routing, static mroutes, and other RPF information sources.

## (S,G) Forwarding State

An `(S,G)` entry identifies forwarding state for one multicast source and group.

For example, R4 might maintain:

```text
(10.10.10.10, 239.1.1.1)

Incoming Interface: Ethernet0/0 (toward R2)
Outgoing Interface:
  Ethernet0/2 (toward receiver)
```

The main components are:

| Component | Purpose |
|---|---|
| Source (S) | Identifies the multicast source |
| Group (G) | Identifies the multicast group |
| RPF Interface | Expected incoming interface for SPT forwarding |
| RPF Neighbor | Upstream PIM neighbor toward the source |
| OIL | Downstream interfaces requiring the source's traffic |
| SPT-bit | Indicates whether source-tree forwarding is active |
| Join/Prune State | Tracks downstream source-specific interest and upstream participation |

### RPF Toward the Source

For `(S,G)` source-tree forwarding, the RPF lookup is performed toward **S**, not toward the RP.

For example:

```text
Source: 10.10.10.10
Group:  239.1.1.1
RP:     3.3.3.3
```

R4 performs an RPF lookup toward `10.10.10.10`.

Suppose the lookup selects:

```text
RPF Interface: Ethernet0/0
RPF Neighbor:  R2
```

R4 sends its upstream `(S,G)` Join toward R2.

Once the SPT is active, multicast packets from `10.10.10.10` are expected to arrive through Ethernet0/0.

If a packet arrives on another interface, it fails the source-tree RPF check and is not forwarded through the SPT.

### Outgoing Interface List

The `(S,G)` OIL identifies interfaces that should receive traffic from the specified source.

Downstream interest can come from:

- **PIM `(S,G)` Joins:** Source-specific requests from downstream routers.
- **IGMPv3 or MLDv2 membership:** Local receivers requesting a particular source.
- **Inherited `(*,G)` interest:** Group-level receiver interest that also applies to the source.

For example, suppose R2 has two downstream interfaces:

```text
                  Source
                     |
                     R1
                     |
                     R2
                    /  \
                   /    \
                  R3    R4
                  |      |
                 PC1    PC2
```

Both downstream branches require traffic from `10.10.10.10`.

R2's `(S,G)` forwarding state can therefore have:

```text
(10.10.10.10, 239.1.1.1)

Incoming Interface: Ethernet0/0
Outgoing Interface List:
  Ethernet0/1 → R3
  Ethernet0/2 → R4
```

R2 receives one copy of each multicast packet and replicates it onto the eligible outgoing interfaces.

## Inheritance from (*,G) State

An important characteristic of PIM-SM is that **`(S,G)` forwarding state can inherit downstream interest from `(*,G)` state**.

Suppose a router has:

```text
(*,239.1.1.1)

OIL:
  Ethernet0/1
  Ethernet0/2
```

Both interfaces have downstream receivers interested in the group.

When the router establishes `(S,G)` forwarding state for `10.10.10.10`, those receivers may also need traffic from that source, even if they did not explicitly send source-specific Joins.

Therefore, the effective `(S,G)` OIL can include interfaces inherited from `(*,G)` state.

However, this inheritance is not unconditional.

PIM also considers:

- Source-specific `(S,G)` Joins.
- Source-specific RPT pruning, represented by `(S,G,rpt)` state.
- IGMPv3/MLDv2 source exclusions.
- PIM Assert results.

RFC 7761 refers to the resulting forwarding list as **`inherited_olist(S,G)`**.

This allows PIM to preserve group-level receiver interest while selectively forwarding or pruning individual multicast sources.

## Multiple Sources Require Separate (S,G) State

Unlike `(*,G)` state, which can represent multiple sources, `(S,G)` state identifies one specific source.

Suppose two sources send to the same multicast group:

```text
Source 1: 10.10.10.10
Source 2: 10.20.20.20
Group:    239.1.1.1
```

The router can maintain:

```text
(10.10.10.10, 239.1.1.1)

(10.20.20.20, 239.1.1.1)
```

Each entry can have a different:

- RPF interface and neighbor.
- Outgoing Interface List.
- SPT-bit state.
- Upstream Join/Prune state.

For example:

| State | RPF Interface | OIL |
|---|---|---|
| `(10.10.10.10,239.1.1.1)` | Ethernet0/0 | Ethernet0/2 |
| `(10.20.20.20,239.1.1.1)` | Ethernet0/1 | Ethernet0/2 |

Both sources deliver traffic to the same receiver, but their source trees follow different upstream paths.

This provides source-specific forwarding flexibility at the cost of maintaining additional multicast state.

## (S,G) State Does Not Always Mean Active SPT Forwarding

An important distinction is that **the existence of `(S,G)` state does not necessarily mean that the router is already forwarding along the SPT**.

For example, during SPT switchover, the LHR can create `(S,G)` state and send source-specific Joins while multicast traffic continues arriving over the shared tree.

PIM uses the **SPT-bit** to distinguish these situations.

| SPT-bit | Meaning |
|---|---|
| 0 | Source-tree forwarding is not yet active; traffic can continue using `(*,G)` forwarding state |
| 1 | Source-tree forwarding is active; `(S,G)` forwarding state is used together with applicable shared-tree state |

On IOS XE, the SPT-bit is displayed using the **`T` flag** in `show ip mroute`.

For example:

```text
(10.10.10.10, 239.1.1.1), flags: T
```

This indicates that the SPT-bit is set.

The distinction is important because `(S,G)` state can also exist:

- On the **FHR**, which maintains source-specific registration state.
- On the **RP**, which can join the source tree to receive native multicast traffic.
- On **intermediate routers**, which maintain source-specific forwarding state.
- On the **LHR**, including during SPT switchover.

**Do not assume that every `(S,G)` entry represents a completed receiver-side SPT switchover.**

The precise conditions for setting the SPT-bit are covered in [Shortest-Path Tree Switchover](spt-switchover.md).

## (S,G) vs. (S,G,rpt) State

PIM-SM also uses **`(S,G,rpt)` state**, which is different from normal `(S,G)` source-tree state.

| State | Purpose |
|---|---|
| `(S,G)` | Source-specific forwarding and upstream Joins toward S |
| `(S,G,rpt)` | Source-specific Join/Prune state associated with the shared tree toward the RP |

A common use of `(S,G,rpt)` state is to prune one particular source's traffic from the shared tree after SPT switchover.

For example:

```text
Join (10.10.10.10, 239.1.1.1)
→ Toward the source

Prune (10.10.10.10, 239.1.1.1, RPT-bit set)
→ Toward the RP
```

These messages have different purposes:

- The `(S,G)` Join requests source-specific traffic along the SPT.
- The `(S,G,rpt)` Prune suppresses that source's traffic along the RPT.

Other sources can continue using the shared tree for the same group.

## Verification

### `show ip mroute`

Use:

```text
R4# show ip mroute 239.1.1.1
```

Example output after SPT switchover:

```text
(*, 239.1.1.1), 00:05:00/00:02:59, RP 3.3.3.3, flags: S
  Incoming interface: Ethernet0/1, RPF nbr 10.0.34.3
  Outgoing interface list:
    Ethernet0/2, Forward/Sparse, 00:05:00/00:02:59

(10.10.10.10, 239.1.1.1), 00:03:00/00:02:59, flags: T
  Incoming interface: Ethernet0/0, RPF nbr 10.0.24.2
  Outgoing interface list:
    Ethernet0/2, Forward/Sparse, 00:03:00/00:02:59
```

Notice the different incoming interfaces:

```text
(*,G):
  Incoming interface: Ethernet0/1 (toward RP)

(S,G):
  Incoming interface: Ethernet0/0 (toward Source)
```

Also notice the `T` flag on the `(S,G)` entry, indicating active source-tree forwarding.

Both entries can coexist without causing duplicate forwarding of the same source's packets.

### Additional Verification

| Command | Purpose |
|---|---|
| `show ip mroute <group>` | Display `(*,G)` and `(S,G)` state |
| `show ip mroute <group> count` | Display multicast forwarding and drop counters |
| `show ip rpf <source>` | Verify the RPF path toward the source |
| `show ip pim neighbor` | Verify PIM neighbor relationships |
| `show ip pim rp mapping` | Verify the RP used for ASM groups |
| `show ipv6 mroute` | Display IPv6 multicast forwarding state |
| `show ipv6 rpf <source>` | Verify the IPv6 source-based RPF path |

## Key Points

- The **Source Tree (SPT)** is rooted at a particular multicast source.
- `(S,G)` state identifies traffic from a specific source S to group G.
- PIM establishes source trees using **`(S,G)` Joins toward the source**.
- The RPF lookup for source-tree forwarding is performed toward **S**, not the RP.
- The effective `(S,G)` OIL can inherit downstream interest from `(*,G)` state.
- Multiple sources sending to the same group can have different `(S,G)` forwarding entries and RPF paths.
- **An `(S,G)` entry does not necessarily mean that SPT forwarding is active.**
- The `T` flag in `show ip mroute` indicates that the SPT-bit is set.
- `(S,G,rpt)` state is associated with source-specific pruning and forwarding control on the shared tree.
- After SPT switchover, `(*,G)` and `(S,G)` state can coexist without duplicate forwarding.