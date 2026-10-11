# Static Multicast Routes

**Static multicast routes (static mroutes)** allow administrators to control the Reverse Path Forwarding (RPF) path independently of normal unicast routing.

Normally, a multicast router uses unicast routing information to determine the upstream interface and neighbor toward a multicast source or Rendezvous Point (RP).

However, the unicast routing path is not always the desired multicast path.

Static mroutes solve this problem by specifying an alternative RPF path without modifying the normal unicast routing table.

In Cisco IOS XE, static mroutes are configured using:

| Protocol | Command |
|---|---|
| IPv4 | `ip mroute` |
| IPv6 | `ipv6 route ... multicast` |

**A static mroute does not directly forward multicast packets.** It supplies routing information used for RPF lookups.

Static mroutes also have the following characteristics:

- They are local to the router on which they are configured.
- They are not advertised or redistributed by routing protocols.
- They do not automatically create `(S,G)` or `(*,G)` multicast forwarding state.
- They can influence both source-based and RP-based RPF lookups.

## IPv4 Static Mroute Configuration

The basic syntax is:

```text
R1(config)# ip mroute <source-prefix> <mask> {<rpf-next-hop> | <interface>} [distance]
```

The arguments are:

| Argument | Description |
|---|---|
| `source-prefix` | Source or RP address/prefix used for RPF matching |
| `mask` | Subnet mask of the RPF lookup prefix |
| `rpf-next-hop` | IPv4 address of the desired upstream next hop |
| `interface` | Interface to use as the RPF interface |
| `distance` | Optional administrative distance (default 0) |

For example:

```text
R3(config)# ip mroute 10.10.10.0 255.255.255.0 10.0.23.2
```

This instructs R3 to use `10.0.23.2` as the RPF next hop for multicast sources in `10.10.10.0/24`, provided that the static mroute is selected during RPF route selection.

Alternatively, specify an interface:

```text
R3(config)# ip mroute 10.10.10.0 255.255.255.0 Tunnel0
```

This configures Tunnel0 as the RPF interface for the specified source prefix.

The interface-only form is especially useful on point-to-point interfaces. On multiaccess Ethernet networks, specifying the next-hop IP address is preferable because it explicitly identifies the upstream router.

### Recursive RPF Next-Hop Resolution

When a static mroute specifies a next-hop IP address, IOS XE uses the **unicast routing table to resolve that next hop**.

For example:

```text
R3(config)# ip mroute 10.10.10.0 255.255.255.0 10.0.23.2
```

Suppose the unicast routing table contains:

```text
C 10.0.23.0/24 is directly connected, Ethernet0/1
```

R3 resolves the next hop to:

```text
RPF Next Hop:   10.0.23.2
RPF Interface:  Ethernet0/1
```

Notice that the lookup for `10.0.23.2` is separate from the lookup for the original source `10.10.10.10`.

The static mroute controls the RPF path toward the source, while normal unicast routing information is used to resolve the specified RPF next hop.

## Example 1: Fixing an RPF Failure Caused by Asymmetric Routing

One of the most common uses of static mroutes is to correct RPF failures when unicast and multicast traffic follow different paths.

Consider this topology:

```text
                 Source
              10.10.10.10
                   |
              Multicast Core
                /      \
               /        \
              R1        R2
              |          |
         E0/0 |          | E0/1
              \          /
               \        /
                   R3
                    |
                 Receiver
                239.1.1.1
```

Suppose R3's normal unicast route toward `10.10.10.10` points through R1:

```text
R3# show ip route 10.10.10.10

Routing entry for 10.10.10.0/24
  Known via "ospf 1", distance 110, metric 20
  Routing Descriptor Blocks:
  * 10.0.13.1, via Ethernet0/0
```

The default RPF path is:

```text
RPF Interface: Ethernet0/0
RPF Neighbor:  10.0.13.1 (R1)
```

However, multicast traffic is intended to arrive through R2 on Ethernet0/1.

If a packet from `10.10.10.10` arrives on Ethernet0/1 while R3's RPF interface is Ethernet0/0, the packet fails the RPF check.

### Solution: Configure a Static Mroute

Configure R3 to use R2 as the RPF next hop:

```text
R3(config)# ip mroute 10.10.10.0 255.255.255.0 10.0.23.2
```

The static mroute changes the selected RPF information to:

```text
RPF Interface: Ethernet0/1
RPF Neighbor:  10.0.23.2 (R2)
```

Verify:

```text
R3# show ip rpf 10.10.10.10

RPF information for ? (10.10.10.10)
  RPF interface: Ethernet0/1
  RPF neighbor: ? (10.0.23.2)
  RPF route/mask: 10.10.10.0/255.255.255.0
  RPF type: static mroute
  RPF recursion count: 0
  Doing distance-preferred lookups across tables
```

Now R3 can build its upstream multicast path toward R2, provided the necessary PIM connectivity and forwarding state exist.

Importantly, the **unicast routing table is unchanged**:

```text
Unicast traffic toward 10.10.10.10
→ Ethernet0/0 (R1)

Multicast RPF toward 10.10.10.10
→ Ethernet0/1 (R2)
```

This allows multicast and unicast traffic to use different paths through the network.

## Example 2: Multicast over a GRE Tunnel

Static mroutes are also useful when multicast traffic must traverse a GRE tunnel while unicast traffic follows the normal physical network.

Consider:

```text
         Multicast Source
            10.10.10.10
                 |
                 R1
                 |
                 R2 ==================|
                 |                    |
                 |                    |
                 |            Tunnel0 |
                 |                    |
             IP Network               |
                 |                    |
                 R3 ==================|
                 |
              Receiver
             239.1.1.1
```

R2 and R3 have established a GRE tunnel.

The tunnel carries multicast traffic, while ordinary unicast traffic toward the source network uses the underlying IP network.

Suppose R3 has the following unicast route:

```text
10.10.10.0/24
→ Ethernet0/0
```

However, multicast packets from `10.10.10.10` arrive through Tunnel0.

Without an appropriate RPF route, these packets fail the RPF check because Tunnel0 is not the expected incoming interface.

### Solution

Configure a static mroute on R3:

```text
R3(config)# ip mroute 10.10.10.0 255.255.255.0 Tunnel0
```

The resulting paths are:

```text
Unicast route toward source
→ Ethernet0/0

Multicast RPF toward source
→ Tunnel0
```

PIM must also be configured appropriately on the tunnel endpoints so that multicast distribution-tree state can be established.

For example:

```text
R2(config)# interface Tunnel0
R2(config-if)# ip pim sparse-mode

R3(config)# interface Tunnel0
R3(config-if)# ip pim sparse-mode
```

The GRE tunnel's source and destination addresses must remain reachable through the underlying IP network.

> **Important:** The static mroute does not create the GRE tunnel or enable multicast forwarding through it. It only selects Tunnel0 as the RPF interface.

This approach is particularly useful when multicast traffic must cross a network that does not support native multicast routing.

## Example 3: Changing the RPF Path Toward an RP

Static mroutes can also influence the RPF path toward a **Rendezvous Point (RP)**.

In PIM Sparse Mode, routers use an RPF lookup toward the RP when building the shared tree, represented by `(*,G)` state.

Suppose:

```text
RP Address:       1.1.1.1
Multicast Group:  239.1.1.1
```

R3's unicast route toward the RP points through R1:

```text
RPF Lookup Toward RP: 1.1.1.1

Unicast Route:
→ Ethernet0/0 (R1)
```

However, the desired shared-tree path is through R2.

Configure:

```text
R3(config)# ip mroute 1.1.1.1 255.255.255.255 10.0.23.2
```

Now the RPF lookup toward the RP selects R2:

```text
RPF Lookup: 1.1.1.1

RPF Interface: Ethernet0/1
RPF Neighbor:  10.0.23.2
```

This influences the upstream direction used to build the RP-rooted shared tree.

### Source RPF vs. RP RPF

An important distinction is that the static mroute matches the **address used for the RPF lookup**.

For example:

```text
ip mroute 1.1.1.1 255.255.255.255 10.0.23.2
```

affects RPF lookups toward `1.1.1.1`.

If `1.1.1.1` is the RP address, this can affect `(*,G)` state.

However, `(S,G)` state uses an RPF lookup toward the source itself.

For example:

```text
Source: 10.10.10.10
RP:     1.1.1.1
```

The RPF lookups are:

```text
(*,239.1.1.1)
→ RPF lookup toward 1.1.1.1

(10.10.10.10,239.1.1.1)
→ RPF lookup toward 10.10.10.10
```

Consequently, changing the RPF route toward the RP does not necessarily change the RPF route toward the multicast source.

## Static Mroutes Match Sources, Not Multicast Groups

The source prefix configured with `ip mroute` is matched against the **RPF lookup address**, not the multicast destination group address.

For example:

```text
R3(config)# ip mroute 10.10.10.10 255.255.255.255 10.0.23.2
```

can affect RPF lookups for multiple groups sourced by `10.10.10.10`:

```text
(10.10.10.10, 239.1.1.1)
(10.10.10.10, 239.1.1.2)
(10.10.10.10, 239.1.1.3)
```

All three use the same source address for their `(S,G)` RPF lookup.

The command does not restrict the configuration to one multicast group.

Similarly, a static mroute matching an RP address can influence the RPF path for multiple groups mapped to that RP.

## Static Mroute Prefix Matching

When multiple IPv4 static mroutes match the same RPF lookup address, IOS XE selects the **longest matching prefix within the static mroute table**.

For example:

```text
R3(config)# ip mroute 10.0.0.0 255.0.0.0 10.0.13.1
R3(config)# ip mroute 10.10.10.0 255.255.255.0 10.0.23.2
```

The resulting static mroute matches are:

| RPF Lookup Address | Selected Static Mroute | Next Hop |
|---|---|---|
| 10.10.10.10 | 10.10.10.0/24 | 10.0.23.2 |
| 10.10.20.10 | 10.0.0.0/8 | 10.0.13.1 |

This lookup is independent of the order in which the routes were configured.

Remember that the selected static mroute must still compete against the other RPF information sources according to the RPF route-selection rules.

## Static Mroute Administrative Distance

The default administrative distance of an IPv4 static mroute is **0**.

For example:

```text
R3(config)# ip mroute 10.10.10.0 255.255.255.0 10.0.23.2
```

is equivalent to:

```text
R3(config)# ip mroute 10.10.10.0 255.255.255.0 10.0.23.2 0
```

You can configure a different administrative distance:

```text
R3(config)# ip mroute 10.10.10.0 255.255.255.0 10.0.23.2 150
```

This is useful when the static mroute should serve as a **backup RPF path** rather than the preferred path.

### Example: Floating Static Mroute

Suppose R3 has:

```text
Unicast OSPF route:
10.10.10.0/24
AD 110
Via Ethernet0/0

Static mroute:
10.10.10.0/24
AD 150
Via Ethernet0/1
```

The unicast OSPF route is preferred for RPF:

```text
RPF Interface: Ethernet0/0
```

If the OSPF route disappears and no other RPF information source with a better AD remains, the static mroute can become the selected RPF route:

```text
RPF Interface: Ethernet0/1
```

Note that an alternative PIM path must exist for multicast traffic to continue flowing.

### Administrative Distance Tie-Breaking

When the selected static mroute and a route from another RPF information source have equal ADs, the normal IPv4 preference is:

```text
Static mroute > DVMRP > MBGP > Unicast
```

However, Cisco documents two exceptions: directly connected unicast routes and default unicast routes are preferred over static mroutes when their ADs are equal.

The complete RPF selection rules, including `ip multicast longest-match`, are covered in [RPF Lookup and Tie-Breaking](rpf-lookup.md).

## Default Static Mroute

An IPv4 static mroute can match all RPF lookup addresses using a default prefix:

```text
R3(config)# ip mroute 0.0.0.0 0.0.0.0 10.0.23.2
```

This provides a matching static mroute for any IPv4 source or RP address.

However, it can unintentionally influence RPF lookups for many different multicast trees.

For example:

```text
Source 10.10.10.10 → RPF via 10.0.23.2
Source 172.16.1.10 → RPF via 10.0.23.2
RP 1.1.1.1         → RPF via 10.0.23.2
```

The actual selected RPF route still depends on the other available RPF information sources and their administrative distances.

**More specific static mroutes are generally preferable** when only particular sources or RPs require an alternative RPF path.

## IPv6 Static Multicast Routes

IPv6 also supports static multicast routing, but the configuration differs from IPv4.

Instead of using a separate `ipv6 mroute` command, IOS XE uses the standard `ipv6 route` command with options controlling whether the route is used for unicast forwarding, multicast RPF selection, or both.

### IPv6 Route Types

| Configuration | Unicast Forwarding | Multicast RPF |
|---|---|---|
| `ipv6 route ...` | Yes | Yes |
| `ipv6 route ... unicast` | Yes | No |
| `ipv6 route ... multicast` | No | Yes |

For example, suppose R3 should use:

```text
Unicast path toward source:
→ Ethernet0/0, FE80::1

Multicast RPF path toward source:
→ Ethernet0/1, FE80::2
```

Configure a unicast-only route:

```text
R3(config)# ipv6 route 2001:DB8:10::/64 Ethernet0/0 FE80::1 100 unicast
```

Configure a multicast-only route:

```text
R3(config)# ipv6 route 2001:DB8:10::/64 Ethernet0/1 FE80::2 100 multicast
```

These routes allow unicast forwarding and multicast RPF selection to use different upstream paths toward the same IPv6 prefix.

Because `FE80::1` and `FE80::2` are link-local addresses, the outgoing interfaces are explicitly specified.

### Separate Unicast and Multicast Administrative Distances

By default, the standard **`ipv6 route` command applies to both unicast routing and multicast RPF selection**, even without the `multicast` keyword.

The `unicast` and `multicast` keywords can restrict a static route to one purpose:

| Configuration | Unicast Routing | Multicast RPF |
|---|---|---|
| `ipv6 route ...` | Yes | Yes |
| `ipv6 route ... unicast` | Yes | No |
| `ipv6 route ... multicast` | No | Yes |

For example, a normal IPv6 static route:

```text
R3(config)# ipv6 route 2001:DB8:10::/64 2001:DB8:12::2 100
```

is eligible for **both unicast forwarding and multicast RPF selection**, using an AD of `100` for both.

However, IOS XE also allows separate administrative distances for unicast and multicast:

```text
R3(config)# ipv6 route 2001:DB8:10::/64 2001:DB8:12::2 100 200
```

In this example:

```text
Unicast administrative distance:       100
Multicast RPF administrative distance: 200
```

The first AD applies to the unicast route, while the second applies to multicast RPF selection.

This allows the same IPv6 static route to have different preferences for unicast and multicast purposes.

**If the second AD is omitted**, the multicast RPF AD defaults to the unicast AD, unless the `unicast` keyword explicitly excludes the route from multicast RPF selection.

### IPv6 Verification

Verify the IPv6 RPF route toward a source:

```text
R3# show ipv6 rpf 2001:DB8:10::10
```

For example, the expected RPF information might be:

```text
RPF Interface: Ethernet0/1
RPF Neighbor:  FE80::2
```

Also examine the IPv6 multicast forwarding state:

```text
R3# show ipv6 mroute
```

As with IPv4, a correctly configured static multicast route does not guarantee that multicast forwarding state exists.

## Static Mroutes and VRFs

Static mroutes can also be configured in a VRF.

For example:

```text
R3(config)# ip mroute vrf BLUE 10.10.10.0 255.255.255.0 10.0.23.2
```

This configures RPF information for the specified source prefix in VRF `BLUE`.

For multicast VPN extranet applications, IOS XE also supports the `fallback-lookup` option to resolve RPF lookups using another VRF or the global routing table.

For example:

```text
R3(config)# ip mroute vrf BLUE 10.10.10.0 255.255.255.0 fallback-lookup vrf RED
```

This allows an RPF lookup originating in the receiver VRF `BLUE` to continue in the source VRF `RED`.

This is an advanced multicast VPN feature, not a general replacement for unicast VRF route leaking.

## Useful Commands

| Command | Purpose |
|---|---|
| `ip mroute ...` | Configure an IPv4 static mroute |
| `show ip rpf <address>` | Verify selected IPv4 RPF information |
| `show ip route <address>` | Inspect IPv4 unicast routing information |
| `show ip mroute <group>` | Inspect multicast forwarding state |
| `show ip mroute <group> count` | Examine multicast forwarding and drop counters |
| `show ip pim neighbor` | Verify IPv4 PIM neighbors |
| `ipv6 route ... multicast` | Configure an IPv6 multicast-only static route |
| `show ipv6 rpf <address>` | Verify IPv6 RPF information |
| `show ipv6 mroute` | Inspect IPv6 multicast forwarding state |

## Key Points

- **Static mroutes influence RPF selection**, not normal unicast forwarding.
- IPv4 uses `ip mroute`; IPv6 uses `ipv6 route ... multicast`.
- Static mroutes match the source or RP address used for RPF lookup, **not the multicast group address**.
- They are useful for asymmetric routing, GRE tunnels, and choosing alternative paths toward sources or RPs.
- IPv4 static mroutes have a **default AD of 0**, which can be changed.
- When multiple static mroutes match, the most specific prefix within the static mroute table is selected as its candidate.
- Static mroutes are local to the router and cannot be redistributed.
- IPv6 static routes can be unicast-only, multicast-only, or usable for both, with independently configurable administrative distances.
- A static mroute does not create multicast forwarding state or replace PIM.
- Use `show ip rpf` and `show ipv6 rpf` to verify the actual RPF selection.