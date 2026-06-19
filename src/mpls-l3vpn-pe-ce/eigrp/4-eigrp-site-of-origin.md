# EIGRP Site of Origin

**Site of Origin** is an extended community used for loop prevention in MPLS L3VPN PE-CE EIGRP designs.

It is usually written as **SoO**.

SoO identifies the customer site where a route originally entered the MPLS VPN.

The basic idea is:

```text
This route originated from Site B.
Do not advertise it back into Site B.
```

SoO is most important when a customer site has more than one path into the VPN.

Examples:

```text
Dual-homed customer site
Multiple PE connections to the same site
Customer backdoor link
Complex EIGRP/VPN topology
```

This page uses one main topology:

```text
                        PE2
                        // \
CE1 --- PE1 === MPLS VPN    CE2   SITE B
  SITE A                \\ /
                        PE3
```

In this topology:

```text
Site A:
CE1 connects to PE1.

Site B:
CE2 connects to both PE2 and PE3.
```

CE2 is a dual-homed customer router.

PE2 and PE3 are both connected to the same customer site.

---

## Why SoO Exists

In a simple single-homed site, EIGRP PE-CE routing is straightforward.

```text
CE1 --- PE1 === MPLS VPN === PE2 --- CE2
```

CE1 advertises a route to PE1.

PE1 exports it into VPNv4.

PE2 imports it and advertises it to CE2.

There is no obvious way for the same route to return to the site it came from.

But in a dual-homed site, this can happen.

```text
                        PE2
                        // \
CE1 --- PE1 === MPLS VPN    CE2   SITE B
  SITE A                \\ /
                        PE3
```

The problem is not that Site B learns routes from Site A.

That is normal.

The problem is that a route from Site B can enter the MPLS VPN through one PE and then return to Site B through another PE.

Example:

```text
1. CE2 advertises 11.11.11.0/24 to PE2.
2. PE2 exports the route into VPNv4.
3. PE3 imports the VPNv4 route.
4. PE3 tries to advertise the route back to CE2.
```

That is usually undesirable.

The route came from Site B, so it should not be advertised back into Site B through another PE.

SoO prevents this.

The PE attaches a Site of Origin value to routes learned from a site.

Example:

```text
Site B SoO:
65000:202
```

PE2 attaches `65000:202` to routes learned from CE2.

PE3 is also connected to Site B, so its CE-facing interface uses the same SoO value.

When PE3 receives a VPNv4 route with SoO `65000:202`, it recognizes that the route came from the same site as its CE-facing interface.

PE3 does not advertise that route back to CE2.

```text
Route SoO:
65000:202

PE3 CE-facing interface SoO:
65000:202

Result:
Do not advertise the route to CE2.
```

This prevents a route from leaving a customer site through one PE and re-entering the same customer site through another PE.

Important:

```text
SoO does not block routes from other sites.

A route from Site A can still be advertised to Site B.

A route from Site B is blocked from being advertised back into Site B.
```

---

## What SoO Is

SoO is a BGP extended community.

Example value:

```text
65000:202
```

This identifies a site.

Example design:

```text
Site A SoO:
65000:101

Site B SoO:
65000:202
```

Use a unique SoO value per site.

Use the same SoO value on all PE-CE links that connect to the same site.

```text
PE1-to-CE1:
SoO 65000:101

PE2-to-CE2:
SoO 65000:202

PE3-to-CE2:
SoO 65000:202
```

---

## Where SoO Is Configured for EIGRP PE-CE

For EIGRP PE-CE, SoO is commonly configured on the PE-CE interface.

The configuration has two parts:

```text
1. A route map that sets the SoO extended community.
2. The ip vrf sitemap command under the PE-CE interface.
```

Example:

```text
route-map SITE-B-SOO permit 10
 set extcommunity soo 65000:202
!
interface Ethernet0/1
 ip vrf sitemap SITE-B-SOO
```

The `ip vrf sitemap` command is applied on the PE interface facing the CE.

---

## Basic SoO Configuration

Site B is dual-homed to PE2 and PE3.

```text
                        PE2
                        // \
CE1 --- PE1 === MPLS VPN    CE2   SITE B
  SITE A                \\ /
                        PE3
```

Both PE-facing interfaces connected to Site B should use the same SoO value.

```text
Site B SoO:
65000:202
```

### PE2

```text
route-map SITE-B-SOO permit 10
 set extcommunity soo 65000:202
!
interface Ethernet0/1
 description To CE2 - Site B
 vrf forwarding CUSTOMER_A
 ip vrf sitemap SITE-B-SOO
 ip address 192.168.2.1 255.255.255.0
```

### PE3

```text
route-map SITE-B-SOO permit 10
 set extcommunity soo 65000:202
!
interface Ethernet0/1
 description To CE2 - Site B
 vrf forwarding CUSTOMER_A
 ip vrf sitemap SITE-B-SOO
 ip address 192.168.3.1 255.255.255.0
```

The important point:

```text
All PE-CE interfaces connected to the same site use the same SoO value.
```

Do not use the same SoO value for every customer site.

Different sites should use different SoO values.

Example:

```text
Site A: 65000:101
Site B: 65000:202
Site C: 65000:303
```

---

## How SoO Is Added to a Route

CE2 advertises `11.11.11.0/24` to PE2.

PE2 receives the route on this interface:

```text
interface Ethernet0/1
 ip vrf sitemap SITE-B-SOO
```

The site map sets:

```text
SOO:65000:202
```

The route learned from CE2 is a normal EIGRP route when it arrives at PE2.

The CE does not need to send SoO.

```text
CE2 sends normal EIGRP update.
PE2 receives the route.
PE2 associates the route with the interface SoO.
```

When PE2 exports the route into VPNv4, the route carries the SoO extended community.

Example:

```text
PE2# show bgp vpnv4 unicast all 11.11.11.0/24
BGP routing table entry for 65000:1:11.11.11.0/24, version 146
Paths: (1 available, best #1, table CUSTOMER_A)
  Advertised to update-groups:
     5         
  Refresh Epoch 1
  Local
    192.168.2.2 (via vrf CUSTOMER_A) from 0.0.0.0 (4.4.4.4)
      Origin incomplete, metric 3584000, localpref 100, weight 32768, valid, sourced, best
      Extended Community: SoO:65000:202 RT:1:1 
        Cost:pre-bestpath:128:3584000 (default-2143899647) 0x8800:32768:0 
        0x8801:100:153600 0x8802:65281:256000 0x8803:65281:1500 
        0x8806:0:185273089
      mpls labels in/out 21/nolabel
      rx pathid: 0, tx pathid: 0x0
      Updated on Jun 18 2026 05:22:25 UTC
```

The route is now identified as originating from Site B.

```text
Prefix:
11.11.11.0/24

Originating site:
Site B

SoO:
65000:202
```

---

## How SoO Prevents Re-advertisement to the Same Site

PE2 advertises the route to PE3 via BGP VPNv4.

```text
VPNv4 route:
11.11.11.0/24
SOO:65000:202
```

PE3 has a CE-facing interface for Site B.

```text
interface Ethernet0/1
 ip vrf sitemap SITE-B-SOO
```

The interface SoO is also:

```text
65000:202
```

PE3 compares the route's SoO to the outgoing CE-facing interface SoO.

```text
Route SoO:              65000:202
Outgoing interface SoO: 65000:202
Result:                 Match
```

Because the values match, PE3 does not advertise the route to CE2.

```text
Route originated from Site B.
Do not send it back into Site B.
```

This prevents the route from returning to its own site through another PE.

> The reverse also happens: CE2 advertises 11.11.11.0/24 to PE3 using EIGRP,
> PE3 adds SoO 65000:202 and advertises it to PE2 via BGP VPNv4.

---

## SoO Direction: CE to VPN

In the CE-to-VPN direction, the CE normally advertises a normal EIGRP route.

```text
CE2 advertises 11.11.11.0/24 to PE2.
```

The route does not need to carry SoO in the EIGRP update.

PE2 accepts the route into the EIGRP topology table and VRF routing table.

Then PE2 exports it into VPNv4 with the SoO value from the PE-CE interface.

```text
CE2 EIGRP route
  ↓
PE2 EIGRP topology table
  ↓
PE2 VRF routing table
  ↓
BGP VPNv4 with SoO 65000:202
```

---

## SoO Direction: VPN to CE

In the VPN-to-CE direction, the PE receives a VPNv4 route that may already carry SoO.

Example:

```text
PE3 receives 11.11.11.0/24 from VPNv4.
Route carries SoO 65000:202.
```

PE3 redistributes BGP into EIGRP.

Before advertising the route out a CE-facing interface, PE3 checks the route SoO against the interface SoO.

```text
Route SoO:
65000:202

PE3 interface SoO:
65000:202
```

Because the values match, PE3 does not advertise the route to CE2.

If the route has no SoO, or if the SoO does not match the interface SoO, the route can be advertised.

Example route from Site A:

```text
Route:
10.10.10.0/24

Route SoO:
65000:101

PE3 interface SoO:
65000:202

Result:
No match.
Route can be advertised to CE2.
```

SoO blocks same-site re-entry, not normal inter-site reachability.

---

## SoO and EIGRP Topology Table Behavior

Do not think of normal CE-originated EIGRP updates as already carrying SoO.

In the usual CE-to-PE direction:

```text
CE sends normal EIGRP route to PE.
The route has no SoO.
PE accepts the route.
PE attaches SoO when exporting the route into VPNv4.
```

Example:

```text
CE2 advertises 11.11.11.0/24 to PE2.
PE2 receives the route on an interface with SoO 65000:202.
PE2 accepts the route into EIGRP.
PE2 exports the route into VPNv4 with SoO 65000:202.
```

The VPN-to-CE direction is where the SoO filtering is usually seen.

Example:

```text
PE3 imports VPNv4 route 11.11.11.0/24.
The route carries SoO 65000:202.
PE3 considers advertising it to CE2 on an interface with SoO 65000:202.
```

Result:

```text
SoO matches.
Do not advertise the route to CE2.
```

SoO can also be used in more advanced Cisco EIGRP backdoor-router scenarios, where SoO-aware EIGRP routers can carry and compare SoO values. But for normal PE-CE operation, the clean model is:

```text
CE-to-PE:
PE applies SoO before VPNv4 export.

VPN-to-CE:
PE checks SoO before CE advertisement.
```

---

## SoO with a Backdoor Link

A backdoor link is a customer link outside the MPLS VPN.

For this example, Site B has two CE routers:

```text
                        PE2 --- CE2
                        //       |
CE1 --- PE1 === MPLS VPN         | SITE B
  SITE A                \\       |
                        PE3 --- CE3
```

Site B includes both CE2 and CE3.

```text
Site B:
CE2 connects to PE2.
CE3 connects to PE3.
CE2 and CE3 are connected inside the customer site.
```

The CE2-to-CE3 link is the backdoor link.

It is not part of the MPLS VPN.

Use the same SoO value on both PE-facing links for Site B.

```text
PE2-to-CE2:
SoO 65000:202

PE3-to-CE3:
SoO 65000:202
```

Now assume CE2 advertises `11.11.11.0/24` to PE2.

PE2 exports the route into VPNv4 with SoO `65000:202`.

PE3 imports the VPNv4 route.

Because PE3's CE3-facing interface is also connected to Site B, PE3 should not advertise that route to CE3.

```text
Route:
11.11.11.0/24

Route SoO:
65000:202

PE3-to-CE3 interface SoO:
65000:202

Result:
Do not advertise the route to CE3.
```

CE3 should learn `11.11.11.0/24` through the CE2-to-CE3 customer link instead.

```text
CE3# show ip eigrp topology all-links 
P 11.11.11.0/24, 1 successors, FD is 409600, serno 53
        via 192.168.23.1 (409600/128256), Ethernet0/1
```

A route from Site A is different.

```text
Route:
10.10.10.0/24

Route SoO:
65000:101

PE3-to-CE3 interface SoO:
65000:202

Result:
Advertise the route to CE3.
```

The Site A route is allowed into Site B.

The Site B route is blocked from returning into Site B.

---

## Important Backdoor Link Caveat

SoO is powerful, but it can also filter routes you may have wanted to use as backup paths.

In this topology, CE2 and CE3 are treated as the same site.

```text
                        PE2 --- CE2
                        //       |
CE1 --- PE1 === MPLS VPN         | SITE B
  SITE A                \\       |
                        PE3 --- CE3
```

Because CE2 and CE3 are the same site, both PE-facing links use the same SoO value.

```text
PE2-to-CE2:
SoO 65000:202

PE3-to-CE3:
SoO 65000:202
```

If the CE2-to-CE3 link fails, the MPLS VPN may look like a possible backup path between CE2 and CE3.

However, SoO can prevent that backup path from being used.

Example:

```text
1. CE2 advertises 11.11.11.0/24 to PE2.
2. PE2 exports the route into VPNv4 with SoO 65000:202.
3. PE3 imports the VPNv4 route.
4. PE3 sees that its CE3-facing interface also uses SoO 65000:202.
5. PE3 does not advertise the route to CE3.
```

Result:

```text
CE3 may not learn CE2's route through the MPLS VPN.
```

Same site means:

```text
A route that came from CE2 should not be advertised back into Site B through CE3.
```

That is exactly what SoO is designed to enforce.

For the CCIE, pay close attention to what the task calls a "site."

```text
Same site:
Use the same SoO.

Different sites:
Use different SoO values.
```

If the VPN must be used as a backup between CE2 and CE3 when the CE2-to-CE3 link fails, you may need to treat CE2 and CE3 as different sites, use different SoO values, or design a more specific policy.

---

## SoO Does Not Replace Route Targets

Route targets still control VPN membership.

Example:

```text
route-target export 1:1
route-target import 1:1
```

SoO does not decide whether the remote PE imports the route into the VRF.

The RT does that.

SoO decides whether the route should be advertised to a particular site.

```text
Route Target:
Import/export control between VRFs.

Site of Origin:
Loop prevention at the site edge.
```

A VPNv4 route can carry both.

Example:

```text
Extended Community: SoO:65000:202 RT:1:1 
```

---

## Common Problems

### Same SoO used for all sites

Wrong design:

```text
Site A: 65000:202
Site B: 65000:202
Site C: 65000:202
```

This can cause legitimate routes to be filtered.

Use unique SoO values per site.

```text
Site A: 65000:101
Site B: 65000:202
Site C: 65000:303
```

### Different SoO values on the same site

Wrong design:

```text
PE2 to Site B: 65000:202
PE3 to Site B: 65000:303
```

Now PE3 may not recognize PE2's route as having originated from the same site.

The route can be advertised back into Site B.

Correct design:

```text
PE2 to Site B: 65000:202
PE3 to Site B: 65000:202
```

### SoO route map configured but not applied to the interface

This creates the route map but does nothing useful:

```text
route-map SITE-B-SOO permit 10
 set extcommunity soo 65000:202
```

You must apply it to the PE-CE interface.

```text
interface Ethernet0/1
 ip vrf sitemap SITE-B-SOO
```

### Forgetting that SoO can block backup paths

If a site is partitioned, a matching SoO can filter routes that you expected to use through the VPN or a backdoor link.

Check the site definition carefully.

```text
Is this really the same site?
Or should these be treated as separate sites with different SoO values?
```

---

## Key Points

* Site of Origin is usually written as SoO.
* SoO is a BGP extended community used for per-site loop prevention.
* SoO identifies the site where a route originated.
* In EIGRP PE-CE L3VPN, SoO is commonly configured with a route map and applied to the PE-CE interface using `ip vrf sitemap`.
* Use the same SoO value on all PE-facing interfaces for the same customer site.
* Use different SoO values for different customer sites.
* A route with an SoO matching the local site interface is filtered from being advertised back into that site.
* A route with no SoO or a different SoO can be advertised.
* A normal CE-originated EIGRP route does not need to carry SoO toward the PE.
* The PE attaches SoO when exporting the route into VPNv4.
* SoO helps with dual-homed sites, backdoor links, and complex EIGRP VPN topologies.
* SoO does not replace route targets.
* SoO does not replace metric preservation.
* SoO does not decide whether a route is `D` or `D EX`.
* Route type preservation, metric preservation, and SoO are three separate EIGRP L3VPN behaviors.
