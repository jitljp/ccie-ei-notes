# 1.6.c (vi) PIMv6 Anycast RP

**PIMv6 Anycast RP** provides Rendezvous Point (RP) redundancy and load distribution for IPv6 PIM Sparse Mode (PIM-SM) networks.

In traditional PIM-SM, a single RP is responsible for connecting multicast sources and receivers for a particular group. If that RP fails, multicast communication may be disrupted until another RP becomes available.

**Anycast RP** addresses this problem by allowing multiple routers to share the **same RP address**.

Each RP advertises reachability to the shared address through the unicast routing infrastructure, allowing PIM routers to reach the closest RP according to their unicast routing tables.

This provides:

- **Redundancy:** Multiple routers can serve as the RP for the same multicast group.
- **Load distribution:** Different parts of the network can use different RPs simultaneously.
- **Fast failover:** When an RP fails, unicast routing convergence redirects traffic toward a surviving RP.
- **Consistent RP mapping:** All PIM routers can use the same group-to-RP mapping, regardless of which physical RP they reach.

PIM-based Anycast RP is defined in **RFC 4610**, *Anycast-RP Using Protocol Independent Multicast (PIM)*.

Although commonly associated with IPv6, the mechanism supports both IPv4 and IPv6.

## Basic Anycast RP Concept

Consider the following network:

```text
                   Anycast RP Address
                    2001:db8::100
                           |
               +-----------+-----------+
               |                       |
            R3 (RP1)                R4 (RP2)
         2001:db8::3             2001:db8::4
               |                       |
              R1                      R2
             (FHR)                   (LHR)
               |                       |
             Source                 Receiver
```

Both R3 and R4 have the same Anycast RP address configured on a loopback interface:

```text
R3: 2001:db8::100/128
R4: 2001:db8::100/128
```

However, they also have unique IPv6 addresses:

```text
R3: 2001:db8::3
R4: 2001:db8::4
```

The addresses have different purposes:

| Address | Purpose |
|---|---|
| Shared Anycast RP address | Used by PIM routers to locate the RP |
| Unique RP addresses | Used for communication between members of the Anycast RP set |

Both RPs advertise reachability to `2001:db8::100/128` through the unicast routing infrastructure.

As a result:

- R1 may reach the Anycast RP address through R3.
- R2 may reach the same Anycast RP address through R4.
- Both routers use the **same group-to-RP mapping**, even though they communicate with different physical RPs.

There is no separate RP election between the Anycast RP members. The unicast routing table determines which RP a router reaches.

## PIM Register Replication

A fundamental challenge with Anycast RP is **source discovery**.

In ordinary PIM-SM, a source-facing Designated Router (DR) sends PIM Register messages to the RP to announce an active multicast source.

However, with Anycast RP, the source-facing DR sends these messages only to the RP selected by unicast routing.

For example:

1. R1 receives multicast traffic from a source.
2. R1 sends a PIM Register to `2001:db8::100`, which reaches R3.
3. R3 learns about the active source.
4. R3 forwards a copy of the PIM Register to R4 using R4's unique IPv6 address.
5. R4 also learns about the source and can establish an `(S,G)` path toward it.

This allows receivers using R4 to receive traffic from a source originally registered with R3.

**PIM Register replication** is the central mechanism of RFC 4610.

Importantly, this mechanism does not synchronize the complete multicast routing tables of the RPs. Instead, it distributes source registration information, allowing each RP to establish its own multicast forwarding state.

An RP that receives a replicated Register from another RP does **not** replicate that message again to the other members. This prevents replication loops.

## Anycast RP Failover

Suppose R3 fails.

Before the failure:

```text
R1 → R3 (Anycast RP)
R2 → R4 (Anycast RP)
```

After unicast routing converges:

```text
R1 → R4 (Anycast RP)
R2 → R4 (Anycast RP)
```

R1 continues to use `2001:db8::100` as the RP address, but its unicast routing table now directs packets toward R4.

The group-to-RP mapping does not need to change.

This allows PIM Register messages and shared-tree Joins to reach a surviving RP without requiring a new RP discovery or election process.

Existing multicast forwarding state may need to be rebuilt, so failover is not necessarily lossless.

## PIM-Based Anycast RP vs. MSDP-Based Anycast RP

Two mechanisms can be used to implement Anycast RP.

| Feature | PIM-Based Anycast RP | MSDP-Based Anycast RP |
|---|---|---|
| Protocol | PIM | MSDP |
| IPv4 support | Yes | Yes |
| IPv6 support | Yes | No |
| Source discovery between RPs | PIM Register replication | MSDP Source-Active (SA) messages |
| Separate source discovery protocol required | No | Yes |
| Relevant RFC | RFC 4610 | RFC 3618 (MSDP) |

**PIM-based Anycast RP does not require MSDP.**

This is especially important for IPv6 because MSDP is an IPv4-only protocol.

IPv4 Anycast RP using MSDP is covered separately in [1.6.c (vii) IPv4 Anycast RP Using MSDP](../msdp-anycast-rp/overview.md).

## Key Points

- PIMv6 Anycast RP provides RP redundancy and load distribution for IPv6 PIM-SM.
- Multiple RPs share the same IPv6 RP address, but each also has a unique IPv6 address.
- Unicast routing determines which physical RP receives PIM Register messages and shared-tree Joins.
- PIM Register messages are replicated between Anycast RP peers to distribute source discovery information.
- Replicated Register messages are not replicated again, preventing loops.
- When an RP fails, unicast routing convergence redirects traffic toward a surviving RP.
- The group-to-RP mapping remains unchanged during failover.
- PIM-based Anycast RP is defined in RFC 4610 and supports both IPv4 and IPv6.
- Unlike MSDP-based Anycast RP, it does not require a separate source discovery protocol.

## Further Reading

- [RFC 4610: Anycast-RP Using Protocol Independent Multicast (PIM)](https://www.rfc-editor.org/rfc/rfc4610)
- [Cisco IOS XE 17.x: PIMv6 Anycast RP Solution](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-multicast/b-ip-multicast/m_ip6-pimv6-anycast-rp-xe.html)