# EIGRP Metric Preservation

EIGRP metric preservation is one of the most important EIGRP-specific behaviors in MPLS L3VPN PE-CE routing.

In a normal redistribution design, when one protocol is redistributed into another protocol, the redistributing router assigns a new seed metric.

Example:

```text
BGP route
  ↓ ordinary redistribution
EIGRP route with manually configured seed metric
```

EIGRP PE-CE over MPLS L3VPN is different.

When an EIGRP route enters the MPLS VPN, the PE can carry EIGRP metric information inside the VPNv4 route.

Then the remote PE can reintroduce the route into EIGRP using the preserved EIGRP metric information.

```text
EIGRP route
  ↓
BGP VPNv4 route carrying EIGRP metric information
  ↓
EIGRP route
```

This is why VPN-learned EIGRP routes do not necessarily need a manually configured seed metric when redistributed from BGP back into EIGRP.

> This was largely covered in previous pages, but this page focuses on the concept in more detail.

---

## Basic Topology

Use the same topology as the previous EIGRP pages:

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

## Why Metric Preservation Matters

EIGRP path selection depends heavily on metric.

The classic EIGRP metric is based mainly on:

```text
Minimum bandwidth
Cumulative delay
```

Other values are also carried:

```text
Reliability
Load
MTU
Hop count
```

If a route crosses the MPLS VPN and then receives an arbitrary new seed metric at the remote PE, the remote EIGRP site may make a poor path decision.

Example problem:

```text
Site 1 and Site 2 have two possible paths:

1. MPLS VPN path
2. Backdoor EIGRP path
```

If the MPLS VPN path loses its original EIGRP metric information, EIGRP cannot compare the VPN path and backdoor path in a meaningful way.

Metric preservation lets the remote site compare paths using EIGRP-style metric information instead of an arbitrary redistribution metric.

---

## Ordinary Redistribution vs EIGRP L3VPN

This distinction is very important.

### Ordinary BGP into EIGRP

In ordinary redistribution, BGP into EIGRP requires a seed metric.

Example:

```text
router eigrp 100
 redistribute bgp 65000 metric 10000 100 255 1 1500
```

Or:

```text
router eigrp 100
 default-metric 10000 100 255 1 1500
 redistribute bgp 65000
```

Without a metric, an ordinary BGP route may not be advertised into EIGRP.

### VPN-learned EIGRP route into EIGRP

In EIGRP PE-CE L3VPN, a VPNv4 route that originally came from EIGRP can carry EIGRP metric information.

Because the metric information is already carried with the VPNv4 route, the remote PE does not need a manually configured seed metric for that route.

Example:

```text
router eigrp CUSTOMER_A
 address-family ipv4 vrf CUSTOMER_A autonomous-system 100
  topology base
   redistribute bgp 65000
```

This can work because the BGP route is not just a plain BGP route.

It is a VPNv4 route that carries EIGRP information.

```text
Plain BGP route into EIGRP:
Seed metric required.

VPNv4 route carrying EIGRP metric information into EIGRP:
Seed metric not required.
```

---

## Ingress PE Behavior

Assume CE1 advertises `10.10.10.0/24` to PE1 using EIGRP.

PE1 learns the route in the customer VRF.

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

PE1 exports the route into VPNv4.

The VPNv4 route carries EIGRP information as BGP extended communities.

Example:

```text
PE1# show bgp vpnv4 unicast all 10.10.10.0
BGP routing table entry for 65000:1:10.10.10.0/24, version 133
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
      Updated on Jun 18 2026 04:33:17 UTC
```

The key part is:

```text
      Extended Community: RT:1:1 
        Cost:pre-bestpath:128:3584000 (default-2143899647) 0x8800:32768:0 
        0x8801:100:153600 0x8802:65281:256000 0x8803:65281:1500 
```

For EIGRP MPLS VPN PE-CE, this cost community carries EIGRP route type and metric information.

---

## Egress PE Behavior

PE2 receives the VPNv4 route from PE1.

In the VRF routing table, the route may appear as a BGP route.

```text
PE2# show ip route vrf CUSTOMER_A 10.10.10.0
B        10.10.10.0 [200/3584000] via 1.1.1.1, 00:02:43
```

PE2 redistributes BGP into EIGRP.

```text
router eigrp CUSTOMER_A
 address-family ipv4 vrf CUSTOMER_A autonomous-system 100
  topology base
   redistribute bgp 65000
```

No manual seed metric is required for this VPN-learned EIGRP route.

PE2 uses the EIGRP information carried in the VPNv4 route.

In the EIGRP topology table, the route appears as VPNv4 sourced.

```text
PE2# show ip eigrp vrf CUSTOMER_A topology 10.10.10.0/24
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

This shows that the route was reintroduced into EIGRP with preserved EIGRP information.

---

## Remote CE Result

CE2 can learn CE1's route through EIGRP.

```text
CE2# show ip route eigrp
D        10.10.10.0 [90/435200] via 192.168.2.1, 00:02:48, Ethernet0/0
```

The route is internal EIGRP.

The route also has an EIGRP metric that is meaningful to CE2.

This is not the result of a manually configured seed metric on PE2.

It is the result of EIGRP metric information being preserved through VPNv4.

---

## What the Cost Community Does

The BGP cost community is important because the route is carried through MP-BGP VPNv4 inside the provider network.

Normally, BGP best path selection would use BGP attributes such as:

```text
Weight
Local preference
AS path
Origin
MED
IGP metric to next hop
```

But EIGRP wants metric-based behavior.

If a PE has multiple possible VPNv4 paths for the same EIGRP prefix, it should prefer the path with the better EIGRP metric, not simply the path that wins because of normal BGP attributes.

The `Cost:pre-bestpath` community tells BGP to consider the EIGRP metric before normal BGP best-path comparisons.

```text
Cost:pre-bestpath:128:3584000
```

Practical meaning:

```text
Use the preserved EIGRP metric when comparing EIGRP VPN paths.
```

This helps EIGRP behave more naturally across the MPLS VPN.

---

## Metric Preservation and Backdoor Links

Metric preservation is especially important when customer sites also have a backdoor path.

Example:

```text
            MPLS L3VPN
        PE1 ========== PE2
         |              |
        CE1 ---------- CE2
          Backdoor link
```

CE2 might have two possible routes to CE1's LAN:

```text
1. Route through the MPLS VPN
2. Route through the backdoor EIGRP link
```

Unlike OSPF, EIGRP does not need a sham link to make the VPN path appear intra-area.

So the important question is:

```text
Which path has the better EIGRP metric?
```

Metric preservation makes this possible.

If the VPN path has a better preserved EIGRP metric, EIGRP can prefer the VPN path.

If the backdoor path has a better metric, EIGRP can prefer the backdoor path.

You can influence the decision using normal EIGRP tools.

Examples:

```text
interface bandwidth
interface delay
offset-list
```

The key point:

```text
OSPF uses sham links to fix route-type preference.
EIGRP uses preserved metrics so VPN and backdoor paths can be compared by metric.
```

---

## What Is Preserved

The VPNv4 route can carry EIGRP information such as:

```text
Route type
Composite metric
Bandwidth
Delay
Reliability
Load
MTU
Hop count
Originating router
External route information, for external EIGRP routes
```

You can see some of this information in the EIGRP topology table.

Example internal route:

```text
PE2# show ip eigrp vrf CUSTOMER_A topology 10.10.10.0/24
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

Example external route:

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

This is why the remote CE can see the correct EIGRP route type and a preserved metric.

---

## Configuration

The basic configuration does not require a special metric-preservation command.

Ingress PE:

```text
router bgp 65000
 address-family ipv4 vrf CUSTOMER_A
  redistribute eigrp 100
```

Egress PE:

```text
router eigrp CUSTOMER_A
 address-family ipv4 vrf CUSTOMER_A autonomous-system 100
  topology base
   redistribute bgp 65000
```

No seed metric is configured in the egress PE example because the route is a VPN-learned EIGRP route.

The VPNv4 route already carries the EIGRP metric information.

If you redistribute a plain BGP route that does not carry EIGRP metric information, use a seed metric or default metric.

Example:

```text
router eigrp CUSTOMER_A
 address-family ipv4 vrf CUSTOMER_A autonomous-system 100
  topology base
   redistribute bgp 65000 metric 10000 100 255 1 1500
```

or:

```text
router eigrp CUSTOMER_A
 address-family ipv4 vrf CUSTOMER_A autonomous-system 100
  topology base
   default-metric 10000 100 255 1 1500
   redistribute bgp 65000
```

Use this only when the BGP route does not have EIGRP metric information.

---

## Key Points

* EIGRP metric preservation lets EIGRP information survive the `EIGRP → VPNv4 → EIGRP` route flow.
* VPN-learned EIGRP routes can be redistributed from BGP back into EIGRP without a manually configured seed metric.
* This is different from ordinary BGP-to-EIGRP redistribution, which requires a seed metric or default metric.
* The VPNv4 route carries EIGRP information using BGP extended communities.
* `Cost:pre-bestpath` is the key sign that EIGRP metric information is being carried.
* The preserved information includes EIGRP route type and metric information.
* Metric preservation helps EIGRP compare VPN paths and backdoor paths using EIGRP metrics.
* EIGRP does not need an OSPF-style sham link.
* Use normal EIGRP metric tools, such as delay or offset lists, to influence path preference.
