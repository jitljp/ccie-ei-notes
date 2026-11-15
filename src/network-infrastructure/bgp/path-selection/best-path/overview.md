# 1.5.b (ii) Best Path Selection Algorithm

The **BGP Best Path Selection Algorithm** determines which path is preferred when a BGP router has multiple paths to the same destination prefix.

Unlike OSPF and EIGRP, BGP does not calculate a single cumulative metric. Instead, it compares path attributes and other selection criteria in a specific order.

**The first applicable criterion that distinguishes two eligible paths determines which path is preferred.**

By default, BGP selects one best path per prefix in a BGP address family.

---

## Best Path Selection Process

The BGP path selection process consists of three main stages:

1. **Path Eligibility:** Determine which paths are valid candidates.
2. **Best Path Selection:** Compare eligible paths using the ordered selection criteria.
3. **RIB Installation:** Attempt to install the selected best path into the IP routing table.

```text
           Multiple BGP Paths
                    |
                    v
           +------------------+
           | Path Eligibility |
           +------------------+
                    |
                    v
           +------------------+
           | Best Path        |
           | Selection        |
           +------------------+
                    |
                    v
           +------------------+
           | BGP Best Path    |
           +------------------+
                    |
                    v
           +------------------+
           | RIB Installation |
           +------------------+
                    |
                    v
           IP Routing Table
          (if successfully installed)
```

A path can be selected as the BGP best path without being installed in the IP routing table.

---

## Path Eligibility

Before comparing path attributes, BGP determines whether each path is eligible for best path selection.

Examples of paths that are not eligible include:

- Paths with an **unreachable NEXT_HOP**.
- Paths rejected by inbound routing policies.
- Paths rejected by AS_PATH loop prevention.

Paths rejected by inbound policy or AS_PATH loop prevention are normally excluded before best path selection.

For example, consider two paths to `192.0.2.0/24`:

| Attribute | Path A | Path B |
|---|---|---|
| NEXT_HOP | 10.0.12.2 | 10.0.13.3 |
| NEXT_HOP Reachable | No | Yes |
| LOCAL_PREF | 200 | 100 |
| AS_PATH | 65002 | 65003 |

Although Path A has a higher LOCAL_PREF, it is not eligible because its NEXT_HOP is unreachable.

**BGP selects Path B**, assuming it satisfies all other eligibility requirements.

See [NEXT_HOP](../attributes/next-hop.md) for details.

---

## Best Path Selection Order

Cisco IOS XE uses the following selection criteria when comparing eligible BGP paths. Cost Communities and AIGP add comparisons that apply only when the relevant features and attributes are present.

| Order | Criterion | Preferred Path |
|---|---|---|
| 1 | Cost Community (`pre-bestpath`), when applicable | Lowest cost at the first distinguishing community ID |
| 2 | Weight | Highest |
| 3 | LOCAL_PREF | Highest |
| 4 | Local origination | Locally originated |
| 5 | AIGP, when applicable | Lowest accumulated cost |
| 6 | AS_PATH length | Shortest |
| 7 | ORIGIN | IGP > EGP > Incomplete |
| 8 | MED | Lowest, when compared |
| 9 | EBGP vs. IBGP | EBGP |
| 10 | IGP metric to NEXT_HOP | Lowest |
| 11 | Cost Community (`igp`), when applicable | Lowest cost at the first distinguishing community ID |
| 12 | Path age | Oldest EBGP path, when applicable |
| 13 | BGP router ID | Lowest |
| 14 | CLUSTER_LIST length | Shortest |
| 15 | Neighbor IP address | Lowest |

**Important considerations:**

- **Weight** is Cisco-specific and is not transmitted in BGP UPDATE messages.
- **LOCAL_PREF** defaults to 100 on Cisco IOS XE unless modified.
- **Local origination** prefers locally originated paths over learned paths. Local routes originated using `network` or redistribution are preferred over locally generated aggregates.
- **AIGP** is evaluated after local origination and before AS_PATH length when applicable. It can favor a longer AS_PATH.
- **AS_PATH length** normally counts AS_SEQUENCE entries, with an AS_SET counting as one. Confederation segments are excluded from the length.
- **MED** is compared only between paths from the same neighboring AS by default. Configuration can change this behavior.
- **IGP metric to NEXT_HOP** refers to the cost of reaching the BGP next hop, not the destination prefix.
- **Cost Communities** are compared at their configured point of insertion (POI). The `pre-bestpath` POI precedes Weight; the `igp` POI follows the IGP next-hop metric comparison.
- **Path age** is considered only when the remaining paths are EBGP paths and the relevant conditions are satisfied.
- **Router ID** comparison uses ORIGINATOR_ID instead when the path contains that route reflector attribute.
- **CLUSTER_LIST** is relevant to routes reflected through BGP route reflectors.

**Multipath eligibility is not a preference criterion.** BGP evaluates additional paths for multipath installation separately while still designating one best path.

The complete algorithm and its exceptions are covered in [BGP Best Path Selection Steps](selection-steps.md).

---

## BGP Cost Community

The **BGP Cost Community** is a nontransitive extended community that influences best-path selection at a configurable **Point of Insertion (POI)**.

Cisco IOS XE supports two POIs:

| POI | Selection Position | Preference |
|---|---|---|
| `pre-bestpath` | Before Weight | Lower cost |
| `igp` | After IGP metric to NEXT_HOP | Lower cost |

For a given POI, BGP compares Cost Communities in ascending community-ID order and prefers the lower cost at the first distinguishing ID. Because `pre-bestpath` is evaluated before Weight, it can override differences in Weight and LOCAL_PREF.

See [BGP Cost Communities and Points of Insertion](cost-community.md) for comparison rules, examples, configuration, and verification.

---

## Accumulated IGP (AIGP)

**Accumulated IGP (AIGP)** is an optional nontransitive BGP path attribute used to carry accumulated IGP cost across routing domains under coordinated administration.

When AIGP comparison applies, Cisco IOS XE evaluates it **after local origination and before AS_PATH length**. The comparison accounts for the accumulated AIGP metric and the local IGP cost of reaching the BGP NEXT_HOP. Consequently, a path with a longer AS_PATH can still be preferred if its accumulated cost is lower.

See [Accumulated IGP (AIGP)](../attributes/aigp.md) for metric calculation, propagation, configuration, and verification.

---

## Example: Best Path Selection

Consider R1 receiving two EBGP paths to `192.0.2.0/24`:

```text
R2 (AS 65002) ---- EBGP ---- R1 (AS 65001) ---- EBGP ---- R3 (AS 65003)
```

Assume the paths have no distinguishing Cost Communities or AIGP values:

| Attribute | Via R2 | Via R3 |
|---|---|---|
| Weight | 0 | 0 |
| LOCAL_PREF | 200 | 100 |
| AS_PATH | 65002 65010 65020 | 65003 |
| ORIGIN | IGP | IGP |
| MED | 100 | 10 |
| NEXT_HOP | Reachable | Reachable |

BGP performs the following comparisons:

1. **Weight:** Both paths have Weight 0. Continue.
2. **LOCAL_PREF:** The path through R2 has LOCAL_PREF 200, compared to 100 through R3.
3. **Best Path:** R2 is preferred.

BGP does not need to compare AS_PATH length, ORIGIN, MED, or later criteria.

Despite having a longer AS_PATH and a higher MED, **the path through R2 is selected because of its higher LOCAL_PREF**.

### Changing LOCAL_PREF

Suppose both paths now have LOCAL_PREF 100.

1. Weight is equal.
2. LOCAL_PREF is equal.
3. Neither path is locally originated.
4. No applicable AIGP comparison distinguishes them.
5. R3 has a shorter AS_PATH.

**The path through R3 is now preferred.**

This illustrates why selection order matters when troubleshooting unexpected BGP decisions.

---

## Configurable Best Path Behavior

Cisco IOS XE provides commands that modify the default best path selection process.

| Command | Effect |
|---|---|
| `bgp bestpath as-path ignore` | Skips the AS_PATH length comparison |
| `bgp bestpath as-path multipath-relax` | Relaxes AS_PATH equality requirements for multipath |
| `bgp always-compare-med` | Compares MED across paths from different neighboring ASs |
| `bgp deterministic-med` | Groups paths by neighboring AS for consistent MED comparisons |
| `bgp bestpath med missing-as-worst` | Treats missing MED as the worst value |
| `bgp bestpath med confed` | Enables MED comparison for applicable confederation paths |
| `bgp bestpath compare-routerid` | Skips the usual oldest-EBGP-path preference in favor of router ID comparison |
| `bgp bestpath cost-community ignore` | Ignores Cost Communities during best path selection |
| `bgp bestpath aigp ignore` | Ignores AIGP during best path selection |

Important distinctions:

- `bgp deterministic-med` does not make MED values from different neighboring ASs directly comparable.
- `bgp always-compare-med` enables comparisons across different neighboring ASs.
- `bgp bestpath as-path multipath-relax` affects multipath eligibility; it does not simply disable AS_PATH comparison for the single best path.

See [BGP Best Path Selection Configuration](configuration.md) for examples.

---

## Best Path Selection vs. RIB Installation

The **BGP best path** and the **route installed in the IP routing table** are not necessarily the same.

For example, R1 learns `192.0.2.0/24` through both IBGP and OSPF:

| Protocol | Administrative Distance |
|---|---|
| IBGP | 200 |
| OSPF | 110 |

Assume the IBGP path has a reachable NEXT_HOP and is selected as the BGP best path.

However, the OSPF route is preferred for installation in the IP routing table because of its lower administrative distance.

The result is:

- The IBGP path remains the BGP best path.
- The OSPF route is installed in the IP routing table.
- The BGP path experiences a **RIB failure**.

**Administrative distance is not a criterion in the BGP best path selection algorithm.** It is used when the routing table selects between routes offered by different protocols or sources.

See [BGP Path Selection and RIB Installation](rib-installation.md).

---

## Best Path Selection vs. Multipath

By default, BGP selects one best path per prefix.

However, Cisco IOS XE supports **BGP multipath**, allowing multiple eligible BGP paths to be installed in the routing table.

BGP still identifies a single best path. Additional paths may qualify for multipath installation if they meet the applicable requirements.

```text
                AS 65002

           R2 -------- R3
            |          |
            | EBGP     | EBGP
            |          |
            +---- R1 --+
                AS 65001
```

R1 receives two EBGP paths to the same prefix from R2 and R3 in AS 65002. With the appropriate multipath configuration and eligible paths, R1 may install both paths for load sharing while maintaining one selected BGP best path.

See [BGP Load Balancing](../load-balancing.md).

---

## Verification

Useful commands for investigating BGP best path selection include:

```text
R1# show bgp ipv4 unicast
R1# show bgp ipv4 unicast 192.0.2.0/24
R1# show ip route 192.0.2.0
R1# show ip cef 192.0.2.1
R1# show route-map
R1# show running-config | section router bgp
```

The detailed BGP prefix output can show:

- Available paths and the selected best path.
- Weight, LOCAL_PREF, AS_PATH, ORIGIN, and MED.
- Next-hop reachability and associated IGP cost.
- AIGP metric, when present.
- Extended communities, including Cost Communities, when present.
- Best-path selection information and potential RIB failures.

Common BGP status codes include:

| Code | Meaning |
|---|---|
| `*` | Valid path |
| `>` | Best path |
| `r` | RIB failure |
| `m` | Multipath |

For example:

```text
R1# show bgp ipv4 unicast

   Network          Next Hop            Metric LocPrf Weight Path
*> 192.0.2.0/24     10.0.12.2                0             0 65002 i
*                   10.0.13.3                0             0 65003 i
```

Both paths are valid, but only the path through `10.0.12.2` is selected as best. Use the detailed prefix command to investigate the attributes behind the decision.

---

## Topics Covered

This section covers three aspects of BGP best path selection:

1. [BGP Best Path Selection Steps](selection-steps.md): Detailed selection order, exceptions, and tie-breakers.
2. [BGP Cost Communities and Points of Insertion](cost-community.md): Cost Community comparison and configuration.
3. [BGP Best Path Selection Configuration](configuration.md): Commands that modify the selection algorithm.
4. [BGP Path Selection and RIB Installation](rib-installation.md): Best paths, administrative distance, RIB failures, and route installation.

AIGP is covered separately in [Accumulated IGP (AIGP)](../attributes/aigp.md).

---

## References

- [Cisco: Select BGP Best-path Algorithm](https://www.cisco.com/c/en/us/support/docs/ip/border-gateway-protocol-bgp/13753-25.html)
- [Cisco: BGP Cost Community](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/ip-routing/b-ip-routing/m_irg-cost-comm-0.html)
- [Cisco: BGP Accumulated IGP](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/ip-routing/b-ip-routing/m_irg-bgp-accumulated-igp.html)
- [Cisco: Configure the AIGP Metric Attribute for BGP](https://www.cisco.com/c/en/us/support/docs/ip/border-gateway-protocol-bgp/119001-configure-aigp-00.html)
- [RFC 4271: A Border Gateway Protocol 4 (BGP-4)](https://www.rfc-editor.org/rfc/rfc4271.html)
