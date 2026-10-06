# PE-CE EIGRP

EIGRP can be used as the PE-CE routing protocol in an MPLS L3VPN.

From the CE router's point of view, this is normal EIGRP.

```text
CE1 --- PE1 === MPLS Core === PE2 --- CE2
```

The CE routers do not run MPLS.

The CE routers do not run MP-BGP VPNv4.

They only exchange customer routes with the PE routers using EIGRP.

The PE routers then carry those routes across the MPLS VPN using MP-BGP VPNv4.

---

## Basic Topology

Use the same topology as the earlier PE-CE examples:

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

Assume the provider core already works:

```text
IGP reachability between PE loopbacks
LDP transport labels
MP-BGP VPNv4 peering between PE1 and PE2
VRF CUSTOMER_A on both PEs
Matching route targets
```

This page focuses only on the EIGRP PE-CE part.

---

## What PE-CE EIGRP Does

PE-CE EIGRP exchanges customer routes between the CE router and the PE router.

Example:

```text
CE1 advertises 10.10.10.0/24 to PE1 using EIGRP.

PE1 installs 10.10.10.0/24 in VRF CUSTOMER_A.

PE1 redistributes the EIGRP route into BGP under the VRF address family.

PE1 exports the route into MP-BGP VPNv4.

PE2 imports the VPNv4 route into VRF CUSTOMER_A.

PE2 redistributes the BGP route into EIGRP.

CE2 learns 10.10.10.0/24 using EIGRP.
```

The reverse direction works the same way for `11.11.11.0/24`.

---

## EIGRP Runs Inside the VRF on the PE

On the PE router, EIGRP must run inside the customer VRF.

The customer-facing interface must be assigned to the VRF.

Example on PE1:

```text
interface Ethernet0/1
 description Link to CE1
 vrf forwarding CUSTOMER_A
 ip address 192.168.1.1 255.255.255.0
```

Then EIGRP is enabled for that VRF.

Example using named EIGRP:

```text
router eigrp CUSTOMER_A
 !
 address-family ipv4 vrf CUSTOMER_A autonomous-system 100
  network 192.168.1.0 0.0.0.255
 exit-address-family
```

This EIGRP process is separate from the provider core IGP.

```text
Provider core IGP:
Used for PE/P loopback and core link reachability.

Customer VRF EIGRP:
Used between PE and CE for customer routes.
```

The customer routes should be in the VRF.

The provider loopbacks and MPLS core routes should be in the global table.

---

## Basic PE-CE EIGRP Configuration

This is a simple configuration for CE1, PE1, PE2, and CE2.

The EIGRP autonomous system used for the customer is `100`.

```text
Customer EIGRP AS:
100
```

The provider BGP AS is `65000`.

```text
Provider BGP AS:
65000
```

---

## CE1 Configuration

CE1 runs normal EIGRP.

```text
interface Loopback0
 ip address 10.10.10.1 255.255.255.0

interface Ethernet0/0
 description Link to PE1
 ip address 192.168.1.2 255.255.255.0
 no shutdown

router eigrp 100
 network 10.10.10.0 0.0.0.255
 network 192.168.1.0 0.0.0.255
```

CE1 does not need MPLS, VRFs, RDs, RTs, VPN labels, or VPNv4 BGP.

It only runs EIGRP with PE1.

---

## PE1 Configuration

The PE interface facing CE1 is placed in the VRF.

```text
interface Ethernet0/1
 description Link to CE1
 vrf forwarding CUSTOMER_A
 ip address 192.168.1.1 255.255.255.0
 no shutdown
```

EIGRP runs inside the VRF.

```text
router eigrp CUSTOMER_A
 !
 address-family ipv4 vrf CUSTOMER_A autonomous-system 100
  network 192.168.1.0 0.0.0.255
```

PE1 learns CE1's routes through EIGRP.

But learning the routes in the VRF is not enough.

PE1 must export EIGRP-learned customer routes into BGP under the VRF address family.

```text
router bgp 65000
 address-family ipv4 vrf CUSTOMER_A
  redistribute eigrp 100
```

This is the PE1 export direction:

```text
EIGRP in VRF CUSTOMER_A
  ↓
BGP IPv4 VRF address family
  ↓
VPNv4 export
```

PE1 also needs to advertise remote VPN routes back into EIGRP so CE1 can learn routes from CE2.

```text
router eigrp CUSTOMER_A
 !
 address-family ipv4 vrf CUSTOMER_A autonomous-system 100
  topology base
   redistribute bgp 65000
```

This is the PE1 import direction:

```text
VPNv4 import
  ↓
BGP IPv4 VRF route
  ↓
EIGRP in VRF CUSTOMER_A
  ↓
CE1
```

---

## PE2 Configuration

PE2 uses the same structure.

The CE-facing interface is in the VRF.

```text
interface Ethernet0/1
 description Link to CE2
 vrf forwarding CUSTOMER_A
 ip address 192.168.2.1 255.255.255.0
 no shutdown
```

EIGRP runs inside the VRF.

```text
router eigrp CUSTOMER_A
 !
 address-family ipv4 vrf CUSTOMER_A autonomous-system 100
  network 192.168.2.0 0.0.0.255
```

PE2 exports CE2's EIGRP routes into BGP.

```text
router bgp 65000
 address-family ipv4 vrf CUSTOMER_A
  redistribute eigrp 100
```

PE2 advertises remote VPN routes back into EIGRP.

```text
router eigrp CUSTOMER_A
 !
 address-family ipv4 vrf CUSTOMER_A autonomous-system 100
  topology base
   redistribute bgp 65000
```

---

## CE2 Configuration

CE2 runs normal EIGRP.

```text
interface Loopback0
 ip address 11.11.11.1 255.255.255.0

interface Ethernet0/0
 description Link to PE2
 ip address 192.168.2.2 255.255.255.0
 no shutdown

router eigrp 100
 network 11.11.11.0 0.0.0.255
 network 192.168.2.0 0.0.0.255
```

---

## Complete Route Flow

For CE1's LAN to reach CE2, the route flow is:

```text
CE1:
Advertises 10.10.10.0/24 to PE1 using EIGRP.

PE1:
Learns 10.10.10.0/24 in EIGRP address-family ipv4 vrf CUSTOMER_A.

PE1:
Installs the route in the CUSTOMER_A VRF routing table.

PE1:
Redistributes EIGRP into BGP under address-family ipv4 vrf CUSTOMER_A.

PE1:
Exports the route into MP-BGP VPNv4 with RD, RT, next hop, and VPN label.

PE2:
Receives the VPNv4 route.

PE2:
Imports the route into VRF CUSTOMER_A because the RT matches.

PE2:
Redistributes BGP into EIGRP address-family ipv4 vrf CUSTOMER_A.

CE2:
Learns 10.10.10.0/24 using EIGRP.
```

The return route follows the same process in the opposite direction.

---

## Why a Seed Metric Is Not Required for VPN-Learned EIGRP Routes

In ordinary redistribution, BGP routes redistributed into EIGRP need a seed metric.

Example of ordinary BGP-to-EIGRP redistribution:

```text
router eigrp 100
 redistribute bgp 65000 metric 10000 100 255 1 1500
```

EIGRP PE-CE over MPLS L3VPN is different.

When an EIGRP route enters the VPN, the PE carries EIGRP information in the VPNv4 route.

This can include:

```text
EIGRP route type
EIGRP metric information
```

Because the VPNv4 route already carries the EIGRP metric information, the remote PE can redistribute the VPNv4 route back into EIGRP without manually setting a seed metric.

Example on the PE:

```text
router eigrp CUSTOMER_A
 address-family ipv4 vrf CUSTOMER_A autonomous-system 100
  topology base
   redistribute bgp 65000
```

This is one of the important differences between ordinary redistribution and EIGRP-aware L3VPN behavior.

```text
Ordinary BGP route redistributed into EIGRP:
 Seed metric required.

VPNv4 route that carries EIGRP metric information:
 Seed metric not required.
```

---

## Verification

Start with the EIGRP neighbor relationship.

On PE1:

```text
PE1# show ip eigrp vrf CUSTOMER_A neighbors
EIGRP-IPv4 VR(CUSTOMER_A) Address-Family Neighbors for AS(100)
           VRF(CUSTOMER_A)
H   Address                 Interface              Hold Uptime   SRTT   RTO  Q  Seq
                                                   (sec)         (ms)       Cnt Num
0   192.168.1.2             Et0/1                    13 00:04:41    1   100  0  5
```

Check the VRF routing table.

```text
PE1# show ip route vrf CUSTOMER_A
D        10.10.10.0 [90/3584000] via 192.168.1.2, 00:04:56, Ethernet0/1
B        11.11.11.0 [200/3584000] via 4.4.4.4, 00:02:43
```

Check the BGP VRF table.

```text
PE1# show bgp vpnv4 unicast vrf CUSTOMER_A
     Network          Next Hop            Metric LocPrf Weight Path
Route Distinguisher: 65000:1 (default for vrf CUSTOMER_A)
 *>   10.10.10.0/24    192.168.1.2        3584000         32768 ?
 *>i  11.11.11.0/24    4.4.4.4            3584000    100      0 ?
```

Check VPN labels.

```text
PE1# show bgp vpnv4 unicast all labels
   Network          Next Hop      In label/Out label
Route Distinguisher: 65000:1 (CUSTOMER_A)
   10.10.10.0/24    192.168.1.2     21/nolabel
   11.11.11.0/24    4.4.4.4         nolabel/21
```

On CE2, check that CE2 learns CE1's route through EIGRP.

```text
CE2# show ip route eigrp
D        10.10.10.0 [90/435200] via 192.168.2.1, 00:04:18, Ethernet0/0
```

The route appears as an EIGRP internal route.

Although the route crossed the MPLS VPN using MP-BGP VPNv4, EIGRP PE-CE L3VPN support preserves EIGRP route information across the VPN.

So an internal EIGRP route learned from CE1 can appear as an internal EIGRP route on CE2, although it was redistributed from VPNv4.

The details of internal and external route preservation are covered in the next page.

---

## PE-CE EIGRP Is Simpler Than PE-CE OSPF

EIGRP PE-CE does not have OSPF's superbackbone behavior.

There is no OSPF Domain ID.

There is no Down Bit.

There is no OSPF Domain Tag.

There are no OSPF sham links.

However, EIGRP still has L3VPN-specific behavior.

Important topics include:

```text
EIGRP route type preservation
EIGRP metric preservation, including backdoor path behavior
EIGRP Site of Origin for multihomed sites
```

These topics matter because a route moves like this:

```text
EIGRP
  ↓
BGP VPNv4
  ↓
EIGRP
```

The remote EIGRP process needs enough information to represent the route correctly.

---

## Route Types in EIGRP L3VPN

EIGRP has internal and external routes.

```text
D:
EIGRP internal route.

D EX:
EIGRP external route.
```

In basic redistribution, you would expect all routes redistributed from BGP into EIGRP to appear as `D EX`.

However, EIGRP L3VPN support can preserve EIGRP route information across the VPN so that routes can behave more like native EIGRP routes at the remote site.

This is covered in the route-types page.

For now, remember:

```text
Normal route handoff:
EIGRP route enters PE.
PE carries EIGRP information in BGP.
Remote PE reintroduces the route into EIGRP.
```

---

## Metric Preservation

EIGRP path selection depends on metric information.

In normal redistribution, the redistributing router assigns a new metric.

But in EIGRP PE-CE L3VPN, the PE can carry EIGRP metric information across MP-BGP so the remote side can make a better EIGRP decision.

This is one of the most important differences between simple redistribution and EIGRP-aware L3VPN behavior.

The metric-preservation page covers this in detail.

---

## Site of Origin

Site of Origin is important in multihomed EIGRP L3VPN designs.

Example:

```text
One customer site connects to two PEs.
The site also has internal or backdoor connectivity.
```

A route that originated at a site should not be sent back into the same site from another PE.

Site of Origin helps identify where the route came from for loop prevention.

This is covered in the Site of Origin page.

---

## Key Points

* EIGRP can be used as the PE-CE routing protocol in an MPLS L3VPN.
* The CE routers run normal EIGRP.
* The PE routers run EIGRP inside the customer VRF.
* Customer routes learned by EIGRP are installed in the VRF.
* To export CE routes into VPNv4, redistribute EIGRP into BGP under `address-family ipv4 vrf`.
* To advertise remote VPN routes to the CE, redistribute BGP into EIGRP under the EIGRP VRF address family.
* Ordinary BGP-to-EIGRP redistribution requires a seed metric, but VPNv4 routes that carry EIGRP metric information can be redistributed back into EIGRP without manually setting a metric.
* PE-CE EIGRP does not use OSPF concepts such as the superbackbone, Domain ID, Down Bit, Domain Tag, or sham links.
* EIGRP L3VPN still has special behavior for route type preservation, metric preservation, and Site of Origin.
