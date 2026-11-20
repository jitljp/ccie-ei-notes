# 1.5.e (ii) Aggregation, as-set

**BGP route aggregation** combines multiple more-specific prefixes into a single, less-specific prefix. It is a scalability mechanism that reduces the number of routes advertised and potentially stored by other BGP routers.

Cisco IOS XE supports BGP aggregation using the `aggregate-address` command.

Aggregation introduces several important considerations:

- **Route generation:** A matching more-specific BGP route is required for an aggregate to be generated.
- **Route advertisement:** By default, the aggregate and its contributing more-specific routes can both be advertised.
- **Route suppression:** `summary-only` and `suppress-map` control which more-specific routes are advertised.
- **Attribute preservation:** `as-set` can preserve information from the AS_PATHs of the contributing routes.
- **Attribute manipulation:** `advertise-map` and `attribute-map` influence the attributes assigned to the aggregate.
- **Forwarding:** Aggregation can create a discard route to prevent forwarding loops for destinations without a reachable more-specific route.

The `as-set` option is relevant to the CCIE Enterprise Infrastructure blueprint. However, **RFC 9774 (May 2025) deprecated AS_SET and AS_CONFED_SET in BGP**, so their use in modern interdomain routing requires special consideration.

---

## Why BGP Aggregation Is Used

Suppose an AS advertises the following routes:

```text
192.0.2.0/24
192.0.3.0/24
```

These two contiguous /24 prefixes can be represented by one /23:

```text
192.0.2.0/23
```

A downstream router that receives only the aggregate needs one BGP route rather than two.

For large networks, aggregation can reduce:

- The number of advertised prefixes.
- The number of BGP entries maintained by downstream routers.
- The amount of update processing caused by changes to individual prefixes.
- The memory and CPU required to maintain routing information.

Aggregation can also hide internal topology changes from external peers. However, losing more-specific routing information can reduce path visibility and influence traffic engineering or failover behavior.

---

## Example Topology

Consider the following network:

```text
       AS 65001             AS 65000            AS 65004

          R1 ----------\
                        \
                         R3 ---------------- R4
                        /
          R2 ----------/
       AS 65002

      R1 advertises: 192.0.2.0/24
      R2 advertises: 192.0.3.0/24

      R3 generates:  192.0.2.0/23
      R4 receives:   192.0.2.0/23

                All sessions: EBGP
```

R3 learns the two more-specific prefixes from separate external autonomous systems.

| Prefix | Learned From | AS_PATH at R3 |
|---|---|---|
| `192.0.2.0/24` | R1 | `65001` |
| `192.0.3.0/24` | R2 | `65002` |

R3 can aggregate these routes into `192.0.2.0/23` before advertising routing information to R4.

**Important:** An aggregate does not have to cover only two prefixes, and the router does not need to learn every possible more-specific prefix within the aggregate range.

---

## Basic Aggregation Configuration

Configure the following on R3:

```text
R3(config)# router bgp 65000
R3(config-router)# address-family ipv4 unicast
R3(config-router-af)# aggregate-address 192.0.2.0 255.255.254.0
```

This creates an aggregate for `192.0.2.0/23` if at least one eligible more-specific BGP route exists within that prefix.

### Aggregate Generation Requirements

The following conditions are important:

1. The `aggregate-address` command must be configured under the appropriate BGP address family.
2. At least one eligible, strictly more-specific route covered by the aggregate must be available in the local BGP routing information.
3. The contributing routes do **not** need to be learned from the same AS or the same neighbor.
4. All possible contributing more-specific prefixes do **not** need to be present.

For example, R3 can generate `192.0.2.0/23` when it knows only `192.0.2.0/24`.

If all eligible contributing routes disappear, the dynamically generated aggregate is withdrawn.

**Important:** `aggregate-address` is different from the BGP `network` command. A `network` statement requires an exact matching prefix in the IP routing table, whereas `aggregate-address` uses available more-specific BGP routing information to generate the summary.

### Advertisement Without `summary-only`

By default, configuring `aggregate-address` does **not** suppress the contributing more-specific routes.

R3 can advertise the following to R4:

```text
192.0.2.0/23   (aggregate)
192.0.2.0/24   (more-specific)
192.0.3.0/24   (more-specific)
```

Actual advertisements remain subject to normal BGP eligibility rules, neighbor configuration, and outbound policies.

Although an aggregate exists, this configuration does not substantially reduce the number of prefixes advertised to R4.

---

## Suppressing More-Specific Routes

The `summary-only` keyword suppresses the advertisement of contributing more-specific routes.

```text
R3(config)# router bgp 65000
R3(config-router)# address-family ipv4 unicast
R3(config-router-af)# aggregate-address 192.0.2.0 255.255.254.0 summary-only
```

R3 now advertises the aggregate while suppressing the corresponding more-specific routes:

```text
Advertised to R4:

192.0.2.0/23
```

The more-specific routes remain in R3's BGP table. They are suppressed from BGP advertisements, not removed from the local routing information.

In `show ip bgp`, the status code `s` identifies a suppressed route.

### Selective Suppression

The `suppress-map` keyword provides more granular control by suppressing only the more-specific routes matched by a route map.

For example:

```text
R3(config)# ip prefix-list SUPPRESS-ROUTES permit 192.0.2.0/24
R3(config)# route-map SUPPRESS permit 10
R3(config-route-map)# match ip address prefix-list SUPPRESS-ROUTES

R3(config)# router bgp 65000
R3(config-router)# address-family ipv4 unicast
R3(config-router-af)# aggregate-address 192.0.2.0 255.255.254.0 suppress-map SUPPRESS
```

In this example, R3 suppresses `192.0.2.0/24`, but it can still advertise `192.0.3.0/24` together with the aggregate.

**Cisco IOS behavior:** When `summary-only` and `suppress-map` are configured together, the selective suppression behavior of `suppress-map` takes precedence over the blanket suppression normally provided by `summary-only`.

### Unsuppressing Routes for Specific Neighbors

The `neighbor unsuppress-map` command permits selected suppressed routes to be advertised to a particular neighbor.

For example:

```text
R3(config)# ip prefix-list UNSUPPRESS-ROUTES permit 192.0.2.0/24
R3(config)# route-map UNSUPPRESS permit 10
R3(config-route-map)# match ip address prefix-list UNSUPPRESS-ROUTES

R3(config)# router bgp 65000
R3(config-router)# address-family ipv4 unicast
R3(config-router-af)# aggregate-address 192.0.2.0 255.255.254.0 summary-only
R3(config-router-af)# neighbor 10.0.34.4 unsuppress-map UNSUPPRESS
```

Assuming `10.0.34.4` is R4's BGP neighbor address, R3 can advertise `192.0.2.0/24` to R4 despite its global suppression by `summary-only`.

Other neighbors continue to receive the aggregate without that more-specific route, unless their own policies allow it.

See [Summary-Only, Suppress-Map, and Unsuppress-Map](suppression.md) for additional examples and interactions.

---

## Aggregation Without `as-set`

By default, an aggregate created without `as-set` does not retain the complete AS_PATH information of its contributing routes.

Recall that R3 receives:

```text
192.0.2.0/24   AS_PATH: 65001
192.0.3.0/24   AS_PATH: 65002
```

With this configuration:

```text
R3(config-router-af)# aggregate-address 192.0.2.0 255.255.254.0 summary-only
```

R3 originates the aggregate as a locally generated BGP route.

When R4 receives it through EBGP, its AS_PATH can appear as:

```text
65000
```

The original ASNs (`65001` and `65002`) are no longer represented in the aggregate's AS_PATH.

### ATOMIC_AGGREGATE and AGGREGATOR

BGP defines two attributes associated with aggregation:

| Attribute | Purpose |
|---|---|
| `ATOMIC_AGGREGATE` | Signals that information was lost during aggregation |
| `AGGREGATOR` | Identifies the ASN and BGP identifier of the router that performed the aggregation |

Cisco IOS XE typically sets `ATOMIC_AGGREGATE` when generating an aggregate without `as-set`. The detailed BGP output can also identify the aggregating router, for example with an `aggregated by` field.

These attributes do **not** restore the original ordering or full details of the contributing AS_PATHs.

---

## Aggregation With `as-set`

The `as-set` keyword preserves the AS numbers represented in the AS_PATHs of contributing routes, using AS_PATH set information rather than simply discarding their histories.

Configure:

```text
R3(config)# router bgp 65000
R3(config-router)# address-family ipv4 unicast
R3(config-router-af)# aggregate-address 192.0.2.0 255.255.254.0 as-set summary-only
```

With both contributing routes present, R4 can receive an AS_PATH such as:

```text
65000 {65001,65002}
```

This contains two different types of AS_PATH segments:

- `65000` is part of an **AS_SEQUENCE**, whose order is significant.
- `{65001,65002}` is an **AS_SET**, whose members are unordered.

The order of ASNs shown within an AS_SET is not significant. Cisco might display `{65002,65001}` instead.

### AS_SET and Loop Prevention

Traditionally, the AS_SET helped preserve AS_PATH loop-detection information after aggregation.

For example, if the aggregate is advertised back to AS 65001, a BGP router in AS 65001 can detect its own ASN inside the AS_SET and reject the route under traditional AS_PATH loop-prevention rules.

Without `as-set`, the aggregate's AS_PATH would not include AS 65001, so this particular loop-detection mechanism would not identify the AS as a contributor.

### AS_SET and Path Selection

AS_SET is a segment of the AS_PATH attribute, not a BGP community.

In traditional BGP AS_PATH length comparison:

- Each ASN in an AS_SEQUENCE contributes to the path length.
- An AS_SET segment counts as **one** AS_PATH element, regardless of how many ASNs it contains.

Therefore, `65000 {65001,65002}` has an AS_PATH length of two for the traditional comparison, not three.

See [AS-SET and Attribute Preservation](as-set.md) for additional attribute and loop-prevention details.

### Modern Standards Note: AS_SET Is Deprecated

**RFC 9774, published in May 2025, prohibits the normal origination and advertisement of AS_SET and AS_CONFED_SET segments.** Unless explicitly configured otherwise for a transition, compliant BGP implementations must not advertise these segments and must use treat-as-withdraw handling when receiving them.

This updates RFC 4271 and obsoletes RFC 6472.

Therefore:

- Understand `aggregate-address ... as-set` for Cisco configuration, legacy networks, and CCIE lab preparation.
- Do not assume that AS_SET-bearing aggregates will be accepted by all modern BGP peers.
- Prefer aggregation designs that do not require AS_SET in modern production interdomain networks.

Whether a particular IOS XE release permits generating or accepting AS_SET-bearing routes must be verified on that release and with its configured policy.

---

## Aggregation Attribute Control

Two additional `aggregate-address` options provide control over aggregate attributes.

### `advertise-map`

The `advertise-map` option selects which more-specific routes contribute to the **attributes** of an aggregate, including AS_PATH set information and communities when applicable.

```text
R3(config-router-af)# aggregate-address 192.0.2.0 255.255.254.0 as-set advertise-map SELECT-ROUTES
```

This is useful when specific contributors have attributes that should not be inherited by the aggregate.

**Important:** The `advertise-map` option of `aggregate-address` is not the same feature as `neighbor <address> advertise-map ... exist-map/non-exist-map`, which is used for BGP conditional advertisement.

### `attribute-map`

The `attribute-map` option applies a route map to modify attributes of the aggregate itself.

For example:

```text
R3(config)# route-map SET-AGGREGATE permit 10
R3(config-route-map)# set community none

R3(config)# router bgp 65000
R3(config-router)# address-family ipv4 unicast
R3(config-router-af)# aggregate-address 192.0.2.0 255.255.254.0 as-set summary-only attribute-map SET-AGGREGATE
```

This removes standard BGP communities from the aggregate, including any inherited `no-export` community, which could otherwise prevent advertisement to external peers.

An `attribute-map` can override relevant inherited attributes selected by an `advertise-map`.

See [Advertise-Map and Attribute-Map](advertise-attribute-map.md) for detailed examples.

---

## Aggregation, Forwarding, and Convergence

### Forwarding to the Aggregate

Cisco IOS XE can install a discard route for a locally generated BGP aggregate, typically pointing to `Null0`.

This helps prevent forwarding loops when a packet matches the aggregate but no reachable more-specific route exists.

Suppose R3 has:

```text
192.0.2.0/23     Null0 (aggregate discard route)
192.0.2.0/24     via R1
```

Traffic to `192.0.2.10` follows the more-specific /24 route to R1 because of longest-prefix matching.

Traffic to `192.0.3.10` matches the /23 aggregate but no longer matches a valid more-specific route, so it can be discarded at R3.

**Important:** `summary-only` suppresses advertisements of more-specific routes. It does not remove those routes from the local BGP table or change the basic longest-prefix-match forwarding rule.

### Partial Failure of Contributing Routes

If R3 learns both:

```text
192.0.2.0/24
192.0.3.0/24
```

and then loses `192.0.3.0/24`, the aggregate may remain advertised because `192.0.2.0/24` still exists.

Consequently, R4 can continue sending traffic toward the entire `/23`, even though part of that address range is no longer reachable behind R3.

This is a tradeoff between reducing routing information and preserving precise reachability information.

### AS_SET Churn

When `as-set` is used, changes to contributing routes can change the aggregate's AS_PATH set and other inherited attributes.

For example:

```text
Both /24s present:      65000 {65001,65002}
Only R1's /24 present:  65000 65001
```

The displayed representation can vary depending on the contributing routes and how the implementation builds its path segments.

Even when the aggregate prefix itself remains advertised, a change in its attributes can trigger a new BGP UPDATE.

This is one reason AS_SET-based aggregation can reduce the stability benefits otherwise associated with aggregation.

---

## Aggregation Options at a Glance

| Command or Keyword | Function |
|---|---|
| `aggregate-address <address> <mask>` | Create an aggregate when eligible more-specific BGP routes exist |
| `summary-only` | Suppress advertisement of all contributing more-specific routes |
| `as-set` | Build AS_PATH set information from contributors (legacy/deprecated by RFC 9774) |
| `suppress-map <route-map>` | Selectively suppress contributing routes |
| `neighbor <IP> unsuppress-map <route-map>` | Allow selected suppressed routes to be advertised to one neighbor |
| `advertise-map <route-map>` | Select contributors used to derive aggregate attributes |
| `attribute-map <route-map>` | Modify attributes of the generated aggregate |

---

## Verification

The following commands help verify aggregation and its effects:

```text
R3# show ip bgp
R3# show ip bgp 192.0.2.0/23
R3# show ip bgp 192.0.2.0/24
R3# show ip bgp neighbors 10.0.34.4 advertised-routes
R3# show ip route 192.0.2.0 255.255.254.0
R3# show running-config | section router bgp
```

Examine the output for:

- The presence of the aggregate prefix.
- The contributing more-specific routes.
- The `s` (suppressed) status code.
- The AS_PATH and any AS_SET information.
- The `atomic-aggregate` indication and `aggregated by` information, when present.
- Whether the aggregate and specific routes are advertised to the intended neighbor.
- A `Null0` route associated with the locally generated aggregate, where applicable.

The exact display format can vary by IOS XE version and address family.

---

## Topics Covered

The following pages examine BGP aggregation in greater detail:

1. [BGP Aggregation Configuration](configuration.md)
   - Aggregate origination and requirements.
   - `aggregate-address` versus `network`.
   - Basic aggregation and verification.
2. [AS-SET and Attribute Preservation](as-set.md)
   - AS_SEQUENCE versus AS_SET.
   - AS_PATH loop prevention, attribute inheritance, and standards deprecation.
   - ATOMIC_AGGREGATE and AGGREGATOR.
3. [Summary-Only, Suppress-Map, and Unsuppress-Map](suppression.md)
   - Global and selective suppression.
   - Neighbor-specific unsuppression.
   - Interactions between suppression options.
4. [Advertise-Map and Attribute-Map](advertise-attribute-map.md)
   - Selecting contributing routes for aggregate attribute calculation.
   - Modifying communities and other aggregate attributes.
   - Interaction with `as-set` and `suppress-map`.

---

## References

- [Cisco: Understand Route Aggregation in BGP](https://www.cisco.com/c/en/us/support/docs/ip/border-gateway-protocol-bgp/5441-aggregation.html)
- [Cisco: BGP Command Reference, aggregate-address](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_bgp/command/irg-cr-book/bgp-a1.html)
- [Cisco: BGP Command Reference, neighbor unsuppress-map](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_bgp/command/irg-cr-book/bgp-m1.html)
- [RFC 4271: A Border Gateway Protocol 4 (BGP-4)](https://www.rfc-editor.org/rfc/rfc4271.html)
- [RFC 9774: Deprecation of AS_SET and AS_CONFED_SET in BGP](https://www.rfc-editor.org/rfc/rfc9774.html)
