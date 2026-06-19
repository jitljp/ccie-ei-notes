# OSPF Route Types in L3VPN

OSPF route type is one of the most important details in PE-CE OSPF over MPLS L3VPN.

A route that enters the MPLS VPN from one CE can appear on the remote CE as one of several OSPF route types.

Examples:

```text
O
O IA
O E1
O E2
O N1
O N2
```

The result depends on several things:

```text
Original OSPF route type
OSPF Domain ID
Area type
Sham link usage
Redistribution policy
```

The goal of this page is to answer this question:

```text
When a route crosses the MPLS VPN, what OSPF route type does the remote CE see?
```

---

## Normal OSPF Route Type Preference

OSPF does not compare all routes only by metric.

It compares the route type first.

The general order is:

```text
1. Intra-area routes
2. Inter-area routes
3. External type 1 routes
4. External type 2 routes
```

In Cisco output:

| Code | Meaning | LSA type |
| :--- | :--- | :--- |
| `O` | Intra-area | Type 1 / Type 2 |
| `O IA` | Inter-area | Type 3 |
| `O E1` | External Type 1 | Type 5 |
| `O E2` | External Type 2 | Type 5 |
| `O N1` | NSSA Type 1 | Type 7 |
| `O N2` | NSSA Type 2 | Type 7 |

This matters in L3VPN because the route type can change when a route crosses the MPLS VPN.

Example:

```text
Backdoor path:
O 10.10.10.0/24, cost 100

MPLS VPN path:
O IA 10.10.10.0/24, cost 20
```

The backdoor path is preferred because `O` is better than `O IA`.

The lower metric on the `O IA` route does not matter because the route type is less preferred.

---

## Why L3VPN Changes the Route Type

In normal OSPF, LSAs are flooded through the OSPF domain.

In MPLS L3VPN PE-CE OSPF, customer LSAs are not flooded across the provider core.

Instead, the PE routers translate between OSPF and MP-BGP VPNv4.

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

Because the route is translated, the remote PE must decide what type of OSPF LSA to generate toward the CE.

That decision is based mainly on:

```text
Was the original route internal or external?
Did the Domain ID match?
Is the PE-CE link in a normal area or NSSA?
Is there a sham link?
```

---

## The Main Route Type Rules

### Internal route, same Domain ID

If a route was learned from OSPF as an internal route and the Domain IDs match, the remote PE advertises the route as inter-area.

```text
Original route at CE1:
O or O IA

Remote CE result:
O IA
```

Example:

```text
CE1:
O 10.10.10.0/24

CE2:
O IA 10.10.10.0/24
```

This is normal superbackbone behavior.

The route is still treated as internal to the customer OSPF domain, but it crossed the MPLS VPN superbackbone, so the remote PE advertises it as a summary route.

### Internal route, different Domain ID

If a route was learned from OSPF as an internal route but the Domain IDs do not match, the remote PE treats it as external.

```text
Original route at CE1:
O or O IA

Remote CE result:
O E2 by default
```

Example:

```text
CE1:
O 10.10.10.0/24

CE2:
O E2 10.10.10.0/24
```

If the remote PE-CE link is in an NSSA area, the route may be advertised as an NSSA route instead.

```text
Remote CE result in NSSA:
O N2
```

### External route

If the route was already external when it entered the MPLS VPN, it remains external.

```text
Original route at CE1:
O E1 or O E2

Remote CE result:
O E1 or O E2
```

If the remote PE-CE link is in an NSSA area, the PE advertises the external route as an NSSA external route.

```text
Remote CE result in NSSA:
O N1 or O N2
```

### NSSA route

NSSA routes are treated like external routes when carried through the MPLS VPN.

```text
Original route at CE1:
O N1 or O N2
```

The remote result depends on the area type toward the CE.

```text
Remote normal area:
O E1 or O E2

Remote NSSA area:
O N1 or O N2
```

### Sham link route

A sham link is the exception to the normal superbackbone route-type behavior.

With a sham link, the PEs form an OSPF adjacency and exchange LSAs directly over the VPN.

This can make the VPN path appear as an intra-area path.

```text
Without sham link:
O IA

With sham link:
O
```

This is why sham links are used when a customer backdoor link is preferred because it is intra-area.

---

## Route Type Summary

| Original route entering VPN | Domain ID | Remote PE-CE area | Remote CE route type |
| :--- | :--- | :--- | :--- |
| `O` | Same | Normal area | `O IA` |
| `O IA` | Same | Normal area | `O IA` |
| `O` or `O IA` | Different | Normal area | `O E2` by default |
| `O` or `O IA` | Different | NSSA | `O N2` by default |
| `O E1` | Any | Normal area | `O E1` |
| `O E2` | Any | Normal area | `O E2` |
| `O E1` | Any | NSSA | `O N1` |
| `O E2` | Any | NSSA | `O N2` |
| `O N1` | Any | Normal area | `O E1` |
| `O N2` | Any | Normal area | `O E2` |
| `O N1` | Any | NSSA | `O N1` |
| `O N2` | Any | NSSA | `O N2` |
| `O` with sham link | Same | Same area | `O` |

---

## OSPF Route Type Extended Community

When a PE exports an OSPF route into MP-BGP VPNv4, it attaches OSPF information to the VPNv4 route.

One of these extended communities is the **OSPF Route Type** extended community.

In Cisco output, this appears as `OSPF RT`.

Do not confuse this with the route target.

```text
RT:1:1
OSPF RT:0.0.0.0:2:0
```

These are different things.

```text
RT:1:1
Route Target.
Used for VPN route import/export.

OSPF RT:0.0.0.0:2:0
OSPF Route Type.
Used to preserve OSPF route information.
```

Example VPNv4 output:

```text
PE1# show bgp vpnv4 unicast all 10.10.10.0/24
BGP routing table entry for 65000:1:10.10.10.0/24, version 2
Paths: (1 available, best #1, table CUSTOMER_A)
  Local
    192.168.1.2 (via vrf CUSTOMER_A) from 0.0.0.0 (1.1.1.1)
      Origin incomplete, metric 11, localpref 100, weight 32768, valid, sourced, best
      Extended Community: RT:1:1 OSPF DOMAIN ID:0x0005:0x0000000A0200
        OSPF RT:0.0.0.0:2:0 OSPF ROUTER ID:192.168.1.1:0
      mpls labels in/out 21/nolabel
```

Focus on this part:

```text
OSPF RT:0.0.0.0:2:0
```

It has three fields:

```text
Area ID:
0.0.0.0

Route type:
2

Options:
0
```

The OSPF Route Type extended community is encoded as:

```text
4 bytes:
Area number

1 byte:
Route type

1 byte:
Options
```

---

## OSPF Route Type Codes

The route type field uses these values:

| Code | Meaning |
| :--- | :--- |
| `1` | Intra-area route, from a Type 1 LSA |
| `2` | Intra-area route, from a Type 2 LSA |
| `3` | Inter-area route |
| `5` | External route |
| `7` | NSSA route |

Example:

```text
OSPF RT:0.0.0.0:2:0
```

This means:

```text
Area ID:
0.0.0.0

Original OSPF route type:
Intra-area

Route type code:
2
```

> In IOS XE, it seems you'll always see `2` here for an intra-area route,
> even if the prefix was derived from a type 1 LSA.

But this does not mean the remote CE must see the route as `O`.

For route type codes `1`, `2`, or `3`, the remote PE still checks the Domain ID.

```text
Same Domain ID:
Advertise as O IA, unless a sham link is used.

Different Domain ID:
Advertise as O E2 by default.
```

This is a key point.

The `OSPF RT` field tells the remote PE what kind of OSPF route entered the VPN.

It does not always directly equal the route code shown on the remote CE.

---

## Options Field

The options field is mainly relevant for external and NSSA routes.

For external routes, it indicates the external metric type.

```text
Type 1 metric:
O E1 or O N1

Type 2 metric:
O E2 or O N2
```

For internal routes, the options field is usually not important.

Example:

```text
OSPF RT:0.0.0.0:2:0
```

The route type is `2`, which means intra-area.

The options value is `0`, but it is not important for this internal route.

For an external route, you might see `0` or `1`:

```
0 = Type 1 metric
1 = Type 2 metric
```

Example:

```
PE1# show bgp vpnv4 unicast all 10.10.20.0
BGP routing table entry for 65000:1:10.10.20.0/24, version 8
Paths: (1 available, best #1, table CUSTOMER_A)
  Advertised to update-groups:
     4         
  Refresh Epoch 1
  Local
    192.168.1.2 (via vrf CUSTOMER_A) from 0.0.0.0 (1.1.1.1)
      Origin incomplete, metric 20, localpref 100, weight 32768, valid, sourced, best
      Extended Community: RT:1:1 OSPF DOMAIN ID:0x0005:0x0000000A0200 
        OSPF RT:0.0.0.0:5:1 OSPF ROUTER ID:192.168.1.1:0
```

---

## Same Domain ID Example

Assume both PEs use the same Domain ID.

PE1:

```text
router ospf 10 vrf CUSTOMER_A
 domain-id 0.0.0.10
```

PE2:

```text
router ospf 10 vrf CUSTOMER_A
 domain-id 0.0.0.10
```

CE1 advertises `10.10.10.0/24` as an intra-area route.

On PE1:

```text
PE1# show ip route vrf CUSTOMER_A ospf
O        10.10.10.0 [110/11] via 192.168.1.2, Ethernet0/1
```

PE1 exports the route into VPNv4 and sends it to PE2.

```text
PE2#show bgp vpnv4 unicast all 10.10.10.0/24
...
      Extended Community: RT:1:1 OSPF DOMAIN ID:0x0005:0x0000000A0200 
        OSPF RT:0.0.0.0:2:0 OSPF ROUTER ID:192.168.1.1:0
```

Because the Domain IDs match, PE2 advertises the route to CE2 as inter-area.

```text
CE2# show ip route ospf
O IA     10.10.10.0 [110/21] via 192.168.2.1, Ethernet0/0
```

The route entered the VPN as intra-area.

The route leaves the VPN as inter-area.

```text
Ingress CE route:
O

VPNv4 OSPF RT:
0.0.0.0:2:0

Remote CE route:
O IA
```

---

## Different Domain ID Example

Now assume the PEs use different Domain IDs.

PE1:

```text
router ospf 10 vrf CUSTOMER_A
 domain-id 0.0.0.10
```

PE2:

```text
router ospf 10 vrf CUSTOMER_A
 domain-id 0.0.0.20
```

PE2 receives the VPNv4 route from PE1.

The route entered the VPN as an internal OSPF route.

But the Domain IDs do not match.

```text
Incoming Domain ID:
0.0.0.10

Local Domain ID:
0.0.0.20

Result:
Different OSPF domain
```

PE2 treats the route as external when advertising it to CE2.

```text
CE2# show ip route ospf
O E2     10.10.10.0 [110/11] via 192.168.2.1, Ethernet0/0
```

The route entered the VPN as intra-area.

The route leaves the VPN as external.

```text
Ingress CE route:
O

VPNv4 OSPF RT:
0.0.0.0:2:0


Remote CE route:
O E2
```

---

## Same Area Does Not Mean Intra-Area Across the VPN

A common misconception is that if CE1 and CE2 both use area 0, remote routes should be intra-area.

Example:

```text
CE1 --- PE1 === MPLS VPN === PE2 --- CE2
   area 0                       area 0
```

CE1 and CE2 both use area 0.

But without a sham link, they do not share an LSDB.

The Type 1 and Type 2 LSAs from CE1 are not flooded across the provider core to CE2.

So CE2 sees CE1's routes as inter-area.

```text
CE2# show ip route ospf
O IA     10.10.10.0 [110/21] via 192.168.2.1, Ethernet0/0
```

This is true even if both sites use the same area number.

```text
Same area number:
Does not mean same LSDB across the MPLS VPN.

Same Domain ID:
Means the remote internal route can be treated as same-domain and advertised as O IA.

Sham link:
Creates an actual OSPF adjacency between PEs so the VPN path can be intra-area.
```

---

## Sham Link Effect on Route Types

A sham link changes the route type behavior.

Without a sham link, remote same-domain internal routes appear as inter-area.

```text
CE2# show ip route ospf
O IA     10.10.10.0 [110/21] via 192.168.2.1, Ethernet0/0
```

With a sham link, the PEs form an OSPF adjacency over the VPN.

They exchange LSAs directly over that adjacency.

The remote route can then appear as intra-area.

```text
CE2# show ip route ospf
O        10.10.10.0 [110/22] via 192.168.2.1, Ethernet0/0
```

This is the reason sham links solve the backdoor link problem.

```text
Without sham link:
VPN path = O IA
Backdoor path = O
Backdoor wins by route type.

With sham link:
VPN path = O
Backdoor path = O
Lowest cost wins.
```

A sham link does not simply change a tag.

It creates an OSPF adjacency between the PEs inside the VRF, and LSAs are flooded over that logical link.

---

## External Routes and NSSA Areas

External routes remain external across the VPN.

Example on CE1:

```text
CE1# show ip route ospf
O E2     172.16.10.0 [110/20] via 10.1.1.1, Ethernet0/1
```

PE1 exports the route into VPNv4 with OSPF route type information.

PE2 imports the VPNv4 route.

If the PE2-CE2 link is in a normal area, CE2 sees an external route.

```text
CE2# show ip route ospf
O E2     172.16.10.0 [110/20] via 192.168.2.1, Ethernet0/0
```

If the PE2-CE2 link is in an NSSA area, CE2 sees an NSSA route.

```text
CE2# show ip route ospf
O N2     172.16.10.0 [110/20] via 192.168.2.1, Ethernet0/0
```

The same logic applies to Type 1 metrics.

```text
O E1 → O E1 or O N1
O E2 → O E2 or O N2
```

The area type toward the receiving CE determines whether the route is advertised as Type 5 external or Type 7 NSSA external.

---

## Key Points

* OSPF route type takes precedence over metric during OSPF path selection.
* In PE-CE OSPF, routes are translated between OSPF and MP-BGP VPNv4 or VPNv6.
* OSPF LSAs are not normally flooded end to end across the MPLS core.
* OSPF Route Type information is carried in BGP extended communities.
* Cisco displays this as `OSPF RT`, not to be confused with the VPN route target.
* Same-domain internal OSPF routes normally appear on the remote CE as `O IA`.
* Different-domain internal OSPF routes normally appear as `O E2` or `O N2`.
* External routes remain external across the VPN.
* A sham link creates an OSPF adjacency between PEs and can make the VPN path appear as intra-area.
* Same area number does not mean same LSDB across the MPLS VPN.
* Domain ID, route type, area type, and sham link behavior together determine the final route type.
