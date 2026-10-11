# 1.6.c (i) Sparse Mode

**PIM Sparse Mode (PIM-SM)** is a multicast routing protocol that uses a **pull model** to deliver multicast traffic only to parts of the network where receivers have explicitly requested it.

Unlike PIM Dense Mode, which initially floods multicast traffic and then prunes unnecessary branches, PIM-SM builds multicast distribution trees using **PIM Join messages**.

PIM-SM is particularly suitable for networks where multicast receivers are sparsely distributed.

In traditional **Any-Source Multicast (ASM)**, PIM-SM uses a **Rendezvous Point (RP)** to connect multicast sources and receivers.

The basic process is:

1. Receivers join a multicast group.
2. Routers build a shared tree toward the RP.
3. Sources register with the RP.
4. The RP forwards multicast traffic along the shared tree.
5. Routers can switch to a source-specific Shortest-Path Tree (SPT).

## PIM-SM Distribution Trees

PIM-SM uses two types of multicast distribution trees.

| Tree | State | Root | Purpose |
|---|---|---|---|
| **Shared Tree (RPT)** | `(*,G)` | RP | Deliver traffic from multiple sources through a common distribution tree |
| **Source Tree (SPT)** | `(S,G)` | Source | Deliver traffic from a particular source along a source-specific path |

### Shared Tree (RPT)

The **Rendezvous Point Tree (RPT)** is a shared distribution tree rooted at the RP.

Routers send `(*,G)` Join messages toward the RP to establish the tree.

Because `(*,G)` state does not identify a particular source, the shared tree can deliver traffic from multiple sources sending to the same group.

### Source Tree (SPT)

The **Shortest-Path Tree (SPT)** is rooted at a particular multicast source.

Routers send `(S,G)` Join messages toward the source to establish the tree.

Source-tree forwarding can provide a more direct path between the source and receivers, avoiding unnecessary traffic through the RP.

## Example: PIM-SM Operation

Consider the following topology:

```text id="r9l2kp"
                 R3 (RP)
                 3.3.3.3
                /      \
               /        \
              R1         R4
            (FHR)       (LHR)
              |          |
           Source      Receiver
        10.10.10.10   239.1.1.1
```

Assume that R1 and R4 also have a path between them that does not pass through R3.

- **R1:** First-Hop Router (FHR), directly connected to the multicast source.
- **R3:** Rendezvous Point (RP) for `239.1.1.1`.
- **R4:** Last-Hop Router (LHR), directly connected to the receiver.

### Step 1: Receiver Joins the Group

The receiver sends an **IGMP Membership Report** for `239.1.1.1`.

R4, acting as the receiver-facing PIM Designated Router (DR), sends a `(*,G)` Join toward the RP.

```text id="sb99nd"
R4 → R3: (*, 239.1.1.1) Join
```

The Join travels hop-by-hop toward the RP, establishing the shared tree.

At this stage, the receiver has requested traffic, but the source may not yet be known.

### Step 2: Source Registers with the RP

When the source begins sending multicast traffic, R1 receives the packets.

As the source-facing DR, R1 encapsulates the initial multicast packets in **PIM Register messages** and unicasts them to the RP.

```text id="q7x2lw"
R1 → R3: PIM Register
         Source: 10.10.10.10
         Group:  239.1.1.1
```

This informs the RP about the active multicast source.

If receivers have joined the shared tree, the RP can decapsulate the registered packets and forward the multicast traffic downstream.

### Step 3: RP Joins the Source Tree

The RP then sends an `(S,G)` Join toward the source to establish a native multicast forwarding path.

```text id="k5f8xz"
R3 → R1: (10.10.10.10, 239.1.1.1) Join
```

Once native multicast traffic arrives along this path, the RP sends a **Register-Stop** message to R1.

R1 stops encapsulating data packets in PIM Register messages.

Traffic can now be delivered natively from the source toward the RP and along the shared tree to the receiver.

### Step 4: SPT Switchover

The receiver-facing router can subsequently switch from the RPT to an SPT.

R4 sends an `(S,G)` Join toward the source:

```text id="la40sr"
R4 → Source: (10.10.10.10, 239.1.1.1) Join
```

This builds a source-specific distribution tree.

Once R4 begins receiving traffic along the SPT, it can prune that source's traffic from the shared tree.

The resulting path is:

```text id="bkrj19"
Source → R1 → [SPT path] → R4 → Receiver
```

The RP is no longer required in the forwarding path for that source-to-receiver flow.

**On IOS XE, the default SPT threshold is 0 kbps**, meaning that the LHR normally initiates SPT switchover upon receiving the first multicast data packet.

> The `ip pim spt-threshold infinity` command can be used to prevent SPT switchover.
> In modern IOS XE, the only option are `0` and `infinity`.

> **Note:** This example illustrates the typical PIM-SM ASM process when receiver interest already exists. Source registration can also occur before any receivers join the group.

## Important PIM-SM State

PIM-SM uses several types of multicast state.

| State | Purpose |
|---|---|
| `(*,G)` | Shared-tree state toward the RP |
| `(S,G)` | Source-specific multicast state |
| `(S,G,rpt)` | Source-specific state associated with the shared tree, including pruning a source from the RPT |

The distinction between `(S,G)` and `(S,G,rpt)` is particularly important during SPT switchover.

These state types are examined in the following chapters.

## Basic IOS XE Configuration

Enable multicast routing globally:

```text id="9dsu0i"
R1(config)# ip multicast-routing
```

Enable PIM Sparse Mode on the relevant interfaces:

```text id="73osbd"
R1(config)# interface Ethernet0/0
R1(config-if)# ip pim sparse-mode
```

Configure an RP for ASM groups:

```text id="2w1hc3"
R1(config)# ip pim rp-address 3.3.3.3
```

The RP address must be reachable, and routers must have consistent group-to-RP mappings.

Alternatively, RP information can be distributed dynamically using BSR or Auto-RP.

Useful verification commands include:

```text id="gk7j4d"
R1# show ip pim neighbor
R1# show ip pim interface
R1# show ip pim rp mapping
R1# show ip mroute
R1# show ip rpf 3.3.3.3
```

> **Note:** Source-Specific Multicast (SSM) also uses PIM's sparse, explicit-join behavior, but does not require an RP, source registration, or a shared tree. SSM is covered separately in [Source-Specific Multicast](../ssm/overview.md).

## Section Overview

The following chapters examine the individual mechanisms of PIM-SM.

### Multicast Distribution Trees

- [PIM-SM Shared Tree](shared-tree.md) and [(*,G) State](star-g-state.md): Shared-tree construction and forwarding state.
- [PIM-SM Source Tree](source-tree.md) and [(S,G) State](s-g-state.md): Source-specific distribution trees and forwarding state.

### PIM Control Messages and Processes

- [PIM Join and Prune Messages](join-prune.md): Building and removing multicast distribution-tree branches.
- [Receiver Join Process](receiver-join.md): How receiver membership triggers upstream PIM signaling.
- [Source Registration](source-registration.md): How the RP discovers multicast sources.
- [PIM Register Messages](register.md) and [PIM Register-Stop Messages](register-stop.md): Source registration and suppression of encapsulated traffic.

### SPT Switchover

- [Shortest-Path Tree Switchover](spt-switchover.md): Transitioning from shared-tree forwarding to source-tree forwarding.
- [SPT Threshold](spt-threshold.md): Controlling when the LHR switches to the SPT.

### Multicast Forwarding Conflicts

- [PIM Assert](assert.md): Selecting a single multicast forwarder on a shared multiaccess network.

## Key Points

- **PIM-SM uses a pull model**, building multicast forwarding paths only when requested.
- Traditional PIM-SM ASM requires a **Rendezvous Point (RP)**.
- The **Shared Tree (RPT)** is rooted at the RP and uses `(*,G)` state.
- The **Source Tree (SPT)** is rooted at the multicast source and uses `(S,G)` state.
- Receiver-facing routers send PIM Joins toward the RP to establish the shared tree.
- Source-facing DRs use **PIM Register messages** to inform the RP about active sources.
- The RP can join the source tree and use **Register-Stop** to suppress continued packet encapsulation.
- Receiver-facing routers can switch to an SPT to forward traffic directly from the source.
- PIM-SM relies on **RPF lookups** to determine upstream paths toward the RP and multicast sources.