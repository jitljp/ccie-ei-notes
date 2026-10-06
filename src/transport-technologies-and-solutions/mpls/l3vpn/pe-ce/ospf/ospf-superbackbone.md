# OSPF Superbackbone

When OSPF is used as the PE-CE routing protocol in an MPLS L3VPN, the provider network does not behave like a normal OSPF transit network.

The P routers do not run customer OSPF, do not flood customer LSAs, and do not have customer routes.

Instead, the PE routers exchange customer OSPF routes through MP-BGP VPNv4.

OSPF treats this VPN transport as a special backbone above the customer's normal OSPF topology.

This is called the **OSPF superbackbone**.

```text
CE1 --- PE1 === MPLS VPN Superbackbone === PE2 --- CE2
```

The superbackbone is one of the most important concepts in PE-CE OSPF.

It explains why routes learned across an MPLS L3VPN often appear as OSPF inter-area routes instead of external routes.

---

## The Problem OSPF Must Solve

OSPF was designed around areas and LSAs.

In a normal OSPF network, routers in the same area share an LSDB.

ABRs connect areas to the backbone area.

```text
Area 1 --- ABR --- Area 0 --- ABR --- Area 2
```

But an MPLS L3VPN does not extend the customer's OSPF LSDB across the provider core.

Example:

```text
CE1 --- PE1 --- P1 --- P2 --- PE2 --- CE2
 |       | |                  | |      |
 --OSPF--  ----MP-BGP VPNv4---- --OSPF--
```

The provider also does not want customer LSAs in the core, as the P routers shouldn't participate in the customer's OSPF at all.

So the provider core cannot simply become part of the customer's OSPF area 0.

At the same time, the customer wants OSPF routes to move between sites.

The solution is:

```text
PE-CE link:
Normal OSPF.

PE-to-PE route exchange:
MP-BGP VPNv4.

OSPF model:
MPLS VPN superbackbone.
```

---

## What the Superbackbone Is

The OSPF superbackbone is a logical OSPF backbone created by the MPLS VPN service.

It is not a real OSPF area.

It is not OSPF area 0 in the provider core.

It is not built with OSPF adjacencies between PEs.

It is a conceptual model used by PE routers when they translate between OSPF and MP-BGP VPNv4.

```text
Customer OSPF route
  ↓
PE OSPF VRF process
  ↓
BGP IPv4 VRF table
  ↓
MP-BGP VPNv4
  ↓
Remote PE BGP IPv4 VRF table
  ↓
Remote PE OSPF VRF process
  ↓
Remote CE
```

From OSPF's point of view, the MPLS VPN backbone sits above all of the customer OSPF areas.

That is why it is called the superbackbone.

---

## What the Superbackbone Is Not

Do not think of the superbackbone as a normal OSPF link.

```text
The superbackbone is not:
- A physical link
- A virtual link
- Area 0 in the provider core
- An OSPF adjacency between PE routers
- A shared LSDB between customer sites
- A place where P routers learn customer routes
```

The PE routers exchange customer routes with MP-BGP VPNv4.

The P routers only switch labeled packets.

```text
P routers need:
IGP reachability to PE loopbacks
LDP transport labels
MPLS forwarding

P routers do not need:
Customer OSPF
Customer LSAs
Customer routes
VRFs
VPNv4 BGP
```

---

## PE Routers Act Like ABRs

In PE-CE OSPF, a PE router conceptually acts like an ABR between the customer OSPF area and the MPLS VPN superbackbone.

Example:

```text
CE1 area 0 --- PE1 === Superbackbone === PE2 --- CE2 area 0
```

PE1 learns OSPF routes from CE1.

PE1 exports them into MP-BGP VPNv4.

PE2 imports them from MP-BGP VPNv4.

PE2 advertises them into OSPF toward CE2.

Because the route crossed the superbackbone, it is usually advertised into the remote OSPF area as an inter-area route.

Example on CE2:

```text
CE2# show ip route ospf
O IA     10.10.10.0/24 [110/11] via 192.168.2.1, Ethernet0/0
```

This is a major difference from simple redistribution.

With ordinary redistribution from BGP into OSPF, you would normally expect an external route:

```text
O E2     10.10.10.0/24
```

With PE-CE OSPF in the same OSPF domain, the PE can advertise the route as inter-area instead.

That is due to the superbackbone.

---

## Why Remote Routes Often Become `O IA`

Assume this simple design:

```text
CE1 ----- PE1 === MPLS VPN === PE2 ----- CE2
    area 0                         area 0
```

CE1 advertises `10.10.10.0/24` to PE1 as an intra-area OSPF route.

```
PE1# show ip route vrf CUSTOMER_A ospf

O        10.10.10.0 [110/11] via 192.168.1.2, 00:02:09, Ethernet0/1
```

PE1 exports the route into VPNv4.

PE2 imports the route and advertises it to CE2.

On CE2, the route appears as:

```text
CE2# show ip route

O IA     10.10.10.0 [110/21] via 192.168.2.1, 00:00:53, Ethernet0/0
```

Both CE sites may be in area 0, but the route still appears as inter-area.

Reason:

```text
CE1 and CE2 are not in the same real OSPF area.
They do not share Type 1 and Type 2 LSAs.
The route crossed the MPLS VPN superbackbone.
The remote PE advertises it into the local OSPF area as a summary route.
```

The provider VPN backbone does not flood CE1's Type 1 router LSA to CE2.

Instead, PE2 originates an OSPF summary LSA into the local area.

So CE2 sees the route as inter-area.

Here is PE2's OSPF database for the CUSTOMER_A VRF:

```
PE2# show ip ospf 10 database 

            OSPF Router with ID (192.168.2.1) (Process ID 10)

                Router Link States (Area 0)

Link ID         ADV Router      Age         Seq#       Checksum Link count
11.11.11.1      11.11.11.1      93          0x80000004 0x00E869 3         
192.168.2.1     192.168.2.1     92          0x80000005 0x003CFA 2         

                Summary Net Link States (Area 0)

Link ID         ADV Router      Age         Seq#       Checksum
10.10.10.0      192.168.2.1     124         0x80000001 0x00E640
192.168.1.0     192.168.2.1     124         0x80000001 0x002FB5
```

10.10.10.0 is a Type-3 (Summary) LSA, so the route appears as O IA.

---

## LSAs Are Not Flooded Across the MPLS Core

In normal OSPF, routers in the same area share LSAs.

In PE-CE OSPF over MPLS L3VPN, customer LSAs are not flooded across the provider core.

Example:

```text
CE1 originates Type 1 LSA for 10.10.10.0/24.
PE1 receives it in the CUSTOMER_A OSPF process.
PE1 does not flood that Type 1 LSA across the MPLS core.
```

Instead, PE1 converts the route information into BGP VPNv4 information.

The VPNv4 route carries information that lets the remote PE reconstruct appropriate OSPF behavior.

This can include OSPF-related attributes such as:

```text
OSPF domain information
OSPF router ID information
OSPF route tag information
```

The remote PE then generates new OSPF LSAs toward its local CE.

Here you can see the above attributes as BGP extended communities in the VPNv4 route:

```
PE2# show bgp vpnv4 unicast all 10.10.10.0/24 
BGP routing table entry for 65000:1:10.10.10.0/24, version 5
Paths: (1 available, best #1, table CUSTOMER_A)
  Flag: 0x100
  Not advertised to any peer
  Refresh Epoch 1
  Local
    1.1.1.1 (metric 31) (via default) from 1.1.1.1 (1.1.1.1)
      Origin incomplete, metric 11, localpref 100, valid, internal, best
      Extended Community: RT:1:1 OSPF DOMAIN ID:0x0005:0x0000000A0200 
        OSPF RT:0.0.0.0:2:0 OSPF ROUTER ID:192.168.1.1:0
      mpls labels in/out nolabel/21
      rx pathid: 0, tx pathid: 0x0
      Updated on Jun 17 2026 21:34:24 UTC
```

The key point:

```text
Routes are carried as VPNv4 across the MPLS VPN.
Original LSAs are not simply flooded end to end.
```

---

## Same Area Does Not Mean Same LSDB

A common misconception is that if CE1 and CE2 both use area 0, they are in the same OSPF area.

In an MPLS L3VPN, this is not true in the normal sense.

Example:

```text
CE1 ----- PE1 === MPLS VPN === PE2 ----- CE2
    area 0                         area 0
```

CE1 and CE2 both use area 0.

But they are separated by the MPLS VPN superbackbone.

They do not form an OSPF adjacency.

They do not share Type 1 and Type 2 LSAs.

They do not calculate one SPF tree across both sites.

Each site has its own local OSPF area.

The PE routers exchange reachability through VPNv4 BGP.

```text
Same area number:
Not necessarily same LSDB across the VPN.

Same OSPF domain:
Allows routes to be treated as internal/inter-area instead of external.
```

---

## Area 0 Is Not Required Across the Provider Core

In normal OSPF, non-backbone areas must connect to area 0.

In MPLS L3VPN PE-CE OSPF, the superbackbone provides the provider-side backbone function.

This means different customer sites can connect to the provider using different OSPF areas.

Example:

```text
CE1 ------ PE1 === MPLS VPN === PE2 ------ CE2
    area 10                         area 20
```

The provider core still does not run customer OSPF.

PE1 and PE2 connect the customer OSPF areas to the VPN superbackbone.

This is why PE-CE OSPF can support customer sites that are not connected by a continuous customer-controlled area 0.

However, the normal OSPF rules still matter inside each local customer site.

---

## Superbackbone and Domain ID

The superbackbone concept is closely tied to the OSPF Domain ID.

The Domain ID helps a PE decide whether a route learned from a remote PE belongs to the same OSPF domain.

In simple terms:

```text
Same OSPF domain:
Advertise remote internal OSPF routes as inter-area routes.

Different OSPF domain:
Advertise remote OSPF routes as external routes.
```

Example with the same domain:

```text
CE2# show ip route 10.10.10.0
O IA 10.10.10.0/24 via 192.168.2.1
```

Example with a different domain:

```text
CE2# show ip route 10.10.10.0
O E2 10.10.10.0/24 via 192.168.2.1
```

> The next page addresses the OSPF domain ID.

---

## Superbackbone and OSPF Route Types

PE-CE OSPF tries to preserve OSPF route behavior where possible.

For routes from the same OSPF domain, internal OSPF routes are usually carried through the VPN and reintroduced as inter-area routes.

For external or NSSA routes, the PE may preserve external route information.

Examples of route types a CE may see:

```text
O IA
O E1
O E2
O N1
O N2
```

The exact result depends on several factors:

```text
Original route type
Domain ID
Area type
Redistribution behavior
Sham link usage
```

For this page, the main point is:

```text
The superbackbone explains why remote OSPF routes are not just
normal redistributed external routes.
```

The detailed route-type behavior is covered is a later page.

---

## Superbackbone and Loop Prevention

The superbackbone creates a route exchange path between OSPF and MP-BGP.

That means route loops are possible if routes are reintroduced incorrectly.

Example loop risk:

```text
1. PE1 learns a route from CE1.
2. PE1 advertises it into MP-BGP VPNv4.
3. PE2 imports it and advertises it into OSPF.
4. The route somehow returns to another PE through OSPF.
   → e.g., another PE connects to the same customer site.
5. That PE might try to advertise it back into MP-BGP again.
```

To prevent this, OSPF over MPLS L3VPN uses special loop prevention mechanisms.

Important mechanisms:

```text
Down Bit:
Used mainly for routes advertised from the VPN into OSPF.

Domain Tag:
Used mainly for external OSPF routes.
```

These mechanisms are covered in later pages.

---

## Superbackbone vs Sham Link

By default, routes that cross the MPLS VPN superbackbone are usually treated as inter-area routes.

This matters when the customer also has a backdoor link between sites.

Example:

```text
CE1 -------- backdoor link -------- CE2
 |                                  |
PE1 ========= MPLS VPN ============ PE2
```

If CE1 and CE2 are in the same OSPF area over the backdoor link, OSPF may prefer the backdoor path because intra-area routes are preferred over inter-area routes.

```text
O > O IA > O E1 > O E2
```

The MPLS VPN path may have a better cost, but OSPF route-type preference can still cause the backdoor link to win.

> OSPF route type takes precedence over cost.

A sham link can make the MPLS VPN path appear as an intra-area OSPF path.

That allows OSPF to compare costs more directly.

Sham links are covered in a later page.

For now, remember:

```text
Normal superbackbone behavior:
Remote routes often appear as inter-area.

Sham link behavior:
Remote routes can appear as intra-area across the VPN.
```

---

## Basic Configuration Review

There is special `superbackbone` configuration.

The behavior comes from using OSPF as the PE-CE protocol inside a VRF and exchanging routes through MP-BGP VPNv4.

Example on PE1:

```text
interface Ethernet0/1
 description Link to CE1
 vrf forwarding CUSTOMER_A
 ip address 192.168.1.1 255.255.255.0

router ospf 10 vrf CUSTOMER_A
 router-id 192.168.1.1
 network 192.168.1.0 0.0.0.255 area 0
 redistribute bgp 65000

router bgp 65000
 address-family ipv4 vrf CUSTOMER_A
  redistribute ospf 10
```

The important pieces are:

```text
router ospf 10 vrf CUSTOMER_A:
Runs OSPF with the CE inside the VRF.

`redistribute ospf 10` under BGP IPv4 VRF AF:
Allows CE-learned OSPF routes to be exported into VPNv4.

`redistribute bgp 65000` under OSPF VRF process:
Allows VPNv4-learned routes to be advertised to the CE through OSPF.
```

The superbackbone behavior occurs during the OSPF-to-BGP and BGP-to-OSPF translation.

---

## Verification

Start with the PE-CE OSPF adjacency.

On the CE:

```text
CE1# show ip ospf neighbor

Neighbor ID     Pri   State           Dead Time   Address         Interface
192.168.1.1       0   FULL/  -        00:00:39    192.168.1.1     Ethernet0/0
```

On the PE:

```text
PE1# show ip ospf vrf CUSTOMER_A neighbor
```

Check the OSPF process in the VRF:

```text
PE1# show ip ospf 10 neighbor            

Neighbor ID     Pri   State           Dead Time   Address         Interface
10.10.10.1        0   FULL/  -        00:00:39    192.168.1.2     Ethernet0/1
```

Check routes in the VRF:

```text
PE1#show ip route vrf CUSTOMER_A ospf

O        10.10.10.0 [110/11] via 192.168.1.2, 00:17:52, Ethernet0/1

PE2# show ip route vrf CUSTOMER_A ospf

O        11.11.11.0 [110/11] via 192.168.2.2, 00:16:22, Ethernet0/1
```

Check VPNv4 routes:

```text
PE1# show bgp vpnv4 unicast all

     Network          Next Hop            Metric LocPrf Weight Path
Route Distinguisher: 65000:1 (default for vrf CUSTOMER_A)
 *>   10.10.10.0/24    192.168.1.2             11         32768 ?
 *>i  11.11.11.0/24    4.4.4.4                 11    100      0 ?
 *>   192.168.1.0      0.0.0.0                  0         32768 ?
 *>i  192.168.2.0      4.4.4.4                  0    100      0 ?

PE2# show bgp vpnv4 unicast all

     Network          Next Hop            Metric LocPrf Weight Path
Route Distinguisher: 65000:1 (default for vrf CUSTOMER_A)
 *>i  10.10.10.0/24    1.1.1.1                 11    100      0 ?
 *>   11.11.11.0/24    192.168.2.2             11         32768 ?
 *>i  192.168.1.0      1.1.1.1                  0    100      0 ?
 *>   192.168.2.0      0.0.0.0                  0         32768 ?
```

Check what the CE learned from the remote site:

```text
CE1#show ip route ospf

O IA     11.11.11.0 [110/21] via 192.168.1.1, 00:01:41, Ethernet0/0
O IA  192.168.2.0/24 [110/11] via 192.168.1.1, 00:01:41, Ethernet0/0

CE2# show ip route ospf

      10.0.0.0/24 is subnetted, 1 subnets
O IA     10.10.10.0 [110/21] via 192.168.2.1, 00:01:59, Ethernet0/0
O IA  192.168.1.0/24 [110/11] via 192.168.2.1, 00:01:59, Ethernet0/0
```

Check the OSPF database on the CE:

```text
CE1# show ip ospf database

            OSPF Router with ID (10.10.10.1) (Process ID 1)

                Router Link States (Area 0)

Link ID         ADV Router      Age         Seq#       Checksum Link count
10.10.10.1      10.10.10.1      1213        0x80000003 0x00BF9F 3         
192.168.1.1     192.168.1.1     1204        0x80000003 0x001828 2         

                Summary Net Link States (Area 0)

Link ID         ADV Router      Age         Seq#       Checksum
11.11.11.0      192.168.1.1     147         0x80000001 0x00C95B
192.168.2.0     192.168.1.1     147         0x80000001 0x002BB9

CE2# show ip ospf database

            OSPF Router with ID (11.11.11.1) (Process ID 1)

                Router Link States (Area 0)

Link ID         ADV Router      Age         Seq#       Checksum Link count
11.11.11.1      11.11.11.1      1092        0x80000004 0x00E869 3         
192.168.2.1     192.168.2.1     1093        0x80000005 0x003CFA 2         

                Summary Net Link States (Area 0)

Link ID         ADV Router      Age         Seq#       Checksum
10.10.10.0      192.168.2.1     134         0x80000001 0x00E640
192.168.1.0     192.168.2.1     134         0x80000001 0x002FB5
```

If the remote route appears as `O IA`, that is normal same-domain superbackbone behavior.

---

## Example Route Walk

Assume CE1 advertises `10.10.10.0/24` to PE1.

```text
CE1:
10.10.10.0/24 is an intra-area OSPF route.
```

PE1 learns it through OSPF:

```text
PE1 VRF CUSTOMER_A:
O 10.10.10.0/24 via 192.168.1.2
```

PE1 redistributes it into BGP under the VRF address family:

```text
router bgp 65000
 address-family ipv4 vrf CUSTOMER_A
  redistribute ospf 10
```

PE1 exports it as a VPNv4 route:

```text
VPNv4 route:
65000:1:10.10.10.0/24

RT:
1:1

Next hop:
1.1.1.1

VPN label:
Allocated by PE1
```

PE2 receives and imports the route into `CUSTOMER_A`.

PE2 redistributes BGP into OSPF:

```text
router ospf 10 vrf CUSTOMER_A
 redistribute bgp 65000
```

CE2 learns the route:

```text
CE2# show ip route 10.10.10.0
O IA     10.10.10.0 [110/21] via 192.168.2.1, 00:01:59, Ethernet0/0
```

The route is `O IA` because it crossed the MPLS VPN superbackbone.

---

## Common Misunderstandings

### The provider core is running customer OSPF

It is not.

Only PE routers run customer OSPF with CE routers.

P routers do not participate in customer OSPF, although they might run OSPF to provide reachability within the MPLS core.

### The MPLS backbone is area 0

Not exactly.

The MPLS VPN acts as a superbackbone, not as a normal customer area 0 in the provider core.

### Type 1 and Type 2 LSAs are flooded between sites

They are not.

Routes are carried through VPNv4 BGP.

The remote PE generates new OSPF LSAs toward its local CE.

### Same area number means same area across the VPN

Not in the normal OSPF LSDB sense.

CE1 area 0 and CE2 area 0 do not share an LSDB just because both use area 0.

The superbackbone allows for a discontiguous area 0 like this.

### BGP-to-OSPF always creates external routes

Although normal redisitrubtion creates external routes,
that is not the case with PE-CE OSPF in the same OSPF domain.

---

## Key Points

* The OSPF superbackbone is the logical backbone created by MPLS L3VPN PE-CE OSPF.
* It is not a physical link, a normal OSPF area, or an OSPF adjacency between PE routers.
* P routers do not run customer OSPF and do not know customer routes.
* PE routers translate routes between OSPF and MP-BGP VPNv4.
* Customer LSAs are not flooded across the MPLS core.
* Remote internal OSPF routes in the same OSPF domain are often advertised as inter-area routes.
* This is why remote routes often appear as `O IA` on the CE.
* Domain ID affects whether remote routes are treated as same-domain or different-domain routes.
* Down Bit and Domain Tag provide loop prevention for routes that cross the VPN.
* Sham links can change the normal superbackbone route-type behavior when customer backdoor links exist.
* There is no `superbackbone` command. The behavior comes from PE-CE OSPF inside a VRF plus MP-BGP VPNv4 route exchange.
