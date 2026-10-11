# RPF Lookup and Tie-Breaking

An **RPF lookup** determines which interface and upstream neighbor a multicast router should use to reach the root of a multicast distribution tree.

Although RPF normally relies on unicast routing information, IOS XE can also use multicast-specific routing information to select a different upstream path.

An RPF lookup involves two main processes:

1. **RPF Route Selection:** Select the best route toward the multicast source or Rendezvous Point (RP).

2. **RPF Path Selection:** If the selected route has multiple equal-cost paths, choose the RPF interface and neighbor.

Understanding the distinction is important because the selection rules are different.

## RPF Lookup Address

The address used for an RPF lookup depends on the type of multicast state.

| Multicast State | RPF Lookup Address |
|---|---|
| `(S,G)` | Multicast source (S) |
| `(*,G)` | Rendezvous Point (RP) |

For example:

```text
Multicast Source: 10.10.10.10
Multicast Group:  239.1.1.1
RP:               1.1.1.1
```

The RPF lookup for `(10.10.10.10, 239.1.1.1)` is performed toward `10.10.10.10`.

However, the RPF lookup for `(*, 239.1.1.1)` is performed toward `1.1.1.1`.

These lookups can produce different RPF interfaces and neighbors.

## RPF Information Sources

In IPv4, Cisco IOS XE can use several sources of routing information for RPF lookups:

| RPF Information Source | Description |
|---|---|
| Unicast Routing Table | Routes learned through OSPF, EIGRP, BGP, static routes, etc. |
| Static Mroute Table | Multicast-specific static routes configured using `ip mroute` |
| MBGP Routing Table | Routes learned through the BGP IPv4 multicast address family |
| DVMRP Routing Table | Routes learned through the legacy Distance Vector Multicast Routing Protocol |

The **unicast routing table** is the most commonly used source.

However, when other RPF information sources are available, IOS XE compares their candidate routes to determine which should be used.

> **Important:** The static mroute table and the multicast forwarding table are different. Static mroutes supply routing information for RPF lookups, whereas the multicast forwarding table contains entries such as `(S,G)` and `(*,G)`.

## RPF Route Selection

By default, IOS XE uses the following rules when selecting an IPv4 RPF route:

1. Within each RPF information source, determine the **most specific match**.

   - The most specific match from each source becomes a candidate.

2. Compare the candidate routes from the different RPF information sources and prefer the **lowest AD**.

3. If the ADs are equal, use the following preference order:

   1. Static mroutes

   2. DVMRP routes

   3. MBGP routes

   4. Unicast routes

There is an important distinction between this process and ordinary unicast routing.

**Within a single routing table, longest-prefix matching takes precedence over administrative distance. However, when comparing routes from different RPF information sources, administrative distance takes precedence over prefix length.**

> **Exceptions:** When ADs are equal, directly connected unicast routes and default unicast routes are preferred over static mroutes, rather than following the normal route-type preference of Static mroute > DVMRP > MBGP > Unicast.

### Example 1: Longest-Prefix Match Within the Unicast Routing Table

Suppose R3 has the following routes:

```text
Source: 10.10.10.10
Unicast Routing Table:
10.0.0.0/8
  Static, AD 1
  Via 10.0.23.2, Ethernet0/1
10.10.10.0/24
  OSPF, AD 110
  Via 10.0.13.1, Ethernet0/0
```

Both routes match `10.10.10.10`.

Which route is selected?

**The OSPF route (`10.10.10.0/24`) is selected**, even though it has a higher AD.

This is because both routes belong to the same unicast routing table, where longest-prefix matching takes precedence.

```text
RPF Lookup: 10.10.10.10
10.0.0.0/8       AD 1
10.10.10.0/24    AD 110  <-- SELECTED
RPF Interface: Ethernet0/0
RPF Neighbor:  10.0.13.1
```

The administrative distance is not used to compare these two different-length prefixes within the same unicast routing table.

### Example 2: Administrative Distance Across Different RPF Information Sources

Now suppose R3 has the following routing information:

```text
Source: 10.10.10.10
Unicast Routing Table:
10.10.10.0/24
  OSPF, AD 110
  Via 10.0.13.1, Ethernet0/0
MBGP Multicast Routing Table:
10.10.0.0/16
  eBGP, AD 20
  Via 10.0.23.2, Ethernet0/1
```

Both routes match `10.10.10.10`, but they belong to different RPF information sources.

IOS XE compares their administrative distances:

| RPF Source | Prefix | AD | Selected |
|---|---|---|---|
| Unicast (OSPF) | 10.10.10.0/24 | 110 | No |
| MBGP (eBGP) | 10.10.0.0/16 | 20 | Yes |

**The MBGP route is selected**, even though it has a shorter prefix.

```text
RPF Lookup: 10.10.10.10
Unicast: 10.10.10.0/24, AD 110
MBGP:    10.10.0.0/16,  AD 20  <-- SELECTED
RPF Interface: Ethernet0/1
RPF Neighbor:  10.0.23.2
```

This behavior is called **distance-preferred lookup across tables**.

It explains the following message commonly displayed by `show ip rpf`:

```text
Doing distance-preferred lookups across tables
```

> **Key distinction:** An RPF lookup does not necessarily select the longest matching prefix across all available RPF information sources.

### Example 3: Equal Administrative Distances

Suppose R3 has the following:

```text
Unicast Routing Table:
10.10.10.0/24
  OSPF, AD 110
  Via Ethernet0/0
Static Mroute Table:
10.10.0.0/16
  Static mroute, AD 110
  Via Ethernet0/1
```

The administrative distances are equal.

IOS XE prefers the **static mroute** over the unicast route.

```text
RPF Interface: Ethernet0/1
RPF Type:      static mroute
```

The static mroute wins despite having a shorter prefix, because the RPF information sources are different and static mroutes have higher priority when the ADs are equal.

If the static mroute's AD were increased to `150`, the unicast OSPF route with AD `110` would instead be preferred.

The configuration and additional selection rules for static mroutes are covered in [Static Multicast Routes](static-mroutes.md).

### `ip multicast longest-match`

By default, IPv4 RPF lookups are **distance-preferred across RPF information sources**. The `ip multicast longest-match` command changes the comparison to **longest-prefix-match preferred across sources**.

On IOS XE platforms that support the command, configure it globally:

```text
R3(config)# ip multicast longest-match
```

With this command configured, RPF route selection follows these rules:

1. Across all RPF information sources, **select the most specific matching prefix**.
2. If multiple routes have the same prefix length, prefer the **lowest AD**.
3. If ADs are equal, use the **route-type preference**: Static mroute > DVMRP > MBGP > Unicast (subject to the exceptions noted above).

**Example:** Revisit Example 2, in which R3 has the following routes toward `10.10.10.10`:

| RPF Information Source | Prefix | AD | Interface |
|---|---|---|---|
| Unicast (OSPF) | `10.10.10.0/24` | 110 | Ethernet0/0 |
| MBGP (eBGP) | `10.10.0.0/16` | 20 | Ethernet0/1 |

- **Default:** MBGP wins because its AD (20) is lower, despite the shorter `/16` prefix.
- **With `ip multicast longest-match`:** OSPF wins because its `/24` prefix is more specific, despite the higher AD (110).

The new RPF path is therefore:

```text
RPF Interface: Ethernet0/0
RPF Neighbor:  10.0.13.1
RPF Type:      unicast (ospf 1)
```

Verify the active selection mode with:

```text
R3# show ip rpf 10.10.10.10
...
  Doing longest-match lookups across tables
```

To restore the default behavior, where AD takes precedence across sources:

```text
R3(config)# no ip multicast longest-match
```

**Note**: Using this command in modern IOS XE gives the following warning:

```
*Oct 11 04:17:17.336: %PARSER-5-HIDDEN: Warning!!! ' ip multicast longest-match ' is a hidden command. Use of this command is not recommended/supported and will be removed in future.
```

So it might not be available in newer versions of IOS XE.

## RPF Path Selection and ECMP

Once the router selects the best RPF route, it must determine which next hop to use.

If the selected route has only one next hop, the selection is straightforward.

However, **Equal-Cost Multipath (ECMP)** routing can result in multiple eligible upstream paths.

Consider the following topology:

```text
             Source
          10.10.10.10
               |
        Upstream Network
           /       \
          /         \
         R1         R2
    10.0.13.1    10.0.23.2
          \         /
       E0/0\       /E0/1
            \     /
              R3
               |
            Receiver
```

Suppose R3 learns two equal-cost OSPF routes toward `10.10.10.10`:

```text
R3# show ip route 10.10.10.10
Routing entry for 10.10.10.0/24
  Known via "ospf 1", distance 110, metric 20
  Routing Descriptor Blocks:
    10.0.13.1, via Ethernet0/0
    10.0.23.2, via Ethernet0/1
```

Both next hops are equally preferred by OSPF.

For unicast traffic, Cisco Express Forwarding (CEF) can load-share traffic across these paths.

**IPv4 multicast behaves differently by default.**

Rather than distributing multicast traffic across all equal-cost RPF paths, IOS XE selects one upstream path for a given multicast tree.

### Default ECMP Tie-Breaking

By default, IOS XE uses the **highest PIM neighbor** method for IPv4 multicast.

When multiple equal-cost next hops exist:

1. Prefer the **PIM neighbor with the highest IP address** among the eligible next hops.

2. If none of the eligible next hops are PIM neighbors, select the next hop with the **highest IP address**.

An eligible next hop is considered a PIM neighbor when it appears in the output of `show ip pim neighbor`.

**Example A: Both Next Hops Are PIM Neighbors**

```text
R1: 10.0.13.1 (PIM neighbor)
R2: 10.0.23.2 (PIM neighbor)
```

R3 selects R2:

```text
RPF Interface: Ethernet0/1
RPF Neighbor:  10.0.23.2
```

R2 has the higher IP address.

**Example B: Only R1 Is a PIM Neighbor**

```text
R1: 10.0.13.1 (PIM neighbor)
R2: 10.0.23.2 (NOT a PIM neighbor)
```

R3 selects R1:

```text
RPF Interface: Ethernet0/0
RPF Neighbor:  10.0.13.1
```

Even though R2 has the higher IP address, R1 is preferred because it is the only eligible PIM neighbor.

**Example C: Neither Next Hop Is a PIM Neighbor**

```text
R1: 10.0.13.1 (NOT a PIM neighbor)
R2: 10.0.23.2 (NOT a PIM neighbor)
```

R3 selects R2:

```text
RPF Interface: Ethernet0/1
RPF Neighbor:  10.0.23.2
```

The next hop with the highest IP address is selected.

However, selecting an RPF next hop does not establish a PIM adjacency. Without an appropriate PIM neighbor, upstream Join/Prune processing may not operate correctly.

### Multicast Multipath

By default, IOS XE selects a single RPF path using the highest-PIM-neighbor method.

The `ip multicast multipath` command changes this behavior, allowing different multicast streams to use different equal-cost upstream paths.

IOS XE supports several multicast load-splitting algorithms, including source-based hashing, source-group hashing, and next-hop-based source-group hashing.

These mechanisms are covered in [Multicast Multipath](../pim/multipath/overview.md).

## RPF Lookup with Static Mroutes

Static mroutes can override the normal unicast-derived RPF path.

For example, suppose R3's unicast route toward `10.10.10.10` points through R1:

```text
Unicast route:
10.10.10.0/24 via 10.0.13.1
Outgoing interface: Ethernet0/0
```

An administrator wants multicast traffic to arrive through R2 instead.

Configure a static mroute:

```text
R3(config)# ip mroute 10.10.10.0 255.255.255.0 10.0.23.2
```

The default administrative distance of an IPv4 static mroute is `0`.

The resulting RPF lookup selects the static mroute:

```text
RPF Interface: Ethernet0/1
RPF Neighbor:  10.0.23.2
RPF Type:      static mroute
```

However, normal unicast forwarding toward `10.10.10.10` continues to use Ethernet0/0.

If the static mroute uses a recursively resolved next-hop address, IOS XE uses unicast routing information to resolve that next hop to an outgoing interface.

A static mroute changes RPF selection; it does not directly create multicast forwarding state.

Additional IPv4 and IPv6 static mroute options are covered in [Static Multicast Routes](static-mroutes.md).

## IPv6 RPF Lookup

IPv6 multicast uses the same fundamental RPF lookup concepts:

- Determine the RPF lookup address (source or RP).
- Select the appropriate routing information.
- Resolve the upstream interface and next hop.
- Use the resulting RPF information for multicast forwarding and PIM operation.

However, IPv6 has some important differences.

For example, IPv6 static routes can be configured for unicast only, multicast RPF only, or both:

```text
! Unicast only
R3(config)# ipv6 route 2001:DB8:10::/64 Ethernet0/0 FE80::1 100 unicast
! Multicast RPF only
R3(config)# ipv6 route 2001:DB8:10::/64 Ethernet0/1 FE80::2 100 multicast
```

In this example, the unicast route uses Ethernet0/0, while the multicast-specific route directs RPF toward Ethernet0/1.

Verify:

```text
R3# show ipv6 rpf 2001:DB8:10::10
```

IPv6 RPF neighbors are identified by their **link-local addresses**, so the associated outgoing interface is important for next-hop resolution.

## Verification

### `show ip rpf`

The primary command for checking IPv4 RPF selection is:

```text
R3# show ip rpf 10.10.10.10
```

For example:

```text
RPF information for ? (10.10.10.10)
  RPF interface: Ethernet0/1
  RPF neighbor: ? (10.0.23.2)
  RPF route/mask: 10.10.10.0/255.255.255.0
  RPF type: unicast (ospf 1)
  RPF recursion count: 0
  Doing distance-preferred lookups across tables
```

This output identifies:

- The selected RPF interface and neighbor
- The route used for the lookup
- The source of RPF routing information
- Whether recursive next-hop resolution occurred

To display the routing metric and preference:

```text
R3# show ip rpf 10.10.10.10 metric
```

This can display additional fields such as:

```text
Metric preference: 110
Metric: 20
```

The metric preference represents the route's AD, while the metric is the routing protocol's metric.

### Group-Specific RPF Verification

When using multicast multipath hashing, include the group address to examine the RPF selection for a particular multicast stream:

```text
R3# show ip rpf 10.10.10.10 239.1.1.1
R3# show ip rpf 10.10.10.10 239.1.1.2
```

These lookups can resolve to different RPF paths depending on the configured algorithm.

### Additional Commands

| Command | Purpose |
|---|---|
| `show ip route <source>` | Display unicast routing information |
| `show ip rpf <source>` | Display selected RPF interface and neighbor |
| `show ip rpf <source> metric` | Display RPF metric and preference |
| `show ip rpf <source> <group>` | Check group-specific RPF selection |
| `show ip pim neighbor` | Verify eligible PIM neighbors |
| `show ip mroute <group>` | View multicast state and incoming interface |
| `show running-config \| include ip multicast multipath` | Verify multicast multipath configuration |
| `show ipv6 rpf <address>` | Display IPv6 RPF information |
| `show ipv6 route <address>` | Inspect IPv6 unicast routing information |

Remember that `show ip route` shows the unicast routing table, but not necessarily all the routing information considered during RPF selection.

**Always verify the actual RPF interface and neighbor using `show ip rpf` or `show ipv6 rpf`.**

## Key Points

- RPF lookup involves **route selection** followed by **path selection** when ECMP routes are available.
- `(S,G)` RPF lookups are performed toward the source; `(*,G)` lookups are performed toward the RP.
- IPv4 RPF can use the unicast RIB, static mroutes, MBGP, and DVMRP routing information.
- Within each RPF information source, **longest-prefix match** determines the candidate route.
- By default, across different RPF information sources, **administrative distance** takes precedence. A shorter prefix can therefore win.
- `ip multicast longest-match` instead prioritizes the **longest matching prefix across sources**, then AD and route type.
- When ADs are equal, the preference order is **Static Mroute > DVMRP > MBGP > Unicast**.
- By default, IOS XE IPv4 multicast ECMP selects the **highest-IP-address PIM neighbor** among eligible next hops, falling back to the highest next-hop IP address if none are PIM neighbors.
- By default, IPv4 multicast traffic is not load-split across ECMP RPF paths.
- Static mroutes can influence RPF selection without changing normal unicast forwarding.
- Use `show ip rpf` and `show ipv6 rpf` to verify the actual RPF selection.
