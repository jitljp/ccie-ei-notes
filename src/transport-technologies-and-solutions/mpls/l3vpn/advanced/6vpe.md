# 6VPE

6VPE means **IPv6 VPN Provider Edge over MPLS**.

6VPE allows a provider to offer IPv6 MPLS L3VPN service over an IPv4 MPLS core.

```text
              Customer A                         Customer A
           2001:DB8:1::/64                    2001:DB8:2::/64
                 |                                  |
                CE1                                CE2
                 |                                  |
                 | native IPv6                      | native IPv6
                 |                                  |
                PE1 -------- P1 -------- P2 ------- PE2
                       IPv4 IGP + LDP/MPLS core
```

The customer sites use IPv6.

The provider core can remain IPv4/MPLS only.

```text
CE routers: native IPv6
PE routers: IPv4 + IPv6 + VRFs
P routers:  IPv4 + MPLS only
```

6VPE is basically **MPLS L3VPN for IPv6 customers**.

If you're comfortable with MPLS L3VPNs using IPv4, you should have no problems with this.

---

## 6VPE vs 6PE

6PE and 6VPE sound similar, but they have a few significant differences.

```text
6PE:
Global IPv6 over an IPv4 MPLS core.

6VPE:
IPv6 MPLS L3VPN over an IPv4 MPLS core.
```

6PE does not use VRFs.

6VPE uses VRFs.

```text
6PE:
IPv6 global routing table
No RD
No RT
No VPNv6 route

6VPE:
IPv6 VRF routing table
RD
RT
VPNv6 route
```

In 6PE, the remote IPv6 route is carried in the normal IPv6 unicast BGP address family with a label.

In 6VPE, the remote IPv6 route is carried in the **VPNv6** BGP address family.

---

## Why 6VPE Exists

A provider may already have an MPLS L3VPN backbone for IPv4 customers.

The provider wants to add IPv6 VPN service without converting the entire core to IPv6.

6VPE allows this.

```text
Customer side:
IPv6

PE-CE side:
IPv6

Provider core:
IPv4 + MPLS

PE-PE BGP:
VPNv6 over MP-BGP
```

The P routers do not need IPv6 routes.

The P routers do not need customer VRFs.

The P routers only switch MPLS labels.

---

## 6VPE Components

6VPE uses the same major pieces as regular MPLS L3VPN.

```text
1. VRFs on the PE routers
2. RDs to make customer prefixes unique
3. RTs to control import and export
4. MP-BGP VPNv6 between PE routers
5. MPLS transport through the provider core
6. PE-CE IPv6 routing
```

The main difference is the customer route family.

```text
Regular IPv4 L3VPN:
Customer route = IPv4 prefix
BGP route type = VPNv4

6VPE:
Customer route = IPv6 prefix
BGP route type = VPNv6
```

---

## VPNv6 Routes

6VPE uses VPNv6 routes.

A VPNv6 route is an IPv6 prefix plus an RD.

```text
VPNv6 route = RD + IPv6 prefix
```

Example:

```text
Customer IPv6 prefix:
2001:DB8:2::/64

RD:
65000:2

VPNv6 route:
65000:2:2001:DB8:2::/64
```

The RD does not identify the VPN membership.

The RD makes otherwise identical IPv6 prefixes unique in BGP.

Example:

```text
Customer A:
RD 65000:1
2001:DB8:1::/64

Customer B:
RD 65000:20
2001:DB8:1::/64
```

Both customers can use the same IPv6 prefix because BGP sees different VPNv6 routes.

```text
65000:1:2001:DB8:1::/64
65000:20:2001:DB8:1::/64
```

---

## Route Targets

RTs control which VRFs import and export VPNv6 routes.

```text
RD:
Makes the route unique.

RT:
Controls VPN membership.
```

Example:

```text
Customer A RT:
1:1
```

PE1 exports CE1's route with RT `1:1`.

PE2 imports routes with RT `1:1` into its Customer A VRF.

```text
PE1 VRF CUSTOMER_A:
 export RT 1:1

PE2 VRF CUSTOMER_A:
 import RT 1:1
```

For a simple any-to-any VPN, both PE routers use the same import and export RT.

```text
route-target export 1:1
route-target import 1:1
```

---

## Control Plane

The PE routers form MP-BGP sessions over IPv4.

```text
PE1 Loopback0: 1.1.1.1/32
PE2 Loopback0: 4.4.4.4/32
```

The BGP TCP session is IPv4.

```text
PE1 1.1.1.1 ---------------- PE2 4.4.4.4
              IPv4 TCP/179
```

Inside that IPv4 BGP session, the PE routers exchange VPNv6 routes.

```text
BGP session transport: IPv4
BGP address family:    VPNv6
Customer routes:       IPv6 prefixes
```

The VPNv6 address family must be activated between the PE routers.

```text
router bgp 65000
 address-family vpnv6
  neighbor 4.4.4.4 activate
  neighbor 4.4.4.4 send-community extended
```

The `send-community extended` command is critical because RTs are extended communities.

> Like in VPNv4, the `send-community extended` command is automatically added in modern IOS XE when you `activate` the neighbor.

---

## VPNv6 Next Hop

6VPE uses an IPv4-mapped IPv6 next hop when the transport core is IPv4.

Example:

```text
PE2 IPv4 loopback:        4.4.4.4
VPNv6 BGP next hop:       ::ffff:4.4.4.4
```

This means:

```text
To reach this VPNv6 prefix,
send the packet toward PE2's IPv4 loopback.
```

The ingress PE does not need an IPv6 route to the remote PE.

It needs:

```text
1. An IPv4 route to the remote PE loopback
2. An MPLS label path to the remote PE loopback
```

---

## Label Stack

6VPE uses a two-label stack.

```text
Top label:    transport label
Bottom label: VPN label
Payload:      IPv6 packet
```

Example:

```text
+-----------------------------+
| LDP transport label to PE2  |
+-----------------------------+
| VPN label for Customer A    |
+-----------------------------+
| IPv6 packet                 |
+-----------------------------+
```

The transport label gets the packet across the IPv4 MPLS core.

The VPN label tells the egress PE how to process the customer IPv6 packet.

```text
Transport label:
How do I get to PE2?

VPN label:
Which VRF or VPN route should PE2 use?
```

This is the same basic idea as IPv4 MPLS L3VPN.

The payload is IPv6 instead of IPv4.

---

## 6VPE Packet Walk

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

PE1 receives the packet on an interface in VRF `CUSTOMER_A`.

PE1 performs an IPv6 lookup in the Customer A VRF.

```text
VRF:         CUSTOMER_A
Destination: 2001:DB8:2::1
Best route:  2001:DB8:2::/64
Next hop:    ::ffff:4.4.4.4
VPN label:   24
```

PE1 resolves PE2's IPv4 loopback in the global IPv4 table.

```text
4.4.4.4/32 via P1
Transport label: 19
```

PE1 imposes two labels.

```text
PE1 -> P1:

Label 19
Label 24
IPv6 packet
```

P1 might swap the top label according to its LFIB:

```text
P1 -> P2:

Label 18
Label 24
IPv6 packet
```

The penultimate P router may pop the outer label.

```text
P2 -> PE2:

Label 24
IPv6 packet
```

PE2 receives the VPN label.

PE2 uses the VPN label to identify the correct VRF or forwarding context.

```text
Received label: 24
VRF:            CUSTOMER_A
Payload:        IPv6 packet
```

PE2 pops the VPN label and forwards the native IPv6 packet to CE2.

```text
PE2 -> CE2:

IPv6 packet
```

The P routers never perform an IPv6 lookup.

The P routers never know about the customer VRF.

---

## Basic Lab Topology

This page uses the same addressing style as the 6PE page.

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

VRF: CUSTOMER_A
PE1 RD: 65000:1
PE2 RD: 65000:2
RT: 1:1
```

The core-facing interfaces run IPv4, OSPF, and MPLS.

The CE-facing interfaces run IPv6 inside a VRF.

The P routers do not run IPv6.

---

## PE1 Configuration

Core-facing IPv4 and MPLS configuration is assumed to already be working.

```text
PE1(config)# ipv6 unicast-routing
```

Create the VRF.

```text
PE1(config)# vrf definition CUSTOMER_A
PE1(config-vrf)# rd 65000:1
PE1(config-vrf)# address-family ipv6
PE1(config-vrf-af)# route-target export 1:1
PE1(config-vrf-af)# route-target import 1:1
```

Configure the CE-facing interface in the VRF.

```text
PE1(config)# interface Ethernet0/2
PE1(config-if)# description TO-CE1
PE1(config-if)# vrf forwarding CUSTOMER_A
PE1(config-if)# ipv6 address 2001:DB8:12:12::1/64
PE1(config-if)# no shutdown
```

Add a static route for CE1's IPv6 LAN in the VRF.

```text
PE1(config)# ipv6 route vrf CUSTOMER_A 2001:DB8:1::/64 2001:DB8:12:12::10
```

Configure MP-BGP.

```text
PE1(config)# router bgp 65000
PE1(config-router)# no bgp default ipv4-unicast
PE1(config-router)# neighbor 4.4.4.4 remote-as 65000
PE1(config-router)# neighbor 4.4.4.4 update-source Loopback0
PE1(config-router)# address-family vpnv6
PE1(config-router-af)# neighbor 4.4.4.4 activate
PE1(config-router-af)# neighbor 4.4.4.4 send-community extended
PE1(config-router-af)# exit-address-family
PE1(config-router)# address-family ipv6 vrf CUSTOMER_A
PE1(config-router-af)# redistribute static
```

Important points:

```text
address-family vpnv6:
 Used between PE routers.

address-family ipv6 vrf CUSTOMER_A:
 Used to inject IPv6 VRF routes into BGP.
```

There is no `send-label` command like in 6PE.

---

## PE2 Configuration

```text
PE2(config)# ipv6 unicast-routing
```

Create the VRF.

```text
PE2(config)# vrf definition CUSTOMER_A
PE2(config-vrf)# rd 65000:2
PE2(config-vrf)# route-target export 1:1
PE2(config-vrf)# route-target import 1:1
PE2(config-vrf)# address-family ipv6
```

Configure the CE-facing interface in the VRF.

```text
PE2(config)# interface Ethernet0/2
PE2(config-if)# description TO-CE2
PE2(config-if)# vrf forwarding CUSTOMER_A
PE2(config-if)# ipv6 address 2001:DB8:21:21::1/64
PE2(config-if)# no shutdown
```

Add a static route for CE2's IPv6 LAN in the VRF.

```text
PE2(config)# ipv6 route vrf CUSTOMER_A 2001:DB8:2::/64 2001:DB8:21:21::10
```

Configure MP-BGP.

```text
PE2(config)# router bgp 65000
PE2(config-router)# no bgp default ipv4-unicast
PE2(config-router)# neighbor 1.1.1.1 remote-as 65000
PE2(config-router)# neighbor 1.1.1.1 update-source Loopback0
PE2(config-router)# address-family vpnv6
PE2(config-router-af)# neighbor 1.1.1.1 activate
PE2(config-router-af)# neighbor 1.1.1.1 send-community extended
PE2(config-router-af)# exit-address-family
PE2(config-router)# address-family ipv6 vrf CUSTOMER_A
PE2(config-router-af)# redistribute static
```

---

## CE1 Configuration

```text
CE1(config)# ipv6 unicast-routing
```

Configure the link to PE1.

```text
CE1(config)# interface Ethernet0/0
CE1(config-if)# description TO-PE1
CE1(config-if)# ipv6 address 2001:DB8:12:12::10/64
CE1(config-if)# no shutdown
```

Configure CE1's LAN.

```text
CE1(config)# interface Loopback1
CE1(config-if)# ipv6 address 2001:DB8:1::1/64
```

Add a default route toward PE1.

```text
CE1(config)# ipv6 route ::/0 2001:DB8:12:12::1
```

---

## CE2 Configuration

```text
CE2(config)# ipv6 unicast-routing
```

Configure the link to PE2.

```text
CE2(config)# interface Ethernet0/0
CE2(config-if)# description TO-PE2
CE2(config-if)# ipv6 address 2001:DB8:21:21::10/64
CE2(config-if)# no shutdown
```

Configure CE2's LAN.

```text
CE2(config)# interface Loopback2
CE2(config-if)# ipv6 address 2001:DB8:2::1/64
```

Add a default route toward PE2.

```text
CE2(config)# ipv6 route ::/0 2001:DB8:21:21::1
```

---

## Verification

First, verify the IPv4 MPLS core.

```text
PE1# show ip route | i 4.4.4.4
O        4.4.4.4 [110/31] via 10.1.1.2, 00:02:23, Ethernet0/0

PE1# show mpls ldp neighbor 
    Peer LDP Ident: 2.2.2.2:0; Local LDP Ident 1.1.1.1:0
        TCP connection: 2.2.2.2.15400 - 1.1.1.1.646
        State: Oper; Msgs sent/rcvd: 93/92; Downstream
        Up time: 01:11:29
        LDP discovery sources:
          Ethernet0/0, Src IP addr: 10.1.1.2
        Addresses bound to peer LDP Ident:
          10.1.1.2        10.1.2.1        2.2.2.2  

PE1# show mpls forwarding-table 4.4.4.4
Local      Outgoing   Prefix           Bytes Label   Outgoing   Next Hop
Label      Label      or Tunnel Id     Switched      interface
22         19         4.4.4.4/32       0             Et0/0      10.1.1.2
```

PE1 must have IPv4 reachability to PE2's loopback.

PE1 must also have an MPLS label path to PE2's loopback.

---

## Verify the VRF

Check the VRF.

```
PE1# show run | s vrf def
vrf definition CUSTOMER_A
 rd 65000:1
 !
 address-family ipv6
  route-target export 1:1
  route-target import 1:1
 exit-address-family
```

Check the VRF IPv6 route to the local CE LAN.

```text
PE1# show ipv6 route vrf CUSTOMER_A static          
S   2001:DB8:1::/64 [1/0]
     via 2001:DB8:12:12::10
```

The local CE route must exist in the VRF before BGP can advertise it.

I used static routing for simplicity, but a dynamic routing protocol like OSPFv3 is more common.

---

## Verify VPNv6 BGP

Check the VPNv6 BGP session.

```text
PE1# show bgp vpnv6 unicast all summary  

Neighbor        V           AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
4.4.4.4         4        65000      15      16        4    0    0 00:10:51        1
```

Check the VPNv6 BGP table.

```text
PE1# show bgp vpnv6 unicast all        

     Network          Next Hop            Metric LocPrf Weight Path
Route Distinguisher: 65000:1 (default for vrf CUSTOMER_A)
 *>   2001:DB8:1::/64  2001:DB8:12:12::10
                                                0         32768 ?
 *>i  2001:DB8:2::/64  ::FFFF:4.4.4.4           0    100      0 ?
Route Distinguisher: 65000:2
 *>i  2001:DB8:2::/64  ::FFFF:4.4.4.4           0    100      0 ?
```

The remote route should appear as a VPNv6 route with PE2's RD (65000:2).

Check the details for the remote prefix.

```text
PE1# show bgp vpnv6 unicast all 2001:db8:2::/64
BGP routing table entry for [65000:1]2001:DB8:2::/64, version 4
Paths: (1 available, best #1, table CUSTOMER_A)
  Flag: 0x100
  Not advertised to any peer
  Refresh Epoch 1
  Local, imported path from [65000:2]2001:DB8:2::/64 (global)
    ::FFFF:4.4.4.4 (metric 31) (via default) from 4.4.4.4 (4.4.4.4)
      Origin incomplete, metric 0, localpref 100, valid, internal, best
      Extended Community: RT:1:1
      mpls labels in/out nolabel/23
      rx pathid: 0, tx pathid: 0x0
      Updated on Jun 20 2026 00:02:23 UTC
BGP routing table entry for [65000:2]2001:DB8:2::/64, version 3
Paths: (1 available, best #1, no table)
  Flag: 0x100
  Not advertised to any peer
  Refresh Epoch 1
  Local
    ::FFFF:4.4.4.4 (metric 31) (via default) from 4.4.4.4 (4.4.4.4)
      Origin incomplete, metric 0, localpref 100, valid, internal, best
      Extended Community: RT:1:1
      mpls labels in/out nolabel/23
      rx pathid: 0, tx pathid: 0x0
      Updated on Jun 20 2026 00:02:23 UTC
```

---

## Verify VRF Import

Check whether the remote IPv6 route was imported into the VRF.

```text
PE1# show ipv6 route vrf CUSTOMER_A bgp             

B   2001:DB8:2::/64 [200/0]
     via 4.4.4.4%default, indirectly connected
```

The route is in VRF `CUSTOMER_A`, but the next hop is resolved through the global IPv4 table.

```text
Customer route lookup:
VRF IPv6 table

Transport next-hop lookup:
Global IPv4 table
```

Check CEF.

```text
PE1# show ipv6 cef vrf CUSTOMER_A 2001:db8:2::/64 detail 
2001:DB8:2::/64, epoch 0, flags [rib defined all labels]
  recursive via 4.4.4.4 label 23
    nexthop 10.1.1.2 Ethernet0/0 label 19-(local:22)
```

This shows the two-label operation.

```text
label 19 = transport label toward PE2
label 23 = VPN label for Customer A
```

---

## End-to-End Test

CE1 and CE2 should now have reachability:

```
CE1#ping 2001:db8:2::1 source l0
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 2001:DB8:2::1, timeout is 2 seconds:
Packet sent with a source address of 2001:DB8:1::1
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/2/5 ms
```

---

## 6VPE vs IPv4 L3VPN

6VPE is not a new VPN architecture.

It is simply the IPv6 version of MPLS L3VPN, running over the same IPv4 MPLS core.

| Feature                        | IPv4 L3VPN  | 6VPE        |
| ------------------------------ | ----------- | ----------- |
| Customer route                 | IPv4 prefix | IPv6 prefix |
| PE-PE BGP AF                   | VPNv4       | VPNv6       |
| VRF                            | Yes         | Yes         |
| RD                             | Yes         | Yes         |
| RT                             | Yes         | Yes         |
| VPN label                      | Yes         | Yes         |
| Transport label                | Yes         | Yes         |
| Core requirement               | IPv4 + MPLS | IPv4 + MPLS |
| P routers know customer routes | No          | No          |

---

## 6VPE vs 6PE

| Feature             | 6PE                      | 6VPE                     |
| ------------------- | ------------------------ | ------------------------ |
| Purpose             | Global IPv6 over MPLS    | IPv6 VPN over MPLS       |
| Customer separation | No                       | Yes                      |
| VRF                 | No                       | Yes                      |
| RD                  | No                       | Yes                      |
| RT                  | No                       | Yes                      |
| BGP AF between PEs  | IPv6 unicast with labels | VPNv6                    |
| Route format        | IPv6 prefix              | RD + IPv6 prefix         |
| Inner label meaning | IPv6 forwarding context  | VRF or VPN route context |
| PE-CE routing       | Native IPv6 global table | Native IPv6 inside a VRF |
| P routers run IPv6  | No                       | No                       |

A simple way to remember it:

```text
6PE  = IPv6 over MPLS
6VPE = IPv6 VPN over MPLS
```

---

## PE-CE Routing Options

This lab uses static routes for simplicity.

```text
PE1:
ipv6 route vrf CUSTOMER_A 2001:DB8:1::/64 2001:DB8:12:12::10

CE1:
ipv6 route ::/0 2001:DB8:12:12::1
```

In real deployments, PE-CE IPv6 routing may use a dynamic protocol.

Common options include:

```text
OSPFv3
EIGRP for IPv6
IS-IS for IPv6
BGP IPv6 unicast
```

The PE learns the customer's IPv6 route in the VRF.

Then the PE advertises that route to other PEs using VPNv6.

---

## Common Failure Cases

### VPNv6 Address Family Not Activated

Problem:

```text
The PE routers have a BGP session,
but they do not exchange VPNv6 routes.
```

Check:

```text
PE1# show bgp vpnv6 unicast all summary
```

Fix:

```text
router bgp 65000
 address-family vpnv6
  neighbor 4.4.4.4 activate
  neighbor 4.4.4.4 send-community extended
```

### Missing Extended Communities

Problem:

```text
PE1 receives the VPNv6 route,
but it does not import it into the VRF.
```

The RT is missing.

Check:

```text
PE1# show bgp vpnv6 unicast all 2001:DB8:2::/64
```

Look for:

```text
Extended Community: RT:65000:100
```

Fix:

```text
router bgp 65000
 address-family vpnv6
  neighbor 4.4.4.4 send-community extended
```

### Missing IPv6 Address Family Under the VRF

Problem:

```text
The VRF exists,
but IPv6 routing is not enabled inside the VRF definition.
```

Check:

```text
show run | section vrf definition CUSTOMER_A
```

Fix:

```text
vrf definition CUSTOMER_A
 address-family ipv6
 exit-address-family
```

### Interface Not in the VRF

Problem:

```text
The PE-CE interface is in the global table instead of the customer VRF.
```

Check:

```text
PE1# show vrf
PE1# show ipv6 interface brief
PE1# show ipv6 route vrf CUSTOMER_A connected
```

Fix:

```text
interface Ethernet0/2
 vrf forwarding CUSTOMER_A
 ipv6 address 2001:DB8:12:12::1/64
```

In 6PE, the PE-CE interface is in the global routing instance.

In 6VPE, it is in the customer VRF.

### Local CE Route Not Injected into BGP

Problem:

```text
PE1 knows CE1's route in the VRF,
but PE1 does not advertise it as VPNv6.
```

Check:

```text
PE1# show ipv6 route vrf CUSTOMER_A 2001:DB8:1::/64
PE1# show bgp ipv6 unicast vrf CUSTOMER_A
PE1# show bgp vpnv6 unicast all 2001:DB8:1::/64
```

Fix:

```text
router bgp 65000
 address-family ipv6 vrf CUSTOMER_A
  redistribute static
```

If using a PE-CE routing protocol, redistribute that protocol instead.

---

## Key Points

- 6VPE provides IPv6 MPLS L3VPN service over an MPLS core.
- The customer routes are IPv6.
- The PE-PE route family is VPNv6.
- The core can remain IPv4/MPLS only.
- The P routers do not run IPv6.
- The P routers do not know customer routes.
- 6VPE uses VRFs, RDs, RTs, and VPN labels.
- A VPNv6 route is: RD + IPv6 prefix
- The RT controls import and export.
- The ingress PE pushes two labels.
  - Transport label to reach the egress PE
  - VPN label for the IPv6 VPN route
- The egress PE uses the VPN label to identify the correct IPv6 VRF forwarding context.
- 6PE is global IPv6 over MPLS.
- 6VPE is IPv6 VPN over MPLS.

---

## References

* [RFC 4659: BGP-MPLS IP Virtual Private Network (VPN) Extension for IPv6 VPN](https://datatracker.ietf.org/doc/html/rfc4659)
* [Configuring IPv6 VPN Provider Edge over MPLS (6VPE)](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9300/software/release/26-x/configuration_guide/mpls/b_26x_mpls_9300_cg/configuring_ipv6_vpn___provider_edge_over_mpls__6vpe_.html)
