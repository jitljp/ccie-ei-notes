# 1.5.d (iii) Regular Expressions

**BGP regular expressions** allow administrators to match routes based on the contents of their AS_PATH attributes.

Unlike prefix lists, which match destination network prefixes, BGP regular expressions examine the **sequence of Autonomous System Numbers (ASNs)** a route has traversed.

Regular expressions are commonly used to:

- Accept or reject routes originating from particular autonomous systems.
- Filter routes that have traversed specified autonomous systems.
- Match routes received directly from particular ASes.
- Match locally originated routes with empty AS_PATHs.
- Identify routes containing AS_PATH prepending.
- Apply routing policies based on AS_PATH contents.

On Cisco IOS XE, regular expressions are primarily used with:

- `ip as-path access-list`
- `neighbor filter-list`
- `match as-path` in route maps
- `show ip bgp regexp`

Regular expressions do not modify the AS_PATH themselves. They identify routes that can then be filtered or processed by routing policies.

---

## AS_PATH and Regular Expressions

The AS_PATH attribute records the autonomous systems a BGP route has traversed.

For example:

```text
65003 65002 65001
```

This indicates that the route originated in AS 65001, subsequently traversed AS 65002, and was most recently advertised by AS 65003.

The ASNs appear in reverse traversal order:

- **Leftmost ASN:** Most recently traversed AS.
- **Rightmost ASN:** Originating AS, for a normal AS_SEQUENCE path.

Regular expressions match the textual representation of this AS_PATH.

For example:

```text
^65003
```

Matches paths beginning with the characters `65003`.

However, because it does not specify an ASN boundary after `65003`, it can also match longer ASN values beginning with those digits.

A more precise expression is:

```text
^65003_
```

This matches AS_PATHs whose leftmost complete ASN is 65003.

Similarly:

```text
_65001$
```

Matches AS_PATHs whose rightmost complete ASN is 65001.

And:

```text
_65002_
```

Matches AS_PATHs containing AS 65002 anywhere.

**Important:** BGP regular expressions operate on characters, not directly on ASN objects. Correctly matching complete ASN values requires understanding regular-expression boundaries.

---

## Regular Expression Characters

Cisco IOS XE supports several special characters for AS_PATH regular-expression matching.

### Basic Characters

| Character | Meaning |
|---|---|
| `^` | Beginning of the string |
| `$` | End of the string |
| `.` | Any single character |
| `*` | Zero or more occurrences of the preceding expression |
| `+` | One or more occurrences of the preceding expression |
| `?` | Zero or one occurrence of the preceding expression |
| `[]` | Match one character from a set or range |
| `()` | Group expressions |
| `\|` | Alternation (OR) |
| `_` | Cisco-specific AS_PATH delimiter |
| `\` | Escape a special character |
| `\1`, `\2` | Backreference to a captured group |

These characters can be combined to construct complex matching patterns.

### Beginning of String: ^

The caret (`^`) anchors a pattern to the beginning of the AS_PATH.

For example:

```text
^65001
```

Matches AS_PATH strings beginning with `65001`.

Examples:

```text
65001
65001 65002
65001 65002 65003
```

However, it can also match:

```text
650010 65002
```

Because the expression only checks the initial characters.

To match AS 65001 as a complete ASN, use:

```text
^65001_
```

### End of String: $

The dollar sign (`$`) anchors a pattern to the end of the AS_PATH.

For example:

```text
65001$
```

Matches strings ending with the digits `65001`.

Examples:

```text
65001
65002 65001
65003 65002 65001
```

However, it can also match:

```text
165001
```

To match AS 65001 as the complete final ASN, use:

```text
_65001$
```

### Beginning and End Together

Combining both anchors requires the entire string to match the expression.

For example:

```text
^65001$
```

Matches:

```text
65001
```

But does not match:

```text
65001 65002
65002 65001
65001 65001
```

This expression is commonly used to match routes whose complete AS_PATH consists of exactly one occurrence of AS 65001.

---

## The Underscore: _

The underscore (`_`) is especially important in Cisco BGP regular expressions.

It matches an AS_PATH boundary or delimiter.

Cisco documents that `_` can match:

- Beginning of the string.
- End of the string.
- Space.
- Comma (`,`).
- Opening or closing brace (`{` or `}`).
- Opening or closing parenthesis (`(` or `)`).

These delimiters are used in the textual representation of different AS_PATH segment types.

### Matching an ASN Anywhere

Consider:

```text
_65001_
```

This matches AS 65001 as a complete ASN anywhere in the AS_PATH.

Examples:

| AS_PATH | Match? |
|---|---|
| `65001` | Yes |
| `65001 65002` | Yes |
| `65002 65001` | Yes |
| `65002 65001 65003` | Yes |
| `165001` | No |
| `650010` | No |
| `65001 650010` | Yes |

In the last example, the expression matches the first ASN, `65001`, but not `650010`.

### Why Not Just Match 65001?

Consider:

```text
65001
```

Without anchors or delimiters, this expression matches the character sequence `65001` anywhere.

Therefore, it can match:

```text
65001
165001
650010
65002 65001
```

This can produce unintended results when filtering routes.

**Use underscores when matching complete ASN values rather than substrings of larger ASN values.**

### Underscore vs. Space

A literal space matches only a space.

The underscore matches several AS_PATH delimiters.

For example, AS_SET segments are displayed using braces:

```text
65010 {65001,65002}
```

The expression:

```text
_65001_
```

Can match AS 65001 within the AS_SET because the opening brace and comma provide ASN boundaries.

However, an expression using literal spaces may fail to match the same ASN within an AS_SET.

See [AS_PATH](../path-selection/attributes/as-path.md) for information about AS_SEQUENCE, AS_SET, and confederation segments.

---

## Common BGP Regular Expressions

The following expressions are particularly useful for CCIE-level BGP configuration.

| Expression | Matches |
|---|---|
| `.*` | Any AS_PATH, including empty |
| `^$` | Empty AS_PATH |
| `^65001$` | AS_PATH containing exactly one ASN, 65001 |
| `^65001_` | Leftmost ASN is 65001 |
| `_65001$` | Rightmost ASN is 65001 |
| `_65001_` | AS 65001 appears anywhere |
| `^65001_65002$` | Exactly `65001 65002` |
| `^65001_65002_` | AS_PATH begins with `65001 65002` |
| `_65001_65002$` | AS_PATH ends with `65001 65002` |
| `^[0-9]+$` | AS_PATH containing exactly one numeric ASN |
| `^[0-9]+_[0-9]+$` | AS_PATH containing exactly two numeric ASNs |
| `^65001(_65001)*$` | AS_PATH containing only AS 65001, potentially repeated |

These examples assume ordinary AS_SEQUENCE paths unless otherwise specified.

---

## Matching Empty AS_PATHs

The following regular expression matches an empty AS_PATH:

```text
^$
```

There are no characters between the beginning (`^`) and end (`$`) anchors.

### When Is AS_PATH Empty?

Locally originated BGP routes initially have empty AS_PATHs because they have not traversed another autonomous system.

For example:

```text
R1(config)# router bgp 65001
R1(config-router)# network 10.10.10.0 mask 255.255.255.0
```

Assuming an exact matching route exists in the routing table, R1 originates `10.10.10.0/24`.

Its local BGP table contains an empty AS_PATH for the locally originated path.

However, an empty AS_PATH does **not necessarily mean the route was locally originated by the router inspecting it**.

An IBGP neighbor can also advertise a route with an empty AS_PATH, because IBGP normally does not prepend the local ASN.

### Configuration Example

```text
R1(config)# ip as-path access-list 10 permit ^$
```

This matches BGP paths with empty AS_PATHs.

**Important:** `^$` matches an empty AS_PATH, not a specific prefix or a specific BGP ORIGIN attribute.

---

## Matching Routes from a Specific Origin AS

To match routes whose AS_PATH ends with AS 65001, use:

```text
_65001$
```

For ordinary AS_SEQUENCE paths, the rightmost ASN normally identifies the originating AS.

### Examples

| AS_PATH | Match? |
|---|---|
| `65001` | Yes |
| `65002 65001` | Yes |
| `65003 65002 65001` | Yes |
| `65001 65002` | No |
| `65002 65001 65003` | No |
| `165001` | No |

### Configuration

```text
R1(config)# ip as-path access-list 10 permit _65001$
```

This matches paths ending in AS 65001 regardless of how many ASes precede it.

### Difference Between ^65001$ and _65001$

These expressions have different meanings.

```text
^65001$
```

Matches only:

```text
65001
```

Whereas:

```text
_65001$
```

Matches:

```text
65001
65002 65001
65003 65002 65001
```

**Important:** If the objective is to match routes originating from AS 65001 regardless of their transit ASes, use `_65001$`, not `^65001$`.

### AS_PATH Prepending Consideration

A route originating in AS 65001 may have a prepended AS_PATH:

```text
65001 65001 65001
```

The expression:

```text
_65001$
```

Still matches because the rightmost ASN is 65001.

However:

```text
^65001$
```

Does not match because the complete AS_PATH contains multiple ASN occurrences.

---

## Matching Routes Received from a Specific AS

To match routes whose leftmost ASN is 65001, use:

```text
^65001_
```

For ordinary EBGP routes, the leftmost ASN normally identifies the neighboring AS that most recently advertised the route.

### Examples

| AS_PATH | Match? |
|---|---|
| `65001` | Yes |
| `65001 65002` | Yes |
| `65001 65003 65004` | Yes |
| `65002 65001` | No |
| `65003 65001 65002` | No |
| `165001 65002` | No |

### Configuration

```text
R1(config)# ip as-path access-list 10 permit ^65001_
```

This matches routes whose AS_PATH begins with AS 65001.

**Important:** The leftmost ASN does not necessarily identify the directly connected neighbor in every BGP design.

Exceptions include:

- IBGP, which does not normally prepend ASNs.
- Confederations.
- Route servers that do not prepend their ASN.
- Local-AS and AS_PATH manipulation features.

Therefore, the expression matches the leftmost ASN, not the neighbor IP address.

---

## Matching an ASN Anywhere

To match routes containing AS 65001 anywhere in the AS_PATH, use:

```text
_65001_
```

### Examples

| AS_PATH | Match? |
|---|---|
| `65001` | Yes |
| `65001 65002` | Yes |
| `65002 65001` | Yes |
| `65002 65001 65003` | Yes |
| `65002 65003` | No |

This is useful when:

- Preventing routes that traverse a particular AS.
- Applying special policies to routes passing through an AS.
- Identifying routes containing a particular private ASN.
- Troubleshooting unexpected AS_PATH advertisements.

For example:

```text
R1(config)# ip as-path access-list 10 deny _65001_
R1(config)# ip as-path access-list 10 permit .*
```

This AS-path access list rejects any route containing AS 65001 and permits all other paths.

---

## Regular Expression Quantifiers

Regular expressions support operators that control how many times a preceding character or expression can occur.

### Asterisk: *

The asterisk means **zero or more occurrences** of the preceding expression.

For example:

```text
65*
```

Matches a `6` followed by zero or more occurrences of the character `5`.

Examples:

```text
6
65
655
6555
```

It does not mean zero or more occurrences of ASN 65.

This distinction is important because regular expressions operate on characters.

### Plus: +

The plus sign means **one or more occurrences** of the preceding expression.

For example:

```text
[0-9]+
```

Matches one or more numeric digits.

Examples:

```text
1
100
65001
4200000001
```

### Question Mark: ?

The question mark means **zero or one occurrence** of the preceding character or grouped expression.

For example:

```text
^65001(_65002)?$
```

Matches:

```text
65001
65001 65002
```

Does not match:

```text
65001 65002 65002
65001 65003
65001 65002 65003
```

The expression works as follows:

- `^65001`: The AS_PATH must begin with AS 65001.
- `(_65002)?`: Optionally followed by one occurrence of AS 65002.
- `$`: The AS_PATH must end here.

The `^` and `$` anchors ensure that no additional ASNs are permitted.

Without these anchors, the expression could match a substring of a longer AS_PATH.

When using quantifiers, parentheses group characters so the quantifier applies to the entire group.

---

## Character Classes

Square brackets (`[]`) match a single character from a specified set or range.

For example:

```text
[0-9]
```

Matches one numeric digit.

```text
[0-9]+
```

Matches one or more numeric digits.

### Match an AS_PATH with Exactly One ASN

```text
^[0-9]+$
```

This matches:

```text
100
65001
4200000001
```

But does not match:

```text
100 200
65001 65002
```

The anchors require the entire AS_PATH to consist of one or more digits.

### Match an AS_PATH with Exactly Two ASNs

```text
^[0-9]+_[0-9]+$
```

Examples:

| AS_PATH | Match? |
|---|---|
| `100 200` | Yes |
| `65001 65002` | Yes |
| `100` | No |
| `100 200 300` | No |

The two `[0-9]+` expressions match the two ASNs.

The underscore matches the delimiter between them.

**Important:** These numeric patterns match the displayed AS_PATH representation. They are not replacements for the actual BGP AS_PATH length calculation, particularly with AS_SET or confederation segments.

---

## Grouping and Alternation

Parentheses can group multiple characters or expressions.

The alternation operator (`|`) means OR.

### Match Multiple Origin ASes

Suppose R1 should accept routes originating from either AS 65001 or AS 65002.

Use:

```text
_(65001|65002)$
```

This matches:

```text
65001
65002
65003 65001
65004 65002
```

But not:

```text
65001 65003
65002 65004
```

### Match Multiple Neighbor ASes

To match routes whose leftmost ASN is either 65001 or 65002:

```text
^(65001|65002)_
```

Examples:

```text
65001 65003
65002 65004
65001
65002
```

This is more concise than configuring separate expressions.

### Alternative: Multiple AS-Path ACL Entries

The same matching logic can be implemented with multiple permit entries.

```text
R1(config)# ip as-path access-list 10 permit ^65001_
R1(config)# ip as-path access-list 10 permit ^65002_
```

The access list permits a route if either expression matches.

---

## Backreferences

A **backreference** matches the exact text previously matched by a parenthesized **capturing group**.

Backreferences are particularly useful for identifying **AS_PATH prepending without knowing the ASN in advance**. Unlike a repeated `[0-9]+` expression, a backreference requires the repeated ASN to have the *same value*.

| Syntax | Meaning |
|---|---|
| `(expression)` | Capture the text matched by the expression |
| `\1` | Match the exact text captured by the first parenthesized group |
| `\2` | Match the exact text captured by the second parenthesized group |

Backreference numbers refer to **capturing groups in left-to-right order**, not to the number of times an ASN occurs.

### Basic Example: Match the Same ASN Twice

```text
^([0-9]+)_\1$
```

This expression matches an AS_PATH consisting of **exactly two identical ASNs**.

| Component | Meaning |
|---|---|
| `^` | Beginning of the AS_PATH |
| `([0-9]+)` | Capture the first ASN as group 1 |
| `_` | AS_PATH delimiter |
| `\1` | Repeat exactly the ASN captured by group 1 |
| `$` | End of the AS_PATH |

Matches:

```text
65001 65001
100 100
4200000001 4200000001
```

Does not match:

```text
65001
65001 65002
65001 65001 65001
65001 65002 65001
```

The expression `^[0-9]+_[0-9]+$` would also match `65001 65002`, because each `[0-9]+` independently matches any ASN. A backreference ensures the two values are identical.

### Match One ASN Repeated Any Number of Times

```text
^([0-9]+)(_\1)*$
```

This matches an AS_PATH consisting entirely of **one ASN repeated one or more times**.

- `([0-9]+)` captures the first ASN.
- `(_\1)*` allows zero or more additional occurrences of the same ASN.
- The anchors require the entire AS_PATH to have this structure.

Matches:

```text
65001
65001 65001
65001 65001 65001
100 100 100 100
```

Does not match:

```text
65001 65002
100 100 200
65001 65002 65001
```

To require **at least two** occurrences, change `*` to `+`:

```text
^([0-9]+)(_\1)+$
```

This no longer matches a single ASN such as `65001`.

### Match Consecutive Repeated ASNs Anywhere

To identify a repeated ASN anywhere in a longer AS_PATH, use:

```text
_([0-9]+)(_\1)+_
```

For example, it matches the consecutive repeated ASNs in these AS_PATHs:

```text
65001 65001
65002 65003 65003
65004 65004 65005
65006 65007 65007 65007 65008
```

It does not match:

```text
65001 65002
65001 65002 65001
65001 65002 65003
```

The final `_` ensures an ASN boundary after the last matched occurrence, avoiding unintended substring matches with longer ASN values.

Cisco training examples also use the shorter expression `_([0-9]+)(_\1)+` to identify repeated ASNs. Both are useful, but including the final ASN boundary makes the match more precise.

### Configuration Example: Reject Paths Containing Repeated ASNs

An AS-path access list can reject routes containing consecutive occurrences of the same ASN, regardless of which ASN is repeated:

```text
R1(config)# ip as-path access-list 10 deny _([0-9]+)(_\1)+_
R1(config)# ip as-path access-list 10 permit .*

R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 filter-list 10 in
```

The first entry rejects paths matching the backreference expression. The second permits other paths.

This is useful as a lab exercise, but be careful using it in production: **AS_PATH prepending is normal BGP behavior**, and rejecting every prepended path may discard otherwise valid routes.

### Multiple Capturing Groups

Backreferences can refer to different groups:

```text
^([0-9]+)_([0-9]+)_\2$
```

Here:

- `\1` would refer to the first ASN.
- `\2` refers to the second ASN.
- The expression requires exactly three ASNs, with the **second and third identical**.

Matches:

```text
65001 65002 65002
100 200 200
```

Does not match:

```text
65001 65002 65001
65001 65002 65003
```

**Important:** `\1` is a backreference to a captured value. It is not the same as `\` used to escape a special character, such as `\.` for a literal period in an asdot ASN.

---

## Matching AS_PATH Prepending

Regular expressions can identify AS_PATHs containing repeated ASN occurrences.

For example:

```text
65001 65001 65001
```

This is a common result of AS_PATH prepending.

### Match Consecutive ASN Occurrences

To match paths beginning with two consecutive occurrences of AS 65001:

```text
^65001_65001_
```

This matches:

```text
65001 65001
65001 65001 65002
65001 65001 65001 65003
```

But not:

```text
65001 65002 65001
```

The first two ASNs must both be 65001.

### Match Only Repeated AS 65001

Consider:

```text
^65001(_65001)*$
```

Breaking down the expression:

| Component | Meaning |
|---|---|
| `^65001` | Start with AS 65001 |
| `(_65001)` | A delimiter followed by AS 65001 |
| `*` | Repeat the grouped expression zero or more times |
| `$` | End of AS_PATH |

This matches:

```text
65001
65001 65001
65001 65001 65001
65001 65001 65001 65001
```

But not:

```text
65001 65002
65002 65001
65001 65001 65002
```

### Require At Least Two Occurrences

To require at least two consecutive ASN occurrences:

```text
^65001(_65001)+$
```

The `+` operator requires at least one additional occurrence of the grouped expression.

This matches:

```text
65001 65001
65001 65001 65001
```

But not:

```text
65001
65001 65002
```

These expressions are useful when testing or filtering AS_PATH prepending.

However, regular-expression filtering does not necessarily distinguish between deliberate prepending and an ASN repeated for another reason.

---

## AS-Path Access Lists

Cisco IOS XE supports AS-path access lists for matching routes based on regular expressions.

The syntax is:

```text
R1(config)# ip as-path access-list <NUMBER> {permit | deny} <REGEX>
```

For example:

```text
R1(config)# ip as-path access-list 10 permit _65001$
```

This permits paths whose rightmost ASN is 65001.

### Access-List Evaluation

AS-path access lists are evaluated in order.

The first matching entry determines whether the AS_PATH is permitted or denied.

If no entry matches, an **implicit deny** applies.

For example:

```text
R1(config)# ip as-path access-list 10 deny _65001_
R1(config)# ip as-path access-list 10 permit .*
```

This configuration:

1. Denies routes containing AS 65001.
2. Permits other routes.

The second entry is necessary if all routes not matching the deny condition should be permitted.

Without it, unmatched routes are also rejected by the implicit deny.

### AS-Path ACL Number Range

Cisco IOS XE supports numbered AS-path access lists.

The documented range is:

```text
1-500
```

The exact supported naming and configuration options can vary by platform and software release.

---

## Filtering Routes with neighbor filter-list

An AS-path access list can be applied directly to a BGP neighbor using `neighbor filter-list`.

Syntax:

```text
R1(config-router-af)# neighbor <IP> filter-list <NUMBER> {in | out}
```

### Example: Reject Routes Traversing AS 65050

R1 should reject routes received from R2 if their AS_PATH contains AS 65050.

#### Topology

```text
        AS 65001                    AS 65002
           R1 ------------------------- R2
                      EBGP
             10.0.12.1      10.0.12.2
```

Configure the AS-path access list:

```text
R1(config)# ip as-path access-list 10 deny _65050_
R1(config)# ip as-path access-list 10 permit .*
```

Apply it inbound:

```text
R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 filter-list 10 in
```

### Result

| Received AS_PATH | Result |
|---|---|
| `65002 65003` | Permitted |
| `65002 65050` | Denied |
| `65002 65003 65050` | Denied |
| `65002 65050 65003` | Denied |
| `65002 65004` | Permitted |

Any AS_PATH containing AS 65050 is denied.

Other paths are permitted.

### Outbound Filtering

The same access list can be applied outbound:

```text
R1(config-router-af)# neighbor 10.0.12.2 filter-list 10 out
```

This controls which routes R1 is permitted to advertise to R2.

**Important:** `filter-list` applies AS_PATH-based filtering. It does not directly match destination IP prefixes.

Use prefix lists for prefix-based filtering.

---

## Filtering and Attribute Manipulation with Route Maps

AS-path access lists can also be referenced in route maps.

The command is:

```text
R1(config-route-map)# match as-path <NUMBER>
```

A route map allows the AS_PATH match to be combined with other match conditions and `set` actions.

### Example: Set LOCAL_PREF Based on Origin AS

R1 should assign LOCAL_PREF 200 to routes originating from AS 65050.

First, create an AS-path access list:

```text
R1(config)# ip as-path access-list 10 permit _65050$
```

Create a route map:

```text
R1(config)# route-map FROM-R2 permit 10
R1(config-route-map)# match as-path 10
R1(config-route-map)# set local-preference 200
R1(config)# route-map FROM-R2 permit 20
```

Apply it inbound:

```text
R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 route-map FROM-R2 in
```

### Result

Routes whose AS_PATH ends with AS 65050 receive LOCAL_PREF 200.

Other accepted routes are permitted by sequence 20 without that modification.

### Combining AS_PATH and Prefix Matching

Suppose R1 should assign LOCAL_PREF 200 only to routes that:

1. Originate from AS 65050.
2. Belong to `10.10.0.0/16` or its more-specific prefixes up to `/24`.

Configure:

```text
R1(config)# ip as-path access-list 10 permit _65050$
R1(config)# ip prefix-list CUSTOMER permit 10.10.0.0/16 le 24
R1(config)# route-map FROM-R2 permit 10
R1(config-route-map)# match as-path 10
R1(config-route-map)# match ip address prefix-list CUSTOMER
R1(config-route-map)# set local-preference 200
R1(config)# route-map FROM-R2 permit 20
```

Both match conditions must be satisfied for sequence 10 to apply.

The route must:

- Match the AS-path access list.
- Match the prefix list.

Otherwise, processing continues to sequence 20.

---

## AS-Path ACL Permit/Deny vs. Route-Map Permit/Deny

A critical distinction is how AS-path access-list permit and deny entries interact with route maps.

Within a route map, an AS-path access list is used to **identify matching routes**.

Only AS_PATHs permitted by the access list satisfy the `match as-path` condition.

A deny entry in the AS-path ACL means the route does not satisfy that match condition.

It does not automatically mean the entire BGP route is denied.

### Example 1: Deny Routes Using a Filter List

```text
R1(config)# ip as-path access-list 10 deny _65050_
R1(config)# ip as-path access-list 10 permit .*
R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 filter-list 10 in
```

Routes containing AS 65050 are denied.

### Example 2: Deny Routes Using a Route Map

The equivalent route-map configuration uses an AS-path ACL **permit** statement to identify the routes to reject.

```text
R1(config)# ip as-path access-list 10 permit _65050_
R1(config)# route-map FILTER-AS deny 10
R1(config-route-map)# match as-path 10
R1(config)# route-map FILTER-AS permit 20
R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 route-map FILTER-AS in
```

Here:

1. The AS-path ACL permits paths containing AS 65050.
2. Those paths match route-map sequence 10.
3. Sequence 10 is a `deny` sequence, so the matching BGP routes are rejected.
4. Other routes continue to sequence 20 and are permitted.

### Comparison

| Feature | Determines |
|---|---|
| AS-path ACL `permit` | Whether an AS_PATH satisfies an ACL match |
| AS-path ACL `deny` | Whether an AS_PATH fails that ACL match |
| Route-map `permit` | Allow matching routes and apply any set actions |
| Route-map `deny` | Reject matching routes |

**Important:** When using `match as-path` in a route map, the AS-path ACL identifies the routes matched by the route-map sequence. The route-map sequence determines whether those routes are accepted or rejected.

This is one of the most important distinctions when combining AS-path ACLs and route maps.

---

## Regular Expressions and 4-Byte ASNs

Cisco IOS XE supports 4-byte ASNs.

For example:

```text
100000
```

Modern IOS XE normally displays 4-byte ASNs using **asplain notation**.

An AS_PATH might appear as:

```text
100000 65001 65002
```

The following regular expression matches AS 100000 anywhere:

```text
_100000_
```

### Asplain vs. Asdot

A 4-byte ASN can also be represented using asdot notation.

For example:

```text
65550 = 1.14
```

Cisco IOS XE supports changing ASN display notation:

```text
R1(config-router)# bgp asnotation dot
```

When asdot notation is enabled, regular expressions must match the asdot representation.

For example:

```text
_1\.14_
```

The backslash is required because the period (`.`) is a regular-expression wildcard.

Without the backslash:

```text
_1.14_
```

The expression could match other character sequences, not just `1.14`.

**Important:** Regular expressions match the ASN representation used by the router. With asdot notation enabled, a pattern using the equivalent asplain value will not match the displayed asdot ASN.

See [4-Byte AS Numbers](../peering/four-byte-asn.md).

---

## AS_PATH Segment Types and Regular Expressions

AS_PATH is composed of one or more segments.

Important segment types include:

| Segment | Description |
|---|---|
| AS_SEQUENCE | Ordered ASN sequence |
| AS_SET | Unordered collection of ASNs |
| AS_CONFED_SEQUENCE | Ordered confederation member ASNs |
| AS_CONFED_SET | Unordered confederation member ASNs |

Regular expressions operate on the textual representation of these segments.

### AS_SET Example

An AS_SET might be displayed as:

```text
65010 {65001,65002}
```

The expression:

```text
_65001_
```

Can match AS 65001 within the set.

However:

```text
_65001$
```

Does not match this example because the AS_PATH ends with a closing brace, not the digits `65001`.

Also, AS_SET elements are unordered, so their displayed order should not be interpreted as a traversal sequence.

### Confederation Example

A confederation AS_PATH might be displayed as:

```text
(65002 65001) 65100
```

The expression:

```text
_65002_
```

Can match AS 65002 inside the confederation sequence.

The underscore recognizes parentheses as delimiters.

**Important:** AS_PATH regular expressions operate on the displayed AS_PATH representation. Be careful when using expressions that assume every path consists of a simple AS_SEQUENCE.

---

## Verification and Troubleshooting

Cisco IOS XE provides several useful commands for testing regular expressions and AS_PATH filtering.

### Test a Regular Expression

Use:

```text
R1# show ip bgp regexp <REGEX>
```

For example:

```text
R1# show ip bgp regexp _65001_
```

This displays BGP paths whose AS_PATH contains AS 65001.

### Match Routes from a Specific Origin AS

```text
R1# show ip bgp regexp _65001$
```

This displays paths whose rightmost complete ASN is 65001.

### Match Routes from a Particular Neighboring AS

```text
R1# show ip bgp regexp ^65001_
```

This displays paths whose leftmost complete ASN is 65001.

### Match Empty AS_PATHs

```text
R1# show ip bgp regexp ^$
```

This displays paths with empty AS_PATHs.

### Match AS_PATHs with Exactly One ASN

```text
R1# show ip bgp regexp ^[0-9]+$
```

This displays ordinary AS_SEQUENCE paths containing exactly one ASN.

### Display AS-Path Access Lists

```text
R1# show ip as-path-access-list
```

This displays configured AS-path access lists.

### Display Routes Matching an AS-Path ACL

```text
R1# show ip bgp filter-list 10
```

This displays BGP routes matching the permitted entries of AS-path access list 10.

### Verify Neighbor Configuration

```text
R1# show ip bgp neighbors 10.0.12.2
R1# show running-config | section router bgp
```

Check which filter lists or route maps are applied to the neighbor.

### Verify Accepted Routes

```text
R1# show ip bgp neighbors 10.0.12.2 routes
```

This displays accepted BGP routes from the specified neighbor.

### Reapply Filtering Policies

After modifying an inbound AS_PATH filter:

```text
R1# clear ip bgp 10.0.12.2 soft in
```

This can trigger Route Refresh when supported by the neighbor.

For outbound filtering:

```text
R1# clear ip bgp 10.0.12.2 soft out
```

These operations normally allow the updated policies to be applied without resetting the established BGP session.

---

## Common Regular Expression Mistakes

### Matching a Substring Instead of a Complete ASN

Incorrect:

```text
65001
```

This can match ASNs such as `165001` or `650010`.

Better:

```text
_65001_
```

### Confusing Origin AS with Leftmost AS

```text
^65001_
```

Matches paths beginning with AS 65001.

```text
_65001$
```

Matches paths ending with AS 65001.

For ordinary AS_SEQUENCE paths:

- `^65001_` identifies the most recently traversed AS.
- `_65001$` identifies the originating AS.

### Forgetting the Implicit Deny

Consider:

```text
ip as-path access-list 10 deny _65050_
```

If applied directly as a filter list, this rejects every route:

- Routes matching AS 65050 are explicitly denied.
- Other routes are implicitly denied.

To deny AS 65050 while permitting everything else:

```text
ip as-path access-list 10 deny _65050_
ip as-path access-list 10 permit .*
```

### Confusing * with an ASN Wildcard

The expression:

```text
65001*
```

Does not mean any number of repetitions of ASN 65001.

The asterisk applies to the final character `1`.

To match one or more consecutive occurrences of ASN 65001, use a grouped expression such as:

```text
^65001(_65001)*$
```

### Forgetting Route-Map Semantics

An AS-path ACL deny entry does not directly deny a BGP route when referenced by `match as-path`.

The ACL determines whether the route matches the route-map sequence.

The route-map sequence's permit or deny action controls whether the matching route is accepted.

---

## CCIE-Level Regular Expression Examples

The following examples assume ordinary AS_SEQUENCE paths unless noted otherwise.

| Requirement | Expression |
|---|---|
| Match any AS_PATH | `.*` |
| Match empty AS_PATH | `^$` |
| Match exactly AS 65001 | `^65001$` |
| Match routes originating from AS 65001 | `_65001$` |
| Match routes whose leftmost ASN is 65001 | `^65001_` |
| Match routes that traversed AS 65001 | `_65001_` |
| Match paths beginning with AS 65001 then AS 65002 | `^65001_65002_` |
| Match paths ending with AS 65001 then AS 65002 | `_65001_65002$` |
| Match exactly two numeric ASNs | `^[0-9]+_[0-9]+$` |
| Match one numeric ASN | `^[0-9]+$` |
| Match paths containing only repeated AS 65001 | `^65001(_65001)*$` |
| Match exactly two occurrences of any identical ASN | `^([0-9]+)_\1$` |
| Match two or more identical ASNs consecutively | `_([0-9]+)(_\1)+_` |
| Match two or more occurrences of only AS 65001 | `^65001(_65001)+$` |
| Match origin AS 65001 or 65002 | `_(65001\|65002)$` |
| Match leftmost AS 65001 or 65002 | `^(65001\|65002)_` |

### Practice Example

Consider these AS_PATHs:

```text
A: 65001
B: 65002 65001
C: 65001 65001 65001
D: 65003 65002 65001
E: 65001 65002
F: 65002 65003
G: (empty)
```

Which paths match each expression?

| Expression | Matching Paths |
|---|---|
| `^65001$` | A |
| `_65001$` | A, B, C, D |
| `^65001_` | A, C, E |
| `_65001_` | A, B, C, D, E |
| `^$` | G |
| `^[0-9]+$` | A |
| `^65001(_65001)*$` | A, C |
| `^65001_65002$` | E |

Understanding these patterns is important for configuring and troubleshooting AS_PATH filters.

---

## Command Summary

| Command | Purpose |
|---|---|
| `ip as-path access-list <N> permit <REGEX>` | Permit matching AS_PATHs |
| `ip as-path access-list <N> deny <REGEX>` | Deny matching AS_PATHs |
| `neighbor <IP> filter-list <N> in` | Apply AS_PATH filtering inbound |
| `neighbor <IP> filter-list <N> out` | Apply AS_PATH filtering outbound |
| `match as-path <N>` | Match an AS-path access list in a route map |
| `show ip bgp regexp <REGEX>` | Display routes matching a regular expression |
| `show ip bgp filter-list <N>` | Display routes matching an AS-path access list |
| `show ip as-path-access-list` | Display configured AS-path access lists |
| `show route-map` | Display configured route maps |
| `clear ip bgp <IP> soft in` | Reprocess inbound BGP policy |
| `clear ip bgp <IP> soft out` | Reprocess outbound BGP policy |

---

## Key Points

- BGP regular expressions match the textual representation of the AS_PATH attribute.
- They can be used for route filtering and routing-policy matching.
- `^` matches the beginning of the AS_PATH string.
- `$` matches the end of the AS_PATH string.
- `_` matches Cisco AS_PATH boundaries and delimiters.
- `.*` matches any AS_PATH, including empty.
- `^$` matches an empty AS_PATH.
- `^65001$` matches an AS_PATH containing exactly one occurrence of AS 65001.
- `^65001_` matches paths whose leftmost ASN is 65001.
- `_65001$` matches paths whose rightmost ASN is 65001.
- `_65001_` matches paths containing AS 65001 anywhere.
- Without anchors or delimiters, a numeric expression can match part of a larger ASN.
- `*`, `+`, and `?` control repetitions of the preceding expression.
- Parentheses allow multiple characters or expressions to be grouped.
- `[]` matches a single character from a specified set or range.
- Alternation allows matching one of multiple possible patterns.
- `\1`, `\2`, etc. refer to previously captured groups and can match repeated ASN values.
- `^([0-9]+)(_\1)+$` matches a whole AS_PATH consisting of at least two consecutive copies of the same ASN.
- `_([0-9]+)(_\1)+_` matches consecutive copies of an ASN anywhere in an AS_PATH.
- AS-path access lists have an implicit deny.
- `neighbor filter-list` applies an AS-path access list directly to a neighbor.
- `match as-path` allows AS-path access lists to be used in route maps.
- **An AS-path ACL deny entry does not directly deny a BGP route when used by `match as-path`**.
- `show ip bgp regexp` allows regular expressions to be tested before configuring route filtering.
- Regular expressions match the current ASN display notation, including asplain or asdot for 4-byte ASNs.
- AS_SET and confederation segments can affect regular-expression matching because they use additional delimiters.

---

## References

- [Cisco: Use Regular Expressions in BGP](https://www.cisco.com/c/en/us/support/docs/ip/border-gateway-protocol-bgp/13754-26.html)
- [Cisco: Examine Border Gateway Protocol Case Studies](https://www.cisco.com/c/en/us/support/docs/ip/border-gateway-protocol-bgp/26634-bgp-toc.html)
- [Cisco IOS XE: Connecting to a Service Provider Using External BGP](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/ip-routing/b-ip-routing/m_irg-external-sp-0.html)
- [Cisco: BGP Command Reference, ip as-path access-list](https://www.cisco.com/c/en/us/td/docs/ios/iproute_bgp/command/reference/irg_book/irg_bgp2.html)
- [Cisco: BGP Command Reference, show ip bgp regexp](https://www.cisco.com/c/en/us/td/docs/ios/iproute_bgp/command/reference/irg_book/irg_bgp5.html)
- [Cisco IOS XE: Regular Expressions](https://www.cisco.com/c/en/us/td/docs/ios/termserv/configuration/guide/xe_16/tsv-xe-book/tsv_reg_express.html)
- [Cisco Community: BGP AS Regex Question (Backreference Examples)](https://community.cisco.com/t5/routing-and-sd-wan/bgp-as-regex-question/td-p/2097125)
