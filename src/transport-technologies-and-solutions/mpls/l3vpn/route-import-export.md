# Route Import and Export

MPLS L3VPNs use VRFs to keep customer routes separate on each PE.

But VRFs are local to one router.

To connect customer sites through the provider core, the PE routers must move routes between two different places:

```text
Customer VRF routing table
VPNv4 MP-BGP table
```

This process has two directions:

```text
Export:
Move a route from a local VRF into MP-BGP VPNv4.

Import:
Move a received VPNv4 route into a local VRF.
```

Route targets control this import and export process.

```text
route-target export:
Attach this RT to routes exported from the VRF.

route-target import:
Import VPNv4 routes that carry this RT.
```

---

## Basic Topology

Use the same L3VPN topology:

```text
Customer A Site 1                    Customer A Site 2

10.10.10.0/24                        11.11.11.0/24
      |                                      |
     CE1                                    CE2
      |                                      |
     PE1 -------- P1 --------- P2 --------- PE2
    1.1.1.1     2.2.2.2      3.3.3.3      4.4.4.4
```

Assume this VRF exists on both PE routers:

```text
vrf definition CUSTOMER_A
 rd 65000:1
 !
 address-family ipv4
  route-target export 1:1
  route-target import 1:1
 exit-address-family
```

The goal is simple:

```text
Routes from CE1 should reach CE2.
Routes from CE2 should reach CE1.
```

The PE routers accomplish this by exporting local VRF routes into VPNv4 and importing remote VPNv4 routes back into the correct VRF.

---

## The Route Lifecycle

For a route to move from CE1 to CE2, it follows this lifecycle:

```text
1. CE1 advertises a route to PE1.
2. PE1 installs the route in VRF CUSTOMER_A.
3. PE1 makes the route available to BGP under the VRF address family.
4. PE1 exports the route into the VPNv4 table.
5. PE1 attaches the export RT.
6. PE1 attaches a VPN label.
7. PE1 advertises the VPNv4 route to PE2.
8. PE2 receives the VPNv4 route.
9. PE2 checks the route's RTs.
10. PE2 imports the route into VRF CUSTOMER_A if an import RT matches.
11. CE2 can learn the route from PE2.
```

---

## Export Direction

Export means taking a route from a local VRF and advertising it as a VPNv4 route.

Example:

```text
CE1 LAN:
10.10.10.0/24
```

PE1 learns this route inside `CUSTOMER_A`.

```text
PE1 VRF CUSTOMER_A:
10.10.10.0/24
```

PE1 then exports it into MP-BGP VPNv4.

During export, PE1 adds several VPN-specific values:

```text
RD
RT
VPN label
BGP next hop
BGP path attributes
```

Example exported route:

```text
VPNv4 NLRI: 65000:1:10.10.10.0/24
RT:         1:1
Next hop:   1.1.1.1
VPN label:  21
```

The RD makes the route unique in the VPNv4 table.

The RT controls which remote VRFs can import the route.

The VPN label tells the egress PE how to forward packets for this route.

---

## Routes Must Enter BGP Before They Are Exported

A route being present in the VRF routing table is not always enough by itself.

MP-BGP advertises VPNv4 routes.

So the route must be present in the BGP table for that VRF address family before it can be exported as a VPNv4 route.

For example, if PE1 learns a static route in the VRF:

```text
PE1(config)# ip route vrf CUSTOMER_A 10.10.10.0 255.255.255.0 192.168.1.2
```

The route exists in the VRF routing table:

```text
PE1# show ip route vrf CUSTOMER_A 10.10.10.0

Routing Table: CUSTOMER_A
S        10.10.10.0 [1/0] via 192.168.1.2
```

But BGP will not automatically advertise every static route in the VRF.

You need to redistribute it under the VRF address family:

```text
PE1(config)# router bgp 65000
PE1(config-router)# address-family ipv4 vrf CUSTOMER_A
PE1(config-router-af)# redistribute static
```

Now the static route can enter BGP for the VRF and be exported into VPNv4.

The same concept applies to connected, OSPF, and EIGRP routes.

```text
Route source in VRF:
Static, connected, OSPF, EIGRP, or BGP

BGP VRF address family:
The route must be present here to be exported into VPNv4.
```

---

## Export with PE-CE BGP

If the PE learns the route from a CE using BGP, the route is already learned inside the VRF BGP address family.

Example:

```text
PE1(config)# router bgp 65000
PE1(config-router)# address-family ipv4 vrf CUSTOMER_A
PE1(config-router-af)# neighbor 192.168.1.2 remote-as 65100
PE1(config-router-af)# neighbor 192.168.1.2 activate
```

CE1 advertises `10.10.10.0/24` to PE1.

PE1 learns it in the VRF BGP table.

```text
PE1# show bgp ipv4 unicast vrf CUSTOMER_A 10.10.10.0/24

BGP routing table entry for 10.10.10.0/24, version 5
Paths: (1 available, best #1, table CUSTOMER_A)
  65100
    192.168.1.2 from 192.168.1.2
      valid, external, best
```

Because the route is already in BGP under the VRF, it is eligible for VPNv4 export.

PE1 then advertises it as a VPNv4 route to other PE routers.

> For this reason, BGP is perhaps the simplest option for PE-CE routing from the perspective of configuration.
> More info on PE-CE routing in another section.

---

## Export with Static Routes

I am using static routing in these examples for simplicity's sake.

Example on PE1:

```text
PE1(config)# ip route vrf CUSTOMER_A 10.10.10.0 255.255.255.0 192.168.1.2
```

This installs the route in the VRF routing table.

To export it into VPNv4, redistribute static routes into the BGP VRF address family:

```text
PE1(config)# router bgp 65000
PE1(config-router)# address-family ipv4 vrf CUSTOMER_A
PE1(config-router-af)# redistribute static
```

Now PE1 can create a VPNv4 route using the VRF's RD and export RT.

Example:

```text
VPNv4 route:
65000:1:10.10.10.0/24

Export RT:
1:1

Next hop:
1.1.1.1

VPN label:
21
```

Without `redistribute static`, the route can exist in the VRF routing table but not appear in VPNv4 BGP.

---

## Export with Connected Routes

Connected routes are also not automatically exported just because they exist in a VRF.

Example interface:

```text
interface Ethernet0/1
 vrf forwarding CUSTOMER_A
 ip address 192.168.1.1 255.255.255.0
```

PE1 has a connected route in the VRF:

```text
C        192.168.1.0/24 is directly connected, Ethernet0/1
```

To advertise this connected route as a VPN route, redistribute connected under the VRF address family:

```text
router bgp 65000
 address-family ipv4 vrf CUSTOMER_A
  redistribute connected
```

---

## Route-Target Export

The `route-target export` command controls which RTs are attached to routes exported from the VRF.

Example:

```text
vrf definition CUSTOMER_A
 rd 65000:1
 !
 address-family ipv4
  route-target export 1:1
  route-target import 1:1
 exit-address-family
```

When PE1 exports a route from `CUSTOMER_A`, it attaches RT `1:1`.

Example:

```text
Route in VRF:
10.10.10.0/24

Exported VPNv4 route:
65000:1:10.10.10.0/24

Attached RT:
1:1
```

The export RT does not decide what PE1 imports.

It decides what RT is attached to routes leaving PE1's VRF.

Remote PEs compare that RT to their local import RTs.

---

## Multiple Export RTs

A VRF can attach more than one RT to exported routes.

Example:

```text
vrf definition CUSTOMER_A
 rd 65000:1
 !
 address-family ipv4
  route-target export 1:1
  route-target export 1:100
  route-target import 1:1
 exit-address-family
```

Routes exported from this VRF carry both RTs:

```text
RT:1:1
RT:1:100
```

A remote VRF can import the route if it imports either matching RT.

Example:

```text
Remote VRF imports 1:1:
Route is imported.

Remote VRF imports 1:100:
Route is imported.

Remote VRF imports neither:
Route is not imported.
```

Multiple export RTs are useful for shared services and extranet designs.

---

## Import Direction

Import means taking a received VPNv4 route and installing it into a local VRF.

Example received by PE2:

```text
VPNv4 NLRI: 65000:1:10.10.10.0/24
RT:         1:1
Next hop:   1.1.1.1
VPN label:  21
```

PE2 checks its local VRFs.

```text
vrf definition CUSTOMER_A
 address-family ipv4
  route-target import 1:1
```

Because the route carries RT `1:1`, PE2 imports the route into `CUSTOMER_A`.

Inside the VRF, the route appears as a normal IPv4 route:

```text
PE2# show ip route vrf CUSTOMER_A 10.10.10.0

Routing Table: CUSTOMER_A
B        10.10.10.0 [200/0] via 1.1.1.1, 00:01:12
```

The RD is not shown in the VRF routing table.

The RD belongs to the VPNv4 NLRI.

After import, the route is just an IPv4 route inside the VRF.

---

## Route-Target Import

The `route-target import` command controls which VPNv4 routes are imported into the VRF.

Example:

```text
vrf definition CUSTOMER_A
 rd 65000:1
 !
 address-family ipv4
  route-target export 1:1
  route-target import 1:1
 exit-address-family
```

This means:

```text
Import any received VPNv4 route that carries RT 1:1.
```

It does not mean:

```text
Import routes with RD 1:1.
```

RD and RT are different.

The RD makes the route unique.

The RT controls import and export.

---

## Multiple Import RTs

A VRF can import multiple RTs.

Example:

```text
vrf definition CUSTOMER_A
 rd 65000:1
 !
 address-family ipv4
  route-target export 1:1
  route-target import 1:1
  route-target import 1:999
 exit-address-family
```

This VRF imports routes that carry either RT:

```text
RT 1:1
RT 1:999
```

This is useful when one VRF needs to receive routes from multiple VPNs or from a shared-services VRF.

Example:

```text
CUSTOMER_A imports:
1:1     Customer A private routes
1:999   Shared services routes
```

The route only needs one matching RT to be imported.

---

## `route-target both`

A common simple configuration uses the same RT for import and export.

Instead of this:

```text
route-target export 1:1
route-target import 1:1
```

You can use this:

```text
route-target both 1:1
```

Example:

```text
vrf definition CUSTOMER_A
 rd 65000:1
 !
 address-family ipv4
  route-target both 1:1
 exit-address-family
```

In the running configuration, IOS XE will show the separate commands:

```text
route-target export 1:1
route-target import 1:1
```

For simple full-mesh customer VPNs, `route-target both` is common.

---

## Full-Mesh VPN Example

A simple customer VPN usually imports and exports the same RT on every PE.

Example:

```text
PE1 CUSTOMER_A:
 route-target export 1:1
 route-target import 1:1

PE2 CUSTOMER_A:
 route-target export 1:1
 route-target import 1:1

PE3 CUSTOMER_A:
 route-target export 1:1
 route-target import 1:1
```

Result:

```text
Every site exports routes with RT 1:1.
Every site imports routes with RT 1:1.
All sites learn all other site routes.
```

---

## Asymmetric Import and Export RTs

The import and export RTs do not have to be the same on one PE.

Example:

```text
PE1 CUSTOMER_A:
 route-target export 1:1
 route-target import 2:2

PE2 CUSTOMER_A:
 route-target export 2:2
 route-target import 1:1
```

Result:

```text
PE1 exports routes with RT 1:1.
PE2 imports RT 1:1.

PE2 exports routes with RT 2:2.
PE1 imports RT 2:2.
```

The VPN still works because each PE imports the RT exported by the other PE.

This is useful for understanding that the RT value does not have to be identical everywhere.

What matters is the relationship between exported RTs and imported RTs.

---

## Hub-and-Spoke VPN Example

Route targets can be used to create hub-and-spoke behavior.

In this design, spokes should reach the hub, but spokes should not learn routes from other spokes.

Example:

```text
          Hub
           |
        PE-HUB
        /     \
     PE-S1   PE-S2
      |        |
   Spoke1   Spoke2
```

Use two RTs:

```text
Hub routes RT:    65000:100
Spoke routes RT:  65000:200
```

Hub VRF:

```text
vrf definition CUSTOMER_A_HUB
 rd 65000:100
 !
 address-family ipv4
  route-target export 65000:100
  route-target import 65000:200
 exit-address-family
```

Spoke VRF:

```text
vrf definition CUSTOMER_A_SPOKE
 rd 65000:200
 !
 address-family ipv4
  route-target export 65000:200
  route-target import 65000:100
 exit-address-family
```

Result:

```text
Hub exports hub routes with RT 65000:100.
Spokes import RT 65000:100.

Spokes export spoke routes with RT 65000:200.
Hub imports RT 65000:200.
```

The spokes do not import RT `65000:200`, so they do not import routes from other spokes.

---

## Shared Services Example

A shared-services VRF is commonly used when multiple customer VRFs need access to common services.

Examples:

```text
DNS
DHCP
Monitoring
Internet firewall
Management services
```

Assume these RTs:

```text
CUSTOMER_A private RT:        65000:100
CUSTOMER_B private RT:        65000:200
SHARED_SERVICES service RT:   65000:999
```

CUSTOMER_A:

```text
vrf definition CUSTOMER_A
 rd 65000:100
 !
 address-family ipv4
  route-target export 65000:100
  route-target import 65000:100
  route-target import 65000:999
 exit-address-family
```

CUSTOMER_B:

```text
vrf definition CUSTOMER_B
 rd 65000:200
 !
 address-family ipv4
  route-target export 65000:200
  route-target import 65000:200
  route-target import 65000:999
 exit-address-family
```

SHARED_SERVICES:

```text
vrf definition SHARED_SERVICES
 rd 65000:999
 !
 address-family ipv4
  route-target export 65000:999
  route-target import 65000:100
  route-target import 65000:200
 exit-address-family
```

Result:

```text
CUSTOMER_A imports shared-services routes.
CUSTOMER_B imports shared-services routes.
SHARED_SERVICES imports CUSTOMER_A and CUSTOMER_B routes.
CUSTOMER_A and CUSTOMER_B do not import each other's private routes.
```

---

## Local Import on the Same PE

Import and export do not only matter between different PEs.

They can also matter between VRFs on the same PE.

Example:

```text
PE1 has two VRFs:
CUSTOMER_A
SHARED_SERVICES
```

If `SHARED_SERVICES` exports RT `65000:999`, and `CUSTOMER_A` imports RT `65000:999`, PE1 can import the route locally.

The route does not have to physically travel across the MPLS core first.

Conceptually:

```text
Route in SHARED_SERVICES
  ↓
Exported with RT 65000:999
  ↓
Matched by CUSTOMER_A import RT 65000:999
  ↓
Installed in CUSTOMER_A
```

So the export and import can happen within the same router to leak routes between VRFs.

---

## Import and BGP Best Path

BGP still runs its best-path algorithm.

If multiple VPNv4 paths exist for the same VPNv4 NLRI, BGP selects the best path.

The selected path is the one normally imported into the VRF.

This is where RD design matters.

If two PEs advertise the same customer prefix with the same RD, they advertise the same VPNv4 NLRI.

Example:

```text
PE1 advertises:
65000:1:10.10.10.0/24

PE2 advertises:
65000:1:10.10.10.0/24
```

BGP treats these as different paths for the same VPNv4 route.

If the PEs use different RDs, BGP treats them as different VPNv4 routes.

Example:

```text
PE1 advertises:
1.1.1.1:100:10.10.10.0/24

PE2 advertises:
4.4.4.4:100:10.10.10.0/24
```

This can improve path visibility in larger designs, especially when route reflectors are used.

The RT still controls whether the route is imported.

The RD controls route uniqueness in the VPNv4 table.

---

## Import Map and Export Map

Basic route-target configuration imports or exports routes based only on RT membership.

For more selective control, IOS XE supports import and export maps under the VRF address family.

Example goal:

```text
Export only selected routes from CUSTOMER_A.
```

Example:

```text
ip prefix-list CUSTOMER_A_EXPORT seq 5 permit 10.10.10.0/24

route-map CUSTOMER_A_EXPORT permit 10
 match ip address prefix-list CUSTOMER_A_EXPORT

vrf definition CUSTOMER_A
 rd 65000:1
 !
 address-family ipv4
  route-target export 1:1
  route-target import 1:1
  export map CUSTOMER_A_EXPORT
 exit-address-family
```

This gives you extra control during export.

The `route-target export` command says which RT should be attached.

The `export map` can restrict or modify which routes are exported.

Similarly, an import map can filter routes during import:

```text
ip prefix-list CUSTOMER_A_IMPORT seq 5 permit 11.11.11.0/24

route-map CUSTOMER_A_IMPORT permit 10
 match ip address prefix-list CUSTOMER_A_IMPORT

vrf definition CUSTOMER_A
 rd 65000:1
 !
 address-family ipv4
  route-target export 1:1
  route-target import 1:1
  import map CUSTOMER_A_IMPORT
 exit-address-family
```

If the RT matches but the route is still not in the VRF, check import policy.

---

## `send-community extended`

Route targets are BGP extended communities.

The remote PE must receive the RT extended community to import the route into the correct VRF.

VPNv4 neighbor configuration should include extended communities:

```text
router bgp 65000
 address-family vpnv4
  neighbor 4.4.4.4 activate
  neighbor 4.4.4.4 send-community extended
```

> This command is added automatically in modern IOS XE when you `activate` the neighbor.

---

## Verification: Export Side

Start on the PE that owns the customer route.

Example route behind CE1:

```text
10.10.10.0/24
```

Check the VRF routing table:

```text
PE1# show ip route vrf CUSTOMER_A 10.10.10.0

Routing Table: CUSTOMER_A
Routing entry for 10.10.10.0/24
  Known via "static", distance 1, metric 0
  Redistributing via bgp 65000
  Advertised by bgp 65000
  Routing Descriptor Blocks:
  * 192.168.1.2
      Route metric is 0, traffic share count is 1
```

Check the VPNv4 table:

```text
PE1# show bgp vpnv4 unicast vrf CUSTOMER_A 11.11.11.0/24
BGP routing table entry for 65000:1:11.11.11.0/24, version 6
Paths: (1 available, best #1, table CUSTOMER_A)
  Not advertised to any peer
  Refresh Epoch 3
  Local
    4.4.4.4 (metric 31) (via default) from 4.4.4.4 (4.4.4.4)
      Origin incomplete, metric 0, localpref 100, valid, internal, best
      Extended Community: RT:1:1
      mpls labels in/out nolabel/21
      rx pathid: 0, tx pathid: 0x0
      Updated on Jun 16 2026 22:36:39 UTC
```

Check route advertisement:

```
PE1#show bgp vpnv4 unicast all neighbors 4.4.4.4 advertised-routes 
BGP table version is 10, local router ID is 1.1.1.1
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal, 
              r RIB-failure, S Stale, m multipath, b backup-path, f RT-Filter, 
              x best-external, a additional-path, c RIB-compressed, 
              t secondary path, L long-lived-stale,
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI validation codes: V valid, I invalid, N Not found

     Network          Next Hop            Metric LocPrf Weight Path
Route Distinguisher: 65000:1 (default for vrf CUSTOMER_A)
 *>   10.10.10.0/24    192.168.1.2              0         32768 ?
 *>   10.10.20.0/24    192.168.1.2              0         32768 ?
 *>   10.10.30.0/24    192.168.1.2              0         32768 ?

Total number of prefixes 3 
```

You want to confirm:

```text
1. The route exists in the VRF.
2. The route exists in BGP under the VRF address family.
3. The route exists in VPNv4.
4. The correct export RT is attached.
5. A VPN label is allocated.
6. The route is advertised to the remote PE or route reflector.
```

---

## Verification: Import Side

On the receiving PE, check whether the VPNv4 route was received.

```text
PE2# show bgp vpnv4 unicast all 10.10.10.0/24
BGP routing table entry for 65000:1:10.10.10.0/24, version 29
Paths: (1 available, best #1, table CUSTOMER_A)
  Not advertised to any peer
  Refresh Epoch 2
  Local
    1.1.1.1 (metric 31) (via default) from 1.1.1.1 (1.1.1.1)
      Origin incomplete, metric 0, localpref 100, valid, internal, best
      Extended Community: RT:1:1
      mpls labels in/out nolabel/21
      rx pathid: 0, tx pathid: 0x0
      Updated on Jun 16 2026 22:37:31 UTC
```

Check the local VRF import RT:

```text
PE2# show vrf detail CUSTOMER_A
VRF CUSTOMER_A (VRF Id = 1); default RD 65000:1; default VPNID <not set>
  New CLI format, supports multiple address-families
  Flags: 0x180C
  Interfaces:
    Et0/1                   
Address family ipv4 unicast (Table ID = 0x1):
  Flags: 0x0
  Export VPN route-target communities
    RT:1:1                  
  Import VPN route-target communities
    RT:1:1                  
  No import route-map
  No global export route-map
  No export route-map
  VRF label distribution protocol: not configured
  VRF label allocation mode: per-prefix
Address family ipv6 unicast not active
Address family ipv4 multicast not active
Address family ipv6 multicast not active
```

Then check whether the route was imported:

```text
PE2# show ip route vrf CUSTOMER_A 10.10.10.0

Routing Table: CUSTOMER_A
Routing entry for 10.10.10.0/24
  Known via "bgp 65000", distance 200, metric 0, type internal
  Last update from 1.1.1.1 00:08:21 ago
  Routing Descriptor Blocks:
  * 1.1.1.1 (default), from 1.1.1.1, 00:08:21 ago
      opaque_ptr 0x71DF321D1458 
      Route metric is 0, traffic share count is 1
      AS Hops 0
      MPLS label: 21
      MPLS Flags: MPLS Required
```

You want to confirm:

```text
1. PE2 received the VPNv4 route.
2. The route carries the expected RT.
3. CUSTOMER_A imports that RT.
4. No import map filters the route.
5. The route is installed in the VRF routing table.
6. The VPNv4 next hop is reachable.
```

---

## Import and Export Checklist

When troubleshooting route import and export, follow the route in order.

On the exporting PE:

```text
1. Is the route in the VRF routing table?
2. Is the route in BGP under the VRF address family?
3. Is the route exported into the VPNv4 table?
4. Does the VPNv4 route have the correct RD?
5. Does the VPNv4 route have the correct export RT?
6. Does the VPNv4 route have a VPN label?
7. Is the route advertised to the remote PE or RR?
```

On the importing PE:

```text
1. Is the VPNv4 route received?
2. Does the route carry the expected RT?
3. Does the local VRF import that RT?
4. Is there an import map filtering the route?
5. Is the VPNv4 next hop reachable?
6. Is the route installed in the VRF routing table?
7. Is CEF programmed with the correct label stack?
```

Separate control-plane import/export problems from data-plane forwarding problems.

If the route is not in the VRF, troubleshoot import/export.

If the route is in the VRF but traffic fails, troubleshoot labels, CEF, LSP reachability, and MTU.

---

## Key Points

* Export moves a route from a local VRF into MP-BGP VPNv4.
* Import moves a received VPNv4 route into a local VRF.
* `route-target export` attaches RTs to exported VPNv4 routes.
* `route-target import` tells the VRF which RTs to import.
* The RD makes the VPNv4 route unique.
* The RT controls VPN membership.
* A route must be present in BGP under the VRF address family before it can be exported into VPNv4.
* Static and connected routes usually need redistribution under `address-family ipv4 vrf` before export.
* PE-CE BGP routes are already learned under the VRF BGP address family.
* A route can carry multiple RTs.
* A VRF can import multiple RTs.
* `route-target both` configures both import and export for the same RT.
* Matching RDs are not required for import.
* Matching RTs are required for import.
* Route targets are BGP extended communities.
* VPNv4 neighbors must send extended communities so RTs are carried.
* Import maps and export maps can add another policy layer.
* If a route exists in VPNv4 but not in the VRF, check RTs, extended communities, import maps, and next-hop reachability.
* If a route exists in the VRF but not in VPNv4, check BGP VRF redistribution, export RTs, export maps, and RD configuration.
