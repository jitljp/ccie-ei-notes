# Anycast RP Addressing and MSDP Operation

**IPv4 Anycast RP using MSDP** allows multiple physical routers to serve as the Rendezvous Point (RP) for the same multicast groups.

All Anycast RPs share a common **Anycast RP address**, but each RP also has a **unique IPv4 address** for MSDP peering and Source-Active (SA) message origination.

Unicast routing directs PIM Register messages and shared-tree Joins toward the nearest RP, while **Multicast Source Discovery Protocol (MSDP)** allows the RPs to exchange information about active multicast sources.

## Anycast RP Addressing

Consider the following topology:

```text
Src --- R1 --- RP1 --- RP2 --- R4 --- Rec
```

- **Src:** Multicast source
- **R1:** First-Hop Router (FHR)
- **RP1:** First Anycast RP
- **RP2:** Second Anycast RP
- **R4:** Last-Hop Router (LHR)
- **Rec:** Multicast receiver

RP1 and RP2 use the following addresses:

| Address | RP1 | RP2 |
|---|---|---|
| Anycast RP | `10.0.0.100/32` | `10.0.0.100/32` |
| Unique MSDP address | `10.0.0.1/32` | `10.0.0.2/32` |

Each RP has two loopback interfaces:

- **Loopback0:** Shared Anycast RP address
- **Loopback1:** Unique MSDP peering address

For example, on RP1:

```text
RP1(config)# interface Loopback0
RP1(config-if)# ip address 10.0.0.100 255.255.255.255
RP1(config-if)# ip pim sparse-mode

RP1(config)# interface Loopback1
RP1(config-if)# ip address 10.0.0.1 255.255.255.255
```

On RP2:

```text
RP2(config)# interface Loopback0
RP2(config-if)# ip address 10.0.0.100 255.255.255.255
RP2(config-if)# ip pim sparse-mode

RP2(config)# interface Loopback1
RP2(config-if)# ip address 10.0.0.2 255.255.255.255
```

The shared Anycast RP address must be configured as a **/32 host address**.

Both RPs advertise `10.0.0.100/32` into the unicast routing domain, typically using an IGP such as OSPF or EIGRP.

Their unique addresses must also be reachable through unicast routing.

> **Important:** MSDP peering must use unique addresses. The shared Anycast RP address must not be used as an MSDP peering address or as the originating RP address in SA messages.

## Group-to-RP Mapping

All PIM routers, **including the RPs themselves**, use the same Group-to-RP mapping.

For example:

```text
Group: 239.1.1.1
RP:    10.0.0.100
```

This can be configured statically on all four routers:

```text
R1(config)# ip pim rp-address 10.0.0.100
RP1(config)# ip pim rp-address 10.0.0.100
RP2(config)# ip pim rp-address 10.0.0.100
R4(config)# ip pim rp-address 10.0.0.100
```

Although RP1 and RP2 share the same RP address, each router independently maintains its Group-to-RP mapping.

Alternatively, the mapping can be distributed dynamically using BSR or Auto-RP.

**No special Group-to-RP mapping mechanism is required for Anycast RP.**

The key difference is that multiple physical routers own the same RP address.

### Unicast Routing Determines the Physical RP

Although R1 and R4 have identical Group-to-RP mappings, their unicast routing tables may select different paths toward `10.0.0.100`.

For example:

| Router | Destination | Next Hop |
|---|---|---|
| R1 | `10.0.0.100/32` | Toward RP1 |
| R4 | `10.0.0.100/32` | Toward RP2 |

As a result:

- R1 sends PIM Registers toward RP1.
- R4 sends `(*,G)` Joins toward RP2.

Neither router needs to know that multiple physical RPs exist.

**Anycast RP selection is based on normal unicast routing, not a PIM RP election.**

If the unicast routing metrics change, a router may begin reaching the shared address through a different RP without changing its Group-to-RP mapping.

## The Source Discovery Problem

Suppose Src begins sending multicast traffic:

```text
Source (S): 192.0.2.10
Group (G):  239.1.1.1
```

R1 receives the multicast packets and sends PIM Register messages to the configured RP address:

```text
10.0.0.100
```

Because R1's unicast routing table directs this address toward RP1, RP1 receives the Registers.

RP1 now knows that `192.0.2.10` is an active source for `239.1.1.1`.

Meanwhile, Rec sends an IGMP Membership Report for the same group.

R4 sends a PIM `(*,G)` Join toward `10.0.0.100`, which reaches RP2.

The resulting state is:

| RP | Source Information | Receiver Interest |
|---|---|---|
| RP1 | Knows about `(192.0.2.10, 239.1.1.1)` | No downstream receiver interest |
| RP2 | Has not learned about the source | Has downstream `(*,G)` state |

RP1 and RP2 maintain their multicast routing state independently.

**Sharing an IP address does not automatically synchronize multicast source information between the RPs.**

Without MSDP, RP2 may not discover the source registered with RP1, preventing the receiver from receiving multicast traffic.

## How MSDP Solves the Problem

MSDP allows RP1 and RP2 to exchange information about active multicast sources.

The RPs establish a TCP session using their unique addresses:

```text
RP1                               RP2
10.0.0.1                         10.0.0.2
   |                                |
   |--------- TCP Port 639 ---------|
   |                                |
```

MSDP peering is explicitly configured using `ip msdp peer`.

Unlike PIM neighbor relationships, **MSDP peers do not need to be directly connected**. Their unique addresses simply need to be reachable through unicast routing.

### Step 1: Source Registers with RP1

Src sends traffic to `239.1.1.1`.

R1 encapsulates the multicast packets in PIM Register messages and sends them to the Anycast RP address.

Unicast routing delivers the Registers to RP1.

```text
Src ---> R1 ---> RP1
         PIM Register
```

RP1 learns the source and group:

```text
(S,G) = (192.0.2.10, 239.1.1.1)
```

### Step 2: RP1 Sends an MSDP SA Message

RP1 originates an **MSDP Source-Active (SA) message** to inform RP2 about the active source.

The SA message identifies:

```text
Originating RP: 10.0.0.1
Source (S):     192.0.2.10
Group (G):      239.1.1.1
```

The originating RP address is RP1's unique address, not the shared Anycast RP address.

```text
RP1 ----------------------> RP2
        MSDP SA Message
```

An initial SA message can also carry an encapsulated multicast data packet. Subsequent periodic SA advertisements normally carry source information without multicast data.

RP2 processes the SA message and can store the source information in its **SA cache**.

> **Note:** SA messages advertise active sources. They do not replicate PIM Register messages or synchronize complete multicast routing tables.

### Step 3: RP2 Joins the Source Tree

RP2 already has downstream `(*,G)` state because R4 joined `239.1.1.1`.

After learning about the source through MSDP, RP2 performs an RPF lookup toward `192.0.2.10`.

It sends a PIM `(S,G)` Join toward the source:

```text
RP2 ---> RP1 ---> R1
       PIM (S,G) Join
```

This establishes a source-specific path from R1 toward RP2.

**MSDP provides source discovery, while PIM establishes the actual multicast forwarding path.**

If RP2 had no interested receivers, it would not need to join the source tree merely because it received an SA message.

### Step 4: Multicast Traffic Reaches the Receiver

Once the source tree is established, multicast traffic can flow natively from Src toward RP2 and then along the existing shared tree toward Rec.

```text
Src ---> R1 ---> RP1 ---> RP2 ---> R4 ---> Rec
```

The receiver can now receive traffic from `192.0.2.10`.

R4 may subsequently switch to a Shortest-Path Tree (SPT) toward the source, depending on its PIM configuration.

Once R4 is receiving traffic along the SPT, the traffic no longer needs to pass through RP2 as an RP.

## What MSDP Actually Exchanges

MSDP does not synchronize all multicast-related information between RPs.

| Information | Exchanged by MSDP? |
|---|---|
| Active `(S,G)` source information | Yes |
| Originating RP address | Yes |
| Initial encapsulated multicast data | Possible |
| IGMP membership information | No |
| PIM `(*,G)` state | No |
| Complete PIM `(S,G)` routing state | No |
| PIM Register messages | No |
| Unicast routing information | No |

Each RP independently maintains its own PIM state.

For example, RP1 may have an active `(S,G)` entry because R1 registered the source.

RP2 may have:

- An MSDP SA cache entry identifying the source.
- `(*,G)` state associated with R4.
- An `(S,G)` entry after joining the source tree.

These are distinct types of state.

The SA cache is not equivalent to the multicast routing table.

## MSDP SA Origination

An important distinction is that an RP **originates SA messages for locally registered sources**, not for every source it learns about through MSDP.

In the example:

- RP1 learns about Src through PIM Register messages from R1.
- RP1 originates an SA message for the source.
- RP2 receives the SA message and learns about the source.
- RP2 does not originate a new SA message for that source simply because it received RP1's advertisement.

In larger MSDP topologies, an MSDP speaker may forward received SA messages to other peers, subject to Peer-RPF and mesh-group rules.

This is different from **originating** an SA message for a locally registered source.

SA generation, propagation, caching, and filtering are covered in [Source-Active Messages and SA Cache](sa-messages.md).

## MSDP Originator ID

MSDP SA messages identify the RP that originated the advertisement.

Normally, the RP address can serve this purpose.

However, with Anycast RP, all RPs share the same address.

Therefore, each RP must use a **unique Originator ID** for its SA messages.

For the example topology:

| RP | Anycast RP Address | MSDP Originator ID |
|---|---|---|
| RP1 | `10.0.0.100` | `10.0.0.1` |
| RP2 | `10.0.0.100` | `10.0.0.2` |

Configure the Originator ID using the unique loopback interface:

```text
RP1(config)# ip msdp originator-id Loopback1
RP2(config)# ip msdp originator-id Loopback1
```

This causes each RP to use its Loopback1 address as the originating RP address in locally generated SA messages.

If both RPs use `10.0.0.100` as their MSDP Originator ID, SA messages can fail Peer-RPF checks or be rejected as invalid.

The unique Originator ID is therefore an essential part of the Anycast RP design.

## Anycast RP and Unicast Routing

Anycast RP depends on the unicast routing system for two separate purposes.

### Reaching the Anycast RP Address

Both RPs advertise the shared `/32` address:

```text
10.0.0.100/32
```

Routers select their preferred route to this prefix.

If RP1 becomes unavailable and an alternative path to RP2 exists, R1 can begin sending PIM Registers toward RP2.

The Group-to-RP mapping remains unchanged.

### Reaching the Unique MSDP Addresses

The unique RP addresses must also be reachable:

```text
RP1: 10.0.0.1/32
RP2: 10.0.0.2/32
```

These addresses are used for MSDP peering and SA origination.

Their reachability is independent of the route to the shared Anycast RP address.

For example, an MSDP session may fail even though both RPs remain reachable through the shared address.

Conversely, MSDP peering may remain established even when a routing problem prevents some routers from reaching the appropriate Anycast RP.

> **Design Note:** The linear topology used in this section illustrates Anycast RP and MSDP operation. Actual RP failover requires redundant physical and routing paths. If RP1 fails in a strictly linear topology, R1 would lose connectivity to RP2.

### Router ID Considerations

When running an IGP on the RPs, the shared Anycast RP address should not be selected as the routing protocol's Router ID.

For example, configure unique OSPF Router IDs:

```text
RP1(config-router)# router-id 10.0.0.1
RP2(config-router)# router-id 10.0.0.2
```

A duplicate Router ID can cause routing protocol problems.

Using the unique MSDP addresses as routing protocol Router IDs is a convenient design choice.

## Key Points

- **Anycast RP** allows multiple physical routers to share the same RP address.
- The shared Anycast RP address is typically configured on a loopback interface using a `/32` mask.
- Each RP also requires a **unique IPv4 address** for MSDP peering and SA origination.
- All PIM routers use the same Group-to-RP mapping.
- **Unicast routing determines which physical RP receives PIM Registers and shared-tree Joins.**
- Different sources and receivers may use different physical RPs for the same multicast group.
- Sharing an RP address does not automatically synchronize multicast routing information.
- **MSDP exchanges active source information using SA messages.**
- MSDP uses TCP port `639` and does not require directly connected peers.
- An RP receiving an SA message can establish an `(S,G)` source tree when it has interested receivers.
- MSDP does not exchange IGMP membership information or synchronize complete PIM routing state.
- Each Anycast RP must use a unique **MSDP Originator ID**.
- Anycast RP failover is driven primarily by unicast routing convergence and requires alternative paths to the surviving RP.

## References

- [Cisco IOS XE: Configuring Basic IP Multicast, Anycast RP](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-multicast/b-ip-multicast/m_imc_basic_cfg-0_2.html)
- [Cisco IOS XE: MSDP](https://www.cisco.com/c/en/us/td/docs/switches/lan/c9000/multicast/multicast-configuration-guide/msdp.html)
- [RFC 3446: Anycast Rendezvous Point Mechanism Using PIM and MSDP](https://www.rfc-editor.org/rfc/rfc3446)
- [RFC 3618: Multicast Source Discovery Protocol](https://www.rfc-editor.org/rfc/rfc3618)