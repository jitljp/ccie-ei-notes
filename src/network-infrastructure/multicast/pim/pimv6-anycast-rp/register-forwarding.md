# PIM Register Replication and Source Discovery

**PIM Register replication** allows all members of an Anycast RP set to discover active multicast sources, even when those sources register with different physical RPs.

In traditional PIM Sparse Mode (PIM-SM), a source-facing Designated Router (DR) sends PIM Register messages to the Rendezvous Point (RP).

With Anycast RP, the DR sends Registers to the **shared Anycast RP address**, and unicast routing delivers them to one physical RP.

However, receivers may have joined shared trees rooted at other members of the Anycast RP set.

To solve this problem, **the RP receiving a Register from a DR replicates that Register to the other Anycast RP members**.

This mechanism is defined in RFC 4610 and eliminates the need for MSDP-based source discovery within the Anycast RP set.

## Register Replication Operation

Consider the following topology:

```text
          Shared Anycast RP: 2001:db8:100::100
           RP1 --------------------- RP2
       2001:db8:100::1          2001:db8:100::2
             |                       |
             |                       |
             R1                      R4
            (FHR)                   (LHR)
             |                       |
           Source                 Receiver
     2001:db8:10::10              FF15::100
```

Both RPs share:

```text
2001:db8:100::100
```

Their unique addresses are:

| Router | Unique RP Address |
|---|---|
| RP1 | `2001:db8:100::1` |
| RP2 | `2001:db8:100::2` |

Assume:

- R1 reaches the Anycast RP address through RP1.
- R4 reaches the Anycast RP address through RP2.
- The receiver has already joined `FF15::100`.
- All routers use the same group-to-RP mapping.

### Step 1: Source Registers with the Anycast RP

The source begins sending multicast traffic:

```text
Source: 2001:db8:10::10
Group:  FF15::100
```

R1, acting as the source-facing DR, encapsulates the multicast packet in a PIM Register.

The Register is unicasted to:

```text
2001:db8:100::100
```

Unicast routing delivers it to RP1.

```text
Source
   |
   R1 (DR)
   |
   | PIM Register
   | Destination: 2001:db8:100::100
   v
 RP1
```

R1 does not know that multiple physical RPs share this address.

**No changes to normal DR registration behavior are required for Anycast RP.**

### Step 2: RP1 Processes the Register

RP1 receives the Register and performs normal PIM-SM processing.

It identifies:

```text
(S,G) = (2001:db8:10::10, FF15::100)
```

RP1 creates `(S,G)` state for the source.

If RP1 has interested downstream receivers, it can decapsulate the original multicast packet and forward it along its local shared tree.

RP1 can also initiate an `(S,G)` Join toward the source to establish native multicast forwarding.

### Step 3: RP1 Replicates the Register to RP2

RP1 checks the **source address of the received Register**.

Because the Register came from R1, which is not a member of the Anycast RP set, RP1 recognizes it as a Register originating from a DR.

RP1 then sends a copy to RP2.

```text
 R1 (DR)
    |
    | Register
    | Destination: 2001:db8:100::100
    v
 RP1
    |
    | Replicated Register
    | Source:      2001:db8:100::1
    | Destination: 2001:db8:100::2
    v
 RP2
```

Notice the difference between the two Register messages:

| Field | Original Register | Replicated Register |
|---|---|---|
| Outer IPv6 source | R1 (DR) | RP1's unique address |
| Outer IPv6 destination | Shared Anycast RP address | RP2's unique address |
| Inner IPv6 source | `2001:db8:10::10` | `2001:db8:10::10` |
| Inner IPv6 destination | `FF15::100` | `FF15::100` |

**The outer IPv6 header changes, but the encapsulated multicast packet remains the same.**

Replicated Registers are sent using unicast routing. RP1 and RP2 do not need to be directly connected.

### Step 4: RP2 Learns the Source

RP2 receives the replicated Register from RP1.

It identifies the original multicast source and group:

```text
(S,G) = (2001:db8:10::10, FF15::100)
```

RP2 then:

1. Creates `(S,G)` state.
2. Decapsulates the multicast packet.
3. Forwards the packet along its local shared tree if downstream receivers exist.
4. Can initiate an `(S,G)` Join toward the source to establish native multicast forwarding.

The receiver connected to R4 can now receive traffic from the source, even though R1 originally registered the source with RP1.

```text
Source
   |
   R1 (DR)
   |
   | Register
   v
 RP1
   |
   | Replicated Register
   v
 RP2
   |
   | Multicast traffic
   v
   R4
   |
Receiver
```

This allows sources and receivers to use different physical RPs without requiring them to select the same RP through unicast routing.

## Preventing Register Replication Loops

An important requirement is that **replicated Registers must not be replicated again**.

Suppose an Anycast RP set contains three routers:

```text
RP1: 2001:db8::1
RP2: 2001:db8::2
RP3: 2001:db8::3
```

A DR sends a Register to RP1.

RP1 replicates the Register to RP2 and RP3:

```text
              DR
               |
               | Register
               v
              RP1
             /   \
            /     \
           v       v
          RP2     RP3
```

RP2 and RP3 must not forward these Registers to each other.

Otherwise, replication loops could occur.

### Register Source Address Check

RFC 4610 prevents replication loops by examining the **outer source IP address** of received Register messages.

The rule is:

| Register Source | RP Behavior |
|---|---|
| Source is not an Anycast RP member | Replicate to other RP members |
| Source is an Anycast RP member | Process normally, but do not replicate again |

For example:

```text
Register received from DR:
  Source = 2001:db8:10::1
  Not in Anycast RP set
  → Replicate
```

```text
Register received from RP1:
  Source = 2001:db8::1
  Member of Anycast RP set
  → Do not replicate
```

This is why every Anycast RP must have a consistent list of peer addresses.

> **Additional protection:** RFC 4610 also requires a replicating RP to preserve the original Register's outer IP TTL/Hop Limit when sending copies. This helps limit the effects of inconsistent Anycast RP peer lists.

## Register-Stop Processing

Register-Stop messages continue to operate according to normal PIM-SM procedures.

However, there are two separate relationships:

1. Between the original DR and RP1, which receives its Register.
2. Between the replicating RP and the other Anycast RP members.

### Register-Stop to the Original DR

After RP1 receives a Register from R1, it can send a Register-Stop to R1.

```text
R1 (DR)                      RP1
   |                            |
   |-------- Register --------->|
   |                            |
   |<------ Register-Stop -------|
```

R1 suppresses further data Register encapsulation for the affected `(S,G)` according to the normal PIM Register suppression process.

### Register-Stop Between RPs

After RP2 receives a replicated Register from RP1, it may send a Register-Stop back to RP1.

```text
RP1                     RP2
   |                            |
   |-------- Register --------->|
   |                            |
   |<------ Register-Stop -------|
```

RP2 may delay its Register-Stop if it decides to establish an `(S,G)` source tree and wait for native multicast traffic.

**An important distinction is that the replicating RP does not use a peer's Register-Stop to control its own Register replication state in the same way that a normal DR uses Register-Stop to suppress data Registers.**

Under RFC 4610, the receiving RP processes the peer's Register-Stop without a specific additional action.

Implementations may also suppress Register-Stop responses to Registers received from other RP members.

## Null-Register Replication

Normal PIM-SM uses **Null-Register messages** to probe the RP after Register-Stop suppression.

A Null-Register contains the source/group identification but does not carry the original multicast data payload.

For example:

```text
R1 → RP1: Null-Register(S,G)
```

In an Anycast RP network, these messages are also replicated.

```text
R1 (DR)
   |
   | Null-Register
   v
 RP1
   |
   | Replicated Null-Register
   v
 RP2
```

**RFC 4610 requires Null-Registers to be replicated to every other member of the Anycast RP set.**

This allows the RP members to maintain or refresh source discovery state for active multicast sources even when ordinary data Registers are being suppressed.

If replication fails temporarily, later Register or Null-Register messages provide opportunities for the other RPs to discover the source again.

## Source State vs. Forwarding State

PIM Register replication is often described as **state synchronization**, but it is important to understand what is actually synchronized.

The Anycast RP members do not exchange their entire multicast routing tables.

Instead, replicated Registers provide source information that allows each RP to maintain its own `(S,G)` state.

For example:

| State or Information | Shared Between RPs? |
|---|---|
| Active source/group information `(S,G)` | Yes, through Register replication |
| `(*,G)` shared-tree state | No |
| Outgoing Interface List (OIL) | No |
| RPF interface and neighbor | No |
| Complete multicast routing table | No |

Each RP independently maintains its own multicast forwarding state.

### An RP Without Receivers

Suppose RP3 receives a replicated Register but has no downstream receivers for that group.

RP3 still creates `(S,G)` state.

However, it does not need to forward the encapsulated multicast data because it has no interested receivers.

For example:

```text
RP1
 |
 | Replicated Register
 v
RP3
 |
 | No receivers
 |
 +-- Create (S,G) state
 +-- Discard unnecessary multicast data
```

If a receiver subsequently joins a shared tree rooted at RP3, RP3 already knows the source and can establish a source tree toward it.

**RFC 4610 requires the RP to create `(S,G)` state when processing the Register, but initiating an immediate `(S,G)` Join toward the source is optional.**

## Native Multicast Forwarding

Register replication is primarily a source discovery mechanism. It is not intended to carry multicast data indefinitely between RP members.

After discovering the source, an RP with downstream interest can establish a native `(S,G)` path toward the source.

For example:

```text
          Source
             |
             R1
            /  \
           /    \
          RP1     RP2
                  |
                  R4
                  |
               Receiver
```

Initially, RP2 may receive multicast data through replicated Registers from RP1.

After RP2 establishes an `(S,G)` path toward the source, it can receive the traffic natively through its own RPF path.

The resulting multicast forwarding path can be:

```text
Source → R1 → RP2 → R4 → Receiver
```

RP1 is no longer required to relay the data through Register encapsulation.

The receiver-facing router may subsequently switch to a Shortest-Path Tree (SPT), potentially bypassing the RP entirely.

## IOS XE Configuration

PIM Register replication is enabled by configuring the Anycast RP peer relationships.

The command is:

```text
ipv6 pim anycast-rp <anycast-rp-address> <peer-address>
```

For the two-RP example:

**On RP1:**

```text
RP1(config)# ipv6 pim anycast-rp 2001:db8:100::100 2001:db8:100::1
RP1(config)# ipv6 pim anycast-rp 2001:db8:100::100 2001:db8:100::2
```

**On RP2:**

```text
RP2(config)# ipv6 pim anycast-rp 2001:db8:100::100 2001:db8:100::1
RP2(config)# ipv6 pim anycast-rp 2001:db8:100::100 2001:db8:100::2
```

Both RPs have the same complete peer list, including their own unique addresses.

No additional command is needed to enable Register replication.

The full configuration, including IPv6 addressing, group-to-RP mapping, routing, and failover, is covered in [Anycast RP Configuration and Failover](configuration.md).

## IOS XE Verification

### Verify Anycast RP Peers

The primary verification command is:

```text
RP1# show ipv6 pim anycast-rp 2001:db8:100::100
```

Cisco IOS XE displays the configured Anycast RP peers and information about received Register and Register-Stop messages.

The output includes:

```text
Anycast RP Peers For 2001:DB8:100::100
Last Register/Register-Stop received
```

The peer entries identify the Anycast RP members and show their Register-related activity.

Check that:

- All expected RP members appear.
- The configured unique RP addresses are correct.
- Register-related activity is observed when sources are active.

### Verify Multicast Routing State

On RP1 and RP2:

```text
RP1# show ipv6 mroute FF15::100
RP2# show ipv6 mroute FF15::100
```

Check for the expected `(S,G)` state:

```text
(2001:db8:10::10, FF15::100)
```

Both RPs should be able to learn about the source.

However, their OILs and RPF information may be different because they maintain their forwarding state independently.

### Verify Group-to-RP Mapping

```text
R1# show ipv6 pim group-map FF15::100
```

Verify that the group maps to the shared Anycast RP address:

```text
2001:db8:100::100
```

### Verify Register Tunnels

```text
R1# show ipv6 pim tunnel
RP1# show ipv6 pim tunnel
RP2# show ipv6 pim tunnel
```

PIM Register tunnels provide encapsulation and decapsulation functionality for PIM Register messages.

The tunnel information can help identify Register-related problems.

### Debug PIM Operation

When detailed troubleshooting is necessary:

```text
RP1# debug ipv6 pim
```

Use debugging carefully on production devices.

## Common Problems

| Problem | Possible Cause |
|---|---|
| Only one RP learns the source | Register replication is not working |
| Replicated Registers do not reach peers | Unique RP addresses are unreachable |
| Register replication behaves inconsistently | Anycast RP peer lists differ between members |
| An RP learns `(S,G)` but has no traffic to forward | No downstream receiver interest, or no native source tree established |
| Receiver joins one RP but cannot receive traffic from a source registered elsewhere | Source discovery or subsequent multicast forwarding has failed |
| Excessive replicated Register traffic | Excessive source registration activity or an improperly configured RP set |

When troubleshooting, distinguish between:

1. **Source registration:** Does the DR successfully register the source with an RP?
2. **Register replication:** Does that RP distribute the source information to the other RP members?
3. **Native multicast forwarding:** Can the receiving RP establish the necessary multicast forwarding path?

Successful source discovery does not guarantee successful multicast data forwarding.

## Key Points

- The source-facing DR sends PIM Registers to the **shared Anycast RP address**.
- Unicast routing delivers the Register to one physical RP.
- That RP replicates the Register to the other Anycast RP members using their **unique IPv6 addresses**.
- The replicated Register's outer source address is the replicating RP's unique address.
- Replicated Registers are **not replicated again**, preventing loops.
- Every RP receiving a Register creates `(S,G)` source state.
- Each RP maintains its own `(*,G)` and `(S,G)` forwarding state.
- Register-Stop messages operate according to normal PIM-SM procedures, with no special synchronization action required at the originating RP.
- **Null-Registers must also be replicated** to maintain source discovery information.
- An RP can learn an active source even when it has no interested downstream receivers.
- Register replication does not synchronize the entire multicast routing table.
- Native `(S,G)` forwarding can replace the initial delivery of multicast data through replicated Registers.
- RFC 4610 defines the PIM-based Anycast RP mechanism for both IPv4 and IPv6, without requiring MSDP.

## Further Reading

- [RFC 4610: Anycast-RP Using Protocol Independent Multicast (PIM)](https://www.rfc-editor.org/rfc/rfc4610)
- [Cisco IOS XE 17.x: PIMv6 Anycast RP Solution](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-multicast/b-ip-multicast/m_ip6-pimv6-anycast-rp-xe.html)
