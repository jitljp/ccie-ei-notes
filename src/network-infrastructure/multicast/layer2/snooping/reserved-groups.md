# Reserved Multicast Groups

The IPv4 multicast range:

```text
224.0.0.0/24
```

is the **Reserved Link-Local** or **Local Network Control** multicast range.

These addresses are used by routing protocols and other network-control protocols on the local Layer 2 segment. Cisco identifies `224.0.0.0` through `224.0.0.255` as reserved for routing protocols and other control traffic.

Examples include:

```text
224.0.0.1    All systems on this subnet
224.0.0.2    All routers on this subnet
224.0.0.5    OSPF AllSPFRouters
224.0.0.6    OSPF AllDRouters
224.0.0.9    RIPv2 routers
224.0.0.10   EIGRP routers
224.0.0.13   PIM routers
224.0.0.18   VRRP
224.0.0.22   IGMPv3 Membership Reports
```

## Link-Local Scope

Traffic sent to:

```text
224.0.0.0/24
```

is intended to remain on the local network segment.

IP routers do **not** forward these packets between subnets. They are also commonly transmitted with:

```text
TTL = 1
```

For example:

```text
       VLAN 10                 VLAN 20

R1 ---------------- Hosts
 |
 | OSPF 224.0.0.5
 |
SW1
 |
R2
```

OSPF packets sent to `224.0.0.5` can reach OSPF routers on the local VLAN, but R1 does not route those packets into another subnet.

This is different from addresses beginning at:

```text
224.0.1.0
```

which are not part of the link-local control block and can be forwarded by multicast routers when appropriate.

## Why IGMP Snooping Treats These Groups Differently

Normal IGMP snooping assumes that multicast receivers signal their interest using IGMP.

For example:

```text
Receiver
   |
   | IGMP Report for 239.1.1.1
   v
  SW1

239.1.1.1
→ listener port
```

The switch can then prune `239.1.1.1` from ports where no receiver exists.

Network-control multicast is different.

Consider OSPF:

```text
224.0.0.5
```

An OSPF router needs to receive OSPF packets because it is participating in OSPF, not because it behaves like an ordinary multicast application and sends an IGMP Membership Report for `224.0.0.5`.

If IGMP snooping required ordinary receiver membership before forwarding this traffic:

```text
no IGMP Report
      ↓
no listener port
      ↓
prune 224.0.0.5
```

critical control protocols could break.

Therefore, multicast traffic in the reserved link-local range receives **special forwarding treatment** rather than ordinary IGMP-snooping listener pruning.

## Reserved Groups Are Normally Flooded Locally

IGMP snooping floods multicast groups in `224.0.0.0/24` to the forwarding ports of the VLAN instead of constraining them according to normal IGMP listener state.

Conceptually:

```text
                 SW1
           /      |      \
         E0/1    E0/2    E0/3
          |       |       |
         R1      R2      PC1
```

Suppose R1 sends:

```text
OSPF Hello
Destination = 224.0.0.5
```

The switch does not require an ordinary snooping listener entry such as:

```text
224.0.0.5
→ E0/2
```

before delivering the control multicast across the VLAN.

Cisco Catalyst documentation describes the reserved `224.0.0.0/24` groups as routing-control groups that are flooded to VLAN forwarding ports rather than pruned using ordinary IGMP membership.

## Reserved Multicast Is Not the Same as Unknown Multicast

Both can result in flooding, but for different reasons.

### Unknown Multicast

```text
239.1.1.1
```

might be flooded because:

```text
no snooping entry exists for G
```

Once the switch learns:

```text
239.1.1.1
→ E0/1
```

the traffic can be selectively forwarded.

### Reserved Link-Local Multicast

```text
224.0.0.5
```

is treated specially because it belongs to the network-control multicast range.

The distinction is:

```text
Unknown multicast
→ flooded because G is not known yet

224.0.0.0/24 control multicast
→ special local forwarding behavior
→ not ordinary listener-based snooping traffic
```

So `224.0.0.5` should not simply be thought of as:

```text
"an unknown multicast group"
```

## IGMP Itself Uses Reserved Groups

IGMP control messages also use addresses from this range.

### IGMP General Query

An IGMP General Query is sent to:

```text
224.0.0.1
```

which means:

```text
All Systems on this Subnet
```

The Query must reach the hosts on the VLAN so that they can report their memberships.

### IGMPv2 Leave

An IGMPv2 Leave is sent to:

```text
224.0.0.2
```

the All-Routers address.

The snooping switch processes the Leave and forwards it appropriately toward multicast-router ports.

### IGMPv3 Membership Reports

IGMPv3 Reports are sent to:

```text
224.0.0.22
```

All IGMPv3-capable multicast routers listen to this address. Hosts send their Reports to `224.0.0.22` but do not listen for other hosts' Reports at that address.

Therefore, `224.0.0.22` is special even within the reserved range:

```text
224.0.0.22
→ IGMP control-plane destination
→ not an ordinary application multicast group
```

Some Catalyst documentation explicitly lists `224.0.0.22` as an exception to the normal flooding treatment of the rest of `224.0.0.0/24`, because the switch recognizes and processes IGMPv3 Reports destined to it.

## IGMPv3 Hosts Do Not Join 224.0.0.22

An IGMPv3 host sends Reports **to**:

```text
224.0.0.22
```

but that does not mean the host has joined `224.0.0.22` as a multicast application group.

For example:

```text
Host wants:
239.1.1.1 from 10.1.1.1

IGMPv3 Report:
Destination IP = 224.0.0.22

Group Record:
239.1.1.1
Source:
10.1.1.1
```

The actual receiver membership is:

```text
(10.1.1.1,239.1.1.1)
```

not:

```text
(*,224.0.0.22)
```

`224.0.0.22` is simply the destination used to deliver the IGMPv3 control message to multicast routers.

## Layer 2 Multicast MAC Addresses

IPv4 multicast addresses map into Ethernet multicast MAC addresses beginning with:

```text
01:00:5e
```

For the `224.0.0.0/24` block, examples include:

```text
224.0.0.1
→ 01:00:5e:00:00:01

224.0.0.2
→ 01:00:5e:00:00:02

224.0.0.5
→ 01:00:5e:00:00:05

224.0.0.22
→ 01:00:5e:00:00:16
```

Older Catalyst platforms often describe the special reserved range using the corresponding MAC range:

```text
01:00:5e:00:00:00
through
01:00:5e:00:00:ff
```

Modern Catalyst 9000 switches, however, perform IGMP snooping forwarding based on the **multicast IP group address**, rather than relying solely on multicast MAC entries.

## Example: PIM

PIM routers use:

```text
224.0.0.13
```

for PIM control messages.

This is particularly important to IGMP snooping because observing PIM traffic can also help a switch identify **mrouter ports**.

For example:

```text
R1
 |
 | PIM Hello → 224.0.0.13
 v
SW1
```

SW1 can use that PIM traffic as evidence that:

```text
R1's port
→ multicast-router port
```

So reserved multicast control traffic can also directly influence snooping state.

## 224.0.1.x Is Different

Do not confuse:

```text
224.0.0.x
```

with:

```text
224.0.1.x
```

The first is the reserved **link-local control block**:

```text
224.0.0.0/24
```

The second belongs to the broader globally scoped multicast range.

Cisco explicitly notes that IANA assigns protocol/application addresses from `224.0.1.x`, and multicast routers can forward these addresses.

For example:

```text
224.0.1.40
```

is **not** covered by the special `224.0.0.0/24` link-local rule.

It can therefore appear as an ordinary IGMP snooping membership:

```text
Vlan      Group/source        Type     Version    Port List
----------------------------------------------------------------
10        224.0.1.40          I        v3         Et0/0
```

The boundary is important:

```text
224.0.0.255
→ link-local control range

224.0.1.0
→ outside the link-local control range
```

## Verification

Normal listener state can be inspected with:

```text
SW1# show ip igmp snooping groups
```

However, you should not expect every control multicast protocol in `224.0.0.0/24` to require an ordinary listener entry before its traffic is forwarded.

For example, the absence of:

```text
224.0.0.5 → E0/1
```

from the IGMP snooping group table does not imply that OSPF multicast will be dropped.

This is precisely why these reserved control groups must be distinguished from ordinary snooped multicast groups.

## Key Points

```text
224.0.0.0/24
→ reserved link-local network-control multicast

Used by protocols such as:
→ IGMP
→ OSPF
→ EIGRP
→ PIM
→ VRRP

IP routers
→ do not forward this range between subnets

Typical TTL
→ 1

Ordinary application multicast
→ IGMP snooping learns listener ports
→ traffic can be selectively forwarded

224.0.0.0/24 control multicast
→ receives special local forwarding treatment
→ does not depend on ordinary IGMP listener membership

Unknown multicast
→ flooded because no snooping state exists

Reserved link-local multicast
→ special behavior because of the address range itself

224.0.0.22
→ IGMPv3 Membership Report destination
→ special IGMP control address

224.0.1.x
→ NOT part of the 224.0.0.0/24 link-local block
→ can be routed and snooped normally
```