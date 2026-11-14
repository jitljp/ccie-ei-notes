# AS_PATH

The **AS_PATH** attribute records the autonomous systems that a BGP route has traversed.

It serves three primary purposes:

1. **Loop prevention:** Prevents routes from being accepted by an AS they have already traversed.
2. **Path selection:** BGP normally prefers the shortest AS_PATH when higher-priority selection criteria are equal.
3. **Traffic engineering:** AS_PATH manipulation can influence how remote autonomous systems select routes.

AS_PATH is a **well-known mandatory, transitive BGP path attribute**.

| Property | Value |
|---|---|
| Attribute type code | 2 |
| Attribute category | Well-known mandatory |
| Attribute flags | `0x40` (without extended length) |
| Default path selection | Shortest AS_PATH preferred |
| Modified during EBGP advertisement? | Normally yes |
| Modified during IBGP advertisement? | Normally no |

---

## AS_PATH Operation

When a BGP router advertises a route to an EBGP neighbor, it normally **prepends its local ASN** to the AS_PATH attribute.

When advertising a route to an IBGP neighbor, the AS_PATH is normally preserved.

Consider the following topology:

```text
       AS 65001              AS 65002              AS 65003

          R1 ------------------- R2 ------------------- R3
                 EBGP                   EBGP

     192.0.2.0/24
```

R1 originates `192.0.2.0/24`.

The AS_PATH changes as follows:

| Router | AS_PATH |
|---|---|
| R1 (local BGP table) | Empty |
| R2 | `65001` |
| R3 | `65002 65001` |

### AS_PATH Order

The AS_PATH is displayed with the **most recently traversed AS on the left**.

For example:

```text
65003 65002 65001
```

- `65003` is the most recently added AS.
- `65001` is the origin AS.
- The route has traversed three autonomous systems.

The AS_PATH represents the **AS-level path**, not the number of physical router hops.

Multiple routers within the same AS can forward a route without increasing its AS_PATH length.

### IBGP Advertisement

Consider the following network:

```text
       AS 65001                    AS 65002

          R1 ---------- R2 ---------- R3 ---------- R4
                EBGP          IBGP          IBGP

     192.0.2.0/24
```

R1 originates `192.0.2.0/24`.

| Router | AS_PATH |
|---|---|
| R1 | Empty |
| R2 | `65001` |
| R3 | `65001` |
| R4 | `65001` |

The AS_PATH remains unchanged as the route is advertised through IBGP.

**Important:** Normal IBGP split-horizon rules prevent an IBGP-learned route from being advertised to another IBGP peer. The example assumes route reflection or another valid advertisement mechanism where required.

### Empty AS_PATH

A locally originated route initially has an empty AS_PATH because it has not traversed another AS.

However, **an empty AS_PATH does not necessarily mean the route was locally originated on the receiving router**.

For example, when R1 originates a route and advertises it to an IBGP peer, the receiving router can also see an empty AS_PATH.

Local origination and AS_PATH length are separate criteria in the BGP best path selection algorithm.

---

## AS_PATH Segment Types

The AS_PATH attribute consists of one or more **AS path segments**.

Four segment types are defined:

| Segment Type | Type Code | Description |
|---|---|---|
| AS_SET | 1 | Unordered set of ASNs |
| AS_SEQUENCE | 2 | Ordered sequence of ASNs |
| AS_CONFED_SEQUENCE | 3 | Ordered sequence of confederation member ASNs |
| AS_CONFED_SET | 4 | Unordered set of confederation member ASNs |

The first two are defined by RFC 4271. Confederation segment types are defined by RFC 5065.

### AS_SEQUENCE

**AS_SEQUENCE** is an ordered list of autonomous systems that a route has traversed.

For example:

```text
65003 65002 65001
```

This indicates that the route originated in AS 65001 and subsequently traversed AS 65002 and AS 65003.

AS_SEQUENCE is the normal segment type used for EBGP route advertisement.

Each ASN in an AS_SEQUENCE contributes **1** to the AS_PATH length.

### AS_SET

**AS_SET** is an unordered collection of autonomous systems.

It is primarily associated with BGP route aggregation using the `as-set` option.

For example:

```text
{65001,65002,65003}
```

The set indicates that the aggregate contains routing information associated with these autonomous systems.

However, it does not preserve their traversal order.

The most important characteristics are:

- AS_SET is unordered.
- It is commonly generated during route aggregation.
- It helps retain AS-level loop-prevention information.
- **The entire AS_SET counts as 1 when calculating AS_PATH length**, regardless of how many ASNs it contains.

For example:

```text
65010 {65001,65002,65003}
```

The effective AS_PATH length is:

```text
AS_SEQUENCE: 65010                 = 1
AS_SET:      {65001,65002,65003}   = 1
                                    ---
Total AS_PATH length                 2
```

If the local ASN appears in an AS_SET, the route is still subject to AS_PATH loop prevention.

See [AS-SET and Attribute Preservation](../../scalability/aggregation/as-set.md).

### Confederation Segments

BGP confederations divide an autonomous system into multiple member ASNs while presenting a single ASN to external peers.

Two additional AS_PATH segment types are used:

- **AS_CONFED_SEQUENCE:** An ordered sequence of member ASNs.
- **AS_CONFED_SET:** An unordered set of member ASNs.

For example, a router might display:

```text
(65002 65001) 65100
```

The parentheses represent an AS_CONFED_SEQUENCE.

The confederation member ASNs are used for routing within the confederation.

When routes are advertised outside the confederation, the confederation segments are removed, and the external confederation identifier is used.

**Confederation segments are not included in AS_PATH length during Cisco BGP best path selection.**

---

## AS_PATH Length Calculation

By default, BGP prefers the **shortest AS_PATH**, provided earlier selection criteria have not already determined the best path.

Cisco calculates AS_PATH length using these rules:

| Segment Type | Contribution to AS_PATH Length |
|---|---|
| AS_SEQUENCE | 1 per ASN |
| AS_SET | 1 per segment |
| AS_CONFED_SEQUENCE | 0 |
| AS_CONFED_SET | 0 |

### Example 1: AS_SEQUENCE

```text
Path A: 65001 65002 65003
Path B: 65004 65005
```

| Path | Length |
|---|---|
| A | 3 |
| B | 2 |

**Path B is preferred** if the comparison reaches AS_PATH length.

### Example 2: AS_SET

```text
Path A: 65001 {65002,65003,65004}
Path B: 65005 65006 65007
```

| Path | Length |
|---|---|
| A | 2 |
| B | 3 |

Path A is preferred because the entire AS_SET contributes only 1.

### Example 3: Confederation Segments

```text
Path A: (65001 65002 65003) 65100
Path B: 65200 65300
```

| Path | Length |
|---|---|
| A | 1 |
| B | 2 |

Path A is preferred because its confederation member ASNs are excluded from the length calculation.

### Example 4: Repeated ASNs

```text
Path A: 65001 65001 65001
Path B: 65002 65003
```

| Path | Length |
|---|---|
| A | 3 |
| B | 2 |

Repeated ASNs in an AS_SEQUENCE are counted individually.

This is the principle behind **AS_PATH prepending**.

---

## AS_PATH in Best Path Selection

AS_PATH length is evaluated after several other criteria.

The relevant default Cisco BGP selection order is:

1. Highest Weight.
2. Highest LOCAL_PREF.
3. Locally originated path.
4. **Shortest AS_PATH**.
5. Lowest ORIGIN type.
6. Lowest MED, when applicable.
7. EBGP over IBGP.
8. Lowest IGP metric to NEXT_HOP.

Additional comparisons are performed when necessary.

### Example

R1 receives two paths to `192.0.2.0/24`:

| Attribute | Path A | Path B |
|---|---|---|
| Weight | 0 | 0 |
| LOCAL_PREF | 100 | 100 |
| Locally originated? | No | No |
| AS_PATH | `65001 65002 65003` | `65004 65005` |
| ORIGIN | IGP | IGP |
| MED | 0 | 100 |

Assuming both paths are valid and eligible, R1 selects **Path B**.

Although Path B has a higher MED, its AS_PATH is shorter.

BGP does not compare MED because AS_PATH length already determines the best path.

**Important:** BGP does not automatically prefer the shortest AS_PATH if a higher-priority criterion differs.

See [BGP Best Path Selection Steps](../best-path/selection-steps.md).

---

## AS_PATH Loop Prevention

AS_PATH is BGP's primary mechanism for preventing **inter-AS routing loops**.

By default, a BGP router rejects an EBGP-learned route if its own ASN appears anywhere in the AS_PATH.

Consider the following topology:

```text
           AS 65001
              R1
             /  \
            /    \
           R2----R3
        AS 65002 AS 65003
```

R1 originates `192.0.2.0/24`.

R1 advertises the prefix to R2:

```text
AS_PATH: 65001
```

R2 advertises the route to R3:

```text
AS_PATH: 65002 65001
```

If R3 attempts to advertise the route back to R1, the AS_PATH becomes:

```text
65003 65002 65001
```

R1 sees its own ASN, `65001`, in the received AS_PATH.

**R1 rejects the route to prevent an AS-level routing loop.**

The local ASN does not need to appear at the beginning of the AS_PATH to trigger this check.

### Disabling AS_PATH Loop Prevention

Cisco supports `neighbor allowas-in`, which permits a router to accept routes containing its own ASN, subject to a configured occurrence limit.

For example:

```text
R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 allowas-in 2
```

This allows routes containing the local ASN up to two times.

This is useful in specialized scenarios involving customer networks or an AS that appears at multiple locations.

However, it weakens normal AS_PATH loop prevention and must be used carefully.

See [Allowas-in](../../as-path/as-number-manipulation/allowas-in.md).

### AS_PATH vs. IBGP Loop Prevention

AS_PATH is not sufficient to prevent all routing loops within an AS because IBGP normally does not prepend the local ASN.

IBGP therefore uses other mechanisms:

- IBGP split-horizon rules.
- Route reflector ORIGINATOR_ID.
- Route reflector CLUSTER_LIST.

These mechanisms complement AS_PATH loop prevention.

---

## Enforcing the First AS

Cisco IOS XE normally requires an EBGP route's AS_PATH to begin with the ASN of the neighbor that advertised it.

This behavior is enabled by default.

For example, R1 has an EBGP neighbor in AS 65002:

```text
R1(config)# router bgp 65001
R1(config-router)# neighbor 10.0.12.2 remote-as 65002
```

The following AS_PATH is valid:

```text
65002 65003 65004
```

However, the following AS_PATH violates the first-AS requirement:

```text
65003 65004
```

The first ASN is `65003`, but the advertising EBGP neighbor belongs to AS 65002.

### Configuration

To explicitly enable the default behavior:

```text
R1(config-router)# bgp enforce-first-as
```

To disable it:

```text
R1(config-router)# no bgp enforce-first-as
```

Disabling this check can be necessary when connecting to an **EBGP route server** that forwards routes without prepending its own ASN.

A first-AS violation can cause the UPDATE to be rejected and the BGP session to be reset.

### First-AS Check vs. AS Loop Detection

These are separate mechanisms.

| Mechanism | Checks |
|---|---|
| AS_PATH loop detection | Whether the local ASN appears in the AS_PATH |
| First-AS enforcement | Whether the first ASN matches the advertising EBGP neighbor's ASN |

Disabling first-AS enforcement does **not** disable normal AS_PATH loop detection.

Likewise, `allowas-in` does not automatically disable first-AS enforcement.

---

## Ignoring AS_PATH Length

Cisco IOS XE supports the following command:

```text
R1(config)# router bgp 65001
R1(config-router)# bgp bestpath as-path ignore
```

This instructs BGP to **skip the AS_PATH length comparison** during best path selection.

For example:

| Attribute | Path A | Path B |
|---|---|---|
| Weight | 0 | 0 |
| LOCAL_PREF | 100 | 100 |
| AS_PATH | `65002 65003 65004` | `65005` |
| ORIGIN | IGP | Incomplete |

Normally, Path B is preferred because it has a shorter AS_PATH.

With `bgp bestpath as-path ignore`, BGP skips AS_PATH length and continues to ORIGIN.

Therefore, **Path A is preferred** because its ORIGIN type is IGP.

### Important Distinction

`bgp bestpath as-path ignore` does not:

- Remove AS_PATH from UPDATE messages.
- Stop normal ASN prepending.
- Disable AS_PATH loop detection.
- Disable AS_PATH filtering.
- Make all routes equally eligible for installation.

It only disables AS_PATH length as a best-path selection criterion.

Restore normal behavior:

```text
R1(config-router)# no bgp bestpath as-path ignore
```

---

## AS_PATH and BGP Multipath

BGP multipath allows multiple eligible paths to be installed for load sharing.

By default, multipath eligibility requires compatible AS_PATH information in addition to other best-path attributes.

For example:

```text
Path A: 65002 65004
Path B: 65003 65004
```

Both paths have an AS_PATH length of 2, but their neighboring ASNs differ.

Even though their lengths are equal, normal EBGP multipath restrictions can prevent the paths from qualifying together.

### Relaxing AS_PATH Multipath Requirements

On supported IOS XE platforms, use:

```text
R1(config)# router bgp 65001
R1(config-router)# bgp bestpath as-path multipath-relax
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# maximum-paths 2
```

The `multipath-relax` command relaxes AS_PATH matching requirements so that paths through different autonomous systems can qualify for multipath when their AS_PATH lengths are equal.

Other multipath eligibility requirements must still be satisfied, including compatible Weight, LOCAL_PREF, ORIGIN, MED, and next-hop conditions.

**Important:** This command affects multipath eligibility, not the ordinary preference for the shortest AS_PATH during best-path selection.

BGP still identifies one best path, even when multiple paths qualify for installation.

See [BGP Load Balancing](../load-balancing.md).

---

## Limiting AS_PATH Length

Cisco IOS XE supports `bgp maxas-limit` to reject routes containing excessively long AS_PATH information.

For example:

```text
R1(config)# router bgp 65001
R1(config-router)# bgp maxas-limit 10
```

This configures a limit of **10 ASNs** in received AS_PATH information.

Routes exceeding the configured limit are discarded.

The limit is inclusive, so a route containing exactly 10 ASNs does not exceed the limit.

The command can help protect BGP from unusually long or malformed AS_PATH information.

The command reference documents a configurable range of 1-254 and additional restrictions on AS_PATH segment counts, depending on software support.

Restore the default behavior:

```text
R1(config-router)# no bgp maxas-limit
```

**Important:** Unlike `bgp bestpath as-path ignore`, which changes how eligible paths are compared, `bgp maxas-limit` can reject paths before best-path selection.

---

## AS_PATH and 4-Byte ASNs

Originally, AS_PATH used 2-byte ASNs.

RFC 6793 extended BGP to support **4-byte ASNs**.

When both BGP peers support 4-byte ASNs, the existing AS_PATH attribute carries the full 4-byte ASN values.

For example:

```text
100000 65001
```

The path includes two autonomous systems, even though `100000` cannot be represented as a 2-byte ASN.

### AS_TRANS

AS_TRANS is the reserved ASN **23456**, used for compatibility with legacy BGP speakers.

When a router supporting 4-byte ASNs communicates with an older router that supports only 2-byte ASNs, non-mappable ASNs are represented by AS_TRANS in the 2-byte AS_PATH.

For example:

```text
Original AS_PATH:

100000 65001
```

The legacy-compatible AS_PATH can contain:

```text
23456 65001
```

The actual 4-byte ASN information is preserved separately using the **AS4_PATH** attribute.

| Attribute | Type Code | Category |
|---|---|---|
| AS_PATH | 2 | Well-known mandatory |
| AS4_PATH | 17 | Optional transitive |

A 4-byte-capable BGP router can use the AS_PATH and AS4_PATH information to reconstruct the original path, subject to the reconstruction rules in RFC 6793.

**AS4_PATH is not normally exchanged between two peers that both support 4-byte ASNs.**

See [4-Byte AS Numbers](../../peering/four-byte-asn.md).

---

## AS_PATH Manipulation

AS_PATH can be modified to influence how routes are selected by other autonomous systems.

The primary techniques include:

| Technique | Purpose |
|---|---|
| AS_PATH prepending | Artificially lengthens the AS_PATH to make a route less preferred |
| Local-AS | Presents an alternate ASN to specific EBGP neighbors |
| Allowas-in | Allows received routes containing the local ASN |
| AS override | Replaces occurrences of a neighbor's ASN in advertised paths |
| Remove-private-as | Removes private ASNs from outbound advertisements |
| AS_PATH filtering | Matches AS_PATH contents using regular expressions |

### AS_PATH Prepending

For example, suppose AS 65001 advertises a prefix through two providers:

```text
             ISP-A             ISP-B
            AS 65002          AS 65003
               |                 |
               |                 |
               +------ R1 -------+
                     AS 65001

                  192.0.2.0/24
```

R1 normally advertises:

```text
AS_PATH: 65001
```

To make ISP-B less attractive to external networks, R1 can prepend additional copies of its own ASN when advertising through ISP-B.

For example:

```text
Via ISP-A: 65001
Via ISP-B: 65001 65001 65001
```

Remote networks may prefer the path through ISP-A because it has a shorter AS_PATH.

However, this is not guaranteed. Higher-priority attributes, such as LOCAL_PREF in the remote AS, can override the AS_PATH length difference.

Detailed configuration is covered in:

- [AS Path Manipulations](../../as-path/overview.md)
- [AS Path Prepending](../../as-path/prepending.md)
- [Local-AS](../../as-path/as-number-manipulation/local-as.md)
- [Remove-Private-AS](../../as-path/as-number-manipulation/remove-private-as.md)
- [BGP AS Path Regular Expressions](../../as-path/regular-expressions.md)

---

## AS_PATH Filtering

BGP can match routes based on their AS_PATH using **AS-path access lists** and regular expressions.

For example:

```text
R1(config)# ip as-path access-list 10 permit ^65002$
```

This matches paths containing exactly one ASN, `65002`.

Other common expressions include:

| Expression | Meaning |
|---|---|
| `^$` | Empty AS_PATH |
| `^65002$` | AS_PATH contains only 65002 |
| `^65002_` | AS_PATH begins with 65002 |
| `_65002$` | AS_PATH ends with 65002 |
| `_65002_` | AS_PATH contains 65002 |
| `.*` | Any AS_PATH |

The underscore (`_`) is a special delimiter in Cisco BGP AS-path regular expressions.

To use an AS-path access list for inbound filtering:

```text
R1(config)# ip as-path access-list 10 permit ^65002$

R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 filter-list 10 in
```

Only routes matching the AS-path access list are permitted by this filter.

Alternatively, AS-path access lists can be referenced by route maps using `match as-path` or `match ip as-path`, depending on the supported command syntax.

See [BGP AS Path Regular Expressions](../../as-path/regular-expressions.md) for detailed matching behavior.

---

## Verification

### Display AS_PATH

```text
R1# show bgp ipv4 unicast
R1# show bgp ipv4 unicast 192.0.2.0/24
R1# show bgp ipv6 unicast
```

Example output:

```text
R1# show bgp ipv4 unicast

   Network          Next Hop            Metric LocPrf Weight Path
*  192.0.2.0/24     10.0.12.2                0             0 65002 65004 i
*>                  10.0.13.3                0             0 65003 i
```

In this example:

- Path A has AS_PATH length 2.
- Path B has AS_PATH length 1.
- Path B is selected as best.

The trailing `i` is the **ORIGIN code**, not part of AS_PATH and not an indication that the path was learned through IBGP.

### Detailed Path Information

```text
R1# show bgp ipv4 unicast 192.0.2.0/24
```

The detailed output identifies each path's AS_SEQUENCE and other attributes, along with validity and best-path status.

### Display Routes Matching an AS_PATH

```text
R1# show bgp ipv4 unicast regexp ^65002_
```

This displays BGP routes whose AS_PATH matches the specified regular expression.

### Display Advertised Routes

```text
R1# show bgp ipv4 unicast neighbors 10.0.12.2 advertised-routes
```

This is useful when verifying AS_PATH prepending and outbound advertisements.

### Verify AS_PATH Filtering

```text
R1# show ip as-path-access-list
R1# show running-config | section router bgp
R1# show route-map
```

### Useful Troubleshooting Commands

| Command | Purpose |
|---|---|
| `show bgp ipv4 unicast` | Displays BGP routes and AS_PATH information |
| `show bgp ipv4 unicast <prefix>` | Displays detailed attributes for a prefix |
| `show bgp ipv4 unicast regexp <regex>` | Filters BGP routes by AS_PATH |
| `show ip as-path-access-list` | Displays AS-path access lists |
| `show bgp ipv4 unicast neighbors <peer>` | Displays neighbor information |
| `show bgp ipv4 unicast neighbors <peer> advertised-routes` | Displays routes advertised to a neighbor |
| `show running-config \| section router bgp` | Displays BGP configuration |

---

## Key Takeaways

- AS_PATH is a **well-known mandatory** BGP path attribute.
- EBGP normally prepends the local ASN; IBGP normally preserves AS_PATH.
- AS_PATH is displayed with the most recently added ASN on the left.
- **AS_SEQUENCE:** Ordered list; each ASN counts as 1.
- **AS_SET:** Unordered set; the entire segment counts as 1.
- **AS_CONFED_SEQUENCE / AS_CONFED_SET:** Excluded from normal AS_PATH length calculations.
- BGP normally prefers the **shortest AS_PATH**, after higher-priority criteria.
- Routes containing the local ASN are normally rejected through AS_PATH loop prevention.
- `bgp enforce-first-as` checks that the first ASN matches the advertising EBGP neighbor.
- `bgp bestpath as-path ignore` disables AS_PATH length comparison, not loop prevention.
- `bgp bestpath as-path multipath-relax` can allow multipath across different AS paths of equal length.
- `bgp maxas-limit` discards routes whose AS_PATH exceeds the configured limit.
- AS_PATH prepending can influence inbound traffic selection by remote autonomous systems, but does not guarantee a particular path.

---

## References

- [Cisco: Select BGP Best-path Algorithm](https://www.cisco.com/c/en/us/support/docs/ip/border-gateway-protocol-bgp/13753-25.html)
- [Cisco: BGP Configuration Guide, Understanding BGP](https://www.cisco.com/c/en/us/td/docs/switches/lan/c9000/lyr3-fwd/bgp/bgp-configuration-guide/routing-bgp.html)
- [Cisco: BGP Command Reference, Commands A through B](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_bgp/command/irg-cr-book/bgp-a1.html)
- [RFC 4271: A Border Gateway Protocol 4 (BGP-4)](https://www.rfc-editor.org/rfc/rfc4271.html)
- [RFC 5065: Autonomous System Confederations for BGP](https://www.rfc-editor.org/rfc/rfc5065.html)
- [RFC 6793: BGP Support for Four-Octet Autonomous System Number Space](https://www.rfc-editor.org/rfc/rfc6793.html)
