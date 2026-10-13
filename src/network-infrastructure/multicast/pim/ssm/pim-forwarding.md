
# SSM Receiver Signaling and PIM Forwarding

**Source-Specific Multicast (SSM)** uses source-specific receiver signaling and **PIM `(S,G)` Join/Prune messages** to establish multicast distribution trees directly toward known sources.

Unlike traditional PIM Sparse Mode ASM, SSM does not require an RP, shared tree, source registration, or SPT switchover.

SSM uses three main components:

| Component | IPv4 | IPv6 |
|---|---|---|
| Receiver signaling | IGMPv3 | MLDv2 |
| Multicast routing | PIM `(S,G)` Join/Prune | PIM `(S,G)` Join/Prune |
| Reverse Path Forwarding | IPv4 RPF lookup toward S | IPv6 RPF lookup toward S |

The key distinction is that **IGMPv3/MLDv2 communicate receiver interest, while PIM establishes the distribution tree needed to deliver the requested traffic**.

## SSM Receiver Signaling

Native SSM requires a receiver to identify both a multicast source and group.

### IPv4: IGMPv3

An IPv4 receiver uses **IGMPv3 INCLUDE mode** to request one or more sources.

For example:

```text
Group:   232.1.1.1
Mode:    INCLUDE
Sources: {10.1.1.10}
```

This requests the SSM channel:

```text
(10.1.1.10, 232.1.1.1)
```

When the receiver first subscribes, it sends an IGMPv3 Membership Report containing a source-specific state-change record, typically `CHANGE_TO_INCLUDE_MODE` or `ALLOW_NEW_SOURCES`, depending on its previous membership state.

Subsequent Queries can trigger `MODE_IS_INCLUDE` Reports to refresh the receiver's state.

The last-hop router processes these Reports and determines which `(S,G)` channels require forwarding onto the receiver-facing interface.

### IPv6: MLDv2

IPv6 uses **MLDv2 INCLUDE mode** in the same way.

For example:

```text
Group:   ff35::fd00:101
Mode:    INCLUDE
Sources: {2001:db8:10::10}
```

This requests:

```text
(2001:db8:10::10, ff35::fd00:101)
```

MLDv2 uses the same six Multicast Address Record types and INCLUDE/EXCLUDE model as IGMPv3.

The IPv6 last-hop router uses the source-specific listener information to establish the required PIM forwarding state.

### INCLUDE Mode and SSM

A receiver may request multiple sources for one group:

```text
INCLUDE {S1, S2}
```

The router can establish two independent channels:

```text
(S1,G)
(S2,G)
```

Each channel has its own source-specific multicast forwarding state and RPF path.

For a group classified as SSM:

- `INCLUDE {S1,S2}` requests traffic from S1 and S2.
- `INCLUDE {}` requests no channels.
- EXCLUDE-mode membership does not establish SSM forwarding state.

**SSM requires explicit source information.** A traditional group-only IGMPv1/IGMPv2 or MLDv1 Report cannot establish native SSM forwarding state.

> **Important:** An IGMPv3 or MLDv2 INCLUDE-mode Report does not automatically make PIM operate in SSM mode. PIM must classify the multicast group as SSM. An INCLUDE-mode Report for an ASM group does not necessarily bypass RP-based ASM forwarding.

## PIM-SSM Forwarding Process

SSM uses the source-tree mechanisms of PIM Sparse Mode.

Consider the following topology:

```text
                   Source S
                      |
                      |
                    R1 (FHR)
                      |
                      |
                      R2
                      |
                      |
                    R3 (LHR)
                      |
                      |
                   Receiver
```

Assume:

| Parameter | IPv4 | IPv6 |
|---|---|---|
| Source S | `10.1.1.10` | `2001:db8:10::10` |
| Group G | `232.1.1.1` | `ff35::fd00:101` |
| Receiver protocol | IGMPv3 | MLDv2 |

All routers have multicast routing enabled and an RPF path toward the source.

### Step 1: Receiver Requests (S,G)

The receiver sends an IGMPv3 or MLDv2 Report requesting the source-specific channel.

For IPv4:

```text
IGMPv3 Report

Group:   232.1.1.1
Mode:    INCLUDE
Sources: {10.1.1.10}
```

For IPv6:

```text
MLDv2 Report

Group:   ff35::fd00:101
Mode:    INCLUDE
Sources: {2001:db8:10::10}
```

R3 receives the Report and learns that a downstream receiver wants traffic from source S for group G.

### Step 2: Last-Hop Router Performs RPF Lookup

R3 performs an **RPF lookup toward S**, not toward an RP.

For example:

```text
RPF Destination: 10.1.1.10

RPF Interface:  Ethernet0/0
RPF Neighbor:   R2
```

The RPF information identifies the upstream path toward the source.

IPv6 follows the same process using the IPv6 multicast RPF information.

### Step 3: Last-Hop Router Sends (S,G) Join

R3 sends a PIM `(S,G)` Join toward R2.

```text
R3 → R2

Join(S,G)
```

For IPv4:

```text
Join(10.1.1.10, 232.1.1.1)
```

For IPv6:

```text
Join(2001:db8:10::10, ff35::fd00:101)
```

The Join requests traffic from the specific source.

Unlike traditional ASM, R3 does **not** first send a `(*,G)` Join toward an RP.

### Step 4: Intermediate Routers Build (S,G) State

R2 receives the Join and adds the interface toward R3 to the channel's **Outgoing Interface List (OIL)**.

For example:

```text
(S,G)

Incoming Interface: Ethernet0/0
Outgoing Interface List:
  Ethernet0/1 → R3
```

R2 performs its own RPF lookup toward S and sends an `(S,G)` Join toward R1.

The same process continues hop-by-hop toward the source.

```text
              Source S
                 |
                 R1
                 |
                 R2
                 |
                 R3
                 |
              Receiver

PIM Join direction:

R3 → R2 → R1
```

Each router establishes the source-specific forwarding state required to deliver the stream downstream.

**The SSM distribution tree can be established before the source begins transmitting multicast traffic.**

Unlike ASM, there is no need to wait for PIM Register messages or RP-based source discovery.

### Step 5: Source Sends Multicast Traffic

When the source begins transmitting, packets follow the established distribution tree.

```text
              Source S
                 |
                 v
                 R1
                 |
                 v
                 R2
                 |
                 v
                 R3
                 |
                 v
              Receiver
```

At each router:

1. The router receives a packet with source S and destination G.
2. It checks whether the packet arrived on the expected RPF interface.
3. If the RPF check succeeds, it forwards the packet through the eligible interfaces in the `(S,G)` OIL.
4. If the RPF check fails, the packet is discarded.

SSM therefore uses the same fundamental multicast source-tree forwarding mechanisms as PIM-SM.

However, **SSM does not need to transition from an RPT to an SPT**, because its distribution tree is source-specific from the beginning.

## PIM Join/Prune Messages in SSM

PIM-SSM uses the standard PIMv2 **Join/Prune message (Type 3)**.

| Field | IPv4 | IPv6 |
|---|---|---|
| Destination | `224.0.0.13` | `ff02::d` |
| IP Protocol / Next Header | 103 | 103 |
| TTL / Hop Limit | 1 | 1 |
| PIM Message Type | 3 | 3 |

The Join/Prune message identifies the upstream PIM neighbor and contains the relevant multicast group and source entries.

For an SSM `(S,G)` Join, the encoded source address uses:

```text
Wildcard (W) = 0
RPT (R)      = 0
```

This identifies source-specific SPT state rather than `(*,G)` or `(S,G,rpt)` state.

SSM does not use:

- `Join(*,G)` to establish a shared tree.
- `Prune(S,G,rpt)` to remove a source from an RPT.
- PIM Register/Register-Stop messages.

### Periodic Join Refresh

Like PIM-SM, PIM-SSM uses soft state.

Routers periodically refresh upstream `(S,G)` Joins while downstream interest remains.

The normal PIM Join/Prune defaults are:

```text
Join/Prune Period: 60 seconds
Join/Prune Holdtime: 210 seconds
```

A router does not need to wait for the source to transmit packets before refreshing its Join state.

## Multiple Receivers and Sources

### Multiple Receivers for One Channel

Suppose two receivers request the same channel.

```text
                 Source S
                    |
                    R1
                    |
                    R2
                   /  \
                  /    \
                 R3    R4
                 |      |
               Rec1   Rec2
```

Both receivers request:

```text
(S,G)
```

R2 receives `(S,G)` Joins from R3 and R4.

Its forwarding state becomes:

```text
(S,G)

Incoming Interface: Ethernet0/0
Outgoing Interface List:
  Ethernet0/1 → R3
  Ethernet0/2 → R4
```

R2 maintains **one upstream `(S,G)` Join**, while replicating multicast packets onto both downstream interfaces.

### Multiple Sources for One Group

Suppose receivers request:

```text
(S1,G)
(S2,G)
```

SSM maintains separate source-specific forwarding state.

For example:

```text
(S1,G)
  RPF Neighbor: R1

(S2,G)
  RPF Neighbor: R5
```

Each source may have a different RPF interface, upstream neighbor, and distribution tree.

The same principle applies to IPv4 and IPv6.

## Leaving an SSM Channel

When receivers stop requesting an `(S,G)` channel, the last-hop router must determine whether any downstream interest remains.

For IPv4, an IGMPv3 receiver may advertise a source removal using `BLOCK_OLD_SOURCES` or a transition to an empty INCLUDE list.

For IPv6, MLDv2 uses equivalent Multicast Address Records.

For example:

```text
Old State:
INCLUDE {S1, S2}

New State:
INCLUDE {S2}
```

The receiver no longer wants S1.

The last-hop router can send group-and-source-specific Queries to determine whether other receivers still require S1.

If no downstream interest remains, the router removes the relevant forwarding interest and can send an upstream:

```text
Prune(S1,G)
```

The Prune propagates toward the source only as far as necessary.

If an intermediate router still has another downstream receiver for `(S1,G)`, that router retains the required upstream Join.

**Removing one SSM channel does not affect other channels using the same multicast group.**

## IOS XE Configuration

### IPv4 SSM

Enable IPv4 multicast routing and the standard SSM range:

```text
R1(config)# ip multicast-routing
R1(config)# ip pim ssm default
```

Enable PIM Sparse Mode on the relevant interfaces:

```text
R1(config)# interface Ethernet0/0
R1(config-if)# ip pim sparse-mode
R1(config-if)# exit
R1(config)# interface Ethernet0/1
R1(config-if)# ip pim sparse-mode
```

On the last-hop router, enable IGMPv3 on the receiver-facing interface:

```text
R3(config)# interface Ethernet0/1
R3(config-if)# ip pim sparse-mode
R3(config-if)# ip igmp version 3
```

IGMPv3 is necessary for native source-specific receiver signaling.

The standard SSM range is configured globally, not separately on each interface.

#### Configure an IOS XE Router as an IGMPv3 Receiver

For lab testing, an IOS XE router can act as an IGMPv3 receiver:

```text
Rec1(config)# interface Ethernet0/0
Rec1(config-if)# ip igmp version 3
Rec1(config-if)# ip igmp join-group 232.1.1.1 source 10.1.1.10
```

The `source` keyword requests the specific SSM channel:

```text
(10.1.1.10, 232.1.1.1)
```

Without IGMPv3 enabled on the interface, source-specific local state may be created without transmitting the required IGMPv3 Membership Reports.

To request multiple sources, configure additional source-specific `join-group` commands.

To remove a channel:

```text
Rec1(config-if)# no ip igmp join-group 232.1.1.1 source 10.1.1.10
```

#### Configure Static Source-Specific Forwarding

To make a router forward a particular channel onto an interface without making the router itself a local receiver:

```text
R3(config)# interface Ethernet0/1
R3(config-if)# ip igmp static-group 232.1.1.1 source 10.1.1.10
```

This creates static downstream interest in the specified channel.

Unlike `ip igmp join-group`, `ip igmp static-group` does not require the router to receive and process the multicast stream as a local application.

### IPv6 SSM

Enable IPv6 multicast routing:

```text
R1(config)# ipv6 unicast-routing
R1(config)# ipv6 multicast-routing
```

On IOS XE, IPv6 multicast routing enables PIM and MLD processing on eligible IPv6-enabled interfaces.

Unlike IPv4, IPv6 SSM does not normally require an equivalent of `ip pim ssm default`.

MLDv2 is required for native IPv6 source-specific listener signaling.

#### Configure an IOS XE Router as an MLDv2 Receiver

```text
Rec1(config)# interface Ethernet0/0
Rec1(config-if)# ipv6 mld join-group ff35::fd00:101 include 2001:db8:10::10
```

This requests:

```text
(2001:db8:10::10, ff35::fd00:101)
```

For source-specific membership, INCLUDE mode is used. It is also the default when a source address is specified without an explicit mode.

To remove the membership:

```text
Rec1(config-if)# no ipv6 mld join-group ff35::fd00:101 include 2001:db8:10::10
```

#### Configure Static Source-Specific Forwarding

```text
R3(config)# interface Ethernet0/1
R3(config-if)# ipv6 mld static-group ff35::fd00:101 include 2001:db8:10::10
```

This creates static downstream forwarding interest without making the router itself a local receiver.

## Verification

### IPv4 Verification

```text
show ip igmp interface Ethernet0/1
show ip igmp groups detail
show ip pim neighbor
show ip mroute
show ip rpf 10.1.1.10
show ip mfib
```

| Command | Purpose |
|---|---|
| `show ip igmp groups detail` | Verify source-specific receiver membership |
| `show ip mroute` | Verify `(S,G)` state, RPF interface, and OIL |
| `show ip rpf <source>` | Verify the RPF path toward the source |
| `show ip pim neighbor` | Verify upstream PIM adjacency |
| `show ip mfib` | Inspect multicast forwarding entries |

In `show ip mroute`, the flag `s` identifies an SSM group on IOS XE platforms that display this flag. The `I` flag can indicate that a source-specific host report was received.

The exact flags and output format vary by software and platform.

### IPv6 Verification

```text
show ipv6 mld interface Ethernet0/1
show ipv6 mld groups detail
show ipv6 pim neighbor
show ipv6 pim group-map
show ipv6 pim range-list
show ipv6 pim topology
show ipv6 mroute
show ipv6 rpf 2001:db8:10::10
show ipv6 mfib
```

Particularly useful commands are:

- `show ipv6 mld groups detail`: Check MLDv2 INCLUDE-mode receiver interest and source lists.
- `show ipv6 pim group-map`: Verify that the group is classified as SSM.
- `show ipv6 pim range-list`: Display the SSM ranges recognized by PIM.
- `show ipv6 pim topology`: Inspect PIM source-specific topology and upstream Join state.
- `show ipv6 mroute`: Inspect multicast routing entries and forwarding interfaces.
- `show ipv6 rpf <source>`: Check the IPv6 RPF path toward S.

## Troubleshooting SSM Forwarding

SSM troubleshooting follows the receiver-to-source signaling path.

| Symptom | Possible Cause |
|---|---|
| No source-specific membership | IGMPv3/MLDv2 not operating correctly, or receiver did not request S |
| Membership exists, but no `(S,G)` forwarding | Incorrect SSM range classification, PIM configuration, or RPF information |
| `(S,G)` state exists, but no upstream Join | Missing or incorrect RPF neighbor, or PIM adjacency issue |
| Upstream Join exists, but no traffic | Source not transmitting, missing upstream forwarding, or RPF failure |
| Traffic reaches router but not receiver | Missing OIL, listener state expired, snooping issue, or forwarding restriction |
| IPv4 SSM group attempts ASM forwarding | SSM not enabled or inconsistent SSM range configuration |

A useful troubleshooting sequence is:

```text
Receiver IGMPv3/MLDv2 state
             |
             v
Last-hop router membership state
             |
             v
Group classified as SSM?
             |
             v
RPF lookup toward source
             |
             v
PIM (S,G) Join toward source
             |
             v
(S,G) state on intermediate routers
             |
             v
Source traffic passes RPF check
             |
             v
Forwarding through OIL to receiver
```

A valid `(S,G)` Join does not guarantee that packets will arrive. The source must transmit, the multicast distribution path must be established, and packets must pass the RPF check.

## Key Points

- Native IPv4 SSM uses **IGMPv3**, while native IPv6 SSM uses **MLDv2**.
- Receivers specify the desired multicast channels using INCLUDE-mode source lists.
- A source-specific IGMPv3/MLDv2 Report does not necessarily imply PIM-SSM. The group must be classified as SSM.
- The last-hop router sends **PIM `(S,G)` Joins toward the source**, not toward an RP.
- Intermediate routers maintain `(S,G)` state and outgoing interface lists.
- SSM trees can be established before multicast data begins arriving.
- SSM does not require `(*,G)` shared-tree forwarding, PIM Registers, or SPT switchover.
- PIM `(S,G)` Joins are refreshed periodically while downstream interest remains.
- When the last receiver for a channel leaves, upstream `(S,G)` forwarding interest can be pruned.
- IOS XE supports source-specific receiver testing using `ip igmp join-group ... source ...` and `ipv6 mld join-group ... include ...`.
- Verification should examine receiver membership, PIM Join state, RPF information, and multicast forwarding entries.
