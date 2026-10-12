# Shortest-Path Tree Switchover

In PIM Sparse Mode (PIM-SM), **Shortest-Path Tree (SPT) Switchover** allows a receiver-facing router to switch from the shared tree (RPT) to a source-specific tree (SPT).

Initially, receivers join the shared tree toward the Rendezvous Point (RP).

Multicast traffic follows:

```text
Source → RP → Receiver
```

However, forwarding traffic through the RP may introduce an unnecessary detour.

After receiving traffic from an active source, the Last-Hop Router (LHR) can establish a more direct path toward that source.

```text
Source → Receiver
```

This is called **RPT-to-SPT switchover**.

By default, IOS XE initiates SPT switchover upon receiving the first multicast packet from a new source.

## Example Topology

Consider the following network:

```text
       Source (S)
      10.10.10.10
           |
           R1 (FHR)
           |
           R2 -------- R3 (RP)
            \          / 3.3.3.3
             \        /
              R4 (LHR)
                 |
              Receiver
             239.1.1.1
```

Assume:

- R1 is the source-facing First-Hop Router (FHR).
- R3 is the RP for `239.1.1.1`.
- R4 is the receiver-facing Last-Hop Router (LHR).
- R4 reaches the RP through R3.
- R4 reaches the source through R2.

The RPF paths toward the RP and source are different.

## Step 1: Traffic Arrives Over the Shared Tree

The receiver sends an IGMP Membership Report for `239.1.1.1`.

R4 establishes `(*,G)` state and sends a `(*,G)` Join toward R3.

Meanwhile, R1 registers source `10.10.10.10` with the RP.

The RP establishes a native `(S,G)` path toward the source and forwards the traffic down the shared tree.

The resulting forwarding path is:

```text
Source → R1 → R2 → R3 (RP) → R4 → Receiver
```

At R4, the shared-tree state looks conceptually like this:

```text
(*,239.1.1.1)

Incoming Interface: Ethernet0/0 (toward RP)
Outgoing Interface:
  Ethernet0/2 (toward receiver)
```

Packets from source `10.10.10.10` arrive through R4's RPF interface toward the RP.

Although the RP receives the multicast traffic natively through an `(S,G)` tree, **R4 is still receiving that traffic over the RPT**.

> **Important:** The RP establishing an `(S,G)` source tree does not mean that the LHR has switched to the SPT. These are separate events.

## Step 2: LHR Initiates SPT Switchover

When R4 receives the first packet from `10.10.10.10`, the default SPT threshold of **0 kbps** is met.

R4 decides to switch to the source tree.

It performs an RPF lookup toward the source:

```text
Source:        10.10.10.10
RPF Interface: Ethernet0/1
RPF Neighbor:  R2
```

R4 sends a source-specific Join toward R2:

```text
R4 → R2: Join(10.10.10.10,239.1.1.1)
```

R2 already has a source-tree path toward R1, so it can add its interface toward R4 to the `(S,G)` outgoing interface list.

Alternatively, if an upstream router does not yet have the necessary `(S,G)` state, the Join continues hop by hop toward the source.

The resulting SPT is:

```text
Source → R1 → R2 → R4 → Receiver
```

This path bypasses the RP.

## Step 3: LHR Receives Traffic on the SPT

During the transition, R4 may temporarily receive packets from the same source over both paths:

```text
RPT: Source → R1 → R2 → R3 → R4

SPT: Source → R1 → R2 ─────→ R4
```

**R4 must not prune the RPT before the SPT is operational**, because doing so could interrupt multicast delivery.

When the source-tree forwarding conditions are satisfied, R4 sets the **SPT-bit** for `(10.10.10.10,239.1.1.1)`.

In the usual case where the source and RP RPF interfaces differ, receiving traffic on the correct source-facing RPF interface confirms that the SPT is operational.

The LHR can then forward traffic using `(S,G)` state instead of the shared-tree path.

Conceptually:

```text
(10.10.10.10,239.1.1.1)

Incoming Interface: Ethernet0/1 (toward source)
Outgoing Interface:
  Ethernet0/2 (toward receiver)

SPT-bit: Set
```

The SPT-bit prevents the router from forwarding duplicate copies received over the RPT.

> **Note:** The presence of an `(S,G)` entry alone does not prove that SPT switchover has completed. The SPT-bit is the important indicator that source-tree forwarding is active.

## Step 4: LHR Prunes the Source from the RPT

Once R4 has switched to the SPT, it no longer needs traffic from `10.10.10.10` over the shared tree.

R4 sends a **`Prune(S,G,rpt)`** toward the RP:

```text
R4 → R3: Prune(10.10.10.10,239.1.1.1,rpt)
```

This tells the upstream router:

**Stop forwarding traffic from source S for group G along the shared tree toward me.**

It does not remove the entire `(*,G)` shared-tree membership.

For example, suppose another source begins sending to the same group:

```text
Source 1: 10.10.10.10
Source 2: 10.20.20.20
Group:    239.1.1.1
```

R4 may receive traffic from Source 1 over its SPT while continuing to receive traffic from Source 2 over the RPT.

Therefore:

- `Prune(S,G,rpt)` removes shared-tree forwarding for one source.
- `Prune(*,G)` removes shared-tree interest for the entire group.

The `(S,G,rpt)` Prune propagates toward the RP only as far as necessary. An upstream router may still need the source's traffic on the shared tree for other downstream receivers.

## Step 5: SPT Switchover Completes

After switchover, multicast traffic follows:

```text
Source → R1 → R2 → R4 → Receiver
```

R4 now maintains two kinds of forwarding state:

| State | Purpose |
|---|---|
| `(*,G)` | Maintains shared-tree interest toward the RP |
| `(S,G)` | Forwards packets from the specific source over the SPT |

The `(*,G)` state can remain even after R4 has switched to the SPT for one source.

**SPT switchover is performed per `(S,G)`, not necessarily for the entire multicast group.**

## SPT-Bit

The **SPT-bit** indicates that a router has completed the relevant source-tree forwarding transition.

In Cisco IOS XE, it is represented by the `T` flag in `show ip mroute`.

The SPT-bit is not carried in a PIM Join/Prune message. It is local forwarding state.

RFC 7761 defines conditions for setting the bit, including:

- The router has appropriate `(S,G)` Join state.
- Traffic arrives through the correct RPF interface toward the source.
- Additional conditions establish that source-tree forwarding is safe, particularly when the RPF interfaces toward the source and RP overlap.

For example, when the RPF interfaces toward the source and RP are different, receiving packets over the source-facing interface is normally sufficient to confirm SPT completion.

When both RPF paths use the same interface, identifying SPT completion may require additional information, including the upstream RPF neighbor or PIM Assert state.

The essential rule is that **the router must not prematurely stop RPT forwarding while the SPT is still being established**.

## SPT Threshold

The SPT threshold determines when an LHR should initiate switchover from the RPT to the SPT.

It is configured globally using:

```text
R1(config)# ip pim spt-threshold {0 | infinity}
```

The two choices are:

| Threshold | Behavior |
|---|---|
| `0` | Switch to the SPT upon receiving the first packet from a source |
| `infinity` | Remain on the RPT; do not initiate receiver-side SPT switchover |

The default is **0**.

### Modern IOS XE Limitation

Historically, the command allowed a bandwidth threshold in kilobits per second.

For example, older documentation describes configuring a threshold of 1000 kbps, allowing low-bandwidth traffic to remain on the RPT while higher-bandwidth traffic triggers SPT switchover.

However, on modern Catalyst IOS XE platforms, **only 0 and infinity are supported**.

Nonzero numeric thresholds are not supported on modern IOS XE.

### Disable SPT Switchover

To force the LHR to remain on the shared tree:

```text
R4(config)# ip pim spt-threshold infinity
```

Now, traffic continues to follow:

```text
Source → R1 → R2 → R3 (RP) → R4 → Receiver
```

R4 does not initiate an `(S,G)` Join toward the source merely because it receives multicast traffic over the RPT.

However, this command does **not** prevent the RP from joining the source tree during source registration.

The RP may still establish a native `(S,G)` path to receive traffic from the FHR.

The threshold controls **receiver-side RPT-to-SPT switchover**, not the RP's source-registration process.

## Apply the SPT Threshold to Specific Groups

The optional `group-list` parameter allows the SPT threshold to apply only to selected multicast groups.

For example, suppose:

- Groups in `239.1.1.0/24` must remain on the RPT.
- Other multicast groups should use the default SPT switchover behavior.

Configure a standard ACL:

```text
R4(config)# ip access-list standard STAY-RPT
R4(config-std-nacl)# permit 239.1.1.0 0.0.0.255
```

Apply the policy:

```text
R4(config)# ip pim spt-threshold infinity group-list STAY-RPT
```

The result is:

| Group | SPT Threshold |
|---|---|
| `239.1.1.1` | Infinity |
| `239.1.1.100` | Infinity |
| `239.2.2.2` | Default (0) |

The ACL matches **multicast group addresses**, not source addresses.

Only permitted groups are subject to the configured infinity threshold. Other groups retain the default behavior.

This command does not block multicast traffic or prevent receivers from joining the specified groups. It only controls whether the router attempts to switch to an SPT.

## IPv6 SPT Threshold

IPv6 PIM-SM supports the same basic RPT-to-SPT switchover concept.

However, the IPv6 command exposes only the `infinity` option.

Disable automatic SPT switchover:

```text
R4(config)# ipv6 pim spt-threshold infinity
```

Apply it to selected groups:

```text
R4(config)# ipv6 pim spt-threshold infinity group-list IPV6-STAY-RPT
```

The group list must reference a suitable IPv6 ACL.

Restore the default behavior:

```text
R4(config)# no ipv6 pim spt-threshold infinity
```

By default, the LHR switches to the SPT after receiving traffic from a new source.

> **Important:** Cisco documents that configuring IPv6 `spt-threshold infinity` after a router has already joined the SPT does not automatically cause it to switch back to the RPT. Configure the desired policy before testing a new flow.

## IOS XE Verification

### Verify SPT Threshold Configuration

```text
R4# show running-config | include ip pim spt-threshold
```

If no threshold is configured, the default is 0.

### Verify the RPF Paths

Check the RPF path toward the RP:

```text
R4# show ip rpf 3.3.3.3
```

Check the RPF path toward the source:

```text
R4# show ip rpf 10.10.10.10
```

In the example topology, these lookups should identify different upstream interfaces.

### Verify SPT Switchover

Use:

```text
R4# show ip mroute 239.1.1.1
```

After SPT switchover, illustrative output might look like:

```text
(*, 239.1.1.1), 00:05:00/00:02:59, RP 3.3.3.3, flags: S
  Incoming interface: Ethernet0/0, RPF nbr 10.0.34.3
  Outgoing interface list:
    Ethernet0/2, Forward/Sparse, 00:05:00/00:02:59

(10.10.10.10, 239.1.1.1), 00:04:30/00:02:59, flags: T
  Incoming interface: Ethernet0/1, RPF nbr 10.0.24.2
  Outgoing interface list:
    Ethernet0/2, Forward/Sparse, 00:04:30/00:02:59
```

Notice:

- The `(*,G)` incoming interface points toward the RP.
- The `(S,G)` incoming interface points toward the source.
- The `(S,G)` entry has the **`T` flag**, indicating that the SPT-bit is set.
- Both entries can coexist.

The `(*,G)` outgoing interface remains present because the receiver still has group membership. Source-specific RPT pruning prevents duplicate traffic for the source that has switched to the SPT.

### Important Multicast Routing Flags

| Flag | Meaning |
|---|---|
| `S` | Sparse Mode |
| `J` | Join SPT |
| `T` | SPT-bit set |
| `R` | RP-bit set, associated with source-specific state toward the RP |
| `P` | Pruned |

The `J` flag indicates that the applicable SPT threshold condition has been met and the router should join the SPT.

The `T` flag indicates that the `(S,G)` SPT-bit has been set.

**The `J` flag alone does not prove that SPT switchover has completed.**

To confirm forwarding over the SPT, examine the `(S,G)` entry, its incoming interface, the `T` flag, and actual packet forwarding.

## Why Disable SPT Switchover?

Keeping receivers on the RPT can be useful in networks with many sources transmitting to the same multicast group.

For example:

```text
100 sources → Same multicast group
```

With automatic SPT switchover, an LHR may establish source-specific forwarding state for many active sources.

This can increase multicast state and resource consumption across the network.

Using `ip pim spt-threshold infinity` avoids receiver-initiated SPT construction for those groups.

However, remaining on the RPT can result in:

- Longer forwarding paths through the RP.
- Increased bandwidth usage on links near the RP.
- Higher latency compared with a direct SPT path.

The trade-off is between **more direct forwarding paths** and **reduced source-specific forwarding state**.

## Key Points

- SPT switchover allows an LHR to receive multicast traffic directly along a source-specific tree instead of through the RP.
- The LHR initiates switchover by sending a **`Join(S,G)` toward the source**.
- The router continues using the RPT until the SPT is ready, avoiding premature loss of multicast traffic.
- After switching, a **`Prune(S,G,rpt)`** suppresses forwarding of the specific source over the shared tree.
- The **SPT-bit (`T` flag)** indicates that source-tree forwarding is active.
- SPT switchover is **per `(S,G)`**, so other sources may continue using the RPT.
- The default `ip pim spt-threshold` is **0**, triggering switchover after the first packet.
- On modern Catalyst IOS XE, only **`0` and `infinity`** are supported, not arbitrary nonzero kbps thresholds.
- `ip pim spt-threshold infinity` prevents receiver-side SPT switchover but does not prevent the RP from joining the source tree.
- The optional `group-list` parameter applies the policy to selected multicast groups.
- IPv6 uses `ipv6 pim spt-threshold infinity` to disable automatic switchover.
