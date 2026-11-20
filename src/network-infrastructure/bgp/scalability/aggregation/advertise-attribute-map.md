# Advertise-Map and Attribute-Map

Cisco IOS XE provides two route-map options for controlling **the path attributes of a BGP aggregate**:

- **`advertise-map`** selects which more-specific BGP routes contribute their attributes to the aggregate.
- **`attribute-map`** applies `set` actions directly to the aggregate after it is constructed.

Both options are configured as part of the `aggregate-address` command:

```text
R1(config)# router bgp 100
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# aggregate-address 10.1.0.0 255.255.254.0 as-set summary-only advertise-map SELECT attribute-map SET-ATTR
```

These features are particularly useful with `as-set`, which can preserve AS_PATH information and communities from contributing routes. Without additional controls, an undesirable community inherited from one more-specific route can affect the **entire aggregate**.

**Important:** The `advertise-map` used with `aggregate-address` is **not** the same as the `neighbor advertise-map ... exist-map` / `non-exist-map` feature used for [BGP Conditional Advertisement](../../routing-policies/conditional-advertisement.md).

---

## Aggregation Attribute Processing

Consider the following simplified process:

1. BGP identifies the more-specific routes within the aggregate's address range.
2. If an `advertise-map` is configured, the routes selected by its route map are used to build the aggregate's inherited attributes, such as the AS_PATH and communities.
3. If `as-set` is configured, BGP can retain AS_PATH information from those selected routes, along with applicable inherited attributes.
4. If an `attribute-map` is configured, its `set` actions modify the resulting aggregate's attributes.
5. The aggregate is advertised to eligible neighbors according to normal BGP advertisement rules, communities, and outbound policy.

The **aggregation range**, **attribute contributors**, **suppressed more-specific routes**, and **neighbors receiving an advertisement** are different concepts.

| Mechanism | What It Controls |
|---|---|
| `aggregate-address` | Defines the aggregate prefix |
| `as-set` | Preserves AS_PATH information, and applicable attributes, from contributors |
| `advertise-map` | Selects more-specific routes used to construct inherited aggregate attributes |
| `attribute-map` | Changes attributes on the aggregate itself |
| `summary-only` | Suppresses advertisement of all contributing more-specific routes |
| `suppress-map` | Suppresses advertisement of selected more-specific routes |
| `neighbor ... unsuppress-map` | Readvertises selected suppressed more-specific routes to one neighbor |
| `neighbor ... route-map out` | Applies outbound policy to routes advertised to a particular neighbor |

**Key distinction:** `advertise-map` does not mean "advertise these /24s." That is a common misinterpretation of its name.

---

## Example Topology

The examples use four EBGP routers:

```text
                         AS 100
                           R1
                       Aggregator
                        /  |  \
                       /   |   \
                      /    |    \
                     R3    R4    R2
                   AS 300 AS 400 AS 200
                     |     |      |
                 10.1.0.0/24   Receives
                       10.1.1.0/24 aggregate

                 All connections: EBGP
                 Aggregate: 10.1.0.0/23
```

Assume R1 learns these best paths:

| Prefix | AS_PATH on R1 | Standard Community |
|---|---|---|
| `10.1.0.0/24` | `300` | `NO_EXPORT` |
| `10.1.1.0/24` | `400` | None |

R3 has set the `NO_EXPORT` community on its advertised prefix, and the community is received by R1. For example, R3 can configure:

```text
R3(config)# route-map SET-NO-EXPORT permit 10
R3(config-route-map)# set community no-export

R3(config)# router bgp 300
R3(config-router)# neighbor 13.1.1.1 remote-as 100
R3(config-router)# neighbor 13.1.1.1 send-community
R3(config-router)# neighbor 13.1.1.1 route-map SET-NO-EXPORT out
```

Assume both /24 routes have already been originated into BGP and that R1 has established its EBGP sessions. R1's address on its R3-facing link is `13.1.1.1`.

### Baseline: `as-set` Without an Advertise Map or Attribute Map

On R1:

```text
R1(config)# router bgp 100
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# aggregate-address 10.1.0.0 255.255.254.0 as-set summary-only
```

The aggregate's AS_PATH incorporates both contributing ASNs:

```text
AS_PATH on R1:     {300,400}
```

The aggregate also inherits `NO_EXPORT` from `10.1.0.0/24`.

The community prevents R1 from advertising the aggregate to its ordinary EBGP neighbor R2:

```text
Aggregate:     10.1.0.0/23
AS_PATH:       {300,400}
Community:     NO_EXPORT
Advertised to R2: No
```

Even though only **one** contributing route has `NO_EXPORT`, the community affects the aggregate as a whole.

There are two main ways to address this, each with different effects:

1. Use `advertise-map` to **exclude that route's attributes** from the aggregate.
2. Use `attribute-map` to **remove or replace the community on the aggregate**.

---

## Using `advertise-map`

The `advertise-map` keyword specifies a route map identifying which more-specific routes should be considered when constructing the aggregate's attributes.

Syntax:

```text
aggregate-address <network> <mask> as-set advertise-map <route-map-name>
```

It is especially useful when only certain contributors should influence:

- The aggregate's **AS_PATH**, including AS_SEQUENCE and AS_SET construction.
- The aggregate's **communities**.
- Other inherited aggregate attributes, where supported by the platform.

It does **not** directly suppress or unsuppress the contributing prefixes.

### Example: Exclude the Route Carrying `NO_EXPORT`

Suppose R1 should inherit attributes **only** from `10.1.1.0/24`, learned from AS 400.

First, configure a prefix list:

```text
R1(config)# ip prefix-list SELECT-AS400 seq 5 permit 10.1.1.0/24
```

Then create a route map:

```text
R1(config)# route-map SELECT-AS400 permit 10
R1(config-route-map)# match ip address prefix-list SELECT-AS400
```

Apply the route map to the aggregate:

```text
R1(config)# router bgp 100
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# aggregate-address 10.1.0.0 255.255.254.0 as-set summary-only advertise-map SELECT-AS400
```

The intended attribute contributors are now:

| Route | Selected by `advertise-map`? | AS_PATH Used? | Community Inherited? |
|---|---|---|---|
| `10.1.0.0/24`, AS 300 | No | No | No (`NO_EXPORT` excluded) |
| `10.1.1.0/24`, AS 400 | Yes | Yes | None |

Consequently, the aggregate's AS_PATH on R1 is:

```text
400
```

No AS_SET is needed because the selected contributor has only the single AS_SEQUENCE `400`.

When R1 advertises the aggregate to R2 over EBGP, it prepends its own ASN:

```text
AS_PATH on R2: 100 400
```

The `NO_EXPORT` community is not inherited, so it does not block the aggregate's advertisement to R2. Other BGP policies can still prevent advertisement.

### What Happens to the Excluded Prefix?

The `advertise-map` has **not** removed `10.1.0.0/24` from R1's BGP table.

It has excluded that route from **attribute construction**.

Because the example also uses `summary-only`, both /24 routes remain suppressed from normal BGP advertisements. The aggregate still covers both address ranges:

```text
10.1.0.0/23
  10.1.0.0/24    Suppressed by summary-only
  10.1.1.0/24    Suppressed by summary-only
```

If `summary-only` were removed, the eligible more-specific routes could also be advertised. Changing the `advertise-map` alone is not a substitute for `summary-only` or `suppress-map`.

### Important Loop-Prevention Implication

Excluding AS 300 from the aggregate's AS_PATH changes loop detection.

Compare the AS_PATH R2 receives:

| Configuration | AS_PATH on R2 |
|---|---|
| `as-set` without `advertise-map` | `100 {300,400}` |
| `as-set advertise-map SELECT-AS400` | `100 400` |

A router in AS 300 would reject the first aggregate under normal AS_PATH loop prevention because its own ASN appears in the AS_SET.

The second aggregate does **not** contain AS 300, so that particular loop check would not reject it.

**Warning:** Excluding an ASN from the AS_PATH is not merely cosmetic. Verify the intended routing design before advertising the resulting aggregate back into a contributing AS.

### Route-Map Permit and Deny Behavior

A route-map **permit** sequence selects matching routes as attribute contributors.

A route-map **deny** sequence excludes matching routes. Routes that do not match a permitting sequence are not selected.

For example, to exclude the AS 300 prefix but select the AS 400 prefix:

```text
R1(config)# ip prefix-list INCLUDE-400 seq 5 permit 10.1.1.0/24

R1(config)# route-map SELECT permit 20
R1(config-route-map)# match ip address prefix-list INCLUDE-400
```

The single permitting sequence is sufficient; a separate deny sequence is not required.

**Important:** A prefix-list `deny` is not the same thing as a route-map `deny`. A `match ip address prefix-list` clause matches prefixes **permitted** by the prefix list.

### What If the Advertise Map Matches No Routes?

An aggregate still requires suitable more-specific routes in BGP. The availability of routes matching the `advertise-map` can also affect conditional aggregate generation on Cisco IOS XE.

Do not assume that an `advertise-map` referencing a nonexistent or unmatched prefix will produce the intended aggregate. Confirm the behavior on the software version in use:

```text
R1# show ip bgp 10.1.0.0/23
R1# show ip bgp
R1# show route-map SELECT-AS400
R1# show ip prefix-list SELECT-AS400
```

The normal BGP conditional advertisement feature, using `neighbor ... advertise-map ... exist-map` or `non-exist-map`, is separate from this aggregate-specific feature.

---

## Using `attribute-map`

The `attribute-map` keyword references a route map containing **`set` actions to apply to the aggregate itself**.

Syntax:

```text
aggregate-address <network> <mask> attribute-map <route-map-name>
```

Unlike `advertise-map`, an `attribute-map` does not select which /24 routes contribute their AS_PATH information.

Instead, it changes the attributes of the resulting aggregate.

### Example: Remove an Inherited `NO_EXPORT` Community

Start with the original contributing routes:

```text
10.1.0.0/24    AS_PATH: 300    Community: NO_EXPORT
10.1.1.0/24    AS_PATH: 400    Community: none
```

Configure a route map on R1:

```text
R1(config)# route-map CLEAR-AGG-COMM permit 10
R1(config-route-map)# set community none
```

Then apply it to the aggregate:

```text
R1(config)# router bgp 100
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# aggregate-address 10.1.0.0 255.255.254.0 as-set summary-only attribute-map CLEAR-AGG-COMM
```

The expected result is:

```text
Aggregate:         10.1.0.0/23
AS_PATH on R1:     {300,400}
Community:         none
AS_PATH on R2:     100 {300,400}
```

**The AS_PATH information from both contributors is still preserved.**

The `attribute-map` has removed the inherited standard communities rather than excluding the AS 300 route from the AS_PATH construction.

The aggregate is now eligible for EBGP advertisement to R2, assuming it passes other advertisement checks.

This is a major difference from the preceding `advertise-map` example:

| Method | AS_PATH on R2 | Inherited `NO_EXPORT`? |
|---|---|---|
| `as-set` alone | `100 {300,400}` | Yes |
| `as-set advertise-map SELECT-AS400` | `100 400` | No |
| `as-set attribute-map CLEAR-AGG-COMM` | `100 {300,400}` | No |

### Setting a Standard Community on the Aggregate

The `attribute-map` can assign a new community to the aggregate:

```text
R1(config)# route-map SET-AGG-COMM permit 10
R1(config-route-map)# set community 100:50

R1(config)# router bgp 100
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# aggregate-address 10.1.0.0 255.255.254.0 as-set summary-only attribute-map SET-AGG-COMM
```

This **replaces** the aggregate's existing standard communities with `100:50`.

To send the standard community to R2:

```text
R1(config-router-af)# neighbor 12.1.1.2 send-community
```

The neighbor must already be configured for BGP peering.

#### `set community` Versus `set community additive`

These commands are different:

```text
set community 100:50
```

Replaces the existing standard communities.

```text
set community 100:50 additive
```

Adds `100:50` to the existing standard communities.

If the aggregate inherits `NO_EXPORT`, using `additive` **does not remove `NO_EXPORT`**. The aggregate will still be ineligible for ordinary EBGP advertisement.

Use `set community none` when the goal is to remove all inherited standard communities.

### Changing ORIGIN and MED

An attribute map can also manipulate other supported BGP attributes using route-map `set` commands.

For example:

```text
R1(config)# route-map SET-AGG-ATTR permit 10
R1(config-route-map)# set origin incomplete
R1(config-route-map)# set metric 50

R1(config)# router bgp 100
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# aggregate-address 10.1.0.0 255.255.254.0 summary-only attribute-map SET-AGG-ATTR
```

The route map requests:

- **ORIGIN:** `incomplete` (`?` in BGP table output).
- **MED:** `50`.

Because this example does **not** use `as-set`, it does not preserve the contributing routes' AS_PATH information. It is still possible to set attributes explicitly on the aggregate.

Verify the resulting attributes rather than assuming that the aggregate has simply copied one contributing route's attributes.

### LOCAL_PREF and WEIGHT Considerations

Other route-map `set` actions may be useful, but distinguish **local BGP attributes** from attributes transmitted to external neighbors:

- **LOCAL_PREF** is propagated within an AS through IBGP, not to ordinary EBGP neighbors.
- **WEIGHT** is Cisco-local and is not a BGP path attribute sent to neighbors.
- **MED**, **ORIGIN**, and **communities** have different propagation rules.

An attribute shown on R1 is not necessarily an attribute that R2 will receive.

---

## Combining `advertise-map` and `attribute-map`

The two options can be used together.

For example, suppose the goal is to:

1. Use only the AS 400 route to construct the aggregate's AS_PATH.
2. Set a new standard community `100:50` on the aggregate.
3. Suppress advertisement of the more-specific /24 routes.

Configure:

```text
R1(config)# ip prefix-list SELECT-AS400 seq 5 permit 10.1.1.0/24

R1(config)# route-map SELECT-AS400 permit 10
R1(config-route-map)# match ip address prefix-list SELECT-AS400

R1(config)# route-map SET-AGG-COMM permit 10
R1(config-route-map)# set community 100:50

R1(config)# router bgp 100
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# aggregate-address 10.1.0.0 255.255.254.0 as-set summary-only advertise-map SELECT-AS400 attribute-map SET-AGG-COMM
R1(config-router-af)# neighbor 12.1.1.2 send-community
```

The expected aggregate is:

```text
On R1:
  Prefix:       10.1.0.0/23
  AS_PATH:      400
  Community:    100:50

On R2 (AS 200):
  Prefix:       10.1.0.0/23
  AS_PATH:      100 400
  Community:    100:50
```

The attribute-selection logic is:

```text
10.1.0.0/24  (AS 300, NO_EXPORT) ----- excluded by advertise-map

10.1.1.0/24  (AS 400) --------------- selected by advertise-map
                                               |
                                               v
                                      Aggregate AS_PATH: 400
                                               |
                                               v
                                    attribute-map sets 100:50
                                               |
                                               v
                                     Advertise 10.1.0.0/23
```

The `attribute-map` takes precedence for the attributes that it explicitly changes. It does not automatically replace the AS_PATH that the `advertise-map` selected.

---

## Interactions with Suppression

Attribute construction and prefix suppression are independent decisions.

Consider:

```text
R1(config-router-af)# aggregate-address 10.1.0.0 255.255.254.0 as-set suppress-map SUP advertise-map SELECT-AS400 attribute-map SET-AGG-COMM
```

Each component has a separate function:

1. `as-set` enables the aggregate to preserve appropriate contributing AS_PATH information.
2. `suppress-map SUP` determines which more-specific routes are suppressed.
3. `advertise-map SELECT-AS400` selects routes used to construct the aggregate's inherited attributes.
4. `attribute-map SET-AGG-COMM` changes the aggregate's standard communities.

**A route suppressed by `suppress-map` can still contribute attributes to an aggregate** if it is selected by the `advertise-map`.

Conversely, a route excluded by `advertise-map` can remain eligible for advertisement as a more-specific route if it is not suppressed.

If `summary-only` and `suppress-map` appear together, Cisco documents that `suppress-map` takes precedence; `summary-only` does not additionally suppress the unmatched routes.

See [Summary-Only, Suppress-Map, and Unsuppress-Map](suppression.md) for detailed suppression behavior.

---

## Aggregate `advertise-map` Versus Neighbor Conditional Advertisement

BGP uses the same keyword in two unrelated configurations.

### Aggregate Attribute Selection

```text
R1(config-router-af)# aggregate-address 10.1.0.0 255.255.254.0 as-set advertise-map SELECT
```

This is part of the **aggregate configuration**. The selected BGP routes influence aggregate construction and inherited attributes.

### Neighbor Conditional Advertisement

```text
R1(config-router-af)# neighbor 12.1.1.2 advertise-map ADV exist-map EXIST
```

This is part of a **neighbor-specific advertisement policy**. It advertises routes selected by `ADV` when the condition identified by `EXIST` is met.

A variation uses `non-exist-map` to advertise selected routes when a condition is absent.

| Feature | Aggregate `advertise-map` | Neighbor `advertise-map` |
|---|---|---|
| Configured under | `aggregate-address` | `neighbor` |
| Main purpose | Select contributors for aggregate attributes | Conditionally advertise routes to a neighbor |
| Uses `as-set` | Yes | No |
| Uses `exist-map` / `non-exist-map` | No | Yes |
| Changes the aggregate's inherited AS_PATH | Yes, when used with `as-set` | No, not by itself |
| Per-neighbor policy | No | Yes |

The two features should not be confused, particularly in troubleshooting tasks involving route maps named `ADV` or `ADVERTISE`.

---

## Verification

### Inspect the Contributing Routes

```text
R1# show ip bgp 10.1.0.0/24
R1# show ip bgp 10.1.1.0/24
```

Check the original AS_PATH, communities, ORIGIN, and other BGP attributes.

Also confirm that both more-specific routes are actually present in the BGP table.

### Inspect the Aggregate

```text
R1# show ip bgp 10.1.0.0/23
R1# show ip bgp
```

Look for:

- The aggregate's AS_PATH, with or without an AS_SET.
- The `Community:` line, including `no-export` or `no-advertise`.
- ORIGIN and MED changes introduced by `attribute-map`.
- `aggregated` and `atomic-aggregate` markers where applicable.
- Suppression (`s`) of the contributing more-specific routes.

**Important:** `ATOMIC_AGGREGATE` is not the same thing as `AGGREGATOR` or an AS_SET. With `as-set`, Cisco generally does not add ATOMIC_AGGREGATE merely because the route is aggregated; an inherited ATOMIC_AGGREGATE can still be present.

### Inspect the Route Maps

```text
R1# show route-map SELECT-AS400
R1# show route-map SET-AGG-COMM
R1# show ip prefix-list SELECT-AS400
```

Confirm that the route maps contain the intended **permit** sequences, `match` clauses, and `set` actions.

### Inspect Advertisements to R2

```text
R1# show ip bgp neighbors 12.1.1.2 advertised-routes
```

Check that the aggregate appears and that the /24 contributors are suppressed when `summary-only` is used.

On R2:

```text
R2# show ip bgp 10.1.0.0/23
```

Examine the AS_PATH and communities from R2's perspective. The aggregate is advertised by AS 100, so an EBGP neighbor normally sees AS 100 prepended to the AS_PATH constructed on R1.

### Common Problems

| Symptom | Likely Cause or Check |
|---|---|
| Aggregate is not generated | No eligible more-specific BGP route; `advertise-map` matches no suitable contributors; verify actual BGP state |
| Aggregate exists on R1 but is not advertised to R2 | Inherited `NO_EXPORT` / `NO_ADVERTISE`, outbound filtering, or other BGP advertisement restriction |
| Unexpected ASNs appear in AS_SET | The `advertise-map` is absent or selects more contributors than intended |
| An expected ASN is missing from AS_PATH | The contributing route was excluded by `advertise-map` |
| Community was not removed | `attribute-map` not applied as intended, or `set community ... additive` retained it |
| R2 does not see a new community | Verify `neighbor ... send-community` and outbound policy |
| A /24 is still advertised | `advertise-map` does not suppress prefixes; check `summary-only` and `suppress-map` |
| Aggregate is rejected by a remote AS | Its own ASN may appear in the aggregate's AS_PATH, including AS_SET |

---

## Key Points

- **`advertise-map` selects attribute contributors**. With `as-set`, it affects which ASNs and communities become part of the aggregate.
- **`attribute-map` sets attributes on the aggregate**. It can remove `NO_EXPORT` without removing the contributing ASNs from the AS_SET.
- **`attribute-map` overrides selected inherited attributes** that its `set` commands explicitly modify.
- **`advertise-map` does not replace `summary-only`, `suppress-map`, or `unsuppress-map`**. Those features control advertisement of the individual more-specific routes.
- **Suppressing a route does not necessarily remove its attributes from the aggregate**.
- **Excluding a contributing ASN changes AS_PATH loop-prevention behavior**. Understand the consequences before manipulating AS_SET membership.
- **`aggregate-address ... advertise-map` and `neighbor ... advertise-map ... exist-map` are different mechanisms**.

---

## Related Pages

- [Aggregation and AS-SET Overview](overview.md)
- [BGP Aggregation Configuration](configuration.md)
- [AS-SET and Attribute Preservation](as-set.md)
- [Summary-Only, Suppress-Map, and Unsuppress-Map](suppression.md)
- [BGP Conditional Advertisement](../../routing-policies/conditional-advertisement.md)

---

## References

- [Cisco: Understand Route Aggregation in BGP](https://www.cisco.com/c/en/us/support/docs/ip/border-gateway-protocol-bgp/5441-aggregation.html)
- [Cisco: BGP Command Reference, `aggregate-address`](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_bgp/command/irg-cr-book/bgp-a1.html)
- [Cisco Press: BGP Fundamentals, Route Summarization](https://www.ciscopress.com/articles/article.asp?p=2756480&seqNum=13)
