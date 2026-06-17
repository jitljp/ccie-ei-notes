# MP-BGP VPNv4

MPLS L3VPNs use **MP-BGP** (Multiprotocol BGP) to exchange customer VPN routes between PE routers.

Normal IPv4 BGP carries IPv4 unicast routes.

MPLS L3VPNs use the **VPNv4 address family**.

```text
IPv4 unicast BGP:
Carries normal IPv4 routes.

VPNv4 MP-BGP:
Carries MPLS L3VPN routes.
```

VPNv4 routes are exchanged between PE routers.

---

## Why MP-BGP Is Needed

In an MPLS L3VPN, customer routes live in VRFs on the PE routers.

```text
CE1 --- PE1 === MPLS Core === PE2 --- CE2
```

CE1 advertises customer routes to PE1.

CE2 advertises customer routes to PE2.

The PE routers need to exchange those customer routes across the provider network.

But there are two problems:

```text
1. Different customers can use overlapping IPv4 prefixes.
2. The receiving PE must know which VRF should import the route.
```

Normal IPv4 BGP does not solve this by itself.

MP-BGP VPNv4 solves it by carrying:

```text
RD + IPv4 prefix
RT extended communities
VPN label
BGP next hop
```

---

## What VPNv4 Carries

A VPNv4 route is not just a normal IPv4 route.

It includes an RD, which makes the route unique even if multiple customers use the same IPv4 prefix.

```text
RD + IPv4 prefix = VPNv4 NLRI
```

Example:

```text
RD:          65000:1
IPv4 prefix: 10.10.10.0/24

VPNv4 NLRI: 65000:1:10.10.10.0/24
```

The VPNv4 route also carries attributes.

Important attributes include:

```text
Next hop
Route targets
VPN label
BGP path attributes
```

Example:

```text
VPNv4 route: 65000:1:10.10.10.0/24
Next hop:    1.1.1.1
RT:          1:1
Label:       18
```

- The RD makes the IPv4 route a unique VPNv4 route even with overlapping customer prefixes.
- The RT controls import/export into/out of VRFs.
- The VPN label is used by the egress PE to forward the packet.

---

## MP-BGP Is a PE-to-PE Control Plane

P routers do not need MP-BGP VPNv4.

Only PE routers need to exchange VPN routes.

```text
CE1 --- PE1 --- P1 --- P2 --- PE2 --- CE2
       MP-BGP ============== MP-BGP
```

The MP-BGP session is usually built between PE loopbacks.

Example:

```text
PE1 Loopback0: 1.1.1.1
PE2 Loopback0: 4.4.4.4
```

PE1 and PE2 form an IBGP VPNv4 session.

```text
PE1# show bgp vpnv4 unicast all summary
BGP router identifier 1.1.1.1, local AS number 65000
BGP table version is 4, main routing table version 4
2 network entries using 528 bytes of memory
2 path entries using 272 bytes of memory
2/2 BGP path/bestpath attribute entries using 624 bytes of memory
1 BGP extended community entries using 24 bytes of memory
0 BGP route-map cache entries using 0 bytes of memory
0 BGP filter-list cache entries using 0 bytes of memory
BGP using 1448 total bytes of memory
BGP activity 10/8 prefixes, 10/8 paths, scan interval 60 secs
2 networks peaked at 04:10:11 Jun 16 2026 UTC (01:41:54.747 ago)

Neighbor        V           AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
4.4.4.4         4        65000       6       5        4    0    0 00:01:49        1
```

The P routers only need:

```text
IGP reachability
LDP labels
MPLS forwarding
```

They do not need:

```text
VRFs
VPNv4 BGP
Customer routes
Route targets
Route distinguishers
```

---

## Basic Control Plane Pieces

MPLS L3VPN uses multiple control-plane protocols together.

```text
IGP:
Provides reachability to PE loopbacks.

LDP:
Provides transport labels to PE loopbacks.

MP-BGP VPNv4:
Carries customer VPN routes between PE routers.

PE-CE routing:
Carries customer routes between CE and PE.
```

Example:

```text
CE2 advertises 11.11.11.0/24 to PE2.
PE2 installs it in VRF CUSTOMER_A.
PE2 exports it into MP-BGP VPNv4.
PE2 advertises it to PE1.
PE1 imports it into VRF CUSTOMER_A.
CE1 can now reach 11.11.11.0/24.
```

MP-BGP carries the customer route across the provider control plane.

MPLS labels carry the traffic across the provider data plane.

---

## VPNv4 Next Hop

The BGP next hop for a VPNv4 route is usually the advertising PE's loopback.

Example:

```text
PE2 advertises:
11.11.11.0/24

BGP next hop:
4.4.4.4
```

PE1 must have reachability to `4.4.4.4`.

PE1 also needs a transport label for `4.4.4.4`.

That is why the provider core must have working IGP and LDP before L3VPN traffic can work.

```text
VPNv4 route says:
Send traffic to next hop 4.4.4.4.

LDP says:
Use this transport label to reach 4.4.4.4.
```

The VPNv4 next hop is resolved through the global routing table, not through the customer VRF.

---

## VPNv4 Route Example

CE2 has an internal LAN `11.11.11.0/24`.

PE2 has this route in its VRF:

```text
PE2#show ip route vrf CUSTOMER_A | i 11.11.11.0
S        11.11.11.0 [1/0] via 192.168.2.2
```

> For simplicity's sake, I'm using static routing in this example.

PE2 exports the route into VPNv4 BGP.

Here is the VPNv4 route:

```text
PE2# show bgp vpnv4 unicast all 11.11.11.0/24 
BGP routing table entry for 65000:1:11.11.11.0/24, version 3
Paths: (1 available, best #1, table CUSTOMER_A)
  Advertised to update-groups:
     5         
  Refresh Epoch 1
  Local
    192.168.2.2 (via vrf CUSTOMER_A) from 0.0.0.0 (4.4.4.4)
      Origin incomplete, metric 0, localpref 100, weight 32768, valid, sourced, best
      Extended Community: RT:1:1
      mpls labels in/out 24/nolabel
      rx pathid: 0, tx pathid: 0x0
      Updated on Jun 16 2026 05:56:30 UTC
```

PE2 advertises the route to PE1.

Here is the VPNv4 route on PE1:

```
PE1# show bgp vpnv4 unicast all 11.11.11.0/24
BGP routing table entry for 65000:1:11.11.11.0/24, version 7
Paths: (1 available, best #1, table CUSTOMER_A)
  Flag: 0x100
  Not advertised to any peer
  Refresh Epoch 1
  Local
    4.4.4.4 (metric 31) (via default) from 4.4.4.4 (4.4.4.4)
      Origin incomplete, metric 0, localpref 100, valid, internal, best
      Extended Community: RT:1:1
      mpls labels in/out nolabel/24
      rx pathid: 0, tx pathid: 0x0
      Updated on Jun 16 2026 05:56:32 UTC
```

If PE1 has a VRF importing RT `1:1`, PE1 imports the route into that VRF.

Inside the VRF, PE1 sees it as a normal IPv4 route:

```text
PE1# show ip route vrf CUSTOMER_A | i 11.11.11.0
B        11.11.11.0 [200/0] via 4.4.4.4, 00:01:26
```

The RD is not shown as part of the route inside the VRF routing table.
It is part of the VPNv4 control-plane route.

---

## Basic VPNv4 BGP Configuration

Assume this topology:

```text
PE1 Loopback0: 1.1.1.1
PE2 Loopback0: 4.4.4.4
Provider AS:   65000
```

Let's look at a basic IBGP neighbor configuration.

PE1:

```text
PE1(config)# router bgp 65000
PE1(config-router)# neighbor 4.4.4.4 remote-as 65000
PE1(config-router)# neighbor 4.4.4.4 update-source Loopback0
```

PE2:

```text
PE2(config)# router bgp 65000
PE2(config-router)# neighbor 1.1.1.1 remote-as 65000
PE2(config-router)# neighbor 1.1.1.1 update-source Loopback0
```

This creates the base BGP neighbor relationship, but it alone does not exchange VPNv4 routes.

You must activate the neighbor under the VPNv4 address family.

PE1:

```text
PE1(config-router)# address-family vpnv4
PE1(config-router-af)# neighbor 4.4.4.4 activate
PE1(config-router-af)# neighbor 4.4.4.4 send-community extended
```

PE2:

```text
PE2(config-router)# address-family vpnv4
PE2(config-router-af)# neighbor 1.1.1.1 activate
PE2(config-router-af)# neighbor 1.1.1.1 send-community extended
```

> As mentioned previously, the `send-community extended` command is added automatically after activating the neighbor.
> I include it here just to emphasize that it is necessary.

---

## Why `send-community extended` Matters

Route targets are BGP extended communities.

If a PE advertises a VPNv4 route without the RT extended community, the receiving PE may still receive the VPNv4 route, but it will not know which VRF should import it.

Symptoms:

```text
The route exists in the VPNv4 table.
The route is not imported into the VRF.
The route has no RT extended community.
```

Verification:

```text
PE1# show bgp vpnv4 unicast all 11.11.11.0/24       
BGP routing table entry for 65000:1:11.11.11.0/24, version 7
Paths: (1 available, best #1, table CUSTOMER_A)
  Flag: 0x100
  Not advertised to any peer
  Refresh Epoch 1
  Local
    4.4.4.4 (metric 31) (via default) from 4.4.4.4 (4.4.4.4)
      Origin incomplete, metric 0, localpref 100, valid, internal, best
      Extended Community: RT:1:1
      mpls labels in/out nolabel/24
      rx pathid: 0, tx pathid: 0x0
      Updated on Jun 16 2026 05:56:32 UTC
```

Look for the extended community:

```text
Extended Community: RT:1:1
```

---

## VPNv4 Address Family vs IPv4 Address Family

BGP can have multiple address families.

For L3VPNs, be careful about where commands are configured.

```text
address-family ipv4:
Normal IPv4 unicast routes.

address-family vpnv4:
VPNv4 routes between PE routers.

address-family ipv4 vrf CUSTOMER_A:
Customer IPv4 routes inside a specific VRF.
```

Example:

```text
router bgp 65000
 address-family vpnv4
  neighbor 4.4.4.4 activate
  neighbor 4.4.4.4 send-community extended

 address-family ipv4 vrf CUSTOMER_A
  redistribute static
```

The VPNv4 address family exchanges VPN routes between PEs.

The IPv4 VRF address family controls routes inside a customer VRF.

---

## Routes Must Be Exported from the VRF

MP-BGP VPNv4 does not automatically advertise every route on the PE.

A route must first exist in a VRF, and then the PE must export it into VPNv4 BGP.

For example, in BGP PE-CE routing:

```text
router bgp 65000
 address-family ipv4 vrf CUSTOMER_A
  neighbor 192.168.1.2 remote-as 65100
  neighbor 192.168.1.2 activate
```

Routes learned from the CE neighbor enter the VRF BGP table.

Then they can be exported into VPNv4 using the VRF's RD and export RT.

```
vrf definition CUSTOMER_A
 rd 65000:1
 !
 address-family ipv4
  route-target export 1:1
  route-target import 1:1
 exit-address-family
 ```

More details on PE-CE routing will be covered in another section.

For this section, the important point is:

```text
The route must be in the VRF first.
Then it can be exported as a VPNv4 route.
```

---

## VPNv4 Route Import

When a PE receives a VPNv4 route, it does not automatically install it into every VRF.

It checks the route's RTs.

Example received route:

```text
VPNv4 route:
65000:1:11.11.11.0/24

Extended community:
RT: 1:1
```

Local VRF on PE1:

```text
vrf definition CUSTOMER_A
 address-family ipv4
  route-target import 1:1
```

Because the route has RT `1:1`, PE1 imports it into `CUSTOMER_A`.

If the import RT does not match, the route can exist in the VPNv4 table but not appear in the VRF routing table.

This is one of the most common L3VPN troubleshooting cases.

---

## VPNv4 Route Advertisement and Labels

When a PE advertises a VPNv4 route, it also advertises a VPN label.

Example on PE1:

```text
PE1# show bgp vpnv4 unicast all labels
   Network          Next Hop      In label/Out label
Route Distinguisher: 65000:1 (CUSTOMER_A)
   10.10.10.0/24    192.168.1.2     18/nolabel
   11.11.11.0/24    4.4.4.4         nolabel/24
```

From PE1's point of view:

```text
10.10.10.0/24:
PE1 locally assigned VPN label 18.
PE1 advertises this label to other PEs.

11.11.11.0/24:
PE1 learned remote VPN label 24 from PE2.
PE1 uses this label when sending traffic to PE2.
```

The VPN label is the bottom label in the data-plane label stack.

The transport label is imposed separately based on the route to the BGP next hop.

---

## What the P Routers See

P routers do not see VPNv4 routes.

They only forward labeled packets based on the top label.

Example L3VPN packet in the core:

```text
Top label:    Transport label to PE2
Bottom label: VPN label advertised by PE2
Payload:      Customer IP packet
```

P routers use the top label.

They do not inspect the VPN label for VPN forwarding, and they do not need to know the customer prefix.

This is why the provider core can remain BGP-free for customer routes.

---

## VPNv4 Route Reflectors

In small labs, PE routers often peer directly with each other.

```text
PE1 --- VPNv4 IBGP --- PE2
```

In larger networks, PE routers often peer with route reflectors.

```text
        RR1
       /   \
     PE1   PE2
```

The route reflector reflects VPNv4 routes between PEs.

Important points:

```text
The RR must support and activate the VPNv4 address family.
The RR must send extended communities.
The RR usually does not need VRFs for the customers.
The RR does not need to be in the data path.
```

The RR reflects VPNv4 control-plane routes.

It does not forward customer traffic.

Example RR configuration:

```text
router bgp 65000
 neighbor 1.1.1.1 remote-as 65000
 neighbor 1.1.1.1 update-source Loopback0
 neighbor 4.4.4.4 remote-as 65000
 neighbor 4.4.4.4 update-source Loopback0

 address-family vpnv4
  neighbor 1.1.1.1 activate
  neighbor 1.1.1.1 send-community extended
  neighbor 1.1.1.1 route-reflector-client
  neighbor 4.4.4.4 activate
  neighbor 4.4.4.4 send-community extended
  neighbor 4.4.4.4 route-reflector-client
```

A VPNv4 RR normally carries VPNv4 routes, RDs, RTs, labels, and BGP attributes.

It does not need to import routes into customer VRFs unless it is also a PE.

---

## Next-Hop Behavior with VPNv4 Route Reflectors

For IBGP VPNv4, the BGP next hop is usually the advertising PE loopback.

When a route reflector reflects a VPNv4 route, it normally does not change the next hop.

Example:

```text
PE2 advertises route to RR:
11.11.11.0/24
Next hop: 4.4.4.4

RR reflects route to PE1:
11.11.11.0/24
Next hop: 4.4.4.4
```

This is important.

PE1 must have IGP and LDP reachability to PE2's loopback, not just to the route reflector.

If PE1 can reach the RR but not PE2, the VPNv4 session may be up, but data-plane forwarding can fail.

---

## Common Problems

### VPNv4 neighbor is not activated

The base BGP neighbor can be up, but the VPNv4 address family may not be active.

Check:

```text
show bgp vpnv4 unicast all summary
```

Fix:

```text
router bgp 65000
 address-family vpnv4
  neighbor 4.4.4.4 activate
```

---

### Missing `send-community extended`

The route target is an extended community.

If it is not sent, the remote PE cannot import the route into the correct VRF.

Fix:

```text
router bgp 65000
 address-family vpnv4
  neighbor 4.4.4.4 send-community extended
```

> This issue is rarer in modern IOS XE because the command is added automatically.

---

### Route exists in VPNv4 table but not in VRF

This usually means an RT import mismatch.

Check the route's RT:

```text
show bgp vpnv4 unicast all <prefix> detail
```

Check the VRF import RT:

```text
show vrf detail
```

The route's RT must match the VRF import RT.

---

### VPNv4 next hop is unreachable

The route may be received, but not installed or not usable.

Check:

```text
show ip route <remote-PE-loopback>
show mpls forwarding-table <remote-PE-loopback>
```

The PE must have IGP and LDP reachability to the VPNv4 next hop.

---

## Key Points

* MP-BGP VPNv4 carries customer VPN routes between PE routers.
* P routers do not need VPNv4 BGP.
* VPNv4 NLRI is the RD plus the IPv4 prefix.
* Route targets are carried as BGP extended communities.
* `send-community extended` is required so RTs are sent to VPNv4 neighbors.
* VPNv4 routes usually use the advertising PE loopback as the BGP next hop.
* The VPNv4 next hop is resolved through the provider global routing table.
* PE routers advertise VPN labels with VPNv4 routes.
* The VPN label is used by the egress PE to forward the packet in the correct VRF.
* A VPNv4 route can exist in the VPNv4 table but not be imported into a VRF.
  * In that case, check RT import/export.
* If the route is imported but traffic fails, check transport LSP reachability to the remote PE loopback.
* With route reflectors, the VPNv4 next hop usually remains the originating PE, not the route reflector.
