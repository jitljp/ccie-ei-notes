# BGP Cost Communities and Points of Insertion

The **BGP Cost Community** is an optional nontransitive extended community that influences BGP best path selection by assigning cost values to routes.

Unlike traditional BGP attributes, Cost Communities can influence path selection at different points in the algorithm using a **Point of Insertion (POI)**.

Cisco IOS XE supports two POIs:

| POI | Selection Position | Preferred Value |
|---|---|---|
| `pre-bestpath` | Before Weight | Lowest cost |
| `igp` | After IGP metric to NEXT_HOP | Lowest cost |

The default POI is `igp`.

Cost Communities are primarily used within an AS or BGP confederation to customize route preference. They are also important in **EIGRP MPLS L3VPNs**, where they can preserve EIGRP route preferences across the provider network.

---

## Cost Community Format

A Cost Community contains three important components:

```text
Cost:<POI>:<Community-ID>:<Cost-Value>
```

For example:

```text
Cost:igp:10:100
Cost:pre-bestpath:20:200
```

| Component | Description |
|---|---|
| POI | Determines when the Cost Community is evaluated |
| Community ID | Identifies the Cost Community for comparison |
| Cost Value | Determines preference; lower is preferred |

The valid ranges are:

| Field | Range |
|---|---|
| Community ID | 0-255 |
| Cost Value | 0-4294967295 |

Cost Communities are distinct from ordinary BGP communities such as `65001:100`.

A Cost Community is an **extended community**, not a standard community.

### Configuring Cost Communities

The general IOS XE syntax is:

```text
R1(config-route-map)# set extcommunity cost [igp | pre-bestpath] <community-id> <cost-value>
```

For example:

```text
R1(config)# route-map SET-COST permit 10
R1(config-route-map)# set extcommunity cost igp 10 100
```

This configures:

- POI: `igp`
- Community ID: 10
- Cost: 100

Because `igp` is the default POI, the following is equivalent:

```text
R1(config-route-map)# set extcommunity cost 10 100
```

---

## Cost Community Comparison Rules

When multiple BGP paths contain Cost Communities at the same POI, Cisco IOS XE compares them using their community IDs and cost values.

The comparison process is:

1. Collect the Cost Communities for the applicable POI.
2. Compare the lowest community ID first.
3. For that ID, prefer the path with the lowest cost value.
4. If the costs are equal, continue to the next-lowest community ID.
5. If all applicable Cost Communities compare equally, continue with the normal BGP best path selection algorithm.

**The community ID determines the comparison order, while the cost value determines preference at each ID.**

### Example 1: Same Community ID

Consider two paths to `192.0.2.0/24`.

| Attribute | Path A | Path B |
|---|---|---|
| POI | `igp` | `igp` |
| Community ID | 10 | 10 |
| Cost | 100 | 200 |

Both paths use community ID 10.

**Path A is preferred** because its cost value is lower.

### Example 2: Multiple Community IDs

Suppose the paths contain multiple Cost Communities:

| Community ID | Path A Cost | Path B Cost |
|---|---|---|
| 10 | 100 | 100 |
| 20 | 300 | 200 |
| 30 | 50 | 100 |

The comparison proceeds as follows:

1. Community ID 10: Both paths have cost 100. Continue.
2. Community ID 20: Path A has cost 300, while Path B has cost 200.
3. **Path B is preferred.**

Community ID 30 is not evaluated because ID 20 already determined the preferred path.

---

## Default Cost Value

When a path does not contain a Cost Community for an ID being evaluated, BGP assigns a default comparison cost of:

```text
2147483647
```

This is the midpoint of the valid 32-bit unsigned cost range.

### Example: Cost Community vs. No Cost Community

| Attribute | Path A | Path B |
|---|---|---|
| Cost Community | `igp:10:100` | Not configured |
| Comparison Cost | 100 | 2147483647 |

**Path A is preferred**, assuming earlier best-path criteria are equal.

However, the default cost is not the worst possible value.

For example:

| Attribute | Path A | Path B |
|---|---|---|
| Cost Community | `igp:10:3000000000` | Not configured |
| Comparison Cost | 3000000000 | 2147483647 |

**Path B is preferred**, because its default comparison cost is lower.

Therefore:

- Cost values below 2147483647 are preferred over the default.
- Cost values above 2147483647 are less preferred than the default.
- A cost equal to 2147483647 ties with the default at that community ID.

---

## Point of Insertion: `pre-bestpath`

The **`pre-bestpath` POI** is evaluated before the normal BGP best path selection criteria, including Weight and LOCAL_PREF.

The beginning of the algorithm becomes:

```text
1. Cost Community: pre-bestpath
2. Weight
3. LOCAL_PREF
4. Local origination
5. AIGP
6. AS_PATH length
...
```

This makes `pre-bestpath` particularly powerful.

### Example

R1 receives two BGP paths:

| Attribute | Path A | Path B |
|---|---|---|
| Cost Community | `pre-bestpath:10:100` | `pre-bestpath:10:200` |
| Weight | 0 | 500 |
| LOCAL_PREF | 100 | 200 |
| AS_PATH length | 3 | 1 |

**Path A is preferred**, despite having:

- Lower Weight
- Lower LOCAL_PREF
- Longer AS_PATH

This is because `pre-bestpath` is evaluated before all of those criteria.

### Configuring `pre-bestpath`

Configure an inbound route map:

```text
R1(config)# ip prefix-list PFX permit 192.0.2.0/24

R1(config)# route-map SET-COST permit 10
R1(config-route-map)# match ip address prefix-list PFX
R1(config-route-map)# set extcommunity cost pre-bestpath 10 100

R1(config)# route-map SET-COST permit 20

R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 route-map SET-COST in
```

This assigns the `pre-bestpath` Cost Community to `192.0.2.0/24` received from `10.0.12.2`.

Other routes are permitted by route-map sequence 20.

**Important:** Because `pre-bestpath` is evaluated before Weight, it can influence the selection of paths even when their Weight and LOCAL_PREF values differ.

---

## Point of Insertion: `igp`

The **`igp` POI** is evaluated after BGP compares the IGP metric to the NEXT_HOP.

The relevant portion of the algorithm is:

```text
 8. MED
 9. EBGP over IBGP
10. IGP metric to NEXT_HOP
11. Cost Community: igp
12. Oldest EBGP path
...
```

Unlike `pre-bestpath`, the `igp` POI cannot override earlier criteria such as Weight, LOCAL_PREF, AS_PATH length, or IGP metric to NEXT_HOP.

### Example: Equal IGP Metrics

R1 receives two paths:

| Attribute | Path A | Path B |
|---|---|---|
| IGP metric to NEXT_HOP | 20 | 20 |
| Cost Community | `igp:10:100` | `igp:10:200` |

Assuming all earlier criteria are equal:

**Path A is preferred**, because its IGP Cost Community has a lower cost.

### Example: Different IGP Metrics

Now consider:

| Attribute | Path A | Path B |
|---|---|---|
| IGP metric to NEXT_HOP | 30 | 10 |
| Cost Community | `igp:10:100` | `igp:10:200` |

**Path B is preferred**, despite having a higher Cost Community value.

The IGP metric to NEXT_HOP is compared first, so the Cost Community comparison is never reached.

### Configuring `igp`

Configure an inbound route map:

```text
R1(config)# route-map SET-COST permit 10
R1(config-route-map)# set extcommunity cost igp 10 100

R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 route-map SET-COST in
```

This assigns an IGP Cost Community with ID 10 and cost 100 to accepted routes from `10.0.12.2`.

The following command is equivalent:

```text
R1(config-route-map)# set extcommunity cost 10 100
```

---

## Cost Community Propagation

Cost Communities are **nontransitive extended communities**.

They are intended for use within an AS or BGP confederation.

| BGP Relationship | Propagation |
|---|---|
| IBGP | Supported |
| Confederation EBGP | Supported |
| Ordinary EBGP | Not propagated |

To propagate Cost Communities to IBGP neighbors, extended community exchange must be enabled.

For example:

```text
R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 send-community extended
```

Alternatively:

```text
R1(config-router-af)# neighbor 10.0.12.2 send-community both
```

The `both` keyword enables standard and extended community advertisement.

**Important:** `send-community extended` does not override the nontransitive behavior of Cost Communities. Ordinary EBGP advertisement does not propagate these communities outside the AS or confederation.

---

## Cost Communities in EIGRP MPLS L3VPNs

One important application of BGP Cost Communities is **EIGRP PE-CE routing over MPLS L3VPNs**.

Consider two customer sites connected through an MPLS VPN and a separate backdoor link:

```text
             MPLS Provider Network
        +-----------------------------+
        |                             |
       PE1 ------------------------- PE2
        |                             |
        |                             |
       CE1 ------------------------- CE2
               EIGRP Backdoor

                Customer AS
```

The customer network has two possible paths between sites:

1. Through the MPLS VPN.
2. Through the EIGRP backdoor link.

Without special handling, BGP route selection might not preserve the route preference intended by EIGRP.

### Automatic `pre-bestpath` Cost Community

Cisco introduced the `pre-bestpath` POI to support EIGRP MPLS VPN environments containing VPN paths and backdoor links.

In supported PE-CE EIGRP deployments, Cisco automatically generates Cost Communities when EIGRP routes are redistributed into BGP.

These communities encode information associated with:

- EIGRP route type
- EIGRP metric

Because the `pre-bestpath` POI is evaluated before Weight and LOCAL_PREF, the route information can influence BGP best-path selection before the normal preference criteria.

**Manual `set extcommunity cost pre-bestpath` configuration is not required for this automatic EIGRP functionality.**

See [PE-CE Routing with EIGRP](../../../..//../transport-technologies-and-solutions/mpls/l3vpn/pe-ce/eigrp/overview.md) for related MPLS L3VPN behavior.

---

## Cost Communities and Aggregate Routes

Cisco IOS XE supports Cost Communities on BGP aggregate routes.

When multiple component routes carry the same Cost Community ID, the aggregate normally inherits the **highest cost value** for that ID.

For example:

| Component Route | Community ID | Cost |
|---|---|---|
| 192.0.2.0/25 | 10 | 100 |
| 192.0.2.128/25 | 10 | 200 |

The aggregate uses:

```text
Cost:igp:10:200
```

This preserves the less-preferred cost among the contributing routes.

If some component routes lack a Cost Community or use different community IDs, Cisco may assign the default cost value of 2147483647 to the corresponding ID in the aggregate.

Cost Communities are also supported with BGP multipath, subject to the applicable aggregation and multipath rules.

---

## Disabling Cost Community Comparison

Use the following command to prevent BGP from evaluating Cost Communities during best path selection:

```text
R1(config)# router bgp 65001
R1(config-router)# bgp bestpath cost-community ignore
```

The command can also be configured under the appropriate address family on supported IOS XE releases:

```text
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# bgp bestpath cost-community ignore
```

When configured:

- Cost Communities may still be present on BGP routes.
- BGP ignores their cost values when selecting the best path.
- Normal best path selection continues using Weight, LOCAL_PREF, and subsequent criteria.

Restore Cost Community comparison:

```text
R1(config-router-af)# no bgp bestpath cost-community ignore
```

This is useful for troubleshooting unexpected path selection caused by Cost Communities.

---

## Verification

### Display BGP Cost Communities

Use the detailed BGP prefix command:

```text
R1# show bgp ipv4 unicast 192.0.2.0/24
```

Example output:

```text
BGP routing table entry for 192.0.2.0/24
Paths: (2 available, best #1)

  65002
    10.0.12.2 from 10.0.12.2
      Origin IGP, localpref 100, valid, external, best
      Extended Community: Cost:igp:10:100

  65003
    10.0.13.3 from 10.0.13.3
      Origin IGP, localpref 100, valid, external
      Extended Community: Cost:igp:10:200
```

The extended community field identifies:

- POI: `igp`
- Community ID: 10
- Cost: 100 or 200

Path selection still depends on all earlier BGP best-path criteria.

### Verify Route-Map Configuration

```text
R1# show route-map SET-COST
```

Example:

```text
route-map SET-COST, permit, sequence 10
  Match clauses:
  Set clauses:
    extended community Cost:igp:10:100
```

### Verify BGP Neighbor Configuration

```text
R1# show running-config | section router bgp
```

Confirm:

- The appropriate inbound or outbound route map is applied.
- `send-community extended` or `send-community both` is configured when propagation is required.
- `bgp bestpath cost-community ignore` is not unintentionally configured.

### Troubleshooting Checklist

If Cost Communities are not influencing path selection:

1. Verify that the routes actually carry the Cost Communities.
2. Verify the configured POI.
3. Check community IDs and cost values.
4. Confirm that earlier best-path criteria are equal when using the `igp` POI.
5. Verify extended community advertisement between IBGP peers.
6. Check whether Cost Community comparison is disabled.
7. Confirm that the feature is supported on all participating routers.

---

## Cost Community Summary

| Feature | `pre-bestpath` | `igp` |
|---|---|---|
| Evaluation position | Before Weight | After IGP metric to NEXT_HOP |
| Can override Weight? | Yes | No |
| Can override LOCAL_PREF? | Yes | No |
| Can override AS_PATH length? | Yes | No |
| Can override IGP metric to NEXT_HOP? | Yes | No |
| Default POI? | No | Yes |
| Typical application | EIGRP MPLS L3VPN | Custom path preference among otherwise equivalent paths |

**Key points:**

- Lower Cost Community values are preferred at the applicable POI.
- Community IDs determine the order of cost comparisons.
- The default comparison cost for a missing Cost Community is 2147483647.
- `pre-bestpath` is evaluated before normal BGP path preference criteria.
- `igp` is evaluated after the IGP metric to NEXT_HOP.
- Cost Communities are nontransitive extended communities intended for IBGP and confederation peers.

---

## References

- [Cisco: BGP Cost Community](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/ip-routing/b-ip-routing/m_irg-cost-comm-0.html)
- [Cisco: Select BGP Best-path Algorithm](https://www.cisco.com/c/en/us/support/docs/ip/border-gateway-protocol-bgp/13753-25.html)
- [Cisco: BGP Command Reference - set extcommunity cost](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_bgp/command/irg-cr-book/bgp-n1.html)
