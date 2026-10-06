# PE-CE Route Redistribution

This page explains route redistribution from the perspective of MPLS L3VPN PE-CE routing.

Redistribution in general is covered in the routing protocol sections.

Here, the focus is narrower:

```text
How do customer routes move between the PE-CE routing protocol and MP-BGP VPNv4?
```

In an MPLS L3VPN, redistribution is often the glue between these two worlds:

```text
Customer-facing routing protocol
Provider VPN control plane
```

The PE learns routes from the CE inside a VRF.

Then the PE must export those routes into MP-BGP VPNv4 so remote PEs can learn them.

In the reverse direction, the PE imports remote VPNv4 routes into the VRF.

Then the PE may need to advertise those routes to the local CE.

---

## Basic Topology

Use the same topology:

```text
Customer A Site 1                    Customer A Site 2

10.10.10.0/24                        11.11.11.0/24
      |                                      |
     CE1                                    CE2
      |                                      |
     PE1 -------- P1 --------- P2 --------- PE2
    1.1.1.1     2.2.2.2      3.3.3.3      4.4.4.4
```

Assume the provider core already works:

```text
IGP reachability between PE loopbacks
LDP transport labels
MP-BGP VPNv4 session between PE1 and PE2
VRF CUSTOMER_A on PE1 and PE2
Matching route targets for CUSTOMER_A
```

This page focuses only on the PE-CE route handoff.

---

## The L3VPN Redistribution Problem

A route can exist in a VRF routing table without being advertised as a VPNv4 route.

Example:

```text
PE1# show ip route vrf CUSTOMER_A 10.10.10.0

Routing Table: CUSTOMER_A
S        10.10.10.0/24 [1/0] via 192.168.1.2
```

This route exists in the VRF.

But MP-BGP VPNv4 does not automatically export every route in the VRF routing table.

The route must be present in the BGP table for the VRF address family.

```text
VRF RIB:
PE1# show ip route vrf CUSTOMER_A 10.10.10.0

Routing Table: CUSTOMER_A
Routing entry for 10.10.10.0/24
  Known via "static", distance 1, metric 0
  Redistributing via bgp 65000
  Advertised by bgp 65000
  Routing Descriptor Blocks:
  * 192.168.1.2
      Route metric is 0, traffic share count is 1

BGP VPNv4 table:
PE1# show bgp vpnv4 unicast vrf CUSTOMER_A
BGP table version is 17, local router ID is 1.1.1.1
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal, 
              r RIB-failure, S Stale, m multipath, b backup-path, f RT-Filter, 
              x best-external, a additional-path, c RIB-compressed, 
              t secondary path, L long-lived-stale,
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI validation codes: V valid, I invalid, N Not found

     Network          Next Hop            Metric LocPrf Weight Path
Route Distinguisher: 65000:1 (default for vrf CUSTOMER_A)
 *>   10.10.10.0/24    192.168.1.2              0         32768 ?
 *>i  11.11.11.0/24    4.4.4.4                  0    100      0 ?
```

The normal export path is:

```text
PE-CE route
  ↓
VRF routing table
  ↓
BGP IPv4 VRF address family
  ↓
VPNv4 export
  ↓
MP-BGP to remote PE
```

This is why redistribution under `address-family ipv4 vrf` is so important.

---

## Two Redistribution Directions

There are usually two directions to think about.

### CE Routes Toward the MPLS VPN

This direction advertises local customer routes into MP-BGP VPNv4.

Example:

```text
CE1 route: 10.10.10.0/24
```

PE1 learns the route from CE1.

Then PE1 must make the route available to BGP under the VRF address family.

```text
CE1 → PE1 VRF → BGP ipv4 vrf → VPNv4 → PE2
```

This is usually configured under BGP:

```text
router bgp 65000
 address-family ipv4 vrf CUSTOMER_A
  redistribute <PE-CE-route-source>
```

Examples:

```text
redistribute static
redistribute connected
redistribute ospf 10
redistribute eigrp 100
```

### MPLS VPN Routes Toward the CE

This direction advertises remote customer routes from MP-BGP into the local PE-CE protocol.

Example:

```text
Remote route: 11.11.11.0/24
```

PE1 imports the route from VPNv4 into `CUSTOMER_A`.

Then PE1 may need to advertise it to CE1.

```text
PE2 → VPNv4 → PE1 VRF → PE-CE protocol → CE1
```

For OSPF or EIGRP PE-CE routing, this usually means redistributing BGP into the VRF routing protocol.

Example:

```text
router ospf 10 vrf CUSTOMER_A
 redistribute bgp 65000
```

Example:

```text
router eigrp CUSTOMER_A
 address-family ipv4 vrf CUSTOMER_A autonomous-system 100
  topology base
   redistribute bgp 65000 metric 10000 100 255 1 1500
```

With PE-CE BGP, this is usually simpler.

The remote VPN route is already a BGP route in the VRF address family, so it can be advertised to the CE BGP neighbor without protocol redistribution.

---

## Static PE-CE Routing

Static routing is the simplest case.

Example on PE1:

```text
ip route vrf CUSTOMER_A 10.10.10.0 255.255.255.0 192.168.1.2
```

This installs the route into the `CUSTOMER_A` VRF.

But to advertise it to PE2 as a VPNv4 route, redistribute static routes under the BGP VRF address family.

```text
router bgp 65000
 address-family ipv4 vrf CUSTOMER_A
  redistribute static
```

The route flow is:

```text
Static VRF route
  ↓
BGP IPv4 VRF table
  ↓
VPNv4 route with RD, RT, next hop, and VPN label
```

On the CE side, the CE also needs a route back through the PE.

Example on CE1:

```text
ip route 11.11.11.0 255.255.255.0 192.168.1.1
```

Or, for a simple customer site:

```text
ip route 0.0.0.0 0.0.0.0 192.168.1.1
```

Static PE-CE routing does not dynamically advertise remote VPN routes to the CE.

You configure those routes manually on the CE or use a default route.

---

## Connected Routes

Connected routes in a VRF are not automatically exported into VPNv4.

Example:

```text
interface Ethernet0/1
 description Link to CE1
 vrf forwarding CUSTOMER_A
 ip address 192.168.1.1 255.255.255.0
```

This creates a connected route in the VRF:

```text
C        192.168.1.0/24 is directly connected, Ethernet0/1
```

To export connected routes into VPNv4:

```text
router bgp 65000
 address-family ipv4 vrf CUSTOMER_A
  redistribute connected
```

Redistributing connected routes may or may not be necessary,
but it can be a good idea to allow CE routers to ping each other's interfaces.

---

## OSPF PE-CE Redistribution

With OSPF PE-CE routing, the PE and CE form an OSPF neighbor relationship inside the VRF.

Example:

```text
router ospf 10 vrf CUSTOMER_A
 router-id 1.1.1.1
 network 192.168.1.0 0.0.0.255 area 0
```

CE1 advertises its customer routes to PE1 using OSPF.

PE1 installs those routes in the `CUSTOMER_A` VRF.

To export OSPF routes into MP-BGP VPNv4, redistribute OSPF under the BGP VRF address family.

```text
router bgp 65000
 address-family ipv4 vrf CUSTOMER_A
  redistribute ospf 10
```

In the reverse direction, PE1 must advertise remote VPN routes back into OSPF so CE1 can learn them.

```text
router ospf 10 vrf CUSTOMER_A
 redistribute bgp 65000
```

The basic two-way OSPF handoff is:

```text
CE OSPF routes → PE VRF OSPF → BGP IPv4 VRF → VPNv4
VPNv4 → BGP IPv4 VRF → PE VRF OSPF → CE
```

OSPF in MPLS L3VPN has special behavior beyond normal redistribution.

Examples:

```text
OSPF superbackbone
Domain ID
OSPF route type preservation
Down bit
Domain tag
Sham links
```

Those topics will be covered in the OSPF PE-CE pages.

For now, remember the basic route handoff:

```text
BGP exports OSPF-learned customer routes into VPNv4.
OSPF advertises imported VPNv4 routes back to the CE.
```

---

## EIGRP PE-CE Redistribution

With EIGRP PE-CE routing, the PE runs EIGRP inside the customer VRF.

Example using named EIGRP:

```text
router eigrp CUSTOMER_A
 address-family ipv4 vrf CUSTOMER_A autonomous-system 100
  network 192.168.1.0 0.0.0.255
```

CE1 advertises EIGRP routes to PE1.

PE1 installs those routes in the `CUSTOMER_A` VRF.

To export EIGRP routes into MP-BGP VPNv4:

```text
router bgp 65000
 address-family ipv4 vrf CUSTOMER_A
  redistribute eigrp 100
```

In the reverse direction, PE1 redistributes BGP into EIGRP so CE1 can learn routes from remote VPN sites.

```text
router eigrp CUSTOMER_A
 address-family ipv4 vrf CUSTOMER_A autonomous-system 100
  topology base
   redistribute bgp 65000 metric 10000 100 255 1 1500
```

When redistributing into EIGRP, a metric is required unless a default metric is configured.

Example metric values:

```text
Bandwidth: 10000 Kbit/s
Delay:     100 tens of microseconds
Reliability: 255
Load:        1
MTU:         1500
```

EIGRP L3VPN also has special behavior related to metric preservation and route type.

Remote EIGRP routes carried through the MPLS VPN should retain useful EIGRP metric information when they are reintroduced at the remote site.

EIGRP Site of Origin is also important for loop prevention in multihomed customer sites.

Those topics will be covered in the EIGRP PE-CE pages.

---

## BGP PE-CE Routing

PE-CE BGP is usually the cleanest case from a redistribution perspective.

Example:

```text
router bgp 65000
 address-family ipv4 vrf CUSTOMER_A
  neighbor 192.168.1.2 remote-as 65100
  neighbor 192.168.1.2 activate
```

CE1 advertises routes to PE1 using eBGP.

Those routes are already in the BGP IPv4 VRF table.

```text
CE EBGP route
  ↓
BGP IPv4 VRF table
  ↓
VPNv4 export
```

So you normally do not need to redistribute the CE BGP route into BGP.

It is already a BGP route.

In the reverse direction, imported VPNv4 routes are also BGP routes in the VRF address family.

The PE can advertise them to the CE BGP neighbor, subject to normal BGP policy.

```text
VPNv4 route imported into VRF
  ↓
BGP IPv4 VRF table
  ↓
Advertised to CE eBGP neighbor
```

This is one reason PE-CE BGP is common in service provider MPLS VPN designs.

However, PE-CE BGP has its own L3VPN-specific issues:

```text
Same customer AS at multiple sites
AS path loop prevention
AS override
Allowas-in
Site of Origin
Multihomed customer sites
```

Those topics will be covered in the BGP PE-CE pages.

---

## Route Targets Are Still Required

Redistribution makes routes available to BGP.

It does not replace route targets.

Example:

```text
router bgp 65000
 address-family ipv4 vrf CUSTOMER_A
  redistribute static
```

This can put the static route into the BGP VRF table.

But for the route to be imported by remote PEs, the VRF still needs an export RT.

```text
vrf definition CUSTOMER_A
 rd 65000:1
 !
 address-family ipv4
  route-target export 1:1
  route-target import 1:1
 exit-address-family
```

During export, the PE attaches the export RT.

The remote PE imports the route only if its VRF has a matching import RT.

The route flow is:

```text
Redistribution:
Makes the route eligible for VPNv4 export.

Route target export:
Attaches VPN membership information.

Route target import:
Controls whether the remote PE imports the VPNv4 route.
```

---

## Route Maps and Policy

In CCIE-level configurations, avoid uncontrolled redistribution.

Use route maps to control which routes are exported or re-advertised.

Example:

```text
ip prefix-list CUSTOMER_A_EXPORT seq 5 permit 10.10.10.0/24
ip prefix-list CUSTOMER_A_EXPORT seq 10 permit 10.10.20.0/24
!
route-map CUSTOMER_A_STATIC_TO_BGP permit 10
 match ip address prefix-list CUSTOMER_A_EXPORT
!
router bgp 65000
 address-family ipv4 vrf CUSTOMER_A
  redistribute static route-map CUSTOMER_A_STATIC_TO_BGP
```

This prevents unwanted static routes from being exported into the VPN.

Route maps can also be used to set attributes.

Examples:

```text
Set BGP communities
Set extended communities
Set local preference
Set metric
Set route tags
Filter routes
```

For PE-CE OSPF and EIGRP, route tags and Site of Origin can be important for loop prevention.

For PE-CE BGP, AS path behavior and SoO are common tools.

---

## Default Routes

Default routes are common in PE-CE designs.

A small customer site may not need every remote VPN prefix.

It may only need a default route toward the provider.

Example on CE1:

```text
ip route 0.0.0.0 0.0.0.0 192.168.1.1
```

The provider may also advertise a default route to the CE using the PE-CE protocol.

Examples:

```text
OSPF:
default-information originate

EIGRP:
redistribute static with a default route, or advertise a summary/default as appropriate

BGP:
neighbor 192.168.1.2 default-originate
```

The default route itself can also be carried through the MPLS VPN if desired.

Example:

```text
ip route vrf CUSTOMER_A 0.0.0.0 0.0.0.0 192.168.1.2
!
router bgp 65000
 address-family ipv4 vrf CUSTOMER_A
  redistribute static
```

Be careful with default routes in MPLS VPNs.

A default route can attract traffic from many remote sites.

Use policy to control where defaults are advertised and imported.

---

## Local Routes vs Remote VPN Routes

A PE can have both local PE-CE routes and remote VPNv4 routes in the same VRF.

Example on PE1:

```text
Local route from CE1:
10.10.10.0/24 via 192.168.1.2

Remote route from PE2:
11.11.11.0/24 via 4.4.4.4
```

The local route is exported into VPNv4.

The remote route is imported from VPNv4.

Do not accidentally re-advertise routes back to the site they came from.

This matters especially in multihomed designs.

Example:

```text
CE site connects to PE1 and PE2.
Route enters the VPN through PE1.
The same route returns through PE2.
PE2 advertises it back into the same site.
```

L3VPN protocols use different loop-prevention mechanisms:

```text
OSPF:
Down bit and domain tag.

EIGRP:
Site of Origin and route tagging behavior.

BGP:
AS path, AS override, allowas-in, and Site of Origin.
```

The protocol-specific pages will cover the details.

---

## Common Problems

### Route Exists in the VRF but Not in VPNv4

Likely cause:

```text
Missing redistribution into BGP under address-family ipv4 vrf.
```

Example fix for static:

```text
router bgp 65000
 address-family ipv4 vrf CUSTOMER_A
  redistribute static
```

Example fix for OSPF:

```text
router bgp 65000
 address-family ipv4 vrf CUSTOMER_A
  redistribute ospf 10
```

### Remote VPNv4 Route Exists but Is Not in the VRF

Likely cause:

```text
RT import mismatch.
```

Check:

```text
show bgp vpnv4 unicast all <prefix>
show vrf detail CUSTOMER_A
```

### Route Is in the Remote PE VRF but Not on the CE

Likely cause:

```text
Missing redistribution from BGP into the PE-CE protocol.
```

Example for OSPF:

```text
router ospf 10 vrf CUSTOMER_A
 redistribute bgp 65000
```

Example for EIGRP:

```text
router eigrp CUSTOMER_A
 address-family ipv4 vrf CUSTOMER_A autonomous-system 100
  topology base
   redistribute bgp 65000 metric 10000 100 255 1 1500
```

### EIGRP Does Not Advertise Redistributed Routes

Likely cause:

```text
Missing EIGRP redistribution metric.
```

Fix:

```text
redistribute bgp 65000 metric 10000 100 255 1 1500
```

Or configure a default metric.

### Unwanted Routes Are Exported

Likely cause:

```text
Broad redistribution without route maps.
```

Example:

```text
redistribute connected
redistribute static
```

Use route maps and prefix lists to control exports.

---

## Key Points

* PE-CE redistribution connects the customer routing protocol to MP-BGP VPNv4.
* A route in the VRF routing table is not always enough for VPNv4 export.
* Static, connected, OSPF, and EIGRP routes usually must be redistributed into BGP under `address-family ipv4 vrf`.
* PE-CE BGP routes are already in the BGP VRF address family, so no BGP-to-BGP redistribution is normally needed.
* Remote VPNv4 routes imported into a VRF may need to be redistributed into the PE-CE protocol.
* OSPF redistribution from BGP usually uses `redistribute bgp <asn> subnets`.
* EIGRP redistribution from BGP requires a metric unless a default metric is configured.
* Redistribution does not replace route targets.
* Route targets still control VPNv4 import and export.
* Use route maps to avoid uncontrolled redistribution.
* In multihomed designs, consider loop prevention mechanisms such as OSPF Down bit, domain tag, EIGRP SoO, and BGP SoO.
* Troubleshoot route flow step by step: CE, local PE VRF, BGP VRF table, VPNv4 table, remote PE VRF, remote CE.
