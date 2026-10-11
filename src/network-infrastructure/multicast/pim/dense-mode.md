# PIM Dense Mode

**PIM Dense Mode (PIM-DM)** is a multicast routing protocol that uses a **push model** to distribute multicast traffic throughout a network.

Dense Mode assumes that multicast receivers are widespread, so multicast traffic is initially flooded throughout the PIM-DM network.

Routers that do not need the traffic send **PIM Prune messages** upstream to stop unnecessary forwarding.

This approach is known as **flood and prune**.

PIM-DM is a CCIE-EI topic, but understanding its operation helps illustrate the differences between Dense Mode and Sparse Mode.

## Push vs. Pull Model

The fundamental difference between PIM Dense Mode and Sparse Mode is how multicast distribution begins.

| Feature | PIM Dense Mode | PIM Sparse Mode |
|---|---|---|
| Model | **Push** | **Pull** |
| Initial forwarding | Flood multicast traffic | Forward only when explicitly requested |
| Receiver interest | Assumed until pruned | Required to establish forwarding |
| Distribution tree | Source Tree (SPT) | Shared Tree (RPT) and/or SPT |
| Multicast state | `(S,G)` | `(*,G)` and `(S,G)` |
| Rendezvous Point (RP) | Not required | Required for traditional ASM |
| Primary control messages | Prune, Graft | Join, Prune |

In **Dense Mode**, multicast traffic is pushed throughout the network unless routers explicitly reject it.

In **Sparse Mode**, multicast traffic is pulled toward receivers through explicit PIM Join messages.

## Flood and Prune Operation

Consider the following topology:

```text id="7t2xnm"
              Source
           10.10.10.10
                 |
                 R1
                 |
                 R2
                /  \
               /    \
              R3    R4
               |     |
            Receiver No Receiver
           239.1.1.1
```

### Step 1: Initial Flooding

When the source begins sending traffic to `239.1.1.1`, R1 forwards the multicast packets toward R2.

R2 performs an **RPF check** toward the source to verify that packets arrived on the expected incoming interface.

If the RPF check passes, R2 initially forwards the traffic onto its eligible downstream PIM-DM interfaces.

```text id="bd9zex"
            Source
               |
               R1
               |
               R2
              /  \
             v    v
            R3    R4
             |     |
          Receiver No Receiver
```

Both R3 and R4 initially receive the multicast stream.

### Step 2: Pruning Unnecessary Branches

R4 has no interested receivers or downstream routers requiring the traffic.

It therefore sends a **PIM Prune message** upstream toward R2.

```text id="vj93i2"
R4 → R2: Prune (10.10.10.10, 239.1.1.1)
```

R2 removes R4's interface from the active forwarding branches for that `(S,G)` state.

Multicast traffic continues toward R3, which has an interested receiver.

```text id="9bm09w"
            Source
               |
               R1
               |
               R2
              /  \
             v    X
            R3    R4
             |
          Receiver

X = Pruned branch
```

If R3 also had no interested receivers, it could prune its branch. If R2 then had no remaining downstream interest, R2 could send a Prune toward R1.

In this way, unnecessary branches are progressively removed from the multicast distribution tree.

## PIM Graft Messages

What happens if a new receiver becomes interested in a group on a previously pruned branch?

Suppose a host connected to R4 sends an IGMP Membership Report for `239.1.1.1`.

R4 needs the multicast traffic again, but its upstream branch is currently pruned.

Rather than waiting for the Prune state to expire, R4 sends a **PIM Graft message** toward R2.

```text id="p4djrn"
R4 → R2: Graft (10.10.10.10, 239.1.1.1)
R2 → R4: Graft-Ack
```

The Graft removes the relevant Prune state and restores forwarding toward R4.

The Graft-Ack confirms that R2 received the request. If no acknowledgment arrives, R4 can retransmit the Graft.

If R2's upstream branch was also pruned, R2 can send a Graft toward R1.

**Graft messages allow previously pruned branches to rejoin the distribution tree without waiting for their Prune timers to expire.**

## Prune Timers and Periodic Reflooding

PIM-DM Prune state is temporary.

A typical Prune Holdtime is **180 seconds (3 minutes)**.

When the Prune state expires, forwarding can resume on the previously pruned branch, even if no new receiver has joined.

The downstream router may then send another Prune.

This creates a recurring cycle:

```text id="ulxx4n"
Initial Flood
     |
     v
Prune Unwanted Branches
     |
     v
Prune Timer Expires
     |
     v
Traffic Flooded Again
     |
     v
Prune Unwanted Branches
     |
    ...
```

This periodic reflooding can waste bandwidth, particularly when multicast receivers exist on only a small number of network segments.

### PIM Dense Mode State Refresh

**PIM State Refresh** reduces unnecessary reflooding.

The router directly connected to the multicast source periodically originates State Refresh messages, which propagate through the PIM-DM network and refresh the existing Prune state.

This allows routers to preserve pruned branches without periodically flooding multicast data again.

On supporting IOS XE platforms, State Refresh processing and forwarding are enabled by default, but **State Refresh origination is disabled by default**.

Enable origination on the FHR's source-facing interface:

```text id="ppkk4d"
R1(config)# interface Ethernet0/0
R1(config-if)# ip pim state-refresh origination-interval 60
```

The default origination interval, when enabled without specifying an interval, is **60 seconds**.

To disable State Refresh processing and forwarding globally:

```text id="s50h23"
R1(config)# ip pim state-refresh disable
```

> `ip pim state-refresh disable` also prevents the first-hop DR from originating State Refresh messages.

## Basic IOS XE Configuration

Enable multicast routing globally:

```text id="xq2f0f"
R1(config)# ip multicast-routing
```

Enable PIM Dense Mode on the relevant interfaces:

```text id="b364dt"
R1(config)# interface Ethernet0/0
R1(config-if)# ip pim dense-mode
R1(config-if)# exit
R1(config)# interface Ethernet0/1
R1(config-if)# ip pim dense-mode
```

No RP configuration is necessary.

To verify Dense Mode operation:

```text id="cdotjc"
R1# show ip pim interface
R1# show ip pim neighbor
R1# show ip mroute
```

In `show ip mroute`, the `D` flag identifies Dense Mode multicast state.

For example, an outgoing interface may appear as:

```text id="xhb495"
Ethernet0/1, Prune/Dense, 00:02:00/00:01:00
```

This indicates that the interface is currently pruned for the multicast entry.

> **Note:** Some IOS XE switching platforms do not support PIM Dense Mode. Availability depends on the platform and software release.

## Other PIM-DM Characteristics

A few additional points are worth noting:

- **RPF:** PIM-DM uses source-based RPF checks to prevent forwarding loops during flooding.
- **PIM Assert:** On multiaccess networks, Assert elections select a single forwarder when multiple routers attempt to forward the same multicast stream.
- **PIM Join messages:** Unlike PIM-SM, PIM-DM does not use periodic Joins to maintain its distribution tree. Joins can be used to override Prunes on multiaccess networks.
- **No RP:** PIM-DM builds only source-rooted `(S,G)` trees, so it does not require an RP, Register messages, or SPT switchover.

## Key Points

- **PIM Dense Mode uses a push model**, while Sparse Mode uses a pull model.
- PIM-DM initially **floods** multicast traffic and then **prunes** branches without interested receivers.
- PIM-DM builds **source-based `(S,G)` distribution trees**, without an RP.
- **PIM Prune** stops unwanted forwarding.
- **PIM Graft** restores forwarding to a previously pruned branch, with a Graft-Ack confirming receipt.
- Prune state is temporary, typically lasting about **3 minutes**. Without State Refresh, expiration can cause periodic reflooding.
- **PIM State Refresh** maintains Prune state without requiring repeated multicast data flooding.
- PIM-DM uses RPF checks and PIM Assert to help prevent forwarding loops and duplicate traffic.
- PIM-DM is generally less suitable than Sparse Mode for networks where multicast receivers are sparsely distributed.