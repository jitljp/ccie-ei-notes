# 1.6.c (vii) IPv4 Anycast RP Using MSDP

**Anycast Rendezvous Point (Anycast RP)** provides **RP redundancy and load sharing** in IPv4 PIM Sparse Mode (PIM-SM) networks.

Instead of assigning different RP addresses to different routers, multiple RPs share the **same IPv4 address**.

Unicast routing determines which physical RP each router reaches when sending PIM control-plane messages toward the shared RP address.

However, a problem arises when a multicast source registers with one RP while receivers join another.

**Multicast Source Discovery Protocol (MSDP)** solves this problem by allowing RPs to exchange information about active multicast sources.

## Why Anycast RP Is Needed

In a traditional PIM-SM network, a multicast group is associated with an RP.

The source-facing Designated Router (DR) sends PIM Register messages to that RP, while the Last-Hop Router (LHR) sends `(*,G)` Joins toward it.

Using a single RP creates several limitations:

- **Single point of failure:** If the RP fails, new source registrations and receiver joins may be disrupted.
- **Centralized control-plane load:** One RP processes registrations and shared-tree joins for its assigned groups.
- **Suboptimal RP placement:** Sources and receivers throughout the network may need to reach a distant RP.

Anycast RP addresses these limitations by allowing multiple physical routers to serve as the RP for the same multicast groups.

Each RP advertises the same anycast address into the unicast routing system.

Routers use their normal routing tables to reach the closest RP in terms of routing metrics.

**All PIM routers use the same Group-to-RP mapping, even though they may reach different physical RPs.**

## Anycast RP Addressing

Consider the following topology:

```text
Src --- R1 --- RP1 --- RP2 --- R4 --- Rec
```

The devices have the following roles:

| Device | Role |
|---|---|
| Src | Multicast source |
| R1 | First-Hop Router (FHR) |
| RP1 | First Anycast RP |
| RP2 | Second Anycast RP |
| R4 | Last-Hop Router (LHR) |
| Rec | Multicast receiver |

RP1 and RP2 have the following addresses:

| Address | RP1 | RP2 |
|---|---|---|
| Anycast RP address | `10.0.0.100/32` | `10.0.0.100/32` |
| Unique MSDP address | `10.0.0.1/32` | `10.0.0.2/32` |

Both RPs advertise `10.0.0.100/32` into the unicast routing system.

The other routers use the same Group-to-RP mapping:

```text
Group: 239.1.1.1
RP:    10.0.0.100
```

However, based on the unicast routing topology:

- R1 reaches `10.0.0.100` through RP1.
- R4 reaches `10.0.0.100` through RP2.

The two RPs also establish an MSDP peering session using their **unique addresses**, not the shared anycast address.

> **Note:** This is a simplified lab topology. Actual RP redundancy requires alternative routing paths so that routers can continue reaching the surviving RP after a failure.

## The Source Discovery Problem

Suppose Src begins sending multicast traffic:

```text
Source (S): 192.0.2.10
Group (G):  239.1.1.1
```

R1 sends PIM Register messages toward the anycast RP address:

```text
10.0.0.100
```

Because RP1 is the closest RP according to R1's routing table, the Register messages reach RP1.

Meanwhile, Rec joins `239.1.1.1`.

R4 sends a PIM `(*,G)` Join toward the same anycast address, but its routing table directs the Join toward RP2.

The result is:

| RP | State |
|---|---|
| RP1 | Knows about the active source |
| RP2 | Knows about the interested receiver |

**Without an additional mechanism, RP2 does not automatically learn that a source has registered with RP1.**

Although both routers share the same RP address, their multicast routing state is maintained independently.

This is why MSDP is needed.

## Multicast Source Discovery Protocol (MSDP)

**MSDP** allows multicast routers, typically RPs, to exchange information about active multicast sources.

MSDP was originally developed to exchange source information between separate PIM-SM domains, each with its own RP.

In an Anycast RP deployment, MSDP performs this function between RPs within the same multicast domain.

MSDP has three important characteristics:

| Characteristic | Description |
|---|---|
| Transport | TCP port `639` |
| Peering | Explicitly configured between MSDP speakers |
| Source discovery | Uses Source-Active (SA) messages to advertise active `(S,G)` sources |

MSDP does not exchange IGMP membership information, replicate `(*,G)` state, or synchronize the complete multicast routing tables of the RPs.

Instead, it distributes information about active sources so that other RPs can establish their own multicast forwarding state when needed.

### Source-Active Messages

When RP1 learns about an active source, it originates an **MSDP Source-Active (SA) message**.

The SA message identifies information including:

- The source address, `S`.
- The multicast group address, `G`.
- The originating RP address or MSDP Originator ID.

For example:

```text
Originating RP: 10.0.0.1
Source:         192.0.2.10
Group:          239.1.1.1
```

RP1 sends the SA message to RP2 over their MSDP session.

RP2 can then learn that `192.0.2.10` is an active source for `239.1.1.1`.

If RP2 has receivers interested in the group, it can initiate a PIM `(S,G)` Join toward the source.

**MSDP advertises source availability. PIM establishes the multicast forwarding path.**

## Basic Anycast RP Operation

Using the previous topology:

```text
Src --- R1 --- RP1 --- RP2 --- R4 --- Rec
```

Assume:

- Src sends traffic to `239.1.1.1`.
- R1 reaches the anycast RP address through RP1.
- R4 reaches the anycast RP address through RP2.
- RP1 and RP2 have an established MSDP session.

The basic operation is:

1. **Receiver registration:** Rec sends an IGMP Membership Report for `239.1.1.1`. R4 sends a PIM `(*,G)` Join toward the anycast RP address, reaching RP2.
2. **Source registration:** Src begins sending multicast traffic. R1 sends PIM Register messages to the anycast RP address, reaching RP1.
3. **Source advertisement:** RP1 originates an MSDP SA message advertising `(192.0.2.10, 239.1.1.1)` to RP2.
4. **SA processing:** RP2 receives the SA message and learns that the source is active.
5. **Source-tree construction:** Because RP2 has interested receivers for the group, it sends a PIM `(S,G)` Join toward the source.
6. **Multicast forwarding:** Multicast traffic reaches RP2 through the source tree and can be forwarded toward R4 and Rec along the shared tree.

The LHR may subsequently switch to a Shortest-Path Tree (SPT) directly toward the source, bypassing the RP.

The important point is that **RP1 and RP2 can independently serve sources and receivers while exchanging the source information needed to establish multicast connectivity**.

## MSDP Originator ID

In an Anycast RP deployment, all RPs share the same anycast address.

However, **MSDP SA messages must identify the originating RP using a unique address**.

For the example topology:

```text
Anycast RP: 10.0.0.100

RP1 Originator ID: 10.0.0.1
RP2 Originator ID: 10.0.0.2
```

Using the shared anycast address as the SA Originator ID would cause ambiguity and can result in failed MSDP Peer-RPF checks.

IOS XE supports explicitly configuring the SA Originator ID:

```text
RP1(config)# ip msdp originator-id Loopback1
RP2(config)# ip msdp originator-id Loopback1
```

Each router's `Loopback1` has its own unique IPv4 address.

The originator ID and MSDP peering addresses are covered in [MSDP Peering and Originator ID](peering.md).

## MSDP Peer-RPF and Mesh Groups

MSDP can advertise source information through multiple peers.

Without loop prevention, an SA message could circulate between MSDP speakers.

MSDP therefore uses a specialized **Peer Reverse Path Forwarding (Peer-RPF)** mechanism to determine whether received SA messages should be accepted.

Unlike multicast data-plane RPF checks, MSDP Peer-RPF operates on **SA control messages**.

Peer-RPF selection can use BGP routing information, but MSDP also supports exceptions that allow simple Anycast RP deployments without BGP.

For larger deployments, MSDP supports **Mesh Groups**.

A mesh group consists of fully meshed MSDP peers. SA messages received from another peer in the same mesh group are not forwarded to other members of that mesh group.

These mechanisms are covered in [MSDP Peer-RPF and Mesh Groups](peer-rpf.md).

## Anycast RP Failover

Anycast RP provides redundancy by making the shared RP address reachable through multiple physical routers.

Suppose RP1 fails.

The unicast routing system withdraws the route to the anycast address through RP1.

If another path to RP2 exists, R1 can begin reaching `10.0.0.100` through RP2 instead.

As a result:

- New PIM Register messages from R1 reach RP2.
- New receiver Joins can reach RP2.
- RP2 continues serving as an RP for the same multicast groups.
- Existing traffic using an established SPT that bypasses RP1 may continue without interruption.

**Anycast RP failover is driven by unicast routing convergence**, rather than an RP election between RP1 and RP2.

Failover behavior also depends on connectivity, multicast forwarding state, and whether the failed RP was part of an active multicast distribution tree.

## MSDP vs. PIMv6 Anycast RP

IPv4 MSDP-based Anycast RP and PIMv6 Anycast RP provide similar functionality, but use different mechanisms.

| Feature | IPv4 Anycast RP Using MSDP | PIMv6 Anycast RP |
|---|---|---|
| Shared anycast RP address | Yes | Yes |
| Unique addresses for individual RPs | Yes | Yes |
| Exchange mechanism | MSDP | PIM |
| TCP peering | Yes, port `639` | No |
| Source information exchange | MSDP SA messages | PIM Register replication |
| MSDP Peer-RPF | Yes | No |
| RP redundancy | Yes | Yes |

**IPv4 Anycast RP using MSDP advertises active sources between RPs, whereas PIMv6 Anycast RP replicates PIM Register messages between RPs.**

MSDP is an IPv4-only protocol.

## Section Overview

The following pages examine IPv4 Anycast RP and MSDP in greater detail:

- [Anycast RP Addressing and MSDP Operation](msdp-overview.md): Shared and unique RP addresses, routing behavior, and the interaction between PIM-SM and MSDP.
- [MSDP Peering and Originator ID](peering.md): TCP peering, peer configuration, Originator IDs, authentication, timers, and verification.
- [Source-Active Messages and SA Cache](sa-messages.md): SA message generation, propagation, caching, source discovery, and filtering.
- [MSDP Peer-RPF and Mesh Groups](peer-rpf.md): SA message loop prevention, Peer-RPF rules, BGP considerations, and MSDP Mesh Groups.

## Key Points

- **Anycast RP** allows multiple physical RPs to share the same IPv4 RP address.
- Unicast routing determines which physical RP receives PIM Registers and shared-tree Joins.
- Different sources and receivers may use different physical RPs for the same multicast group.
- **MSDP** enables RPs to exchange information about active multicast sources.
- MSDP uses **TCP port 639** and explicitly configured peering sessions.
- **SA messages** advertise active `(S,G)` sources and their originating RPs.
- Each Anycast RP requires a unique address for MSDP peering and SA origination.
- MSDP does not synchronize complete multicast routing state or receiver membership information.
- **Peer-RPF** prevents SA message loops, while **Mesh Groups** simplify SA propagation in fully meshed deployments.
- RP failover relies on unicast routing convergence and continued reachability of the surviving RP.
- MSDP operates with IPv4, whereas PIMv6 Anycast RP uses PIM Register replication without MSDP.

## References

- [Cisco IOS XE: Configuring Basic IP Multicast (Anycast RP)](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-multicast/b-ip-multicast/m_imc_basic_cfg-0_2.html)
- [Cisco IOS XE: Using MSDP to Interconnect Multiple PIM-SM Domains](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-multicast/b-ip-multicast/m_imc_msdp_im_pim_sim.html)
- [RFC 3446: Anycast Rendevous Point (RP) Mechanism Using PIM and MSDP](https://www.rfc-editor.org/rfc/rfc3446)
- [RFC 3618: Multicast Source Discovery Protocol (MSDP)](https://www.rfc-editor.org/rfc/rfc3618)