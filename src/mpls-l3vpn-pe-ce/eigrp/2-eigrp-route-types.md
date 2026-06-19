# EIGRP Route Types in L3VPN

EIGRP has two main route types:

| Code | Meaning |
| :--- | :--- |
| `D` | EIGRP internal route |
| `D EX` | EIGRP external route |

In a basic redistribution design, you might expect a route redistributed from BGP into EIGRP to become external.

```text
BGP route
  |
  ↓ redistribution
EIGRP external route
```

But EIGRP PE-CE over MPLS L3VPN is not just normal redistribution.

When EIGRP is used as the PE-CE protocol, Cisco can carry EIGRP route information across the VPNv4 control plane.

This means an EIGRP internal route at one site can appear as an EIGRP internal route at the remote site.

```text
CE1 route:
D 10.10.10.0/24

CE2 route:
D 10.10.10.0/24
```

That is the key point of this page.

---

## Basic Topology

Use the same topology as the EIGRP overview page:

```text
Customer A Site 1                    Customer A Site 2

10.10.10.0/24                        11.11.11.0/24
      |                                      |
     CE1                                    CE2
      |                                      |
     PE1 -------- P1 --------- P2 --------- PE2
    1.1.1.1     2.2.2.2      3.3.3.3      4.4.4.4
```

PE-CE EIGRP:

```text
CE1 --- PE1:
EIGRP AS 100

CE2 --- PE2:
EIGRP AS 100
```

Provider VPN control plane:

```text
PE1 --- PE2:
MP-BGP VPNv4
```

Route flow:

```text
CE1 EIGRP
  ↓
PE1 EIGRP VRF process
  ↓
BGP IPv4 VRF address family
  ↓
MP-BGP VPNv4
  ↓
PE2 BGP IPv4 VRF address family
  ↓
PE2 EIGRP VRF process
  ↓
CE2 EIGRP
```

---

## Normal Redistribution Expectation

In normal routing, if a plain BGP route is redistributed into EIGRP, the route is external.

Ordinary BGP-to-EIGRP redistribution also needs a seed metric, unless a default metric is configured.

Example of ordinary redistribution:

```text
router eigrp 100
 redistribute bgp 65000 metric 10000 100 255 1 1500
```

A router receiving this route would normally show:

```text
D EX     10.10.10.0/24
```

EIGRP PE-CE L3VPN behavior is different when the BGP route is a VPNv4-imported route that still carries EIGRP information.

---

## EIGRP L3VPN Behavior

When an EIGRP route is redistributed into BGP on a PE, Cisco attaches EIGRP information to the VPNv4 route.

This information includes details such as:

```text
EIGRP route type
EIGRP metric information
```

The remote PE can use this information when it redistributes the VPNv4 route back into EIGRP.

So the remote CE does not simply see every VPN-learned route as `D EX`.

Example:

```text
CE1 route:
D 10.10.10.0/24

Route crosses MPLS VPN:
EIGRP → BGP VPNv4 → EIGRP

CE2 route:
D 10.10.10.0/24
```

This is route type preservation.

---

## Internal Routes Across the VPN

CE1 advertises its loopback as an EIGRP internal route.

PE1 learns the route from CE1 in the customer VRF.

```text
PE1# show ip route vrf CUSTOMER_A eigrp
D        10.10.10.0 [90/3584000] via 192.168.1.2, 00:26:02, Ethernet0/1
```

PE1 redistributes EIGRP into BGP under the VRF address family.

```text
router bgp 65000
 address-family ipv4 vrf CUSTOMER_A
  redistribute eigrp 100
```

The VPNv4 route carries EIGRP information across the provider core.

PE2 imports the VPNv4 route and redistributes BGP into EIGRP.

```text
router eigrp CUSTOMER_A
 address-family ipv4 vrf CUSTOMER_A autonomous-system 100
  topology base
   redistribute bgp 65000
```

No manual seed metric is required for this VPN-learned EIGRP route, because the VPNv4 route carries the EIGRP metric information.

CE2 can learn the route as EIGRP internal.

```text
CE2# show ip route eigrp
D        10.10.10.0 [90/435200] via 192.168.2.1, 00:04:18, Ethernet0/0
```

The important point:

```text
The route crossed BGP VPNv4.
But the remote CE still sees D, not D EX.
```

This is different from ordinary redistribution behavior.

---

## External Routes Across the VPN

Now assume CE1 has an external EIGRP route.

For example, CE1 redistributes a static route into EIGRP.

```text
ip route 10.10.20.0 255.255.255.0 Null0
!
router eigrp 100
 redistribute static
```

CE1 advertises the route as external.

PE1 learns the external EIGRP route and exports it into VPNv4.

```
PE1# show ip route vrf CUSTOMER_A eigrp
D EX     10.10.20.0 [170/3584000] via 192.168.1.2, 00:15:01, Ethernet0/1

PE1# show bgp vpnv4 unicast all 10.10.20.0
BGP routing table entry for 65000:1:10.10.20.0/24, version 127
Paths: (1 available, best #1, table CUSTOMER_A)
  Advertised to update-groups:
     4         
  Refresh Epoch 1
  Local
    192.168.1.2 (via vrf CUSTOMER_A) from 0.0.0.0 (1.1.1.1)
      Origin incomplete, metric 3584000, localpref 100, weight 32768, valid, sourced, best
      Extended Community: RT:1:1 
        Cost:pre-bestpath:129:3584000 (default-2143899647) 0x8800:0:0 
        0x8801:100:153600 0x8802:65281:256000 0x8803:65281:1500 
        0x8804:0:168432641 0x8805:11:0 0x8806:0:168432641
      mpls labels in/out 22/nolabel
      rx pathid: 0, tx pathid: 0x0
      Updated on Jun 18 2026 03:51:40 UTC
```

The VPNv4 route carries EIGRP route-type information.

PE2 imports the route and redistributes it back into EIGRP.

CE2 should see the route as external.

```text
CE2# show ip route eigrp
D EX     10.10.20.0 [170/435200] via 192.168.2.1, 00:00:06, Ethernet0/0
```

The route entered the VPN as EIGRP external, so it is reintroduced as EIGRP external.

---

## Route Type Summary

| Route entering VPN from CE1 | Route seen by CE2 | Notes |
| :--- | :--- | :--- |
| `D` | `D` | EIGRP internal route type is preserved |
| `D EX` | `D EX` | EIGRP external route type is preserved |

This is much simpler than OSPF PE-CE route type behavior.

There is no OSPF Domain ID.

There is no superbackbone decision between `O IA` and `O E2`.

For EIGRP, the main question is:

```text
Did the route enter the VPN as EIGRP internal or EIGRP external?
```

---

## Why the Route Can Stay Internal

The PE does not simply throw away EIGRP information when it exports the route to VPNv4.

Instead, EIGRP-related information is attached to the BGP route.

One important mechanism is the BGP cost community.

In VPNv4 output, you'll 'see extended communities similar to this:

```text
      Extended Community: RT:1:1 
        Cost:pre-bestpath:128:3584000 (default-2143899647) 0x8800:32768:0 
        0x8801:100:153600 0x8802:65281:256000 0x8803:65281:1500 
        0x8806:0:168432641
```

I won't explain the details of the output, but remember this:

```text
The VPNv4 route carries EIGRP route type and metric information.
The remote PE uses that information when reintroducing the route into EIGRP.
```

This is why an internal route can remain `D` on the remote CE.

---

## Verifying in BGP

Check the VPNv4 route.

```text
PE1# show bgp vpnv4 unicast all 10.10.10.0 
BGP routing table entry for 65000:1:10.10.10.0/24, version 121
Paths: (1 available, best #1, table CUSTOMER_A)
  Advertised to update-groups:
     4         
  Refresh Epoch 1
  Local
    192.168.1.2 (via vrf CUSTOMER_A) from 0.0.0.0 (1.1.1.1)
      Origin incomplete, metric 3584000, localpref 100, weight 32768, valid, sourced, best
      Extended Community: RT:1:1 
        Cost:pre-bestpath:128:3584000 (default-2143899647) 0x8800:32768:0 
        0x8801:100:153600 0x8802:65281:256000 0x8803:65281:1500 
        0x8806:0:168432641
      mpls labels in/out 21/nolabel
      rx pathid: 0, tx pathid: 0x0
      Updated on Jun 18 2026 03:27:43 UTC
```

Look for extended communities.

This tells you the route is not just a plain VPNv4 route.

It is carrying EIGRP-related cost-community information.

You do not usually need to decode every opaque value in the output.

For route-type troubleshooting, the important questions are:

```text
Was the route learned from EIGRP at the ingress PE?
Was it redistributed into BGP under the correct VRF address family?
Did the VPNv4 route keep the EIGRP extended communities?
Was the route redistributed from BGP back into EIGRP at the egress PE?
Does the CE see D or D EX?
```

---

## Verifying in the EIGRP Topology Table

Check the EIGRP topology table on the egress PE.

```text
PE2#show ip eigrp vrf CUSTOMER_A topology 10.10.10.0/24
EIGRP-IPv4 VR(CUSTOMER_A) Topology Entry for AS(100)/ID(192.168.2.1)
           Topology(base) TID(0) VRF(CUSTOMER_A)
EIGRP-IPv4(100): Topology base(0) entry for 10.10.10.0/24
  State is Passive, Query origin flag is 1, 1 Successor(s), FD is 458752000
  Descriptor Blocks:
  1.1.1.1, from VPNv4 Sourced, Send flag is 0x0
      Composite metric is (458752000/0), route is Internal (VPNv4 Sourced)
      Vector metric:
        Minimum bandwidth is 10000 Kbit
        Total delay is 6000000000 picoseconds
        Reliability is 255/255
        Load is 1/255
        Minimum MTU is 1500
        Hop count is 1
        Originating router is 10.10.20.1
```

For a VPN-learned route, you'll see the route as sourced from VPNv4.

The important part is:

```text
from VPNv4 Sourced
route is Internal (VPNv4 Sourced)
```

That explains why the CE can receive the route as `D`.

For an external route, the topology table should identify the route as external.

```text
PE2# show ip eigrp vrf CUSTOMER_A topology 10.10.20.0/24
EIGRP-IPv4 VR(CUSTOMER_A) Topology Entry for AS(100)/ID(192.168.2.1)
           Topology(base) TID(0) VRF(CUSTOMER_A)
EIGRP-IPv4(100): Topology base(0) entry for 10.10.20.0/24
  State is Passive, Query origin flag is 1, 1 Successor(s), FD is 458752000
  Descriptor Blocks:
  1.1.1.1, from VPNv4 Sourced, Send flag is 0x0
      Composite metric is (458752000/0), route is External (VPNv4 Sourced)
      Vector metric:
        Minimum bandwidth is 10000 Kbit
        Total delay is 6000000000 picoseconds
        Reliability is 255/255
        Load is 1/255
        Minimum MTU is 1500
        Hop count is 1
        Originating router is 10.10.20.1
      External data:
        AS number of route is 0
        External protocol is Connected, external metric is 0
        Administrator tag is 0 (0x00000000)
```

---

## Common Misconception

A common mistake is to think:

```text
The egress PE redistributes BGP into EIGRP.
Therefore all remote routes must become D EX.
```

That is true for ordinary redistribution logic.

But it is not the full behavior of EIGRP PE-CE over MPLS L3VPN.

```text
Ingress PE:
Learns EIGRP route from CE.
Exports route into VPNv4 with EIGRP information.

Egress PE:
Imports VPNv4 route.
Uses carried EIGRP information when reintroducing the route into EIGRP.

Remote CE:
Can see the original EIGRP route type.
```

So internal EIGRP routes can remain internal across the VPN.

---

## EIGRP vs OSPF Route Types in L3VPN

EIGRP route-type behavior is simpler than OSPF route-type behavior.

OSPF PE-CE has special concepts such as:

```text
OSPF superbackbone
Domain ID
O IA vs O E2 behavior
Down Bit
Domain Tag
Sham links
```

EIGRP does not use those concepts.

For EIGRP, the main L3VPN behavior is:

```text
Preserve EIGRP route type.
Preserve EIGRP metric information.
Use Site of Origin for loop prevention in multihomed designs.
```

That is why an internal EIGRP route can cross MP-BGP VPNv4 and still appear as internal EIGRP on the remote CE.

---

## Key Points

* EIGRP has internal routes (`D`) and external routes (`D EX`).
* In normal redistribution, BGP into EIGRP would usually produce external EIGRP routes and require a seed metric.
* EIGRP PE-CE over MPLS L3VPN has special behavior beyond simple redistribution.
* EIGRP route type and metric information are carried across the VPNv4 control plane.
* An internal EIGRP route appears as internal EIGRP at the remote site.
* An external EIGRP route appears as external EIGRP at the remote site.
* In VPNv4 output, EIGRP-related information appears as BGP cost-community information.
* `Cost:pre-bestpath` is a key sign that EIGRP metric and route-type information is being carried.
* On the CE, `D` with AD 90 means internal EIGRP.
* On the CE, `D EX` with AD 170 means external EIGRP.
