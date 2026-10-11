# RPF Interface and Neighbor

The **Reverse Path Forwarding (RPF)** process determines the upstream direction from which a multicast router expects to receive multicast traffic.

Two important components of this process are:

- **RPF Interface:** The local interface on which the router expects to receive multicast traffic.
- **RPF Neighbor:** The upstream router selected as the next hop toward the root of the multicast distribution tree.

Although closely related, these serve different purposes.

The **RPF interface** is used to validate incoming multicast packets, while the **RPF neighbor** is used by multicast routing protocols such as PIM to build and maintain multicast distribution trees.

## RPF Interface

The **RPF interface** is the interface selected as the correct incoming interface for multicast traffic.

For a source-based distribution tree, the router normally determines this interface by performing a routing lookup toward the multicast source.

Consider the following topology:

```text
       Source
    10.10.10.10
         |
         | E0/0
        R1
         | E0/1
         |
         | E0/0
        R2
         | E0/1
         | 10.0.23.2
         |
         | 10.0.23.3
         | E0/0
        R3
         | E0/1
         |
      Receiver
     239.1.1.1
```

R3 has learned the following route through OSPF:

```text
R3# show ip route 10.10.10.10
Routing entry for 10.10.10.0/24
  Known via "ospf 1", distance 110, metric 20
  Routing Descriptor Blocks:
  * 10.0.23.2, via Ethernet0/0
```

The route toward the source points through Ethernet0/0.

Therefore, assuming no other multicast-specific routing information overrides this path:

```text
Source:        10.10.10.10
RPF Interface: Ethernet0/0
```

When R3 receives multicast traffic from `10.10.10.10`:

- Traffic arriving on Ethernet0/0 passes the RPF interface check.
- Traffic arriving on another interface fails the RPF interface check and is dropped.

The RPF check therefore helps ensure that multicast traffic is accepted only from the expected upstream direction.

> **Note:** The RPF interface is determined by the selected RPF route, which does not necessarily match the normal unicast forwarding path. Static mroutes and multicast-specific routing information can influence the selection.

## RPF Neighbor

The **RPF neighbor** is the upstream next-hop router selected along the RPF path.

In the previous example, R3's route toward `10.10.10.10` points to:

```text
Next hop: 10.0.23.2
Interface: Ethernet0/0
```

Therefore:

```text
RPF Interface: Ethernet0/0
RPF Neighbor:  10.0.23.2 (R2)
```

Notice that the **RPF neighbor is not necessarily the multicast source**.

R3's RPF neighbor is R2, even though the actual source is `10.10.10.10`, which is connected to R1.

Each router along the multicast distribution tree independently determines its own RPF interface and neighbor.

For example:

| Router | RPF Interface | RPF Neighbor |
|---|---|---|
| R3 | Ethernet0/0 (toward R2) | R2 |
| R2 | Ethernet0/0 (toward R1) | R1 |
| R1 | Ethernet0/0 (toward source) | None (source directly connected) |

The resulting upstream path is:

```text
R3 → R2 → R1 → Source
```

### RPF Neighbor and PIM

The RPF neighbor is particularly important for PIM Sparse Mode.

PIM uses upstream **Join/Prune messages** to establish and maintain multicast distribution trees.

For example, when R3 wants to receive multicast traffic from a particular source, it can send an upstream PIM Join toward its RPF neighbor, R2.

R2 can then send its own Join toward R1.

```text
Source
  |
  R1
  ^  PIM Join
  |
  R2
  ^  PIM Join
  |
  R3
  |
Receiver
```

The Join messages travel upstream, while multicast data travels downstream.

```text
PIM Join messages:
R3 → R2 → R1

Multicast data:
R1 → R2 → R3
```

The RPF neighbor therefore identifies the upstream router toward which PIM should build the multicast distribution tree.

> **Important:** The RPF neighbor is derived from multicast routing information, whereas a **PIM neighbor adjacency** is established through PIM Hello messages. An RPF lookup can identify a next hop even when a working PIM adjacency does not exist. In PIM Sparse Mode, this can prevent the router from successfully building the expected upstream distribution tree.

## Directly Connected Multicast Sources

When a multicast source is directly connected to a router, there is no upstream router between the source and the local router.

In this case, the RPF interface is normally the interface connected to the source, but **there is no upstream RPF neighbor**.

For example:

```text
Source
10.10.10.10
    |
    | Ethernet0/0
    R1
    |
    | Ethernet0/1
    R2
```

On R1, the `(S,G)` multicast routing entry may display:

```text
R1# show ip mroute 239.1.1.1

(10.10.10.10, 239.1.1.1), 00:05:00/00:02:59, flags: FT
  Incoming interface: Ethernet0/0, RPF nbr 0.0.0.0
  Outgoing interface list:
    Ethernet0/1, Forward/Sparse, 00:05:00/00:02:59
```

Notice:

```text
Incoming interface: Ethernet0/0
RPF nbr:           0.0.0.0
```

The value `0.0.0.0` indicates that no upstream RPF neighbor is required because the source is directly connected.

**The multicast source does not need to be a PIM router or establish a PIM neighbor adjacency.**

A similar absence of an upstream RPF neighbor occurs at the RP for a `(*,G)` shared tree, because the RP itself is the root of that tree.

## RPF Interface vs. RPF Neighbor on Multiaccess Networks

On a point-to-point link, the RPF interface generally leads to a single upstream router.

However, on a multiaccess network such as Ethernet, multiple PIM routers can share the same interface.

```text
   R1              R2
10.0.123.1      10.0.123.2
     \             /
      \           /
           SW1
            |
            |
            R3
        Ethernet0/0
```

R3 might select:

```text
RPF Interface: Ethernet0/0
RPF Neighbor:  10.0.123.2 (R2)
```

Even though both R1 and R2 are reachable through Ethernet0/0, R3 has selected R2 as its upstream RPF neighbor.

### RPF Neighbor Selection

The RPF neighbor is normally determined by the **next-hop address of the selected RPF route**, not simply by which PIM neighbors exist on the interface.

If multiple equal-cost next hops are available, IOS XE uses the following default IPv4 selection rules:

1. Prefer the **PIM neighbor** with the **highest IP address**.
2. If none of the eligible next hops are PIM neighbors, select the next hop with the **highest IP address**.

For example, suppose R3 has two equal-cost routes toward the multicast source:

```text
Via 10.0.123.1 (R1)
Via 10.0.123.2 (R2)
```

If both are PIM neighbors, R3 selects R2 because `10.0.123.2` is higher.

However, if only R1 is an established PIM neighbor, R3 selects R1 instead.

**These tie-breaking rules apply only when multiple equal-cost RPF paths exist.** If the selected route has only one next hop, another PIM neighbor with a higher IP address does not automatically become the RPF neighbor.

The `ip multicast multipath` command can change the default ECMP selection behavior. This is covered in [RPF Lookup and Tie-Breaking](rpf-lookup.md).

### RPF Neighbor vs. PIM Assert Winner

It is important to distinguish between:

- **RPF interface:** The interface on which multicast packets are expected to arrive.
- **RPF neighbor:** The selected upstream next-hop router.
- **PIM Assert winner:** The router elected to forward multicast traffic onto a shared LAN when duplicate forwarding is detected.

On a shared network, checking the incoming interface alone does not distinguish packets forwarded by different routers on that same interface.

**PIM Assert** resolves duplicate forwarding and can also influence the upstream RPF neighbor used for a particular multicast tree.

### PIM Assert and RPF Neighbor Selection

**PIM Assert can override the normal RPF neighbor selection** when multiple routers compete to forward the same multicast traffic onto a shared LAN.

For example, suppose R3's RPF lookup toward source `10.10.10.10` selects:

```text
RPF Interface: Ethernet0/0
RPF Neighbor:  10.0.123.2 (R2)
```

However, R1 wins a PIM Assert election for `(10.10.10.10, 239.1.1.1)` on that shared LAN.

R3 can then use R1 as its upstream neighbor for that `(S,G)` state:

```text
Before Assert:
RPF Interface: Ethernet0/0
RPF Neighbor:  R2

After Assert:
RPF Interface: Ethernet0/0
Upstream PIM Neighbor: R1 (Assert winner)
```

**The RPF interface and underlying routing information remain unchanged.** The Assert winner takes precedence over the normally selected next hop for the affected multicast state.

RFC 7761 defines this Assert-aware neighbor selection as **RPF' (RPF prime)**:

- `RPF'(S,G)`: Assert-aware upstream neighbor toward the source.
- `RPF'(*,G)`: Assert-aware upstream neighbor toward the RP.

PIM Join/Prune messages are directed toward the effective upstream neighbor, which may differ from the next hop selected by the multicast routing information base (MRIB).

The Assert override applies to the affected multicast state on the selected RPF interface, not globally to every multicast group or source.

The complete election process is covered in [PIM Assert](assert.md).

## RPF for (S,G) and (*,G) State

The RPF interface and neighbor depend on the root of the multicast distribution tree.

| State | RPF Lookup Toward | Upstream Direction |
|---|---|---|
| `(S,G)` | Source (S) | Toward the multicast source |
| `(*,G)` | Rendezvous Point (RP) | Toward the RP |

For `(S,G)` state, the router determines its upstream path toward the multicast source.

For `(*,G)` state in PIM Sparse Mode, the upstream path is determined toward the RP.

Consequently, the same router may have different RPF interfaces and neighbors for `(S,G)` and `(*,G)` state associated with the same multicast group.

These differences are covered in the following RPF sections.

## RPF Interface and Neighbor Changes

The selected RPF interface or neighbor can change when the network's routing information changes.

For example, if R3's preferred route toward the source changes from R2 to another router, its RPF selection may also change.

Other factors that can affect the selection include:

- Static multicast routes
- Equal-cost multipath routing (ECMP)
- Changes in IGP or BGP routing information
- Multicast-specific routing information
- PIM Assert results on multiaccess networks

A change in RPF selection can require PIM to update its upstream Join/Prune state.

The route-selection rules, including RPF tie-breaking, are covered in [RPF Lookup and Tie-Breaking](rpf-lookup.md).

## IPv6 RPF Interface and Neighbor

IPv6 multicast uses the same fundamental RPF concepts as IPv4.

The router determines the RPF interface and upstream neighbor toward the appropriate IPv6 address.

An important difference is that **IPv6 RPF neighbors are identified using link-local IPv6 addresses**.

For example:

```text
R3# show ipv6 rpf 2001:DB8:10::10
RPF information for 2001:DB8:10::10
  RPF interface:Ethernet0/0
  RPF neighbor:FE80::2
  RPF route/mask:2001:DB8:10::/64
  RPF type:Unicast
  RPF recursion count:0
  Metric preference:110
  Metric:20
```

In this example:

```text
RPF Interface: Ethernet0/0
RPF Neighbor:  FE80::2
```

The address `FE80::2` identifies the upstream next-hop router on the local link.

Because IPv6 link-local addresses have link-local scope, the RPF interface is also important for identifying the correct neighbor.

## Verification

### `show ip rpf`

Use `show ip rpf` to display the RPF information for a particular IPv4 address.

```text
R3# show ip rpf 10.10.10.10
RPF information for ? (10.10.10.10)
  RPF interface: Ethernet0/0
  RPF neighbor: ? (10.0.23.2)
  RPF route/mask: 10.10.10.0/255.255.255.0
  RPF type: unicast (ospf 1)
  RPF recursion count: 0
  Doing distance-preferred lookups across tables
```

The output identifies several important values:

| Field | Description |
|---|---|
| RPF interface | Expected incoming interface |
| RPF neighbor | Selected upstream next hop |
| RPF route/mask | Route used for the RPF lookup |
| RPF type | Source of routing information used for RPF |
| RPF recursion count | Number of recursive next-hop resolutions |

The `?` indicates that IOS XE could not resolve the address to a hostname.

The optional `metric` keyword can display additional routing metric information:

```text
R3# show ip rpf 10.10.10.10 metric
```

### `show ip mroute`

The multicast routing table also displays the incoming interface and RPF neighbor for each relevant multicast entry.

```text
R3# show ip mroute 239.1.1.1

(10.10.10.10, 239.1.1.1), 00:05:00/00:02:59, flags: T
  Incoming interface: Ethernet0/0, RPF nbr 10.0.23.2
  Outgoing interface list:
    Ethernet0/1, Forward/Sparse, 00:05:00/00:02:59
```

Notice:

```text
Incoming interface: Ethernet0/0
RPF nbr:           10.0.23.2
```

These values should be consistent with the expected RPF path for the multicast entry.

### Additional Verification Commands

The following commands are useful for checking RPF-related information:

| Command | Purpose |
|---|---|
| `show ip rpf <address>` | Display IPv4 RPF interface and neighbor |
| `show ip rpf <address> metric` | Include RPF metric information |
| `show ip route <address>` | Inspect the IPv4 unicast route |
| `show ip mroute <group>` | Inspect IPv4 multicast forwarding state |
| `show ip pim neighbor` | Display IPv4 PIM adjacencies |
| `show ip pim interface` | Display IPv4 PIM-enabled interfaces |
| `show ipv6 rpf <address>` | Display IPv6 RPF interface and neighbor |
| `show ipv6 route <address>` | Inspect the IPv6 unicast route |
| `show ipv6 mroute` | Inspect IPv6 multicast forwarding state |
| `show ipv6 pim neighbor` | Display IPv6 PIM adjacencies |

Remember that `show ip route` and `show ipv6 route` display normal unicast routing information. They do not necessarily reveal multicast-specific routes that influence RPF selection.

For this reason, `show ip rpf` and `show ipv6 rpf` are the primary commands for verifying the actual RPF selection.

## Key Points

- The **RPF interface** is the expected incoming interface for multicast traffic.
- The **RPF neighbor** is the selected upstream next-hop router toward the multicast tree root.
- The RPF neighbor is not necessarily the multicast source itself.
- PIM uses RPF neighbor information to determine where to send upstream Join/Prune messages.
- When the source is directly connected, the RPF neighbor may be displayed as `0.0.0.0`.
- On multiaccess networks, multiple routers can share the RPF interface. PIM Assert can influence which router forwards traffic.
- `(S,G)` and `(*,G)` state can have different RPF interfaces and neighbors.
- IPv6 uses the same RPF concepts, commonly identifying upstream neighbors by their link-local IPv6 addresses.
- Use `show ip rpf`, `show ipv6 rpf`, and multicast routing table commands to verify RPF selection.