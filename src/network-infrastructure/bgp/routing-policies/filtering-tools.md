# BGP Prefix Lists, AS-Path ACLs, and Route Maps

Cisco IOS XE provides several tools for matching BGP routes and implementing routing policies.

Three of the most important are:

- **Prefix lists:** Match routes based on their network prefixes and prefix lengths.
- **AS-path access lists:** Match routes based on their AS_PATH attributes using regular expressions.
- **Route maps:** Combine match conditions to filter routes or modify their attributes.

These tools can be applied directly to BGP neighbors or combined to implement more complex routing policies.

---

## BGP Prefix Lists

**Prefix lists** match routes based on their network addresses and prefix lengths.

They are commonly used to:

- Permit or deny specific prefixes.
- Filter routes based on prefix-length ranges.
- Prevent advertisement of unauthorized prefixes.
- Match routes in BGP route maps.

### Prefix-List Syntax

```text
R1(config)# ip prefix-list <name> [seq <number>] {permit | deny} <prefix> [ge <length>] [le <length>]
```

| Parameter | Description |
|---|---|
| `name` | Prefix-list name |
| `seq` | Optional sequence number |
| `permit` | Permit matching prefixes |
| `deny` | Deny matching prefixes |
| `prefix` | Base network and prefix length |
| `ge` | Minimum matching prefix length |
| `le` | Maximum matching prefix length |

Prefix lists are evaluated in ascending sequence-number order.

**The first matching entry determines the result.**

If no entry matches, an implicit deny applies to a nonempty prefix list.

### Exact Prefix Matching

Without `ge` or `le`, a prefix-list entry matches only the specified prefix and prefix length.

```text
R1(config)# ip prefix-list PL1 permit 10.10.0.0/16
```

| Prefix | Match? |
|---|---|
| `10.10.0.0/16` | Yes |
| `10.10.0.0/24` | No |
| `10.10.1.0/24` | No |
| `10.11.0.0/16` | No |

The prefix and prefix length must both match exactly.

### Prefix-Length Range Matching

The `ge` and `le` options allow the prefix list to match more-specific prefixes.

For example:

```text
R1(config)# ip prefix-list PL1 permit 10.10.0.0/16 le 24
```

This matches prefixes within `10.10.0.0/16` whose prefix lengths are between `/16` and `/24`, inclusive.

| Prefix | Match? |
|---|---|
| `10.10.0.0/16` | Yes |
| `10.10.0.0/17` | Yes |
| `10.10.1.0/24` | Yes |
| `10.10.1.0/25` | No |
| `10.11.0.0/16` | No |

The base prefix restricts the address range, while `ge` and `le` restrict the allowed prefix lengths.

### Using `ge` and `le` Together

```text
R1(config)# ip prefix-list PL1 permit 10.10.0.0/16 ge 24 le 28
```

This matches routes that:

1. Are contained within `10.10.0.0/16`.
2. Have prefix lengths between `/24` and `/28`, inclusive.

| Prefix | Match? |
|---|---|
| `10.10.0.0/16` | No |
| `10.10.0.0/23` | No |
| `10.10.1.0/24` | Yes |
| `10.10.1.128/25` | Yes |
| `10.10.1.128/28` | Yes |
| `10.10.1.128/29` | No |
| `10.11.1.0/24` | No |

### Prefix-List Matching Rules

The allowed prefix-length ranges are:

| Configuration | Matching Prefix Lengths |
|---|---|
| `10.0.0.0/8` | Exactly `/8` |
| `10.0.0.0/8 ge 16` | `/16` through `/32` |
| `10.0.0.0/8 le 24` | `/8` through `/24` |
| `10.0.0.0/8 ge 16 le 24` | `/16` through `/24` |
| `10.0.0.0/8 le 32` | `/8` through `/32` |

All entries match only networks contained within the specified base prefix.

### Matching the Default Route

To match only the default route:

```text
R1(config)# ip prefix-list DEFAULT permit 0.0.0.0/0
```

This matches `0.0.0.0/0` only.

To match every possible IPv4 prefix:

```text
R1(config)# ip prefix-list ANY permit 0.0.0.0/0 le 32
```

The difference is important:

- `0.0.0.0/0` matches only the default route.
- `0.0.0.0/0 le 32` matches all IPv4 prefixes.

An additional example:

```text
R1(config)# ip prefix-list ANY-SUBNET permit 0.0.0.0/0 ge 1 le 32
```

This matches all IPv4 prefixes **except the default route**.

---

## Prefix-List Sequence Numbers

Prefix-list entries are evaluated from the lowest sequence number to the highest.

Consider:

```text
R1(config)# ip prefix-list FILTER seq 10 deny 10.10.10.0/24
R1(config)# ip prefix-list FILTER seq 20 permit 10.0.0.0/8 le 32
```

The result is:

- `10.10.10.0/24` is denied by sequence 10.
- Other prefixes within `10.0.0.0/8` are permitted by sequence 20.
- All remaining prefixes are denied by the implicit deny.

### Order Matters

Consider the reversed configuration:

```text
R1(config)# ip prefix-list FILTER seq 10 permit 10.0.0.0/8 le 32
R1(config)# ip prefix-list FILTER seq 20 deny 10.10.10.0/24
```

Now `10.10.10.0/24` is **permitted** because it matches sequence 10.

Sequence 20 is never evaluated for that prefix.

### Editing Prefix Lists

Insert an entry:

```text
R1(config)# ip prefix-list FILTER seq 15 deny 10.20.20.0/24
```

Remove an entry:

```text
R1(config)# no ip prefix-list FILTER seq 15
```

Resequence a prefix-list:

```text
R1# ip prefix-list FILTER resequence 10 10
```

This renumbers entries beginning at 10, with increments of 10, on platforms supporting the command.

**Note:** Cisco documents that a completely empty prefix list permits all prefixes when used as a direct BGP prefix filter.

---

## Applying Prefix Lists to BGP Neighbors

Prefix lists can be applied inbound or outbound.

### Inbound

```text
R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 prefix-list FILTER in
```

This controls which routes received from the neighbor are accepted.

### Outbound

```text
R1(config-router-af)# neighbor 10.0.12.2 prefix-list FILTER out
```

This controls which routes are advertised to the neighbor.

A prefix list can be applied in both directions using separate commands.

To remove an inbound filter:

```text
R1(config-router-af)# no neighbor 10.0.12.2 prefix-list FILTER in
```

See [BGP Route Filtering](route-filtering.md) for further discussion of filtering directions.

---

## BGP AS-Path Access Lists

**AS-path access lists** match BGP routes based on their AS_PATH attributes.

Unlike prefix lists, which examine destination prefixes, AS-path access lists examine the autonomous systems recorded in the path.

AS-path access lists use **regular expressions** to match ASN sequences.

### Syntax

```text
R1(config)# ip as-path access-list <number> {permit | deny} <regular-expression>
```

For example:

```text
R1(config)# ip as-path access-list 10 permit ^65002$
```

This permits routes whose AS_PATH consists of exactly AS 65002.

Other AS_PATH values are denied by the implicit deny.

### AS_PATH Order

Consider this AS_PATH:

```text
65002 65003 65004
```

- `65002` is the leftmost AS, normally the neighboring AS for a standard EBGP-learned path.
- `65004` is the rightmost AS, normally the originating AS.

AS-path regular expressions match the displayed AS_PATH representation.

---

## AS_PATH Regular Expressions

Cisco IOS XE supports special characters for matching AS_PATH values.

| Character | Meaning |
|---|---|
| `^` | Beginning of the AS_PATH |
| `$` | End of the AS_PATH |
| `_` | AS boundary, including separators and beginning/end boundaries |
| `.` | Any single character |
| `*` | Zero or more occurrences of the preceding expression |
| `+` | One or more occurrences of the preceding expression |
| `?` | Zero or one occurrence of the preceding expression |
| `[ ]` | Character class |
| `( )` | Grouping |
| `\|` | Alternation (OR) |

The underscore (`_`) is especially important because it matches AS boundaries, including spaces, commas, braces, parentheses, and the beginning or end of the AS_PATH.

### Common AS_PATH Expressions

| Expression | Matches |
|---|---|
| `^$` | Empty AS_PATH |
| `^65002$` | Exactly AS 65002 |
| `^65002_` | Paths beginning with AS 65002 |
| `_65002$` | Paths originating in AS 65002 |
| `_65002_` | Paths containing AS 65002 anywhere |
| `^65002_65003$` | Exactly the sequence 65002 65003 |
| `^65002_65003_` | Paths beginning with 65002 65003 |
| `.*` | Any AS_PATH, including an empty path |

These patterns are interpreted as AS_PATH regular expressions, not prefix or interface address matches.

### Example 1: Match Locally Originated Paths

```text
R1(config)# ip as-path access-list 10 permit ^$
```

This matches an empty AS_PATH.

A locally originated BGP route normally has an empty AS_PATH until it is advertised to an EBGP neighbor.

However, IBGP-learned paths can also have empty AS_PATH values.

Therefore, `^$` is **not guaranteed to match only routes originated by the local router**.

### Example 2: Match a Specific Origin AS

```text
R1(config)# ip as-path access-list 10 permit _65002$
```

This matches routes whose rightmost ASN is 65002.

Examples:

| AS_PATH | Match? |
|---|---|
| `65002` | Yes |
| `65003 65002` | Yes |
| `65004 65003 65002` | Yes |
| `65002 65003` | No |

### Example 3: Match a Specific Neighboring AS

```text
R1(config)# ip as-path access-list 10 permit ^65002_
```

This matches AS_PATH values beginning with AS 65002.

Examples:

| AS_PATH | Match? |
|---|---|
| `65002` | Yes |
| `65002 65003` | Yes |
| `65002 65003 65004` | Yes |
| `65003 65002` | No |

### Example 4: Match an AS Anywhere in the Path

```text
R1(config)# ip as-path access-list 10 permit _65002_
```

This matches AS_PATH values containing AS 65002 at an AS boundary.

Examples:

| AS_PATH | Match? |
|---|---|
| `65002` | Yes |
| `65003 65002` | Yes |
| `65003 65002 65004` | Yes |
| `65003 165002` | No |

Using underscores avoids accidentally matching ASN 65002 as part of another ASN.

### Example 5: Deny Routes Traversing a Specific AS

```text
R1(config)# ip as-path access-list 10 deny _65002_
R1(config)# ip as-path access-list 10 permit .*
```

This rejects paths containing AS 65002 and permits all other paths.

The `permit .*` statement is necessary because the access list otherwise has an implicit deny.

---

## Applying AS-Path Access Lists

AS-path access lists can be applied directly to BGP neighbors using **filter lists**.

### Inbound Filter List

```text
R1(config)# ip as-path access-list 10 permit ^65002$

R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 filter-list 10 in
```

Only routes with an AS_PATH of exactly `65002` are accepted from the neighbor.

### Outbound Filter List

```text
R1(config-router-af)# neighbor 10.0.12.2 filter-list 10 out
```

This applies the AS_PATH filter to routes being advertised.

**Important:** An outbound AS-path filter evaluates AS_PATH information as processed by the advertising router for the policy. Account for the distinction between the local BGP table's AS_PATH and the AS_PATH ultimately transmitted to the neighbor.

AS-path filtering is covered further in [BGP AS Path Regular Expressions](../as-path/regular-expressions.md).

---

## BGP Route Maps

**Route maps** combine match conditions with permit/deny actions and optional attribute modifications.

They are more flexible than prefix lists or AS-path access lists because they can match multiple aspects of a BGP route.

### Route-Map Syntax

```text
R1(config)# route-map <name> {permit | deny} <sequence-number>
R1(config-route-map)# match ...
R1(config-route-map)# set ...
```

A route map consists of one or more ordered sequences.

Each sequence can contain:

- **Match statements:** Define which routes match the sequence.
- **Set statements:** Modify attributes of matching routes.
- **Permit or deny action:** Determines whether matching routes are accepted or rejected when used for BGP filtering.

### Route-Map Processing

Route-map sequences are evaluated in ascending sequence-number order.

For ordinary BGP filtering without `continue`:

1. Evaluate the first sequence.
2. If its match conditions fail, move to the next sequence.
3. If its match conditions succeed:
   - `permit`: Accept the route and apply configured `set` actions.
   - `deny`: Reject the route.
4. Stop processing after a matching sequence.
5. If no sequence matches, reject the route through the implicit deny.

A sequence without any `match` statements matches all routes reaching that sequence.

---

## Route-Map Permit and Deny

Consider:

```text
R1(config)# ip prefix-list BLOCKED permit 10.10.10.0/24

R1(config)# route-map FILTER deny 10
R1(config-route-map)# match ip address prefix-list BLOCKED

R1(config)# route-map FILTER permit 20
```

The result is:

- `10.10.10.0/24` matches sequence 10 and is rejected.
- All other routes fail sequence 10 and are accepted by sequence 20.

### Prefix-List Deny vs. Route-Map Deny

This distinction is essential.

Consider:

```text
R1(config)# ip prefix-list PL1 deny 10.10.10.0/24
R1(config)# ip prefix-list PL1 permit 0.0.0.0/0 le 32

R1(config)# route-map RM1 deny 10
R1(config-route-map)# match ip address prefix-list PL1

R1(config)# route-map RM1 permit 20
```

What happens to `10.10.10.0/24`?

1. The prefix matches the **deny** entry in PL1.
2. Therefore, `match ip address prefix-list PL1` returns **no match**.
3. Route-map sequence 10 does not match.
4. Processing continues to sequence 20.
5. Sequence 20 permits the route.

**The route is permitted, not denied.**

A prefix-list `deny` does not automatically trigger a route-map `deny`.

For route-map matching purposes:

- A prefix-list `permit` identifies a successful match.
- A prefix-list `deny` produces no successful match for that condition.

The same principle applies to AS-path access lists used in route-map match statements.

---

## Multiple Route-Map Match Conditions

Route maps can contain multiple match conditions.

The distinction between **AND** and **OR** logic is important.

### Different Match Types: AND

Different match types in the same sequence generally use logical **AND**.

For example:

```text
R1(config)# ip prefix-list PL1 permit 203.0.113.0/24
R1(config)# ip as-path access-list 10 permit ^65002$

R1(config)# route-map FILTER permit 10
R1(config-route-map)# match ip address prefix-list PL1
R1(config-route-map)# match as-path 10
```

For the route map to match:

1. The prefix must match PL1.
2. The AS_PATH must match AS-path access list 10.

Both conditions must be satisfied.

| Prefix | AS_PATH | Match? |
|---|---|---|
| `203.0.113.0/24` | `65002` | Yes |
| `203.0.113.0/24` | `65003` | No |
| `198.51.100.0/24` | `65002` | No |
| `198.51.100.0/24` | `65003` | No |

### Multiple Values of the Same Match Type: OR

Multiple alternatives within the same match condition generally use logical **OR**.

For example:

```text
R1(config)# ip prefix-list PL1 permit 203.0.113.0/24
R1(config)# ip prefix-list PL2 permit 198.51.100.0/24

R1(config)# route-map FILTER permit 10
R1(config-route-map)# match ip address prefix-list PL1 PL2
```

The route matches if it is permitted by **either PL1 or PL2**.

Similarly:

```text
R1(config)# ip as-path access-list 10 permit ^65002$
R1(config)# ip as-path access-list 20 permit ^65003$

R1(config)# route-map FILTER permit 10
R1(config-route-map)# match as-path 10 20
```

This matches routes whose AS_PATH is permitted by either access list 10 or 20.

**Summary:**

- Alternatives within the same match type: generally **OR**.
- Different match types in the same route-map sequence: generally **AND**.

Certain match commands and features have special matching rules, so verify platform-specific behavior where necessary.

---

## Route-Map Sequence Evaluation

Consider this route map:

```text
R1(config)# ip prefix-list PL1 permit 10.10.10.0/24
R1(config)# ip prefix-list PL2 permit 10.20.20.0/24

R1(config)# route-map FILTER deny 10
R1(config-route-map)# match ip address prefix-list PL1

R1(config)# route-map FILTER permit 20
R1(config-route-map)# match ip address prefix-list PL2
R1(config-route-map)# set local-preference 200

R1(config)# route-map FILTER permit 30
R1(config-route-map)# set local-preference 100
```

| Prefix | Matching Sequence | Result |
|---|---|---|
| `10.10.10.0/24` | 10 | Denied |
| `10.20.20.0/24` | 20 | Permitted, LOCAL_PREF 200 |
| `10.30.30.0/24` | 30 | Permitted, LOCAL_PREF 100 |

Sequence 30 has no match conditions, so it permits all remaining routes.

### Route Map with Only Set Statements

Consider:

```text
R1(config)# route-map SET-PREF permit 10
R1(config-route-map)# set local-preference 200
```

There are no match conditions.

Therefore, all routes processed by this route map match sequence 10 and are assigned LOCAL_PREF 200.

This is useful when a policy should apply an attribute change to every accepted route.

### Route Map with Only One Permit Sequence

Consider:

```text
R1(config)# route-map FILTER permit 10
R1(config-route-map)# match ip address prefix-list PL1
```

Only routes matching PL1 are permitted.

All remaining routes are rejected by the implicit deny.

To modify selected routes while accepting everything else, add a final permit sequence:

```text
R1(config)# route-map FILTER permit 20
```

---

## Applying Route Maps to BGP Neighbors

Route maps can be applied inbound or outbound under the BGP address family.

### Inbound Route Map

```text
R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 route-map FILTER in
```

Inbound route maps can:

- Reject routes.
- Modify Weight.
- Modify LOCAL_PREF.
- Match AS_PATH values.
- Match communities.
- Modify other supported BGP attributes.

### Outbound Route Map

```text
R1(config-router-af)# neighbor 10.0.12.2 route-map FILTER out
```

Outbound route maps can:

- Control route advertisement.
- Modify MED.
- Prepend AS_PATH values.
- Set communities.
- Modify other attributes sent to the neighbor.

See [BGP Attribute Manipulation](attribute-manipulation.md) for additional examples of `set` commands.

---


## Route-Map `continue`

By default, route-map processing stops after the first matching sequence.

Cisco IOS XE supports the **`continue`** command in BGP route maps, allowing processing to continue after a matching permit sequence.

### Syntax

```text
R1(config-route-map)# continue [sequence-number]
```

- `continue 30`: Continue processing at sequence 30, skipping any intermediate sequences.
- `continue`: Continue processing at the next sequence.

The target sequence must have a higher sequence number than the current sequence.

### Example

```text
R1(config)# route-map POLICY permit 10
R1(config-route-map)# match ip address prefix-list PL1
R1(config-route-map)# set local-preference 200
R1(config-route-map)# continue 30

R1(config)# route-map POLICY permit 20
R1(config-route-map)# set local-preference 100

R1(config)# route-map POLICY permit 30
R1(config-route-map)# match as-path 10
R1(config-route-map)# set community 65001:100 additive
```

If a route matches sequence 10:

1. The LOCAL_PREF action is recorded.
2. Processing jumps to sequence 30, skipping sequence 20.
3. If sequence 30 matches, its community action is also applied.
4. If sequence 30 does not match, processing continues to subsequent sequences, if any.

### Permit and Deny Behavior

When `continue` is used, the final result depends on subsequent matching sequences.

| Subsequent processing | Result |
|---|---|
| Matches another `permit` sequence | Route is permitted; applicable `set` actions are combined |
| Matches an explicit `deny` sequence | Route is denied, overriding the earlier permit |
| No subsequent sequence matches | Route remains permitted because of the earlier permit |
| Reaches the implicit deny | Earlier permit is preserved; implicit deny does not reject the route |

**Important distinctions:**

- An **explicit deny** in a later matching sequence overrides an earlier permit.
- The **implicit deny** at the end of the route map does not override an earlier permit that matched and used `continue`.
- If the route is ultimately denied, the earlier `set` actions have no effective result.
- If the route is permitted, applicable `set` actions from matching sequences are applied.

### Multiple `set` Actions

When multiple permitted sequences match through `continue`, their `set` actions can interact.

- **Absolute-value actions**, such as `set local-preference`, generally use the last configured matching value.
- **Additive actions**, such as `set community ... additive` and `set as-path prepend`, can accumulate values across matching sequences.

For example, if sequence 10 sets LOCAL_PREF 200 and a later matching sequence sets LOCAL_PREF 300, the resulting LOCAL_PREF is 300.

The `continue` command allows complex BGP policies to be constructed without duplicating match conditions across multiple route-map sequences.

See [Cisco: BGP Route-Map Continue](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-routing/b-ip-routing/m_irg-route-map-continue.html).

---

## Combining Prefix and AS_PATH Filtering

Consider the following topology:

```text
        AS 65001                  AS 65002

           R1 ---------------------- R2
       10.0.12.1                10.0.12.2
```

R1 receives the following routes from R2:

| Prefix | AS_PATH |
|---|---|
| `203.0.113.0/24` | `65002` |
| `203.0.113.0/24` | `65002 65010` |
| `198.51.100.0/24` | `65002` |
| `192.0.2.0/24` | `65002 65020` |

Assume the goal is to:

1. Reject `198.51.100.0/24`.
2. Assign LOCAL_PREF 200 to `203.0.113.0/24` when originated by AS 65002.
3. Accept all other routes with LOCAL_PREF 100.

### Step 1: Configure Prefix Lists

```text
R1(config)# ip prefix-list BLOCKED permit 198.51.100.0/24
R1(config)# ip prefix-list PREFERRED permit 203.0.113.0/24
```

### Step 2: Configure AS-Path Access List

```text
R1(config)# ip as-path access-list 10 permit ^65002$
```

This matches routes whose AS_PATH consists only of AS 65002.

### Step 3: Configure Route Map

```text
R1(config)# route-map FROM-ISP deny 10
R1(config-route-map)# match ip address prefix-list BLOCKED

R1(config)# route-map FROM-ISP permit 20
R1(config-route-map)# match ip address prefix-list PREFERRED
R1(config-route-map)# match as-path 10
R1(config-route-map)# set local-preference 200

R1(config)# route-map FROM-ISP permit 30
R1(config-route-map)# set local-preference 100
```

### Step 4: Apply the Route Map

```text
R1(config)# router bgp 65001
R1(config-router)# neighbor 10.0.12.2 remote-as 65002
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 route-map FROM-ISP in
```

### Result

| Prefix | AS_PATH | Result |
|---|---|---|
| `203.0.113.0/24` | `65002` | Permit, LOCAL_PREF 200 |
| `203.0.113.0/24` | `65002 65010` | Permit, LOCAL_PREF 100 |
| `198.51.100.0/24` | `65002` | Deny |
| `192.0.2.0/24` | `65002 65020` | Permit, LOCAL_PREF 100 |

This example illustrates how prefix lists and AS-path access lists can be combined with route maps to create selective routing policies.

The table shows how each path would be processed individually. In a real BGP session, the paths must also satisfy normal AS loop prevention, next-hop validity, and other BGP acceptance requirements.

---

## Verification and Troubleshooting

### Display Prefix Lists

```text
R1# show ip prefix-list
R1# show ip prefix-list PL1
R1# show ip prefix-list detail PL1
```

The `detail` option can show match counters and additional information.

To find the first matching prefix-list entry for a specific prefix:

```text
R1# show ip prefix-list PL1 10.10.1.0/24 first-match
```

### Display AS-Path Access Lists

```text
R1# show ip as-path-access-list
```

### Test AS_PATH Regular Expressions

```text
R1# show ip bgp regexp ^65002$
R1# show ip bgp regexp _65002$
R1# show ip bgp regexp _65002_
```

These commands display BGP routes whose AS_PATH matches the specified expression.

They are useful for testing regular expressions before applying AS-path filters.

### Display Route Maps

```text
R1# show route-map
R1# show route-map FROM-ISP
```

These commands display route-map sequences, match conditions, set actions, and available counters.

Route-map match counters do not always represent the number of prefixes currently accepted by BGP.

### Verify Neighbor Policies

```text
R1# show running-config | section router bgp
R1# show bgp ipv4 unicast neighbors 10.0.12.2
```

The detailed neighbor output can show configured inbound and outbound filtering policies.

### Verify Accepted and Advertised Routes

```text
R1# show bgp ipv4 unicast neighbors 10.0.12.2 routes
R1# show bgp ipv4 unicast neighbors 10.0.12.2 advertised-routes
```

If inbound soft reconfiguration is enabled:

```text
R1# show bgp ipv4 unicast neighbors 10.0.12.2 received-routes
```

This allows comparison of received routes before inbound filtering with routes accepted into the local BGP table.

### Reapply Routing Policies

Inbound:

```text
R1# clear ip bgp 10.0.12.2 soft in
```

Outbound:

```text
R1# clear ip bgp 10.0.12.2 soft out
```

Inbound soft clearing generally uses the Route Refresh capability when supported.

---

## Common Configuration Mistakes

| Mistake | Result |
|---|---|
| `permit 0.0.0.0/0` intended to match all routes | Matches only the default route |
| Prefix list specifies no `ge` or `le` | Matches only the exact prefix length |
| General permit appears before specific deny | Specific deny may never be evaluated |
| Prefix-list deny used inside a route-map deny sequence | Denied prefix does not match that sequence |
| Route map has no final permit sequence | Nonmatching routes are rejected |
| `^65002$` used to match routes originating in AS 65002 through multiple ASes | Matches only a single-AS path |
| `65002` used instead of `_65002_` | Can match ASN digits appearing inside another ASN |
| Different route-map match types assumed to use OR | They generally use AND |
| Routing policy changed but not reapplied | Existing BGP routes may retain previous policy results |

---

## Key Points

- Prefix lists match network addresses and prefix-length ranges.
- Prefix lists use first-match processing and an implicit deny for nonempty lists.
- `ge` and `le` control the range of matching prefix lengths.
- AS-path access lists match BGP AS_PATH attributes using regular expressions.
- `^` matches the beginning of AS_PATH, `$` matches the end, and `_` matches AS boundaries.
- AS-path access lists are applied directly to BGP neighbors using `neighbor filter-list`.
- Route maps combine matching, filtering, and attribute manipulation.
- Different match types in a route-map sequence generally use AND logic.
- Multiple values of the same match type generally use OR logic.
- A prefix-list deny does not constitute a successful route-map match.
- Route-map processing normally stops at the first matching sequence, unless features such as `continue` are used.
- A final route-map permit sequence without match conditions accepts all remaining routes.

---

## References

- [Cisco: Use Regular Expressions in BGP](https://www.cisco.com/c/en/us/support/docs/ip/border-gateway-protocol-bgp/13754-26.html)
- [Cisco: IOS XE 17.x IP Routing Configuration Guide](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-routing/b-ip-routing.pdf)
- [Cisco: Connecting to a Service Provider Using External BGP](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-routing/b-ip-routing/m_irg-external-sp-0.html)
- [Cisco: BGP Route-Map Continue](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-routing/b-ip-routing/m_irg-route-map-continue.html)
