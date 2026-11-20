# Summary-Only, Suppress-Map, and Unsuppress-Map

**BGP aggregation** creates a less-specific prefix from more-specific routes in the local BGP table. By default, creating an aggregate does **not** prevent BGP from advertising the contributing routes as well.

Cisco IOS XE provides three related mechanisms to control those advertisements:

- **`summary-only`:** Suppresses **all** more-specific routes contributing to the aggregate.
- **`suppress-map`:** Suppresses **selected** contributing routes, as identified by a route map.
- **`neighbor unsuppress-map`:** Restores advertisement of **selected previously suppressed routes** to a particular BGP neighbor.

The main distinction is **scope**: `summary-only` and `suppress-map` control suppression at the aggregate, whereas `unsuppress-map` is applied to an individual neighbor.

---

## Example Topology

Consider this network:

```text
           R3 (AS 300)       R4 (AS 400)
           10.1.2.0/24       10.1.0.0/24
           10.1.3.0/24       10.1.1.0/24
                \               /
                 \             /
                  \           /
                   R2 (AS 200)
                    Aggregator
                   10.1.0.0/22
                    /      \
                   /        \
             R1 (AS 100)   R5 (AS 500)
             12.1.1.1      25.1.1.5

         All links represent EBGP sessions.
```

R2 learns the following four routes:

| Prefix | Learned From | AS_PATH on R2 |
|---|---|---|
| `10.1.0.0/24` | R4 | `400` |
| `10.1.1.0/24` | R4 | `400` |
| `10.1.2.0/24` | R3 | `300` |
| `10.1.3.0/24` | R3 | `300` |

The four /24s fit within `10.1.0.0/22` (`255.255.252.0`). R2 can generate that aggregate because it has matching more-specific routes in its **BGP table**.

Assume all BGP sessions are established and there are no additional inbound or outbound filtering policies. For the configurations below, R2 has these EBGP neighbors:

```text
R2(config)# router bgp 200
R2(config-router)# neighbor 12.1.1.1 remote-as 100
R2(config-router)# neighbor 23.1.1.3 remote-as 300
R2(config-router)# neighbor 24.1.1.4 remote-as 400
R2(config-router)# neighbor 25.1.1.5 remote-as 500
```

The examples configure IPv4 unicast under `router bgp`. On platforms using explicit IPv4 address-family configuration, the aggregation and `unsuppress-map` commands can be entered under `address-family ipv4 unicast` instead.

---

## Default Aggregation: No Suppression

Configure the aggregate on R2:

```text
R2(config)# router bgp 200
R2(config-router)# aggregate-address 10.1.0.0 255.255.252.0
```

This adds the locally generated `10.1.0.0/22` route to BGP. **It does not suppress the /24 routes.**

Assuming the routes are otherwise eligible for advertisement, both R1 and R5 receive:

```text
10.1.0.0/22     Aggregate
10.1.0.0/24     More-specific
10.1.1.0/24     More-specific
10.1.2.0/24     More-specific
10.1.3.0/24     More-specific
```

The /24 routes still take precedence for forwarding wherever both the /22 and /24 match, due to **longest-prefix matching**.

Although the /22 reduces the need to advertise individual routes in principle, merely creating it does not reduce the number of BGP prefixes sent to these neighbors.

---

## `summary-only`: Suppress All Contributing Routes

Remove the previous aggregation statement, then recreate it with `summary-only`:

```text
R2(config)# router bgp 200
R2(config-router)# no aggregate-address 10.1.0.0 255.255.252.0
R2(config-router)# aggregate-address 10.1.0.0 255.255.252.0 summary-only
```

The `summary-only` keyword marks the contributing /24 routes as **suppressed**.

An abbreviated, illustrative BGP table on R2 looks like this:

```text
R2# show ip bgp

   Network          Next Hop       Path
*> 10.1.0.0/22      0.0.0.0        i
s> 10.1.0.0/24      24.1.1.4      400 i
s> 10.1.1.0/24      24.1.1.4      400 i
s> 10.1.2.0/24      23.1.1.3      300 i
s> 10.1.3.0/24      23.1.1.3      300 i
```

The **`s` status code** indicates that a BGP path has been suppressed from normal advertisement by aggregation.

Both R1 and R5 now receive only:

```text
10.1.0.0/22
```

**Important:** Suppression does **not** delete the /24s from R2's BGP table or remove them from its IP routing table merely because they are suppressed. The /24s can still be used for forwarding, assuming they are installed in the RIB.

Also, the aggregate can continue to be advertised as long as the aggregate-generation requirements are met, even if some contributing /24s disappear. This is important when considering traffic blackholing.

### Does `summary-only` Suppress Every More-Specific Route in BGP?

No. Its scope is the **more-specific routes covered by that aggregate**, not every prefix known to the BGP process.

For example, aggregating `10.1.0.0/22` does not suppress an unrelated route such as `172.16.10.0/24`.

---

## `suppress-map`: Suppress Selected Contributing Routes

Sometimes the aggregate should be advertised while only **certain** more-specific routes are hidden.

For example, R2 should suppress `10.1.2.0/24` but continue advertising the other three /24s.

### Step 1: Match the Prefix to Suppress

```text
R2(config)# ip prefix-list SUPPRESS-PFX seq 5 permit 10.1.2.0/24
```

This prefix-list entry matches exactly `10.1.2.0/24`.

A route map references the prefix list:

```text
R2(config)# route-map SUP permit 10
R2(config-route-map)# match ip address prefix-list SUPPRESS-PFX
```

**A `permit` route-map entry identifies prefixes to suppress.** A `deny` route-map entry, or the implicit deny at the end of the route map, does **not** select a prefix for suppression.

Notice the double use of `permit`:

1. The prefix list permits `10.1.2.0/24` to match the `match ip address prefix-list` condition.
2. The route map permits the matched route into the **set of routes to suppress**.

In this context, **route-map `permit` does not mean "advertise the route."** It means that the route matches the suppression policy.

### Step 2: Configure the Aggregate with `suppress-map`

Replace the previous `summary-only` command:

```text
R2(config)# router bgp 200
R2(config-router)# no aggregate-address 10.1.0.0 255.255.252.0 summary-only
R2(config-router)# aggregate-address 10.1.0.0 255.255.252.0 suppress-map SUP
```

R2 now generates the /22 aggregate but suppresses only the /24 selected by `SUP`.

An illustrative BGP table on R2 is:

```text
R2# show ip bgp

   Network          Next Hop       Path
*> 10.1.0.0/22      0.0.0.0        i
*> 10.1.0.0/24      24.1.1.4      400 i
*> 10.1.1.0/24      24.1.1.4      400 i
s> 10.1.2.0/24      23.1.1.3      300 i
*> 10.1.3.0/24      23.1.1.3      300 i
```

Assuming no other advertisement restrictions, R1 and R5 receive:

| Prefix | Advertised? | Reason |
|---|---|---|
| `10.1.0.0/22` | Yes | Aggregate |
| `10.1.0.0/24` | Yes | Not matched by `SUP` |
| `10.1.1.0/24` | Yes | Not matched by `SUP` |
| `10.1.2.0/24` | **No** | Matched by `SUP` |
| `10.1.3.0/24` | Yes | Not matched by `SUP` |

### Suppressing Multiple Prefixes

To suppress both `10.1.2.0/24` and `10.1.3.0/24`, add a second prefix-list entry:

```text
R2(config)# ip prefix-list SUPPRESS-PFX seq 10 permit 10.1.3.0/24
```

Both routes now match the existing `SUP` route map. There is no need to create another aggregation statement.

Alternatively, if the goal is to match all /24s within the /22, use:

```text
R2(config)# ip prefix-list SUPPRESS-ALL seq 5 permit 10.1.0.0/22 ge 24 le 24
R2(config)# route-map SUP-ALL permit 10
R2(config-route-map)# match ip address prefix-list SUPPRESS-ALL
```

This matches **/24 prefixes** inside `10.1.0.0/22`. It does not match the /22 itself or /23 prefixes. It does not create the aggregate; the route map must still be referenced by `aggregate-address`.

---

## `summary-only` and `suppress-map` Together

Consider:

```text
R2(config)# router bgp 200
R2(config-router)# aggregate-address 10.1.0.0 255.255.252.0 summary-only suppress-map SUP
```

**On Cisco IOS/IOS XE, when both are configured on the same aggregate, `suppress-map` takes precedence. The `summary-only` keyword has no effect.**

With the original `SUP` route map matching only `10.1.2.0/24`, the result is:

| Prefix | Suppressed? |
|---|---|
| `10.1.0.0/24` | No |
| `10.1.1.0/24` | No |
| `10.1.2.0/24` | **Yes** |
| `10.1.3.0/24` | No |

The /22 aggregate is still generated. This is the same selective suppression behavior as configuring `suppress-map SUP` alone.

**CCIE exam point:** Do not assume `summary-only` wins simply because it sounds more restrictive. Cisco explicitly documents the opposite precedence.

For clarity and easier troubleshooting, configure **only one** of `summary-only` or `suppress-map` unless you have a specific reason to retain both.

---

## `neighbor unsuppress-map`: Restore Routes to One Neighbor

Both `summary-only` and `suppress-map` suppress routes from normal advertisement **to all BGP neighbors**. However, some neighbors may need one or more of those specific prefixes.

The `neighbor unsuppress-map` command selectively restores suppressed routes **to the specified neighbor only**.

Syntax:

```text
R2(config-router)# neighbor <neighbor-ip> unsuppress-map <route-map-name>
```

### Example: Unsuppress a Route Only Toward R1

Return to the selective suppression example in which `SUP` matches only `10.1.2.0/24`:

```text
R2(config)# ip prefix-list SUPPRESS-PFX seq 5 permit 10.1.2.0/24
R2(config)# route-map SUP permit 10
R2(config-route-map)# match ip address prefix-list SUPPRESS-PFX

R2(config)# router bgp 200
R2(config-router)# aggregate-address 10.1.0.0 255.255.252.0 suppress-map SUP
```

Create a separate route map for R1:

```text
R2(config)# ip prefix-list UNSUPPRESS-PFX seq 5 permit 10.1.2.0/24
R2(config)# route-map UNSUP-R1 permit 10
R2(config-route-map)# match ip address prefix-list UNSUPPRESS-PFX
```

Apply it outbound to R1's BGP neighbor relationship:

```text
R2(config)# router bgp 200
R2(config-router)# neighbor 12.1.1.1 unsuppress-map UNSUP-R1
```

This results in the following advertisements, assuming no other restrictions:

| Prefix | R1 (AS 100) | R5 (AS 500) |
|---|---|---|
| `10.1.0.0/22` | Advertised | Advertised |
| `10.1.0.0/24` | Advertised | Advertised |
| `10.1.1.0/24` | Advertised | Advertised |
| `10.1.2.0/24` | **Unsuppressed and advertised** | **Suppressed** |
| `10.1.3.0/24` | Advertised | Advertised |

R1 receives the previously suppressed /24 because its neighbor-specific route map permits that prefix. R5 still does not receive it.

**Important details:**

- `unsuppress-map` **does not remove** the aggregate route.
- It **does not change** the global suppression decision for other neighbors.
- It can restore **only routes previously suppressed by aggregation**; it is not a general-purpose way to override unrelated route filtering.
- The route map's **`permit` entries identify which suppressed prefixes to advertise** to that neighbor.
- Normal BGP route eligibility and other applicable outbound policies still matter.
- R2's BGP table may continue to display an `s` next to an unsuppressed route because the route is still globally marked as suppressed. Inspect the actual routes advertised **to each neighbor** to verify the effect.

### Unsuppress a Route Originally Suppressed by `summary-only`

`unsuppress-map` is also compatible with `summary-only`.

For example, replace the selective aggregate statement with a `summary-only` aggregate and keep R1's unsuppress policy:

```text
R2(config)# router bgp 200
R2(config-router)# no aggregate-address 10.1.0.0 255.255.252.0 suppress-map SUP
R2(config-router)# aggregate-address 10.1.0.0 255.255.252.0 summary-only
R2(config-router)# neighbor 12.1.1.1 unsuppress-map UNSUP-R1
```

With `UNSUP-R1` still matching `10.1.2.0/24`:

- **R1** receives the /22 aggregate **and** `10.1.2.0/24`.
- **R5** receives only the /22 aggregate.

The other /24s remain suppressed toward both neighbors.

### Does `unsuppress-map` Mean `suppress-map` Must Be Configured?

No. `unsuppress-map` selects a subset of **already suppressed** routes. Those routes can have been suppressed by either `summary-only` or `suppress-map`.

---

## Understanding Route-Map Permit and Deny

Route maps are evaluated in sequence order, and the effect of a match depends on **the feature that uses the route map**.

For aggregation suppression:

| Feature | Route-map `permit` match | Route-map `deny` match or implicit deny |
|---|---|---|
| `aggregate-address ... suppress-map` | **Suppress** the contributing prefix | **Do not suppress** through this map |
| `neighbor ... unsuppress-map` | **Unsuppress** the prefix for that neighbor, if previously suppressed | **Do not unsuppress** through this map |

For example:

```text
R2(config)# route-map SUP deny 10
R2(config-route-map)# match ip address prefix-list SUPPRESS-PFX
```

Here, a route matching the prefix list is **not selected for suppression** by this route map. If there is no later matching `permit` sequence, it remains eligible for normal advertisement, subject to other policies.

This is different from a route map used for outbound BGP route filtering, where a route-map `deny` prevents advertisement.

**Do not confuse the route map's control action with the permit/deny action of the referenced prefix list.** The prefix list determines which prefixes match; the route-map sequence determines how the aggregation feature treats those matches.

---

## Suppression Versus Attribute Contribution

Suppression controls **which contributing prefixes are advertised**, not whether they can contribute attributes to the aggregate.

For example, suppose the following prefixes contribute to `10.1.0.0/22`:

```text
10.1.0.0/24    AS_PATH: 400
10.1.2.0/24    AS_PATH: 300
```

Even if `10.1.2.0/24` is suppressed, it can still participate in determining the aggregate's attributes when `as-set` is enabled:

```text
R2(config-router)# aggregate-address 10.1.0.0 255.255.252.0 as-set suppress-map SUP
```

If both paths contribute to the attribute calculation, the aggregate can contain an AS_SET including `300` and `400`:

```text
{300,400}
```

The same principle is important for inherited BGP communities: with `as-set`, a community such as `NO_EXPORT` on a suppressed contributor may affect the aggregate's advertisement.

**Suppressed does not mean excluded from aggregation.** To control which contributors supply attributes, use `advertise-map`; to override selected aggregate attributes, use `attribute-map`.

See [AS-SET and Attribute Preservation](as-set.md) and [Advertise-Map and Attribute-Map](advertise-attribute-map.md).

---

## Verification

### Check the BGP Table on the Aggregating Router

```text
R2# show ip bgp
```

Look for:

- The aggregate prefix, such as `10.1.0.0/22`.
- `s` status codes on contributing routes selected for suppression.
- More-specific routes without the `s` code when they are not suppressed.

Check individual routes:

```text
R2# show ip bgp 10.1.0.0/22
R2# show ip bgp 10.1.2.0/24
```

### Verify the Prefix List and Route Map

```text
R2# show ip prefix-list
R2# show route-map SUP
R2# show route-map UNSUP-R1
R2# show running-config | section router bgp
```

Confirm that:

1. The prefixes match the intended masks and ranges.
2. The route-map sequences use the intended `permit` and `deny` actions.
3. The correct route maps are referenced by the aggregation and neighbor statements.

### Verify Per-Neighbor Advertisement

The most important checks are:

```text
R2# show ip bgp neighbors 12.1.1.1 advertised-routes
R2# show ip bgp neighbors 25.1.1.5 advertised-routes
```

These commands show whether R1 and R5 are being sent different sets of prefixes.

Then confirm the result on the receiving routers:

```text
R1# show ip bgp
R5# show ip bgp
```

Do not conclude that a route is being advertised merely because it is present in R2's BGP table without the `s` flag. Other BGP advertisement rules or policies can still prevent transmission.

Likewise, an `s` flag on R2 does not necessarily mean a route is absent from **every** neighbor's advertisements when `unsuppress-map` is in use.

---

## Troubleshooting

| Symptom | Possible Cause | What to Check |
|---|---|---|
| Aggregate never appears | No qualifying more-specific route exists in the BGP table | `show ip bgp`; aggregate address and mask |
| All /24s are suppressed unexpectedly | `summary-only` is configured without an effective `suppress-map` | `show running-config | section router bgp` |
| Only one /24 is suppressed despite `summary-only` | `suppress-map` takes precedence | Route map referenced by `suppress-map` |
| A prefix expected to be suppressed is still advertised | Prefix-list does not match its exact length, or route map does not permit it | `show ip prefix-list`; `show route-map` |
| A suppressed prefix does not reappear on R1 | No matching `permit` entry in R1's `unsuppress-map`, incorrect neighbor, or another advertisement restriction | Neighbor configuration and `advertised-routes` |
| R1 receives a /24 but R5 does not | Neighbor-specific `unsuppress-map` on R1 | Compare advertised routes toward both neighbors |
| Aggregate has unexpected AS_PATH or communities | Suppressed routes can still contribute attributes when `as-set` is configured | `show ip bgp <aggregate>`; `as-set`, `advertise-map`, `attribute-map` |

### Remember the Difference

| Configuration | Effect |
|---|---|
| `aggregate-address A M` | Advertise aggregate **and** eligible more-specific routes |
| `aggregate-address A M summary-only` | Suppress **all** contributing more-specific routes |
| `aggregate-address A M suppress-map SUP` | Suppress **only** contributors permitted by `SUP` |
| `aggregate-address A M summary-only suppress-map SUP` | **`SUP` wins**; `summary-only` has no effect |
| `neighbor N unsuppress-map UNSUP` | Readvertise selected suppressed routes **only to neighbor N** |

---

## Related Pages

- [Aggregation and AS-SET Overview](overview.md)
- [BGP Aggregation Configuration](configuration.md)
- [AS-SET and Attribute Preservation](as-set.md)
- [Advertise-Map and Attribute-Map](advertise-attribute-map.md)

---

## References

- [Cisco: Understand Route Aggregation in BGP](https://www.cisco.com/c/en/us/support/docs/ip/border-gateway-protocol-bgp/5441-aggregation.html)
- [Cisco: Configuring a Basic BGP Network (IOS XE 17.x)](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-routing/b-ip-routing/m_irg-basic-net-0.html)
- [Cisco: BGP Command Reference, neighbor unsuppress-map](https://www.cisco.com/c/en/us/td/docs/ios/iproute_bgp/command/reference/irg_book/irg_bgp4.html)
