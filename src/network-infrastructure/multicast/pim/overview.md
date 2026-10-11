# 1.6.c PIM

**Protocol Independent Multicast (PIM)** is a multicast routing protocol used to build and maintain multicast distribution trees between sources and receivers.

PIM enables routers to determine where multicast traffic should be forwarded, allowing a single source to deliver traffic to receivers across multiple networks without sending a separate copy to each receiver.

Despite its name, **PIM is not independent of unicast routing information**.

Instead, PIM is *protocol independent* because it does not require any particular unicast routing protocol. It can use routing information learned through OSPF, EIGRP, BGP, static routes, or other sources to perform Reverse Path Forwarding (RPF) lookups.

PIM does not exchange unicast routes or calculate unicast routing paths itself.

## PIM and Multicast Forwarding

PIM operates between multicast routers, coordinating how multicast traffic is distributed through the network.

Consider the following topology:

```text
              Multicast Source
                    |
                    R1
                    |
                    R2
                   /  \
                  /    \
                 R3    R4
                 |      |
              Receiver Receiver
```

PIM allows the routers to establish a multicast distribution tree connecting the source to the interested receivers.

Rather than forwarding multicast traffic throughout the entire network, PIM can maintain forwarding state that identifies:

- The **incoming interface (IIF)** on which multicast traffic is expected to arrive.
- The **Outgoing Interface List (OIL)** identifying the downstream interfaces that require the traffic.

PIM uses the RPF mechanism covered in the previous section to select upstream paths and help prevent multicast forwarding loops.

## PIM vs. IGMP and MLD

PIM, IGMP, and MLD perform different functions.

| Protocol | Purpose |
|---|---|
| IGMP | IPv4 receivers communicate multicast membership to local routers |
| MLD | IPv6 receivers communicate multicast membership to local routers |
| PIM | Multicast routers establish and maintain distribution trees between sources and receivers |

IGMP and MLD operate between hosts and multicast routers, whereas PIM primarily operates between multicast routers.

For example, when a receiver sends an IGMP Membership Report, its local router can use PIM to establish the upstream multicast path needed to deliver the requested traffic.

**IGMP and MLD identify receiver interest; PIM establishes the multicast routing paths needed to satisfy that interest.**

## PIM Operating Modes

PIM supports several operating modes, each using a different approach to multicast distribution.

| Mode | Basic Operation |
|---|---|
| **PIM Dense Mode (PIM-DM)** | Initially floods multicast traffic, then prunes unnecessary branches |
| **PIM Sparse Mode (PIM-SM)** | Builds multicast distribution trees in response to explicit receiver interest |
| **Bidirectional PIM (Bidir-PIM)** | Uses a bidirectional shared tree for traffic from multiple sources |
| **PIM Source-Specific Multicast (PIM-SSM)** | Builds source-specific distribution trees without using an RP |

### PIM Dense Mode

PIM Dense Mode assumes that multicast receivers are relatively widespread throughout the network.

Multicast traffic is initially flooded through the multicast-enabled topology. Routers without downstream interest send **Prune messages** to stop unnecessary forwarding.

This approach is often described as a **flood-and-prune model**.

### PIM Sparse Mode

PIM Sparse Mode assumes that multicast receivers may be sparsely distributed.

Rather than flooding multicast traffic throughout the network, routers explicitly request traffic using **PIM Join messages**.

For traditional Any-Source Multicast (ASM), PIM-SM uses a **Rendezvous Point (RP)** as a meeting point between multicast sources and receivers.

Routers initially build a shared distribution tree rooted at the RP. They can subsequently switch to a source-specific Shortest Path Tree (SPT).

PIM-SM is a major focus of the following chapters.

### Bidirectional PIM

Bidirectional PIM uses a shared distribution tree that supports traffic flowing both toward and away from the RP.

Unlike traditional PIM-SM, Bidirectional PIM does not build separate source trees for individual multicast sources.

It is particularly suited to many-to-many applications involving numerous sources.

### Source-Specific Multicast

Source-Specific Multicast (SSM) allows receivers to request traffic from a particular source and multicast group, identified by an `(S,G)` channel.

Because the source is already known, **SSM does not require an RP, shared tree, or PIM Register process**.

Routers build source-specific distribution trees directly toward the requested multicast sources.

## Common PIM Mechanisms

Several mechanisms are fundamental to PIM operation:

- **PIM Hello Messages:** Discover and maintain PIM neighbor relationships.
- **PIM Designated Router (DR):** Elects a router to perform specific multicast responsibilities on a multiaccess network.
- **PIM Join/Prune Messages:** Establish and remove branches of multicast distribution trees.
- **PIM Assert:** Selects a single multicast forwarder when multiple routers attempt to forward the same traffic onto a shared network.
- **Rendezvous Points (RPs):** Provide a common meeting point for sources and receivers in traditional PIM-SM.

These mechanisms work together to build and maintain multicast distribution trees as receiver membership and network topology change.

## Section Overview

The following chapters examine PIM operation and configuration in greater detail.

### Fundamental PIM Operation

The introductory chapters cover [PIM Hello Messages and Neighbors](hello-neighbors.md), [PIM Designated Router](designated-router.md), [PIM Assert](assert.md), [PIM Dense Mode](dense-mode.md), and [Bidirectional PIM](bidirectional-pim.md).

### CCIE EI Blueprint Topics

1. **[Sparse Mode](sparse-mode/overview.md):** Shared and source trees, multicast state, Join/Prune processing, source registration, and SPT switchover.
2. **[Static RP, BSR, Auto-RP](rp-discovery/overview.md):** Configuring and dynamically discovering Rendezvous Points.
3. **[Group-to-RP Mapping](group-to-rp/overview.md):** Determining which RP is responsible for a multicast group.
4. **[Source-Specific Multicast](ssm/overview.md):** Source-specific forwarding and SSM Mapping.
5. **[Multicast Boundary and RP Announcement Filtering](filtering/overview.md):** Controlling multicast distribution and RP discovery information.
6. **[PIMv6 Anycast RP](pimv6-anycast-rp/overview.md):** Providing redundant RPs for IPv6 multicast.
7. **[IPv4 Anycast RP Using MSDP](msdp-anycast-rp/overview.md):** RP redundancy and source information exchange using MSDP.
8. **[Multicast Multipath](multipath/overview.md):** Distributing multicast streams across equal-cost upstream paths.

Together, these topics explain how PIM discovers neighbors, builds multicast distribution trees, selects RPs, maintains forwarding state, and provides scalable and redundant multicast routing.