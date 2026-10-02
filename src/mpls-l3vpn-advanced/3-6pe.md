# 6PE

6PE means **IPv6 Provider Edge over MPLS**.

6PE allows IPv6 sites to communicate over an existing IPv4 MPLS core.

The core does not need to run IPv6.

```text
              IPv6 site                         IPv6 site
           2001:DB8:1::/64                    2001:DB8:2::/64
                 |                                  |
                CE1                                CE2
                 |                                  |
                 | native IPv6                      | native IPv6
                 |                                  |
                PE1 -------- P1 -------- P2 ------- PE2
                       IPv4 IGP + LDP/MPLS core
```

The provider core remains IPv4/MPLS only.

The PE routers are dual-stack.

```text
CE routers: native IPv6
PE routers: IPv4 + IPv6
P routers:  IPv4 + MPLS only
```

**6PE** is separate from **6VPE**, which is covered in the next page.
- 6PE uses the **IPv6 global routing table** on PEs.
- 6VPE uses **VRFs** on the PEs (MPLS L3VPN).

6PE provides **global IPv6 reachability** over an IPv4 MPLS core.

---

## Why 6PE Exists

A provider may already have an IPv4 MPLS backbone.

The provider wants to offer IPv6 connectivity, but does not want to upgrade every P router to support IPv6.

6PE solves this by requiring IPv6 only on the PE routers.

```text
Without 6PE:

P routers must understand IPv6.

With 6PE:

P routers only switch MPLS labels.
They do not inspect the IPv6 header.
```

The core sees labeled packets.

It does not see IPv6 routing.

---

## 6PE Components

6PE uses three main pieces:

```text
1. IPv4 IGP in the provider core
2. MPLS label distribution in the provider core
3. MP-BGP IPv6 labeled unicast between PE routers
```

The IPv4 IGP provides reachability to PE loopbacks.

LDP provides transport labels to reach those PE loopbacks.

MP-BGP advertises IPv6 routes between PE routers.

MP-BGP also advertises a label for each IPv6 prefix.

---

## 6PE Is Not a VPN

This is one of the **most important** points.

6PE uses MPLS, but it is not an MPLS L3VPN service.

```text
6PE:

IPv6 global table
No VRF
No RD
No RT
No VPNv6 route
No customer separation

6VPE:

IPv6 VRF table
RD
RT
VPNv6 route
Per-customer separation
```

6PE is closer to global IPv6 routing carried across an IPv4 MPLS backbone.

6VPE is IPv6 MPLS L3VPN.

---

## Control Plane

The PE routers form MP-BGP sessions over IPv4.

The BGP session is sourced from an IPv4 loopback.

```text
PE1 loopback: 1.1.1.1/32
PE2 loopback: 4.4.4.4/32
```

The BGP TCP session is IPv4.

```text
PE1 1.1.1.1 ---------------- PE2 4.4.4.4
              IPv4 TCP/179
```

However, inside that IPv4 BGP session, the PE routers exchange IPv6 routes.

```text
BGP session transport: IPv4
BGP address family:    IPv6
BGP route type:        IPv6 labeled unicast
```

In Cisco IOS XE, this is configured under the IPv6 address family with `send-label`.

```text
router bgp 65000
 address-family ipv6 unicast
  neighbor 4.4.4.4 activate
  neighbor 4.4.4.4 send-label
```

The `send-label` command is critical.

Without it, BGP does not advertise MPLS labels with the IPv6 routes.

---

## IPv4-Mapped IPv6 Next Hop

A 6PE route is an IPv6 route, but the next hop points to an IPv4 PE loopback.

BGP represents the IPv4 next hop as an IPv4-mapped IPv6 address.

```text
PE2 IPv4 loopback:        4.4.4.4
IPv4-mapped IPv6 form:    ::ffff:4.4.4.4
```

Example BGP table on PE1:

```text
PE1# show bgp ipv6 unicast

     Network              Next Hop
 *>i 2001:DB8:11:11::/64  ::ffff:4.4.4.4
```

This means:

```text
To reach 2001:DB8:1::/64,
send the packet toward PE2's IPv4 loopback, 4.4.4.4.
```

PE1 does not need an IPv6 route to PE2.

PE1 needs an IPv4 route and MPLS label path to PE2's IPv4 loopback.

### IPv4-Compatible vs IPv4-Mapped Format

There are two similar-looking formats:

```text
IPv4-compatible IPv6 address:  ::4.4.4.4
 = 0000:0000:0000:0000:0000:0000:0404:0404
IPv4-mapped IPv6 address:      ::ffff:4.4.4.4
 = 0000:0000:0000:0000:0000:ffff:0404:0404
```

6PE uses the **IPv4-mapped** format.

```text
PE2 IPv4 loopback:     4.4.4.4
6PE BGP next hop:      ::ffff:4.4.4.4
```

The `ffff` part identifies the final 32 bits as an embedded IPv4 address.

---

## Label Stack

6PE uses a two-label stack.

```text
Top label:    transport label
Bottom label: 6PE / IPv6 prefix label
Payload:      IPv6 packet
```

Example:

```text
+-----------------------------+
| LDP transport label to PE2  |
+-----------------------------+
| BGP label for IPv6 prefix   |
+-----------------------------+
| IPv6 packet                 |
+-----------------------------+
```

This can seem strange at first.

6PE is not an MPLS L3VPN, but it still uses a two-label stack.

```text
Transport label:
How do I get across the IPv4 MPLS core to the egress PE?

BGP IPv6 label:
What should the egress PE do with the packet after it arrives?
```

The top label is the normal MPLS transport label.

It gets the packet from the ingress PE to the egress PE.

```text
PE1 needs to reach PE2's IPv4 loopback.
LDP provides a transport label for PE2's loopback.
```

The P routers only look at this top label.

They swap or pop the transport label as usual.

They do not need to know anything about IPv6.

The bottom label is advertised by MP-BGP along with the IPv6 route.

It tells the egress PE that the payload is an IPv6 packet for the global IPv6 table.

```text
PE2 advertises:

IPv6 prefix: 2001:DB8:1::/64
Next hop:    ::ffff:4.4.4.4
Label:       24
```

When PE1 sends traffic to that IPv6 prefix, it pushes both labels.

```text
Outer label: LDP label to reach PE2
Inner label: BGP label advertised by PE2
Payload:     IPv6 packet
```

This is similar to MPLS L3VPN forwarding, but the meaning of the inner label is different.

```text
L3VPN inner label:
Identifies a VRF or VPN route on the egress PE.

6PE inner label:
Identifies an IPv6 route or IPv6 forwarding context on the egress PE.
```

Theoretically, 6PE could use a single-label approach, but there are two main reasons why a second label is used:

### PHP Without IPv6 on the Penultimate P Router

With only one label, the label would be the transport label to reach PE2.

```text id="9y7orz"
PE1 -> P1:
Transport label
IPv6 packet
```

If PHP is used, the penultimate P router removes the transport label before sending the packet to PE2.

```text id="r7m0np"
Penultimate P router -> PE2:
IPv6 packet
```

That would expose a native IPv6 packet on the last hop.

The penultimate P router would then need to send a native IPv6 packet toward PE2.

That is exactly what 6PE is trying to avoid.

With two labels, PHP can still happen.

```text id="j7xtsy"
Before PHP:

Transport label
BGP IPv6 label
IPv6 packet

After PHP:

BGP IPv6 label
IPv6 packet
```

The penultimate P router removes only the outer transport label.

It still sends an MPLS-labeled packet to PE2.

The P router does not need IPv6 support.

### The Egress PE Needs IPv6 Label Context

The second reason is that the egress PE needs a label context that says, "this is IPv6."

MPLS does not have an IPv4 or IPv6 "EtherType" after the label stack.

When the egress PE receives a labeled packet, it uses the bottom label to decide how to process the payload.

For 6PE, MP-BGP advertises two pieces of information together:

```
IPv6 reachability information
MPLS label for the IPv6 prefix
```

Example:

```
IPv6 prefix: 2001:DB8:1::/64
Next hop:    ::ffff:4.4.4.4
Label:       24
```

When PE2 receives a packet with label 24, it knows that the payload should be processed as IPv6.

```
Received label: 24
Payload:        IPv6 packet
Action:         Pop the label and use IPv6 forwarding
```

So the inner label is not just a VPN-style service label.

In 6PE, the inner label gives the egress PE the correct IPv6 forwarding context.

```
L3VPN inner label:
Identifies a VRF or VPN route.

6PE inner label:
Identifies IPv6 forwarding context.
```

Without that bottom label, the egress PE would not receive explicit MPLS label information
telling it how to interpret the payload after the label stack.

### Key Point

The second label is not needed because 6PE has VRFs.

It is needed because the IPv6 packet is being carried across an IPv4 MPLS core.

```
L3VPN:
Inner label identifies a VRF or VPN route.

6PE:
Inner label identifies IPv6 forwarding context.
```

The egress PE receives the inner label, pops it, and processes the payload as an IPv6 packet.

So 6PE still uses two labels even though it is not an L3VPN.

---

## Forwarding Example

CE1 sends traffic to CE2.

```text
Source:      2001:DB8:1::1
Destination: 2001:DB8:2::1
```

CE1 forwards the native IPv6 packet to PE1.

```text
CE1 -> PE1:

IPv6 packet
```

PE1 performs an IPv6 route lookup.

```text
Destination 2001:DB8:2::1
Best route  2001:DB8:1::/64
Next hop    ::ffff:5.5.5.5
BGP label   24
```

PE1 resolves `4.4.4.4` in the IPv4 routing table.

```text
4.4.4.4/32 via P1
LDP label 300
```

PE1 imposes two labels.

```text
PE1 -> P1:

Label 300
Label 24
IPv6 packet
```

P1 swaps the outer label.

```text
P1 -> P2:

Label 200
Label 24
IPv6 packet
```

P2 is the penultimate hop.

P2 pops the outer label.

```text
P2 -> PE2:

Label 24
IPv6 packet
```

PE2 receives label 24.

PE2 pops label 24 and performs IPv6 forwarding.

```text
PE2 -> CE2:

IPv6 packet
```

The P routers never perform an IPv6 lookup.

---

## 6PE vs 6VPE

| Feature             | 6PE                        | 6VPE                      |
| ------------------- | -------------------------- | ------------------------- |
| Purpose             | Global IPv6 over IPv4 MPLS | IPv6 L3VPN over IPv4 MPLS |
| Customer separation | No                         | Yes                       |
| VRF                 | No                         | Yes                       |
| RD                  | No                         | Yes                       |
| RT                  | No                         | Yes                       |
| BGP address family  | IPv6 unicast with labels   | VPNv6                     |
| Route format        | IPv6 prefix                | RD + IPv6 prefix          |
| Inner label meaning | IPv6 prefix/context label  | VPN label                 |
| CE-facing routing   | Native IPv6                | IPv6 inside a VRF         |
| Core requirement    | IPv4 + MPLS                | IPv4 + MPLS               |

A simple way to remember it:

```text
6PE  = IPv6 global routing over MPLS
6VPE = IPv6 VPN routing over MPLS
```

---

## Basic Lab Topology

This page uses the following topology.

```text
CE1 ------- PE1 ------- P1 ------- P2 ------- PE2 ------- CE2

CE1 LAN: 2001:DB8:1::/64
CE2 LAN: 2001:DB8:2::/64

PE1 Loopback0: 1.1.1.1/32
PE2 Loopback0: 4.4.4.4/32

PE1-CE1 link: 2001:DB8:12:12::/64
PE2-CE2 link: 2001:DB8:21:21::/64

Provider AS: 65000
Core IGP: OSPF
Core label protocol: LDP
```

The core-facing interfaces run IPv4, OSPF, and MPLS.

The CE-facing interfaces run IPv6.

The P routers do not run IPv6.

### PE1 Configuration

Core-facing IPv4 and MPLS configuration is assumed to already be working.

```text
PE1(config)# ipv6 unicast-routing
```

Configure the CE-facing IPv6 interface.

```text
PE1(config)# interface Ethernet0/2
PE1(config-if)# description TO-CE1
PE1(config-if)# ipv6 address 2001:DB8:12:12::1/64
PE1(config-if)# no shutdown
```

Add a static route for CE1's IPv6 LAN.

```text
PE1(config)# ipv6 route 2001:DB8:1::/64 2001:DB8:12:12::10
```

Configure MP-BGP IPv6 labeled unicast toward PE2.

```text
PE1(config)# router bgp 65000
PE1(config-router)# no bgp default ipv4-unicast
PE1(config-router)# neighbor 4.4.4.4 remote-as 65000
PE1(config-router)# neighbor 4.4.4.4 update-source Loopback0
PE1(config-router)# address-family ipv6 unicast
PE1(config-router-af)# neighbor 4.4.4.4 activate
PE1(config-router-af)# neighbor 4.4.4.4 send-label
PE1(config-router-af)# redistribute static
```

The neighbor is an IPv4 neighbor, but the address family is IPv6.

This is normal for 6PE.

### PE2 Configuration

```text
PE2(config)# ipv6 unicast-routing
```

Configure the CE-facing IPv6 interface.

```text
PE2(config)# interface Ethernet0/2
PE2(config-if)# description TO-CE2
PE2(config-if)# ipv6 address 2001:DB8:21:21::1/64
PE2(config-if)# no shutdown
```

Add a static route for CE2's IPv6 LAN.

```text
PE2(config)# ipv6 route 2001:DB8:2::/64 2001:DB8:21:21::10
```

Configure MP-BGP IPv6 labeled unicast toward PE1.

```text
PE2(config)# router bgp 65000
PE2(config-router)# no bgp default ipv4-unicast
PE2(config-router)# neighbor 1.1.1.1 remote-as 65000
PE2(config-router)# neighbor 1.1.1.1 update-source Loopback0
PE2(config-router)# address-family ipv6 unicast
PE2(config-router-af)# neighbor 1.1.1.1 activate
PE2(config-router-af)# neighbor 1.1.1.1 send-label
PE2(config-router-af)# redistribute static
```

### CE1 Configuration

```text
CE1(config)# ipv6 unicast-routing
```

```text
CE1(config)# interface Ethernet0/0
CE1(config-if)# description TO-PE1
CE1(config-if)# ipv6 address 2001:DB8:12:12::10/64
CE1(config-if)# no shutdown
```

```text
CE1(config)# interface Loopback1
CE1(config-if)# ipv6 address 2001:DB8:1::1/64
```

```text
CE1(config)# ipv6 route ::/0 2001:DB8:12:12::1
```

### CE2 Configuration

```text
CE2(config)# ipv6 unicast-routing
```

```text
CE2(config)# interface Ethernet0/0
CE2(config-if)# description TO-PE2
CE2(config-if)# ipv6 address 2001:DB8:21:21::10/64
CE2(config-if)# no shutdown
```

```text
CE2(config)# interface Loopback2
CE2(config-if)# ipv6 address 2001:DB8:2::1/64
```

```text
CE2(config)# ipv6 route ::/0 2001:DB8:21:21::1
```

---

## Verification

First, verify the IPv4 MPLS core.

```text
PE1# show ip route | include 4.4.4.4
O        4.4.4.4 [110/31] via 10.1.1.2, 00:06:15, Ethernet0/0

PE1# show mpls ldp neighbor
    Peer LDP Ident: 2.2.2.2:0; Local LDP Ident 1.1.1.1:0
        TCP connection: 2.2.2.2.30514 - 1.1.1.1.646
        State: Oper; Msgs sent/rcvd: 394/396; Downstream
        Up time: 05:36:06
        LDP discovery sources:
          Ethernet0/0, Src IP addr: 10.1.1.2
        Addresses bound to peer LDP Ident:
          10.1.1.2        10.1.2.1        2.2.2.2

PE1# show mpls forwarding-table 4.4.4.4
Local      Outgoing   Prefix           Bytes Label   Outgoing   Next Hop
Label      Label      or Tunnel Id     Switched      interface
22         19         4.4.4.4/32       0             Et0/0      10.1.1.2

PE1# traceroute 4.4.4.4 source Loopback0 numeric
Type escape sequence to abort.
Tracing the route to 4.4.4.4
VRF info: (vrf in name/id, vrf out name/id)
  1 10.1.1.2 [MPLS: Label 19 Exp 0] 2 msec 1 msec 2 msec
  2 10.1.2.2 [MPLS: Label 19 Exp 0] 1 msec 1 msec 1 msec
  3 10.2.2.2 2 msec *  4 msec
```

PE1 must have IPv4 reachability to PE2's loopback.

PE1 must also have a label path to PE2's loopback.

Then verify the BGP IPv6 session.

```text
PE1# show bgp ipv6 unicast summary
BGP router identifier 1.1.1.1, local AS number 65000
BGP table version is 3, main routing table version 3

Neighbor        V           AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
4.4.4.4         4        65000       8       8        3    0    0 00:03:44        1
```

Check that the neighbor supports the IPv6 MPLS label capability.

```text
PE1# show bgp ipv6 unicast neighbors 4.4.4.4
BGP neighbor is 4.4.4.4, remote AS 65000, internal link
  BGP version 4, remote router ID 4.4.4.4
  BGP state = Established, up for 00:04:03
  Neighbor capabilities:
    Route refresh: advertised and received(new)
    Four-octets ASN Capability: advertised and received
    Address family IPv6 Unicast: advertised and received
    ipv6 MPLS Label capability: advertised and received
```

Check the IPv6 BGP table.

```text
PE1# show bgp ipv6 unicast

     Network          Next Hop            Metric LocPrf Weight Path
 *>   2001:DB8:1::/64
                      2001:DB8:12:12::10
                                                0         32768 ?
 *>i  2001:DB8:2::/64
                      ::ffff:4.4.4.4            0    100      0 ?
```

The remote IPv6 prefix should have an IPv4-mapped IPv6 next hop.

```text
::ffff:4.4.4.4
```

Check the IPv6 routing table.

```text
PE1# show ipv6 route bgp
B   2001:DB8:2::/64 [200/0]
     via 4.4.4.4%default, indirectly connected
```

The key point is that the route is learned through BGP and recursively resolved through the IPv4 default table.

Check the IPv6 CEF entry.

```text
PE1# show ipv6 cef 2001:DB8:2::/64 detail
2001:DB8:2::/64, epoch 0, flags [rib defined all labels]
  recursive via 4.4.4.4 label 24
    nexthop 10.1.1.2 Ethernet0/0 label 19-(local:22)
```

This shows the two-label operation.

```text
label 19 = transport label toward PE2
label 24 = BGP label for the IPv6 prefix
```

Finally, test end-to-end IPv6 reachability.

```text
CE1# ping 2001:DB8:2::1 source 2001:DB8:1::1
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 2001:DB8:2::1, timeout is 2 seconds:
Packet sent with a source address of 2001:DB8:1::1
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/2/3 ms
```

> This example uses static routing for simplicity.
>
> In most real networks, you would use a dynamic routing protocol like OSPFv3 for PE-CE routing.

---

## Explicit Null for 6PE

6PE may use a normal BGP-assigned label for the IPv6 prefix.

```text
PE2# show bgp ipv6 unicast 2001:DB8:2::/64
BGP routing table entry for 2001:DB8:2::/64, version 3
Paths: (1 available, best #1, table default)
  Advertised to update-groups:
     2
  Refresh Epoch 1
  Local
    2001:DB8:21:21::10 from 0.0.0.0 (4.4.4.4)
      Origin incomplete, metric 0, localpref 100, weight 32768, valid, sourced, best
      mpls labels in/out 24/nolabel
      rx pathid: 0, tx pathid: 0x0
      Updated on Jun 19 2026 07:29:29 UTC
```

PE2 assigned label `24` for `2001:DB8:2::/64`.

You can also use the reserved IPv6 Explicit Null label `2`.

```text
Label 2 = IPv6 Explicit Null
```

Here is the command.

```text
PE2(config)# router bgp 65000
PE2(config-router)# address-family ipv6 unicast
PE2(config-router-af)# label mode ?
  all-explicit-null  BGP LU All Prefix Explict-Null Config
  explicit-null      BGP LU Local Connected Explict-Null Config
```

Here is the difference.

```text
all-explicit-null:
 Advertise IPv6 Explicit Null for all BGP labeled-unicast prefixes.

explicit-null:
 Advertise IPv6 Explicit Null only for local connected BGP labeled-unicast prefixes.
```

`2001:DB8:2::/64` is not directly connected to PE2.

PE2 learns it through a static route to CE2.

```text
PE2(config)# ipv6 route 2001:DB8:2::/64 2001:DB8:21:21::10
```

So this example uses `all-explicit-null`.

```text
PE2(config)# router bgp 65000
PE2(config-router)# address-family ipv6 unicast
PE2(config-router-af)# label mode all-explicit-null
```

Verify the label again.

```text
PE2# show bgp ipv6 unicast 2001:DB8:2::/64
BGP routing table entry for 2001:DB8:2::/64, version 3
Paths: (1 available, best #1, table default)
  Advertised to update-groups:
     2
  Refresh Epoch 1
  Local
    2001:DB8:21:21::10 from 0.0.0.0 (4.4.4.4)
      Origin incomplete, metric 0, localpref 100, weight 32768, valid, sourced, best
      mpls labels in/out v6-exp-null/nolabel
      rx pathid: 0, tx pathid: 0x0
      Updated on Jun 19 2026 07:29:29 UTC
```

The label changed to `v6-exp-null`.

---

## Review

```
1. CE1 sends a native IPv6 packet to PE1.

2. PE1 looks up the IPv6 destination in the global IPv6 table.

3. The route points to an IPv4-mapped IPv6 BGP next hop.

4. PE1 extracts the IPv4 next-hop value.

5. PE1 resolves the IPv4 next hop in the IPv4 table.

6. PE1 finds an MPLS transport label to the remote PE.

7. PE1 pushes two labels:
   - Transport label
   - BGP IPv6 prefix label

8. P routers switch only the outer label.

9. The penultimate P router may pop the outer label.

10. The egress PE receives the inner label.

11. The egress PE pops the inner label.

12. The egress PE forwards the native IPv6 packet to the CE.
```

---

## Key Points

- 6PE provides global IPv6 reachability over an IPv4 MPLS core.
- The P routers do not run IPv6.
- The PE routers are dual-stack.
- The PE routers exchange IPv6 prefixes using MP-BGP over IPv4.
- The BGP next hop is an IPv4-mapped IPv6 address.
- BGP advertises a label with each IPv6 prefix.
- The ingress PE pushes two labels:
  - Transport label to reach the egress PE
  - BGP label for the IPv6 prefix
- The core switches the outer label.
- The egress PE uses the inner label to identify and forward the IPv6 packet.
- 6PE does not use VRFs, RDs, or RTs.
- 6PE is global IPv6 over MPLS.
- 6VPE is IPv6 MPLS L3VPN.

---

## References

- [RFC 4798: Connecting IPv6 Islands over IPv4 MPLS Using IPv6 Provider Edge Routers](https://datatracker.ietf.org/doc/html/rfc4798)
- [Configuring IPv6 Provider Edge over MPLS (6PE)](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9300/software/release/26-x/configuration_guide/mpls/b_26x_mpls_9300_cg/configuring_ipv6___provider_edge_over_mpls__6pe_.html)
