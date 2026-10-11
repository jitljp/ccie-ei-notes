# 1.6.b Reverse Path Forwarding Check

**Reverse Path Forwarding (RPF)** is a fundamental mechanism used by multicast routers to prevent routing loops and ensure that multicast traffic is received from the correct upstream direction.

Unlike unicast routing, which forwards packets toward a specific destination, multicast routing distributes packets from a source to multiple receivers.

Without loop prevention, multicast packets could circulate between routers, creating duplicate traffic and potentially overwhelming the network.

RPF prevents this by checking whether a multicast packet arrived on the interface the router expects to receive traffic from.

## Basic RPF Operation

The RPF check uses routing information to determine the correct incoming interface for multicast traffic.

For a source-based multicast distribution tree, the router determines which interface it would use to send a **unicast packet back toward the multicast source**.

This interface is called the **RPF interface**.

When a multicast packet arrives:

1. The router determines the expected RPF interface.
2. If the packet arrives on the RPF interface, the **RPF check passes**.
3. If the packet arrives on another interface, the **RPF check fails**, and the packet is dropped.

For example:

```text
             Source
          10.1.1.10
               |
               R1
              /  \
             /    \
            R2    R3
             \    /
              \  /
               R4
               |
            Receiver
```

Suppose R4's best route toward `10.1.1.10` points through R2.

R4 therefore selects its interface toward R2 as the RPF interface.

```text
Multicast packet received from R2
→ Arrives on RPF interface
→ RPF check passes

Multicast packet received from R3
→ Arrives on non-RPF interface
→ RPF check fails
→ Packet dropped
```

Even though both R2 and R3 could potentially deliver traffic from the source, R4 accepts multicast packets only through the appropriate upstream path.

This prevents the router from forwarding duplicate packets received through other paths.

> **Note:** Passing the RPF check does not automatically mean a multicast packet will be forwarded. The router must also have the appropriate multicast forwarding state and outgoing interfaces.

## RPF Interface and RPF Neighbor

Two important concepts are used in RPF:

- **RPF Interface:** The local interface selected as the upstream path toward the relevant multicast tree root.
- **RPF Neighbor:** The upstream next-hop router selected along that path.

The RPF interface determines where multicast packets are expected to arrive.

The RPF neighbor is important for multicast routing protocols such as PIM, which use it to send upstream Join/Prune messages and build multicast distribution trees.

The RPF interface and neighbor are usually derived from routing information, although certain multicast-specific mechanisms can affect their selection.

## RPF Lookup

The process of determining the correct upstream interface and neighbor is called an **RPF lookup**.

RPF normally uses the existing unicast routing information, allowing multicast routing to take advantage of routes learned through protocols such as OSPF, EIGRP, BGP, and static routing.

However, the RPF lookup does not necessarily use the same path as normal unicast forwarding.

When multiple possible upstream paths exist, additional route-selection and tie-breaking rules determine the RPF path.

## Static Multicast Routes

**Static multicast routes (static mroutes)** allow administrators to influence RPF selection independently of normal unicast forwarding.

In IOS XE, IPv4 and IPv6 use different commands to configure static mroutes:

| Protocol | Command |
|---|---|
| IPv4 | `ip mroute` |
| IPv6 | `ipv6 route ... multicast` |

Static mroutes can specify a different upstream interface or next hop for the RPF lookup, allowing multicast traffic to follow a different path from unicast traffic.

For example:

```text
Unicast route toward source
→ Ethernet0/0

Static mroute toward source
→ Ethernet0/1

Selected RPF interface
→ Ethernet0/1
```

Static mroutes are particularly useful when:

- Unicast and multicast traffic must follow different paths.
- Asymmetric routing causes RPF failures.
- Multicast traffic must be received through a tunnel.
- The normal unicast routing information does not provide the desired RPF path.

### IPv6 Static Mroutes

Unlike IPv4, IPv6 static mroutes are configured using the normal `ipv6 route` command.

The optional `multicast` and `unicast` keywords control how the static route is used:

| Configuration | Used for Unicast | Used for Multicast RPF |
|---|---|---|
| `ipv6 route ...` | Yes | Yes |
| `ipv6 route ... unicast` | Yes | No |
| `ipv6 route ... multicast` | No | Yes |

For example, to configure a multicast-only static route:

```text
R1(config)# ipv6 route 2001:DB8:1::/64 Ethernet0/1 FE80::2 100 multicast
```

This route can influence RPF lookups toward `2001:DB8:1::/64` without affecting normal unicast forwarding.

Verify the RPF selection using:

```text
R1# show ipv6 rpf 2001:DB8:1::10
```

**A static mroute does not directly forward multicast packets.** It influences RPF selection without necessarily modifying normal unicast forwarding.

Static mroutes can affect RPF lookups toward multicast sources and Rendezvous Points (RPs), depending on the prefixes they match.

The configuration and selection rules for both IPv4 and IPv6 static mroutes are covered in [Static Multicast Routes](static-mroutes.md).
## RPF for Source Trees and Shared Trees

An important distinction is that the RPF lookup target depends on the type of multicast distribution tree.

| Multicast State | Distribution Tree | RPF Lookup Toward |
|---|---|---|
| `(S,G)` | Source tree | Multicast source (S) |
| `(*,G)` | RP-rooted shared tree | Rendezvous Point (RP) |

For **(S,G) state**, the RPF lookup is performed toward the multicast source.

For **(*,G) state** in PIM Sparse Mode, the RPF lookup is performed toward the RP rather than the original source.

Therefore, the same multicast stream may use different incoming interfaces depending on whether the router is forwarding it along the source tree or the shared tree.

These concepts are particularly important for understanding PIM Sparse Mode, which is covered in the next major section.

## RPF Failures

An **RPF failure** occurs when a multicast packet arrives on an interface that is not accepted as the correct upstream interface.

Common causes include:

- Incorrect or missing routing information
- Asymmetric unicast and multicast paths
- Incorrect static multicast routes
- Routing changes that alter the selected RPF interface
- Incorrect RP reachability or RPF selection

RPF failures can prevent multicast traffic from reaching receivers even when normal unicast connectivity appears to work correctly.

IOS XE provides commands to examine RPF information and multicast forwarding state:

```text
R1# show ip rpf 10.1.1.10
R1# show ip mroute 239.1.1.1
```

These commands are essential for troubleshooting multicast forwarding problems.

## Section Overview

The following sections examine RPF operation in greater detail:

- **[RPF Interface and Neighbor](rpf-interface-neighbor.md):** How routers determine the expected incoming interface and upstream neighbor.
- **[RPF Lookup and Tie-Breaking](rpf-lookup.md):** How routing information and route-selection rules determine the RPF path.
- **[Static Multicast Routes](static-mroutes.md):** Configuring static mroutes to influence RPF selection independently of unicast forwarding.
- **[RPF for (S,G) State](source-tree-rpf.md):** Source-based RPF checks and their role in source-tree forwarding.
- **[RPF for (*,G) State](shared-tree-rpf.md):** RP-based RPF checks and their role in shared-tree forwarding.
- **[RPF Failures](rpf-failures.md):** Common causes of RPF failures, their effects on multicast forwarding, and IOS XE troubleshooting techniques.

Understanding RPF is essential for understanding how PIM builds loop-free multicast distribution trees and forwards traffic from sources toward receivers.