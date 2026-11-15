# Accumulated IGP (AIGP)

**Accumulated Interior Gateway Protocol (AIGP)** is an optional nontransitive BGP path attribute that allows BGP to select paths based on accumulated IGP metrics.

AIGP is primarily designed for networks containing multiple routing domains under the **same administrative control**, such as large service provider networks using multiple IGP domains or BGP autonomous systems.

Unlike the normal BGP best path selection process, which considers AS_PATH length before the IGP metric to the next hop, **AIGP is evaluated before AS_PATH length**.

This allows BGP to prefer a path with a shorter total IGP cost, even if its AS_PATH is longer.

| Property | Value |
|---|---|
| Attribute name | Accumulated IGP (AIGP) |
| Attribute type code | 26 |
| Category | Optional nontransitive |
| Attribute flags | `0x80` (without extended length) |
| AIGP TLV type | 1 |
| AIGP TLV length | 11 bytes |
| AIGP metric size | 8 bytes (64 bits) |
| Preferred value | Lowest |
| Selection order | After local origination, before AS_PATH length |
| Defined in | RFC 7311 |

The AIGP attribute contains a **Type-Length-Value (TLV)** structure. Its AIGP metric is encoded as an unsigned 64-bit integer, allowing accumulated costs to exceed the limits of an individual 32-bit IGP metric.

However, Cisco IOS XE's `set aigp-metric` command accepts a manually configured value from 0 to 4294967295 (32 bits).

---

## Why AIGP Is Needed

Traditional IGPs such as OSPF and IS-IS calculate shortest paths based on cumulative link metrics.

However, BGP does not normally select paths based on the total IGP cost across multiple routing domains.

Consider a provider network divided into two routing domains:

```text
                 Provider Network
              (One Administration)

          IGP Domain 1       IGP Domain 2
         +------------+     +------------+
         |            |     |            |
        PE1 --- P1 --- ASBR1 --- P2 --- PE2
         |            |     |            |
         +------------+     +------------+
                        BGP

                   Destination
                    at PE2
```

Each IGP domain knows its own internal topology and costs, but the IGP metrics are not necessarily carried end-to-end across the BGP boundary.

Without AIGP, BGP might prefer a route because it has:

- A shorter AS_PATH.
- A lower MED.
- A lower local IGP metric to its NEXT_HOP.

However, none of these necessarily identifies the path with the lowest **end-to-end IGP cost**.

AIGP addresses this limitation by carrying accumulated metric information in BGP UPDATE messages.

**Important:** AIGP is intended for networks where the accumulated metrics are comparable. Mixing unrelated metric calculations, such as OSPF cost and EIGRP composite metrics, can produce undesirable results.

---

## AIGP Metric Calculation

When comparing paths carrying AIGP, BGP considers both:

1. The AIGP metric received in the BGP UPDATE.
2. The local IGP metric to reach the BGP NEXT_HOP.

The basic calculation is:

```text
Total AIGP Cost = Received AIGP Metric + IGP Metric to NEXT_HOP
```

The path with the **lowest total AIGP cost** is preferred, assuming earlier selection criteria are equal.

### Example

R1 receives two paths to `192.0.2.0/24` through R2 and R3.

```text
              R2
              |
              | IGP Cost: 20
              |
              R1
              |
              | IGP Cost: 10
              |
              R3

        Destination: 192.0.2.0/24
        Advertised through R2 and R3
```

Assume:

| Attribute | Via R2 | Via R3 |
|---|---|---|
| Received AIGP metric | 100 | 50 |
| IGP metric to NEXT_HOP | 20 | 10 |
| Total AIGP cost | 120 | 60 |
| AS_PATH length | 1 | 3 |

R1 calculates:

```text
Via R2:
100 + 20 = 120

Via R3:
50 + 10 = 60
```

**R1 prefers the path through R3**, even though it has a longer AS_PATH.

This is because AIGP is evaluated before AS_PATH length.

### AIGP vs. LOCAL_PREF

AIGP does not override earlier best-path criteria.

For example:

| Attribute | Path A | Path B |
|---|---|---|
| Weight | 0 | 0 |
| LOCAL_PREF | 200 | 100 |
| Total AIGP cost | 100 | 50 |

**Path A wins**, despite having a higher AIGP cost, because LOCAL_PREF is evaluated first.

---

## AIGP in the Best Path Selection Algorithm

Cisco IOS XE evaluates AIGP after the locally originated route preference and before AS_PATH length.

The relevant portion of the algorithm is:

| Order | Criterion | Preferred Path |
|---|---|---|
| 1 | Cost Community (`pre-bestpath`) | Lowest cost, when applicable |
| 2 | Weight | Highest |
| 3 | LOCAL_PREF | Highest |
| 4 | Local origination | Locally originated |
| 5 | AIGP | Lowest accumulated cost |
| 6 | AS_PATH length | Shortest |
| 7 | ORIGIN | IGP > EGP > Incomplete |
| 8 | MED | Lowest, when compared |

See [BGP Best Path Selection Steps](../best-path/selection-steps.md) for the complete algorithm.

### AIGP Path vs. Non-AIGP Path

By default, Cisco IOS XE prefers an eligible path carrying AIGP over an otherwise comparable path that does not carry AIGP.

For example:

| Attribute | Path A | Path B |
|---|---|---|
| LOCAL_PREF | 100 | 100 |
| AIGP | 200 | Not present |
| AS_PATH length | 3 | 1 |

**Path A wins**, even though its AS_PATH is longer.

AIGP is not treated as having a default cost of 0 when the attribute is missing.

### Configuring `bgp bestpath aigp ignore`

Cisco IOS XE supports the following command:

```text
R1(config)# router bgp 65001
R1(config-router)# bgp bestpath aigp ignore
```

This modifies the comparison between a path carrying AIGP and one without AIGP.

With this command configured:

- **Both paths have AIGP:** BGP still compares their AIGP metrics.
- **Only one path has AIGP:** BGP skips AIGP comparison and proceeds to subsequent criteria.
- **Neither path has AIGP:** Normal best-path selection applies.

**The command does not completely disable AIGP comparison.** It prevents AIGP from automatically winning when only one candidate path carries the attribute.

---

## AIGP Advertisement and Metric Accumulation

AIGP is designed to preserve accumulated path cost as routes are advertised through a network.

The way the AIGP value changes depends on whether the advertising router modifies the BGP NEXT_HOP.

### NEXT_HOP Is Unchanged

When a router advertises an AIGP route without changing its BGP NEXT_HOP, the AIGP value is preserved.

For example:

```text
R1 receives:
    NEXT_HOP: 10.10.10.10
    AIGP:     50

R1 advertises:
    NEXT_HOP: 10.10.10.10
    AIGP:     50
```

R1 does not add its own IGP cost because the next hop remains unchanged.

This commonly applies to IBGP advertisement when the original next hop is preserved.

### NEXT_HOP Is Changed

When a router changes the BGP NEXT_HOP to itself, it must account for the cost of reaching the previous next hop.

For example:

```text
R1 receives:
    NEXT_HOP: 10.10.10.10
    AIGP:     50

R1's IGP cost to 10.10.10.10:
    20

R1 advertises with NEXT_HOP self:
    NEXT_HOP: R1
    AIGP:     70
```

The calculation is:

```text
New AIGP = Received AIGP + Cost to Previous NEXT_HOP

70 = 50 + 20
```

This allows downstream routers to consider the accumulated cost of the earlier network segments.

**Important:** When the next hop is recursively resolved through another BGP route carrying AIGP, the accumulation must also account for that route's AIGP metric.

---

## Configuring AIGP

Cisco IOS XE supports AIGP configuration for individual BGP neighbors and routes.

AIGP processing must be enabled on the relevant BGP sessions.

### Enabling AIGP Between Neighbors

Enable AIGP for an IBGP neighbor:

```text
R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 aigp
```

This enables sending and receiving the AIGP attribute for the neighbor.

Configure AIGP on both sides of the BGP session.

AIGP can also be enabled on supported EBGP sessions **within the same AIGP administrative domain**.

Although AIGP is classified as optional nontransitive, it can be exchanged across AS boundaries when explicitly enabled between cooperating BGP peers.

It should not be advertised to unrelated external autonomous systems.

### Originating AIGP with a Fixed Metric

Use `set aigp-metric` in a route map to assign a specific AIGP metric.

For example:

```text
R1(config)# route-map SET-AIGP permit 10
R1(config-route-map)# set aigp-metric 100

R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# network 192.0.2.0 mask 255.255.255.0 route-map SET-AIGP
```

Assuming `192.0.2.0/24` exists in the local routing table, R1 originates the prefix with an AIGP metric of 100.

The manual metric can be any value from 0 to 4294967295.

### Setting AIGP Based on the IGP Metric

Instead of specifying a fixed value, use:

```text
R1(config-route-map)# set aigp-metric igp-metric
```

This uses the metric of the source route from the routing table.

For example, redistribute OSPF routes into BGP while preserving their IGP metric in AIGP:

```text
R1(config)# route-map SET-AIGP permit 10
R1(config-route-map)# set aigp-metric igp-metric

R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# redistribute ospf 1 route-map SET-AIGP
```

Suppose OSPF provides:

```text
O 192.0.2.0/24 [110/30] via 10.0.12.2, Ethernet0/0
```

The redistributed BGP route can be originated with:

```text
AIGP metric: 30
```

If the underlying OSPF metric changes, the originating router can update the AIGP metric and advertise the changed route.

This is useful when BGP must preserve IGP cost information across routing-domain boundaries.

---

## AIGP and Legacy BGP Routers

Some routers may not support the AIGP attribute.

Cisco IOS XE supports translating AIGP into other BGP attributes when advertising routes to such neighbors.

Two options are:

1. **Cost Community:** Carry the accumulated cost in an extended community.
2. **MED:** Translate the accumulated cost into the MED attribute.

### AIGP to Cost Community

Configure the following command under the BGP address family:

```text
R1(config-router-af)# neighbor 10.0.12.2 aigp send cost-community 100 poi igp-cost transitive
R1(config-router-af)# neighbor 10.0.12.2 send-community extended
```

This translates the accumulated metric into a Cost Community using:

- Community ID: 100
- POI: `igp-cost`
- Transitive Cost Community encoding

The receiving router can then use the Cost Community during its own best-path selection.

The **`igp-cost`** keyword in this AIGP translation command corresponds to the IGP Cost Community POI.

See [BGP Cost Communities and Points of Insertion](../best-path/cost-community.md).

### AIGP to MED

Alternatively:

```text
R1(config-router-af)# neighbor 10.0.12.2 aigp send med
```

This translates AIGP information into MED for the advertised route.

However, MED has different comparison rules:

- MED is evaluated after AS_PATH length.
- MED is normally compared only between paths from the same neighboring AS.
- LOCAL_PREF and other earlier attributes can override MED.

Therefore, translating AIGP to MED does not necessarily preserve the original AIGP path preference.

---

## AIGP Design Considerations

Several considerations are important when deploying AIGP.

### Metric Consistency

AIGP works best when all participating routing domains use comparable metric calculations.

For example, two OSPF domains using consistent cost models may produce meaningful accumulated metrics.

However, mixing unrelated metric models can result in incorrect path preferences.

### Administrative Domain Boundaries

AIGP is intended for routing infrastructure under common administrative control.

It should not be added to ordinary Internet or customer routes that lead outside the AIGP administrative domain.

### Route Availability

AIGP can compare only the paths that BGP actually receives.

If a route reflector advertises only its single best path, a client might not receive alternative paths needed to select the lowest accumulated cost.

Depending on the design, **BGP Additional Paths (Add-Path)** may be required.

### Convergence

Changes in IGP costs can affect accumulated AIGP metrics and trigger BGP best-path reevaluation.

This can increase BGP update activity during IGP instability.

Because of this, AIGP should be implemented with appropriate metric design and convergence considerations.

---

## Verification

### Verify AIGP Neighbor Configuration

```text
R1# show bgp ipv4 unicast neighbors 10.0.12.2 | include AIGP
```

Expected output when enabled:

```text
For address family: IPv4 Unicast
  AIGP is enabled
```

### Verify AIGP Metrics

Display detailed BGP information for a prefix:

```text
R1# show bgp ipv4 unicast 192.0.2.0/24
```

An AIGP-enabled path can include information similar to:

```text
Origin IGP, localpref 100, aigp metric 50, valid, internal, best
Total AIGP metric 60
```

The values represent:

| Field | Meaning |
|---|---|
| `aigp metric 50` | AIGP metric carried by the path |
| `Total AIGP metric 60` | Accumulated metric including local cost to NEXT_HOP |
| `best` | Selected BGP best path |

The exact output format depends on the IOS XE release and platform.

### Verify the IGP Metric to NEXT_HOP

```text
R1# show ip route 10.0.12.2
R1# show ip cef 10.0.12.2 detail
```

These commands help identify how the next hop is resolved.

For example:

```text
O 10.0.12.2/32 [110/10] via 10.0.13.3, Ethernet0/0
```

If the received AIGP metric is 50 and the IGP metric to NEXT_HOP is 10:

```text
Total AIGP = 50 + 10 = 60
```

### Verify BGP Configuration

```text
R1# show running-config | section router bgp
R1# show route-map
```

When troubleshooting, confirm that:

1. The BGP sessions are Established.
2. AIGP is enabled for the required peers.
3. The routes actually contain AIGP metrics.
4. The BGP NEXT_HOP addresses are reachable.
5. Earlier best-path criteria are equal.
6. The AIGP metrics and underlying IGP metrics are correct.

---

## AIGP vs. Other BGP Metrics

| Feature | Purpose | Selection Preference |
|---|---|---|
| AIGP | Accumulated IGP cost across BGP routing domains | Lowest accumulated cost |
| MED | Suggest a preferred entry point into an AS | Lowest MED, when compared |
| IGP metric to NEXT_HOP | Cost to reach the BGP next hop | Lowest IGP metric |
| Cost Community | Influences selection at a configurable POI | Lowest cost at the first distinguishing community ID |

AIGP differs from the normal IGP next-hop metric comparison because **it can account for accumulated costs beyond the local IGP domain**.

It also differs from MED because its comparison occurs before AS_PATH length.

---

## References

- [Cisco: BGP Accumulated IGP](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-routing/b-ip-routing/m_irg-bgp-accumulated-igp.html)
- [Cisco: Configure the AIGP Metric Attribute for BGP](https://www.cisco.com/c/en/us/support/docs/ip/border-gateway-protocol-bgp/119001-configure-aigp-00.html)
- [Cisco: How to Optimize BGP Path Using AIGP](https://www.cisco.com/c/en/us/support/docs/multiprotocol-label-switching-mpls/mpls/221236-how-to-optimize-bgp-path-using-aigp.html)
- [RFC 7311: The Accumulated IGP Metric Attribute for BGP](https://www.rfc-editor.org/rfc/rfc7311.html)
