# LDP Session Protection

This page covers **LDP Session Protection**.

The previous LDP page introduced two discovery types:

```text
Basic discovery
Targeted discovery
```

Basic discovery is used for directly connected LDP peers.

Targeted discovery is used for LDP peers that are not directly connected, or for cases where a directly connected LDP session should survive a local link failure.

That second use case is **LDP Session Protection**.

---

## The Problem LDP Session Protection Solves

In a normal MPLS core, directly connected LSRs discover each other with **LDP Link Hellos**.

Example:

```text
P1 --- P2
```

If `mpls ip` is enabled on the link, P1 and P2 send LDP Hellos on the link.

```text
LDP Link Hello:
- UDP 646
- Sent on the directly connected link
- Multicast destination 224.0.0.2
```

After discovery, the routers build an LDP TCP session.

```text
LDP session:
- TCP 646
- Used to exchange labels
```

If the directly connected link fails, the LDP Link Hello adjacency fails.

Without session protection, the LDP session goes down too.

When the LDP session goes down, label bindings learned from that peer are removed or marked unusable.
When the link comes back, the routers must rediscover each other, rebuild the TCP session, exchange address messages, exchange label mappings, and rebuild usable LFIB entries.

That takes time. LDP Session Protection reduces that delay.

---

## Basic Idea

LDP Session Protection protects an LDP session by adding a **Targeted Hello adjacency** in addition to the normal Link Hello adjacency.

Normal directly connected LDP:

```text
P1 --- P2

Discovery source:
P1-P2 link hello
```

With LDP Session Protection:

```text
P1 --- P2

Discovery sources:
1. P1-P2 link hello
2. P1 loopback to P2 loopback targeted hello
```

The LDP TCP session is now supported by an extra discovery source.

If the direct link fails but the peer is still reachable through another IP path, the targeted hello adjacency can keep the LDP session alive.

---

## Topology Example

Use this topology. We'll focus on P1-P2-P3:

```text
            P3
           /  \
          /    \
 PE1 --- P1 --- P2 --- PE2
```

Loopbacks:

```text
P1 Loopback0: 2.2.2.2/32
P2 Loopback0: 3.3.3.3/32
P3 Loopback0: 5.5.5.5/32
```

P1 and P2 have a directly connected LDP session over the P1-P2 link.

```text
P1 --- P2
```

But P1 can also reach P2's loopback through P3:

```text
P1 → P3 → P2
```

That alternate IP reachability is what makes session protection useful.

If the P1-P2 link fails, P1 and P2 can still reach each other's LDP router IDs through P3.

```text
P1 to 3.3.3.3:
P1 → P3 → P2

P2 to 2.2.2.2:
P2 → P3 → P1
```

So the targeted hello adjacency can remain up even though the directly connected link is down.

---

## Link Hello vs Targeted Hello

LDP uses Hellos for discovery.

There are two major types:

| Hello Type | Used For | Destination | Typical Use |
| :--- | :--- | :--- | :--- |
| Link Hello | Directly connected LDP peers | `224.0.0.2` | Normal LDP core links |
| Targeted Hello | Specific LDP peer | Unicast peer address | Session protection, TE tunnels, pseudowires, targeted LDP |

A **Link Hello** is sent on an LDP-enabled interface.

```text
interface Ethernet0/0
 mpls ip
```

A **Targeted Hello** is sent to a specific peer address.

```text
P1 targeted hello source: 2.2.2.2
P1 targeted hello target: 3.3.3.3
```

In IOS XE output, you'll see something like this:

```text
Targeted Hello 2.2.2.2 -> 3.3.3.3, active, passive
```

That means P1 is sending targeted hellos from `2.2.2.2` to `3.3.3.3`.

---

## Discovery Adjacency vs LDP Session

This distinction is important.

An **LDP discovery adjacency** is created by Hellos.

An **LDP session** is the TCP session used to exchange labels.

```text
Discovery adjacency = Hellos
LDP session         = TCP connection
```

One LDP session can have multiple discovery sources.

Example with session protection:

```text
Peer LDP ID: 2.2.2.2:0

Discovery sources:
- Ethernet0/0, Src IP addr: 10.12.12.2
- Targeted Hello 1.1.1.1 -> 2.2.2.2

LDP TCP session:
- 2.2.2.2.646 to 1.1.1.1.xxxxx
```

The important point is:

```text
The link hello can fail while the LDP TCP session remains up.
```

That is the main purpose of LDP Session Protection.

---

## Behavior Without LDP Session Protection

Topology:

```text
      P3
     /  \
    /    \
   P1 --- P2
```

Assume P1 and P2 have an LDP session over the direct link.

Without session protection, the failure process is roughly:

```text
1. P1-P2 link fails.

2. P1 and P2 stop receiving link hellos from each other.

3. The LDP link adjacency is removed.

4. There is no other discovery source for that peer.

5. The LDP TCP session is terminated.

6. Label bindings learned from that peer are removed or become unusable.

7. LFIB entries that depended on that peer's labels are removed or changed.

8. When the link comes back, LDP must rebuild the session and relearn labels.
```

Even if the physical link comes back quickly, the LDP control plane has work to do.

```text
Rediscovery
TCP session establishment
Initialization
Address exchange
Label Mapping exchange
LIB update
LFIB update
```

This can delay MPLS forwarding after the link recovers.

---

## Behavior With LDP Session Protection

With LDP Session Protection, P1 and P2 have both:

```text
1. Link Hello adjacency over the direct link
2. Targeted Hello adjacency between loopbacks
```

Failure process:

```text
1. P1-P2 link fails.

2. The link hello adjacency fails.

3. P1 and P2 still have IP reachability through P3.

4. The targeted hello adjacency remains up.

5. The LDP TCP session remains up.

6. Label bindings learned from the peer are retained.

7. When the direct link comes back, the LDP session does not need to be rebuilt.
```

In simple form:

```text
Without session protection:
Link fails → LDP session resets → labels relearned later

With session protection:
Link fails → targeted hellos keep session alive → labels retained
```

---

## Why This Improves Convergence

LDP Session Protection improves convergence because the LDP session does not need to be reestablished after a temporary link failure.

When the direct link comes back, the routers already have:

```text
- The same LDP TCP session
- The peer address information
- The peer label bindings
- The LIB entries
```

So the router can reinstall usable LFIB entries more quickly when the IGP selects that neighbor again.

---

## Data Plane Impact During a Failure

Suppose the direct link between P1 and P2 fails:

```text
      P3
     /  \
    /    \
   P1  X  P2
```

P1 can still reach P2 through P3.

```text
P1 → P3 → P2
```

LDP Session Protection can keep the P1-P2 LDP session up.

However, traffic cannot be forwarded over the failed P1-P2 link.
The data plane must follow the new IGP path.

After the failure, P1's IGP next hop to many FECs may change from P2 to P3.

```text
Before failure:
FEC: 2.2.2.2/32
IGP next hop: P2
Outgoing label: label learned from P2

After failure:
FEC: 2.2.2.2/32
IGP next hop: P3
Outgoing label: label learned from P3
```

So, session protection does not mean P1 keeps using P2 as the data-plane next hop while the direct link is down.

The IGP still controls the path.

```text
IGP decides the next hop.
LDP provides the label for that next hop.
```

Session protection helps because the P1-P2 LDP session and labels are still present when the link recovers.

---

## Relationship to Liberal Label Retention

IOS XE uses **Liberal Label Retention**.

That means a router keeps label bindings learned from multiple LDP peers, even if those peers are not currently the IGP next hop.

Example:

```text
FEC: 4.4.4.4/32

Label from P2: 200
Label from P3: 300
```

If the IGP next hop is P2, P1 uses P2's label.

If the IGP next hop changes to P3, P1 uses P3's label.

LDP Session Protection complements this behavior.

```text
Liberal Label Retention:
Keeps labels from multiple peers.

LDP Session Protection:
Keeps the LDP session and learned labels from a peer
when the direct link to that peer temporarily fails.
```

Together, they help MPLS forwarding converge more quickly after topology changes.
Because P1 keeps P2's labels even when P2 isn't the current next hop,
P1 does not have to re-learn them after the P1-P2 link recovers.

---

## Transport Address Requirement

The LDP session uses a TCP connection between transport addresses.

Usually, the transport address is the LDP router ID.

Example:

```text
Local LDP ID:      1.1.1.1:0
Transport address: 1.1.1.1
```

For session protection, this is what you want.

```text
Use loopback-based LDP router IDs.
Advertise the loopbacks in the IGP.
Make sure the loopbacks are reachable through an alternate path.
```

If you use the interface IP as the transport address like this:

```text
interface Ethernet0/0
 mpls ldp discovery transport-address interface
```

It can defeat session protection if the interface address becomes unreachable when the link fails.

For protected sessions, prefer a loopback-based transport address.

---

## Active and Passive Targeted Hellos

Targeted LDP can work in an active/passive model.

```text
Active router:
Sends targeted Hellos to the peer.

Passive router:
Responds to targeted Hellos if configured to accept them.
```

If both routers are configured for session protection, both can actively participate.

```text
P1(config)# mpls ldp session protection
P2(config)# mpls ldp session protection
```

Another option is to configure session protection on one router and configure the other router to accept targeted Hellos.

```text
P1(config)# mpls ldp session protection
P2(config)# mpls ldp discovery targeted-hello accept
```

The second router is not independently initiating protection for all peers, but it can respond to targeted Hellos.

In controlled networks, restrict accepted targeted Hellos with an ACL.

```text
ip access-list standard LDP-TH-ACCEPT
 permit 1.1.1.1
!
mpls ldp discovery targeted-hello accept from LDP-TH-ACCEPT
```

The ACL should match the peer address that sends targeted Hellos.

> Routers **do not** accept targeted hellos by default.
> Either configure session protection on both routers, or configure session protection on one
> and configure the other to accept targeted hellos.

---

## Configuration Options

The main command is:

```text
mpls ldp session protection [vrf <vrf-name>] [for <acl>] [duration {infinite | <seconds>}]
```

Common forms:

```text
mpls ldp session protection
mpls ldp session protection duration infinite
mpls ldp session protection duration 600
mpls ldp session protection for LDP-SP-PEERS
mpls ldp session protection for LDP-SP-PEERS duration 600
```

Meaning:

| Command | Meaning |
| :--- | :--- |
| `mpls ldp session protection` | Protect all eligible LDP sessions using the default duration |
| `mpls ldp session protection duration infinite` | Protect eligible sessions indefinitely after the link adjacency is lost |
| `mpls ldp session protection duration 600` | Retain the targeted hello adjacency for 600 seconds after the link adjacency is lost |
| `mpls ldp session protection for LDP-SP-PEERS` | Protect only peers matched by a standard ACL |
| `mpls ldp session protection for LDP-SP-PEERS duration 600` | Protect only matched peers and retain protection for 600 seconds |

The `for` option uses a **standard IP ACL**.

> The default duration is 24 hours, meaning the targeted hello adjacency is retained for 24 hours after the link is lost.

---

## Basic Configuration

Example topology:

```text
      P3
     /  \
    /    \
   P1 --- P2
```

This configuration protects all eligible LDP sessions on P1 and P2.

### P1

```text
interface Loopback0
 ip address 1.1.1.1 255.255.255.255
 ip ospf 1 area 0
!
interface Ethernet0/0
 description P1-P2
 ip address 10.12.12.1 255.255.255.252
 ip ospf 1 area 0
 mpls ip
!
interface Ethernet0/1
 description P1-P3
 ip address 10.13.13.1 255.255.255.252
 ip ospf 1 area 0
 mpls ip
!
router ospf 1
 router-id 1.1.1.1
!
mpls label protocol ldp
mpls ldp router-id Loopback0 force
mpls ldp session protection duration infinite
```

### P2

```text
interface Loopback0
 ip address 2.2.2.2 255.255.255.255
 ip ospf 1 area 0
!
interface Ethernet0/0
 description P2-P1
 ip address 10.12.12.2 255.255.255.252
 ip ospf 1 area 0
 mpls ip
!
interface Ethernet0/1
 description P2-P3
 ip address 10.23.23.2 255.255.255.252
 ip ospf 1 area 0
 mpls ip
!
router ospf 1
 router-id 2.2.2.2
!
mpls label protocol ldp
mpls ldp router-id Loopback0 force
mpls ldp session protection duration infinite
```

### P3

P3 does not need to be part of the protected P1-P2 targeted session.
It just needs to provide alternate IP reachability between P1 and P2.

```text
interface Loopback0
 ip address 3.3.3.3 255.255.255.255
 ip ospf 1 area 0
!
interface Ethernet0/0
 description P3-P1
 ip address 10.13.13.3 255.255.255.252
 ip ospf 1 area 0
 mpls ip
!
interface Ethernet0/1
 description P3-P2
 ip address 10.23.23.3 255.255.255.252
 ip ospf 1 area 0
 mpls ip
!
router ospf 1
 router-id 3.3.3.3
!
mpls label protocol ldp
mpls ldp router-id Loopback0 force
```

In this example, P3 is an alternate transit router.
If the P1-P2 link fails, P1 and P2 can still reach each other's loopbacks through P3.

> Although the default configuration means P1 and P2 will attempt to protect all sessions,
> P3 won't accept the targeted hellos by default, so the P1-P3 and P2-P3 sessions will not be protected.

---

## Selective Configuration with an ACL

In a larger network, you might not want to protect every LDP session.

Use the `for` keyword with a standard ACL.

Example requirement:

```text
Protect only the P1-P2 LDP session.
Do not protect the P1-P3 or P2-P3 LDP sessions.
Retain protection for 10 minutes.
```

### P1

```text
ip access-list standard LDP-SP-PEERS
 permit 2.2.2.2
!
mpls ldp session protection for LDP-SP-PEERS duration 600
```

### P2

```text
ip access-list standard LDP-SP-PEERS
 permit 1.1.1.1
!
mpls ldp session protection for LDP-SP-PEERS duration 600
```

The ACL matches the peer LDP router IDs to protect.

P1 protects the session to P2's LDP router ID.
P2 protects the session to P1's LDP router ID.

---

## One-Sided Configuration with Targeted Hello Accept

Another option:

```text
One router initiates session protection.
The other router accepts targeted Hellos.
```

Example:

### P1

```text
ip access-list standard LDP-SP-PEERS
 permit 2.2.2.2
!
mpls ldp session protection for LDP-SP-PEERS duration 600
```

### P2

```text
ip access-list standard LDP-TH-ACCEPT
 permit 1.1.1.1
!
mpls ldp discovery targeted-hello accept from LDP-TH-ACCEPT
```

P1 actively sends targeted Hellos to P2.

P2 is allowed to respond to targeted Hellos from P1.

---

## Manual Targeted LDP

You can manually configure a targeted LDP neighbor.

```text
mpls ldp neighbor 5.5.5.5 targeted
```

This tells the router to send targeted Hellos to `5.5.5.5` and try to form a targeted LDP session with that peer.

Manual targeted LDP is commonly used when the LDP peers are not directly connected.

Example:

```text
P1 --- P2 --- P3
```

P1 and P3 are not directly connected, but they can reach each other's loopbacks through P2.

```text
P1(config)# mpls ldp neighbor 5.5.5.5 targeted
```

For the targeted session to form, the remote router must also participate.

One option is to configure the targeted neighbor on both routers:

```text
P1(config)# mpls ldp neighbor 5.5.5.5 targeted
P3(config)# mpls ldp neighbor 2.2.2.2 targeted
```

Another option is to configure one router to initiate the targeted Hellos, and the other router to accept targeted Hellos:

```text
P1(config)# mpls ldp neighbor 5.5.5.5 targeted
P3(config)# mpls ldp discovery targeted-hello accept
```

Manual targeted LDP and LDP Session Protection both use targeted Hellos, but they are not the same feature.

```text
Manual targeted LDP:
Creates a targeted LDP neighbor/session.

LDP Session Protection:
Protects an existing directly connected LDP session by adding targeted Hellos.
```

If a task says to create a targeted LDP session between non-directly-connected routers, use:

```text
mpls ldp neighbor <peer-ldp-router-id> targeted
```

If a task says to protect directly connected LDP sessions during link failure, use:

```text
mpls ldp session protection
```

---

## Verification: Discovery

Check `show mpls ldp discovery`.

Before session protection, you may only see the directly connected interface:

```text
P1# show mpls ldp discovery 
 Local LDP Identifier:
    2.2.2.2:0
    Discovery Sources:
    Interfaces:
        Ethernet0/0 (ldp): xmit/recv
            LDP Id: 1.1.1.1:0
        Ethernet0/1 (ldp): xmit/recv
            LDP Id: 3.3.3.3:0
        Ethernet0/2 (ldp): xmit/recv
            LDP Id: 5.5.5.5:0
```

After session protection, you should also see targeted Hellos:

```text
P1# show mpls ldp discovery 
 Local LDP Identifier:
    2.2.2.2:0
    Discovery Sources:
    Interfaces:
        Ethernet0/0 (ldp): xmit/recv
            LDP Id: 1.1.1.1:0
        Ethernet0/1 (ldp): xmit/recv
            LDP Id: 3.3.3.3:0
        Ethernet0/2 (ldp): xmit/recv
            LDP Id: 5.5.5.5:0
    Targeted Hellos:
        2.2.2.2 -> 3.3.3.3 (ldp): active/passive, xmit/recv
            LDP Id: 3.3.3.3:0
        2.2.2.2 -> 5.5.5.5 (ldp): active, xmit

        2.2.2.2 -> 1.1.1.1 (ldp): active, xmit
```

Important things to check:

```text
Targeted Hellos present
xmit/recv shown
Correct local and remote loopback addresses
Correct peer LDP ID
```

If you only see `xmit` and not `recv`, the remote router is not responding to targeted Hellos.

> In the above output, only 3.3.3.3 (P2) is responding to P1's targeted hellos.

---

## Verification: Neighbor Detail

Check `show mpls ldp neighbor detail`.

Example while the direct link is up:

```text
P1# show mpls ldp neighbor 3.3.3.3 detail 
    Peer LDP Ident: 3.3.3.3:0; Local LDP Ident 2.2.2.2:0
        TCP connection: 3.3.3.3.30080 - 2.2.2.2.646
        Password: not required, none, in use
        State: Oper; Msgs sent/rcvd: 67/67; Downstream; Last TIB rev sent 24
        Up time: 00:46:59; UID: 2; Peer Id 1
        LDP discovery sources:
          Ethernet0/1; Src IP addr: 10.1.2.2 
            holdtime: 15000 ms, hello interval: 5000 ms
          Targeted Hello 2.2.2.2 -> 3.3.3.3, active, passive;
            holdtime: infinite, hello interval: 10000 ms
        Addresses bound to peer LDP Ident:
          10.2.2.1        3.3.3.3         10.1.2.2        10.2.3.1        
        Peer holdtime: 180000 ms; KA interval: 60000 ms; Peer state: estab
        Clients: Dir Adj Client
        LDP Session Protection enabled, state: Ready
            duration: 86400 seconds
! remaining output omitted
```

`Ready` means the session has the targeted hello adjacency needed for protection.

---

## Verification After Link Failure

Now shut down the direct P1-P2 link.

```text
P1(config)# interface Ethernet0/1
P1(config-if)# shutdown
```

If session protection is working and the alternate path through P3 exists, the LDP session should remain up.

Example:

```text
P1# show mpls ldp neighbor 3.3.3.3 detail 
    Peer LDP Ident: 3.3.3.3:0; Local LDP Ident 2.2.2.2:0
        TCP connection: 3.3.3.3.30080 - 2.2.2.2.646
        Password: not required, none, in use
        State: Oper; Msgs sent/rcvd: 70/69; Downstream; Last TIB rev sent 24
        Up time: 00:48:58; UID: 2; Peer Id 1
        LDP discovery sources:
          Targeted Hello 2.2.2.2 -> 3.3.3.3, active, passive;
            holdtime: infinite, hello interval: 10000 ms
        Addresses bound to peer LDP Ident:
          10.2.2.1        3.3.3.3         10.1.2.2        10.2.3.1        
        Peer holdtime: 180000 ms; KA interval: 60000 ms; Peer state: estab
        Clients: Dir Adj Client
        LDP Session Protection enabled, state: Protecting
            duration: 86400 seconds
            holdup time remaining: 86391 seconds
```

Key points:

```text
The Ethernet0/1 discovery source is gone.
The Targeted Hello discovery source remains.
The TCP session remains up.
The session protection state is Protecting.
```

---

## Session Protection States

You may see these states:

| State | Meaning |
| :--- | :--- |
| `Ready` | Targeted hello adjacency is up and ready to protect the LDP session |
| `Protecting` | Link adjacency is gone, but the targeted hello adjacency is keeping the session up |
| `Incomplete` | Session protection is configured, but the targeted hello adjacency is not up yet |

`Incomplete` usually means something is wrong.

Common causes:

```text
No route to the peer LDP router ID
No route back from the peer to the local LDP router ID
Remote peer does not accept targeted Hellos
ACL mismatch
UDP 646 blocked
Wrong LDP router ID in the ACL
```

---

## Verification: Labels Are Retained

Before failure, check the bindings learned from the peer:

```text
P1# show mpls ldp bindings neighbor 3.3.3.3
  lib entry: 1.1.1.1/32, rev 8
        remote binding: lsr: 3.3.3.3:0, label: 17
  lib entry: 2.2.2.2/32, rev 2
        remote binding: lsr: 3.3.3.3:0, label: 16
  lib entry: 3.3.3.3/32, rev 12
        remote binding: lsr: 3.3.3.3:0, label: imp-null
  lib entry: 4.4.4.4/32, rev 16
        remote binding: lsr: 3.3.3.3:0, label: 19
  lib entry: 5.5.5.5/32, rev 22
        remote binding: lsr: 3.3.3.3:0, label: 21
  lib entry: 10.1.1.0/24, rev 4
        remote binding: lsr: 3.3.3.3:0, label: 18
  lib entry: 10.1.2.0/24, rev 28
        remote binding: lsr: 3.3.3.3:0, label: imp-null
  lib entry: 10.1.3.0/24, rev 18
        remote binding: lsr: 3.3.3.3:0, label: 20
  lib entry: 10.2.2.0/24, rev 14
        remote binding: lsr: 3.3.3.3:0, label: imp-null
  lib entry: 10.2.3.0/24, rev 20
        remote binding: lsr: 3.3.3.3:0, label: imp-null
```

After the direct link fails, the LDP session should still be up.
The bindings learned from P2 should still exist.

```text
P1# show mpls ldp bindings neighbor 3.3.3.3
  lib entry: 1.1.1.1/32, rev 8
        remote binding: lsr: 3.3.3.3:0, label: 17
  lib entry: 2.2.2.2/32, rev 2
        remote binding: lsr: 3.3.3.3:0, label: 16
  lib entry: 3.3.3.3/32, rev 12
        remote binding: lsr: 3.3.3.3:0, label: imp-null
  lib entry: 4.4.4.4/32, rev 16
        remote binding: lsr: 3.3.3.3:0, label: 19
  lib entry: 5.5.5.5/32, rev 22
        remote binding: lsr: 3.3.3.3:0, label: 21
  lib entry: 10.1.1.0/24, rev 4
        remote binding: lsr: 3.3.3.3:0, label: 18
  lib entry: 10.1.2.0/24, rev 30
        remote binding: lsr: 3.3.3.3:0, label: imp-null
  lib entry: 10.1.3.0/24, rev 18
        remote binding: lsr: 3.3.3.3:0, label: 20
  lib entry: 10.2.2.0/24, rev 14
        remote binding: lsr: 3.3.3.3:0, label: imp-null
  lib entry: 10.2.3.0/24, rev 20
        remote binding: lsr: 3.3.3.3:0, label: imp-null
```

This is the control-plane benefit.

However, do not assume those labels are currently used in the LFIB.
The LFIB follows the current IGP next hop.

Before P1-P2 link failure:

```text
P1# show mpls forwarding-table 4.4.4.4
Local      Outgoing   Prefix           Bytes Label   Outgoing   Next Hop    
Label      Label      or Tunnel Id     Switched      interface              
19         19         4.4.4.4/32       137           Et0/1      10.1.2.2  
```

After P1-P2 link failure:

```text
P1# show mpls forwarding-table 4.4.4.4
Local      Outgoing   Prefix           Bytes Label   Outgoing   Next Hop    
Label      Label      or Tunnel Id     Switched      interface              
19         16         4.4.4.4/32       77            Et0/2      10.1.3.2  
```

> Although P1 maintains the LDP peering with P2 and retains P2's labels,
> the LSP changes from P1 → P2 to P1 → P3 → P2 

---

## LDP Timers

There are several timers involved in LDP discovery and LDP sessions:

| Timer                      | Purpose                                                                           |
| :------------------------- | :-------------------------------------------------------------------------------- |
| Link Hello interval        | How often directly connected LDP Hellos are sent                                  |
| Link Hello holdtime        | How long a link discovery adjacency is kept without receiving Link Hellos         |
| Targeted Hello interval    | How often targeted LDP Hellos are sent                                            |
| Targeted Hello holdtime    | How long a targeted discovery adjacency is kept without receiving targeted Hellos |
| Session keepalive interval | How often LDP Keepalive messages are sent over the TCP session                    |
| Session holdtime           | How long the LDP session is kept without receiving LDP messages from the peer     |

IOS XE defaults:

```text
P1# show mpls ldp parameters 
!output omitted
Session hold time: 180 sec; keep alive interval: 60 sec
Discovery hello: holdtime: 15 sec; interval: 5 sec
Discovery targeted hello: holdtime: 90 sec; interval: 10 sec
Downstream on Demand max hop count: 255
LDP for targeted sessions
LDP initial/maximum backoff: 15/120 sec
!output omitted
```

In simple terms:

```text
Discovery timers control discovery adjacencies.
Session timers control the LDP TCP session.
```

The discovery timers control whether the router still considers the peer discovered.

The session timers control whether the established LDP TCP session remains alive.

With LDP Session Protection, the link discovery adjacency can fail while the targeted discovery adjacency and the LDP session remain up.

---

## Modifying LDP Timers

LDP discovery timers and LDP session timers are modified with different commands.

Discovery timers control LDP Hello behavior.

Session timers control the LDP TCP session.

```text
Discovery timers = Hellos
Session timers   = LDP messages over TCP
```

---

### Modifying Link Hello Timers

Link Hello timers are used for directly connected LDP neighbors.

Default values:

```text
Discovery hello: holdtime: 15 sec; interval: 5 sec
```

To change the Link Hello interval:

```text
mpls ldp discovery hello interval <seconds>
```

Example:

```text
P1(config)# mpls ldp discovery hello interval 3
```

To change the Link Hello holdtime:

```text
mpls ldp discovery hello holdtime <seconds>
```

Example:

```text
P1(config)# mpls ldp discovery hello holdtime 9
```

This means:

```text
Send Link Hellos every 3 seconds.
Remove the link discovery adjacency if no Link Hellos are received for 9 seconds.
```

---

### Modifying Targeted Hello Timers

Targeted Hello timers are used for targeted LDP discovery.

This includes:

```text
Manual targeted LDP
LDP Session Protection
```

Default values:

```text
Discovery targeted hello: holdtime: 90 sec; interval: 10 sec
```

To change the Targeted Hello interval:

```text
mpls ldp discovery targeted-hello interval <seconds>
```

Example:

```text
P1(config)# mpls ldp discovery targeted-hello interval 5
```

To change the Targeted Hello holdtime:

```text
mpls ldp discovery targeted-hello holdtime <seconds>
```

Example:

```text
P1(config)# mpls ldp discovery targeted-hello holdtime 45
```

This means:

```text
Send targeted Hellos every 5 seconds.
Remove the targeted discovery adjacency if no targeted Hellos are received for 45 seconds.
```

With LDP Session Protection, this timer matters because the targeted Hello adjacency is what keeps the LDP session alive after the link Hello adjacency fails.

---

### Modifying the Session Holdtime

The LDP session holdtime controls how long the LDP TCP session is maintained without receiving LDP messages from the peer.

Default values:

```text
Session hold time: 180 sec; keep alive interval: 60 sec
```

To change the LDP session holdtime:

```text
mpls ldp holdtime <seconds>
```

Example:

```text
P1(config)# mpls ldp holdtime 120
```

This changes the locally proposed LDP session holdtime.

When two LSRs establish an LDP session, the actual session holdtime is the lower value proposed by the two routers.

Example:

```text
P1 configured session holdtime: 120 seconds
P2 configured session holdtime: 180 seconds

Negotiated session holdtime: 120 seconds
```

> The session keepalive interval is derived from the session holdtime (1/3).

---

### Example: Tuning All LDP Timers

Example requirement:

```text
Use 3-second Link Hellos with a 9-second holdtime.
Use 5-second Targeted Hellos with a 45-second holdtime.
Use a 120-second LDP session holdtime.
```

Configuration:

```text
P1(config)# mpls ldp discovery hello interval 3
P1(config)# mpls ldp discovery hello holdtime 9
P1(config)# mpls ldp discovery targeted-hello interval 5
P1(config)# mpls ldp discovery targeted-hello holdtime 45
P1(config)# mpls ldp holdtime 120
```

---

## LDP Session Protection and MPLS L3VPN

In MPLS L3VPN, the packet usually has two labels:

```text
Outer label = transport label
Inner label = VPN label
```

LDP usually provides the **outer transport label**.

MP-BGP provides the **inner VPN label**.

LDP Session Protection protects the LDP session and the transport label bindings.
It does not protect MP-BGP sessions and it does not directly protect VPN label exchange.

Example:

```text
PE1 learns VPNv4 route from PE2.
BGP next hop: PE2 loopback, 2.2.2.2
VPN label: 300
Transport label to 2.2.2.2: learned by LDP
```

If the LDP session to a next-hop LSR fails, the transport label may be lost or become unusable.
That can break forwarding even if the VPNv4 route is still in the BGP table.

Session protection helps keep the transport label control plane stable during temporary link failures.

---

## Common Problem 1: No Alternate Route Between Loopbacks

Symptom:

- LDP Session Protection enabled, state: Incomplete
- or the session drops when the direct link fails.

Check:

```text
show ip route <peer-ldp-router-id>
ping <peer-ldp-router-id> source <local-ldp-router-id>
```

Fix:

```text
Advertise both LDP router IDs in the IGP.
Make sure the IGP has an alternate path after the direct link fails.
```

---

## Common Problem 2: Using an Interface Transport Address

Symptom:

- Session protection works while the link is up,
- but the LDP session still fails when the link goes down.

Possible cause:

- The LDP TCP session uses the directly connected interface address as the transport address.
- When the link fails, that address becomes unreachable.

Check:

```text
show mpls ldp discovery detail
show mpls ldp neighbor detail
```

Look for the transport address.

Fix:

- Use a loopback-based LDP router ID.
- Advertise the loopback in the IGP.
- Do not use interface transport addresses for protected sessions.

Example:

```text
interface Loopback0
 ip address 1.1.1.1 255.255.255.255
 ip ospf 1 area 0
!
mpls ldp router-id Loopback0 force
```

---

## Common Problem 3: Targeted Hellos Not Accepted

Symptom:

```text
show mpls ldp discovery

Targeted Hellos:
    1.1.1.1 -> 2.2.2.2 (ldp): active, xmit
```

But no `recv`.

Possible cause:

- The remote router is ignoring targeted Hello requests.

Fix one of these:

- Configure session protection on both routers.
- or configure the passive router to accept targeted Hellos:

```text
mpls ldp discovery targeted-hello accept
```

Preferably restrict it:

```text
ip access-list standard LDP-TH-ACCEPT
 permit 1.1.1.1
!
mpls ldp discovery targeted-hello accept from LDP-TH-ACCEPT
```

---

## Packet and Control-Plane Summary

Before failure:

```text
P1 and P2 have:
- Link Hello adjacency
- Targeted Hello adjacency
- One LDP TCP session
- Label bindings exchanged
```

During direct link failure:

```text
P1-P2 link hello adjacency fails.
P1 and P2 remain reachable through P3.
Targeted Hello adjacency remains up.
LDP TCP session remains up.
Label bindings are retained.
IGP selects the alternate path.
LFIB follows the IGP-selected next hop.
```

When direct link recovers:

```text
P1-P2 link hello adjacency returns.
The LDP session does not need to be rebuilt.
Label mappings do not need to be relearned.
LFIB can converge more quickly when the IGP selects the direct path again.
```

---

## Common Misunderstandings

### Misunderstanding 1: Session protection protects traffic forwarding

Not exactly.

Session protection protects the **LDP session**.

Traffic forwarding still depends on:

```text
IGP reachability
CEF
LFIB
Labels from the current IGP next-hop LSR
```

---

### Misunderstanding 2: Session protection is useful without an alternate path

Not really.

If the direct link fails and there is no alternate IP path between LDP router IDs, targeted Hellos cannot keep the session alive.

```text
No alternate path = no protected session after link failure
```

---

### Misunderstanding 3: Targeted LDP and session protection are identical

They are related, but not identical.

Targeted LDP is the mechanism.
Session protection is a feature that uses targeted Hellos to protect an LDP session.

---

### Misunderstanding 4: The protected peer's labels are always used during the failure

The router uses the label from the current IGP next-hop LSR.

If the direct link to the protected peer fails, the IGP next hop may change to another router.
The LFIB should then use the label from that new next-hop router.

The protected peer's labels are still retained, which helps when the original path returns.

---

## Key Points

* LDP Session Protection keeps an LDP session up during certain link failures.
* It uses Targeted Hellos in addition to normal Link Hellos.
* Link Hellos discover directly connected LDP peers.
* Targeted Hellos are unicast to a specific LDP peer.
* One LDP session can have multiple discovery sources (link hellos and targeted hellos).
* If the direct link fails but the peer is still reachable through IP, the targeted hello adjacency can keep the LDP session alive.
* Session protection retains LDP bindings, so labels do not need to be relearned after the link recovers.
* The LDP transport address should be stable and reachable, usually a loopback.
* The peer loopbacks must be reachable through an alternate path.
* Use `mpls ldp session protection` to protect all eligible sessions.
* Use `for <acl>` to protect selected peers.
* Use `duration infinite` or an explicit number of seconds when the duration matters.
* Use `mpls ldp discovery targeted-hello accept` when a router needs to respond to targeted Hellos without independently initiating session protection.
* `Ready` means the session is ready for protection.
* `Protecting` means the link adjacency is gone and the targeted hello adjacency is keeping the session alive.
* `Incomplete` means the targeted hello adjacency is not up (e.g., the neighbor is not accepting targeted hellos).
