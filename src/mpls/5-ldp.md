# LDP

**LDP** stands for **Label Distribution Protocol**.

LDP is the protocol commonly used to distribute MPLS labels in a basic MPLS core.

The IGP tells routers how to reach a destination, and LDP tells routers which label to use to get there.

```
IGP = next-hop reachability
LDP = advertising label bindings
```

In a basic MPLS network, each router assigns local labels to IGP prefixes
and uses LDP to adverise those labels to its neighbors.

Example:

```
PE2 loopback: 4.4.4.4/32
LDP label:    200
```

This tells neighbors:

```
Use this label to forward traffic toward 4.4.4.4/32.
```

---

## Where LDP Fits

Example topology:

```text
CE1 --- PE1 --- P1 --- P2 --- PE2 --- CE2
```

Only the provider routers run MPLS and LDP:

```text
PE1 --- P1 --- P2 --- PE2
```

The CE routers do not run MPLS or LDP.

```text
CE routers:
- Send and receive normal IP packets
- Do not understand MPLS labels
- Do not form LDP neighbor relationships

PE/P routers:
- Run the provider IGP
- Run MPLS
- Run LDP
- Build the LFIB
```

---

## What LDP Does

LDP distributes **label bindings**.

A label binding maps a FEC to a label.

```text
FEC + label = label binding
```

In basic MPLS forwarding, the FEC is usually an IGP prefix.

Example:

```text
FEC:   4.4.4.4/32
Label: 200
```

The label binding means:

```text
For traffic going toward 4.4.4.4/32, use label 200.
```

LDP allows neighboring LSRs to exchange these bindings.

```text
P2 advertises label 200 for 4.4.4.4/32.
P1 learns that label from P2.
P1 can use label 200 when forwarding traffic toward P2 for 4.4.4.4/32.
```

---

## What LDP Does Not Do

LDP does **not** choose the path through the network.

The IGP chooses the next hop. LDP provides the label for that next hop.

```text
IGP decides where to forward.
LDP decides which label to use.
```

LDP also does not advertise customer routes in a normal MPLS L3VPN design.

```text
LDP advertises labels for provider infrastructure prefixes.
MP-BGP advertises VPN routes and VPN labels.
```

---

## LDP and the IGP

LDP depends on the IGP.

The IGP must provide reachability between provider routers before LDP can build useful label-switched paths.

Example loopbacks:

```text
PE1 loopback: 1.1.1.1/32
P1 loopback:  2.2.2.2/32
P2 loopback:  3.3.3.3/32
PE2 loopback: 4.4.4.4/32
```

The IGP advertises these loopbacks.

LDP assigns and advertises labels for them.

For example, PE1 needs to reach PE2's loopback:

```text
BGP next hop: 4.4.4.4
```

PE1 resolves `4.4.4.4/32` through the IGP.

Then it uses the LDP label learned from its next-hop LDP neighbor.

```text
BGP route points to PE2 loopback.
IGP tells PE1 to reach PE2 via P1.
LDP tells PE1 which label to use when sending to P1.
```

---

## LDP Neighbors

LDP neighbors are also called **LDP peers**.

Two directly connected MPLS routers can become LDP neighbors if LDP is enabled on the link between them.

Example:

```text
PE1 --- P1
```

If MPLS/LDP is enabled on both sides of the link, PE1 and P1 can discover each other and form an LDP session.

An LDP neighbor relationship has two major parts:

```text
1. Discovery
2. Session establishment
```

Discovery finds potential LDP peers.

The session is used to exchange label information.

---

## LDP Discovery

LDP uses **Hello messages** for discovery.

For directly connected neighbors, routers send LDP Hellos out LDP-enabled interfaces.

```text
PE1 sends LDP Hellos on the PE1-P1 link.
P1 sends LDP Hellos on the P1-PE1 link.
```

If each router receives Hellos from the other side, they discover each other as potential LDP peers.

```text
LDP Hello received
  ↓
LDP discovery adjacency created
  ↓
LDP session can be established
```

Basic LDP discovery uses:

```text
UDP port 646
Multicast destination 224.0.0.2
```

---

## LDP Session

After discovery, LDP establishes a **TCP session**.

The LDP session uses:

```text
TCP port 646
```

The TCP session is used to exchange LDP messages, including label mappings.

```text
LDP discovery:
- Exchange Hellos
- UDP 646

LDP session:
- Exchange labels
- TCP 646
```

---

## LDP Messages

LDP uses several message types.

| Message | Purpose |
| :--- | :--- |
| Hello | Discover LDP peers |
| Initialization | Negotiate session parameters |
| Keepalive | Keep the LDP session alive |
| Address | Advertise interface addresses associated with the LDP peer |
| Label Mapping | Advertise a label binding |
| Label Request | Request a label binding |
| Label Withdraw | Withdraw a label binding |
| Label Release | Release a label binding |
| Notification | Signal errors or session problems |

The most important message for basic forwarding is the **Label Mapping** message.

A Label Mapping message says:

```text
For this FEC, use this label.
```

Example:

```text
FEC:   4.4.4.4/32
Label: 200
```

---

## LDP Identifier

Each LDP speaker has an **LDP Identifier**, or **LDP ID**.

The LDP ID has this format:

```text
<LDP router ID>:<label space ID>
```

Example:

```text
1.1.1.1:0
```

In most cases, the label space ID is `0`.

So, in most basic labs, you will see LDP IDs like this:

```text
1.1.1.1:0
2.2.2.2:0
3.3.3.3:0
4.4.4.4:0
```

The first part is the LDP router ID.

The second part identifies the label space.

---

## LDP Router ID

The LDP router ID identifies the local LSR for LDP.

By default, IOS XE chooses the LDP router ID using a process similar to other router IDs:

```text
1. Choose the highest IP address on an operational loopback interface.
2. If there is no operational loopback, choose the highest IP address on an operational interface.
```

It is best to make the LDP router ID stable by using a loopback.

Example:

```text
interface Loopback0
 ip address 1.1.1.1 255.255.255.255

mpls ldp router-id Loopback0 force
```

> Unlike the OSPF RID (for example), the LDP RID **must** be derived from an interface IP. You cannot specify an arbitrary value.

The loopback should be advertised in the IGP.

If the LDP router ID is not reachable, LDP sessions may fail because peers won't be able to establish the TCP connection to the LDP router ID.

---

### Without the `force` Keyword

If you configure `mpls ldp router-id Loopback0` **without** `force`, the router does not necessarily change the LDP router ID immediately.

Example:

```text
mpls ldp router-id Loopback0
```

This tells the router:

```text
Use Loopback0 as the preferred LDP router ID the next time an LDP router ID must be selected.
```

In other words, the command may not affect current LDP sessions immediately.

The router may continue using the current LDP router ID until LDP needs to reselect the router ID, such as after an interface state change or another event that causes reselection.

So, without `force`:

```text
Existing LDP router ID may remain unchanged.
Existing LDP sessions usually stay up.
The new router ID takes effect later.
```

This is less disruptive, but it can be confusing in labs because the command may appear to have had no immediate effect.

---

### With the `force` Keyword

If you configure `mpls ldp router-id Loopback0 force`, the router tries to change the LDP router ID more immediately.

Example:

```text
mpls ldp router-id Loopback0 force
```

If Loopback0 is up and has an IP address, the router forcibly changes the LDP router ID to Loopback0's IP address.

This can reset existing LDP sessions because the local LDP identifier changes.

So, with `force`:

```text
The LDP router ID changes immediately, if the interface is up.
Existing LDP sessions may be torn down and reestablished.
Learned label bindings from those sessions may be released and relearned.
MPLS forwarding can be briefly interrupted.
```

If the specified interface is down, the router cannot use that IP address yet. In that case, the forced router ID change takes effect when the interface comes up.

> `force` is commonly used in labs to ensure routers use the desired RID. Be cautious about using it in a production network.

---

## LDP Transport Address

The **transport address** is the address used for the LDP TCP session.

LDP discovers neighbors with UDP Hellos, but the actual LDP session uses TCP.

```text
LDP discovery: UDP 646
LDP session:   TCP 646
```

By default, the LDP transport address is usually the LDP router ID.

Example:

```text
Local LDP ID:      1.1.1.1:0
Transport address: 1.1.1.1
```

This is why loopback reachability is important.

If R1 advertises `1.1.1.1` as its transport address, the neighbor must be able to route to `1.1.1.1`.

A common problem is:

```text
LDP Hellos are received on the directly connected link.
But the LDP TCP session does not establish.
```

One possible cause is that the advertised transport address is not reachable.

Example:

```text
R1 advertises transport address 1.1.1.1.
R2 has no route to 1.1.1.1.
R2 cannot establish the LDP TCP session to R1.
```

In that case, the best fix is usually to fix IGP reachability to the loopback.

However, you can also modify the LDP transport address.

---

### Modifying the LDP Transport Address

The LDP transport address can be changed on the MPLS-enabled interface.

Use this interface-level command:

```text
interface <interface>
 mpls ldp discovery transport-address <address>
```

You can specify an exact IP address:

```text
interface GigabitEthernet0/0
 mpls ip
 mpls ldp discovery transport-address 10.0.12.1
```

This tells the router to advertise `10.0.12.1` as the transport address in LDP Hellos sent out that interface.

You can also use the `interface` keyword:

```text
interface GigabitEthernet0/0
 mpls ip
 mpls ldp discovery transport-address interface
```

This tells the router to advertise the IP address of that interface as the transport address.

So the difference is:

| Command                                          | Meaning                                                           |
| :----------------------------------------------- | :---------------------------------------------------------------- |
| `mpls ldp discovery transport-address 10.0.12.1` | Advertise the specified IP address as the transport address       |
| `mpls ldp discovery transport-address interface` | Advertise the interface's own IP address as the transport address |

---

### Example: Using the Interface Address

Topology:

```text
PE1 -- P1
```

Link addresses:

```text
PE1 E0/0: 10.1.2.1/24
P1 E0/0: 10.1.2.2/24
```

PE1's LDP router ID is:

```text
1.1.1.1:0
```

By default, PE1 advertises this transport address:

```text
Transport IP addr: 1.1.1.1
```

If P1 cannot reach `1.1.1.1`, the TCP session may fail.

To make PE1 advertise its directly connected interface address instead:

```text
R1(config)# interface Ethernet0/0
R1(config-if)# mpls ip
R1(config-if)# mpls ldp discovery transport-address interface
```

Now R1 advertises:

```text
Transport IP addr: 10.1.2.1
```

R2 can reach this address directly, so the LDP TCP session can be established.

---

### Which transport address to use?

Using the interface address can make the LDP session easier to establish on a simple directly connected link.

However, using the loopback address (RID) is usually preferred in real MPLS cores because it is more stable.

```text
Prefer a loopback-based LDP router ID and transport address.
Advertise the loopback in the IGP.
```

---

### Verification

Check the advertised and received transport address with:

```text
PE1# show mpls ldp discovery detail 
 Local LDP Identifier:
    1.1.1.1:0
    Discovery Sources:
    Interfaces:
        Ethernet0/0 (ldp): xmit/recv
            Enabled: Interface config
            Hello interval: 5000 ms; Transport IP addr: 1.1.1.1
            LDP Id: 2.2.2.2:0
              Src IP addr: 10.1.1.2; Transport IP addr: 2.2.2.2
              Hold time: 15 sec; Proposed local/peer: 15/15 sec
              Reachable via 2.2.2.2/32
              Password: not required, none, in use
            Clients: IPv4, mLDP 
```

Another option:

```text
PE1# show mpls ldp neighbor 
    Peer LDP Ident: 2.2.2.2:0; Local LDP Ident 1.1.1.1:0
        TCP connection: 2.2.2.2.26789 - 1.1.1.1.646  ← TRANSPORT ADDRESSES
        State: Oper; Msgs sent/rcvd: 32/31; Downstream
        Up time: 00:18:28
        LDP discovery sources:
          Ethernet0/0, Src IP addr: 10.1.1.2
        Addresses bound to peer LDP Ident:
          10.1.1.2        10.1.2.1        2.2.2.2 
```

If you configured the interface address as the transport address, you should see the TCP session using the interface address instead:

```text
PE1(config-if)# mpls ldp discovery transport-address interface 
PE1(config-if)# end

PE1# show mpls ldp discovery detail 
 Local LDP Identifier:
    1.1.1.1:0
    Discovery Sources:
    Interfaces:
        Ethernet0/0 (ldp): xmit/recv
            Enabled: Interface config
            Hello interval: 5000 ms; Transport IP addr: 10.1.1.1
            LDP Id: 2.2.2.2:0
              Src IP addr: 10.1.1.2; Transport IP addr: 2.2.2.2
              Hold time: 15 sec; Proposed local/peer: 15/15 sec
              Reachable via 2.2.2.2/32
              Password: not required, none, in use
            Clients: IPv4, mLDP 

PE1# show mpls ldp neighbor 
    Peer LDP Ident: 2.2.2.2:0; Local LDP Ident 1.1.1.1:0
        TCP connection: 2.2.2.2.646 - 10.1.1.1.28339
        State: Oper; Msgs sent/rcvd: 12/10; Downstream
        Up time: 00:00:08
        LDP discovery sources:
          Ethernet0/0, Src IP addr: 10.1.1.2
        Addresses bound to peer LDP Ident:
          10.1.1.2        10.1.2.1        2.2.2.2 
```

Notice PE1's transport address changes to **10.1.1.1** in the second example.

---

## Basic vs Targeted LDP

There are two discovery types:

```text
Basic discovery
Targeted discovery
```

Basic discovery is used for directly connected LDP neighbors.

```text
PE1 --- P1
```

The routers send link Hellos on the directly connected interface.

Targeted LDP is used for LDP peers that are not directly connected.

```text
PE1 --- P1 --- P2 --- PE2

PE1 and PE2 can form a targeted LDP session.
```

Targeted LDP sends targeted Hellos to a specific address instead of using link-local multicast.

For a basic MPLS core, directly connected LDP sessions are usually enough.
Targeted discovery can be used for **session protection** (to be covered on its own page).

---

## Downstream Label Assignment

LDP uses **downstream label assignment**.

The downstream router tells the upstream router which label to use when forwarding packets.

Example:

```text
PE1 --- P1 --- P2 --- PE2
```

Traffic is going from PE1 toward PE2's loopback:

```text
FEC: 4.4.4.4/32
```

The downstream router from PE1's perspective is P1, so PE1 uses the label advertised by P1.

```text
P1 advertises label 100 for 4.4.4.4/32 to PE1.
PE1 uses label 100 when forwarding traffic to P1.
```

From P1's perspective, the downstream router is P2.

```text
P2 advertises label 200 for 4.4.4.4/32 to P1.
P1 swaps label 100 for label 200.
```

From P2's perspective, the downstream router is PE2.

```text
PE2 advertises implicit null for 4.4.4.4/32 to P2.
P2 pops the label before forwarding to PE2.
```

---

## LIB and LFIB

LDP learned labels are stored in the **LIB**.

The router uses the best label information to build the **LFIB**.

```text
LIB  = Label Information Base
LFIB = Label Forwarding Information Base
```

The LIB contains label bindings learned from LDP.

The LFIB is used to forward labeled packets.

```text
LDP learns label bindings
  ↓
Router stores them in the LIB
  ↓
Best usable label entries are installed in the LFIB
  ↓
LFIB forwards labeled packets
```

A router may learn multiple remote labels for the same FEC from different neighbors.

But the router forwards traffic using the label from the current IGP next hop.

Example:

```text
FEC: 4.4.4.4/32

P1 advertises label 100.
P3 advertises label 300.

If the IGP next hop is P1, use label 100.
If the IGP next hop changes to P3, use label 300.
```

> The IGP-learned underlay path determines the **LSP**.
> LDP may learn labels from multiple neighbors, but the router uses the label advertised by the neighbor selected as the IGP next hop for that FEC.

---

## Label Distribution Modes

LDP supports two label distribution methods:

```text
Downstream Unsolicited
Downstream on Demand
```

The difference is whether a router advertises label bindings automatically, or only after another router requests them.

---

### Downstream Unsolicited

In **Downstream Unsolicited** mode, an LSR advertises label bindings to its LDP peers without waiting for a request.

Example:

```text
P1 learns 4.4.4.4/32 in the IGP.
P1 assigns local label 100 for 4.4.4.4/32.
P1 advertises label 100 to its LDP peers.
```

The peer did not need to explicitly ask for the label first.

> IOS XE routers use Downstream Unsolicited label distribution.

---

### Downstream on Demand

In **Downstream on Demand** mode, an LSR advertises a label binding only after another LSR requests it.

Example:

```text
R1 needs a label for 4.4.4.4/32.
R1 sends a label request to R2.
R2 replies with a label binding for 4.4.4.4/32.
```

This can reduce unnecessary label advertisement, because labels are sent only when requested.

> IOS XE routers **do not** use Downstream on Demand label distribution.
> It cannot be configured (as far as I know).

## Label Retention Modes

Label retention controls which received remote label bindings an LSR keeps in its LIB.

LDP can learn label bindings from multiple neighbors for the same FEC.

Example:

```text
FEC: 4.4.4.4/32

P1 advertises label 100.
P2 advertises label 200.
P3 advertises label 300.
```

The question is:

```text
Should the router keep all of those labels?
Or only the label from the current IGP next hop?
```

This is controlled by the label retention method.

| Method                       | What the Router Keeps                      | Main Benefit       | Main Cost                                 |
| :--------------------------- | :----------------------------------------- | :----------------- | :---------------------------------------- |
| Liberal Label Retention      | Labels from all LDP peers                  | Faster convergence | More memory usage                         |
| Conservative Label Retention | Only labels from the current next-hop peer | Lower memory usage | Slower convergence after next-hop changes |

---

### Liberal Label Retention

In **Liberal Label Retention** mode, an LSR keeps label bindings learned from all LDP peers, even if those peers are not currently the IGP next hop for the FEC.

Example:

```text
FEC: 4.4.4.4/32

P1 advertises label 100.
P2 advertises label 200.
P3 advertises label 300.
```

Assume the current IGP next hop is P1.

With Liberal Label Retention, the router keeps all of the learned labels in the LIB:

```text
Label from P1: 100  ← used in LFIB because P1 is the IGP next hop
Label from P2: 200  ← kept in LIB, but not currently used
Label from P3: 300  ← kept in LIB, but not currently used
```

The router forwards traffic using the label from the current IGP next hop.

However, it keeps the other labels in case the IGP path changes.

Example:

```text
Current IGP next hop: P1
Current outgoing label: 100

IGP reconverges.

New IGP next hop: P2
New outgoing label: 200
```

Because the router already has P2's label in the LIB, it can install the new LFIB entry more quickly.

Main benefit:

```text
Faster convergence after an IGP next-hop change.
```

Main cost:

```text
More memory is used because the router keeps more remote label bindings.
```

> IOS XE routers use Liberal Label Retention.

---

### Conservative Label Retention

In **Conservative Label Retention** mode, an LSR keeps only the label binding from the current IGP next-hop LSR for the FEC.

Example:

```text
FEC: 4.4.4.4/32

P1 advertises label 100.
P2 advertises label 200.
P3 advertises label 300.
```

Assume the current IGP next hop is P1.

With Conservative Label Retention, the router keeps only P1's label:

```text
Label from P1: 100  ← kept and used
Label from P2: 200  ← discarded
Label from P3: 300  ← discarded
```

If the IGP next hop later changes to P2, the router may need to obtain or relearn the label from P2 before forwarding can continue normally.

Main benefit:

```text
Less memory used because fewer remote label bindings are kept.
```

Main cost:

```text
Potentially slower convergence after an IGP next-hop change.
```

> IOS XE routers **do not** use conservative label retention.
> It cannot be configured (as far as I know).

---

## Label Control Modes

Label control determines **when** an LSR is allowed to advertise a label binding for a FEC.

LDP supports two label control modes:

```text
Independent Control Mode
Ordered Control Mode
```

The difference is whether an LSR can advertise a label for a FEC immediately, or whether it must wait until it has received a label from the downstream next-hop LSR.

| Mode                | When Can an LSR Advertise a Label?                                                 | Main Benefit                                              | Main Cost                                                         |
| :------------------ | :--------------------------------------------------------------------------------- | :-------------------------------------------------------- | :---------------------------------------------------------------- |
| Independent Control | As soon as it recognizes the FEC                                                   | Faster label advertisement                                | Upstream labels may exist before the full downstream LSP is ready |
| Ordered Control     | Only if it is the egress LSR, or has received a label from the downstream next hop | Ensures downstream LSP exists before advertising upstream | Slower label advertisement                                        |

---

### Independent Control Mode

In **Independent Control Mode**, an LSR can advertise a label binding for a FEC as soon as it recognizes the FEC.

It does not have to wait until it receives a label from the downstream next-hop router.

Example:

```text
PE1 --- P1 --- P2 --- PE2
```

FEC:

```text
4.4.4.4/32 (PE2 loopback)
```

P1 learns `4.4.4.4/32` in the IGP.

With Independent Control Mode, P1 can assign and advertise a local label for `4.4.4.4/32` to its LDP peers immediately.

```text
P1 learns 4.4.4.4/32 in the IGP.
P1 assigns local label 100.
P1 advertises label 100 to its LDP peers.
```

P1 does not have to wait for P2 to advertise a label first.

This can make label distribution faster, because labels can be advertised without waiting for the full downstream label chain to be built first.

However, it also means an upstream router might learn a label before the complete downstream LSP is fully ready.

> IOS XE routers use Independent Control mode.

---

### Ordered Control Mode

In **Ordered Control Mode**, an LSR does not advertise a label for a FEC unless one of these conditions is true:

```text
1. The LSR is the egress LSR for that FEC.
2. The LSR has already received a label for that FEC from its downstream next-hop LSR.
```

Example:

```text
PE1 --- P1 --- P2 --- PE2
```

FEC:

```text
4.4.4.4/32
```

PE2 is the egress LSR for `4.4.4.4/32`.

With Ordered Control Mode, label advertisement works from the egress side back toward the ingress side.

```text
1. PE2 is the egress LSR for 4.4.4.4/32.
   PE2 advertises implicit null or a local label to P2.

2. P2 receives a label for 4.4.4.4/32 from PE2.
   P2 can now advertise its own label for 4.4.4.4/32 to P1.

3. P1 receives a label for 4.4.4.4/32 from P2.
   P1 can now advertise its own label for 4.4.4.4/32 to PE1.
```

This helps ensure that when an upstream router receives a label binding, the downstream LSP is already built.

The tradeoff is that label distribution may take longer, because advertisements must work backward from the egress LSR.

> IOS XE routers **do not** use Ordered Control Mode.
> It cannot be configured (as far as I know).

> Summary: IOS XE routers use **Downstream Unsolicited**, **Liberal Label Retention**, and **Independent Control**.

---

## Basic LDP Configuration

A simple MPLS/LDP core needs:

```text
1. CEF
2. IGP reachability
3. MPLS enabled globally
4. LDP selected as the label protocol
5. MPLS enabled on core-facing interfaces
```

Example:

```text
ip cef

mpls ip
mpls label protocol ldp

interface GigabitEthernet0/0
 ip address 10.0.12.1 255.255.255.252
 mpls ip
```

On many IOS/IOS XE platforms, `ip cef` and global `mpls ip` are enabled by default, but it is still useful to know the commands.

The important interface-level command is:

```text
mpls ip
```

This enables MPLS forwarding and LDP on that interface.

---

## LDP Router ID Configuration

Use a loopback as the LDP router ID.

Example:

```text
interface Loopback0
 ip address 1.1.1.1 255.255.255.255

mpls ldp router-id Loopback0 force
```

The `force` keyword applies the change immediately, but it can reset existing LDP sessions.

In a lab, that is usually fine.

In a production network, be careful.

---

## LDP Autoconfiguration

Instead of enabling `mpls ip` manually on every core-facing interface, IOS/IOS XE can enable LDP automatically on IGP-enabled interfaces.

For OSPF:

```text
router ospf 1
 mpls ldp autoconfig
```

For IS-IS:

```text
router isis
 mpls ldp autoconfig
```

This is useful in larger labs because it reduces the chance of forgetting `mpls ip` on a core link.

You can disable autoconfig on a specific interface:

```text
interface GigabitEthernet0/0
 no mpls ldp igp autoconfig
```

This page is not covering LDP-IGP synchronization. That feature has its own page.

---

## LDP Transport Address Configuration

You can manually control the transport address advertised in LDP Hellos.

Interface-level examples:

```text
interface GigabitEthernet0/0
 mpls ldp discovery transport-address interface
```

This tells the router to advertise the interface address as the transport address for Hellos sent on that interface.

You can also specify an address:

```text
interface GigabitEthernet0/0
 mpls ldp discovery transport-address 1.1.1.1
```

In most normal designs, using the loopback/LDP router ID as the transport address is preferred because it is stable.

However, the neighbor must have IP reachability to that address.

---

## LDP Authentication

LDP sessions use TCP, so they can be protected with TCP MD5 authentication.

A simple per-neighbor configuration:

```text
mpls ldp neighbor 2.2.2.2 password CISCO
```

Configure the matching password on the other side.

Example:

```text
R1(config)# mpls ldp neighbor 2.2.2.2 password CISCO
R2(config)# mpls ldp neighbor 1.1.1.1 password CISCO
```

The neighbor IP used in the command is the remote LDP router ID, not necessarily the directly connected interface IP.

Depending on platform and configuration style, changing LDP authentication can reset the LDP session.

For CCIE-style labs, the main point is:

```text
Both sides must use the same password for the LDP TCP session.
```

---

## Label Filtering

By default, routers may advertise labels for many IGP prefixes.

In some networks, you may want to restrict which labels are advertised or accepted.

There are two common ideas:

```text
Outbound label filtering = control which local labels you advertise.
Inbound label filtering  = control which remote labels you accept.
```

For basic MPLS forwarding, label filtering is usually not required.

But if labels are missing unexpectedly, always consider whether some filtering policy is blocking label advertisement or acceptance.

---

## Verification Commands

Useful commands:

```text
show mpls interfaces
show mpls ldp discovery
show mpls ldp neighbor
show mpls ldp neighbor detail
show mpls ldp bindings
show mpls forwarding-table
show ip route
show ip cef
```

---

## Verify MPLS Interfaces

Check which interfaces have MPLS enabled:

```text
show mpls interfaces
```

You want to see the core-facing interfaces listed.

Example:

```text
Interface              IP            Tunnel   BGP Static Operational
GigabitEthernet0/0     Yes           No       No  No     Yes
GigabitEthernet0/1     Yes           No       No  No     Yes
```

If an interface is missing, check:

```text
interface GigabitEthernet0/0
 mpls ip
```

Or check whether LDP autoconfig is enabled under the IGP.

---

## Verify LDP Discovery

Check discovery:

```text
show mpls ldp discovery
```

This shows where LDP Hellos are being sent and received.

Useful things to check:

```text
Is the correct interface listed?
Is LDP transmitting and receiving Hellos?
What is the local LDP identifier?
What LDP ID is seen from the neighbor?
```

Example:

```text
Local LDP Identifier:
    1.1.1.1:0

Discovery Sources:
    Interfaces:
        GigabitEthernet0/0 (ldp): xmit/recv
            LDP Id: 2.2.2.2:0
```

If you see `xmit` but not `recv`, the local router is sending Hellos but not receiving them.

Check the neighbor's interface configuration.

---

## Verify LDP Neighbors

Check LDP sessions:

```text
show mpls ldp neighbor
```

A working neighbor should show an operational session.

Useful things to check:

```text
Peer LDP Identifier
Local LDP Identifier
TCP connection
Session state
Discovery sources
Addresses bound to the peer
```

Example:

```text
Peer LDP Ident: 2.2.2.2:0; Local LDP Ident 1.1.1.1:0
    TCP connection: 2.2.2.2.646 - 1.1.1.1.12345
    State: Oper
```

If discovery works but the session is not operational, check reachability to the transport address.

---

## Verify Label Bindings

Check label bindings:

```text
show mpls ldp bindings
```

For a specific prefix:

```text
show mpls ldp bindings 4.4.4.4 32
```

This command shows local and remote label bindings.

You are looking for:

```text
Local binding
Remote binding from the next-hop LDP neighbor
```

Example idea:

```text
lib entry: 4.4.4.4/32
    local binding:  label: 100
    remote binding: lsr: 2.2.2.2:0, label: 200
```

The local binding is what this router advertises to its neighbors.

The remote binding is what this router learned from a neighbor.

---

## Verify LFIB

Check the MPLS forwarding table:

```text
show mpls forwarding-table
```

For a specific prefix:

```text
show mpls forwarding-table 4.4.4.4
```

The LFIB shows the actual forwarding action.

Examples:

```text
Local label  Outgoing label  Prefix
100          200             4.4.4.4/32
```

This means:

```text
If a packet arrives with label 100,
swap it to label 200,
and forward it to the next hop.
```

If the outgoing label is `Pop Label`, PHP is being used.

```text
Local label  Outgoing label  Prefix
200          Pop Label       4.4.4.4/32
```

This means:

```text
Pop the label before forwarding to the next hop.
```

---

## Common LDP Problems

### Problem 1: MPLS is not enabled on the interface

The IGP neighbor may be up, but no LDP neighbor forms.

Check:

```text
show mpls interfaces
show mpls ldp discovery
```

Fix:

```text
interface GigabitEthernet0/0
 mpls ip
```

---

### Problem 2: IGP reachability is missing

LDP depends on IP reachability.

If the remote LDP router ID or transport address is not reachable, the LDP session may fail.

Check:

```text
show ip route <remote-ldp-router-id>
ping <remote-ldp-router-id>
```

Fix the IGP advertisement.

---

### Problem 3: LDP router ID is unstable or unreachable

If the router chooses a physical interface as the LDP router ID, the ID may change if the interface goes down.

It may also choose an address that is not advertised in the IGP.

Fix:

```text
interface Loopback0
 ip address 1.1.1.1 255.255.255.255

router ospf 1
 network 1.1.1.1 0.0.0.0 area 0

mpls ldp router-id Loopback0 force
```

---

### Problem 4: Transport address is unreachable

Discovery may work on a directly connected link, but the TCP session may fail if the advertised transport address is unreachable.

Check:

```text
show mpls ldp discovery detail
show ip route <transport-address>
```

Fix the route to the transport address, or configure the transport address.

---

### Problem 5: Authentication mismatch

If LDP authentication is configured on one side but not the other, or if the passwords do not match, the TCP session fails.

Check:

```text
show mpls ldp neighbor detail
show running-config | include mpls ldp.*password
```

Fix the password on both sides.

---

### Problem 6: Labels are missing

If the LDP neighbor is up but labels are missing, check:

```text
show mpls ldp bindings
show mpls forwarding-table
show ip route
```

Possible causes:

```text
The prefix is not in the routing table.
The prefix is not learned through the expected next hop.
Label filtering is blocking the binding.
LDP is not enabled on the correct interface.
```

---

## LDP in a BGP-Free Core

LDP is what allows the provider core to be BGP-free.

Example:

```text
CE1 --- PE1 --- P1 --- P2 --- PE2 --- CE2
```

PE1 learns an external route through BGP:

```text
Destination: 203.0.113.0/24
BGP next hop: 4.4.4.4
```

The P routers do not know `203.0.113.0/24`.

LDP provides labels for the BGP next hop:

```text
FEC: 4.4.4.4/32
```

So PE1 pushes a label for PE2's loopback.

P1 and P2 forward based on that label.

```text
P routers do not need the final destination route.
They only need labels for the PE loopbacks.
```

That is the key MPLS transport idea.

---

## Common Misunderstandings

### Misunderstanding 1: LDP chooses the path

LDP does not choose the path.

The IGP chooses the path.

LDP provides labels for the IGP-selected next hop.

```text
IGP path changes → LDP uses labels for the new next hop.
```

---

### Misunderstanding 2: LDP replaces the IGP

LDP does not replace the IGP.

Without the IGP, LDP has no useful routing information to build label forwarding entries from.

```text
IGP provides reachability.
LDP provides labels.
```

---

### Misunderstanding 3: The LDP label represents the final customer route

In a BGP-free core, the transport label usually represents the BGP next hop, not the final destination prefix.

Example:

```text
Final destination: 203.0.113.0/24
BGP next hop:      4.4.4.4
LDP label for:     4.4.4.4/32
```

The label gets the packet to the egress PE.

The egress PE then forwards the IP packet normally.

---

### Misunderstanding 4: LDP must run on CE routers

CE routers do not run LDP in a normal MPLS L3VPN design.

Only the provider MPLS routers run MPLS/LDP.

```text
CE = normal IP routing
PE/P = MPLS/LDP
```

---

### Misunderstanding 5: LDP neighbors and IGP neighbors are the same thing

They are different neighbor relationships.

```text
OSPF/IS-IS neighbor = IGP adjacency
LDP neighbor        = label distribution session
```

In a basic directly connected MPLS core, they often exist on the same links.

But they are still separate protocols.

---

## Key Points

* LDP stands for Label Distribution Protocol.
* LDP distributes label bindings.
* A label binding maps a FEC to a label.
* In basic MPLS, the FEC is usually an IGP prefix.
* The IGP chooses the next hop.
* LDP provides the label for that next hop.
* LDP uses UDP 646 for Hellos.
* LDP uses TCP 646 for the LDP session.
* Basic discovery is for directly connected LDP peers.
* Targeted discovery is for non-directly-connected LDP peers.
* The LDP ID is written as `<router-id>:<label-space-id>`.
* Cisco IOS/IOS XE commonly uses a platform-wide label space, so the LDP ID often ends in `:0`.
* Use a stable loopback as the LDP router ID.
* The LDP router ID/transport address must be reachable.
* `mpls ip` enables MPLS/LDP on an interface.
* `show mpls ldp neighbor` verifies LDP sessions.
* `show mpls ldp bindings` verifies label bindings.
* `show mpls forwarding-table` verifies the LFIB.
* LDP is what allows a BGP-free core to forward labeled traffic across the provider network.
