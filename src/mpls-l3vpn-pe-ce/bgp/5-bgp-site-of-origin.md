# BGP Site of Origin

This page explains **BGP Site of Origin**, or **SoO**, in an MPLS L3VPN PE-CE BGP design.

SoO is used to prevent a multihomed customer site from relearning its own routes through the MPLS VPN.

It is especially useful when normal BGP AS_PATH loop prevention is weakened or bypassed by features such as:

```text
AS-Override
Allowas-in
Different AS numbers inside the same customer site
Redistribution between BGP and an IGP
Customer backdoor links
```

## Topology

This page uses the same topology as the PE-CE BGP multihoming page.

```text
                        PE2 --- CE2
                        //       |
CE1 --- PE1 === MPLS VPN         | SITE B
  SITE A                \\       |
                        PE3 --- CE3
```

Site A is single-homed.

Site B is multihomed.

```text
Site A:
CE1 connects to PE1.

Site B:
CE2 connects to PE2.
CE3 connects to PE3.
CE2 and CE3 are part of the same customer site.
```

The important prefixes are:

```text
Site A LAN:      10.10.10.0/24
Site B LAN:      11.11.11.0/24
CE2-CE3 link:    192.168.23.0/24
```

The AS numbers are:

```text
Provider AS: 65000

CE1 AS:      65100
CE2 AS:      65200
CE3 AS:      65200
```

## The Problem

In the multihoming topology, Site B has two PE-CE links.

```text
PE2 --- CE2
        |
        | SITE B
        |
PE3 --- CE3
```

CE2 and CE3 are part of the same customer site.

If CE2 advertises a route to PE2, that route can travel through the MPLS VPN and return to the same site through PE3.

Example:

```text
1. CE2 advertises 11.11.11.0/24 to PE2.

2. PE2 exports the route into MP-BGP VPNv4.

3. PE3 imports the route into the CUSTOMER_A VRF.

4. PE3 advertises the route to CE3.

5. CE3 is in the same site as CE2.
```

This means Site B may receive its own route back through the MPLS VPN.

That is called **route feedback**.

## Why AS_PATH Is Not Enough

In this topology, CE2 and CE3 both use AS `65200`.

If CE2 advertises `11.11.11.0/24` to PE2, the route has this AS_PATH:

```text
AS_PATH: 65200
```

PE2 sends the route through the MPLS VPN to PE3.

PE3 then tries to advertise the route to CE3.

Normally, CE3 rejects the route because its own AS appears in the AS_PATH.

```text
CE3 local AS:     65200
Received AS_PATH: 65000 65200
```

That is normal BGP AS_PATH loop prevention.

However, do not rely only on AS_PATH loop prevention in MPLS VPN multihoming designs.

AS_PATH loop prevention might not protect the site if:

```text
CE2 and CE3 use different AS numbers
AS-Override is configured
Allowas-in is configured
The customer redistributes between BGP and an IGP
etc.
```

SoO provides site-based loop prevention that is independent of the customer AS_PATH.

## What SoO Is

**Site of Origin** is a BGP extended community.

It identifies the site where a route came from.

Example SoO values:

```text
Site A SoO: 65000:100
Site B SoO: 65000:200
```

The value itself is arbitrary.

The important design rule is:

```text
Use the same SoO value on all PE-CE links connected to the same customer site.
Use different SoO values for different customer sites.
```

In this topology:

```text
PE1-to-CE1 SoO: 65000:100
PE2-to-CE2 SoO: 65000:200
PE3-to-CE3 SoO: 65000:200
```

CE2 and CE3 use the same SoO value because they are part of the same site.

CE1 uses a different SoO value because it is a different site.

## How SoO Works

When a PE learns a route from a CE, it tags the route with the SoO value configured for that PE-CE neighbor.

Example:

```text
CE2 advertises 11.11.11.0/24 to PE2.
PE2 has SoO 65000:200 configured toward CE2.
PE2 tags the route with SoO 65000:200.
```

The route is then carried through MP-BGP VPNv4.

```text
VPNv4 route:
RD:       65000:2
Prefix:   11.11.11.0/24
RT:       1:1
SoO:      65000:200
Next hop: PE2
AS_PATH:  65200
```

When another PE prepares to advertise the route to a CE, it compares:

```text
The SoO attached to the route
The SoO configured for the PE-CE neighbor
```

If the values match, the PE does not advertise the route to that CE.

```text
Route SoO:          65000:200
Neighbor CE3 SoO:   65000:200

Result:
Do not advertise the route to CE3.
```

If the values do not match, the PE can advertise the route normally.

```text
Route SoO:          65000:100
Neighbor CE3 SoO:   65000:200

Result:
The route can be advertised to CE3.
```

## SoO, RD, and RT

Do not confuse SoO with RDs and RTs:

| Attribute | Purpose                                        |
| --------- | ---------------------------------------------- |
| RD        | Makes VPNv4 routes unique                      |
| RT        | Controls which VRFs import the route           |
| SoO       | Identifies the site where the route originated |

The RT controls which routes should be imported from VPNv4 into a VRF.

The SoO controls whether a route should be advertised back toward a site.

Example:

```text
RT 1:1:
Import this route into CUSTOMER_A.

SoO 65000:200:
This route came from Site B.
Do not advertise it back to Site B.
```

A PE can still import a route into the VRF even if the route has the same SoO as one of its CE neighbors.

The SoO check is used when advertising the route toward that CE.

## SoO Design

Use one SoO value per customer site.

In this topology:

```text
Site A:
CE1 connects to PE1.
SoO: 65000:100

Site B:
CE2 connects to PE2.
CE3 connects to PE3.
SoO: 65000:200
```

This gives the desired behavior:

```text
Routes from Site A can be advertised to Site B.
Routes from Site B can be advertised to Site A.
Routes from Site B are not advertised back to Site B.
Routes from Site A are not advertised back to Site A.
```

Site A is single-homed in the topology, so SoO isnt actually needed. I'll focus on configuring it on PE2 and PE3 connected to Site B.

## Basic SoO Configuration

On IOS XE, SoO can be configured directly on the PE-CE BGP neighbor under the IPv4 VRF address family.

Syntax:

```text
neighbor <neighbor-address> soo <soo-value>
```

Configure SoO on the PE routers.

PE2:

```text
router bgp 65000
 address-family ipv4 vrf CUSTOMER_A
  neighbor 192.168.2.2 remote-as 65200
  neighbor 192.168.2.2 activate
  neighbor 192.168.2.2 soo 65000:200
```

PE3:

```text
router bgp 65000
 address-family ipv4 vrf CUSTOMER_A
  neighbor 192.168.3.2 remote-as 65200
  neighbor 192.168.3.2 activate
  neighbor 192.168.3.2 soo 65000:200
```

PE2 and PE3 use the same SoO value because CE2 and CE3 are in the same customer site.

```text
PE2-to-CE2 SoO: 65000:200
PE3-to-CE3 SoO: 65000:200
```

## Control Plane Example

### Step 1: CE2 Advertises the Route

CE2 advertises `192.168.23.0/24` to PE2.

```text
CE2 -> PE2

Prefix:  192.168.23.0/24
AS_PATH: 65200
```

PE2 has this configured toward CE2:

```text
neighbor 192.168.2.2 soo 65000:200
```

So PE2 attaches SoO `65000:200` to the route.

### Step 2: PE2 Exports the VPNv4 Route

PE2 exports the route into MP-BGP VPNv4.

```text
PE2 -> PE1 / PE3

RD:       65000:2
Prefix:   192.168.23.0/24
RT:       1:1
SoO:      65000:200
Next hop: 4.4.4.4
AS_PATH:  65200
```

### Step 3: PE3 Imports the Route

PE3 imports the route into `CUSTOMER_A` because the RT matches.

```text
Route RT:        1:1
PE3 import RT:   1:1

Result:
PE3 imports the route into CUSTOMER_A.
```

SoO does not stop PE3 from importing the route.

### Step 4: PE3 Checks the SoO Before Advertising to CE3

PE3 has this configured toward CE3:

```text
neighbor 192.168.3.2 soo 65000:200
```

The imported route also has SoO `65000:200`.

```text
Route SoO:        65000:200
CE3 neighbor SoO: 65000:200
```

The values match.

So PE3 does not advertise the route to CE3.

```text
PE3 -> CE3

Do not advertise 11.11.11.0/24.
```

This prevents Site B from relearning its own route.

## Routes from Other Sites Still Work

Now consider a route from Site A.

CE1 advertises `10.10.10.0/24` to PE1.

PE1 tags it with Site A's SoO.

```text
Route SoO: 65000:100
```

PE3 prepares to advertise the route to CE3.

CE3's neighbor SoO is `65000:200`.

```text
Route SoO:        65000:100
CE3 neighbor SoO: 65000:200
```

The values do not match.

So PE3 can advertise the route to CE3.

```text
PE3 -> CE3

Advertise 10.10.10.0/24.
```

That is the desired result.

SoO blocks routes from returning to the same site, but it does not block routes from other sites.

## SoO with AS-Override

SoO is especially important when AS-Override is configured.

Without AS-Override, CE3 may reject a Site B route because AS `65200` appears in the AS_PATH.

```text
PE3 -> CE3

Prefix:  11.11.11.0/24
AS_PATH: 65000 65200
```

CE3 rejects the route because `65200` is its own AS.

With AS-Override, PE3 can rewrite AS `65200` to AS `65000`.

```text
Before AS-Override:
AS_PATH: 65000 65200

After AS-Override:
AS_PATH: 65000 65000
```

Now CE3 may accept the route.

That means Site B can relearn its own route through the MPLS VPN.

SoO prevents this before the route is even advertised to CE3.

Example PE3 configuration:

```text
router bgp 65000
 address-family ipv4 vrf CUSTOMER_A
  neighbor 192.168.3.2 remote-as 65200
  neighbor 192.168.3.2 activate
  neighbor 192.168.3.2 as-override
  neighbor 192.168.3.2 soo 65000:200
```

With this configuration:

```text
AS-Override solves the same-AS problem.
SoO prevents Site B from relearning its own routes.
```

## SoO with Allowas-in

Allowas-in is configured on the CE.

It allows the CE to accept routes that contain its own AS in the AS_PATH.

Example:

```text
router bgp 65200
 neighbor 192.168.3.1 remote-as 65000
 neighbor 192.168.3.1 allowas-in
```

This can also allow Site B to relearn its own routes.

SoO is configured on the PE to stop the route before it is sent to the CE.

```text
router bgp 65000
 address-family ipv4 vrf CUSTOMER_A
  neighbor 192.168.3.2 soo 65000:200
```

## Alternative Configuration with a Route Map

The direct neighbor command is the simplest method:

```text
neighbor 192.168.2.2 soo 65000:200
```

But you can also use a route map to set the SoO extended community inbound from the CE.

> This is the same method as when configuring SoO for PE-CE EIGRP.

Example:

```text
route-map SET-SOO-SITE-B permit 10
 set extcommunity soo 65000:200
```

Apply it inbound from the CE:

```text
router bgp 65000
 address-family ipv4 vrf CUSTOMER_A
  neighbor 192.168.2.2 route-map SET-SOO-SITE-B in
```

> If both are configured, the neighbor-level SoO configuration takes precedence over the inbound route-map SoO value.

## Verification

Aside from checking the BGP configuration, you can verify SoO by examining the PE's VPNv4 routes:

```
PE3#show bgp vpnv4 unicast all 192.168.23.0
BGP routing table entry for 65000:2:192.168.23.0/24, version 61
Paths: (2 available, best #2, table CUSTOMER_A)
  Advertised to update-groups:
     6         
  Refresh Epoch 11
  65200
    4.4.4.4 (metric 21) (via default) from 4.4.4.4 (4.4.4.4)
      Origin IGP, metric 0, localpref 100, valid, internal
      Extended Community: SoO:65000:200 RT:1:1
      mpls labels in/out 16/23
      rx pathid: 0, tx pathid: 0
      Updated on Jun 19 2026 06:26:16 UTC
  Refresh Epoch 2
  65200
    192.168.3.2 (via vrf CUSTOMER_A) from 192.168.3.2 (192.168.23.3)
      Origin IGP, metric 0, localpref 100, valid, external, best
      Extended Community: SoO:65000:200 RT:1:1
      mpls labels in/out 16/nolabel
      rx pathid: 0, tx pathid: 0x0
      Updated on Jun 19 2026 06:25:14 UTC
```

Both routes to 192.168.23.0/24 have the following community:

```
Extended Community: SoO:65000:200
```

## Common Mistakes

### Mistake 1: Using Different SoO Values for the Same Site

Incorrect:

```text
PE2-to-CE2 SoO: 65000:200
PE3-to-CE3 SoO: 65000:300
```

If CE2 and CE3 are part of the same site, this is wrong.

The SoO values do not match, so PE3 could advertise the route back into Site B.

Correct:

```text
PE2-to-CE2 SoO: 65000:200
PE3-to-CE3 SoO: 65000:200
```

### Mistake 2: Using the Same SoO for Different Sites

Incorrect:

```text
Site A SoO: 65000:100
Site B SoO: 65000:100
```

This can block legitimate routes between different sites.

Correct:

```text
Site A SoO: 65000:100
Site B SoO: 65000:200
```

### Mistake 3: Configuring SoO on the Wrong Neighbor

SoO for PE-CE BGP is configured on the PE toward the CE under the IPv4 VRF address family.

Correct:

```text
router bgp 65000
 address-family ipv4 vrf CUSTOMER_A
  neighbor 192.168.2.2 soo 65000:200
```

Do not configure this on the PE-PE VPNv4 neighbor and expect it to identify customer sites.

### Mistake 4: Forgetting `send-community extended` Between PEs

SoO is an extended community.

VPNv4 route targets are also extended communities.

The PE-PE VPNv4 sessions should send extended communities.

Example:

```text
router bgp 65000
 address-family vpnv4
  neighbor 4.4.4.4 activate
  neighbor 4.4.4.4 send-community extended
  neighbor 6.6.6.6 activate
  neighbor 6.6.6.6 send-community extended
```

Without extended communities, RT and SoO information will not be carried correctly.

> `send-community extended` is added automatically in modern IOS when you `activate` a VPNv4 neighbor.

## Key Points

* SoO is a BGP extended community.
* SoO identifies the site where a route originated.
* Use the same SoO value on all PE-CE links connected to the same customer site.
* Use different SoO values for different customer sites.
* SoO prevents a PE from advertising a route back to a CE if the route's SoO matches the PE-CE neighbor's SoO.
* SoO determines whether a route is advertised into a site, not whether it is imported from VPNv4 into the VRF.
* RT controls VRF import policy.
* SoO controls same-site route feedback prevention.
* SoO is configured on the PE under the IPv4 VRF address family.
* The direct command is `neighbor <neighbor-address> soo <soo-value>`.
* SoO is especially important with multihoming, backdoor links, AS-Override, and Allowas-in.
