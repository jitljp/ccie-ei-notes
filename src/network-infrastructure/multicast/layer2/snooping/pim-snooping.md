# PIM Snooping

**PIM Snooping** allows a Layer 2 switch to inspect PIM control messages exchanged between multicast routers and selectively forward multicast traffic only to the router-facing ports that need it.

It is especially useful when a Layer 2 switch interconnects **multiple multicast routers** on a shared VLAN.

## Example Topology

The examples on this page use the following topology:

```text
                         SW1
                  /       |       \
               E0/1     E0/2     E0/3
                 |         |         |
                R1        R2        R3
                 |         |         |
                SRC       PC2       PC3
```

On SW1:

```text
E0/1 → R1
E0/2 → R2
E0/3 → R3
```

R1, R2, and R3 are PIM routers on the same shared Layer 2 segment.

`SRC` is a multicast source on a separate network behind R1.

PC2 and PC3 are on separate receiver-facing networks behind R2 and R3. Depending on the example, either, both, or neither may be a receiver for multicast group `G`.

For examples that require a specific upstream direction, assume R1 is the upstream PIM neighbor toward the source or RP.

## Why IGMP Snooping Alone Is Not Enough

IGMP snooping can identify:

```text
listener ports
mrouter ports
```

but ordinary IGMP snooping does not know which multicast routers actually need a particular multicast group.

R1, R2, and R3 exchange PIM control traffic on the shared VLAN, so SW1 can identify:

```text
E0/1 → mrouter port
E0/2 → mrouter port
E0/3 → mrouter port
```

Suppose only PC2 is a receiver for:

```text
G = 239.1.1.1
```

R2 needs multicast traffic for `G`, but R1 and R3 may not.

Without PIM snooping, SW1 knows only that all three ports lead toward multicast routers.

If multicast traffic for `G` arrives from R1 on E0/1, ordinary IGMP snooping can therefore forward it toward the other mrouter ports:

```text
SRC
 |
 v
R1
 |
 | G
 v
SW1
├── E0/2 → R2
└── E0/3 → R3
```

even though only R2 has a downstream receiver.

## What PIM Snooping Adds

PIM snooping observes PIM control traffic such as:

```text
PIM Hello
PIM Join/Prune
Bidirectional PIM DF election messages
```

and builds per-VLAN information about:

```text
PIM neighbors
PIM DR
upstream PIM neighbor
downstream router interest
PIM snooping mroute state
Bidirectional PIM Designated Forwarders
```

This lets SW1 distinguish:

```text
R2 → needs G
R1 → does not need G
R3 → does not need G
```

instead of treating every mrouter port identically.

## PIM Snooping Does Not Make SW1 a PIM Router

SW1 remains a Layer 2 switch.

It does **not** need to:

```text
run PIM on an SVI
form PIM adjacencies
perform RPF lookups
participate in multicast routing
```

Instead, it passively inspects PIM messages exchanged by R1, R2, and R3 and uses that information to build Layer 2 forwarding state.

## PIM Hello Snooping

PIM routers periodically send PIM Hello messages to:

```text
224.0.0.13
```

the All-PIM-Routers address.

SW1 sees:

```text
R1 → PIM Hello → E0/1
R2 → PIM Hello → E0/2
R3 → PIM Hello → E0/3
```

and can build a neighbor-to-port mapping:

```text
R1 → E0/1
R2 → E0/2
R3 → E0/3
```

PIM Hello information also allows SW1 to track information such as:

```text
neighbor lifetime
PIM DR
Bidirectional PIM capability
```

The switch observes the PIM neighbor relationships; it does not become a PIM neighbor itself.

## PIM Join/Prune Snooping

PIM snooping also inspects **PIM Join/Prune messages**.

A PIM Join/Prune is sent to:

```text
224.0.0.13
```

but the packet contains an **Upstream Neighbor Address** identifying the PIM router toward which the Join/Prune is directed.

For example, suppose PC2 is a receiver for `G`:

```text
PC2
 |
 | IGMP membership
 v
R2
```

R2 may need to send a PIM Join toward its RPF neighbor for the RP or source.

Assume, for this example, that R1 is R2's upstream PIM neighbor.

R2 sends:

```text
PIM Join/Prune

Destination:
224.0.0.13

Upstream Neighbor:
R1

Joined:
(*,G)
```

SW1 already knows from PIM Hellos that:

```text
R1 → E0/1
```

so it can forward the Join/Prune only toward E0/1.

## Join/Prune Forwarding Without PIM Snooping

Without PIM snooping, the multicast Join/Prune can be flooded to the other router-facing ports:

```text
                 R1
                  ^
                  |
                  |
R2 ---- Join ---> SW1 ----> R3
```

R1 is the intended upstream neighbor, but R3 unnecessarily receives the control packet.

## Join/Prune Forwarding With PIM Snooping

With PIM snooping:

```text
R2
 |
 | PIM Join
 v
SW1
 |
 | E0/1
 v
R1
```

SW1 forwards the message only toward the router identified by the **Upstream Neighbor Address**.

This reduces unnecessary PIM control-plane flooding on the shared Layer 2 segment.

## Learning Downstream Interest

The same Join also tells SW1 that R2 has downstream interest in `G`.

Conceptually, SW1 can learn:

```text
(*,G)

Upstream:
E0/1 → R1

Downstream:
E0/2 → R2
```

If PC3 also becomes a receiver and R3 sends a Join toward the same upstream router:

```text
(*,G)

Upstream:
E0/1 → R1

Downstream:
E0/2 → R2
E0/3 → R3
```

Cisco uses the following terminology in PIM snooping mroute output:

```text
Downstream ports
→ ports on which PIM Joins were received

Upstream ports
→ ports toward the Join/Prune upstream neighbor

Outgoing ports
→ ports programmed for forwarding
```

## Multicast Data Forwarding Without PIM Snooping

Suppose:

```text
PC2 → listening to G
PC3 → not listening
```

SRC sends multicast traffic to R1, and R1 forwards that traffic onto the shared VLAN toward downstream PIM routers.

Without PIM snooping:

```text
SRC
 |
 v
R1
 |
 | G
 v
SW1
├── E0/2 → R2
└── E0/3 → R3
```

because all three are mrouter ports.

## Multicast Data Forwarding With PIM Snooping

With PIM snooping, SW1 can use the snooped PIM state to avoid forwarding `G` toward routers with no downstream interest.

Ignoring the DR-flooding exception for a moment:

```text
SRC
 |
 v
R1
 |
 | G
 v
SW1
└── E0/2 → R2
```

R2 receives the traffic because PC2 is downstream and R2 expressed PIM interest.

R3 can be pruned because it has no relevant PIM state. R1 is the ingress router for this multicast traffic, not a downstream forwarding destination on the SW1 VLAN.

The core distinction is:

```text
IGMP snooping
→ Which host-facing ports need G?

PIM snooping
→ Which router-facing ports need G?
```

## PIM Prunes

Suppose PC2 and PC3 are both receivers.

SW1 has learned:

```text
G
→ E0/2 → R2
→ E0/3 → R3
```

PC2 then leaves.

After R2 no longer has downstream interest, R2 sends a PIM Prune toward its upstream neighbor.

SW1 snoops the Prune and updates its state:

```text
Before:

G
→ R2
→ R3


After R2 Prune:

G
→ R3
```

R2 remains a multicast router, but SW1 no longer needs to forward `G` toward it.

This is the key improvement over basic mrouter-port forwarding.

## PIM Snooping Mroute State

PIM snooping creates multicast-route-like state inside the Layer 2 switch.

For example:

```text
(*,239.1.1.1)

Upstream:
E0/1 → R1

Downstream:
E0/2 → R2
E0/3 → R3

Outgoing:
E0/1
E0/2
E0/3
```

This is a **PIM snooping mroute**, not SW1's Layer 3 multicast routing table.

SW1 is using PIM signaling to decide which Layer 2 ports belong in the forwarding state.

## `(S,G)` State

The PIM snooping database can also display `(S,G)` entries.

However, Catalyst PIM snooping has an important restriction:

```text
(S,G) snooping mroutes
→ processed as (*,G) mroutes
```

So the presence of `(S,G)` state in PIM snooping output does not mean the switch performs fully source-specific Layer 2 forwarding equivalent to a PIM router.

## DR Flooding

Catalyst PIM snooping can also forward multicast traffic to the **PIM DR on the snooped VLAN**, even if that router has not expressed downstream interest in the group.

This behavior is controlled globally with:

```text
SW1(config)# ip pim snooping dr-flood
```

and is enabled by default.

Disable it with:

```text
SW1(config)# no ip pim snooping dr-flood
```

For example, suppose:

```text
PC2 → listening to G
PC3 → not listening
R3  → DR on the SW1 VLAN
```

R2 has downstream interest in `G`, while R3 does not.

Without considering DR flooding, the desired downstream forwarding is:

```text
SRC
 |
 v
R1
 |
 | G
 v
SW1
└── E0/2 → R2
```

With default DR flooding, SW1 can additionally forward the traffic to the VLAN DR:

```text
SRC
 |
 v
R1
 |
 | G
 v
SW1
├── E0/2 → R2   downstream interest
└── E0/3 → R3   PIM DR
```

Therefore:

```text
PIM Join/Prune state
→ identifies router ports with downstream interest

DR flooding
→ can additionally include the VLAN DR
```

Cisco documents the DR as being automatically programmed into the `(S,G)` outgoing-interface list when DR flooding is enabled.

The source-side role of R1 is a separate matter: R1 receives packets directly from SRC on its source-facing interface and performs any required source-side PIM DR functions there.

## Example: PC2 Only

Assume:

```text
PC2 → listening to G
PC3 → not listening
```

and R1 is R2's upstream PIM neighbor toward the source or RP.

R2 sends a Join toward R1.

SW1 learns:

```text
Upstream:
E0/1 → R1

Downstream:
E0/2 → R2
```

Ignoring DR flooding, data forwarding is:

```text
SRC
 |
 v
R1
 |
 | G
 v
SW1
└── E0/2 → R2
```

R3 does not receive `G`.

If R3 happens to be the PIM DR on the SW1 VLAN, default DR flooding can additionally include E0/3.

## Example: PC2 and PC3

Now PC3 also becomes a receiver for `G`.

R3 sends the appropriate PIM Join toward its upstream neighbor.

SW1 can learn:

```text
Upstream:
E0/1 → R1

Downstream:
E0/2 → R2
E0/3 → R3
```

and forward:

```text
SRC
 |
 v
R1
 |
 | G
 v
SW1
├── E0/2 → R2
└── E0/3 → R3
```

If PC2 later leaves and R2 prunes:

```text
SRC
 |
 v
R1
 |
 | G
 v
SW1
└── E0/3 → R3
```

R2 remains a PIM router, but it no longer receives `G`.

## PIM DR vs IGMP Querier

Do not confuse the **PIM DR** with the **IGMP querier**.

On the shared R1/R2/R3 VLAN:

```text
PIM DR
→ elected among PIM routers
→ performs PIM DR responsibilities
```

The attached networks are separate:

```text
R1 ↔ SRC
R2 ↔ PC2
R3 ↔ PC3
```

R2 and R3 can independently act as IGMP queriers on their receiver-facing LANs.

The PIM DR election on the shared SW1 segment is unrelated to the IGMP querier elections on the PC2 and PC3 LANs.

## Bidirectional PIM

PIM snooping can also inspect **Bidirectional PIM Designated Forwarder (DF) election messages**.

For each RP, SW1 can learn which router is the DF on the shared VLAN.

Conceptually:

```text
RP1
→ R2 is DF

RP2
→ R3 is DF
```

The switch can maintain DF state per VLAN/RP and forward Bidirectional PIM traffic toward the required DFs.

## Auto-RP Groups

Two Auto-RP groups receive special treatment:

```text
224.0.1.39
224.0.1.40
```

They are used for:

```text
224.0.1.39
→ Auto-RP RP-Announce

224.0.1.40
→ Auto-RP RP-Discovery
```

On a PIM snooping-enabled VLAN, these groups are flooded to **all PIM-router ports**.

In this topology:

```text
224.0.1.39 / 224.0.1.40
→ E0/1 → R1
→ E0/2 → R2
→ E0/3 → R3
```

rather than being pruned according to normal per-group PIM snooping state.

## State Aging

PIM snooping state is dynamic.

SW1 uses timers carried in PIM control messages:

```text
PIM Hello Holdtime
→ neighbor lifetime

PIM Join/Prune Holdtime
→ multicast state lifetime
```

For example:

```text
R2 Join for G
      ↓
SW1 learns E0/2 as downstream
      ↓
state periodically refreshed
```

If the state is not refreshed:

```text
holdtime expires
      ↓
E0/2 downstream state removed
```

PIM neighbor and mroute state is maintained independently per VLAN.

## Interaction with IGMP Snooping

IGMP snooping and PIM snooping perform complementary jobs.

```text
IGMP Snooping
→ constrains receiver-facing Layer 2 forwarding

PIM Snooping
→ constrains PIM-router-facing Layer 2 forwarding
```

PIM snooping is **not** a replacement for IGMP snooping.

## PIMv2 Requirement

PIM snooping understands **PIMv2 signaling**.

Current Catalyst documentation states that non-PIMv2 multicast routers do not receive multicast traffic when PIM snooping is enabled.

Therefore, participating multicast routers should support PIMv2.

## IPv4 Only

Current Catalyst PIM snooping applies to **IPv4 multicast**.

It does not support IPv6 multicast.

Cisco:

```
PIM snooping is supported only on IPv4 mroutes.
```

## Configuration

PIM snooping is disabled by default.

Enable it globally:

```text
SW1(config)# ip pim snooping
```

It must be enabled BOTH globally and per VLAN to operate. To enable it on a VLAN:

```text
SW1(config)# ip pim snooping vlan 10
```

> I can't verify operation with my current hardware, so I'm unsure if enabling globally automatically enables it per-VLAN or not.

> TO BE DETERMINED

## Verification

> TO BE DETERMINED

## Verify PIM Neighbors

> TO BE DETERMINED

## Verify PIM Snooping Mroute State

> TO BE DETERMINED

## Useful Verification Commands

```text
show ip pim snooping
show ip pim snooping detail

show ip pim snooping vlan 10
show ip pim snooping vlan 10 detail

show ip pim snooping vlan 10 neighbor
show ip pim snooping vlan 10 mroute
show ip pim snooping vlan 10 statistics
```

Dynamic state can be cleared with:

```text
SW1# clear ip pim snooping vlan 10
```

The switch must then relearn the PIM snooping state.

## PIM Snooping vs IGMP Snooping

| Feature | IGMP Snooping | PIM Snooping |
|---|---|---|
| Observes | IGMP | PIM |
| Primary endpoints | Hosts / multicast routers | PIM routers |
| Learns listener ports | Yes | No |
| Learns PIM neighbors | No | Yes |
| Parses PIM Join/Prune | No | Yes |
| Restricts host-facing multicast | Yes | No |
| Restricts mrouter-facing multicast | Normally no | Yes |
| Tracks PIM DR/DF | No | Yes |
| Performs multicast routing | No | No |

The core distinction is:

```text
IGMP Snooping
→ Which host-facing ports need G?

PIM Snooping
→ Which router-facing ports need G?
```

## Key Points

```text
Without PIM snooping
→ multicast is forwarded to all mrouter ports

With PIM snooping
→ router-facing forwarding can be restricted according to PIM interest

PIM Hello
→ learns PIM neighbors
→ R1 = E0/1
→ R2 = E0/2
→ R3 = E0/3
→ identifies the PIM DR


PIM Join/Prune
→ contains Upstream Neighbor
→ switch forwards it only toward the corresponding router port

Downstream port
→ port on which a PIM Join was received

Upstream port
→ port toward the Join/Prune upstream neighbor

Outgoing ports
→ forwarding ports programmed from snooped state

DR flooding
→ enabled by default
→ source traffic is also sent to the PIM DR

SW1 VLAN DR
→ separate election among R1/R2/R3
→ affected by dr-flood

Auto-RP:
224.0.1.39
224.0.1.40
→ flooded to all PIM-router ports

PIM snooping
→ per VLAN
→ PIMv2 / IPv4

Configuration:

ip pim snooping
ip pim snooping vlan 10


Verification:

show ip pim snooping
show ip pim snooping vlan 10
show ip pim snooping vlan 10 neighbor
show ip pim snooping vlan 10 mroute
```
