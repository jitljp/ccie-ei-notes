# BGP Best Path Selection Configuration

Cisco IOS XE supports several commands that modify the default **BGP Best Path Selection Algorithm**.

These commands can change:

- Which path attributes are evaluated.
- How MED values are compared.
- Whether IGP costs influence selection.
- How BGP handles the final tie-breakers.

Unlike route maps that modify individual path attributes, these commands generally modify the **selection algorithm itself**.

See [BGP Best Path Selection Steps](selection-steps.md) for the complete 15-step algorithm.

---

## Best Path Configuration Commands

| Command | Effect |
|---|---|
| `bgp bestpath as-path ignore` | Ignores AS_PATH length |
| `bgp always-compare-med` | Compares MED across different neighboring ASs |
| `bgp deterministic-med` | Makes MED selection deterministic |
| `bgp bestpath med missing-as-worst` | Treats missing MED as the worst value |
| `bgp bestpath med confed` | Enables MED comparison between applicable confederation paths |
| `bgp bestpath aigp ignore` | Changes AIGP comparison when only one path has AIGP |
| `bgp bestpath igp-metric ignore` | Ignores IGP metric to NEXT_HOP |
| `bgp bestpath cost-community ignore` | Ignores Cost Communities |
| `bgp bestpath compare-routerid` | Skips oldest-EBGP-path preference and compares router IDs |

Most commands are configured under `router bgp`. Some also support address-family configuration, depending on the command.

---

## Ignoring AS_PATH Length

By default, BGP prefers the path with the **shortest AS_PATH** after comparing earlier criteria.

Disable the AS_PATH length comparison:

```text
R1(config)# router bgp 65001
R1(config-router)# bgp bestpath as-path ignore
```

### Example

R1 receives two paths:

| Attribute | Path A | Path B |
|---|---|---|
| AS_PATH | 65002 65003 65004 | 65005 |
| AS_PATH length | 3 | 1 |
| ORIGIN | IGP | Incomplete |
| MED | 100 | 50 |

Assume all earlier criteria are equal.

**Default behavior:**

Path B wins because its AS_PATH is shorter.

**With `bgp bestpath as-path ignore`:**

BGP skips AS_PATH length and compares ORIGIN.

Path A wins because ORIGIN IGP is preferred over Incomplete.

### Important

- This command ignores AS_PATH **length during best-path selection**.
- It does not disable AS_PATH loop prevention.
- The AS_PATH attribute is still maintained and advertised normally.

Restore default behavior:

```text
R1(config-router)# no bgp bestpath as-path ignore
```

---

## MED Comparison Configuration

Cisco IOS XE supports several commands that modify how BGP evaluates the **Multi-Exit Discriminator (MED)**.

These commands have different purposes and should not be confused.

### `bgp always-compare-med`

By default, BGP compares MED only between paths from the same neighboring AS.

For example:

| Attribute | Path A | Path B |
|---|---|---|
| Neighboring AS | 65002 | 65003 |
| MED | 100 | 50 |
| IGP metric to NEXT_HOP | 10 | 30 |

Assume earlier selection criteria are equal.

By default, BGP does not compare their MED values because the paths originate from different neighboring ASs.

If both paths are EBGP and remain otherwise equal, Path A wins because of its lower IGP metric to NEXT_HOP.

Enable MED comparison across different neighboring ASs:

```text
R1(config)# router bgp 65001
R1(config-router)# bgp always-compare-med
```

Now Path B wins because its MED is lower.

**Important:** This command should be configured consistently throughout the AS. Inconsistent MED comparison behavior can cause routing loops.

### `bgp deterministic-med`

By default, when BGP receives multiple paths from different neighboring ASs, the result of MED comparisons can depend on the order in which paths are evaluated.

This is because MED is not normally compared between different neighboring ASs.

Enable deterministic MED comparison:

```text
R1(config-router)# bgp deterministic-med
```

This causes BGP to group paths by neighboring AS and compare MED within each group before performing subsequent comparisons.

For example:

| Path | Neighboring AS | MED |
|---|---|---|
| A | 65002 | 100 |
| B | 65003 | 50 |
| C | 65002 | 200 |

With deterministic MED enabled:

1. BGP compares A and C because both originate from AS 65002.
2. Path A wins within AS 65002 because it has a lower MED.
3. The best path from AS 65002 is then compared with Path B using the remaining applicable criteria.

**`bgp deterministic-med` does not enable MED comparison across different neighboring ASs.**

That requires `bgp always-compare-med`.

### `bgp bestpath med missing-as-worst`

By default, Cisco BGP treats a path without a MED attribute as having MED 0.

Therefore, a missing MED is normally preferred over a higher MED.

For example:

| Path | MED |
|---|---|
| A | Missing |
| B | 100 |

By default, Path A wins if MED is the deciding criterion.

Change this behavior:

```text
R1(config-router)# bgp bestpath med missing-as-worst
```

Now BGP treats a missing MED as the worst value.

Path B is preferred.

On IOS XE, the effective worst value is implementation-dependent, with 4294967294 used in Cisco's documented behavior and 4294967295 used in versions incorporating the relevant fix.

The key point is that **missing MED becomes less preferred than an ordinary configured MED**.

This command does not change the rules determining which paths are eligible for MED comparison.

### `bgp bestpath med confed`

By default, Cisco BGP does not compare MED between certain paths originated within different sub-ASs of a BGP confederation.

Enable MED comparison for applicable confederation paths:

```text
R1(config-router)# bgp bestpath med confed
```

This allows MED comparison for paths containing only confederation AS segments.

For example:

| Path | AS_PATH | MED |
|---|---|---|
| A | `(65002)` | 100 |
| B | `(65003)` | 50 |

Both paths were originated within the confederation.

With `bgp bestpath med confed`, Path B is preferred because its MED is lower, assuming earlier criteria are equal.

However, if paths contain external AS_SEQUENCE segments, the normal MED comparison rules still apply.

**Note:** `bgp always-compare-med` and `bgp bestpath med confed` have different purposes.

---

## AIGP Best Path Configuration

By default, AIGP can influence best-path selection before AS_PATH length.

When both paths contain AIGP, BGP compares their accumulated metrics.

When only one path contains AIGP, Cisco BGP normally prefers the path carrying AIGP, assuming earlier criteria are equal.

Configure:

```text
R1(config)# router bgp 65001
R1(config-router)# bgp bestpath aigp ignore
```

This changes the comparison when only one path carries AIGP.

| Paths Being Compared | With `bgp bestpath aigp ignore` |
|---|---|
| Both have AIGP | Compare AIGP metrics |
| Only one has AIGP | Skip AIGP comparison |
| Neither has AIGP | Continue normal selection |

**This command does not completely disable AIGP metric comparison.**

When both paths contain AIGP, BGP can still select the path with the lowest accumulated metric.

See [Accumulated IGP (AIGP)](../attributes/aigp.md).

---

## Ignoring IGP Metric to NEXT_HOP

By default, BGP prefers the path with the **lowest IGP metric to its NEXT_HOP**, assuming earlier selection criteria are equal.

This normally favors the closest exit point from the AS.

For example:

| Attribute | Path A | Path B |
|---|---|---|
| IGP metric to NEXT_HOP | 10 | 30 |
| Cost Community (`igp`) | 200 | 100 |

Assume earlier criteria are equal.

By default, Path A wins because its IGP metric to NEXT_HOP is lower.

### Configuration

Disable the IGP metric comparison:

```text
R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# bgp bestpath igp-metric ignore
```

BGP now skips the normal IGP next-hop metric comparison.

If the Cost Communities are applicable and otherwise comparable, Path B can win because it has the lower `igp` Cost Community.

### Important

- The command skips the IGP metric comparison during best-path selection.
- BGP still requires a reachable NEXT_HOP for path eligibility.
- Recursive next-hop resolution is still performed.
- The command is also associated with BGP Diverse Path designs.

Restore default behavior:

```text
R1(config-router-af)# no bgp bestpath igp-metric ignore
```

---

## Ignoring Cost Communities

By default, BGP can evaluate Cost Communities at their configured points of insertion.

| POI | Selection Position |
|---|---|
| `pre-bestpath` | Before Weight |
| `igp` | After IGP metric to NEXT_HOP |

Disable Cost Community comparison:

```text
R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# bgp bestpath cost-community ignore
```

This causes BGP to ignore Cost Communities during best-path selection.

However:

- The communities can remain attached to BGP paths.
- They can still be displayed in BGP output.
- Normal selection criteria such as Weight and LOCAL_PREF continue to apply.

Restore Cost Community comparison:

```text
R1(config-router-af)# no bgp bestpath cost-community ignore
```

See [BGP Cost Communities and Points of Insertion](cost-community.md).

---

## Comparing Router IDs Instead of Path Age

Near the end of the BGP best-path algorithm, Cisco normally prefers the **oldest EBGP path** when the earlier criteria are equal.

This reduces unnecessary path changes when new equivalent routes become available.

For example:

| Attribute | Path A | Path B |
|---|---|---|
| Path age | 2 hours | 10 minutes |
| BGP router ID | 3.3.3.3 | 2.2.2.2 |

Assume both paths are EBGP and all earlier criteria are equal.

By default, Path A is preferred because it is older.

### Configuration

Configure BGP to compare router IDs rather than preferring the oldest EBGP path:

```text
R1(config)# router bgp 65001
R1(config-router)# bgp bestpath compare-routerid
```

Now Path B is preferred because it has the lower BGP router ID.

### Important

- This command skips the normal oldest-EBGP-path preference.
- It does not move router ID comparison ahead of Weight, LOCAL_PREF, AS_PATH length, or other earlier criteria.
- When ORIGINATOR_ID is present, it is used for the router ID comparison.

Restore default behavior:

```text
R1(config-router)# no bgp bestpath compare-routerid
```

---

## Best Path Selection vs. Attribute Manipulation

The commands covered so far modify the **algorithm**.

Another way to influence BGP best-path selection is to modify the attributes being compared.

For example:

| Attribute | Configuration Command |
|---|---|
| Weight | `set weight` |
| LOCAL_PREF | `set local-preference` |
| AS_PATH | `set as-path prepend` |
| ORIGIN | `set origin` |
| MED | `set metric` |
| AIGP | `set aigp-metric` |
| Cost Community | `set extcommunity cost` |

These commands are typically configured in route maps.

### Example: Prefer Selected Routes Using LOCAL_PREF

Suppose R1 receives `192.0.2.0/24` from two EBGP neighbors.

R1 should prefer the path received from `10.0.12.2`.

```text
R1(config)# ip prefix-list PREFER-R2 permit 192.0.2.0/24

R1(config)# route-map SET-LOCAL-PREF permit 10
R1(config-route-map)# match ip address prefix-list PREFER-R2
R1(config-route-map)# set local-preference 200

R1(config)# route-map SET-LOCAL-PREF permit 20

R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 route-map SET-LOCAL-PREF in
```

The route map:

1. Matches `192.0.2.0/24`.
2. Assigns LOCAL_PREF 200.
3. Permits other routes without changing their LOCAL_PREF.

R1 prefers this path over another eligible path with LOCAL_PREF 100, assuming their Weight and applicable `pre-bestpath` Cost Community preferences do not override it.

**This modifies a path attribute, not the best-path algorithm.**

See [BGP Attribute Manipulation](../../routing-policies/attribute-manipulation.md).

---

## Best Path Selection vs. Multipath Configuration

Some BGP commands influence multipath eligibility without directly changing the selection of the single best path.

For example:

```text
R1(config-router)# bgp bestpath as-path multipath-relax
```

This relaxes AS_PATH equality requirements when evaluating paths for multipath installation.

It is different from:

```text
R1(config-router)# bgp bestpath as-path ignore
```

| Command | Purpose |
|---|---|
| `bgp bestpath as-path ignore` | Ignores AS_PATH length during normal best-path selection |
| `bgp bestpath as-path multipath-relax` | Relaxes AS_PATH requirements for multipath eligibility |

Configure the maximum number of EBGP paths:

```text
R1(config-router-af)# maximum-paths 4
```

Configure IBGP multipath:

```text
R1(config-router-af)# maximum-paths ibgp 4
```

See [BGP Load Balancing](../load-balancing.md) for multipath requirements and configuration.

---

## Verification

### Display BGP Best Paths

```text
R1# show bgp ipv4 unicast
R1# show bgp ipv4 unicast 192.0.2.0/24
```

The detailed prefix command provides information about:

- Available paths.
- Best-path status.
- Weight and LOCAL_PREF.
- AS_PATH, ORIGIN, and MED.
- NEXT_HOP and related metrics.
- AIGP and Cost Communities, when present.

### Verify Best-Path Configuration

```text
R1# show running-config | section router bgp
```

Look for commands such as:

```text
bgp bestpath as-path ignore
bgp always-compare-med
bgp deterministic-med
bgp bestpath med missing-as-worst
bgp bestpath med confed
bgp bestpath aigp ignore
bgp bestpath compare-routerid
```

Also inspect the relevant address-family configuration for:

```text
bgp bestpath igp-metric ignore
bgp bestpath cost-community ignore
```

### Verify Routing Policies

```text
R1# show route-map
R1# show ip prefix-list
```

### Verify Next-Hop Resolution

```text
R1# show ip route 10.10.10.10
R1# show ip cef 10.10.10.10 detail
```

### Apply Updated Inbound Policies

After changing an inbound BGP route map, use an inbound soft refresh when needed:

```text
R1# clear bgp ipv4 unicast 10.0.12.2 soft in
```

This requests updated routing information using Route Refresh when supported, without resetting the BGP TCP session.

Best-path algorithm changes themselves generally trigger reevaluation without requiring an inbound route refresh.

---

## Configuration Summary

| Command | Default Behavior | Configured Behavior |
|---|---|---|
| `bgp bestpath as-path ignore` | Shortest AS_PATH preferred | AS_PATH length ignored |
| `bgp always-compare-med` | MED compared within the same neighboring AS | MED compared across neighboring ASs |
| `bgp deterministic-med` | MED selection can depend on comparison order | Deterministic comparison within neighboring AS groups |
| `bgp bestpath med missing-as-worst` | Missing MED treated as 0 | Missing MED treated as worst |
| `bgp bestpath med confed` | Special confederation paths not compared by MED | MED comparison enabled for applicable confederation paths |
| `bgp bestpath aigp ignore` | Path with AIGP preferred over one without | Mixed AIGP/non-AIGP comparison skipped |
| `bgp bestpath igp-metric ignore` | Lowest next-hop IGP cost preferred | IGP next-hop metric comparison skipped |
| `bgp bestpath cost-community ignore` | Applicable Cost Communities evaluated | Cost Communities ignored |
| `bgp bestpath compare-routerid` | Oldest EBGP path preferred when applicable | Router ID comparison used instead |

**Important:** These commands change specific parts of the best-path process. They do not bypass path eligibility checks, such as NEXT_HOP reachability.

---

## References

- [Cisco: Select BGP Best-path Algorithm](https://www.cisco.com/c/en/us/support/docs/ip/border-gateway-protocol-bgp/13753-25.html)
- [Cisco: BGP Command Reference - A through B](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_bgp/command/irg-cr-book/bgp-a1.html)
- [Cisco: Configuring Internal BGP Features](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-routing/b-ip-routing/m_irg-int-features-0.html)
- [Cisco: BGP Cost Community](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-routing/b-ip-routing/m_irg-cost-comm-0.html)
- [Cisco: Configure the AIGP Metric Attribute for BGP](https://www.cisco.com/c/en/us/support/docs/ip/border-gateway-protocol-bgp/119001-configure-aigp-00.html)
