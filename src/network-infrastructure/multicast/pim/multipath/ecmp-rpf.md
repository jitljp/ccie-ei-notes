
# ECMP and Multicast Load Splitting

**Equal-Cost Multipath (ECMP)** allows a router to use multiple paths with the same routing cost toward a destination.

For unicast traffic, ECMP allows traffic to be distributed across multiple next hops. However, **IPv4 multicast does not perform ECMP load splitting by default**.

Instead, a multicast router selects a single **Reverse Path Forwarding (RPF) interface and neighbor** for each multicast forwarding state.

Cisco IOS XE supports **Multicast Multipath**, which allows different multicast forwarding states to select different equal-cost upstream paths.

## ECMP and RPF Selection

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

Assume OSPF provides R5 with three equal-cost routes toward the source network, `10.1.1.0/24`.

The next hops are:

| Next Hop | R5 Interface |
|---|---|
| R2: `10.2.5.2` | Ethernet0/0 |
| R3: `10.3.5.3` | Ethernet0/1 |
| R4: `10.4.5.4` | Ethernet0/2 |

Verify the routing information:

```text
R5# show ip route 10.1.1.10
```

An abbreviated example:

```text
Routing entry for 10.1.1.0/24
  Known via "ospf 1", distance 110, metric 20
  Routing Descriptor Blocks:
    10.2.5.2, via Ethernet0/0
    10.3.5.3, via Ethernet0/1
    10.4.5.4, via Ethernet0/2
```

All three next hops have the same OSPF metric.

However, this does not mean that multicast traffic from `10.1.1.10` will arrive through all three interfaces.

**An RPF lookup selects one upstream path for a particular multicast forwarding state.**

### RPF Route Selection vs. RPF Path Selection

The RPF lookup involves two distinct processes:

1. **RPF Route Selection:** Determines which route is used to reach the multicast source or RP.
2. **RPF Path Selection:** If the selected route has multiple equal-cost paths, determines which upstream interface and neighbor to use.

Multicast Multipath affects the **second process**, RPF path selection.

It does not change the route-selection rules used to choose between the unicast routing table, static mroutes, MBGP, and other RPF information sources.

## Default ECMP RPF Selection

By default, IOS XE uses the **highest PIM neighbor** method to select an RPF path when multiple equal-cost next hops exist.

The selection rules are:

1. Among the equal-cost next hops that are PIM neighbors, select the neighbor with the **highest IP address**.
2. If none of the equal-cost next hops are PIM neighbors, select the next hop with the **highest IP address**.

A next hop is considered a PIM neighbor if it appears in the output of `show ip pim neighbor`.

### Example 1: All Next Hops Are PIM Neighbors

Suppose R5 has established PIM adjacencies with R2, R3, and R4.

| Next Hop | PIM Neighbor | Selected |
|---|---|---|
| 10.2.5.2 | Yes | No |
| 10.3.5.3 | Yes | No |
| 10.4.5.4 | Yes | **Yes** |

R4 has the highest IP address, so R5 selects R4 as its RPF neighbor.

```text
R5# show ip rpf 10.1.1.10

RPF information for ? (10.1.1.10)
  RPF interface: Ethernet0/2
  RPF neighbor: ? (10.4.5.4)
  RPF route/mask: 10.1.1.0/255.255.255.0
  RPF type: unicast (ospf 1)
```

All sources in `10.1.1.0/24` that resolve to this same ECMP route use R4 as their RPF neighbor, regardless of their multicast group addresses.

```text
(10.1.1.10, 232.1.1.1) → R4
(10.1.1.11, 232.1.1.1) → R4
(10.1.1.12, 232.1.1.2) → R4
```

The paths through R2 and R3 remain available for routing and failover, but they are not used for multicast load splitting.

### Example 2: Only R2 and R3 Are PIM Neighbors

Now suppose R4 remains a valid OSPF next hop but is no longer a PIM neighbor.

| Next Hop | PIM Neighbor | Selected |
|---|---|---|
| 10.2.5.2 | Yes | No |
| 10.3.5.3 | Yes | **Yes** |
| 10.4.5.4 | No | No |

R5 selects R3, even though R4 has the highest IP address.

This is because **PIM neighbors are preferred over non-PIM next hops**.

### Example 3: No PIM Neighbors

If none of the three equal-cost next hops is a PIM neighbor, R5 selects the highest IP address:

```text
RPF interface: Ethernet0/2
RPF neighbor:  10.4.5.4
```

However, selecting an RPF neighbor does not automatically establish a PIM adjacency. Without the necessary PIM neighbor relationship, multicast tree construction may fail.

## Enabling Multicast Load Splitting

IOS XE enables source-based multicast load splitting with the following global configuration command:

```text
R5(config)# ip multicast multipath
```

This enables the **S-hash algorithm**.

Instead of always selecting the highest-addressed PIM neighbor, R5 uses a hash of the RPF address to select one of the available equal-cost paths.

For `(S,G)` states, the RPF address is the multicast source address.

For `(*,G)` states, the RPF address is the RP address.

For example, with multiple sources, the resulting RPF selections might look like this:

| Source | Selected RPF Neighbor |
|---|---|
| 10.1.1.10 | R2 |
| 10.1.1.11 | R4 |
| 10.1.1.12 | R3 |
| 10.1.1.13 | R2 |

*These are illustrative selections. The actual results depend on the hashing algorithm and available next hops.*

R5 sends its PIM `(S,G)` Joins toward the selected RPF neighbors.

The multicast streams can then travel through different upstream paths.

**Multicast Multipath changes RPF neighbor selection. It does not change the unicast routing table or cause multicast packets from an individual stream to be distributed across multiple paths.**

### Important Difference: PIM Neighbor Detection

With the default highest-PIM-neighbor method, IOS XE considers whether each equal-cost next hop is an established PIM neighbor.

However, when multicast multipath is enabled, **PIM neighbor status is not used to select the hashed RPF path**.

The hashing algorithm selects from the eligible equal-cost routing next hops.

Therefore, an equal-cost route through a router that is not a functioning PIM neighbor can still be selected by the hash.

This makes it important to verify both:

- The available equal-cost routes.
- The PIM adjacencies with the corresponding upstream routers.

## Multicast Load Splitting vs. Load Balancing

Multicast Multipath distributes forwarding states across multiple upstream paths.

However, it does **not guarantee equal traffic utilization** across those paths.

For example, suppose R5 selects the following RPF neighbors:

| Source | Traffic Rate | RPF Neighbor |
|---|---|---|
| 10.1.1.10 | 100 Mbps | R2 |
| 10.1.1.11 | 10 Mbps | R3 |
| 10.1.1.12 | 10 Mbps | R4 |

Each path carries one multicast stream, but the traffic rates are very different.

```text
R2 → R5: 100 Mbps
R3 → R5:  10 Mbps
R4 → R5:  10 Mbps
```

Multicast Multipath does not move traffic from R2 to another path merely because R2 is carrying more traffic.

The algorithms distribute multicast states using address-based hashing, rather than measuring link utilization.

Consequently:

- More multicast streams generally provide more opportunities for load distribution.
- A small number of streams may produce an uneven distribution.
- Different streams may have significantly different traffic rates.
- Each `(S,G)` state still uses one selected RPF path.

With source-based hashing, all groups from the same source use the same RPF path when their RPF routing information and ECMP candidates are identical.

The **basic S-G-hash** and **next-hop-based S-G-hash** algorithms allow finer-grained distribution based on both source and group addresses.

These algorithms are covered in the following pages.

## Multicast Multipath and PIM Operation

Multicast Multipath changes the RPF interface and neighbor selected by the downstream router.

It does not change the fundamental operation of PIM.

For example, suppose R5 selects R3 for:

```text
(S,G) = (10.1.1.10, 232.1.1.1)
```

The forwarding process is:

1. R5 selects R3 as its RPF neighbor.
2. R5 sends a PIM `(S,G)` Join to R3.
3. R3 forwards the Join upstream toward R1.
4. Multicast traffic flows from R1 through R3 to R5.
5. R5 accepts packets arriving through its selected RPF interface, Ethernet0/1.
6. R5 forwards the multicast traffic to the interested receiver on its LAN.

If the same multicast stream arrives through R2 or R4 instead, the packets fail R5's normal RPF check.

Multicast Multipath therefore preserves the fundamental rule that **each multicast forwarding state has a selected upstream RPF path**.

### Where Should Multicast Multipath Be Enabled?

Multicast Multipath is configured on routers that have **multiple equal-cost upstream paths** toward multicast sources or RPs.

In this topology, R5 has three upstream paths and performs the relevant RPF selection.

Therefore, R5 is the key router on which to configure multicast multipath.

This differs from unicast ECMP, where load sharing is often viewed from the perspective of the router sending traffic through multiple outgoing interfaces.

For multicast, the relevant decision is made by the **downstream router selecting its incoming interface**.

> **Important:** Cisco recommends configuring the multicast multipath algorithm consistently across routers in a redundant topology. Inconsistent RPF selection can cause forwarding problems, particularly when multiple downstream routers share an upstream LAN.

## ECMP Requirements and Limitations

Several requirements must be satisfied for multicast multipath to operate correctly.

### Equal-Cost Routes Must Exist

Multicast Multipath does not create additional routes.

The routing system must already provide multiple equal-cost paths toward the source or RP.

For example, if R5 learns only one OSPF path toward `10.1.1.0/24`, configuring `ip multicast multipath` will not create additional RPF paths.

### Unique Next-Hop Addresses

The multicast multipath mechanism does not support an ECMP configuration in which the same PIM neighbor IP address is reachable through multiple equal-cost paths.

This can occur with unnumbered interfaces.

Each equal-cost next hop should have a distinct IP address.

### BGP Multipath

BGP normally installs only its selected best path in the routing table.

To use multiple BGP paths for multicast ECMP, BGP must be configured to install eligible additional paths, typically using the `maximum-paths` command.

Alternatively, a BGP next hop that resolves recursively through multiple equal-cost IGP paths can provide multiple RPF candidates.

### Static Mroutes

Static multicast routes configured with `ip mroute` influence RPF route selection, but IOS XE does not support using multiple independent static mroutes for the same prefix to construct an ECMP set.

If static routing is required, equal-cost **unicast static routes** can instead supply the ECMP paths.

## Verification

### Verify the Equal-Cost Routes

First, verify that R5 has multiple equal-cost paths toward the source:

```text
R5# show ip route 10.1.1.10
```

Confirm that R2, R3, and R4 are all installed as next hops.

### Verify PIM Neighbors

```text
R5# show ip pim neighbor
```

Confirm that R2, R3, and R4 are established PIM neighbors.

### Verify RPF Selection

Use `show ip rpf` to examine the selected upstream path.

For example:

```text
R5# show ip rpf 10.1.1.10
```

When testing group-dependent hashing, specify both the source and group:

```text
R5# show ip rpf 10.1.1.10 232.1.1.1
R5# show ip rpf 10.1.1.10 232.1.1.2
```

This is particularly important when testing the S-G-hash algorithms, because different groups from the same source may select different RPF paths.

To examine additional sources:

```text
R5# show ip rpf 10.1.1.11 232.1.1.1
R5# show ip rpf 10.1.1.12 232.1.1.1
R5# show ip rpf 10.1.1.13 232.1.1.1
```

Compare the selected RPF interfaces and neighbors.

### Verify Installed Multicast State

```text
R5# show ip mroute
```

The multicast routing table identifies the incoming interface and RPF neighbor for each installed `(S,G)` state.

For example, an entry for `(10.1.1.10, 232.1.1.1)` might contain:

```text
(10.1.1.10, 232.1.1.1)
  Incoming interface: Ethernet0/1, RPF nbr 10.3.5.3
  Outgoing interface list:
    Ethernet0/3, Forward/Sparse
```

This indicates that R5 receives the stream through R3 and forwards it toward the receiver LAN.

### Verify the Configuration

```text
R5# show running-config | include ip multicast multipath
```

This identifies the configured multicast multipath algorithm.

## Key Points

- ECMP provides multiple equal-cost paths toward a multicast source or RP.
- IPv4 multicast ECMP load splitting is **disabled by default**.
- Without multicast multipath, IOS XE selects the **highest-IP-address PIM neighbor** among equal-cost next hops.
- If no eligible next hop is a PIM neighbor, the highest-IP-address next hop is selected.
- `ip multicast multipath` enables source-based multicast load splitting using the S-hash algorithm.
- When multicast multipath is enabled, **PIM neighbor status is not considered by the hashing algorithm**.
- Multicast Multipath changes **RPF path selection**, not RPF route selection.
- Multicast traffic is distributed **per forwarding state**, not per packet.
- Load splitting does not guarantee equal bandwidth utilization.
- Multicast multipath should be configured consistently across routers in redundant topologies.
- Use `show ip route`, `show ip rpf`, `show ip pim neighbor`, and `show ip mroute` to verify operation.

## References

- [Cisco IOS XE: IP Multicast Load Splitting across Equal-Cost Paths](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/ip-multicast/ip-multicast/m_imc_load_splt_ecmp-0.html)
