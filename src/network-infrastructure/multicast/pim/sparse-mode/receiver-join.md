# Receiver Join Process

In traditional PIM Sparse Mode (PIM-SM), the **Receiver Join Process** establishes a multicast distribution tree connecting interested receivers to the Rendezvous Point (RP).

Unlike PIM Dense Mode, which initially floods multicast traffic, PIM-SM builds forwarding paths only when downstream interest exists.

The process begins when a receiver signals interest in a multicast group using **IGMP (IPv4)** or **MLD (IPv6)**.

The receiver-facing PIM router then sends a **`(*,G)` Join** upstream toward the RP, establishing the shared tree (RPT).

The basic process is:

```text
Receiver joins multicast group
              |
              v
IGMP/MLD membership established
              |
              v
Last-Hop Router (LHR) creates (*,G) state
              |
              v
PIM Join(*,G) sent toward RP
              |
              v
Intermediate routers establish (*,G) state
              |
              v
Shared tree established
```

**The receiver can join the shared tree even when no multicast source is currently active.**

## Example Topology

Consider the following network:

```text
                   R1 (RP)
                   1.1.1.1
                      |
                   E0/0
                      |
                   E0/0
                      R2
                   /      \
                  /        \
                 R3        R4
                (LHR)     (LHR)
                  |         |
                 PC1       PC2
```

Assume:

- R1 is the RP for `239.1.1.1`.
- R2 is an intermediate PIM router.
- R3 and R4 are receiver-facing Last-Hop Routers (LHRs).
- All relevant router interfaces have PIM Sparse Mode enabled.
- All routers know that `1.1.1.1` is the RP for the group.

Initially, there are no interested receivers and no shared-tree forwarding state for `239.1.1.1`.

## Step 1: Receiver Sends an IGMP Membership Report

Suppose PC1 wants to receive multicast traffic for:

```text
239.1.1.1
```

PC1 sends an IGMP Membership Report on its local LAN.

For IGMPv2, the report is sent to the multicast group address itself:

```text
PC1 → 239.1.1.1

IGMPv2 Membership Report
Group: 239.1.1.1
```

R3 receives the report and records that its receiver-facing interface has interested receivers for this group.

For example:

```text
R3:

IGMP Group: 239.1.1.1
Interface:  Ethernet0/1
```

This establishes **local membership state**.

At this point, IGMP has informed R3 about the receiver, but no PIM Join has necessarily been sent upstream yet.

## Step 2: The LHR Creates (*,G) State

R3 determines that it needs traffic for `239.1.1.1`.

As the PIM Designated Router (DR) on the receiver LAN, R3 is responsible for initiating the upstream shared-tree Join.

It first determines the RP associated with the group:

```text
Group: 239.1.1.1
RP:    1.1.1.1
```

R3 performs an RPF lookup toward the RP.

Suppose the result is:

```text
RPF Interface: Ethernet0/0
RPF Neighbor:  R2
```

R3 establishes shared-tree state:

```text
(*,239.1.1.1)

Incoming Interface: Ethernet0/0
Outgoing Interface:
  Ethernet0/1 (toward PC1)
```

Notice that:

- The **incoming interface** points toward the RP.
- The **outgoing interface** points toward the interested receiver.

Because R3 needs upstream traffic and has not already joined the shared tree, it sends a `(*,G)` Join toward R2.

## Step 3: LHR Sends a PIM (*,G) Join

R3 sends:

```text
Join(*,239.1.1.1)
```

The message is sent to the All PIM Routers multicast address:

```text
224.0.0.13
```

However, the **Upstream Neighbor Address** identifies R2 as the intended recipient.

For example:

```text
Source IP:          R3's Ethernet0/0 address
Destination IP:     224.0.0.13
PIM Type:           Join/Prune (3)
Upstream Neighbor:  R2

Group:              239.1.1.1
Joined Source:      RP 1.1.1.1
                    Wildcard = 1
                    RPT = 1
```

Although the source entry contains the RP address, the Wildcard and RPT flags identify it as a **`(*,G)` Join**, not a source-specific `(S,G)` Join.

PIM Join/Prune messages operate hop by hop. The message from R3 is not forwarded unchanged all the way to the RP.

Instead, each upstream router independently determines whether it needs to establish or maintain its own upstream Join.

## Step 4: Intermediate Router Processes the Join

R2 receives R3's `(*,G)` Join.

Because R2 is the intended upstream neighbor, it records downstream interest for `239.1.1.1` on the interface toward R3.

For example:

```text
R2:

(*,239.1.1.1)

Outgoing Interface:
  Ethernet0/1 (toward R3)
```

R2 also determines its RPF path toward the RP:

```text
RP:            1.1.1.1
RPF Interface: Ethernet0/0
RPF Neighbor:  R1
```

Because R2 now has downstream interest and has not yet joined the shared tree, it sends its own `(*,G)` Join toward R1.

```text
R2 → R1: Join(*,239.1.1.1)
```

This process continues hop by hop until the Join reaches the RP or an upstream router that already has the necessary shared-tree state.

**An intermediate router does not need to send a new triggered upstream Join for every downstream Join it receives.**

If it already has an upstream Join for the group, it can simply add the new downstream interface to its forwarding state.

## Step 5: The Join Reaches the RP

R1 receives the `(*,G)` Join from R2.

R1 records downstream interest on its interface toward R2:

```text
R1 (RP):

(*,239.1.1.1)

Outgoing Interface:
  Ethernet0/0 (toward R2)
```

Because R1 is the RP, it does not send another upstream `(*,G)` Join.

The shared tree is established:

```text
                R1 (RP)
                   |
                   |
                   R2
                   |
                   |
                   R3
                   |
                  PC1
```

At this point:

- R1 knows that R2 requires traffic for the group.
- R2 knows that R3 requires traffic for the group.
- R3 knows that PC1 requires traffic for the group.

No source-specific information was required to establish the tree.

If a source subsequently becomes active and the RP obtains its traffic, the RP can forward that traffic along the established shared tree.

Source registration is covered separately in [Source Registration](source-registration.md).

## Multiple Receivers Joining the Same Group

Now suppose PC2, connected to R4, joins `239.1.1.1`.

R4 learns the local IGMP membership and sends a `(*,G)` Join toward R2.

```text
R4 → R2: Join(*,239.1.1.1)
```

R2 already has an upstream shared-tree Join toward R1.

Therefore, R2 adds its interface toward R4 to the OIL without needing to establish another upstream tree branch.

The resulting topology is:

```text
                R1 (RP)
                   |
                   |
                   R2
                  /  \
                 /    \
                R3    R4
                |      |
               PC1    PC2
```

R2 now has:

```text
(*,239.1.1.1)

Incoming Interface: Ethernet0/0
Outgoing Interface List:
  Ethernet0/1 (toward R3)
  Ethernet0/2 (toward R4)
```

When traffic arrives from the RP, R2 forwards copies onto both outgoing interfaces.

**Multiple downstream receivers share the same upstream `(*,G)` tree branch.**

This avoids duplicating upstream multicast forwarding state for every receiver.

## PIM DR vs. IGMP Querier

On a receiver LAN with multiple multicast routers, two separate roles are relevant:

- **IGMP Querier:** Sends IGMP Queries to maintain receiver membership information.
- **PIM DR:** Originates upstream PIM Joins based on local receiver interest.

These roles are elected independently.

For example:

```text
               Multicast Network
                  /       \
                 R1       R2
              10.0.0.1  10.0.0.2
                  \       /
                     SW1
                      |
                     PC1
```

Suppose both routers have the default PIM DR priority of 1.

The elections produce:

| Role | Winner | Reason |
|---|---|---|
| IGMP Querier | R1 | Lowest IPv4 address |
| PIM DR | R2 | Highest IPv4 address |

R1 sends periodic IGMP Queries.

When PC1 sends an IGMP Membership Report, R2 normally assumes responsibility for originating the upstream PIM Join.

**The IGMP Querier does not have to be the router that sends the PIM Join.**

An exception exists when another router wins a PIM Assert election and assumes the relevant multicast forwarding responsibility. This is covered in [PIM Assert](../assert.md).

## Maintaining Receiver Interest

Two different mechanisms maintain the receiver's multicast path:

| Mechanism | Maintains |
|---|---|
| IGMP/MLD | Receiver membership on the local LAN |
| PIM Join/Prune | Multicast distribution-tree branches between routers |

For example:

1. PC1 periodically responds to IGMP Queries, maintaining local membership on R3.
2. R3 periodically sends `(*,G)` Joins toward R2.
3. R2 periodically refreshes its own upstream `(*,G)` Join toward R1.

The default PIM Join/Prune interval is **60 seconds**, with a default advertised Holdtime of **210 seconds**.

These mechanisms operate independently. Receiving an IGMP Membership Report does not require the router to send a new PIM Join every time, provided the necessary upstream Join state is already maintained.

## Receiver Leaves the Group

Suppose PC1 leaves `239.1.1.1`.

With IGMPv2, PC1 can send a Leave Group message. The IGMP Querier then sends Group-Specific Queries to determine whether other receivers remain on the LAN.

Once R3 determines that no local receivers require the group, and it has no other downstream interest, it no longer needs the shared tree.

R3 can send:

```text
R3 → R2: Prune(*,239.1.1.1)
```

R2 removes the downstream interest associated with R3.

However, PC2 is still receiving the group through R4.

Therefore, R2 continues maintaining its upstream Join toward R1.

```text
                R1 (RP)
                   |
                   |
                   R2
                  /  \
                 X    R4
                 R3    |
                       PC2
```

Only when R2 has no remaining downstream interest does it need to remove its own upstream shared-tree Join.

> **Note:** On multiaccess links, Prune processing can be delayed to allow other downstream routers to override the Prune. See [PIM Join and Prune Messages](join-prune.md).

## IOS XE Configuration and Verification

### Basic Configuration

Enable multicast routing and PIM Sparse Mode on the routers:

```text
R3(config)# ip multicast-routing
R3(config)# interface Ethernet0/0
R3(config-if)# ip pim sparse-mode
R3(config-if)# exit
R3(config)# interface Ethernet0/1
R3(config-if)# ip pim sparse-mode
```

Configure the RP mapping:

```text
R3(config)# ip pim rp-address 1.1.1.1
```

The intermediate routers also require PIM on the relevant interfaces and consistent RP information.

### Simulating Receiver Interest

For lab testing, IOS XE can create multicast membership without an external receiver.

Two useful commands are:

```text
R3(config)# interface Ethernet0/1
R3(config-if)# ip igmp join-group 239.1.1.1
```

Or:

```text
R3(config-if)# ip igmp static-group 239.1.1.1
```

Both can create local interest that causes PIM to establish an upstream multicast path.

However, they differ in how the router handles received multicast packets:

| Command | Behavior |
|---|---|
| `ip igmp join-group` | Makes the router itself a multicast group member; traffic is accepted by the router |
| `ip igmp static-group` | Creates forwarding interest without making the router itself a receiver |

**For forwarding tests involving significant multicast traffic, `ip igmp static-group` is generally preferable**, because `ip igmp join-group` can cause multicast traffic to be sent to the router's CPU.

These commands simulate receiver interest; they do not generate an actual IGMP Membership Report from an external PC.

### Verification Commands

| Command | Purpose |
|---|---|
| `show ip igmp groups` | Verify local receiver membership |
| `show ip igmp interface Ethernet0/1` | Verify IGMP operation and querier information |
| `show ip pim rp mapping` | Verify the RP for the group |
| `show ip rpf 1.1.1.1` | Verify the RPF path toward the RP |
| `show ip pim neighbor` | Verify PIM upstream and downstream neighbors |
| `show ip mroute 239.1.1.1` | Verify shared-tree state and outgoing interfaces |

### On the LHR

After PC1 joins, R3 should have `(*,G)` state similar to:

```text
R3# show ip mroute 239.1.1.1

(*, 239.1.1.1), 00:02:00/00:02:59, RP 1.1.1.1, flags: S
  Incoming interface: Ethernet0/0, RPF nbr 10.0.23.2
  Outgoing interface list:
    Ethernet0/1, Forward/Sparse, 00:02:00/00:02:59
```

Notice:

- The incoming interface points toward the RP.
- The outgoing interface points toward the receiver.
- The router has established `(*,G)` state even if no source is active.

### On the Intermediate Router

After both PC1 and PC2 join, R2 should have an OIL containing both downstream branches:

```text
R2# show ip mroute 239.1.1.1

(*, 239.1.1.1), 00:03:00/00:02:59, RP 1.1.1.1, flags: S
  Incoming interface: Ethernet0/0, RPF nbr 10.0.12.1
  Outgoing interface list:
    Ethernet0/1, Forward/Sparse, 00:03:00/00:02:59
    Ethernet0/2, Forward/Sparse, 00:01:30/00:02:59
```

This confirms that R2 has one upstream branch and two downstream branches.

### On the RP

R1 should have:

```text
R1# show ip mroute 239.1.1.1

(*, 239.1.1.1), 00:03:00/00:02:59, RP 1.1.1.1, flags: S
  Incoming interface: Null, RPF nbr 0.0.0.0
  Outgoing interface list:
    Ethernet0/0, Forward/Sparse, 00:03:00/00:02:59
```

The RP has no upstream RPF neighbor for its own `(*,G)` state because it is the root of the shared tree.

> **Note:** These outputs illustrate the state before source-specific SPT switchover. Once multicast data begins arriving, IOS XE may establish `(S,G)` state and switch forwarding to the SPT. Both types of state can then coexist.

## Troubleshooting the Receiver Join Process

When an expected `(*,G)` tree does not form, work from the receiver toward the RP.

| Check | Command | What to Look For |
|---|---|---|
| Receiver membership | `show ip igmp groups` | LHR has learned the correct group |
| PIM DR | `show ip pim interface` | Expected router is DR on the receiver LAN |
| RP mapping | `show ip pim rp mapping` | Correct RP is associated with the group |
| RPF path | `show ip rpf <RP>` | Valid upstream interface and neighbor |
| PIM neighbors | `show ip pim neighbor` | Upstream neighbor relationship exists |
| Multicast state | `show ip mroute <group>` | Expected incoming and outgoing interfaces |

Two important failure cases are:

**Incorrect RP mapping:** If routers disagree about the RP for a group, a router can reject a received `(*,G)` Join whose encoded RP does not match its own mapping.

**Missing RPF path toward the RP:** The LHR may know that receivers want the group but be unable to determine the upstream neighbor to which it should send the `(*,G)` Join.

If the tree forms but packets are not delivered, investigate source registration, source-specific forwarding state, and RPF checks separately.

## Receiver Joins in SSM

The process described above applies to traditional **ASM shared-tree forwarding**.

In Source-Specific Multicast (SSM), the receiver specifies both the source and group, typically using IGMPv3 or MLDv2.

For example:

```text
Source: 10.10.10.10
Group:  232.1.1.1
```

The LHR can send:

```text
Join(10.10.10.10,232.1.1.1)
```

directly toward the source.

No RP or `(*,G)` shared-tree Join is required.

Source-specific receiver signaling is covered in [Source-Specific Multicast](../ssm/overview.md).

## Key Points

- The receiver join process begins with **IGMP/MLD membership**, not a PIM Join sent by the receiver.
- The receiver-facing **PIM DR** normally initiates the upstream `(*,G)` Join.
- `(*,G)` Joins follow the **RPF path toward the RP**.
- PIM Join messages are processed hop by hop; intermediate routers originate their own upstream Joins as needed.
- The shared tree can be established **before any multicast source becomes active**.
- Multiple downstream receivers can share the same upstream `(*,G)` branch.
- Local IGMP/MLD membership and upstream PIM Join state are maintained separately.
- An intermediate router does not need to send a new upstream Join every time another downstream receiver joins.
- A router removes its upstream shared-tree interest only when it no longer requires that tree for any downstream receivers or routers.
- For IOS XE lab testing, `ip igmp join-group` and `ip igmp static-group` can simulate multicast receiver interest.