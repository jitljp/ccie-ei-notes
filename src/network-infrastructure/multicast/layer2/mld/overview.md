# MLD

**Multicast Listener Discovery (MLD)** is the IPv6 equivalent of **Internet Group Management Protocol (IGMP)**.

MLD allows IPv6 hosts and multicast routers to manage multicast group membership on a local network.

MLD allows:

- Hosts to indicate which IPv6 multicast groups they want to receive traffic for
- Multicast routers to discover which groups have interested receivers on a directly connected link
- Routers to determine when multicast traffic no longer needs to be forwarded onto a link

Unlike IGMP, which uses its own IP protocol number (2), **MLD is part of ICMPv6 (Next Header 58)**.

MLD messages are exchanged on the local link using a Hop Limit of 1. They do not travel between routed network segments.

Like IGMP, MLD manages local receiver membership. It does not determine how multicast traffic is routed through the network. PIM performs that function.

## MLD Versions

There are two major versions of MLD:

| Version | Key Features | IPv4 Equivalent |
|---|---|---|
| **MLDv1** | Group membership reporting, leave notification, group-specific queries | IGMPv2 |
| **MLDv2** | Source filtering, INCLUDE/EXCLUDE modes, SSM support | IGMPv3 |

An important distinction is that **MLDv1 corresponds to IGMPv2**, not IGMPv1.

MLDv2 provides backward compatibility with MLDv1.

## MLDv1

**MLDv1** provides basic IPv6 multicast group membership management.

Multicast routers periodically send **Multicast Listener Queries** to discover interested receivers.

Hosts respond with **Multicast Listener Reports** to indicate which multicast groups they want to receive.

When a host stops listening to a group, it can send a **Multicast Listener Done** message. The router can then query the group to determine whether other receivers remain.

MLDv1 supports group-based membership but does not allow receivers to specify particular multicast sources.

## MLDv2

**MLDv2** extends MLDv1 by adding **source filtering**.

Instead of requesting traffic from any source sending to a multicast group, receivers can specify which sources they want or do not want to receive traffic from.

MLDv2 introduces two source-filtering modes:

- **INCLUDE**: Receive multicast traffic only from specified sources.
- **EXCLUDE**: Receive multicast traffic from all sources except those specified.

This makes MLDv2 especially important for **Source-Specific Multicast (SSM)**, where receivers explicitly request a particular source/group `(S,G)` channel.

MLDv2 also introduces more detailed Multicast Listener Reports containing **Multicast Address Records** that describe receiver membership and source-filtering state.

## MLD Querier

Like IGMP, MLD uses a **querier** to periodically check which multicast groups have interested receivers on a link.

When multiple multicast routers are connected to the same link, an election determines which router acts as the querier.

The querier sends Queries, and receivers respond with Reports indicating their multicast membership state.

MLDv2 also supports queries for particular multicast groups and sources, allowing routers to determine whether specific multicast traffic is still required.

## MLD Snooping

**MLD snooping** is the IPv6 equivalent of IGMP snooping.

A Layer 2 switch examines MLD messages exchanged between IPv6 hosts and multicast routers to determine which switch ports have interested receivers.

The switch can then selectively forward IPv6 multicast traffic to the appropriate ports rather than flooding it throughout the VLAN.

MLD snooping operates independently of IPv6 multicast routing. A switch does not need to act as a multicast router to perform MLD snooping.

## MLD Filtering and Limits

Cisco IOS XE also provides mechanisms to control MLD membership on routed interfaces.

These include:

- **MLD filtering**: Restrict which multicast groups or source/group channels receivers can join.
- **MLD limits**: Restrict the amount of multicast membership or routing state created through MLD.

These controls operate on the router's MLD processing and are distinct from MLDv2's INCLUDE/EXCLUDE source-filtering mechanism.

## Key Points

- **MLD is the IPv6 equivalent of IGMP**, used to manage multicast receiver membership.
- MLD uses **ICMPv6**, whereas IGMP uses a separate IP protocol.
- **MLDv1 corresponds to IGMPv2**, including explicit leave notification.
- **MLDv2 corresponds to IGMPv3**, adding source filtering and SSM support.
- MLD uses a querier to discover and maintain multicast listener state.
- **MLD snooping** allows Layer 2 switches to optimize IPv6 multicast forwarding.
- MLD manages local multicast membership, not multicast routing between networks.

The following sections cover IPv6 multicast addressing, MLDv1, MLDv2, source filtering, querier operation, membership restrictions, and MLD snooping.