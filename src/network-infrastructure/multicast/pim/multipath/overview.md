
# 1.6.c (viii) Multicast Multipath

**Multicast Multipath** allows a router to distribute different multicast streams across multiple **equal-cost upstream paths** using Equal-Cost Multipath (ECMP).

In normal unicast routing, ECMP allows traffic to be forwarded through multiple next hops toward the same destination.

Multicast routing operates differently. Because multicast traffic flows from sources toward receivers, the router must select an **upstream Reverse Path Forwarding (RPF) interface** through which it expects to receive each multicast stream.

By default, IOS XE does not distribute IPv4 multicast streams across multiple equal-cost RPF paths. Instead, it selects a single upstream path according to its RPF selection rules.

Multicast Multipath changes this behavior by allowing different multicast forwarding states to select different upstream interfaces.

## Multicast Multipath Topology

Consider the following network:

```text
            Src
     10.1.1.10 - .19
             |
             R1
          /  |  \
         /   |   \
        R2   R3   R4
         \   |   /
          \  |  /
             R5
             |
            Rec
```

R1 connects to the multicast source network, `10.1.1.0/24`.

R5 connects to the receiver network, `10.5.5.0/24`.

Assume OSPF provides three equal-cost paths from R5 toward the multicast source network:

- R5 → R2 → R1
- R5 → R3 → R1
- R5 → R4 → R1

All three paths are valid candidates for RPF selection.

However, **R5 does not automatically distribute multicast streams across these paths**.

Without Multicast Multipath, R5 selects a single RPF neighbor according to the normal RPF tie-breaking rules.

With Multicast Multipath, R5 can select different RPF neighbors for different multicast forwarding states.

For example:

| Multicast Channel | Selected Upstream Neighbor |
|---|---|
| (10.1.1.10, 232.1.1.1) | R2 |
| (10.1.1.11, 232.1.1.1) | R3 |
| (10.1.1.12, 232.1.1.1) | R4 |
| (10.1.1.13, 232.1.1.1) | R2 |

*This table illustrates possible load splitting, not the calculated result of a particular algorithm.*

R5 sends the corresponding PIM `(S,G)` Joins toward the selected upstream neighbors.

Each multicast stream then follows its own selected distribution path toward R5 and the receiver.

## Multicast Load Splitting vs. Load Balancing

An important distinction is that **Multicast Multipath operates on a per-state basis, not a per-packet basis**.

For example, suppose R5 selects R2 as the RPF neighbor for:

```text
(S,G) = (10.1.1.10, 232.1.1.1)
```

All packets belonging to that `(S,G)` stream are expected to arrive through R2.

R5 does not alternate packets from the same stream between R2, R3, and R4.

Instead, different multicast streams can use different upstream paths.

This preserves the RPF mechanism while allowing multiple paths to carry multicast traffic.

**Multicast Multipath distributes multicast forwarding states across equal-cost paths, rather than dividing individual multicast streams across multiple paths.**

It also does not guarantee equal bandwidth utilization. The resulting traffic distribution depends on the number of multicast states, their traffic rates, and the selected hashing algorithm.

## Multicast Multipath Algorithms

IOS XE supports three algorithms for IPv4 multicast ECMP load splitting.

| Algorithm | Hash Inputs | IOS XE Command |
|---|---|---|
| S-hash | Source or RP address | `ip multicast multipath` |
| Basic S-G-hash | Source and group addresses | `ip multicast multipath s-g-hash basic` |
| Next-hop-based S-G-hash | Source, group, and next-hop addresses | `ip multicast multipath s-g-hash next-hop-based` |

### S-Hash

The **S-hash algorithm** selects an RPF path based on the RPF address.

For source-specific `(S,G)` state, this is the multicast source address.

For shared-tree `(*,G)` state, it is the RP address.

Different sources can therefore use different upstream paths.

However, multiple groups originating from the same source select the same path when their RPF address and ECMP candidates are identical.

### Basic S-G Hashing

The **basic S-G-hash algorithm** uses both the source and multicast group addresses.

This allows different groups from the same source to use different RPF paths.

For example, traffic from `10.1.1.10` to `232.1.1.1` may use R2, while traffic from the same source to `232.1.1.2` may use R3.

This provides finer-grained load splitting than S-hash.

### Next-Hop-Based S-G Hashing

The **next-hop-based S-G-hash algorithm** includes the available next-hop addresses in the hash calculation.

This provides two important advantages over the other algorithms:

- **Avoids polarization:** Prevents identical hash calculations on different routers from causing inefficient use of available paths.
- **Improves RPF stability:** When an ECMP path fails, fewer multicast states unnecessarily change their selected upstream paths.

Next-hop-based S-G hashing is generally the preferred algorithm for multicast ECMP deployments.

## Multicast Multipath and RPF

Multicast Multipath does not eliminate or bypass RPF checking.

Each multicast forwarding state still has a selected RPF interface and upstream neighbor.

For an `(S,G)` state:

1. The router identifies the available equal-cost paths toward the source.
2. The configured multicast multipath algorithm selects an upstream path.
3. PIM sends an `(S,G)` Join toward the selected RPF neighbor.
4. Multicast packets arriving on the selected RPF interface can pass the RPF check.
5. Packets arriving on other interfaces fail the normal RPF check.

The difference is that **different multicast forwarding states can select different RPF interfaces**, allowing traffic to be distributed across multiple upstream paths.

Because multicast multipath changes RPF neighbor selection, Cisco recommends configuring the same algorithm consistently across routers in a redundant topology to avoid forwarding problems and potential loops.

## Basic IOS XE Configuration and Verification

IPv4 multicast multipath is disabled by default.

The following global configuration command enables source-based multicast load splitting:

```text
R5(config)# ip multicast multipath
```

The hash algorithm can be changed using the `s-g-hash` options:

```text
R5(config)# ip multicast multipath s-g-hash basic
```

Or:

```text
R5(config)# ip multicast multipath s-g-hash next-hop-based
```

Useful verification commands include:

```text
R5# show ip route 10.1.1.10
R5# show ip rpf 10.1.1.10 232.1.1.1
R5# show ip mroute
```

These commands allow you to verify:

- Whether multiple equal-cost routes exist toward the multicast source.
- Which RPF interface and neighbor are selected for a particular `(S,G)` channel.
- Which multicast forwarding states are installed and which incoming interfaces they use.

Multicast multipath can also be configured for specific VRFs using the `ip multicast vrf` command.

## Section Overview

The following pages examine Multicast Multipath in greater detail:

- **[ECMP and Multicast Load Splitting](ecmp-rpf.md):** Default RPF selection, equal-cost paths, multicast load splitting, and verification.
- **[S-Hash and Basic S-G Hashing](source-hash.md):** Source-based and source-group hashing, configuration, path selection, and polarization.
- **[Next-Hop-Based S-G Hashing](next-hop-hash.md):** Next-hop-aware hashing, polarization avoidance, RPF stability, and failover behavior.

## Key Points

- Multicast Multipath distributes multicast traffic across multiple equal-cost upstream paths.
- IPv4 multicast does not perform ECMP load splitting by default.
- Multicast load splitting operates on a **per-state basis**, not a per-packet basis.
- Each multicast forwarding state retains a selected RPF interface and neighbor.
- **S-hash** selects paths based on the multicast source or RP address.
- **Basic S-G-hash** uses source and group addresses, allowing different groups from the same source to use different paths.
- **Next-hop-based S-G-hash** also includes next-hop addresses, avoiding polarization and improving RPF stability.
- Multicast Multipath must be configured consistently across routers in redundant topologies.

## References

- [Cisco IOS XE: IP Multicast Load Splitting across Equal-Cost Paths](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/ip-multicast/ip-multicast/m_imc_load_splt_ecmp-0.html)
