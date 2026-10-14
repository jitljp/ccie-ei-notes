# Anycast RP Configuration and Failover

**PIMv6 Anycast RP** allows multiple physical routers to share the same Rendezvous Point (RP) address, providing redundancy and allowing different parts of the network to use different physical RPs.

The configuration requires:

1. A **shared Anycast RP address** configured on every RP.
2. A **unique IPv6 address** for each RP.
3. Unicast routing to advertise the shared and unique addresses.
4. A consistent **group-to-RP mapping** throughout the PIM domain.
5. An identical **Anycast RP peer list** on every RP.

When an RP fails, unicast routing redirects traffic destined for the shared Anycast RP address to another member of the Anycast RP set.

This page focuses on IOS XE configuration, verification, and failover. For the source discovery mechanism, see [PIM Register Replication and Source Discovery](register-forwarding.md).

## Example Topology

Consider a network with two Anycast RPs:

```text
                       IPv6 Multicast Network

                               RP1
                       2001:db8:100::1
                                |
                                |
 Source ---- R1 --------- Multicast --------- R4 ---- Receiver
             (FHR)         Network           (LHR)
                                |
                                |
                               RP2
                       2001:db8:100::2

              Shared Anycast RP Address:
                  2001:db8:100::100
```

Both RP1 and RP2 use the same Anycast RP address, but each has its own unique address.

| Device | Address | Purpose |
|---|---|---|
| RP1 Loopback0 | `2001:db8:100::100/128` | Shared Anycast RP |
| RP1 Loopback1 | `2001:db8:100::1/128` | RP1's unique address |
| RP2 Loopback0 | `2001:db8:100::100/128` | Shared Anycast RP |
| RP2 Loopback1 | `2001:db8:100::2/128` | RP2's unique address |
| Source | `2001:db8:1::10` | Multicast source |
| Multicast group | `FF15::1` | ASM group |

Assume:

- All routers use OSPFv3 for IPv6 unicast routing.
- IPv6 multicast routing is enabled throughout the PIM domain.
- R1 is the source-facing PIM Designated Router (DR).
- R4 is the last-hop router (LHR).
- R1's shortest unicast path to `2001:db8:100::100` leads to RP1.
- R4's shortest unicast path to the same address leads to RP2.

The source and receiver can therefore use different physical RPs.

## Step 1: Configure the Anycast RP Addresses

Each RP requires two addresses:

- **Shared address:** Used by PIM routers as the RP address.
- **Unique address:** Used for Register replication between Anycast RP members.

### RP1 Addressing

Configure the shared address on Loopback0:

```text
RP1(config)# interface Loopback0
RP1(config-if)# ipv6 address 2001:db8:100::100/128
```

Configure RP1's unique address on Loopback1:

```text
RP1(config)# interface Loopback1
RP1(config-if)# ipv6 address 2001:db8:100::1/128
```

### RP2 Addressing

Configure the same shared address on RP2:

```text
RP2(config)# interface Loopback0
RP2(config-if)# ipv6 address 2001:db8:100::100/128
```

Configure RP2's unique address:

```text
RP2(config)# interface Loopback1
RP2(config-if)# ipv6 address 2001:db8:100::2/128
```

The resulting addressing is:

```text
                  RP1                        RP2

Loopback0   2001:db8:100::100          2001:db8:100::100
            (Shared Anycast RP)        (Shared Anycast RP)

Loopback1   2001:db8:100::1            2001:db8:100::2
            (Unique RP1 IP)            (Unique RP2 IP)
```

**The shared Anycast RP address must be identical on all RPs, while their unique addresses must be different.**

The unique addresses do not have to belong to the same subnet as the shared address. They only need to be routable IPv6 unicast addresses.

## Step 2: Advertise the RP Addresses

The shared Anycast RP address and each RP's unique address must be reachable through unicast routing.

In this example, OSPFv3 is used.

### RP1 OSPFv3 Configuration

Assuming OSPFv3 process 1 is already configured:

```text
RP1(config)# interface Loopback0
RP1(config-if)# ospfv3 1 ipv6 area 0

RP1(config)# interface Loopback1
RP1(config-if)# ospfv3 1 ipv6 area 0
```

### RP2 OSPFv3 Configuration

```text
RP2(config)# interface Loopback0
RP2(config-if)# ospfv3 1 ipv6 area 0

RP2(config)# interface Loopback1
RP2(config-if)# ospfv3 1 ipv6 area 0
```

OSPFv3 must also be operational on the routed transit interfaces so that the loopback prefixes are distributed through the network.

### Resulting Routing Behavior

Both RP1 and RP2 advertise:

```text
2001:db8:100::100/128
```

Other routers can therefore learn two possible paths to the same destination.

The selected path depends on the unicast routing table.

For example:

```text
R1:
2001:db8:100::100/128 -> RP1

R4:
2001:db8:100::100/128 -> RP2
```

This allows the source-facing and receiver-facing routers to use different physical RPs.

**Anycast RP selection is performed by unicast routing, not by a PIM RP election between the Anycast RP members.**

### Verify RP Reachability

On R1:

```text
R1# show ipv6 route 2001:db8:100::100
```

Verify which physical RP the route leads toward.

Also verify reachability to the unique addresses:

```text
R1# show ipv6 route 2001:db8:100::1
R1# show ipv6 route 2001:db8:100::2
```

On RP1:

```text
RP1# ping 2001:db8:100::2
```

On RP2:

```text
RP2# ping 2001:db8:100::1
```

Successful reachability between the unique addresses is necessary for Register replication.

> **Important:** Configuring the correct loopback addresses is not enough. The loopbacks must also be advertised into the unicast routing domain. If an RP's loopback is missing from OSPFv3, PIM registration or Register replication can fail even when the RP's physical interfaces are reachable.

## Step 3: Enable IPv6 Multicast Routing

Enable IPv6 unicast and multicast routing on the participating routers.

On RP1:

```text
RP1(config)# ipv6 unicast-routing
RP1(config)# ipv6 multicast-routing
```

On RP2:

```text
RP2(config)# ipv6 unicast-routing
RP2(config)# ipv6 multicast-routing
```

On the remaining PIM routers:

```text
R1(config)# ipv6 unicast-routing
R1(config)# ipv6 multicast-routing
```

```text
R4(config)# ipv6 unicast-routing
R4(config)# ipv6 multicast-routing
```

Repeat on intermediate multicast routers.

**All interfaces participating in the multicast distribution paths must have IPv6 PIM operational.**

## Step 4: Configure the Group-to-RP Mapping

All PIM routers must map the relevant multicast groups to the **shared Anycast RP address**, not to either RP's unique address.

The global command is:

```text
ipv6 pim rp-address <anycast-rp-address> [group-access-list]
```

For this example:

```text
RP1(config)# ipv6 pim rp-address 2001:db8:100::100
RP2(config)# ipv6 pim rp-address 2001:db8:100::100
R1(config)# ipv6 pim rp-address 2001:db8:100::100
R4(config)# ipv6 pim rp-address 2001:db8:100::100
```

Configure the same mapping on intermediate PIM routers.

This maps applicable IPv6 ASM groups, including `FF15::1`, to:

```text
2001:db8:100::100
```

The routers do not need separate RP mappings for RP1 and RP2.

Instead, they have a single logical RP address:

```text
Group: FF15::1
RP:    2001:db8:100::100
```

Unicast routing determines which physical RP is reached.

### Optional: Restrict the RP Mapping to Specific Groups

An IPv6 ACL can restrict the mapping to a particular multicast group or range.

For example:

```text
R1(config)# ipv6 access-list ASM-GROUPS
R1(config-ipv6-acl)# permit ipv6 any host FF15::1
```

Apply the ACL:

```text
R1(config)# ipv6 pim rp-address 2001:db8:100::100 ASM-GROUPS
```

The ACL's destination address matches the multicast group.

Apply equivalent configurations consistently throughout the PIM domain.

For more information, see [Static RP](../rp-discovery/static-rp.md).

### Verify the Group-to-RP Mapping

```text
R1# show ipv6 pim group-map FF15::1
```

Verify that the selected RP address is:

```text
2001:db8:100::100
```

Repeat the verification on the other routers.

## Step 5: Configure the Anycast RP Peer Lists

The Anycast RP peer relationships are configured using:

```text
ipv6 pim anycast-rp <anycast-rp-address> <peer-address>
```

The first address identifies the shared Anycast RP.

The second address identifies a unique RP member.

### RP1 Configuration

```text
RP1(config)# ipv6 pim anycast-rp 2001:db8:100::100 2001:db8:100::1
RP1(config)# ipv6 pim anycast-rp 2001:db8:100::100 2001:db8:100::2
```

### RP2 Configuration

```text
RP2(config)# ipv6 pim anycast-rp 2001:db8:100::100 2001:db8:100::1
RP2(config)# ipv6 pim anycast-rp 2001:db8:100::100 2001:db8:100::2
```

Notice that **both routers have identical peer lists**.

| Command | RP1 | RP2 |
|---|---|---|
| Peer `2001:db8:100::1` | Configured (self) | Configured |
| Peer `2001:db8:100::2` | Configured | Configured (self) |

### The Local RP Must Be Included

**Every RP must include its own unique IPv6 address in the Anycast RP peer list.**

For example, RP1 must configure:

```text
RP1(config)# ipv6 pim anycast-rp 2001:db8:100::100 2001:db8:100::1
```

Even though `2001:db8:100::1` belongs to RP1 itself.

Similarly, RP2 must configure:

```text
RP2(config)# ipv6 pim anycast-rp 2001:db8:100::100 2001:db8:100::2
```

Omitting the local RP's address can prevent correct Anycast RP operation and Register replication.

The peer list must contain **all RP members, including the local router**.

This requirement is easy to overlook because a typical peer configuration only identifies remote devices.

> **Important:** The peer list is statically configured. The RPs do not automatically discover additional Anycast RP members, and there is no separate PIM Anycast RP peer election.

### Verify the Peer Lists

On RP1:

```text
RP1# show ipv6 pim anycast-rp 2001:db8:100::100
```

Example output:

```text
Anycast RP Peers For 2001:DB8:100::100   Last Register/Register-Stop received
  2001:DB8:100::2 00:01:30/00:00:09
  2001:DB8:100::1 00:00:31/00:00:31
```

On RP2:

```text
RP2# show ipv6 pim anycast-rp 2001:db8:100::100
```

Example output:

```text
Anycast RP Peers For 2001:DB8:100::100   Last Register/Register-Stop received
  2001:DB8:100::2 00:00:39/00:00:39
  2001:DB8:100::1 00:00:21/00:01:30
```

The output shows:

- The shared Anycast RP address.
- The configured RP members.
- The time since Register and Register-Stop messages were last received.

The timestamps help verify Register-related activity. They are not a separate Anycast RP keepalive or election mechanism.

Check that both RPs display the same two member addresses.

## Step 6: Verify Source Discovery and Multicast Forwarding

Assume the source begins transmitting:

```text
Source: 2001:db8:1::10
Group:  FF15::1
```

R1 sends PIM Registers to:

```text
2001:db8:100::100
```

Based on unicast routing, RP1 receives them.

RP1 replicates the Registers to RP2 using the unique addresses.

RP2 can then discover the source and establish multicast forwarding toward interested receivers.

### Verify the Source-Facing Router

On R1:

```text
R1# show ipv6 mroute FF15::1
```

During registration, an example entry is:

```text
(2001:DB8:1::10, FF15::1), flags: SFT
  Incoming interface: Ethernet0/0
  RPF nbr: 2001:DB8:1::10, Registering
  Immediate Outgoing interface list:
    Tunnel1, Forward
```

The `F` flag indicates Register-related state.

The PIM encapsulation tunnel is used to send Register messages toward the RP.

### Verify RP1

```text
RP1# show ipv6 mroute FF15::1
```

An example entry is:

```text
(2001:DB8:1::10, FF15::1), flags: SP
  Incoming interface: Ethernet0/0
  RPF nbr: FE80::1
  Outgoing interface list: Null
```

This indicates RP1 has learned about the source.

The `P` flag indicates pruned state.

An empty OIL is not necessarily an error. RP1 may have no downstream receivers because those receivers are using RP2.

### Verify RP2

```text
RP2# show ipv6 mroute FF15::1
```

For example:

```text
(*, FF15::1), RP 2001:DB8:100::100, flags: S
  Incoming interface: Tunnel2
  RPF nbr: 2001:DB8:100::100
  Immediate Outgoing interface list:
    Ethernet0/0, Forward

(2001:DB8:1::10, FF15::1), flags: S
  Incoming interface: Ethernet0/1
  RPF nbr: FE80::2
  Inherited Outgoing interface list:
    Ethernet0/0, Forward
```

This demonstrates:

- RP2 has `(*,G)` state with downstream receiver interest.
- RP2 has learned the source's `(S,G)` state.
- RP2 has an outgoing interface toward the receivers.
- RP2 has an RPF path toward the source.

The precise interfaces and flags depend on the topology and forwarding state.

**Successful Register replication allows RP2 to learn the source even though the original DR registered it with RP1.**

### Additional Verification Commands

Check PIM message counters:

```text
RP1# show ipv6 pim traffic
RP2# show ipv6 pim traffic
```

Check Register tunnels:

```text
R1# show ipv6 pim tunnel
RP1# show ipv6 pim tunnel
RP2# show ipv6 pim tunnel
```

Check PIM neighbors:

```text
RP1# show ipv6 pim neighbor
RP2# show ipv6 pim neighbor
```

These commands help distinguish between failures in source registration, peer replication, and native multicast forwarding.

## Anycast RP Failover

The main advantage of Anycast RP is that the failure of one RP does not require routers to learn a different group-to-RP mapping.

The shared Anycast RP address remains unchanged.

### Normal Operation

Assume the source-facing router R1 reaches the Anycast RP through RP1.

The receiver-facing router R4 reaches it through RP2.

```text
                Shared RP: 2001:db8:100::100

Source                                       Receiver
   |                                            |
   R1                                           R4
   |                                            |
   | Registers                                  | (*,G) Joins
   v                                            v
  RP1 -------- Register Replication ---------> RP2
  ::1                                            ::2
```

RP1 receives source registration traffic and replicates it to RP2.

RP2 uses the source information to establish multicast forwarding toward receivers.

### RP1 Fails

Suppose RP1 becomes unreachable.

The unicast route to the shared Anycast RP address through RP1 is withdrawn.

R1 still has the same group-to-RP mapping:

```text
Group: FF15::1
RP:    2001:db8:100::100
```

However, its route to the shared RP address now leads to RP2:

```text
Before failure:

2001:db8:100::100 -> RP1

After failure:

2001:db8:100::100 -> RP2
```

The source-facing DR can continue using the same destination address for PIM Registers.

The network redirects those Registers to RP2.

```text
                  RP1
                 FAILED
                   X

Source
   |
   R1 (FHR)
   |
   | Register
   | Destination: 2001:db8:100::100
   v
  RP2
   |
   | Native multicast forwarding
   v
   R4
   |
Receiver
```

RP2 now acts as the reachable RP for routers whose paths previously led to RP1.

Any surviving Anycast RP members continue serving the group using the same shared address.

### What Changes During Failover?

| Component | Behavior |
|---|---|
| Group-to-RP mapping | Remains unchanged |
| Shared Anycast RP address | Remains unchanged |
| Unicast route to the shared RP | Changes to a surviving RP |
| PIM Registers | Redirected according to the new unicast route |
| `(*,G)` Join path | Reestablished as needed toward the surviving RP |
| `(S,G)` forwarding | Updated according to multicast routing and RPF state |
| Anycast RP peer lists | No immediate configuration changes required |

**Failover depends on unicast routing convergence and the establishment of the necessary multicast forwarding state.**

There is no separate active/standby election between RP1 and RP2.

### Receiver-Side Failover

Suppose R4 originally reached the shared RP address through RP1.

When RP1 fails:

1. R4's unicast route toward the shared RP changes.
2. Its RPF path toward the RP changes.
3. PIM Join processing establishes the shared-tree path toward RP2.
4. RP2 can forward the multicast traffic toward R4.

Because RP2 participates in source discovery before the failure, it may already know about active sources.

This can reduce the time required to resume multicast forwarding.

However, **Anycast RP does not guarantee lossless failover**.

Temporary packet loss can occur while the unicast and multicast routing state converges.

### Impact of Existing SPT Forwarding

A receiver-facing router may already have switched from the shared tree to a Shortest-Path Tree (SPT).

For example:

```text
Source -> R1 -> R2 -> R4 -> Receiver
```

In this case, multicast data no longer needs to pass through the RP.

If RP1 fails, the existing source tree may continue forwarding traffic without interruption, provided its forwarding path is unaffected.

Therefore, **RP failure does not necessarily interrupt an existing multicast stream**.

The impact depends on whether the forwarding state still relies on the failed RP.

## Testing Anycast RP Failover

Failover can be tested in a lab by making RP1 unreachable while RP2 remains operational.

Before testing, verify:

```text
R1# show ipv6 route 2001:db8:100::100
R1# show ipv6 pim group-map FF15::1
RP2# show ipv6 pim anycast-rp 2001:db8:100::100
```

Confirm that R1 currently reaches the shared Anycast RP through RP1.

### Method 1: Withdraw RP1's Anycast Address

Shut down the shared Anycast RP loopback on RP1:

```text
RP1(config)# interface Loopback0
RP1(config-if)# shutdown
```

Assuming this loopback is the source of RP1's OSPFv3 advertisement for the shared address, its local route is withdrawn.

R1 should then select a route toward RP2:

```text
R1# show ipv6 route 2001:db8:100::100
```

The expected change is:

```text
Before:
2001:db8:100::100 -> RP1

After:
2001:db8:100::100 -> RP2
```

The RP mapping itself should remain unchanged:

```text
R1# show ipv6 pim group-map FF15::1
```

Expected RP:

```text
2001:db8:100::100
```

This tests failover of the shared RP address.

> **Note:** Shutting down Loopback0 is not the same as a complete RP1 router failure. RP1 may still be reachable through its unique address and may retain other routing or multicast functionality.

Restore the loopback afterward:

```text
RP1(config)# interface Loopback0
RP1(config-if)# no shutdown
```

### Method 2: Simulate Complete RP Failure

For a more realistic test, shut down RP1's routed connectivity or stop the router entirely in the lab.

This removes both:

```text
2001:db8:100::100/128
2001:db8:100::1/128
```

from the reachable network when the associated routes are withdrawn.

Verify that:

1. R1 changes its route to the shared RP address.
2. R1's Registers can reach RP2.
3. R4 maintains or reestablishes the required multicast state.
4. The receiver can receive traffic from the source.

To test source registration independently of previously established SPT forwarding, start a new multicast stream or join a new ASM group after the failure.

### RP1 Recovery

When RP1 returns:

1. Its shared Anycast RP address becomes reachable again.
2. OSPFv3 advertises its routes.
3. Routers recalculate their unicast paths.
4. Routers that prefer RP1 may begin reaching the shared address through RP1 again.
5. RP1 resumes its role as a member of the Anycast RP set.

The group-to-RP mapping remains:

```text
2001:db8:100::100
```

No separate RP election or manual switchover is required.

A recovered RP can become the preferred RP for some routers while other routers continue using RP2.

This is normal Anycast RP behavior.

## Common Configuration and Failover Problems

| Symptom | Possible Cause |
|---|---|
| R1 cannot reach the shared RP address | Anycast loopbacks are not advertised into unicast routing |
| Only one RP learns the multicast source | Register replication failure |
| RP1 can reach RP2's shared address but not its unique address | Unique RP address is missing from routing |
| RP2 has `(*,G)` but no `(S,G)` | Register replication or source discovery failure |
| Register replication fails even though the RP addresses are reachable | Incomplete or inconsistent Anycast RP peer lists |
| RP is listed in `show ipv6 pim anycast-rp`, but no recent activity appears | No recent relevant Register traffic, or a Register-processing problem |
| Registers reach an RP but receivers do not receive traffic | Missing receiver interest, multicast forwarding state, or RPF problem |
| RP1 fails but traffic remains directed toward it | Stale or incorrect unicast route to the shared address |
| Traffic is interrupted temporarily during failover | Unicast or multicast convergence is still occurring |
| RP1 fails but existing multicast traffic continues | Receivers may already be using an SPT that bypasses the RP |

## Key Points

- Every Anycast RP uses the **same shared RP address** and a **different unique IPv6 address**.
- The shared address must be advertised through unicast routing by every RP member.
- Each RP's unique address must also be reachable through unicast routing.
- All PIM routers use the shared Anycast RP address in their group-to-RP mappings.
- Configure `ipv6 pim anycast-rp <anycast-rp-address> <peer-address>` on the RP members.
- **Each RP must configure the complete peer list, including its own unique address.**
- All RPs must have consistent Anycast RP peer lists.
- The source-facing DR sends Registers to the shared Anycast RP address.
- Register replication uses the unique RP addresses.
- `show ipv6 pim anycast-rp` verifies the configured peers and recent Register-related activity.
- When an RP fails, unicast routing redirects traffic for the shared RP address to a surviving RP.
- The group-to-RP mapping does not need to change during failover.
- There is no separate Anycast RP active/standby election.
- RP failure does not necessarily interrupt multicast traffic already flowing along an SPT.
- Failover depends on unicast routing convergence and multicast forwarding state, and is not guaranteed to be lossless.

## Further Reading

- [RFC 4610: Anycast-RP Using Protocol Independent Multicast (PIM)](https://www.rfc-editor.org/rfc/rfc4610)
- [Cisco IOS XE 17.x: PIMv6 Anycast RP Solution](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-multicast/b-ip-multicast/m_ip6-pimv6-anycast-rp-xe.html)
- [Cisco IOS XE 17.x: IPv6 Multicast PIM Sparse Mode](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-multicast/b-ip-multicast/m_ip6-mcast-pim-sm-0.html)