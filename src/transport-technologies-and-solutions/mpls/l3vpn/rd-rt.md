# RD and RT

MPLS L3VPNs use two important values:

```text
RD = Route Distinguisher
RT = Route Target
```

They are configured together in a VRF, but they do different jobs.

```text
RD:
Makes customer routes unique in MP-BGP.

RT:
Controls which VRFs import and export VPN routes.
```

---

## Route Distinguisher

The **Route Distinguisher** makes a customer IPv4 prefix unique in MP-BGP.

It is prepended to the IPv4 prefix to create a VPNv4 NLRI.

```text
RD + IPv4 prefix = VPNv4 route
```

Example:

```text
RD:          65000:1
IPv4 route:  10.10.10.0/24

VPNv4 route: 65000:1:10.10.10.0/24
```

The RD is usually written in one of these formats:

```text
ASN:nn
IPv4-address:nn
4-byte-ASN:nn
```

Examples:

```text
65000:1
1.1.1.1:100
650000:100
```

A common format in labs is `ASN:number`, such as `65000:1`.

In larger networks, some designs use the PE loopback in the RD so the originating PE is easier to identify.

```text
PE1 loopback: 1.1.1.1
VRF ID:       100

RD:           1.1.1.1:100
```

The RD is part of the VPNv4 route itself.

* It is not a BGP community.
* It is not placed in the customer packet.
* It does not affect the MPLS data plane.

Its job is to make otherwise identical IPv4 prefixes unique in the BGP VPNv4 table.

> Usually you configure the same RD for a VRF on each PE router, but the values actually **don't** have to match.

---

## Route Target

The **Route Target** controls VPN membership.

It is carried as a BGP extended community.

When a PE exports a route from a VRF into MP-BGP, it attaches one or more export RTs.

When another PE receives the VPNv4 route, it checks the route's RTs against the import RTs configured in its local VRFs.

```text
Export RT:
Attached to routes leaving a VRF.

Import RT:
Used to decide which routes enter a VRF.
```

Simple example:

```text
PE1 VRF CUSTOMER_A
 route-target export 1:1

PE2 VRF CUSTOMER_A
 route-target import 1:1
```

PE1 exports the route with RT `1:1`.

PE2 imports the route because its VRF imports RT `1:1`.

Route targets are usually written in the same style as RDs.

Examples:

```text
1:1
65000:100
1.1.1.1:100
```

However, an RT is not the same thing as an RD.

The values may look similar, but the command context determines whether the value is being used as an RD or as an RT.

> For clarity, I am using a different value for the RD (`65000:1`) and RT (`1:1`), but they can be the same.
> But don't confuse the two: they serve different purposes as described above.

In the following example, the same value is used for both the RD and RT, but they still perform different functions.

```text
rd 65000:1
route-target export 65000:1
route-target import 65000:1
```

---

## RD vs RT

| | RD | RT |
| :--- | :--- | :--- |
| **Name** | Route Distinguisher | Route Target |
| **Main purpose** | Makes routes unique | Controls import/export |
| **BGP field** | Part of VPNv4/VPNv6 NLRI | Extended community |
| **Controls VPN membership?** | No | Yes |
| **Must match between PEs?** | No | Usually yes, depending on design |
| **Can have multiple values?** | One RD per VRF address family | Multiple import/export RTs |

---

## Basic Configuration

Example VRF:

```text
PE1(config)# vrf definition CUSTOMER_A
PE1(config-vrf)# rd 65000:1
PE1(config-vrf)# address-family ipv4
PE1(config-vrf-af)# route-target export 1:1
PE1(config-vrf-af)# route-target import 1:1
```

This configuration means:

- `rd 65000:1`
  - Routes from this VRF become VPNv4 routes using RD 65000:1.
- `route-target export 1:1`
  - Routes exported from this VRF carry RT 1:1.
- `route-target import 1:1`
  - This VRF imports VPNv4 routes that carry RT 1:1.

A common shorthand is `route-target both`. This is a macro that adds the `import` and `export` commands:

```text
PE1(config)# vrf definition CUSTOMER_A
PE1(config-vrf)# rd 65000:1
PE1(config-vrf)# address-family ipv4
PE1(config-vrf-af)# route-target both 1:1

PE1# show run | s vrf def
vrf definition CUSTOMER_A
 rd 65000:1
 !
 address-family ipv4
  route-target export 1:1
  route-target import 1:1
 exit-address-family
```

For simplicity's sake, match the `import` and `export` RTs. However, they can be different. For example:
- PE1:
  - `route-target export 1:1`
  - `route-target import 2:2`
- PE2:
  - `route-target export 2:2`
  - `route-target import 1:1`

This configuration will have the same effect:

- PE1 imports routes from PE2.
- PE2 imports routes from PE1.

---

## Route Export Process

Assume CE1 advertises `10.10.10.0/24` to PE1.

```text
CE1 --- PE1 === MPLS Core === PE2 --- CE2
```

PE1 installs the route in the customer VRF.

```text
VRF CUSTOMER_A:
10.10.10.0/24
```

Then PE1 exports the route into MP-BGP VPNv4.

During export, PE1 adds:

```text
RD
RT
BGP next hop
VPN label
```

PE1 then advertises this VPNv4 route to PE2:

```text
VPNv4 NLRI: 65000:1:10.10.10.0/24
RT:         1:1
Next hop:   1.1.1.1
VPN label:  24000
```

The RD makes the IPv4 prefix a unique VPNv4 route in MP-BGP.

The RT tells other PEs which VRFs should import the route.

The VPN label tells the egress PE how to forward packets for the route.

---

## Route Import Process

PE2 receives this VPNv4 route:

```text
VPNv4 NLRI: 65000:1:10.10.10.0/24
RT:         1:1
Next hop:   1.1.1.1
VPN label:  24000
```

PE2 checks its local VRFs.

Example:

```text
PE2 VRF CUSTOMER_A
 route-target import 1:1
```

Because the route has RT `1:1`, PE2 imports the route into `CUSTOMER_A`.

Inside the VRF routing table, the route appears as a normal IPv4 route.

```text
VRF CUSTOMER_A:
10.10.10.0/24 via 1.1.1.1
```

The RD is not shown as part of the customer route in the VRF routing table.

The RD is only used in the VPNv4 control plane.

---

## RD Design

The RD only needs to make routes unique in MP-BGP.

A simple lab design is to use one RD per VRF.

Example:

```text
PE1 CUSTOMER_A: rd 65000:1
PE2 CUSTOMER_A: rd 65000:1
```

This works in many simple designs.

For larger or more complex networks, it is common to use a unique RD per PE per VRF.

Example:

```text
PE1 CUSTOMER_A: rd 1.1.1.1:100
PE2 CUSTOMER_A: rd 4.4.4.4:100
```

or:

```text
PE1 CUSTOMER_A: rd 65000:101
PE2 CUSTOMER_A: rd 65000:102
```

This matters because the RD is part of the VPNv4 NLRI.

If two PEs advertise the same IPv4 prefix with the same RD, MP-BGP sees them as the same VPNv4 route.

If two PEs advertise the same IPv4 prefix with different RDs, MP-BGP sees them as different VPNv4 routes.

---

## Same RD vs Different RD

Assume PE1 and PE2 both advertise this customer prefix:

```text
10.10.10.0/24
```

### Same RD

```text
PE1 advertises:
65000:1:10.10.10.0/24

PE2 advertises:
65000:1:10.10.10.0/24
```

These are the same VPNv4 NLRI.

BGP chooses the best path.

This can be fine if only one best path is needed.

### Different RD

```text
PE1 advertises:
1.1.1.1:1:10.10.10.0/24

PE2 advertises:
4.4.4.4:1:10.10.10.0/24
```

These are different VPNv4 NLRIs.

BGP can keep both VPNv4 routes.

This is useful in designs with multihomed sites, route reflectors, or when you want multiple paths to be visible in the VPNv4 control plane.

---

## RT Design

The RT defines VPN membership.

A simple full-mesh customer VPN usually uses the same import and export RT everywhere.

Example:

```text
PE1 CUSTOMER_A:
 route-target both 1:1

PE2 CUSTOMER_A:
 route-target both 1:1

PE3 CUSTOMER_A:
 route-target both 1:1
```

All sites export routes with RT `1:1`.

All sites import routes with RT `1:1`.

Result:

```text
All CUSTOMER_A sites learn all other CUSTOMER_A site routes.
```

---

## Multiple RTs

A VRF can import and export multiple RTs.

Example:

```text
vrf definition CUSTOMER_A
 rd 65000:1
 address-family ipv4
  route-target export 1:1
  route-target import 1:1
  route-target import 1:999
```

This VRF exports routes with RT `1:1`.

It imports routes with RT `1:1` and `1:999`.

Multiple RTs are used for designs such as:

```text
Hub-and-spoke VPNs
Extranet VPNs
Shared services VPNs
Selective route leaking
```

---

## Hub-and-Spoke Example

RTs can create hub-and-spoke VPN behavior.

Example:

```text
Hub VRF:
 route-target export 65000:100
 route-target import 65000:200

Spoke VRF:
 route-target export 65000:200
 route-target import 65000:100
```

Result:

```text
Hub routes are exported with RT 65000:100.
Spokes import hub routes, but not routes from other spokes.

Spoke routes are exported with RT 65000:200.
The hub imports spoke routes.
```

---

## Extranet Example

An extranet VPN allows selected routes to be shared between different customer VRFs.

Example:

```text
CUSTOMER_A:
 route-target export 65000:100
 route-target import 65000:100
 route-target import 65000:999

CUSTOMER_B :
 route-target export 65000:200
 route-target import 65000:200
 route-target import 65000:999

SHARED_SERVICES:
 route-target export 65000:999
 route-target import 65000:100
 route-target import 65000:200
```

Result:

- CUSTOMER_A imports its own routes and shared services routes.
- CUSTOMER_B imports its own routes and shared services routes.
- CUSTOMER_A and CUSTOMER_B do not import each other's private routes.

---

## Extended Communities and `send-community extended`

Route targets are BGP extended communities.

For a VPNv4 route to carry RTs between PE routers, extended communities must be sent.

In MPLS L3VPN configurations, you will see this under the VPNv4 address family:

```text
router bgp 65000
 address-family vpnv4
  neighbor 4.4.4.4 activate
  neighbor 4.4.4.4 send-community extended
```

If the RT is not sent to the remote PE, the remote PE cannot import the route into the correct VRF.

> In IOS XE, the `send-community extended` command is added automatically when you configure `activate`.

We'll cover details about MP-BGP configuration in other pages.

---

## RD and RT Are Not Data-Plane Labels

RDs and RTs are control-plane values.

They are not MPLS labels.

They are not inserted into the packet as it crosses the MPLS core.

Data-plane forwarding uses MPLS labels.

```text
Transport label:
Gets the packet to the egress PE.

VPN label:
Tells the egress PE how to forward the packet.
```

RDs and RTs help build the VPN routing table.

Labels are used for packet forwardig.

In the following output, you can see the VPN labels associated with the customer prefixes `10.10.10.0/24` and `11.11.11.0/24` from PE1's point of view:

```text
PE1# show bgp vpnv4 unicast all labels
   Network          Next Hop      In label/Out label
Route Distinguisher: 65000:1 (CUSTOMER_A)
   10.10.10.0/24    192.168.1.2     18/nolabel
   11.11.11.0/24    4.4.4.4         nolabel/24
```

For `10.10.10.0/24`, PE1 assigned local VPN label `18`.

PE1 advertises this route and label to its MP-BGP VPNv4 neighbors. When a remote PE sends traffic to `10.10.10.0/24`, it uses label `18` as the VPN label.

For `11.11.11.0/24`, PE1 learned VPN label `24` from the remote PE.

When PE1 sends traffic to `11.11.11.0/24`, it imposes label `24` as the VPN label. PE1 also imposes a transport label to reach the remote PE next hop, `4.4.4.4`.

So, for traffic from CE1 to CE2, PE1 pushes two labels:

```text
Top label:    Transport label to 4.4.4.4
Bottom label: VPN label 24
```

The transport label gets the packet across the MPLS core.

The VPN label tells the egress PE how to forward the packet after it arrives.

> More details on VPN labels in another page.

---

## Key Points

* The RD makes customer routes unique in MP-BGP.
* The RT controls which VRFs import and export routes.
* The RD is part of the VPNv4 or VPNv6 NLRI.
* The RT is a BGP extended community.
* Matching RDs are not required for VPN route import.
* Matching RTs are required for normal VPN route import.
* The VRF name is locally significant and does not need to match between PEs.
* A route can carry multiple RTs.
* A VRF can import and export multiple RTs.
* Same RD plus same IPv4 prefix equals the same VPNv4 NLRI.
* Different RD plus same IPv4 prefix equals different VPNv4 NLRIs.
* Using unique RDs on each PE can improve path visibility in larger designs.
* RDs and RTs are control-plane values, not MPLS labels.
* If a VPNv4 route exists but is not imported into a VRF, check the RTs first.
