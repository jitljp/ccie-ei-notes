# S-Hash and Basic S-G Hashing

Multicast Multipath uses hashing algorithms to distribute multicast forwarding states across equal-cost RPF paths.

This page examines two of the three algorithms supported by Cisco IOS XE:

- **S-hash:** Selects an RPF path based on the multicast source or RP address.
- **Basic S-G-hash:** Selects an RPF path based on both the source (or RP) and multicast group addresses.

The third algorithm, **next-hop-based S-G-hash**, is covered in the following page.

Both algorithms are **deterministic**, meaning the same hash inputs and equal-cost path set produce the same RPF selection.

## S-Hash: Source-Based Load Splitting

The **S-hash algorithm** uses the RPF address to select one of the available equal-cost upstream paths.

The RPF address depends on the multicast forwarding state:

| Multicast State | RPF Address Used for Hashing |
|---|---|
| `(S,G)` | Source address (S) |
| `(*,G)` | Rendezvous Point (RP) address |

The multicast group address is **not included** in the S-hash calculation.

Consequently, multicast states with the same RPF address and equal-cost path set select the same upstream path.

### IOS XE Configuration

Enable S-hash multicast load splitting using the following global configuration command:

```text
R5(config)# ip multicast multipath
```

This enables multicast ECMP load splitting using the S-hash algorithm.

Without this command, IOS XE uses its default highest-PIM-neighbor method for selecting among equal-cost RPF paths.

### S-Hash Example

Consider the following topology:

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

R5 has three equal-cost paths toward the source network, `10.1.1.0/24`.

Assume the source at `10.1.1.10` sends traffic to three multicast groups:

```text
(10.1.1.10, 232.1.1.1)
(10.1.1.10, 232.1.1.2)
(10.1.1.10, 232.1.1.3)
```

Because all three states have the same source address, the S-hash algorithm uses the same hash input:

```text
S = 10.1.1.10
```

Suppose this produces R2 as the selected upstream neighbor.

All three multicast states use R2:

| Multicast State | RPF Neighbor |
|---|---|
| (10.1.1.10, 232.1.1.1) | R2 |
| (10.1.1.10, 232.1.1.2) | R2 |
| (10.1.1.10, 232.1.1.3) | R2 |

**Changing the multicast group address does not change the S-hash result.**

However, if the source address changes, the hash result may change.

For example:

| Source | Group | RPF Neighbor |
|---|---|---|
| 10.1.1.10 | 232.1.1.1 | R2 |
| 10.1.1.11 | 232.1.1.1 | R4 |
| 10.1.1.12 | 232.1.1.1 | R3 |
| 10.1.1.13 | 232.1.1.1 | R2 |

*These selections are illustrative, not calculated hash results.*

Different sources can use different RPF neighbors, distributing multicast traffic across the three upstream paths.

However, different sources are not guaranteed to select different paths. Hash collisions can result in multiple sources selecting the same neighbor.

### S-Hash with Shared Trees

S-hash also operates on `(*,G)` state in PIM Sparse Mode.

For shared trees, the RPF lookup is performed toward the RP rather than an individual multicast source.

Suppose multiple multicast groups use the same RP:

```text
RP: 10.1.1.1

(*, 239.1.1.1)
(*, 239.1.1.2)
(*, 239.1.1.3)
```

All three states use `10.1.1.1` as their RPF address.

Therefore, assuming the same equal-cost paths toward the RP, all three states select the same RPF neighbor.

If different groups use different RPs, their shared trees can select different RPF paths.

**S-hash distributes source trees by source address and shared trees by RP address, not by multicast group address.**

## Basic S-G Hashing

The **basic S-G-hash algorithm** uses both the source and multicast group addresses when selecting an RPF path.

Unlike S-hash, changing either address can change the selected upstream neighbor.

This provides finer-grained load splitting, particularly when a small number of sources transmit traffic to many multicast groups.

### IOS XE Configuration

Enable basic S-G hashing using:

```text
R5(config)# ip multicast multipath s-g-hash basic
```

This selects the basic S-G-hash algorithm instead of S-hash.

The hash is deterministic, so the same source, group, and equal-cost path set produce the same selection.

### Basic S-G-Hash Example

Consider the same topology, with R5 having three equal-cost RPF paths through R2, R3, and R4.

Suppose a single source sends traffic to several multicast groups:

```text
Source: 10.1.1.10

Groups:
232.1.1.1
232.1.1.2
232.1.1.3
232.1.1.4
```

With S-hash, all four multicast states would select the same RPF neighbor because the source address is identical.

With basic S-G hashing, the group address is also included in the calculation.

For example:

| Multicast State | RPF Neighbor |
|---|---|
| (10.1.1.10, 232.1.1.1) | R2 |
| (10.1.1.10, 232.1.1.2) | R4 |
| (10.1.1.10, 232.1.1.3) | R3 |
| (10.1.1.10, 232.1.1.4) | R2 |

*These selections are illustrative.*

Even though all four multicast streams originate from the same source, they can use different upstream paths.

This is particularly useful for applications such as IPTV, where a small number of servers may transmit hundreds or thousands of multicast channels.

**Basic S-G hashing allows multicast groups from the same source to be distributed across multiple RPF paths.**

> **Bidir-PIM:** Cisco IOS XE documents that the basic S-G-hash algorithm ignores bidirectional PIM groups. It does not provide group-based load splitting for those groups.

## S-Hash vs. Basic S-G Hashing

The key difference is whether the multicast group address influences RPF path selection.

| Feature | S-Hash | Basic S-G-Hash |
|---|---|---|
| Command | `ip multicast multipath` | `ip multicast multipath s-g-hash basic` |
| Hash inputs | RPF address | Source/RPF address and group address |
| Different sources can select different paths | Yes | Yes |
| Different groups from the same source can select different paths | No | Yes |
| Deterministic | Yes | Yes |
| Subject to polarization | Yes | Yes |

### When Is Each Algorithm Useful?

**S-hash** is useful when there are many independent multicast sources.

For example, if ten sources each transmit to one multicast group, the source addresses provide ten different hash inputs.

However, if one source transmits to ten groups, S-hash uses only one hash input, so all ten streams use the same RPF path.

**Basic S-G-hash** is useful when multicast sources transmit to multiple groups.

For example, ten sources each transmitting to ten groups provide 100 distinct `(S,G)` combinations that can be hashed independently.

This provides more opportunities to distribute traffic across the available upstream paths.

Neither algorithm guarantees an equal number of multicast streams on every path or equal bandwidth utilization.

## Predictability

Both S-hash and basic S-G-hash use deterministic calculations without randomization.

This means their RPF selections are predictable when the hash inputs and available equal-cost paths are known.

For example, suppose a router has three equal-cost upstream paths.

If a particular `(S,G)` combination selects the second path, another router with the same number of equal-cost paths and the same hashing algorithm can calculate the same path position.

The actual RPF neighbor depends on which next hop occupies that position in the router's ECMP path set.

This predictability can be useful when manually engineering multicast traffic distribution.

For example, administrators can select source or group addresses that produce a desired distribution across a known ECMP topology.

However, deterministic hashing also introduces a potential problem called **polarization**.

## Polarization

**Polarization** occurs when deterministic hashing causes multiple routers to make correlated path selections, resulting in some available links being underutilized.

Both S-hash and basic S-G-hash are subject to polarization.

### Why Polarization Occurs

These algorithms calculate a hash using the source address, or the source and group addresses.

They do not incorporate the actual next-hop IP addresses into the hash calculation.

Consequently, routers with the same number of equal-cost paths can produce the same selection index for a particular multicast state.

Consider two routers, R5 and R6, each with two equal-cost upstream paths.

```text
R5:
  Path 1 → R2
  Path 2 → R3

R6:
  Path 1 → R7
  Path 2 → R8
```

Suppose both routers use S-hash.

For a particular source, both routers calculate a hash that selects Path 1:

```text
R5 → R2
R6 → R7
```

For other sources, both routers may select Path 2.

Because the hash calculations are correlated, traffic arriving at different routers may repeatedly select corresponding path positions.

In more complex topologies, this can cause traffic to concentrate on certain links while other equal-cost links carry little or no traffic.

### Polarization and Multicast Distribution

Polarization is especially problematic in multi-stage ECMP topologies.

For example, suppose a downstream router distributes multicast traffic across two upstream routers.

Each upstream router also has two equal-cost paths toward the sources.

Because all routers use the same deterministic hash algorithm, the upstream routers may repeatedly select the same relative path for the particular subset of multicast states they receive.

As a result, some available links may remain unused even though multiple equal-cost paths exist.

**The problem is not that the hash fails to distribute traffic at an individual router. It is that correlated decisions across multiple routers can produce inefficient distribution throughout the network.**

Changing from S-hash to basic S-G-hash provides more hash inputs but does not eliminate polarization.

The next-hop-based S-G-hash algorithm addresses this problem by including the actual next-hop addresses in the hash calculation.

It is covered in [Next-Hop-Based S-G Hashing](next-hop-hash.md).

## IOS XE Verification

### Verify the Configured Algorithm

Check the running configuration:

```text
R5# show running-config | include ip multicast multipath
```

For S-hash:

```text
ip multicast multipath
```

For basic S-G-hash:

```text
ip multicast multipath s-g-hash basic
```

### Verify RPF Selection with S-Hash

After configuring S-hash, compare RPF selections for different sources:

```text
R5# show ip rpf 10.1.1.10
R5# show ip rpf 10.1.1.11
R5# show ip rpf 10.1.1.12
```

Different source addresses may produce different RPF interfaces and neighbors.

To verify that group changes do not affect S-hash selection, compare:

```text
R5# show ip rpf 10.1.1.10 232.1.1.1
R5# show ip rpf 10.1.1.10 232.1.1.2
R5# show ip rpf 10.1.1.10 232.1.1.3
```

Assuming the same equal-cost RPF paths, all three queries should select the same RPF interface.

### Verify RPF Selection with Basic S-G-Hash

After enabling basic S-G hashing:

```text
R5(config)# ip multicast multipath s-g-hash basic
```

Compare the same source with different groups:

```text
R5# show ip rpf 10.1.1.10 232.1.1.1
R5# show ip rpf 10.1.1.10 232.1.1.2
R5# show ip rpf 10.1.1.10 232.1.1.3
R5# show ip rpf 10.1.1.10 232.1.1.4
```

Different groups may select different RPF neighbors.

For example, one query might display:

```text
RPF information for ? (10.1.1.10)
  RPF interface: Ethernet0/0
  RPF neighbor: ? (10.2.5.2)
  RPF route/mask: 10.1.1.0/255.255.255.0
  RPF type: unicast (ospf 1)
```

Another query might select:

```text
RPF interface: Ethernet0/2
RPF neighbor: 10.4.5.4
```

These are illustrative results. The actual interface selected depends on the hash calculation.

### Verify Installed Multicast States

Use:

```text
R5# show ip mroute
```

For each `(S,G)` entry, examine the incoming interface and RPF neighbor.

A group-specific RPF lookup can be performed even before multicast forwarding state exists, but `show ip mroute` displays the states that have actually been created.

This distinction is useful when testing multicast multipath with several sources and groups.

## Key Points

- **S-hash** uses the RPF address to select an upstream path.
- For `(S,G)` state, the RPF address is the source. For `(*,G)` state, it is the RP.
- Configure S-hash with `ip multicast multipath`.
- Different groups sharing the same source and ECMP path set select the same RPF path with S-hash.
- **Basic S-G-hash** uses both the source/RPF address and group address.
- Configure basic S-G-hash with `ip multicast multipath s-g-hash basic`.
- Basic S-G-hash allows different groups from the same source to select different RPF paths.
- The basic S-G-hash algorithm ignores bidirectional PIM groups.
- Both algorithms are deterministic and allow predictable RPF path selection.
- Both are subject to **polarization**, which can reduce effective path utilization in multi-stage ECMP topologies.
- Neither algorithm guarantees equal bandwidth utilization.
- Use `show ip rpf <source> <group>` to verify group-specific RPF selection.

## References

- [Cisco IOS XE: IP Multicast Load Splitting across Equal-Cost Paths](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/ip-multicast/ip-multicast/m_imc_load_splt_ecmp-0.html)
