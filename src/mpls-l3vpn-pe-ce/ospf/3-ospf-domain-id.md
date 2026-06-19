# OSPF Domain ID

The **OSPF Domain ID** is used when OSPF routes are carried across an MPLS L3VPN.

It helps the PE decide how to advertise a remote VPN route back into OSPF.

The main question is:

```text
Did this route come from the same OSPF domain or from a different OSPF domain?
```

That decision affects the OSPF route type seen by the remote CE.

```text
Same OSPF domain:
Advertise internal OSPF routes as inter-area routes.

Different OSPF domain:
Advertise the route as an external route.
```

This is one of the most important PE-CE OSPF topics for CCIE EI.

---

## Why Domain ID Exists

In normal OSPF, LSAs are flooded inside an OSPF domain.

In MPLS L3VPN, the provider core does not flood customer LSAs across the MPLS backbone.

Instead, the PE routers translate routes between OSPF and MP-BGP VPNv4.

```text
CE1 OSPF
  ↓
PE1 OSPF VRF process
  ↓
MP-BGP VPNv4
  ↓
PE2 OSPF VRF process
  ↓
CE2 OSPF
```

When PE2 receives a VPNv4 route from PE1 and redistributes it into OSPF toward CE2, PE2 needs to decide what kind of OSPF LSA to generate.

For example:

```text
Should PE2 advertise 10.10.10.0/24 as O IA?

or

Should PE2 advertise 10.10.10.0/24 as O E2?
```

The Domain ID helps PE2 make that decision.

---

## What the Domain ID Is

The OSPF Domain ID is carried as a BGP extended community with the VPN route.

When a PE exports an OSPF-learned route into MP-BGP, it attaches OSPF information to the VPNv4 route.

That information can include:

```text
OSPF Domain ID
OSPF route type
OSPF metric
OSPF router ID
```

The Domain ID identifies the OSPF domain from which the route came.

Example:

```text
CE1 advertises 10.10.10.0/24 to PE1 using OSPF.
PE1 exports the route into VPNv4.
PE1 attaches its OSPF Domain ID.
PE2 receives the VPNv4 route.
PE2 compares the received Domain ID to its local OSPF Domain ID.
```

The comparison result affects the OSPF route type advertised to CE2.

---

## Same Domain ID

Assume PE1 and PE2 use the same OSPF Domain ID.

```text
PE1 OSPF Domain ID: 0.0.0.10
PE2 OSPF Domain ID: 0.0.0.10
```

CE1 advertises an internal route to PE1:

```text
10.10.10.0/24
```

PE1 exports it into VPNv4.

PE2 imports it and compares the Domain ID.

```text
Received Domain ID: 0.0.0.10
Local Domain ID:    0.0.0.10
Result:             Same domain
```

Because the route came from the same OSPF domain, PE2 advertises it to CE2 as an inter-area route.

On CE2, the route appears as:

```text
O IA 10.10.10.0/24
```

This is the normal expected result when the customer sites are part of the same OSPF domain.

---

## Different Domain IDs

Now assume PE1 and PE2 use different OSPF Domain IDs.

```text
PE1 OSPF Domain ID: 0.0.0.10
PE2 OSPF Domain ID: 0.0.0.20
```

PE2 receives the VPNv4 route and compares the Domain ID.

```text
Received Domain ID: 0.0.0.10
Local Domain ID:    0.0.0.20
Result:             Different domain
```

Because the route came from a different OSPF domain, PE2 treats it as an external route when redistributing it into OSPF.

On CE2, the route may appear as:

```text
O E2 10.10.10.0/24
```

or, if the PE-CE link is in an NSSA area:

```text
O N2 10.10.10.0/24
```

The route is still reachable.

But the OSPF route type changes.

This can affect path selection because OSPF prefers routes in this general order:

```text
Intra-area
Inter-area
External Type 1
External Type 2
```

So the customer's routing behavior can change depending on whether the domain IDs match.

---

## Internal vs External Route Behavior

The Domain ID mainly affects routes that were originally internal OSPF routes.

If an internal OSPF route is carried across the MPLS VPN:

```text
Same Domain ID:
Advertise as inter-area.

Different Domain ID:
Advertise as external.
```

Example:

```text
CE1 route:
O 10.10.10.0/24

CE2 result with same Domain ID:
O IA 10.10.10.0/24

CE2 result with different Domain ID:
O E2 10.10.10.0/24
```

For routes that were already external when learned by the ingress PE, the route remains external.

Example:

```text
CE1 route:
O E2 10.10.10.0/24

CE2 result:
O E2 10.10.10.0/24
```

It is mainly used to decide whether internal OSPF routes carried through the VPN should remain OSPF-internal from the remote CE's point of view.

---

## OSPF Domain ID Type and Value

The OSPF Domain ID is carried in MP-BGP as a BGP extended community.

A BGP extended community is 8 bytes long.

For the OSPF Domain ID, those 8 bytes are divided like this:

```text
2 bytes:
Type

6 bytes:
Value
```

Example:

```text
OSPF DOMAIN ID:0x0005:0x0000000A0200
```

This means:

```text
Type:  0x0005
Value: 0x0000000A0200
```

Together, the type and value form the complete OSPF Domain ID.

```text
Complete Domain ID:
0x0005:0x0000000A0200
```

---

### Domain ID Types

The OSPF Domain ID types are:

| Domain ID type | Meaning                                           |
| :------------- | :------------------------------------------------ |
| `0005`         | Default OSPFv2 Domain ID type used by IOS XE |
| `0105`         | IPv4-address-specific extended community format   |
| `0205`         | 4-octet AS-specific extended community format     |
| `8005`         | Backward-compatible type, treated like `0005`     |


The details of the domain types are usually not relevant. The important practical point is:

```text
PEs in the same customer OSPF domain should use matching Domain IDs.
```

And for two Domain IDs to match, they must have matching types and values, with a couple of exceptions.

### Exceptions ###

There are two important exceptions. First, type `8005` is treated like type `0005`.

Example:

```
type 0005 value 00000000000A
type 8005 value 00000000000A
```

Result:

```
Domain IDs match.
```

Second, an all-zero value field represents the NULL Domain ID.

Example:

```
type 0005 value 000000000000
type 0105 value 000000000000
type 0205 value 000000000000
type 8005 value 000000000000
```

These all represent the NULL Domain ID. For the NULL Domain ID, the type does not matter.

In most cases, you don't have to worry about the type: keep the default of `0005`. Just be aware that these differnet types exist, and know the above exceptions.

---

### Friendly Display vs Raw VPNv4 Display

Cisco may show the same OSPF Domain ID in two different forms.

In the OSPF process output, IOS XE shows a friendly dotted-decimal value:

```text
PE1# show ip ospf 10
 Routing Process "ospf 10" with ID 192.168.1.1
   Domain ID type 0x0005, value 0.0.0.10
```

This is a 4-byte value represented in dotted-decimal format that is derived from the OSPFv2 process ID by default.

In the VPNv4 BGP table, IOS XE shows the raw BGP extended community:

```text
PE1# show bgp vpnv4 unicast all 10.10.10.0/24
...
Extended Community: RT:1:1 OSPF DOMAIN ID:0x0005:0x0000000A0200
```

These refer to the same Domain ID.

The friendly OSPF value is:

```text
0.0.0.10
```

The raw VPNv4 extended-community value is:

```text
0000000A0200
```

The first 4 bytes correspond to the dotted-decimal value:

```text
0.0.0.10
=
0000000A
```

The final 2 bytes, `0200`, are appended by IOS.

So do not assume that this:

```text
Domain ID value 0.0.0.10
```

is represented in VPNv4 BGP as this:

```text
00000000000A
```

On IOS XE, it is represented as:

```text
0000000A0200
```

---

## OSPFv2 Default Domain ID

In IOS XE, OSPFv2 has a non-null default Domain ID.

The default OSPFv2 Domain ID is based on the OSPF process ID.

Example:

```text
router ospf 10 vrf CUSTOMER_A
```

Default Domain ID shown in the OSPF process:

```text
PE1# show ip ospf 10
 Routing Process "ospf 10" with ID 192.168.1.1
   Domain ID type 0x0005, value 0.0.0.10
```

The same Domain ID may appears the VPNv4 BGP table like this:

```text
OSPF DOMAIN ID:0x0005:0x0000000A0200
```

For process ID 10:

```text
0.0.0.10
↓
0000000A0200
```

If both PEs use the same OSPF process ID for the customer VRF, their default Domain IDs match.

Example:

```text
PE1:
router ospf 10 vrf CUSTOMER_A

PE2:
router ospf 10 vrf CUSTOMER_A

Result:
Domain IDs match by default.
```

Remote internal OSPF routes are advertised as inter-area routes.

```text
CE2# show ip route ospf
O IA 10.10.10.0/24
```

But if the PEs use different process IDs, the default Domain IDs are different.

Example:

```text
PE1:
router ospf 10 vrf CUSTOMER_A

PE2:
router ospf 20 vrf CUSTOMER_A

Result:
Domain IDs do not match by default.
```

Remote internal OSPF routes may be advertised as external routes.

```text
CE2# show ip route ospf
O E2 10.10.10.0/24
```

This is an important detail for CCIE EI.

In normal OSPF, the process ID is locally significant.

In MPLS L3VPN PE-CE OSPF, the process ID can indirectly matter because IOS XE uses it to determine the default OSPFv2 Domain ID.

```text
Normal OSPF:
Process ID is locally significant.

OSPFv2 PE-CE L3VPN:
Process ID is still locally significant for the OSPF process,
but it affects the default Domain ID.
```

---

## Configuring OSPFv2 Domain ID

You can explicitly configure the OSPFv2 Domain ID under the OSPF process.

The simplest method is to configure the dotted-decimal value.

Example on PE1:

```text
PE1(config)# router ospf 10 vrf CUSTOMER_A
PE1(config-router)# domain-id 0.0.0.10
```

Example on PE2:

```text
PE2(config)# router ospf 20 vrf CUSTOMER_A
PE2(config-router)# domain-id 0.0.0.10
```

Even though the OSPF process IDs are different, the Domain IDs now match.

```text
PE1 process ID:     10
PE2 process ID:     20
PE1 Domain ID:      0.0.0.10
PE2 Domain ID:      0.0.0.10
Result:             Same OSPF domain
```

You can also configure a type and value explicitly.

For example, this matches the raw VPNv4 value shown earlier:

```text
router ospf 10 vrf CUSTOMER_A
 domain-id type 0005 value 0000000A0200
```

This is the raw form of:

```text
domain-id 0.0.0.10
```

Do not use this if you are trying to match the default `0.0.0.10` value:

```text
router ospf 10 vrf CUSTOMER_A
 domain-id type 0005 value 00000000000A
```

That is a different raw 6-byte value.

For most labs, use the simpler dotted-decimal form:

```text
router ospf 10 vrf CUSTOMER_A
 domain-id 0.0.0.10
```

Use the `type` and `value` form only when specifically instructed to do so.

---

### OSPFv2 Null Domain ID

OSPFv2 can also use a null Domain ID.

Example:

```text
router ospf 10 vrf CUSTOMER_A
 domain-id null
```

A null Domain ID means the OSPF process is associated with the null domain.

If both the incoming route and the local OSPF process are in the null domain, they are treated as matching.

```text
PE1 Domain ID: NULL
PE2 Domain ID: NULL
Result:        Same domain
```

---

## OSPFv3 Domain ID

OSPFv3 also uses a Domain ID for PE-CE L3VPN route translation.

The concept is the same:

```text
Same Domain ID:
Remote internal OSPFv3 routes can be advertised as inter-area routes.

Different Domain ID:
Remote routes are treated as external.
```

However, the default behavior is different.

For OSPFv3, the default Domain ID is null.

Conceptually:

```text
OSPFv2 default:
Based on the OSPF process ID

OSPFv3 default:
NULL
```

If both OSPFv3 PE processes use the default null Domain ID, they are considered to match.

```text
PE1 OSPFv3 Domain ID: NULL
PE2 OSPFv3 Domain ID: NULL
Result:               Same domain
```

If one side has a configured non-null Domain ID and the other side remains null, they do not match.

```text
PE1 OSPFv3 Domain ID: 000000000001
PE2 OSPFv3 Domain ID: NULL
Result:               Different domain
```

That can cause remote routes to appear as external OSPFv3 routes.

---

## Configuring OSPFv3 Domain ID

The OSPFv3 Domain ID is configured under the OSPFv3 address family.

Example:

```text
router ospfv3 10
 address-family ipv6 unicast vrf CUSTOMER_A
  domain-id type 0005 value 000000000001
```

Configure the same value on the remote PE if both sites belong to the same OSPFv3 domain.

Example on PE1:

```text
PE1(config)# router ospfv3 10
PE1(config-router)# address-family ipv6 unicast vrf CUSTOMER_A
PE1(config-router-af)# domain-id type 0005 value 000000000001
```

Example on PE2:

```text
PE2(config)# router ospfv3 20
PE2(config-router)# address-family ipv6 unicast vrf CUSTOMER_A
PE2(config-router-af)# domain-id type 0005 value 000000000001
```

The OSPFv3 process IDs can differ.

What matters for this decision is the Domain ID.

```text
PE1 OSPFv3 process ID: 10
PE2 OSPFv3 process ID: 20
PE1 Domain ID:         000000000001
PE2 Domain ID:         000000000001
Result:                Same OSPFv3 domain
```

---

## OSPFv2 vs OSPFv3 Defaults

This is the key comparison:

| Version | Cisco default Domain ID | Practical effect |
| :--- | :--- | :--- |
| OSPFv2 | Based on the OSPF process ID | Different process IDs can cause a Domain ID mismatch |
| OSPFv3 | NULL | Two unconfigured PEs match by default |

For OSPFv2, if you use process 10 on PE1 and process 20 on PE2, the default Domain IDs are different.

For OSPFv3, if you do not configure a Domain ID on either PE, both use null and they match.

---

## Same Domain ID Example

Assume this configuration:

PE1:

```text
router ospf 10 vrf CUSTOMER_A
 domain-id 0.0.0.10
 redistribute bgp 65000
```

PE2:

```text
router ospf 20 vrf CUSTOMER_A
 domain-id 0.0.0.10
 redistribute bgp 65000
```

CE1 advertises `10.10.10.0/24` to PE1 as an intra-area route.

PE1 exports it into VPNv4 with Domain ID `0.0.0.10`.

PE2 imports it and compares the Domain ID.

```text
Incoming Domain ID: 0.0.0.10
Local Domain ID:    0.0.0.10
Match:              Yes
```

PE2 advertises the route to CE2 as an inter-area route.

CE2 sees:

```text
CE2# show ip route 10.10.10.0
O IA 10.10.10.0/24 [110/...] via 192.168.2.1
```

The route is not intra-area because LSAs were not flooded across the provider core.

The route is inter-area because the MPLS VPN superbackbone connects the OSPF domains through the PE routers.

---

## Different Domain ID Example

Assume this configuration:

PE1:

```text
router ospf 10 vrf CUSTOMER_A
 domain-id 0.0.0.10
 redistribute bgp 65000
```

PE2:

```text
router ospf 20 vrf CUSTOMER_A
 domain-id 0.0.0.20
 redistribute bgp 65000
```

PE2 receives a VPNv4 route from PE1.

```text
Incoming Domain ID: 0.0.0.10
Local Domain ID:    0.0.0.20
Match:              No
```

PE2 treats the route as external when redistributing it into OSPF.

CE2 sees:

```text
CE2# show ip route

O E2     10.10.10.0 [110/11] via 192.168.2.1, 00:00:02, Ethernet0/0
```

---

## Domain ID and OSPF Areas

The Domain ID is not the same as the OSPF area.

Two PE-CE links can use the same Domain ID but different OSPF areas.

Example:

```text
PE1-CE1 area: 0
PE2-CE2 area: 1
Domain ID:    same
```

The internal OSPF routes carried through the VPN are advertised as inter-area routes, just like when the area ID matches.

---

## Domain ID and Sham Links

A sham link is used when the MPLS VPN path needs to appear as an intra-area OSPF path.

For a sham link to make sense, the PE-CE OSPF links must be in the same OSPF area.

The sites must also be treated as part of the same OSPF domain.

Example:

```text
CE1 --- PE1 ===== MPLS VPN ===== PE2 --- CE2
 |                                        |
 +------------ backdoor link -------------+
```

Without a sham link, routes through the MPLS VPN normally appear as inter-area routes.

```text
O IA
```

A backdoor link inside the same area may provide intra-area routes.

```text
O
```

Because OSPF prefers intra-area over inter-area, the backdoor path is preferred even if its cost is worse.

> The sham link page covers this in detail.

For now, remember:

```text
Domain ID controls whether routes are same-domain or different-domain,
 affecting the route type (inter-area vs external).

Sham links can make the VPN path appear intra-area when needed.
```

---

## Domain ID and Loop Prevention

The Domain ID is sometimes confugred with the Down Bit and Domain Tag, two other OSPF mechanisms for L3VPN.

```text
Domain ID:
Decides same-domain vs different-domain route translation.

Down Bit:
Prevents VPN-learned OSPF routes from being reintroduced into MP-BGP.

Domain Tag:
Loop prevention for external OSPF routes.
```

The Domain ID affects route type.

The Down Bit and Domain Tag are loop prevention mechanisms.

> These concepts are covered in their own pages.

---

## Verification Commands

### Check the OSPF Process

For OSPFv2:

```text
PE1# show ip ospf 10
 Routing Process "ospf 10" with ID 192.168.1.1
   Domain ID type 0x0005, value 0.0.0.10
 Start time: 00:01:15.827, Time elapsed: 02:28:12.711
```

For OSPFv3:

```
PE1# show ospfv3 ipv6 vrf CUSTOMER_A
 OSPFv3 11 address-family ipv6 vrf CUSTOMER_A
 Router ID 192.168.1.1
 Supports NSSA (compatible with RFC 3101)
 Supports Database Exchange Summary List Optimization (RFC 5243)
 Domain ID 0x000000000001, type 0x0005
 Connected to MPLS VPN Superbackbone
```

### Check the VPNv4/v6 Route:

For OSPFv2:

```text
PE1# show bgp vpnv4 unicast vrf CUSTOMER_A 10.10.10.0/24 
BGP routing table entry for 65000:1:10.10.10.0/24, version 2
Paths: (1 available, best #1, table CUSTOMER_A)
  Advertised to update-groups:
     4         
  Refresh Epoch 1
  Local
    192.168.1.2 (via vrf CUSTOMER_A) from 0.0.0.0 (1.1.1.1)
      Origin incomplete, metric 11, localpref 100, weight 32768, valid, sourced, best
      Extended Community: RT:1:1 OSPF DOMAIN ID:0x0005:0x0000000A0200 
        OSPF RT:0.0.0.0:2:0 OSPF ROUTER ID:192.168.1.1:0
      mpls labels in/out 21/nolabel
      rx pathid: 0, tx pathid: 0x0
      Updated on Jun 17 2026 22:39:54 UTC
```

For OSPFv3:

```
PE1# show bgp vpnv6 unicast all 2001:db8:1::/64         
BGP routing table entry for [65000:1]2001:DB8:1::/64, version 10
Paths: (1 available, best #1, table CUSTOMER_A)
  Advertised to update-groups:
     1         
  Refresh Epoch 1
  Local
    FE80::A8BB:CCFF:FE00:2800 (FE80::A8BB:CCFF:FE00:2800) (via vrf CUSTOMER_A) from 0.0.0.0 (1.1.1.1)
      Origin incomplete, metric 11, localpref 100, weight 32768, valid, sourced, best
      Extended Community: RT:1:1 OSPF DOMAIN ID:0x0005:0x000000000001 
        OSPF ROUTER ID:192.168.1.1:0 OSPF RT:0.0.0.0:2:0
      mpls labels in/out 25/nolabel
      rx pathid: 0, tx pathid: 0x0
      Updated on Jun 18 2026 00:09:57 UTC
```

### Check the Route Type on the CE

For OSPFv2:

```
CE2# show ip route ospf
Codes: L - local, C - connected, S - static, R - RIP, M - mobile, B - BGP
       D - EIGRP, EX - EIGRP external, O - OSPF, IA - OSPF inter area 
       N1 - OSPF NSSA external type 1, N2 - OSPF NSSA external type 2
       E1 - OSPF external type 1, E2 - OSPF external type 2, m - OMP
       n - NAT, Ni - NAT inside, No - NAT outside, Nd - NAT DIA
       i - IS-IS, su - IS-IS summary, L1 - IS-IS level-1, L2 - IS-IS level-2
       ia - IS-IS inter area, * - candidate default, U - per-user static route
       H - NHRP, G - NHRP registered, g - NHRP registration summary
       o - ODR, P - periodic downloaded static route, l - LISP
       a - application route
       + - replicated route, % - next hop override, p - overrides from PfR
       & - replicated local route overrides by connected

Gateway of last resort is not set

      10.0.0.0/24 is subnetted, 2 subnets
O IA     10.10.10.0 [110/21] via 192.168.2.1, 00:09:08, Ethernet0/0
```

For OSPFv3:

```
CE2#show ipv6 route ospf 
IPv6 Routing Table - default - 4 entries
Codes: C - Connected, L - Local, S - Static, U - Per-user Static route
       B - BGP, R - RIP, H - NHRP, HG - NHRP registered
       Hg - NHRP registration summary, HE - NHRP External, I1 - ISIS L1
       I2 - ISIS L2, IA - ISIS interarea, IS - ISIS summary, D - EIGRP
       EX - EIGRP external, ND - ND Default, NDp - ND Prefix, DCE - Destination
       NDr - Redirect, RL - RPL, O - OSPF Intra, OI - OSPF Inter
       OE1 - OSPF ext 1, OE2 - OSPF ext 2, ON1 - OSPF NSSA ext 1
       ON2 - OSPF NSSA ext 2, la - LISP alt, lr - LISP site-registrations
       ld - LISP dyn-eid, lA - LISP away, le - LISP extranet-policy
       lp - LISP publications, ls - LISP destinations-summary, a - Application
       m - OMP
OI  2001:DB8:1::/64 [110/21]
     via FE80::A8BB:CCFF:FE00:1510, Ethernet0/0
```

Becuase I configured matching Domain IDs, the routes appear are inter-area.

---

## Common Problems

### Different OSPFv2 Process IDs

Problem:

```text
PE1 uses router ospf 10 vrf CUSTOMER_A.
PE2 uses router ospf 20 vrf CUSTOMER_A.
```

With default OSPFv2 Domain IDs, the process IDs produce different Domain IDs.

Result:

```text
Remote routes appear as O E2 instead of O IA.
```

Fix:

```text
router ospf 10 vrf CUSTOMER_A
 domain-id 10.10.10.10
```

```text
router ospf 20 vrf CUSTOMER_A
 domain-id 10.10.10.10
```

### One OSPFv3 PE Has a Domain ID, the Other Does Not

Problem:

```text
PE1 OSPFv3 Domain ID: configured
PE2 OSPFv3 Domain ID: default null
```

Result:

```text
Domain IDs do not match.
Remote routes may appear as external.
```

Fix:

```text
Configure the same OSPFv3 Domain ID on both PEs.
```

Example:

```text
router ospfv3 10
 address-family ipv6 unicast vrf CUSTOMER_A
  domain-id type 0005 value 000000000001
```

### Confusing Domain ID with RT

Problem:

```text
The VPNv4 route imports correctly, but the CE sees O E2 instead of O IA.
```

This is not an RT problem.

The RT already worked because the route entered the VRF.

This is likely an OSPF Domain ID or route-type issue.

### Expecting Intra-Area Routes Without Sham Links

Problem:

```text
The Domain IDs match, but the remote CE sees O IA instead of O.
```

This is normal.

Having the Domain ID does not make the MPLS VPN path intra-area, even if both PE-CE links are in the same area.

To make the VPN path appear intra-area, you need a sham link.

---

## Key Points

* The OSPF Domain ID is used when OSPF routes are carried across MPLS L3VPN using MP-BGP.
* It is carried with the VPN route as a BGP extended community.
* The receiving PE compares the incoming Domain ID with the local OSPF Domain ID.
* Matching Domain IDs mean the route came from the same OSPF domain.
* Different Domain IDs mean the route came from a different OSPF domain.
* Same-domain internal routes are advertised to the remote CE as inter-area routes.
* Different-domain routes are advertised to the remote CE as external routes.
* OSPFv2 on Cisco uses a default Domain ID based on the OSPF process ID.
* OSPFv3 uses a default null Domain ID.
* Sham links are required if the MPLS VPN path must appear as an intra-area path.