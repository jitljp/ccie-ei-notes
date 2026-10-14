# Anycast RP Addressing and Operation

**PIM Anycast RP** allows multiple routers to share the same Rendezvous Point (RP) address while operating as independent RPs.

Each Anycast RP has two types of IPv6 addresses:

- **Shared Anycast RP address:** Identical on every RP in the Anycast RP set.
- **Unique RP address:** Different on each RP, used for communication between Anycast RP members.

The shared address is advertised into the unicast routing domain by all Anycast RPs. This allows First-Hop Routers (FHRs) and Last-Hop Routers (LHRs) to reach different physical RPs while using the same group-to-RP mapping.

## Anycast RP Addressing

Consider the following topology:

```text
                  Shared Anycast RP Address
                     2001:db8:100::100
                             |
              +--------------+--------------+
              |                             |
             RP1                           RP2
       2001:db8:100::1               2001:db8:100::2
              |                             |
              R1                            R4
             (FHR)                         (LHR)
              |                             |
            Source                       Receiver
```

RP1 and RP2 belong to the same **Anycast RP set**.

Their IPv6 addressing is:

| Router | Anycast RP Address | Unique RP Address |
|---|---|---|
| RP1 | `2001:db8:100::100/128` | `2001:db8:100::1/128` |
| RP2 | `2001:db8:100::100/128` | `2001:db8:100::2/128` |

### Shared Anycast RP Address

The shared address is normally configured on a loopback interface on each RP.

On RP1:

```text
RP1(config)# interface Loopback0
RP1(config-if)# ipv6 address 2001:db8:100::100/128
```

On RP2:

```text
RP2(config)# interface Loopback0
RP2(config-if)# ipv6 address 2001:db8:100::100/128
```

Both routers now have the same IPv6 address configured.

This address is used for:

- Group-to-RP mapping.
- PIM Register messages sent by source-facing DRs.
- RPF lookups and `(*,G)` Joins toward the RP.

**The shared Anycast RP address is not used for Register replication between RP members.**

### Unique RP Addresses

Each RP also requires a unique IPv6 address.

These addresses are used for direct communication between Anycast RP members, particularly for forwarding copies of PIM Register messages.

For example, on RP1:

```text
RP1(config)# interface Loopback1
RP1(config-if)# ipv6 address 2001:db8:100::1/128
```

On RP2:

```text
RP2(config)# interface Loopback1
RP2(config-if)# ipv6 address 2001:db8:100::2/128
```

Although the unique addresses can be assigned to physical interfaces, loopback interfaces are generally preferred because they remain operational independently of individual physical interface failures.

The unique addresses must be reachable through the unicast routing infrastructure.

## Unicast Routing and RP Selection

The Anycast RP address must be advertised into the unicast routing domain by **every member of the Anycast RP set**.

For example, both RP and RP2 advertise:

```text
2001:db8:100::100/128
```

This can be accomplished using an IGP such as OSPFv3 or IS-IS, or another appropriate unicast routing mechanism.

Because multiple routers advertise the same prefix, other routers may have different best paths toward the Anycast RP address.

For example:

| Router | RP Address | Best Path |
|---|---|---|
| R1 | `2001:db8:100::100` | Toward RP1 |
| R4 | `2001:db8:100::100` | Toward RP2 |

When R1 performs a route lookup for `2001:db8:100::100`, its best path leads to RP1.

When R4 performs the same lookup, its best path leads to RP2.

**The physical RP is selected by unicast routing, not by a separate PIM RP election.**

The selected RP is based on the unicast routing metrics, not necessarily the number of hops.

If multiple equal-cost routes exist, ECMP behavior may also influence which physical RP receives a particular packet.

### Routing Requirements

Three important conditions must be satisfied:

1. All Anycast RP members must advertise reachability to the shared Anycast RP address.
2. Every RP member's unique IPv6 address must be reachable by the other members.
3. The unicast routing infrastructure must remove routes toward failed RP members so traffic can be redirected.

The shared Anycast RP address and unique RP addresses serve different purposes and must not be confused.

## Group-to-RP Mapping

Even though multiple physical RPs exist, **all PIM routers use the same RP address for a given multicast group**.

For example:

```text
Multicast Group:  FF15::100
RP Address:       2001:db8:100::100
```

A static mapping can be configured using:

```text
R1(config)# ipv6 pim rp-address 2001:db8:100::100
```

The same mapping can be configured on the other PIM routers.

Alternatively, the shared RP address can be distributed using BSR.

The important requirement is that routers agree on the effective group-to-RP mapping.

For example:

| Router | Group | Mapped RP | Physical RP Reached |
|---|---|---|---|
| R1 | `FF15::100` | `2001:db8:100::100` | RP |
| R4 | `FF15::100` | `2001:db8:100::100` | RP2 |

Notice that **the mapped RP address is identical**, even though the routers reach different physical RPs.

The group-to-RP mapping identifies the logical RP, while unicast routing determines which physical router receives traffic sent toward that address.

> PIM Anycast RP is used for Any-Source Multicast (ASM). Source-Specific Multicast (SSM) does not require an RP.

## Source-Side Operation

When an IPv6 multicast source begins transmitting, the source-facing DR performs normal PIM-SM source registration.

For example, suppose:

```text
Source: 2001:db8:10::10
Group:  FF15::100
RP:     2001:db8:100::100
```

The process is:

1. The source sends a multicast packet to `FF15::100`.
2. R1, acting as the source-facing DR, receives the packet.
3. R1 determines that the RP for `FF15::100` is `2001:db8:100::100`.
4. R1 sends a unicast PIM Register message to the Anycast RP address.
5. Unicast routing delivers the Register to RP.

From R1's perspective, this is **normal PIM-SM source registration**. R1 does not need to know that multiple physical RPs share the address.

However, because RP2 may have receivers using its own shared tree, RP must inform RP2 about the active source.

This is accomplished through **PIM Register replication**, which is covered in [Register Forwarding and State Synchronization](register-forwarding.md).

## Receiver-Side Operation

Receivers also use normal PIM-SM procedures.

For example, suppose a receiver connected to R4 joins `FF15::100`.

The process is:

1. The receiver sends an MLD Report for `FF15::100`.
2. R4, acting as the receiver-facing PIM DR, creates `(*,G)` state.
3. R4 identifies the RP as `2001:db8:100::100`.
4. R4 performs an RPF lookup toward the RP address.
5. The best path leads toward RP2.
6. R4 sends a `(*,G)` Join toward RP2, establishing the shared tree.

The resulting logical shared tree is:

```text
RP2 (RP2)
   |
   | (*, FF15::100)
   |
   R4
   |
Receiver
```

Meanwhile, R1 may be registering the source with RP instead of RP2.

```text
        Source                       Receiver
           |                             |
           R1                            R4
           |                             |
     PIM Register                    (*,G) Join
           |                             |
           v                             v
        RP (RP1)                      RP2 (RP2)
           |                             ^
           |                             |
           +----- Register Copy ---------+
```

This illustrates the fundamental challenge with Anycast RP:

- The source registers with one physical RP.
- The receiver joins a shared tree rooted at another physical RP.
- Both RPs must discover the same active source so multicast traffic can reach the receivers.

PIM Register replication solves this problem.

Once the RPs learn about the source, each RP can independently establish an `(S,G)` source tree when necessary.

Receiver-facing routers can also switch to an SPT, potentially bypassing the RP entirely.

## The Anycast RP Set

The **Anycast RP set** consists of all physical RPs sharing a particular Anycast RP address.

Each member must know the unique IPv6 addresses of the other members.

On IOS XE, these relationships are configured using:

```text
ipv6 pim anycast-rp <anycast-rp-address> <peer-address>
```

For the example topology, the Anycast RP set contains:

```text
Anycast RP: 2001:db8:100::100

Members:
  2001:db8:100::1   (RP)
  2001:db8:100::2   (RP2)
```

The complete member list, **including the local router's unique address**, should be configured identically on every RP in the set.

The local router recognizes its own address and does not replicate Register messages to itself.

Only the RP members require the Anycast RP peer configuration. Ordinary PIM routers such as R1 and R4 simply use the shared RP address.

> Important: Each Anycast RP must include **its own unique IPv6 address** in the `ipv6 pim anycast-rp` peer list, not just the addresses of the other RPs.
> Omitting the local RP address can prevent Register replication and source discovery, even when unicast connectivity is working. **All RPs should have identical peer lists**.

The complete configuration is covered in [Anycast RP Configuration and Failover](configuration.md).

## Verifying RP Reachability and Selection

Several IOS XE commands help verify the addressing and routing behavior.

### Unicast Route to the Anycast RP

```text
R1# show ipv6 route 2001:db8:100::100
```

This identifies the route used to reach the shared RP address.

Run the command on different routers to compare their selected paths.

### Reachability of Unique RP Addresses

```text
RP# show ipv6 route 2001:db8:100::2
RP2# show ipv6 route 2001:db8:100::1
```

The unique addresses must be reachable for Register replication to work.

### Group-to-RP Mapping

```text
R1# show ipv6 pim group-map FF15::100
```

Verify that the group maps to the shared Anycast RP address.

### Anycast RP Membership

On an Anycast RP member:

```text
RP# show ipv6 pim anycast-rp 2001:db8:100::100
```

This command displays Anycast RP information for the specified shared RP address.

It can be used to verify the configured Anycast RP set.

## Key Points

- All members of an Anycast RP set share the **same RP IPv6 address**.
- Each RP also has a **unique IPv6 address** for communication with other RP members.
- Both the shared address and the unique addresses must be reachable through unicast routing.
- The shared Anycast RP address is advertised by every RP member.
- Unicast routing determines which physical RP a router reaches.
- All routers use the same group-to-RP mapping, regardless of which physical RP they reach.
- Source-facing DRs send PIM Registers to the shared Anycast RP address.
- Receiver-facing routers send `(*,G)` Joins toward the shared Anycast RP address.
- Different sources and receivers can use different physical RPs simultaneously.
- PIM Register replication allows the RPs to discover sources registered with other members.
- The Anycast RP member list must be consistent across all RP members and include the local RP's unique address.
- No additional RP election is required for Anycast RP failover.

## Further Reading

- [RFC 4610: Anycast-RP Using Protocol Independent Multicast (PIM)](https://www.rfc-editor.org/rfc/rfc4610)
- [Cisco IOS XE 17.x: PIMv6 Anycast RP Solution](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-multicast/b-ip-multicast/m_ip6-pimv6-anycast-rp-xe.html)