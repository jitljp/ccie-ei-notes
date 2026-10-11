# RPF for Source and Shared Trees

In PIM Sparse Mode, multicast traffic can be forwarded using two types of distribution trees:

- **Source Tree (SPT):** Rooted at the multicast source.
- **Shared Tree (RPT):** Rooted at the Rendezvous Point (RP).

The **Reverse Path Forwarding (RPF)** lookup differs depending on which tree is used.

| Multicast State | Distribution Tree | RPF Lookup Toward |
|---|---|---|
| `(S,G)` | Source Tree (SPT) | Source IP (S) |
| `(*,G)` | Shared Tree (RPT) | RP IP |

Consequently, a router can have different RPF interfaces and neighbors for `(S,G)` and `(*,G)` state associated with the same multicast group.

## RPF for (S,G) State

For source-tree forwarding, the router performs an RPF lookup toward the **multicast source (S)**.

For example:

```text
Source: 10.10.10.10
Group:  239.1.1.1
```

The RPF lookup for:

```text
(10.10.10.10, 239.1.1.1)
```

is performed toward `10.10.10.10`.

The resulting RPF interface is the expected incoming interface for packets forwarded along the SPT.

PIM also uses this lookup to determine the upstream neighbor toward which `(S,G)` Join messages should be sent.

## RPF for (*,G) State

For shared-tree forwarding, the router performs an RPF lookup toward the **RP**, rather than the multicast source.

For example:

```text
RP:     1.1.1.1
Group:  239.1.1.1
```

The RPF lookup for:

```text
(*, 239.1.1.1)
```

is performed toward `1.1.1.1`.

The resulting RPF interface is the expected incoming interface for traffic forwarded along the RPT.

PIM uses this lookup to determine where to send upstream `(*,G)` Join messages.

Because the shared tree is rooted at the RP, **the same RPF interface can be used for traffic from multiple multicast sources**, even if those sources are located in different parts of the network.

## Example: Different RPF Interfaces for (S,G) and (*,G)

Consider the following topology:

```text
    Source (S)                RP
   10.10.10.10              1.1.1.1
        |                      |
        R1                     R5
        |                      |
        R2                     R3
         \                    /
          \                  /
      E0/0 \                / E0/1
            \              /
                   R4
                   |
                 E0/2
                   |
                Receiver
               239.1.1.1
```

R4 has two different upstream paths:

```text
Toward Source (10.10.10.10):
→ Ethernet0/0, via R2

Toward RP (1.1.1.1):
→ Ethernet0/1, via R3
```

### Shared-Tree Forwarding

Initially, R4 joins the shared tree toward the RP.

Its `(*,G)` state uses:

```text
(*, 239.1.1.1)

RPF Lookup:    1.1.1.1
RPF Interface: Ethernet0/1
RPF Neighbor:  R3
```

Multicast traffic forwarded along the shared tree arrives from R3 on Ethernet0/1.

### Source-Tree Forwarding

When R4 switches to the SPT for source `10.10.10.10`, it uses source-based RPF:

```text
(10.10.10.10, 239.1.1.1)

RPF Lookup:    10.10.10.10
RPF Interface: Ethernet0/0
RPF Neighbor:  R2
```

Multicast traffic forwarded along the source tree arrives from R2 on Ethernet0/0.

The difference is:

| State | RPF Target | RPF Interface |
|---|---|---|
| `(*,239.1.1.1)` | RP `1.1.1.1` | Ethernet0/1 |
| `(10.10.10.10,239.1.1.1)` | Source `10.10.10.10` | Ethernet0/0 |

**The multicast source and group remain the same, but the expected incoming interface changes depending on the distribution tree used for forwarding.**

## (S,G) State Does Not Always Mean SPT Forwarding

An important detail is that **an `(S,G)` entry can exist before the router has completed switching to the SPT**.

During SPT switchover, the router may temporarily maintain `(S,G)` state while still forwarding packets using the shared tree.

The **SPT-bit**, displayed as the `T` flag in `show ip mroute`, indicates that the source-tree forwarding state is active.

Until the SPT-bit is set, packets may continue to be forwarded using the RPT.

This prevents traffic interruption while the new source tree is being established.

The SPT switchover process, including PIM Join/Prune signaling, is covered in the PIM Sparse Mode section.

## Verification

Use `show ip mroute` to examine the incoming interfaces associated with both types of state:

```text
R4# show ip mroute 239.1.1.1

(*, 239.1.1.1), 00:05:00/00:02:59, RP 1.1.1.1, flags: S
  Incoming interface: Ethernet0/1, RPF nbr 10.0.34.3
  Outgoing interface list:
    Ethernet0/2, Forward/Sparse, 00:05:00/00:02:59

(10.10.10.10, 239.1.1.1), 00:04:00/00:02:59, flags: T
  Incoming interface: Ethernet0/0, RPF nbr 10.0.24.2
  Outgoing interface list:
    Ethernet0/2, Forward/Sparse, 00:04:00/00:02:59
```

Notice that the incoming interfaces are different:

```text
(*,G) → Ethernet0/1 (toward RP)

(S,G) → Ethernet0/0 (toward Source)
```

Both entries can remain present after SPT switchover. This does not mean that the router forwards duplicate packets from both trees.

You can also verify the underlying RPF lookups independently:

```text
! RPF toward the source
R4# show ip rpf 10.10.10.10

! RPF toward the RP
R4# show ip rpf 1.1.1.1
```

The same concepts apply to IPv6 multicast, using `show ipv6 rpf` and `show ipv6 mroute`.

## Key Points

- **(S,G) source-tree RPF** uses the multicast source address.
- **(*,G) shared-tree RPF** uses the RP address.
- The same multicast group can have different RPF interfaces and neighbors for its source tree and shared tree.
- PIM sends `(S,G)` Joins toward the source and `(*,G)` Joins toward the RP.
- An `(S,G)` entry can exist before SPT switchover is complete. The `T` flag indicates active SPT forwarding.
- Use `show ip mroute` to compare the incoming interfaces for `(S,G)` and `(*,G)` state.