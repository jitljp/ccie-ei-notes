# PE-CE OSPF

OSPF can be used as the PE-CE routing protocol in an MPLS L3VPN.

From the CE router's point of view, it is just running OSPF with a neighboring router.

```text
CE1 --- PE1 === MPLS Core === PE2 --- CE2
```

The CE routers do not run MPLS.

The CE routers do not run MP-BGP VPNv4.

They only exchange customer routes with the PE routers.

The PE routers then carry those routes across the MPLS VPN using MP-BGP VPNv4.

---

## Basic Topology

Use the same topology as the earlier L3VPN examples:

```text
Customer A Site 1                    Customer A Site 2

10.10.10.0/24                        11.11.11.0/24
      |                                      |
     CE1                                    CE2
      |                                      |
     PE1 -------- P1 --------- P2 --------- PE2
    1.1.1.1     2.2.2.2      3.3.3.3      4.4.4.4
```

Customer-facing links:

```text
CE1 --- PE1:
CE1 = 192.168.1.2/24
PE1 = 192.168.1.1/24

CE2 --- PE2:
CE2 = 192.168.2.2/24
PE2 = 192.168.2.1/24
```

Customer LANs:

```text
CE1 Loopback0:
10.10.10.1/24

CE2 Loopback0:
11.11.11.1/24
```

The PE routers use VRF `CUSTOMER_A`.

```text
vrf definition CUSTOMER_A
 rd 65000:1
 !
 address-family ipv4
  route-target export 1:1
  route-target import 1:1
 exit-address-family
```

---

## What PE-CE OSPF Does

PE-CE OSPF handles routing between the customer router and the provider edge router.

Example:

```text
CE1 advertises 10.10.10.0/24 to PE1 using OSPF.

PE1 installs 10.10.10.0/24 in VRF CUSTOMER_A.

PE1 redistributes the OSPF route into BGP under the VRF address family.

PE1 exports the route into MP-BGP VPNv4.

PE2 imports the VPNv4 route into VRF CUSTOMER_A.

PE2 redistributes the BGP route into OSPF toward CE2.

CE2 learns 10.10.10.0/24 using OSPF.
```

The reverse direction works the same way for `11.11.11.0/24`.

---

## OSPF Runs Inside the VRF

On the PE router, OSPF must run inside the customer VRF.

Example on PE1:

```text
router ospf 10 vrf CUSTOMER_A
 router-id 192.168.1.1
 network 192.168.1.0 0.0.0.255 area 0
```

This OSPF process is separate from the provider core IGP.

The provider core IGP might also be OSPF, but that is a different routing domain.

Example:

```text
Provider core OSPF:
Used for PE/P loopback and core link reachability.

Customer VRF OSPF:
Used between PE and CE for customer routes.
```

The customer routes should be in the VRF.

The provider loopbacks and MPLS core routes should be in the global table.

---

## Basic PE-CE OSPF Configuration

This is a basic configuration example.

### CE1

```text
interface Loopback0
 ip address 10.10.10.1 255.255.255.0

interface Ethernet0/0
 description Link to PE1
 ip address 192.168.1.2 255.255.255.0

router ospf 10
 router-id 10.10.10.1
 network 10.10.10.0 0.0.0.255 area 0
 network 192.168.1.0 0.0.0.255 area 0
```

CE1 runs normal OSPF.

It does not know that PE1 is part of an MPLS VPN.

> The customer knows they are subscribing to a service provider's MPLS service,
> but that's not part of the CE router configuration.

### PE1

The PE interface facing CE1 is placed in the VRF:

```text
interface Ethernet0/1
 description Link to CE1
 vrf forwarding CUSTOMER_A
 ip address 192.168.1.1 255.255.255.0
```

OSPF runs inside the VRF:

```text
router ospf 10 vrf CUSTOMER_A
 router-id 192.168.1.1
 network 192.168.1.0 0.0.0.255 area 0
```

PE1 learns CE1's routes through OSPF.

But learning the routes in the VRF is not enough.

PE1 must redistribute those OSPF routes into BGP under the VRF address family so they can be exported into VPNv4:

```text
router bgp 65000
 address-family ipv4 vrf CUSTOMER_A
  redistribute ospf 10
```

This is the PE1 direction:

```text
OSPF in VRF CUSTOMER_A
  ↓
BGP IPv4 VRF address family
  ↓
VPNv4 export
```

### PE2

PE2 has the same basic structure.

The CE-facing interface is in the VRF:

```text
interface Ethernet0/1
 description Link to CE2
 vrf forwarding CUSTOMER_A
 ip address 192.168.2.1 255.255.255.0
```

OSPF runs inside the VRF:

```text
router ospf 10 vrf CUSTOMER_A
 router-id 192.168.2.1
 network 192.168.2.0 0.0.0.255 area 0
```

PE2 must also redistribute OSPF routes into BGP so CE2's routes can be exported:

```text
router bgp 65000
 address-family ipv4 vrf CUSTOMER_A
  redistribute ospf 10
```

PE2 must also advertise routes learned from remote PEs into OSPF toward CE2.

This requires redistribution from BGP into the VRF OSPF process:

```text
router ospf 10 vrf CUSTOMER_A
 redistribute bgp 65000
```

The same is needed on PE1 so routes from CE2 can be advertised to CE1:

```text
router ospf 10 vrf CUSTOMER_A
 redistribute bgp 65000
```

### CE2

Same as on CE1:

```text
interface Loopback0
 ip address 11.11.11.1 255.255.255.0

interface Ethernet0/0
 description Link to PE2
 ip address 192.168.2.2 255.255.255.0

router ospf 10
 router-id 11.11.11.1
 network 11.11.11.0 0.0.0.255 area 0
 network 192.168.2.0 0.0.0.255 area 0
```

---

## Complete Route Flow

For traffic from CE1's site to CE2's site, the route flow is:

```text
CE2:
OSPF advertises 11.11.11.0/24 to PE2.

PE2:
Learns 11.11.11.0/24 in OSPF process 10 vrf CUSTOMER_A.

PE2:
Redistributes OSPF into BGP under address-family ipv4 vrf CUSTOMER_A.

PE2:
Exports the route into MP-BGP VPNv4 with RD, RT, and VPN label.

PE1:
Receives the VPNv4 route.

PE1:
Imports the route into VRF CUSTOMER_A because the RT matches.

PE1:
Redistributes BGP into OSPF process 10 vrf CUSTOMER_A.

CE1:
Learns 11.11.11.0/24 from PE1 using OSPF.
```

The return route follows the same process in the opposite direction.

---

## PE-CE OSPF Is Not Just Normal OSPF

The PE-CE adjacency itself is normal OSPF.

But MPLS L3VPN adds special behavior when OSPF routes are carried through the provider VPN backbone.

Important L3VPN-specific topics include:

```text
OSPF superbackbone
Domain ID
OSPF route type preservation
Down Bit
Domain Tag
Sham links
```

These are the topics covered in the following pages.

---

## OSPF Superbackbone

In MPLS L3VPN with PE-CE OSPF, the provider VPN backbone acts like an OSPF **superbackbone**.

This lets OSPF routes move between customer sites through MP-BGP VPNv4.

Conceptually:

```text
CE1 area 0 --- PE1 === MPLS VPN superbackbone === PE2 --- CE2 area 0
```

The MPLS VPN backbone is not a normal OSPF area.

The P routers do not run customer OSPF.

Instead, the PE routers translate customer OSPF routes into VPNv4 routes, carry them across MP-BGP, and then translate them back into OSPF routes toward the remote CE.

This is why OSPF over L3VPN has special route type and loop prevention behavior.

---

## OSPF Domain ID

The OSPF Domain ID helps the PE decide whether a remote OSPF route belongs to the same OSPF domain.

This affects how routes are advertised back into OSPF.

In simple terms:

```text
Same OSPF domain:
Routes can be advertised as inter-area routes.

Different OSPF domain:
Routes are advertised as external routes.
```

Example:

```text
CE2# show ip route

O IA 10.10.10.0/24 [110/2] via 192.168.2.1
```

or:

```text
CE2# show ip route

O E2 10.10.10.0/24 [110/1] via 192.168.2.1
```

The Domain ID is one of the key reasons OSPF L3VPN behavior can look different from ordinary redistribution.

---

## OSPF Route Types in L3VPN

OSPF route type matters in L3VPN.

A route learned from one CE and transported across the MPLS VPN may appear on another CE as:

```text
O IA
O E1
O E2
O N1
O N2
```

The result depends on factors such as:

```text
Original route type
Area type
Domain ID
Redistribution behavior
Sham link usage
```

This is covered in detail in later pages.

For now, remember:

```text
PE-CE OSPF is designed to preserve OSPF behavior where possible,
but the MPLS VPN backbone changes how routes are represented.
```

---

## Down Bit

The Down Bit is a loop prevention mechanism for OSPF in MPLS L3VPN.

It is helpful when multiple PE routers connect to the same customer site.

When a PE advertises a route learned from MP-BGP into OSPF toward a CE, it sets the Down Bit.

If another PE later receives that same route back from a CE, it can recognize that the route originally came from the MPLS VPN backbone.

Conceptually:

```text
PE1 sends VPN-learned OSPF route to CE1.
CE1 may advertise it somewhere else.
PE2 receives the route from a CE.
PE2 sees the Down Bit.
PE2 does not reintroduce it into the VPN.
```

The Down Bit helps prevent routing loops between OSPF and MP-BGP.

---

## Domain Tag

The Domain Tag is another OSPF loop prevention mechanism used with external routes.

It is especially relevant when routes are redistributed into OSPF as external routes.

The basic idea:

```text
A PE tags external OSPF routes that came from the VPN backbone.

Another PE can use that tag to avoid sending the same route back into MP-BGP.
```

The Down Bit and Domain Tag both help prevent loops, but they apply in different OSPF route situations.

---

## Sham Links

A sham link is used when two customer sites have both:

```text
An MPLS VPN path
A direct OSPF backdoor link
```

Example:

```text
CE1 -------- backdoor link -------- CE2
 |                                  |
PE1 ========= MPLS VPN ============ PE2
```

Without a sham link, OSPF may prefer the backdoor link because intra-area OSPF routes are preferred over inter-area routes.

A sham link can make the MPLS VPN path appear as an intra-area OSPF path between the PE routers.

This allows normal OSPF cost comparison between the MPLS VPN path and the backdoor path.

Sham links are a special OSPF L3VPN tool, not something used in basic PE-CE OSPF designs.

---

## Common Problems

### OSPF adjacency does not form

Check the usual OSPF neighbor requirements:

```text
Area
Hello/dead timers
Authentication
Network type
MTU
Subnet
Passive interface
```

For the PE-CE adjacencies to come up, the PE and CE routers must agree on these settings,
so communication between the customer and service provider is essential.

### Route is in OSPF but not in VPNv4

If PE1 learns a route from CE1 through OSPF but does not advertise it to PE2, check redistribution into BGP:

```text
router bgp 65000
 address-family ipv4 vrf CUSTOMER_A
  redistribute ospf 10
```

### Route is in VPNv4 but not advertised to CE

If PE1 receives a VPNv4 route but CE1 does not learn it, check redistribution from BGP into OSPF:

```text
router ospf 10 vrf CUSTOMER_A
 redistribute bgp 65000
```

Then check the CE route table:

```text
CE1# show ip route
```

### Route type is not what you expected

If a route appears as external instead of inter-area, check the OSPF Domain ID.

If a route is not accepted back into OSPF or VPNv4, check Down Bit and Domain Tag behavior.

These topics are covered in the following pages.

---

## What Comes Next

The rest of this OSPF PE-CE section explains the L3VPN-specific OSPF behavior in more detail:

```text
OSPF Superbackbone:
How MPLS VPN acts as a backbone between customer sites.

OSPF Domain ID:
Why routes may appear as inter-area or external.

OSPF Route Types in L3VPN:
How OSPF route types are preserved or changed across the VPN.

Down Bit:
Loop prevention for OSPF routes carried through the VPN.

Domain Tag:
Loop prevention for OSPF external routes.

OSPF Sham Links:
How to handle customer backdoor links.

OSPF PE-CE Troubleshooting:
How to troubleshoot the complete OSPF L3VPN path.
```

---

## Key Points

* PE-CE OSPF runs between the CE and the PE inside the customer VRF.
* The CE runs normal OSPF and does not need to know about MPLS.
* The PE uses `router ospf <process-id> vrf <vrf-name>` for customer OSPF.
* OSPF routes learned from the CE must be redistributed into BGP under the VRF address family.
* VPNv4 routes imported from remote PEs must be redistributed from BGP into OSPF if the remote CE should learn them through OSPF.
* The provider core does not run customer OSPF.
* P routers do not know customer OSPF routes.
* The MPLS VPN backbone acts like an OSPF superbackbone.
* OSPF over L3VPN uses special mechanisms such as Domain ID, Down Bit, Domain Tag, and sham links.
* Always verify both directions: CE to PE, PE to VPNv4, VPNv4 to remote PE, and remote PE to CE.
