# MSDP Peer-RPF and Mesh Groups
**MSDP Peer Reverse Path Forwarding (Peer-RPF)** prevents Source-Active (SA) messages from circulating endlessly between MSDP peers.

When an MSDP speaker receives an SA message, it normally determines whether the message arrived from the correct MSDP peer for reaching the **originating RP**.

If the Peer-RPF check succeeds, the SA message can be accepted and forwarded to other MSDP peers.

If the check fails, the SA message is discarded.

**MSDP Mesh Groups** simplify this process by allowing fully meshed MSDP peers to accept SA messages from one another without performing Peer-RPF checks.

Mesh groups are particularly useful for IPv4 Anycast RP deployments.

## MSDP Peer-RPF Overview
Consider the following topology:

```text
Src --- R1 --- RP1 --- RP2 --- R4 --- Rec
```

RP1 and RP2 share the following Anycast RP address:

```text
10.0.0.100
```

Their unique MSDP addresses are:

| Router | MSDP Address |
|---|---|
| RP1 | `10.0.0.1` |
| RP2 | `10.0.0.2` |

Suppose RP1 originates an SA message:

```text
Originating RP: 10.0.0.1
Source (S):     192.0.2.10
Group (G):      239.1.1.1
```

RP2 receives this message over its MSDP peering session with RP1.

The fundamental Peer-RPF question is:

**Is RP1 the correct MSDP peer from which RP2 should accept SA messages originated by `10.0.0.1`?**

In this example, the answer is yes.

RP2 is directly peered with the originating RP, so the SA is accepted without requiring a BGP-based Peer-RPF lookup.

## MSDP Peer-RPF vs. Multicast Data-Plane RPF
Although both mechanisms use the term RPF, they operate on different types of information.

| Feature | Multicast Data-Plane RPF | MSDP Peer-RPF |
|---|---|---|
| Applies to | Multicast data packets | MSDP SA messages |
| Examines | Multicast source or RP, depending on tree type | Originating RP address in the SA |
| Checks | Incoming interface and RPF neighbor | MSDP peer that sent the SA |
| Purpose | Prevent multicast forwarding loops | Prevent SA message flooding loops |
| Failure result | Multicast packet discarded | SA message discarded |

For example:

```text
MSDP SA:
Originating RP: 10.0.0.1
Source:         192.0.2.10
Group:          239.1.1.1
```

The Peer-RPF check is performed relative to:

```text
10.0.0.1
```

It is **not performed relative to the multicast source address `192.0.2.10`**.

This is an important distinction.

The multicast source address is used later when the receiving RP performs an ordinary RPF lookup to establish an `(S,G)` source tree.

## Why Peer-RPF Is Necessary
Consider three MSDP speakers:

```text
       RP1
      /   \
     /     \
   RP2 --- RP3
```

Assume:

| Router | MSDP Address |
|---|---|
| RP1 | `10.0.0.1` |
| RP2 | `10.0.0.2` |
| RP3 | `10.0.0.3` |

All three routers have MSDP peering relationships with each other.

Suppose RP1 originates an SA message for:

```text
(192.0.2.10, 239.1.1.1)
```

RP1 sends the message directly to both RP2 and RP3.

Without appropriate loop prevention, the following could occur:

1. RP1 sends the SA to RP2 and RP3.
2. RP2 forwards the SA to RP3.
3. RP3 forwards the SA to RP2.
4. The message continues circulating between peers.

Peer-RPF prevents this behavior by selecting the appropriate peer from which SA messages originated by RP1 should be accepted.

For example, RP3 is directly peered with RP1.

Therefore:

| SA Originator | Sending Peer | RP3 Peer-RPF Result |
|---|---|---|
| RP1 | RP1 | Pass |
| RP1 | RP2 | Fail |

RP3 accepts the SA received directly from RP1 and rejects the copy forwarded by RP2.

**Peer-RPF is performed relative to the originating RP, not merely the peer that most recently forwarded the SA.**

## MSDP Peer-RPF Selection
MSDP needs to determine the correct peer from which to accept SA messages originated by a particular RP.

RFC 3618 defines an ordered set of Peer-RPF selection rules.

The most important rules are summarized below.

| Rule | Peer-RPF Selection |
|---|---|
| 1 | Accept from the originating RP itself, if directly MSDP-peered |
| 2 | Select the eBGP next hop for the route toward the originating RP |
| 3 | Select the peer associated with the route toward the originating RP, using BGP or supported routing-protocol information |
| 4 | Select a peer in the closest AS toward the originating RP, using the highest peer IP address as the tie-breaker |
| 5 | Use a statically configured RPF peer |

The rules are evaluated in order, using the first applicable rule.

Only MSDP peers in the **ESTABLISHED** state are eligible for Peer-RPF selection.

### Direct Peering with the Originating RP
The simplest case occurs when the receiving router is directly MSDP-peered with the RP that originated the SA.

For example:

```text
RP1 ---------------- RP2
10.0.0.1             10.0.0.2
```

RP1 originates:

```text
SA RP Address: 10.0.0.1
```

RP2 receives the SA from its peer:

```text
10.0.0.1
```

The originating RP address matches the sending MSDP peer address.

Therefore, RP2 accepts the message.

**No BGP session is required for this direct-originator case.**

This is particularly important for a simple two-RP Anycast RP deployment.

### Peer-RPF Using Routing Information
When the sending MSDP peer is not the originating RP, the receiving router may need routing information to determine which peer lies on the correct path toward the originator.

For example:

```text
RP1 -------- RP2 -------- RP3
```

Suppose RP1 originates an SA and RP2 forwards it to RP3:

```text
Originating RP: 10.0.0.1
Sending Peer:   10.0.0.2
```

RP3 must determine whether RP2 is the correct Peer-RPF neighbor for reaching `10.0.0.1`.

MSDP can use routing information, particularly BGP, to make this determination.

In traditional interdomain MSDP deployments, the MSDP topology is generally designed to follow the BGP topology.

However, BGP is **not inherently required for IPv4 Anycast RP**.

Direct peering exceptions and MSDP mesh groups allow Anycast RP to operate using ordinary IGP reachability.

## IOS XE Peer-RPF Exceptions
Cisco IOS XE documents several cases in which the normal BGP-based Peer-RPF check is bypassed.

| Condition | Behavior |
|---|---|
| Only one MSDP peer is configured | SA messages from that peer can be accepted without the normal Peer-RPF check |
| Sending peer is the originating RP | SA is accepted without the normal Peer-RPF check |
| Sending peer belongs to the same MSDP mesh group | SA is accepted without the normal Peer-RPF check |
| Sending peer is the selected default peer | SA is accepted without the normal Peer-RPF check |

SA filters and other MSDP validation rules still apply.

### Two-RP Anycast Deployment
For the original topology:

```text
Src --- R1 --- RP1 --- RP2 --- R4 --- Rec
```

Assume RP1 and RP2 have only one MSDP peer each.

The relevant configuration is:

**RP1:**

```text
RP1(config)# ip msdp peer 10.0.0.2 connect-source Loopback1
RP1(config)# ip msdp originator-id Loopback1
```

**RP2:**

```text
RP2(config)# ip msdp peer 10.0.0.1 connect-source Loopback1
RP2(config)# ip msdp originator-id Loopback1
```

RP1 and RP2 can exchange SA messages without BGP.

Neither `ip msdp default-peer` nor a mesh group is required for this simple two-peer arrangement.

> **Important:** Having an established MSDP session does not universally disable Peer-RPF. These exceptions apply under specific conditions. Adding more peers can change which Peer-RPF rules are relevant.

## MSDP Default Peers

A **default MSDP peer** is an explicitly selected peer from which a router accepts Source-Active (SA) messages **without performing the normal Peer-RPF check**. This is useful when the MSDP topology does not follow BGP routing information, such as when a stub multicast domain connects to one or more upstream domains.

A default peer does not replace the MSDP TCP session. The peer must already be configured using `ip msdp peer`.

Syntax:

```text
ip msdp default-peer <peer-address> [prefix-list <name>]
```

If a router has **only one configured MSDP peer**, it automatically accepts that peer as the default for Peer-RPF purposes. In that case, `ip msdp default-peer` is unnecessary.

### Configuring a Default Peer

Suppose RP3 has MSDP sessions with RP1 (`10.0.0.1`) and RP2 (`10.0.0.2`), but should use RP1 as its default peer:

```text
RP3(config)# ip msdp peer 10.0.0.1 connect-source Loopback1
RP3(config)# ip msdp peer 10.0.0.2 connect-source Loopback1
RP3(config)# ip msdp default-peer 10.0.0.1
```

RP3 can accept SA messages received from RP1 without the normal Peer-RPF check, including SAs that RP1 forwarded on behalf of other originating RPs.

**Default-peer status affects SA acceptance, not the selection of the unicast or multicast forwarding path.** SA filters and other MSDP checks still apply.

### Multiple Default Peers Without Prefix Lists

Multiple default peers can provide **primary/backup** behavior.

For example, assuming the MSDP sessions are already configured:

```text
RP3(config)# ip msdp default-peer 10.0.0.1
RP3(config)# ip msdp default-peer 10.0.0.2
```

- RP1 (`10.0.0.1`) is the active default peer while its session is available because it was configured first.
- RP2 (`10.0.0.2`) becomes the active default peer if RP1 or its connectivity fails.
- When RP1 is active, RP2 is not treated as an active *default peer*. Other normal Peer-RPF acceptance rules can still apply to SAs received from RP2.

**Without prefix lists, only one configured default peer is active at a time.**

### Multiple Default Peers With Prefix Lists

The optional `prefix-list` restricts default-peer treatment according to the **originating RP address carried in the SA message**.

It does **not** match the multicast source address or multicast group address.

When prefix lists are associated with multiple default peers, those peers can be **active simultaneously for different originating RP prefixes**.

For example, RP3 receives SAs through two upstream MSDP peers:

- RP1 (`10.0.0.1`) forwards SAs originated by `10.10.10.10`.
- RP2 (`10.0.0.2`) forwards SAs originated by `10.20.20.20`.

Configure RP3:

```text
RP3(config)# ip prefix-list RP1-ORIGINATORS permit 10.10.10.10/32
RP3(config)# ip prefix-list RP2-ORIGINATORS permit 10.20.20.20/32
!
RP3(config)# ip msdp default-peer 10.0.0.1 prefix-list RP1-ORIGINATORS
RP3(config)# ip msdp default-peer 10.0.0.2 prefix-list RP2-ORIGINATORS
```

Assuming both MSDP sessions are established:

| SA Originating RP | Sending MSDP Peer | Default-Peer Acceptance |
|---|---|---|
| `10.10.10.10` | `10.0.0.1` | Accepted without normal Peer-RPF |
| `10.20.20.20` | `10.0.0.2` | Accepted without normal Peer-RPF |
| `10.20.20.20` | `10.0.0.1` | Not covered by RP1's default-peer prefix list |

The third example does not automatically mean the SA is rejected. It means the **default-peer exception does not apply**; normal Peer-RPF rules and other checks determine acceptance.

### Default Peers vs. Mesh Groups

Both features can bypass normal Peer-RPF checks, but they serve different purposes:

| Feature | Typical Use | Key Behavior |
|---|---|---|
| Default peers | Stub or hub-and-spoke MSDP connectivity without BGP-based Peer-RPF | Accept SAs from selected peers, optionally restricted by originating RP prefix |
| Mesh groups | Fully meshed MSDP speakers, especially Anycast RPs | Accept SAs from mesh-group members without normal Peer-RPF and suppress forwarding those SAs to other group members |

Unlike mesh groups, **default peers do not introduce mesh-group SA forwarding suppression**.

For multiple fully meshed Anycast RPs, mesh groups are generally the better choice.

---

## MSDP Mesh Groups
An **MSDP Mesh Group** is a group of MSDP speakers that maintain a full mesh of MSDP peering relationships.

Every member must have an MSDP peering session with every other member.

Mesh groups provide two important benefits:

1. **Peer-RPF simplification:** SA messages received from members of the same mesh group are accepted without normal Peer-RPF checks.
2. **Reduced SA flooding:** SA messages received from a mesh-group member are not forwarded to other members of that mesh group.

Mesh groups are particularly useful when multiple Anycast RPs operate within the same multicast domain.

### Full-Mesh Requirement
Consider three RPs:

```text
          RP1
         /   \\
        /     \\
       /       \\
      RP2 ----- RP3
```

All three MSDP peering sessions must exist:

```text
RP1 <--> RP2
RP1 <--> RP3
RP2 <--> RP3
```

Each RP must also have the other two peers configured as members of the mesh group.

**An MSDP Mesh Group does not establish TCP peering sessions automatically.**

The peering sessions must first be configured using `ip msdp peer`.

The `ip msdp mesh-group` command then identifies which existing MSDP peers belong to the group.

### SA Reception from a Mesh-Group Member
Suppose RP1 originates an SA message.

Because RP1 is peered with both RP2 and RP3, it sends the SA directly to each:

```text
RP1 ---- SA ----> RP2
  \\
   \----- SA ---> RP3
```

RP2 and RP3 accept the message without performing the normal Peer-RPF check against RP1.

The critical forwarding rule is:

**An SA received from a member of the same mesh group is NOT forwarded to other members of that mesh group.**

Therefore:

- RP2 does not forward RP1's SA to RP3.
- RP3 does not forward RP1's SA to RP2.

There is no need for that additional flooding because RP1 already sent the advertisement to both routers.

### Mesh-Group Forwarding Rules
RFC 3618 defines two important cases.

**Case 1: SA Received from a Peer in the Same Mesh Group**

When an MSDP speaker receives an SA from a peer belonging to the same mesh group:

- The SA bypasses the normal Peer-RPF check.
- The SA is not forwarded to other peers in that mesh group.
- The SA may be forwarded to peers outside the mesh group, subject to the normal forwarding and filtering rules.

For example:

```text
       Mesh Group
   +----------------+
   |                |
   |  RP1 --- RP2   |
   |   \     /      |
   |    \   /       |
   |     RP3        |
   |                |
   +----------------+
            |
           RP4
     (Outside Group)
```

If RP2 receives an SA from RP1:

- RP2 does not forward it to RP3.
- RP2 may forward it to RP4, which is outside the mesh group.

**Case 2: SA Received from a Peer Outside the Mesh Group**

When an MSDP speaker receives an SA from a peer outside its mesh group:

1. The normal Peer-RPF check is performed.
2. If the check succeeds, the SA can be forwarded to members of the mesh group.
3. Other members that receive the SA do not forward it again within the mesh group.

For example, suppose RP4 is outside the mesh group and sends an SA to RP2:

```text
RP4 ---- SA ----> RP2
```

If the SA passes RP2's Peer-RPF check, RP2 can forward the SA to RP1 and RP3:

```text
          RP1
           ^
           |
          RP2 <---- RP4
           |
           v
          RP3
```

RP1 and RP3 accept the SA from RP2 because RP2 belongs to their mesh group.

However, neither forwards that SA to the other mesh-group members.

### Mesh-Group Forwarding Summary
| SA Received From | Peer-RPF Check | Forward to Other Mesh Members? | Forward Outside Mesh Group? |
|---|---|---|---|
| Member of same mesh group | Bypassed | No | Yes, if permitted |
| Peer outside mesh group | Required | Yes, if accepted | Yes, if permitted |

**Mesh groups control SA reception and flooding. They do not synchronize PIM multicast routing tables or replace normal multicast data-plane RPF checks.**

## IOS XE Mesh-Group Configuration
The IOS XE command syntax is:

```text
ip msdp mesh-group <name> <peer-address>
```

| Parameter | Purpose |
|---|---|
| `name` | Name identifying the local MSDP mesh group |
| `peer-address` | MSDP peer to add to that mesh group |

The command is configured separately for each peer belonging to the group.

### Two-RP Mesh Group
A mesh group can be configured with only RP1 and RP2, although it is not necessary for basic two-RP Anycast operation.

On RP1:

```text
RP1(config)# ip msdp mesh-group ANYCAST-RPS 10.0.0.2
```

On RP2:

```text
RP2(config)# ip msdp mesh-group ANYCAST-RPS 10.0.0.1
```

These commands place the two MSDP peers in the same logical mesh group.

SA messages exchanged between them bypass normal Peer-RPF checking.

Because there are only two members, the rule preventing forwarding to other members has little practical effect.

### Three-RP Mesh Group
Now extend the Anycast RP design to include RP3:

| RP | Anycast Address | Unique MSDP Address |
|---|---|---|
| RP1 | `10.0.0.100` | `10.0.0.1` |
| RP2 | `10.0.0.100` | `10.0.0.2` |
| RP3 | `10.0.0.100` | `10.0.0.3` |

All three RPs advertise the shared Anycast RP address through unicast routing.

Each RP uses its unique MSDP address for peering and as its SA Originator ID.

**RP1 Configuration**

```text
RP1(config)# ip msdp peer 10.0.0.2 connect-source Loopback1
RP1(config)# ip msdp peer 10.0.0.3 connect-source Loopback1
RP1(config)# ip msdp originator-id Loopback1
RP1(config)# ip msdp mesh-group ANYCAST-RPS 10.0.0.2
RP1(config)# ip msdp mesh-group ANYCAST-RPS 10.0.0.3
```

**RP2 Configuration**

```text
RP2(config)# ip msdp peer 10.0.0.1 connect-source Loopback1
RP2(config)# ip msdp peer 10.0.0.3 connect-source Loopback1
RP2(config)# ip msdp originator-id Loopback1
RP2(config)# ip msdp mesh-group ANYCAST-RPS 10.0.0.1
RP2(config)# ip msdp mesh-group ANYCAST-RPS 10.0.0.3
```

**RP3 Configuration**

```text
RP3(config)# ip msdp peer 10.0.0.1 connect-source Loopback1
RP3(config)# ip msdp peer 10.0.0.2 connect-source Loopback1
RP3(config)# ip msdp originator-id Loopback1
RP3(config)# ip msdp mesh-group ANYCAST-RPS 10.0.0.1
RP3(config)# ip msdp mesh-group ANYCAST-RPS 10.0.0.2
```

All three RPs are now fully meshed.

Each RP:

- Has two MSDP peering sessions.
- Has both remote RPs configured as mesh-group members.
- Uses its unique Loopback1 address for SA origination.
- Accepts SA messages from its mesh-group peers without normal Peer-RPF checks.
- Does not forward SAs received from one mesh-group member to another member.

### Remove a Peer from a Mesh Group
For example, remove RP3 from RP1's mesh group:

```text
RP1(config)# no ip msdp mesh-group ANYCAST-RPS 10.0.0.3
```

This removes the mesh-group membership on RP1.

It does not remove the underlying MSDP peer configuration.

To remove the peering session entirely:

```text
RP1(config)# no ip msdp peer 10.0.0.3
```

Mesh-group membership and MSDP peering are separate configuration elements.

## Mesh-Group Design Considerations
### Full-Mesh Connectivity Is Required
Every mesh-group member must have MSDP peering with every other member.

For example, the following is not a complete mesh:

```text
RP1 -------- RP2 -------- RP3
```

RP1 and RP3 have no direct MSDP peering session.

Configuring all three routers as members of a single mesh group would be incorrect because SA messages received from another member are not forwarded to the remaining mesh-group members.

For example:

1. RP1 originates an SA and sends it to RP2.
2. RP2 accepts the SA.
3. RP2 does not forward it to RP3 because RP3 belongs to the same mesh group.
4. RP3 never receives the advertisement from RP1.

**An incomplete mesh can prevent multicast sources from being discovered by all RPs.**

### Mesh Groups Do Not Replace MSDP Peering
The command:

```text
ip msdp mesh-group ANYCAST-RPS 10.0.0.2
```

does not establish a peering session.

The corresponding peer configuration is still required:

```text
ip msdp peer 10.0.0.2 connect-source Loopback1
```

### Mesh Groups Do Not Bypass SA Filters
Mesh-group membership bypasses the normal Peer-RPF check.

It does not bypass configured SA filtering policies.

For example:

```text
RP2(config)# ip msdp sa-filter in 10.0.0.1
```

This blocks all incoming SA messages from RP1, even if RP1 belongs to the same mesh group.

### Mesh Groups Do Not Replace Unicast Routing
Each member still needs unicast reachability to the other members' unique MSDP addresses.

The shared Anycast RP address must also remain reachable throughout the multicast domain.

These are separate requirements.

## RFC 3618 Peer-RPF Compliance
IOS XE supports enabling the Peer-RPF forwarding rules defined by RFC 3618.

Configure:

```text
RP1(config)# ip msdp rpf rfc3618
```

This enables the RFC 3618 Peer-RPF selection rules.

An important benefit is support for using IGP routing information, including OSPF and IS-IS, when determining the appropriate MSDP Peer-RPF neighbor.

This can allow enterprise MSDP topologies to operate without BGP.

For example, when the route toward an originating RP is learned through OSPF, the RFC 3618 rules can use the IGP next hop to help identify the appropriate MSDP RPF peer.

However, this feature is not normally required for a simple two-RP Anycast deployment or a fully meshed Anycast RP group.

### Verify the Selected RPF Peer
IOS XE provides the following command:

```text
show ip msdp rpf-peer <rp-address>
```

For example, on RP3:

```text
RP3# show ip msdp rpf-peer 10.0.0.1
```

This displays the MSDP peer selected for accepting SA messages originated by RP1.

Use the **unique originating RP address**, not the shared Anycast RP address.

The command is particularly useful when troubleshooting Peer-RPF selection in deployments using multiple MSDP peers without mesh-group exceptions.

## IOS XE Verification
### Verify MSDP Peer Status
```text
RP1# show ip msdp summary
```

Confirm that all required MSDP sessions are established.

For the three-RP mesh group, RP1 should have active sessions with:

```text
10.0.0.2
10.0.0.3
```

Repeat the verification on RP2 and RP3.

### Verify Mesh-Group Membership
Check the running configuration:

```text
RP1# show running-config | include ip msdp mesh-group
```

Expected configuration:

```text
ip msdp mesh-group ANYCAST-RPS 10.0.0.2
ip msdp mesh-group ANYCAST-RPS 10.0.0.3
```

Verify that every RP lists all other members.

Also examine detailed peer information:

```text
RP1# show ip msdp peer 10.0.0.2
RP1# show ip msdp peer 10.0.0.3
```

Check the peer status and any displayed mesh-group information.

### Verify SA Reception
On RP2:

```text
RP2# show ip msdp sa-cache
```

If RP1 has locally registered source `192.0.2.10` for `239.1.1.1`, the SA cache should contain an entry identifying:

```text
Source:         192.0.2.10
Group:          239.1.1.1
Originating RP: 10.0.0.1
Peer:           10.0.0.1
```

In a mesh group, each RP should learn active sources originated by the other RPs.

### Verify SA Counters
```text
RP1# show ip msdp count
```

This displays SA state counters associated with MSDP peers.

Compare the counters on the different RPs to confirm that source advertisements are reaching the expected peers.

### Verify Multicast Forwarding
Finally, check PIM multicast routing state:

```text
RP2# show ip mroute 239.1.1.1
```

If RP2 has interested receivers for the group, it should be able to establish an `(S,G)` source tree toward the advertised source.

Remember that **successful SA reception does not guarantee successful multicast data forwarding**.

The source tree also requires valid PIM state and data-plane RPF.

## Troubleshooting Peer-RPF and Mesh Groups
| Problem | Possible Cause |
|---|---|
| MSDP peer established but SA rejected | Peer-RPF failure or SA filtering |
| SA accepted from one peer but rejected from another | Only one peer is selected as the valid Peer-RPF neighbor |
| SA Originator ID is the shared Anycast RP address | Incorrect or missing `ip msdp originator-id` |
| Some mesh-group members do not learn a source | Incomplete MSDP full mesh |
| Mesh-group SA messages are unexpectedly filtered | Incoming or outgoing SA filter |
| SA information is duplicated across peers | Missing or incorrect mesh-group configuration |
| Peer-RPF fails without BGP | No applicable Peer-RPF exception or routing information |
| MSDP session down | Unicast connectivity, TCP 639, authentication, or peer configuration problem |
| SA cache contains source but multicast fails | PIM Join, data-plane RPF, or multicast forwarding problem |

### Troubleshooting Sequence
A useful troubleshooting sequence is:

1. Verify the MSDP TCP peering sessions using `show ip msdp summary`.
2. Verify the unique Originator ID on each Anycast RP.
3. Check whether the peers belong to the same mesh group.
4. If they do not, examine the selected Peer-RPF neighbor using `show ip msdp rpf-peer`.
5. Check incoming and outgoing SA filters.
6. Examine the SA cache using `show ip msdp sa-cache`.
7. Verify PIM `(S,G)` forwarding state using `show ip mroute`.

The most important distinction is between:

- **MSDP peering:** TCP connectivity between MSDP speakers.
- **MSDP Peer-RPF:** Acceptance of SA messages from the correct peer.
- **PIM RPF:** Forwarding multicast traffic along the appropriate source or RP tree.

These are separate operations.

## Key Points
- **MSDP Peer-RPF** prevents SA message flooding loops.
- Peer-RPF evaluates the **originating RP address carried in the SA**, not the multicast source address.
- Normal Peer-RPF selection identifies the correct MSDP peer from which to accept SAs originated by a particular RP.
- An SA received directly from its originating RP can be accepted without a BGP-based Peer-RPF lookup.
- IOS XE also bypasses normal Peer-RPF in certain cases, including single-peer and mesh-group deployments.
- **MSDP Mesh Groups require a full mesh of MSDP peering sessions.**
- SA messages received from a peer in the same mesh group **bypass normal Peer-RPF checks**.
- SA messages received from a mesh-group member are **not forwarded to other members of that mesh group**.
- SA messages received from outside the mesh group must pass normal Peer-RPF before being forwarded into the group.
- **`ip msdp mesh-group <name> <peer-address>`** assigns a configured peer to a mesh group.
- Mesh-group configuration does not automatically establish MSDP TCP sessions.
- Mesh-group membership does not bypass configured SA filters or normal multicast data-plane RPF.
- **`ip msdp default-peer`** accepts SAs from a selected peer without normal Peer-RPF checks.
- Without prefix lists, multiple default peers provide primary/backup behavior; with prefix lists, multiple peers can be active for different originating RP prefixes.
- The default-peer prefix list matches the **originating RP address** in the SA message, not the multicast source or group.
- **`ip msdp rpf rfc3618`** enables RFC 3618 Peer-RPF rules, including support for IGP-based selection.
- Use `show ip msdp rpf-peer <rp-address>` to examine the selected MSDP RPF peer.
- For a basic two-RP Anycast deployment, a mesh group is optional. For deployments with three or more RPs, a full-mesh MSDP configuration with mesh groups is generally preferred.

## References
- [RFC 3618, Section 10: MSDP Peer-RPF Forwarding](https://www.rfc-editor.org/rfc/rfc3618.html#section-10)
- [RFC 3618, Section 10.2: MSDP Mesh-Group Semantics](https://www.rfc-editor.org/rfc/rfc3618.html#section-10.2)
- [RFC 3446: Anycast RP Mechanism Using PIM and MSDP](https://www.rfc-editor.org/rfc/rfc3446)
- [Cisco IOS XE: Using MSDP to Interconnect Multiple PIM-SM Domains](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/ip-multicast/ip-multicast/m_imc_msdp_im_pim_sim.html)
