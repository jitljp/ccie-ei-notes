# IPv4 Multicast Addressing

IPv4 multicast uses the address range:

```text
224.0.0.0/4
```

This covers `224.0.0.0` through `239.255.255.255`.

A multicast address identifies a **multicast group** and is used as the destination address of a multicast packet.
The source address of a multicast packet is a unicast address.

## 224.0.0.0/24 — Local Network Control

Used for protocol control traffic on the **local link**.

Typically sent with TTL 1, but routers do **not** forward traffic destined for this range to another network segment, regardless of the packet's TTL.

Important addresses include:

```text
224.0.0.1   All Systems on this Subnet
224.0.0.2   All Routers on this Subnet
224.0.0.13  All PIM Routers
224.0.0.22  IGMPv3 Membership Reports
```

Other routing protocols also use addresses in this range, such as OSPF (`224.0.0.5` and `224.0.0.6`) and EIGRP (`224.0.0.10`).

## 224.0.1.0/24 — Internetwork Control

Used for protocol control traffic that **may be forwarded beyond the local link**.

Two important addresses for Cisco multicast are:

```text
224.0.1.39  Cisco-RP-Announce
224.0.1.40  Cisco-RP-Discovery
```

These are used by **Auto-RP**.

Unlike `224.0.0.0/24`, this range is not inherently limited to the local network segment.

## 232.0.0.0/8 — Source-Specific Multicast

Reserved for **Source-Specific Multicast (SSM)**.

With SSM, the receiver specifies both the source and the multicast group:

```text
(S,G)
```

For example:

```text
(10.1.1.10, 232.1.1.1)
```

Because the source is explicitly specified, SSM does not require an RP for source discovery.

`232.0.0.0/24` is reserved. The dynamically allocated SSM group space is:

```text
232.0.1.0 - 232.255.255.255
```

## 233.0.0.0/8 — GLOP

`233/8` is associated with **GLOP addressing**, which provides globally scoped multicast address space based on a 16-bit Autonomous System Number.

If a 16-bit AS number is represented by two octets `X.Y`:

```text
AS number → X.Y
GLOP block → 233.X.Y.0/24
```

More precisely, the current GLOP range is:

```text
233.0.0.0 - 233.251.255.255
```

`233.252.0.0/14` is now **AD-HOC Block III**, not GLOP.

GLOP is primarily of historical interest today, but the addressing method is worth recognizing.

Thanks to SSM, the need for unique group addresses has been reduced.

Instead, a forwarding state is identified by an (S,G) pair, not just the group address.

## 239.0.0.0/8 — Administratively Scoped Multicast

Reserved for **administratively scoped multicast**.

These addresses are intended for multicast traffic within a defined administrative domain and are not globally unique. The same multicast group can therefore be reused in separate domains.

For example:

```text
239.1.1.1
```

could be used by an organization's internal multicast application.

Unlike `224.0.0.0/24`, `239/8` traffic is **not automatically restricted to one link**. Administrators define multicast boundaries to control where the traffic can travel.

On Cisco IOS XE, this can be done with:

```text
ip multicast boundary
```

RFC 2365 also defines:

```text
239.192.0.0/14   Organization Local Scope
239.255.0.0/16   Local Scope
```