# 1.6.c (v) Multicast Boundary, RP Announcement Filter

Multicast routing allows a source to distribute traffic to receivers across multiple networks. However, administrators may need to restrict where multicast traffic can travel or which routers can advertise themselves as Rendezvous Points (RPs).

Cisco IOS XE provides several mechanisms for controlling multicast traffic and signaling:

- **Multicast boundaries:** Restrict multicast forwarding and associated control-plane operations across specified interfaces.
- **IPv6 multicast group-range filtering:** Restricts which IPv6 multicast groups and source-group channels a router is permitted to process.
- **RP announcement filtering:** Controls which Candidate RPs and group ranges an Auto-RP Mapping Agent accepts.
- **BSR borders:** Prevent PIM Bootstrap Messages from crossing between PIM domains.

These mechanisms provide control over both the **multicast data plane** and the **multicast control plane**.

## Multicast Boundaries

A **multicast boundary** limits the propagation of multicast traffic across a network boundary.

For example, consider two multicast domains:

```text
        Multicast Domain A
           239.1.1.1
                |
               R1
                |
                | Ethernet0/0
                |
        === Multicast Boundary ===
                |
                | Ethernet0/0
               R2
                |
        Multicast Domain B
           239.1.1.1
```

Both domains use multicast group `239.1.1.1`, but traffic for that group must remain within its respective domain.

By configuring a multicast boundary between the domains, multicast traffic for `239.1.1.1` can be prevented from crossing the boundary.

This allows the same multicast group addresses to be reused in separate administrative domains without exchanging their multicast traffic.

### Administrative Scope

IPv4 reserves `239.0.0.0/8` for **administratively scoped multicast**, as defined in RFC 2365.

These addresses are intended for multicast distribution within administratively defined boundaries rather than unrestricted global distribution.

Cisco IOS XE supports interface-based multicast boundaries using:

```text
ip multicast boundary <acl>
```

The ACL identifies multicast groups that are permitted or denied across the boundary.

Extended ACLs can also be used with directional filtering to control individual `(S,G)` channels.

IPv6 incorporates multicast scope directly into the multicast address, allowing boundaries to be configured based on the scope field.

See [Multicast Boundaries and Administrative Scope](multicast-boundary.md).

## IPv6 Multicast Group-Range Filtering

IPv6 multicast group-range filtering provides a router-wide mechanism for restricting multicast processing.

Unlike an interface-based multicast boundary, this feature applies to **all interfaces on the router** within the applicable routing context.

The primary command is:

```text
ipv6 multicast group-range <acl>
```

An IPv6 ACL identifies permitted and denied multicast groups or `(S,G)` channels.

For denied groups or channels, the router can suppress multicast protocol operations, including:

- Creating MLD listener states.
- Creating multicast routing states.
- Forwarding PIM Join messages.
- Forwarding multicast data packets.

This provides an additional layer of control over the multicast groups and sources a router is allowed to process.

See [IPv6 Multicast Group-Range Filtering](ipv6-group-range.md).

## RP Announcement Filtering

In PIM Sparse Mode, routers must learn the RP responsible for each multicast group.

With **Auto-RP**, Candidate RPs advertise themselves using RP-Announce messages sent to `224.0.1.39`.

An Auto-RP Mapping Agent receives these announcements, selects the appropriate RPs, and distributes the resulting mappings using RP-Discovery messages sent to `224.0.1.40`.

By default, a Mapping Agent accepts incoming Candidate RP announcements.

However, an administrator may want to restrict which routers can advertise themselves as RPs or which multicast group ranges they can advertise.

Cisco IOS XE supports this using:

```text
ip pim rp-announce-filter rp-list <rp-acl> group-list <group-acl>
```

The command is configured on the **Auto-RP Mapping Agent**.

It provides two filtering criteria:

| Filter | Purpose |
|---|---|
| `rp-list` | Controls which Candidate RP addresses are accepted |
| `group-list` | Controls which multicast group ranges are accepted from those Candidate RPs |

For example, an administrator could allow R3 to advertise itself as an RP for `239.1.0.0/16`, while rejecting announcements from unauthorized Candidate RPs.

This is **control-plane filtering**. It affects which RP information the Mapping Agent accepts, rather than directly filtering multicast data packets.

Auto-RP is an IPv4-only mechanism. IPv6 uses other mechanisms, such as BSR and static RP configuration.

See [Auto-RP Announcement Filtering](rp-announcement-filter.md).

## Comparison of Filtering Mechanisms

| Mechanism | Configuration Scope | Primary Purpose |
|---|---|---|
| `ip multicast boundary` | IPv4 interface | Filter multicast groups or channels across an interface |
| `ipv6 multicast boundary scope` | IPv6 interface | Enforce IPv6 multicast scope boundaries |
| `ipv6 multicast group-range` | IPv6 router or VRF | Restrict multicast groups and channels processed by the router |
| `ip pim rp-announce-filter` | Auto-RP Mapping Agent | Filter accepted Candidate RP announcements |
| `ip pim bsr-border` | IPv4 interface | Block PIM Bootstrap Messages |
| `ipv6 pim bsr border` | IPv6 interface | Block PIM Bootstrap Messages |

**A multicast data boundary and a PIM BSR border are not the same thing.**

A multicast data boundary restricts multicast group or channel forwarding, while a BSR border specifically prevents BSR information from being exchanged across an interface.

A BSR border does not, by itself, prevent ordinary multicast data from crossing the interface.

## Key Points

- **Multicast boundaries** restrict the propagation of multicast traffic and associated signaling.
- IPv4 uses `239.0.0.0/8` for administratively scoped multicast.
- IPv4 multicast boundaries use ACLs to identify restricted multicast groups or `(S,G)` channels.
- IPv6 multicast addresses contain a scope field that can be used to enforce administrative boundaries.
- `ipv6 multicast group-range` provides router-wide filtering of IPv6 multicast groups and channels.
- `ip pim rp-announce-filter` controls which Candidate RP advertisements an Auto-RP Mapping Agent accepts.
- **BSR borders** prevent Bootstrap Messages from crossing PIM domain boundaries without directly blocking multicast data forwarding.
