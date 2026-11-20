# AS-SET and Attribute Preservation

When BGP combines multiple more-specific prefixes into an aggregate, it must decide which **path attributes** to attach to the new route.

The `as-set` keyword changes how Cisco IOS XE constructs the aggregate's AS_PATH and handles certain attributes from the contributing routes.

```text
R2(config)# router bgp 200
R2(config-router)# address-family ipv4 unicast
R2(config-router-af)# aggregate-address 10.1.0.0 255.255.252.0 as-set summary-only
```

The important distinction is:

- **Without `as-set`:** The aggregate is originated with a new AS_PATH, without preserving the individual contributing routes' AS_PATHs or communities.
- **With `as-set`:** BGP uses the contributing paths to construct the aggregate's AS_PATH and can inherit their communities and other applicable attributes.

The AS_SET is an **unordered AS_PATH segment**, not a list of the exact paths taken to reach each contributing prefix. Using `as-set` does **not** mean that every attribute is copied unchanged.

> **Standards note:** RFC 9774 (May 2025) deprecated AS_SET and AS_CONFED_SET in BGP. By default, standards-conforming speakers must not advertise these segments and must treat received routes containing them as withdrawn, unless explicitly configured otherwise. Cisco IOS XE still documents the `as-set` command, so the behavior remains useful to understand and test for CCIE labs. Do not assume an AS_SET aggregate will be accepted across a modern production BGP network.

---

## Example Topology

Consider the following four-router topology:

```text
                     AS 100
                       R1
                        |
                        | EBGP
                        |
                     AS 200
                       R2
                      /  \
                EBGP /    \ EBGP
                    /      \
                  R3        R4
                AS 300    AS 400
```

R2 receives four more-specific prefixes:

| Prefix | Received from | AS_PATH at R2 |
|---|---|---|
| `10.1.0.0/24` | R4 | `400` |
| `10.1.1.0/24` | R4 | `400` |
| `10.1.2.0/24` | R3 | `300` |
| `10.1.3.0/24` | R3 | `300` |

R2 aggregates them into `10.1.0.0/22` and advertises the aggregate to R1.

The following examples assume that all four /24s are present as valid contributing BGP routes on R2, that R2 can advertise the aggregate to R1, and that no other policies modify the attributes.

---

## Aggregation Without AS-SET

Configure R2:

```text
R2(config)# router bgp 200
R2(config-router)# address-family ipv4 unicast
R2(config-router-af)# aggregate-address 10.1.0.0 255.255.252.0 summary-only
```

R2 originates `10.1.0.0/22` as a local aggregate. The contributing AS_PATHs are **not preserved** in the new route.

### AS_PATH at R2

```text
Prefix          AS_PATH
10.1.0.0/24     400
10.1.1.0/24     400
10.1.2.0/24     300
10.1.3.0/24     300
10.1.0.0/22     (empty)
```

The aggregate is locally originated, so its AS_PATH is empty **before** it is advertised through EBGP.

### AS_PATH at R1

When R2 advertises the aggregate to R1, it prepends its own ASN, `200`, as usual for EBGP:

```text
R1# show ip bgp

     Network          Next Hop       Metric LocPrf Weight Path
 *>  10.1.0.0/22      12.1.1.2                          0 200 i
```

R1 does **not** see AS 300 or AS 400 in the aggregate's AS_PATH.

The `i` at the end is the BGP **ORIGIN** code (IGP); it is **not** part of the AS_PATH.

### Consequences

- The aggregate saves routing table entries if the /24s are suppressed.
- The original AS_PATH information is lost.
- The aggregate typically carries **ATOMIC_AGGREGATE**, indicating that path information was discarded.
- The contributing routes' communities are not automatically collected into the aggregate.

**Important:** Without AS_SET, R1 cannot infer from the aggregate's AS_PATH that some of its address space was reached through AS 300 and AS 400.

---

## Aggregation With AS-SET

Replace the aggregation configuration on R2 with:

```text
R2(config)# router bgp 200
R2(config-router)# address-family ipv4 unicast
R2(config-router-af)# no aggregate-address 10.1.0.0 255.255.252.0 summary-only
R2(config-router-af)# aggregate-address 10.1.0.0 255.255.252.0 as-set summary-only
```

R2 now incorporates the ASNs from the contributing routes into an AS_SET.

### AS_PATH at R2

```text
Prefix          AS_PATH
10.1.0.0/24     400
10.1.1.0/24     400
10.1.2.0/24     300
10.1.3.0/24     300
10.1.0.0/22     {300,400}
```

The braces indicate an **AS_SET**.

An AS_SET does not impose an order on its members. `{300,400}` and `{400,300}` have the same meaning.

### AS_PATH at R1

When R2 advertises the aggregate to R1, its own ASN is prepended as an AS_SEQUENCE element:

```text
R1# show ip bgp

     Network          Next Hop       Metric LocPrf Weight Path
 *>  10.1.0.0/22      12.1.1.2                          0 200 {300,400} i
```

The AS_PATH contains two different kinds of segments:

```text
200           {300,400}
 |                 |
AS_SEQUENCE      AS_SET
```

- `200`: R2's ASN, prepended when the aggregate is advertised via EBGP.
- `{300,400}`: ASNs retained from the contributing routes, without their ordering.

This preserves information about the participating autonomous systems, but does **not** preserve the exact AS_PATH of each /24.

---

## How BGP Constructs the Aggregated AS_PATH

The result is not always just one AS_SET.

### Different AS_PATHs

If the contributing paths have no common ordered sequence:

```text
10.1.0.0/24    AS_PATH: 300
10.1.1.0/24    AS_PATH: 400
```

The aggregate can contain:

```text
AS_PATH: {300,400}
```

### Identical AS_PATHs

If the contributing routes have the **same AS_PATH**:

```text
10.1.0.0/24    AS_PATH: 300 500
10.1.1.0/24    AS_PATH: 300 500
```

The aggregate retains that AS_PATH as an ordered sequence:

```text
AS_PATH: 300 500
```

There is no need to convert it to an AS_SET because no ordering information has to be discarded.

### Common Initial AS_SEQUENCE

If the paths share an initial AS_SEQUENCE but differ afterward:

```text
10.1.0.0/24    AS_PATH: 300 500
10.1.1.0/24    AS_PATH: 300 600
```

A valid aggregation result is:

```text
AS_PATH: 300 {500,600}
```

The common ordered portion, `300`, is retained as an AS_SEQUENCE, while the remaining ASNs form an AS_SET.

RFC 4271 describes the rules for forming an aggregate AS_PATH. Implementations have some flexibility in how they construct an AS_PATH that satisfies those rules.

### AS_PATH Length

For the BGP AS_PATH length comparison, an **AS_SET counts as one AS**, regardless of the number of ASNs it contains.

For example:

```text
AS_PATH: 200 {300,400,500,600}
```

Has a path length of **2** for this comparison:

- `200` counts as 1.
- `{300,400,500,600}` counts as 1.

It does **not** count as five AS hops.

---

## AS-SET and AS_PATH Loop Prevention

BGP normally rejects an incoming path that contains the receiving router's own ASN, unless an exception such as `allowas-in` is configured.

ASNs within an AS_SET are included in this loop check.

For example, R2 (AS 200) creates:

```text
10.1.0.0/22    AS_PATH: {300,400}
```

R2 advertises the aggregate to R3 (AS 300):

```text
AS_PATH: 200 {300,400}
```

R3 detects its own ASN (`300`) in the AS_SET and rejects the route through normal AS_PATH loop prevention.

This is an important advantage of preserving the contributing ASNs.

Without `as-set`, R2 would advertise:

```text
AS_PATH: 200
```

That path does not contain `300`, so R3 could accept the aggregate if no other mechanism prevents it.

**The aggregate's AS_SET can protect against some AS-level routing loops, but it does not guarantee that traffic forwarding is loop-free.** An aggregate may cover destinations for which the aggregating router has no reachable more-specific route.

---

## AS-SET and Other BGP Attributes

The `as-set` keyword changes the aggregation of certain **path attributes**, not just the AS_PATH.

However, it is incorrect to say that it blindly copies every attribute from all contributing routes.

| Attribute | Without `as-set` | With `as-set` |
|---|---|---|
| **AS_PATH** | Contributing paths are normally discarded | Contributing ASNs are preserved through AS_SEQUENCE and/or AS_SET segments |
| **ORIGIN** | New Cisco aggregate is commonly originated with `i` | Aggregation can account for contributing ORIGIN values; see below |
| **COMMUNITY** | Contributing communities are not automatically inherited | Communities can be collected from contributing routes |
| **ATOMIC_AGGREGATE** | Normally generated because path information was discarded | Normally not generated solely because of aggregation when AS_PATH information is retained; see note below |
| **AGGREGATOR** | May identify the router/AS performing aggregation | May identify the router/AS performing aggregation |
| **NEXT_HOP** | Newly determined for the aggregate | Newly determined for the aggregate; contributing next hops are not combined |
| **MED** | Not simply copied from an arbitrary contributor | Not preserved as a collection of contributor MEDs; do not assume inheritance |
| **LOCAL_PREF** | Local BGP policy/default applies | Local BGP policy/default applies; it is not a collected list of contributor LOCAL_PREF values |
| **Weight** | Locally determined on Cisco | Locally determined on Cisco; never advertised as a BGP path attribute |

The detailed behavior of some optional attributes depends on the software implementation, contributor selection, and aggregation policy.

### ORIGIN

ORIGIN identifies how a route was introduced into BGP:

| Code | ORIGIN |
|---|---|
| `i` | IGP |
| `e` | EGP |
| `?` | Incomplete |

When ORIGIN values are combined under the standard aggregation rules, the **least-preferred ORIGIN** wins:

```text
IGP + IGP          -> IGP
IGP + EGP          -> EGP
IGP + INCOMPLETE   -> INCOMPLETE
EGP + INCOMPLETE   -> INCOMPLETE
```

That is, **INCOMPLETE takes precedence over EGP, which takes precedence over IGP** for the aggregate's ORIGIN when these contributing values are combined.

For example, if an aggregate combines one path with ORIGIN `i` and another with ORIGIN `?`, the aggregated ORIGIN is `?` under the RFC 4271 rules.

This is **not** the BGP best-path algorithm choosing a preferred path. It is the rule for creating a single aggregate attribute from multiple routes.

**Cisco lab note:** Plain Cisco `aggregate-address` routes without `as-set` commonly appear with ORIGIN `i`. Verify the actual aggregate using `show ip bgp <aggregate-prefix>` when experimenting with different contributor ORIGIN values.

### COMMUNITY

With `as-set`, Cisco IOS can incorporate communities from contributing routes into the aggregate.

Suppose R2 receives:

| Prefix | AS_PATH | COMMUNITY |
|---|---|---|
| `10.1.0.0/24` | `400` | `no-export` |
| `10.1.1.0/24` | `400` | `65000:100` |
| `10.1.2.0/24` | `300` | None |
| `10.1.3.0/24` | `300` | `65000:200` |

Assume R2 actually receives the communities, which requires suitable community transmission on the advertising neighbors.

With:

```text
R2(config-router-af)# aggregate-address 10.1.0.0 255.255.252.0 as-set summary-only
```

The aggregate can inherit:

```text
AS_PATH:   {300,400}
COMMUNITY: no-export 65000:100 65000:200
```

**Important consequence:** A single contributing route carrying `no-export` can cause the entire aggregate to carry `no-export`.

R2 will then **not advertise that aggregate to ordinary EBGP neighbors**, including R1, even though the other contributing routes do not have that community.

Similarly, an inherited `no-advertise` community can prevent the aggregate from being advertised to *any* BGP neighbor.

This is a common cause of a locally present aggregate being missing from a neighbor's BGP table.

**Standard and extended communities are separate attributes.** Do not assume that every kind of extended or large community is merged in precisely the same way on every Cisco platform or software release. Verify the aggregate's actual attributes.

### ATOMIC_AGGREGATE

`ATOMIC_AGGREGATE` is a well-known discretionary BGP attribute that indicates path information may have been lost during aggregation.

When Cisco IOS XE creates an aggregate **without** `as-set`, this attribute is normally attached automatically.

Cisco's IOS BGP command reference states that the attribute is **not automatically added when `as-set` is specified**. However, an aggregate may still carry the attribute if a contributing route already carried it, and some older Cisco example outputs show `atomic-aggregate` even alongside `as-set`.

Do not confuse the two attributes:

- **AS_SET:** A segment within AS_PATH containing unordered ASNs.
- **ATOMIC_AGGREGATE:** A separate BGP attribute indicating that some path information may have been lost.

Neither attribute identifies which router performed aggregation.

### AGGREGATOR

`AGGREGATOR` is a separate optional transitive attribute that identifies the router that performed aggregation.

It contains:

- The **ASN** of the aggregating router.
- The **BGP router ID** of the aggregating router (normally represented as an IPv4 address).

For example, if R2 is in AS 200 and has BGP router ID `24.1.1.2`:

```text
Aggregated by 200 24.1.1.2
```

This identifies the aggregation point. It is **not** a substitute for the AS_PATH, and configuring `as-set` does not make AGGREGATOR unnecessary.

If the contributing paths already contain AGGREGATOR information from upstream aggregations, the local router may attach a new AGGREGATOR attribute instead of copying those upstream values.

### NEXT_HOP, LOCAL_PREF, and MED

These attributes require particular care:

- **NEXT_HOP:** The aggregate is a new route. On R2, it appears as locally originated, often with next hop `0.0.0.0` in the BGP table. When advertised to an ordinary directly connected EBGP neighbor, the next hop is normally R2's outgoing interface address. BGP does not construct an unordered set of contributor next hops.
- **LOCAL_PREF:** LOCAL_PREF is applied within an AS according to BGP policy and is not normally transmitted to an external AS. `as-set` does not cause the aggregate to advertise a combined LOCAL_PREF value through EBGP.
- **MED:** Contributor MED values cannot be represented by an AS_SET. RFC 4271 restricts aggregation of paths with different MED values. Do not assume the MED from any one contributing route becomes the aggregate's MED. Test the behavior of the particular IOS XE release and use explicit policy when a specific advertised MED is required.

The key point is that **AS_SET preserves AS membership, not the complete original set of BGP path attributes or forwarding decisions**.

---

## Controlling Inherited Attributes

Two keywords are especially relevant when `as-set` is configured:

- `advertise-map`: Selects which contributing routes supply information used to build aggregate attributes, including the AS_PATH and communities.
- `attribute-map`: Explicitly modifies the aggregate's attributes after those contributing attributes have been selected.

These two keywords serve **different purposes**.

### Example: Remove an Inherited NO_EXPORT Community

Suppose a contributing route has the `no-export` community, and it has been inherited by the aggregate.

Configure an attribute map on R2:

```text
R2(config)# route-map CLEAN-AGG permit 10
R2(config-route-map)# set community none

R2(config)# router bgp 200
R2(config-router)# address-family ipv4 unicast
R2(config-router-af)# no aggregate-address 10.1.0.0 255.255.252.0 as-set summary-only
R2(config-router-af)# aggregate-address 10.1.0.0 255.255.252.0 as-set summary-only attribute-map CLEAN-AGG
```

The `attribute-map` removes the aggregate's **standard communities**, including `no-export`.

It does **not** remove the communities from the individual contributing routes themselves.

This can allow the aggregate to be advertised to R1. However, removing a community may override an intentional routing restriction, so use this only when the policy design permits it.

See [Advertise-Map and Attribute-Map](advertise-attribute-map.md) for detailed examples.

### Suppressed Routes Still Contribute Attributes

A common misconception is that only **unsuppressed** routes contribute to the AS_SET.

That is not correct.

For example:

```text
R2(config)# ip prefix-list SUPPRESSED permit 10.1.2.0/24
R2(config)# route-map SUP permit 10
R2(config-route-map)# match ip address prefix-list SUPPRESSED

R2(config)# router bgp 200
R2(config-router)# address-family ipv4 unicast
R2(config-router-af)# aggregate-address 10.1.0.0 255.255.252.0 as-set suppress-map SUP
```

`10.1.2.0/24` is suppressed from advertisement, but it can **still contribute AS 300 and its communities** to the aggregate.

With `as-set`, the aggregate is formed using attributes from **both suppressed and unsuppressed contributing routes**, unless `advertise-map` is used to limit the routes whose attributes are considered.

`summary-only` and `suppress-map` control **advertisement of more-specific routes**; they do not, by themselves, specify which contributor attributes go into the aggregate.

See [Summary-Only, Suppress-Map, and Unsuppress-Map](suppression.md) for additional suppression scenarios.

---

## AS-SET and Convergence

Aggregation can hide individual route changes from downstream routers, but `as-set` can reduce that stability benefit.

Suppose R2 has:

```text
10.1.0.0/24     AS_PATH: 400
10.1.1.0/24     AS_PATH: 400
10.1.2.0/24     AS_PATH: 300
10.1.3.0/24     AS_PATH: 300
```

The aggregate has:

```text
AS_PATH: {300,400}
```

If **both** R3 routes disappear, but R4's routes remain, the aggregate can still exist, but its AS_PATH changes:

```text
Before: {300,400}
After:  400
```

The aggregate must be updated because its attributes changed, even though the `10.1.0.0/22` aggregate prefix remains present.

By contrast, without `as-set`, changes among the contributing /24 routes might not change the aggregate's advertised attributes.

Therefore, an AS_SET can preserve useful information at the cost of **additional BGP UPDATE processing** when contributors change.

---

## Verification

Check the routes contributing to the aggregate:

```text
R2# show ip bgp
```

Inspect the aggregate's detailed attributes:

```text
R2# show ip bgp 10.1.0.0/22
```

Look for:

- The AS_SEQUENCE and AS_SET portions of AS_PATH.
- ORIGIN (`i`, `e`, or `?`).
- Community information, especially `no-export` or `no-advertise`.
- `atomic-aggregate`, if present.
- `aggregated by <ASN> <router-ID>`.

Check the routes R2 advertises to R1:

```text
R2# show ip bgp neighbors 12.1.1.1 advertised-routes
```

Then inspect the route as received by R1:

```text
R1# show ip bgp 10.1.0.0/22
R1# show ip bgp
```

The AS_PATH on R1 should include AS 200 prepended to the aggregate's path when R2 advertises it through EBGP.

**Troubleshooting tip:** If the aggregate exists on R2 but not R1, checking the communities on the **aggregate itself** is especially important when `as-set` is configured. The route might have inherited a restrictive community from just one contributor.

---

## Key Points

- `as-set` preserves ASNs from contributing routes within the aggregate's AS_PATH.
- An AS_SET is **unordered**. It does not preserve each contributor's complete path.
- If contributing AS_PATHs are identical, they can remain an AS_SEQUENCE without an AS_SET.
- An AS_SET counts as **one AS** during the AS_PATH length comparison.
- The receiving router's own ASN appearing anywhere in the AS_PATH, including an AS_SET, can trigger AS loop detection.
- With `as-set`, Cisco can also inherit **communities**. A single `no-export` or `no-advertise` community may prevent the aggregate's advertisement.
- ORIGIN follows aggregation rules when contributor attributes are combined; NEXT_HOP, LOCAL_PREF, and MED are **not** simply copied from each contributor.
- `ATOMIC_AGGREGATE` and `AGGREGATOR` are separate attributes from AS_SET.
- Suppressed routes can still contribute attributes. `advertise-map` limits attribute contributors; `attribute-map` changes the final attributes.
- RFC 9774 deprecated AS_SET for standards-conforming BGP deployments; understand Cisco's traditional behavior for lab and compatibility purposes.

---

## References

- [Cisco: Understand Route Aggregation in BGP](https://www.cisco.com/c/en/us/support/docs/ip/border-gateway-protocol-bgp/5441-aggregation.html)
- [Cisco: aggregate-address Command Reference](https://www.cisco.com/c/en/us/td/docs/ios/iproute_bgp/command/reference/irg_book/irg_bgp1.html)
- [Cisco: IP Routing Configuration Guide, BGP 4](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/ip-routing/b-ip-routing/m_irg-bgp4.html)
- [RFC 4271: A Border Gateway Protocol 4 (BGP-4), Section 9.2.2.2](https://www.rfc-editor.org/rfc/rfc4271.html)
- [RFC 9774: Deprecation of AS_SET and AS_CONFED_SET in BGP](https://www.rfc-editor.org/rfc/rfc9774.html)
