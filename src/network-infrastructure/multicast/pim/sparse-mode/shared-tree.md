# PIM-SM Shared Tree and (*,G) State

In traditional PIM Sparse Mode (PIM-SM), the **Shared Tree**, also called the **Rendezvous Point Tree (RPT)**, is a multicast distribution tree rooted at a Rendezvous Point (RP).

Unlike a source-specific tree, which is rooted at a particular multicast source, the shared tree can carry traffic from **multiple sources sending to the same multicast group**.

The shared tree is represented by **`(*,G)` multicast state**, where:

- `*` represents any multicast source.
- `G` represents a particular multicast group.

For example:

```text
(*, 239.1.1.1)
```

This state represents shared-tree forwarding for group `239.1.1.1`, regardless of which source originated the traffic.

## Shared-Tree Operation

PIM-SM uses a **pull model**, meaning that multicast traffic is not initially flooded throughout the network.

Instead, routers establish the shared tree by sending **`(*,G)` Join messages toward the RP** when downstream receivers express interest in a multicast group.

The resulting tree connects the RP to all interested receivers.

Consider the following topology:

```text
                R1 (RP)
                1.1.1.1
                   |
                   |
                   R2
                  /  \
                 /    \
                R3    R4
                |      |
               PC1    PC2
                |      |
             Group  Group
          239.1.1.1 239.1.1.1
```

Both PC1 and PC2 want to receive multicast traffic for `239.1.1.1`.

### Step 1: PC1 Joins the Group

PC1 sends an IGMP Membership Report for `239.1.1.1`.

R3, acting as the receiver-facing PIM Designated Router (DR), creates `(*,G)` state and sends a PIM Join toward the RP.

```text
R3 → R2: Join (*,239.1.1.1)
```

R2 records that R3 requires traffic for the group and adds its interface toward R3 to the Outgoing Interface List (OIL).

Because R2 does not yet have an upstream shared-tree Join for the group, it sends its own Join toward R1:

```text
R2 → R1: Join (*,239.1.1.1)
```

The shared tree is now established:

```text
                 R1 (RP)
                    |
                    R2
                    |
                    R3
                    |
                   PC1
```

> **Important:** The tree can be established even if no multicast source is currently sending traffic. `(*,G)` state represents receiver interest, not the existence of an active source.

### Step 2: PC2 Joins the Same Group

PC2 subsequently sends an IGMP Membership Report for `239.1.1.1`.

R4 sends a `(*,G)` Join toward R2.

R2 already has upstream `(*,G)` state toward R1, so it simply adds its interface toward R4 to the OIL.

```text
                 R1 (RP)
                    |
                    R2
                   /  \
                  /    \
                 R3    R4
                 |      |
                PC1    PC2
```

**R2 does not need to establish another upstream shared-tree branch.** The existing branch toward the RP is shared by both receivers.

Once multicast traffic reaches the RP and is forwarded along the RPT, R2 can replicate it onto both downstream branches.

## (*,G) Forwarding State

A router's `(*,G)` state identifies the upstream path toward the RP and the downstream interfaces that require multicast traffic.

For example, consider R2:

```text
(*,239.1.1.1)

Incoming Interface: Ethernet0/0 (toward RP)
Outgoing Interfaces:
  Ethernet0/1 (toward R3)
  Ethernet0/2 (toward R4)
```

The most important components are:

| Component | Purpose |
|---|---|
| Group (G) | Identifies the multicast group |
| RP | Identifies the root of the shared tree |
| RPF Interface | Expected incoming interface for shared-tree forwarding |
| RPF Neighbor | Upstream PIM neighbor toward the RP |
| OIL | Downstream interfaces requiring multicast traffic |
| Join/Prune State | Tracks downstream interest and upstream participation in the shared tree |

### RPF Toward the RP

For `(*,G)` state, the router performs its RPF lookup toward the **RP**, not toward any individual multicast source.

For example:

```text
RP:     1.1.1.1
Group:  239.1.1.1
```

R2 uses `1.1.1.1` as its RPF lookup address.

Suppose the selected route points through Ethernet0/0.

```text
RPF Lookup:    1.1.1.1
RPF Interface: Ethernet0/0
RPF Neighbor:  R1
```

PIM uses this information to determine the upstream neighbor for `(*,G)` Join/Prune messages.

For data forwarded along the shared tree, Ethernet0/0 is also the expected incoming interface.

### Outgoing Interface List

The OIL is built from downstream multicast interest.

That interest can come from:

- **Local receivers:** IGMP or MLD membership state.
- **Downstream PIM routers:** Received `(*,G)` Join messages.

For example:

```text
R2 OIL for (*,239.1.1.1):

Ethernet0/1 → R3
Ethernet0/2 → R4
```

If multicast traffic arrives along the shared tree, R2 can forward copies onto both interfaces.

The OIL is dynamic. Interfaces are added or removed as receiver interest and PIM state change.

## Multiple Sources Share the Same Tree

Suppose two sources begin transmitting to `239.1.1.1`:

```text
S1: 10.10.10.10
S2: 10.20.20.20
```

Both sources can deliver traffic through the same RP-rooted shared tree.

```text
             S1        S2
              \        /
               \      /
                RP (R1)
                   |
                   R2
                  /  \
                 R3  R4
                 |    |
                PC1  PC2
```

The shared-tree state remains:

```text
(*,239.1.1.1)
```

The router does not need a separate `(*,G)` entry for each source.

However, source-specific `(S,G)` state may also exist as part of source registration, native forwarding toward the RP, or SPT switchover.

**The existence of a shared tree does not mean routers never maintain `(S,G)` state.** It means the RP-rooted distribution tree itself is maintained using `(*,G)` state.

The process of delivering traffic from new sources to the RP is covered in [Source Registration](source-registration.md).

## Maintaining the Shared Tree

PIM-SM maintains shared-tree state using periodically refreshed Join/Prune information.

The RFC 7761 default timers are:

| Timer | Default |
|---|---|
| PIM Join/Prune Period | 60 seconds |
| Join/Prune Holdtime | 210 seconds |

The default Holdtime is calculated as:

```text
Holdtime = 3.5 × Join/Prune Period
         = 3.5 × 60
         = 210 seconds
```

Routers periodically send `(*,G)` Joins upstream to maintain the shared tree.

When a downstream router stops refreshing its Join state, the upstream router eventually removes that downstream interface from its forwarding state.

If no downstream interfaces require the group anymore, the router can send a `(*,G)` Prune toward its upstream neighbor.

For example, if PC2 leaves the group:

```text
                 R1 (RP)
                    |
                    R2
                   /  \
                  R3   X R4
                  |
                 PC1
```

R2 removes the branch toward R4 but continues forwarding toward R3.

Only when R2 no longer has any downstream interest does it need to remove its upstream shared-tree Join.

> **Note:** Join suppression and Prune override mechanisms can affect the timing of actual Join/Prune transmissions on multiaccess LANs. These are covered in [PIM Join and Prune Messages](join-prune.md).

## (*,G) State at the RP

The RP is the root of the shared tree.

Unlike downstream routers, the RP does not need to send a `(*,G)` Join farther upstream toward another RP.

For example:

```text
R1# show ip mroute 239.1.1.1

(*, 239.1.1.1), 00:05:00/00:02:59, RP 1.1.1.1, flags: S
  Incoming interface: Null, RPF nbr 0.0.0.0
  Outgoing interface list:
    Ethernet0/0, Forward/Sparse, 00:05:00/00:02:59
```

Notice:

```text
Incoming interface: Null, RPF nbr 0.0.0.0
```

This is expected for the RP's own `(*,G)` entry.

The RP is the root of the shared tree, so it has no upstream RPF neighbor toward itself.

The outgoing interface identifies the downstream branch containing interested receivers.

## Verification

### `show ip mroute`

Use:

```text
R2# show ip mroute 239.1.1.1
```

Example output:

```text
(*, 239.1.1.1), 00:05:00/00:02:59, RP 1.1.1.1, flags: S
  Incoming interface: Ethernet0/0, RPF nbr 10.0.12.1
  Outgoing interface list:
    Ethernet0/1, Forward/Sparse, 00:05:00/00:02:59
    Ethernet0/2, Forward/Sparse, 00:03:20/00:02:59
```

Important fields include:

| Field | Meaning |
|---|---|
| `(*,239.1.1.1)` | Shared-tree state for the group |
| `RP 1.1.1.1` | Rendezvous Point for the group |
| `flags: S` | Sparse Mode multicast state |
| `Incoming interface` | RPF interface toward the RP |
| `RPF nbr` | Upstream neighbor toward the RP |
| `Outgoing interface list` | Downstream interfaces requiring multicast traffic |

### Additional Commands

| Command | Purpose |
|---|---|
| `show ip mroute` | Display multicast forwarding state |
| `show ip pim rp mapping` | Verify the RP selected for the group |
| `show ip rpf 1.1.1.1` | Verify the RPF path toward the RP |
| `show ip pim neighbor` | Verify upstream and downstream PIM neighbors |
| `show ip igmp groups` | Verify locally learned IGMP memberships |

## (*,G) State After SPT Switchover

An important point is that **`(*,G)` state does not necessarily disappear after an LHR switches to the SPT**.

For example, the router may maintain both:

```text
(*,239.1.1.1)

(10.10.10.10,239.1.1.1)
```

The first entry represents the shared-tree state, while the second represents source-specific state.

After successful SPT switchover, multicast packets from `10.10.10.10` can be forwarded using source-tree state instead of the shared tree.

Source-specific RPT pruning prevents unnecessary copies from arriving along the old shared-tree path.

The router may retain its `(*,G)` state for other sources or receiver interest.

The transition is covered in [Shortest-Path Tree Switchover](spt-switchover.md).

## Key Points

- The **Shared Tree (RPT)** is a multicast distribution tree rooted at the RP.
- `(*,G)` state represents shared-tree forwarding for a group, regardless of source.
- Receiver-facing routers send `(*,G)` Joins upstream toward the RP.
- Intermediate routers add downstream interfaces to their OILs and propagate Joins upstream when necessary.
- **RPF lookups for `(*,G)` state are performed toward the RP**, not toward individual sources.
- Multiple receivers and multicast sources can share the same RP-rooted distribution tree.
- The default Join/Prune Period is **60 seconds**, with a Holdtime of **210 seconds**.
- The RP has no upstream RPF neighbor for its own `(*,G)` state.
- `(*,G)` and `(S,G)` state can coexist, particularly during and after SPT switchover.