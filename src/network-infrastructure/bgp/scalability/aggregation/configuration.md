# BGP Aggregation Configuration

**BGP route aggregation** allows a BGP router to generate a less-specific prefix from one or more more-specific routes in its BGP table.

On Cisco IOS XE, the primary command is:

```text
R2(config-router-af)# aggregate-address <network> <subnet-mask> [as-set] [summary-only] [suppress-map <map-name>] [advertise-map <map-name>] [attribute-map <map-name>]
```

For example:

```text
R2(config)# router bgp 200
R2(config-router)# address-family ipv4 unicast
R2(config-router-af)# aggregate-address 10.1.0.0 255.255.252.0
```

This configures R2 to generate `10.1.0.0/22` when it has at least one eligible, contributing more-specific BGP route within that prefix.

**By default, the aggregate is advertised in addition to the more-specific routes.** Creating an aggregate does not automatically suppress its component routes.

---

## Aggregation Requirements

For the `aggregate-address` command to generate a BGP aggregate:

1. The address and mask must identify the intended aggregate prefix.
2. At least one matching **more-specific route must exist in the local BGP table**.
3. The route must be eligible to contribute to the aggregate.

For example, the aggregate `10.1.0.0/22` covers:

```text
10.1.0.0/24
10.1.1.0/24
10.1.2.0/24
10.1.3.0/24
```

However, **all four /24s do not need to exist**. The presence of a single eligible more-specific BGP route can be sufficient to generate the /22.

A route that exists only in the IP routing table does not, by itself, satisfy the requirement to create an aggregate with `aggregate-address`. It must first be introduced into BGP, for example through a `network` statement or redistribution.

Similarly, an exact `10.1.0.0/22` route is not a *more-specific* component of `10.1.0.0/22`.

### Prefix Alignment

The address and mask must form a valid network prefix.

For example:

| Aggregate | Covered Address Range |
|---|---|
| `10.1.0.0/22` | `10.1.0.0` through `10.1.3.255` |
| `10.1.4.0/22` | `10.1.4.0` through `10.1.7.255` |
| `10.1.0.0/23` | `10.1.0.0` through `10.1.1.255` |

`10.1.1.0/22` is not the correctly aligned network address for a /22. Use `10.1.0.0/22` instead.

---

## Example Topology

Consider the following BGP topology:

```text
               AS 100                AS 200

                R1 ------------------- R2
                  12.1.1.1   12.1.1.2 /  \
                                    /     \
                           23.1.1.2      24.1.1.2
                                  /         \
                                 /           \
                               R3            R4
                          23.1.1.3        24.1.1.4
                            AS 300          AS 400

                        10.1.2.0/24     10.1.0.0/24
                        10.1.3.0/24     10.1.1.0/24
```

All three links are EBGP connections. R3 and R4 originate four /24 routes; R2 aggregates them into `10.1.0.0/22` and advertises the aggregate to R1.

### Addressing

| Link | R2 Address | Peer Address |
|---|---|---|
| R1 to R2 | `12.1.1.2` | R1: `12.1.1.1` |
| R2 to R3 | `23.1.1.2` | R3: `23.1.1.3` |
| R2 to R4 | `24.1.1.2` | R4: `24.1.1.4` |

The directly connected interface IP addresses and BGP sessions are assumed to be operational. The examples focus on BGP route origination and aggregation.

### R3: Originate Two /24s

On R3, AS 300:

```text
R3(config)# interface Loopback1
R3(config-if)# ip address 10.1.2.1 255.255.255.0
R3(config-if)# exit
R3(config)# interface Loopback2
R3(config-if)# ip address 10.1.3.1 255.255.255.0
R3(config-if)# exit

R3(config)# router bgp 300
R3(config-router)# neighbor 23.1.1.2 remote-as 200
R3(config-router)# address-family ipv4 unicast
R3(config-router-af)# network 10.1.2.0 mask 255.255.255.0
R3(config-router-af)# network 10.1.3.0 mask 255.255.255.0
```

### R4: Originate Two /24s

On R4, AS 400:

```text
R4(config)# interface Loopback1
R4(config-if)# ip address 10.1.0.1 255.255.255.0
R4(config-if)# exit
R4(config)# interface Loopback2
R4(config-if)# ip address 10.1.1.1 255.255.255.0
R4(config-if)# exit

R4(config)# router bgp 400
R4(config-router)# neighbor 24.1.1.2 remote-as 200
R4(config-router)# address-family ipv4 unicast
R4(config-router-af)# network 10.1.0.0 mask 255.255.255.0
R4(config-router-af)# network 10.1.1.0 mask 255.255.255.0
```

The connected routes provide exact matches for the BGP `network` statements.

### R2: Configure EBGP Neighbors

On R2, AS 200:

```text
R2(config)# router bgp 200
R2(config-router)# neighbor 12.1.1.1 remote-as 100
R2(config-router)# neighbor 23.1.1.3 remote-as 300
R2(config-router)# neighbor 24.1.1.4 remote-as 400
```

Before configuring aggregation, the expected BGP prefixes on R2 are:

```text
10.1.0.0/24    AS_PATH: 400
10.1.1.0/24    AS_PATH: 400
10.1.2.0/24    AS_PATH: 300
10.1.3.0/24    AS_PATH: 300
```

Verify the routes with:

```text
R2# show ip bgp
```

If one or more routes are missing, verify the originating router's `network` statements, its routing table, and the EBGP session.

---

## Basic Aggregation Without Suppression

On R2:

```text
R2(config)# router bgp 200
R2(config-router)# address-family ipv4 unicast
R2(config-router-af)# aggregate-address 10.1.0.0 255.255.252.0
```

R2 generates `10.1.0.0/22` and retains the four /24 routes.

The expected BGP prefixes on R2 are:

```text
10.1.0.0/22    Aggregate (locally generated)
10.1.0.0/24    Learned from AS 400
10.1.1.0/24    Learned from AS 400
10.1.2.0/24    Learned from AS 300
10.1.3.0/24    Learned from AS 300
```

Assuming all paths are eligible for advertisement and there are no filtering policies, **R2 advertises all five prefixes to R1**.

### What Happens to the AS_PATH?

Without `as-set`, the aggregate does not preserve the AS_PATHs of the contributing /24 routes.

For example, R1 receives the /22 with an AS_PATH of:

```text
200
```

The separately advertised /24s retain their individual AS_PATH information:

```text
10.1.0.0/24    200 400
10.1.1.0/24    200 400
10.1.2.0/24    200 300
10.1.3.0/24    200 300
```

Because information about the original AS_PATHs was lost during aggregation, the default aggregate is normally marked with the **ATOMIC_AGGREGATE** attribute.

The aggregate is locally generated by R2, so its BGP NEXT_HOP on R2 is `0.0.0.0`. R1 normally sees R2's EBGP interface address as the next hop.

---

## Suppress All More-Specific Routes with `summary-only`

To advertise only the aggregate, add `summary-only`:

```text
R2(config)# router bgp 200
R2(config-router)# address-family ipv4 unicast
R2(config-router-af)# aggregate-address 10.1.0.0 255.255.252.0 summary-only
```

The four more-specific prefixes remain in R2's BGP table but are marked as suppressed.

A simplified BGP table on R2 might look like this:

```text
     Network          Next Hop       Path
 *>  10.1.0.0/22      0.0.0.0        i
 s>  10.1.0.0/24      24.1.1.4      400 i
 s>  10.1.1.0/24      24.1.1.4      400 i
 s>  10.1.2.0/24      23.1.1.3      300 i
 s>  10.1.3.0/24      23.1.1.3      300 i
```

The `s` status code indicates that a route is suppressed for ordinary BGP advertisement.

R1 normally receives only:

```text
10.1.0.0/22    AS_PATH: 200
```

**Suppression does not delete the more-specific routes from R2's BGP table or prevent them from being used for local forwarding.** It controls the advertisement of those BGP prefixes.

For complete suppression and unsuppression examples, see [Summary-Only, Suppress-Map, and Unsuppress-Map](suppression.md).

---

## Selective Suppression with `suppress-map`

The `suppress-map` keyword permits selective suppression of contributing routes instead of suppressing every more-specific route.

Suppose R2 should suppress only `10.1.2.0/24`.

### Step 1: Configure a Prefix List

```text
R2(config)# ip prefix-list SUPPRESS-PFX seq 5 permit 10.1.2.0/24
```

### Step 2: Configure the Route Map

```text
R2(config)# route-map SUP permit 10
R2(config-route-map)# match ip address prefix-list SUPPRESS-PFX
```

The route-map `permit` sequence identifies the route to suppress. Other component routes do not match this route map.

### Step 3: Configure the Aggregate

```text
R2(config)# router bgp 200
R2(config-router)# address-family ipv4 unicast
R2(config-router-af)# aggregate-address 10.1.0.0 255.255.252.0 suppress-map SUP
```

The expected suppression status on R2 is:

| Prefix | Suppressed? | Normally Advertised to R1? |
|---|---|---|
| `10.1.0.0/22` | No, aggregate | Yes |
| `10.1.0.0/24` | No | Yes |
| `10.1.1.0/24` | No | Yes |
| `10.1.2.0/24` | Yes | No |
| `10.1.3.0/24` | No | Yes |

These expected advertisements assume that no other BGP policy or attribute prevents advertisement.

### Combining `summary-only` and `suppress-map`

Cisco documents an important precedence rule:

```text
R2(config-router-af)# aggregate-address 10.1.0.0 255.255.252.0 summary-only suppress-map SUP
```

When both keywords appear, **`suppress-map` takes precedence and `summary-only` has no effect**.

In this example, only `10.1.2.0/24` is suppressed. The other three /24s remain eligible for advertisement along with the /22.

For clarity, use `summary-only` when suppressing every more-specific route, and `suppress-map` when suppressing selected routes.

---

## Preserve Contributor ASNs with `as-set`

The `as-set` keyword instructs BGP to include AS_PATH information from the contributing routes in an **AS_SET**:

```text
R2(config)# router bgp 200
R2(config-router)# address-family ipv4 unicast
R2(config-router-af)# aggregate-address 10.1.0.0 255.255.252.0 as-set summary-only
```

In the example topology, R1 can receive the aggregate with an AS_PATH resembling:

```text
200 {300,400}
```

- `200` is the AS_SEQUENCE element prepended by R2 when advertising to R1.
- `{300,400}` is an **unordered AS_SET** containing ASNs contributed by the more-specific routes.

The AS_SET can participate in AS_PATH loop detection: a router normally rejects an update containing its own ASN, including when that ASN appears within an AS_SET.

Using `as-set` can cause an aggregate's attributes to change when its contributing routes change, generating additional updates.

**Standards note:** RFC 9774 (May 2025) deprecated the origination of AS_SET and AS_CONFED_SET. Except where explicitly configured for a transition, compliant speakers must not advertise these path segment types and must treat received updates containing them as withdrawals. Cisco IOS XE continues to document the command, so it remains useful to understand in older implementations and CCIE labs. Do not assume AS_SET-based aggregation is interoperable with current Internet BGP deployments.

See [AS-SET and Attribute Preservation](as-set.md) for the detailed rules and attribute behavior.

---

## Other Aggregation Options

Several keywords allow finer control over how aggregates are generated and advertised.

| Keyword | Purpose |
|---|---|
| No keyword | Generate an aggregate; leave more-specific routes eligible for advertisement |
| `summary-only` | Suppress advertisement of all contributing more-specific routes |
| `suppress-map <map>` | Select the more-specific routes to suppress |
| `as-set` | Include contributing AS_PATH information in an AS_SET |
| `advertise-map <map>` | Select contributing routes whose attributes are used to build the aggregate |
| `attribute-map <map>` | Modify attributes of the generated aggregate |

An additional **neighbor-level** option is available:

```text
R2(config-router-af)# neighbor 12.1.1.1 unsuppress-map UNSUP
```

`unsuppress-map` selectively allows previously suppressed more-specific prefixes to be advertised to a particular neighbor. It does not unsuppress those routes globally for every neighbor.

**Important:** `aggregate-address ... advertise-map` is not the same feature as `neighbor ... advertise-map ... exist-map` or `non-exist-map` used for BGP conditional advertisement. In an aggregation command, `advertise-map` controls **which component routes contribute attributes to the aggregate**, not which neighbors receive the aggregate.

For the full configurations, see:

- [Summary-Only, Suppress-Map, and Unsuppress-Map](suppression.md)
- [Advertise-Map and Attribute-Map](advertise-attribute-map.md)

---

## The Aggregate's Null0 Route

When Cisco IOS XE generates a BGP aggregate, it installs a **discard route to Null0** for that aggregate in the IP routing table, subject to normal route installation behavior.

For example:

```text
R2# show ip route 10.1.0.0 255.255.252.0
```

The aggregate appears as a route toward `Null0`.

The Null0 route prevents packets covered by the aggregate from looping when no more-specific route is available for the destination.

Consider traffic to `10.1.2.10`:

- If R2 has a usable `10.1.2.0/24` route, the /24 wins by **longest-prefix matching**.
- If the /24 disappears but the /22 aggregate remains installed, R2 may discard the packet using the aggregate's Null0 route.

Thus, the aggregate can remain advertised even when some, but not all, contributing routes disappear. A single remaining eligible component can keep the /22 active.

**You do not need to manually configure a static Null0 route merely to make `aggregate-address` work.**

---

## `aggregate-address` vs. Static Null0 Summarization

BGP also supports originating an aggregate by configuring a static Null0 route and advertising that exact route using a `network` statement.

For example:

```text
R2(config)# ip route 10.1.0.0 255.255.252.0 Null0

R2(config)# router bgp 200
R2(config-router)# address-family ipv4 unicast
R2(config-router-af)# network 10.1.0.0 mask 255.255.252.0
```

These methods are not equivalent.

| Feature | `aggregate-address` | Static Null0 + `network` |
|---|---|---|
| Required trigger | Eligible more-specific BGP component route | Exact matching aggregate route in IP RIB |
| Aggregate disappears when the last BGP component disappears | Normally yes | No, not while the static route remains active and is originated |
| Automatic component suppression options | Yes | No |
| AS_SET generation | Supported with `as-set` on implementations that allow it | No |
| Null0 discard route | Normally generated automatically | Manually configured |

The static route approach can be useful when an aggregate must be advertised independently of individual component routes, but its persistence can also cause traffic to be discarded if the underlying destinations become unreachable.

---

## Verification

Always verify the **local aggregate**, its **component routes**, and its **advertisements to neighbors** separately.

### 1. Confirm the Aggregate Exists

```text
R2# show ip bgp
R2# show ip bgp 10.1.0.0/22
```

The aggregate should normally appear with a locally originated next hop of `0.0.0.0`.

### 2. Examine the Aggregate's Attributes

```text
R2# show ip bgp 10.1.0.0/22
```

Check for:

- The aggregate prefix and best-path status.
- `atomic-aggregate` and `aggregated by` details where displayed.
- AS_PATH information and any AS_SET.
- Communities that could affect advertisement, especially `NO_EXPORT` or `NO_ADVERTISE`.

### 3. Identify Suppressed Routes

```text
R2# show ip bgp
```

Look for the `s` status code beside contributing routes.

**Do not confuse `s` (suppressed) with `S` (stale), `r` (RIB failure), or `*` (valid).**

### 4. Confirm Actual Neighbor Advertisements

```text
R2# show ip bgp neighbors 12.1.1.1 advertised-routes
```

This is especially important when using a `suppress-map`.

A route that lacks the `s` flag in `show ip bgp` is **not necessarily advertised**. Outbound policy, community-based advertisement restrictions, and other BGP rules can still prevent its advertisement.

On R1, confirm what was accepted:

```text
R1# show ip bgp
R1# show ip bgp neighbors 12.1.1.2 routes
```

### 5. Inspect the Discard Route and Contributing Routes

```text
R2# show ip route 10.1.0.0 255.255.252.0
R2# show ip route 10.1.2.0 255.255.255.0
```

The more-specific route normally wins for traffic matching its destination. The aggregate discard route covers any remaining destination addresses within the /22.

---

## Common Troubleshooting Scenarios

### The Aggregate Does Not Appear

Check that at least one eligible **more-specific BGP route** exists inside the aggregate's prefix.

```text
R2# show ip bgp
R2# show ip route
```

A route present only in `show ip route` is insufficient unless it has also been introduced into BGP.

Also verify that the aggregate prefix and mask are correct.

### The Aggregate Appears, but the /24s Still Reach R1

This is normal when `summary-only` or an appropriate `suppress-map` is not configured. Aggregation alone does **not** suppress contributors.

### Only One /24 Is Marked Suppressed

If the aggregate uses `suppress-map SUP`, inspect the route map and its referenced prefix list:

```text
R2# show route-map SUP
R2# show ip prefix-list
R2# show ip bgp
```

Only contributing routes permitted by `SUP` should be marked suppressed.

Even if `summary-only` is also present, `suppress-map` takes precedence.

### An Unsuppressed /24 Is Missing from R1

A missing `s` status code is **not proof that the prefix was sent**.

First, check R2's advertisements:

```text
R2# show ip bgp neighbors 12.1.1.1 advertised-routes
R2# show ip bgp 10.1.0.0/24
```

If the route is not advertised, examine outbound route maps, prefix lists, BGP communities, and other advertisement restrictions.

If R2 reports that the route is advertised, check R1's inbound policies and BGP table.

Do not assume that the base address `10.1.0.0` in `10.1.0.0/24` conflicts with the aggregate `10.1.0.0/22`. **They are distinct prefixes because their mask lengths differ.**

### Traffic Is Discarded Despite the Aggregate Being Present

The aggregate can remain active after one of its component routes disappears. Traffic to the missing component's address space may match the aggregate Null0 route.

Confirm which more-specific routes remain in the forwarding table instead of assuming that reachability to every address covered by the aggregate is guaranteed.

---

## Key Takeaways

- `aggregate-address` requires an eligible, more-specific **BGP** route to generate the aggregate.
- One component is sufficient; all possible component prefixes are not required.
- Without suppression options, **both** the aggregate and eligible more-specific routes are advertised.
- `summary-only` suppresses all contributing more-specific routes.
- `suppress-map` selectively suppresses the routes matched by its route map and takes precedence over `summary-only` if both are configured.
- `as-set` preserves contributor ASN information in an AS_SET, but the feature is deprecated by RFC 9774.
- Cisco IOS XE normally generates a Null0 discard route for the aggregate.
- Use `show ip bgp neighbors <neighbor> advertised-routes` to verify what is actually advertised, not merely the `s` flags in the local BGP table.

---

## References

- [Cisco: Understand Route Aggregation in BGP](https://www.cisco.com/c/en/us/support/docs/ip/border-gateway-protocol-bgp/5441-aggregation.html)
- [Cisco: BGP Command Reference, `aggregate-address`](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_bgp/command/irg-cr-book/bgp-a1.html)
- [Cisco: IP Routing Configuration Guide, Cisco IOS XE 17.x, BGP 4](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-routing/b-ip-routing/m_irg-bgp4.html)
- [Cisco: Troubleshoot BGP Routes That Do Not Advertise](https://www.cisco.com/c/en/us/support/docs/ip/border-gateway-protocol-bgp/19345-bgp-noad.html)
- [RFC 9774: Deprecation of AS_SET and AS_CONFED_SET in BGP](https://www.rfc-editor.org/rfc/rfc9774.html)
