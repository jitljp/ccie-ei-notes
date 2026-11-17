# BGP Community Configuration and Filtering

Cisco IOS XE provides several mechanisms for configuring, matching, modifying, and filtering BGP communities.

The main configuration tools are:

- **Community lists:** Define standard or extended community values to match.
- **Route maps:** Apply policies based on community-list matches.
- **Set commands:** Add, replace, or remove communities.
- **Neighbor configuration:** Control community advertisement between BGP peers.

The most important distinction is between commands that **match communities**, commands that **assign communities**, and commands that **delete selected communities**.

| Command | Purpose |
|---|---|
| `ip community-list` | Define a list matching standard BGP communities |
| `ip extcommunity-list` | Define a list matching extended BGP communities |
| `match community` | Match a standard community list in a route map |
| `match extcommunity` | Match an extended community list in a route map |
| `set community` | Assign, replace, or clear standard communities |
| `set extcommunity` | Assign or replace supported extended-community values |
| `set comm-list ... delete` | Selectively remove standard communities |
| `set extcomm-list ... delete` | Selectively remove extended communities |
| `neighbor ... send-community` | Enable community advertisement to a BGP neighbor |

**Important:** Despite their names, `set comm-list` and `set extcomm-list` do not assign communities. They are used to **delete communities selected by community lists**.

---

## Configuring Community Lists

Community lists define which community values should match a routing policy.

Cisco IOS XE supports two categories of community lists:

1. **Standard community lists:** Match 32-bit standard BGP communities.
2. **Extended community lists:** Match 64-bit extended BGP communities.

Each category supports standard and expanded lists.

| Community Category | List Type | Matching Method |
|---|---|---|
| Standard BGP community | Standard list | Explicit community values |
| Standard BGP community | Expanded list | Regular expressions |
| Extended BGP community | Standard list | Explicit extended-community values |
| Extended BGP community | Expanded list | Regular expressions |

All four types support named and numbered lists.

### Standard BGP Community Lists

Use `ip community-list` to configure standard BGP community lists.

Numbered standard lists use numbers from 1 to 99:

```text
R1(config)# ip community-list 10 permit 65001:100
```

Named standard lists use the following syntax:

```text
R1(config)# ip community-list standard CUSTOMER permit 65001:100
```

Expanded community lists use numbers from 100 to 500:

```text
R1(config)# ip community-list 100 permit 65001:1[0-9][0-9]
```

Or a name:

```text
R1(config)# ip community-list expanded CUSTOMER-RANGE permit 65001:1[0-9][0-9]
```

Expanded lists match regular-expression patterns against community string representations.

### Extended BGP Community Lists

Use `ip extcommunity-list` to configure extended community lists.

For example, match a Route Target:

```text
R1(config)# ip extcommunity-list standard BLUE-RT permit rt 65001:100
```

Match a Site-of-Origin:

```text
R1(config)# ip extcommunity-list standard SITE-A permit soo 65001:10
```

The `rt` and `soo` keywords identify different extended-community types.

For example:

```text
RT:65001:100
SoO:65001:100
```

These are distinct extended communities, even though their numerical fields are identical.

An expanded extended community list can match a pattern of RT values:

```text
R1(config)# ip extcommunity-list 128 permit rt 65001:1[0-9][0-9]
```

This matches RT values within `65001:100` through `65001:199`.

---

## Community-List Matching Logic

Community lists are evaluated in entry order.

**The first matching permit or deny entry determines the community-list result.**

If no entry matches, the list has an implicit deny.

### Multiple Communities in One Standard Entry

Consider:

```text
R1(config)# ip community-list standard BOTH permit 65001:100 65001:200
```

Both communities must be present for the entry to match.

This is a logical AND condition.

| Communities on Route | Match? |
|---|---|
| `65001:100` | No |
| `65001:200` | No |
| `65001:100 65001:200` | Yes |
| `65001:100 65001:200 65001:300` | Yes |

The order of the communities does not matter.

### Separate Community-List Entries

Consider:

```text
R1(config)# ip community-list standard EITHER permit 65001:100
R1(config)# ip community-list standard EITHER permit 65001:200
```

The route matches if either entry matches.

This is a logical OR condition, with first-match processing.

### Permit and Deny Entries

Consider:

```text
R1(config)# ip community-list standard TEST deny 65001:100
R1(config)# ip community-list standard TEST permit 65001:200
```

| Communities on Route | List Result |
|---|---|
| `65001:100` | Deny |
| `65001:200` | Permit |
| `65001:100 65001:200` | Deny |
| `65001:300` | Implicit deny |

The third route is denied because the first entry matches before the second entry is evaluated.

**Important:** A community-list deny means the community-list match fails. It does not independently cause a BGP route to be rejected.

The route map using the community list determines the final action.

### Expanded Community Lists

Expanded community lists use regular expressions rather than exact value comparisons.

For example:

```text
R1(config)# ip community-list expanded CUSTOMER permit 65001:1[0-9][0-9]
```

This matches community strings containing values within `65001:100` through `65001:199`.

Because expanded lists use string matching, the ordering of communities in the matched string can matter when the regular expression specifies a sequence.

Standard community lists are generally preferable when the requirement is to match particular community values regardless of ordering.

---

## Matching Communities in Route Maps

Route maps can match both standard and extended communities.

The commands are:

```text
match community <community-list>
match extcommunity <extended-community-list>
```

The `match` commands do not change communities by themselves.

They determine whether a route satisfies the conditions of a route-map sequence.

### Matching Standard Communities

Configure a community list:

```text
R1(config)# ip community-list standard CUSTOMER permit 65001:100
```

Configure a route map:

```text
R1(config)# route-map CUSTOMER-IN permit 10
R1(config-route-map)# match community CUSTOMER
R1(config-route-map)# set local-preference 200

R1(config)# route-map CUSTOMER-IN permit 20
```

Apply it inbound:

```text
R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 route-map CUSTOMER-IN in
```

The policy:

1. Matches routes carrying community `65001:100`.
2. Sets LOCAL_PREF to 200.
3. Permits other routes unchanged through sequence 20.

The route-map permit sequence does not remove the matched community.

### Exact Community Matching

Normally, a community-list entry can match a route carrying additional communities.

For example:

```text
R1(config)# ip community-list standard CUSTOMER permit 65001:100
```

Matches:

```text
65001:100
65001:100 65001:200
```

To require an exact community-set match:

```text
R1(config)# route-map EXACT-MATCH permit 10
R1(config-route-map)# match community CUSTOMER exact-match
```

The `exact-match` keyword requires that the route's complete standard community set match the permitted community set.

| Communities on Route | Normal Match | Exact Match |
|---|---|---|
| `65001:100` | Yes | Yes |
| `65001:100 65001:200` | Yes | No |
| `65001:200` | No | No |

**Important:** The full IOS XE keyword is `exact-match`. Some Cisco documentation uses `exact` as an abbreviated form.

### Matching Extended Communities

Configure an extended community list:

```text
R1(config)# ip extcommunity-list standard BLUE permit rt 65001:100
```

Configure a route map:

```text
R1(config)# route-map MATCH-BLUE permit 10
R1(config-route-map)# match extcommunity BLUE
R1(config-route-map)# set local-preference 200

R1(config)# route-map MATCH-BLUE permit 20
```

This matches routes carrying RT `65001:100`.

**Important:** `match extcommunity` operates on the extended-community attribute. It does not match standard communities.

The ordinary IOS XE `match extcommunity` syntax does not document the same `exact-match` option used by `match community`. Check the CLI on the target platform rather than assuming both commands support identical options.

### Matching Both Standard and Extended Communities

A route map can contain both match types:

```text
R1(config)# ip community-list standard CUSTOMER permit 65001:100
R1(config)# ip extcommunity-list standard BLUE permit rt 65001:200

R1(config)# route-map MATCH-BOTH permit 10
R1(config-route-map)# match community CUSTOMER
R1(config-route-map)# match extcommunity BLUE
R1(config-route-map)# set local-preference 200

R1(config)# route-map MATCH-BOTH permit 20
```

The two different match types form a logical AND condition.

Sequence 10 matches only routes that carry:

- Standard community `65001:100`.
- Extended Route Target `65001:200`.

If either match fails, processing continues to sequence 20.

This can be useful for VPN routes that carry both standard and extended communities.

---

## Route-Map Permit and Deny Behavior

Community lists and route maps have separate permit/deny decisions.

Consider the following requirement:

- Reject routes carrying community `65001:100`.
- Accept all other routes.

Configure:

```text
R1(config)# ip community-list standard REJECT permit 65001:100

R1(config)# route-map FILTER-IN deny 10
R1(config-route-map)# match community REJECT

R1(config)# route-map FILTER-IN permit 20
```

Apply the route map inbound:

```text
R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 route-map FILTER-IN in
```

The behavior is:

| Community on Route | Community-List Result | Route-Map Result |
|---|---|---|
| `65001:100` | Permit (match) | Deny route |
| `65001:200` | No match | Permit through sequence 20 |
| No community | No match | Permit through sequence 20 |

The community list identifies the routes to match.

The route map determines whether those routes are accepted or rejected.

**A community-list permit is not equivalent to a route-map permit.**

---

## Setting Standard Communities

The `set community` command modifies the standard BGP COMMUNITIES attribute.

It supports three main operations:

1. Replace existing communities.
2. Add communities while preserving existing values.
3. Remove all standard communities.

### Replacing Communities

```text
R1(config)# route-map SET-COMMUNITY permit 10
R1(config-route-map)# set community 65001:100
```

Suppose the route initially carries:

```text
65001:200 65002:300
```

After the route map:

```text
65001:100
```

The original standard communities are replaced.

### Adding Communities with `additive`

```text
R1(config)# route-map SET-COMMUNITY permit 10
R1(config-route-map)# set community 65001:100 additive
```

Before:

```text
65001:200 65002:300
```

After:

```text
65001:100 65001:200 65002:300
```

The new community is added while preserving the existing values.

**Important:** The `additive` keyword is necessary when existing standard communities must be preserved.

### Setting Multiple Communities

Multiple values can be specified:

```text
R1(config-route-map)# set community 65001:100 65001:200 additive
```

Both communities are added.

Without `additive`, they replace the existing standard community set.

### Setting Well-Known Communities

Well-known communities can be configured using their keywords:

```text
R1(config-route-map)# set community no-export additive
```

Other examples include:

```text
R1(config-route-map)# set community no-advertise additive
R1(config-route-map)# set community local-as additive
```

These communities impose their predefined BGP advertisement restrictions.

### Removing All Standard Communities

Use:

```text
R1(config)# route-map CLEAR-COMMUNITIES permit 10
R1(config-route-map)# set community none
```

This removes the standard COMMUNITIES attribute from matching routes.

**Important:** `set community none` removes standard communities, not extended communities.

---

## Selectively Deleting Standard Communities

The `set comm-list ... delete` command removes individual standard communities identified by a community list.

Unlike `set community none`, it does not necessarily remove every standard community.

The syntax is:

```text
R1(config-route-map)# set comm-list <community-list> delete
```

### Example: Removing One Community

Suppose a route carries:

```text
65001:100 65001:200 65002:300
```

The requirement is to remove `65001:100` while preserving the other values.

First, configure a community list:

```text
R1(config)# ip community-list standard REMOVE permit 65001:100
```

Then configure a route map:

```text
R1(config)# route-map DELETE-COMMUNITY permit 10
R1(config-route-map)# set comm-list REMOVE delete
```

The result is:

```text
65001:200 65002:300
```

Only the matched community is removed.

### Example: Removing Multiple Communities

Suppose the route contains:

```text
65001:100 65001:200 65001:300 65002:400
```

The requirement is to remove both `65001:100` and `65001:200`.

Configure separate community-list entries:

```text
R1(config)# ip community-list standard REMOVE permit 65001:100
R1(config)# ip community-list standard REMOVE permit 65001:200
```

Then:

```text
R1(config)# route-map DELETE-COMMUNITIES permit 10
R1(config-route-map)# set comm-list REMOVE delete
```

The result is:

```text
65001:300 65002:400
```

### Important: One Community per Entry

When using a standard community list with `set comm-list ... delete`, **each permit entry should contain only one community**.

Use:

```text
R1(config)# ip community-list standard REMOVE permit 65001:100
R1(config)# ip community-list standard REMOVE permit 65001:200
```

Do not rely on:

```text
R1(config)# ip community-list standard REMOVE permit 65001:100 65001:200
```

Although the second entry is valid for ordinary community matching, Cisco documents that this format does not work as intended for deleting multiple individual communities.

For selective deletion, use separate permit entries.

### Deleting Communities Using an Expanded List

Expanded community lists can delete groups of communities using regular expressions.

For example, remove communities matching a particular administrative pattern:

```text
R1(config)# ip community-list expanded REMOVE-OLD permit 65001:.*
```

Configure the route map:

```text
R1(config)# route-map DELETE-OLD permit 10
R1(config-route-map)# set comm-list REMOVE-OLD delete
```

This removes communities matching the regular expression.

The expression should be carefully constructed to avoid accidentally removing unrelated community values.

Expanded lists also support deny entries to exclude particular values from deletion.

---

## Combining `set comm-list` and `set community`

The two commands can be used in the same route-map sequence.

This allows communities to be selectively removed and new values to be assigned.

**Important: Cisco IOS XE performs the `set comm-list ... delete` operation before the `set community` operation, regardless of their displayed configuration order.**

### Example: Replacing One Community

Suppose a route carries:

```text
65001:100 65001:200 65002:300
```

The requirement is to:

- Remove `65001:100`.
- Add `65001:400`.
- Preserve the other communities.

Configure:

```text
R1(config)# ip community-list standard OLD permit 65001:100

R1(config)# route-map REPLACE-COMMUNITY permit 10
R1(config-route-map)# set comm-list OLD delete
R1(config-route-map)# set community 65001:400 additive
```

The operations are:

**Step 1: Delete matching communities**

Before:

```text
65001:100 65001:200 65002:300
```

After deletion:

```text
65001:200 65002:300
```

**Step 2: Add the new community**

Final result:

```text
65001:200 65001:400 65002:300
```

The `additive` keyword preserves the communities remaining after deletion.

### What Happens Without `additive`?

Consider:

```text
R1(config)# route-map REPLACE-COMMUNITY permit 10
R1(config-route-map)# set comm-list OLD delete
R1(config-route-map)# set community 65001:400
```

The deletion operation happens first.

However, the subsequent `set community` command replaces the remaining standard community set.

The final result is:

```text
65001:400
```

Therefore, when the objective is to replace only selected communities while preserving others, use:

```text
set comm-list OLD delete
set community 65001:400 additive
```

**Key distinction:** `set comm-list` selects existing communities to remove. `set community` assigns the new values.

---

## Setting Extended Communities

Cisco IOS XE supports the `set extcommunity` command for assigning supported extended-community types.

Common examples include:

```text
set extcommunity rt <value>
set extcommunity soo <value>
set extcommunity cost <community-id> <cost>
```

### Setting a Route Target

```text
R1(config)# route-map SET-RT permit 10
R1(config-route-map)# set extcommunity rt 65001:100
```

This assigns RT `65001:100`.

Without `additive`, existing RT values are replaced.

### Adding a Route Target

```text
R1(config)# route-map ADD-RT permit 10
R1(config-route-map)# set extcommunity rt 65001:100 additive
```

Suppose a route initially carries:

```text
RT:65001:200
```

After the route map:

```text
RT:65001:100 RT:65001:200
```

Both Route Targets are retained.

### Setting Multiple Route Targets

```text
R1(config-route-map)# set extcommunity rt 65001:100 65001:200 additive
```

This adds both values while preserving existing RTs.

### Setting Site-of-Origin

```text
R1(config)# route-map SET-SOO permit 10
R1(config-route-map)# set extcommunity soo 65001:10
```

This assigns the specified Site-of-Origin extended community.

Unlike `set extcommunity rt`, the documented ordinary IOS XE `set extcommunity soo` syntax does not provide an `additive` keyword.

### Setting Cost Communities

```text
R1(config)# route-map SET-COST permit 10
R1(config-route-map)# set extcommunity cost igp 10 100
```

Or:

```text
R1(config-route-map)# set extcommunity cost pre-bestpath 10 100
```

Cost Communities affect BGP best-path selection at the configured Point of Insertion.

See [BGP Cost Communities and Points of Insertion](../../path-selection/best-path/cost-community.md).

---

## Selectively Deleting Extended Communities

The `set extcomm-list ... delete` command removes extended communities matching an extended community list.

The syntax is:

```text
R1(config-route-map)# set extcomm-list <extended-community-list> delete
```

### Example: Deleting One Route Target

Suppose a VPN route carries:

```text
RT:65001:100
RT:65001:200
SoO:65001:10
```

The requirement is to remove RT `65001:100`.

Configure:

```text
R1(config)# ip extcommunity-list standard REMOVE-RT permit rt 65001:100
```

Then:

```text
R1(config)# route-map DELETE-RT permit 10
R1(config-route-map)# set extcomm-list REMOVE-RT delete
```

The resulting extended communities are:

```text
RT:65001:200
SoO:65001:10
```

The unrelated RT and SoO remain.

### Example: Deleting Multiple Route Targets

Suppose a route carries:

```text
RT:65001:100
RT:65001:200
RT:65001:300
```

Configure an extended community list matching the first two RTs:

```text
R1(config)# ip extcommunity-list standard REMOVE-RT permit rt 65001:100
R1(config)# ip extcommunity-list standard REMOVE-RT permit rt 65001:200
```

Then:

```text
R1(config)# route-map DELETE-RT permit 10
R1(config-route-map)# set extcomm-list REMOVE-RT delete
```

The remaining RT is:

```text
RT:65001:300
```

### Deleting a Site-of-Origin

Extended community lists can also match SoO values:

```text
R1(config)# ip extcommunity-list standard REMOVE-SOO permit soo 65001:10

R1(config)# route-map DELETE-SOO permit 10
R1(config-route-map)# set extcomm-list REMOVE-SOO delete
```

This removes the matching Site-of-Origin extended community.

**Important:** Removing SoO information can interfere with VPN route loop prevention. Use this type of policy only when the intended behavior is understood.

---

## Combining `set extcomm-list` and `set extcommunity`

A common application is replacing one Route Target with another.

For example:

```text
RT:65001:100
```

Must be replaced by:

```text
RT:65001:400
```

Other RT values must remain unchanged.

### Example: Route Target Rewrite

Suppose the original route carries:

```text
RT:65001:100
RT:65001:200
SoO:65001:10
```

Configure a list identifying the old RT:

```text
R1(config)# ip extcommunity-list standard OLD-RT permit rt 65001:100
```

Create the route map:

```text
R1(config)# route-map REWRITE-RT permit 10
R1(config-route-map)# match extcommunity OLD-RT
R1(config-route-map)# set extcomm-list OLD-RT delete
R1(config-route-map)# set extcommunity rt 65001:400 additive

R1(config)# route-map REWRITE-RT permit 20
```

The operations are:

1. Match VPN routes carrying RT `65001:100`.
2. Delete that RT.
3. Add RT `65001:400`.
4. Preserve the other existing RTs.
5. Permit routes not matching sequence 10 through sequence 20.

The final extended communities are:

```text
RT:65001:200
RT:65001:400
SoO:65001:10
```

### Why `additive` Matters

Without `additive`:

```text
R1(config-route-map)# set extcommunity rt 65001:400
```

The command replaces existing Route Target values rather than preserving them.

Therefore, the intended replacement of only one RT requires:

```text
set extcomm-list OLD-RT delete
set extcommunity rt 65001:400 additive
```

Cisco documents this combination for Route Target rewriting in BGP VPN policies.

**Important:** Route Target rewriting can change which VRFs import a route. Always verify the resulting VPN routing table and import/export policies.

---

## `set community` vs. `set comm-list`

The following table summarizes the difference.

| Command | Operation | Effect on Existing Communities |
|---|---|---|
| `set community 65001:100` | Replace | Removes existing standard communities and assigns the specified value |
| `set community 65001:100 additive` | Add | Preserves existing standard communities |
| `set community none` | Clear all | Removes the standard COMMUNITIES attribute |
| `set comm-list LIST delete` | Selective deletion | Removes only standard communities selected by the list |

For example, given:

```text
65001:100 65001:200 65002:300
```

The results are:

| Command | Result |
|---|---|
| `set community 65001:400` | `65001:400` |
| `set community 65001:400 additive` | `65001:100 65001:200 65001:400 65002:300` |
| `set community none` | No standard communities |
| `set comm-list REMOVE-100 delete` | `65001:200 65002:300`, assuming the list permits `65001:100` |

---

## `set extcommunity` vs. `set extcomm-list`

The corresponding extended-community operations are:

| Command | Operation |
|---|---|
| `set extcommunity rt 65001:100` | Set/replace Route Targets |
| `set extcommunity rt 65001:100 additive` | Add a Route Target while preserving existing RTs |
| `set extcommunity soo 65001:10` | Assign a Site-of-Origin |
| `set extcommunity cost igp 10 100` | Assign a Cost Community |
| `set extcomm-list LIST delete` | Delete extended communities selected by an extended community list |

The `set extcommunity` command operates on particular extended-community types.

The `set extcomm-list ... delete` command removes selected existing extended communities.

There is no need to remove every extended community when only one RT or SoO must be deleted.

---

## Inbound vs. Outbound Community Manipulation

Community-related route maps can operate in either direction.

### Inbound Manipulation

An inbound route map processes routes received from a neighbor.

For example:

```text
R1(config)# route-map TAG-IN permit 10
R1(config-route-map)# set community 65001:100 additive

R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 route-map TAG-IN in
```

Received routes are tagged with community `65001:100`.

The modified communities are associated with the accepted BGP paths.

They can subsequently be used for local routing policies and propagated to eligible neighbors.

### Outbound Manipulation

An outbound route map modifies routes advertised to a particular neighbor.

For example:

```text
R1(config)# route-map TAG-OUT permit 10
R1(config-route-map)# set community 65001:200 additive

R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 route-map TAG-OUT out
R1(config-router-af)# neighbor 10.0.12.2 send-community standard
```

Routes advertised to the neighbor are tagged with community `65001:200`.

However, the outbound route map does not directly modify the community values stored in the local BGP table.

**Important:** An outbound route map can modify the attributes sent to one neighbor without changing the attributes advertised to other neighbors.

---

## Advertising Communities to Neighbors

Configuring communities locally does not guarantee they will be advertised.

For ordinary IOS XE IPv4-unicast BGP, community advertisement must be enabled for the neighbor.

### Standard Communities

```text
R1(config-router-af)# neighbor 10.0.12.2 send-community standard
```

The following abbreviated command also enables standard community advertisement:

```text
R1(config-router-af)# neighbor 10.0.12.2 send-community
```

### Extended Communities

```text
R1(config-router-af)# neighbor 10.0.12.2 send-community extended
```

### Both Types

```text
R1(config-router-af)# neighbor 10.0.12.2 send-community both
```

| Command | Community Types Advertised |
|---|---|
| `send-community` | Standard |
| `send-community standard` | Standard |
| `send-community extended` | Extended |
| `send-community both` | Standard and extended |

**Important:** The command is configured on the advertising router.

For example, if R1 must send communities to R2, configure `send-community` on R1 for its neighbor R2.

Also, community advertisement does not override well-known restrictions such as `no-export` or `no-advertise`.

---

## Complete Example: Community-Based Inbound Filtering

Consider:

```text
          AS 65001                 AS 65002

             R1 --------------------- R2
          10.0.12.1               10.0.12.2
```

R2 advertises three routes to R1.

| Prefix | Community |
|---|---|
| `192.0.2.0/24` | `65002:100` |
| `198.51.100.0/24` | `65002:200` |
| `203.0.113.0/24` | `65002:300` |

Requirements:

1. Reject routes carrying `65002:300`.
2. Set LOCAL_PREF 200 on routes carrying `65002:100`.
3. Remove community `65002:200` from accepted routes.
4. Permit all other routes.

### Step 1: Configure Community Lists

```text
R1(config)# ip bgp-community new-format

R1(config)# ip community-list standard REJECT permit 65002:300
R1(config)# ip community-list standard PREFERRED permit 65002:100
R1(config)# ip community-list standard REMOVE permit 65002:200
```

### Step 2: Configure the Route Map

```text
R1(config)# route-map FROM-R2 deny 10
R1(config-route-map)# match community REJECT

R1(config)# route-map FROM-R2 permit 20
R1(config-route-map)# match community PREFERRED
R1(config-route-map)# set local-preference 200

R1(config)# route-map FROM-R2 permit 30
R1(config-route-map)# set comm-list REMOVE delete
```

The route map is evaluated as follows:

**Sequence 10**

- Matches routes carrying `65002:300`.
- Rejects those routes.

**Sequence 20**

- Matches routes carrying `65002:100`.
- Sets LOCAL_PREF 200.
- Permits those routes.

**Sequence 30**

- Matches remaining routes because no match condition is configured.
- Removes community `65002:200` if present.
- Permits the routes.

### Step 3: Apply the Route Map

```text
R1(config)# router bgp 65001
R1(config-router)# neighbor 10.0.12.2 remote-as 65002
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 activate
R1(config-router-af)# neighbor 10.0.12.2 route-map FROM-R2 in
```

R2 must enable standard community advertisement toward R1:

```text
R2(config)# router bgp 65002
R2(config-router)# address-family ipv4 unicast
R2(config-router-af)# neighbor 10.0.12.1 send-community standard
```

### Expected Result

| Prefix | Accepted? | LOCAL_PREF | Community After Policy |
|---|---|---|---|
| `192.0.2.0/24` | Yes | 200 | `65002:100` |
| `198.51.100.0/24` | Yes | 100 (default) | Removed |
| `203.0.113.0/24` | No | N/A | N/A |

**Important:** The route-map sequences terminate processing when a matching permit or deny sequence is reached, unless a supported `continue` action explicitly changes the processing flow.

A route matching sequence 20 does not automatically continue into sequence 30.

---

## Complete Example: Route Target Rewrite

Consider a VPN route carrying:

```text
RT:65000:100
RT:65000:200
SoO:65000:10
```

The requirement is to replace RT `65000:100` with RT `65000:300`, while preserving the other extended communities.

### Step 1: Configure an Extended Community List

```text
PE1(config)# ip extcommunity-list standard OLD-RT permit rt 65000:100
```

### Step 2: Configure the Route Map

```text
PE1(config)# route-map REWRITE-RT permit 10
PE1(config-route-map)# match extcommunity OLD-RT
PE1(config-route-map)# set extcomm-list OLD-RT delete
PE1(config-route-map)# set extcommunity rt 65000:300 additive

PE1(config)# route-map REWRITE-RT permit 20
```

### Step 3: Apply to a VPNv4 Neighbor

For example, on a supported PE router:

```text
PE1(config)# router bgp 65000
PE1(config-router)# address-family vpnv4
PE1(config-router-af)# neighbor 10.0.12.2 route-map REWRITE-RT out
```

Assume the VPNv4 neighbor relationship is otherwise configured.

### Expected Result

Before:

```text
RT:65000:100
RT:65000:200
SoO:65000:10
```

After:

```text
RT:65000:200
RT:65000:300
SoO:65000:10
```

This demonstrates the difference between:

- `match extcommunity`: Identifies the routes to process.
- `set extcomm-list ... delete`: Removes selected existing extended communities.
- `set extcommunity rt ... additive`: Adds new Route Targets while preserving remaining RTs.

---

## Verification Commands

### Display Standard Communities

```text
R1# show bgp ipv4 unicast 192.0.2.0/24
```

Example output:

```text
BGP routing table entry for 192.0.2.0/24
Paths: (1 available, best #1)
  65002
    10.0.12.2 from 10.0.12.2
      Origin IGP, localpref 200, valid, external, best
      Community: 65002:100
```

### Display Routes Matching a Community List

```text
R1# show ip bgp community-list PREFERRED
```

### Display Standard Community Lists

```text
R1# show ip community-list
R1# show ip community-list PREFERRED
```

### Display Extended Community Lists

```text
R1# show ip extcommunity-list
R1# show ip extcommunity-list OLD-RT
```

### Display VPN Route Extended Communities

```text
PE1# show bgp vpnv4 unicast all
```

For a specific VPNv4 route:

```text
PE1# show bgp vpnv4 unicast rd 65000:1 192.0.2.0/24
```

The detailed output can show Route Targets, Site-of-Origin, and other extended communities.

### Display Route Maps

```text
R1# show route-map
R1# show route-map FROM-R2
```

### Verify Community Advertisement

```text
R1# show running-config | section router bgp
```

Check the applicable `neighbor ... send-community` configuration.

### Reapply Inbound Policies

```text
R1# clear ip bgp 10.0.12.2 soft in
```

### Reapply Outbound Policies

```text
R1# clear ip bgp 10.0.12.2 soft out
```

These commands can reapply policies without resetting the established session when the required BGP capabilities are available.

---

## Troubleshooting

| Problem | Possible Cause |
|---|---|
| `match community` never matches | Community absent, incorrect list, or `send-community` missing on sender |
| `match extcommunity` never matches | Wrong RT/SoO type, absent extended community, or incorrect extended community list |
| Route unexpectedly rejected | Route-map deny sequence or implicit deny |
| Community-list deny does not reject route | Deny prevents the community-list match; route-map action determines route filtering |
| Existing communities disappear | `set community` used without `additive` |
| `set comm-list ... delete` does not delete expected values | Community-list entry contains multiple values or does not match |
| Unexpected communities removed | Expanded community-list regex is too broad |
| RTs disappear during rewriting | `set extcommunity rt` used without `additive` |
| RT rewrite does not occur | Route fails `match extcommunity` or the route map is not applied correctly |
| Community present locally but missing remotely | Community advertisement not enabled or an outbound policy modifies it |
| Community change does not affect best-path selection | Community has no automatic preference effect, or a policy does not modify the relevant path attribute |

### Troubleshooting Checklist

1. Verify the communities carried by the route.
2. Verify whether the intended value is a standard or extended community.
3. Examine the relevant community list.
4. Check permit/deny entry ordering.
5. Verify whether all required community values are present.
6. Check `exact-match` behavior for standard community matching.
7. Verify the route-map sequence and permit/deny action.
8. Check for missing `additive` keywords.
9. Check whether `set comm-list` or `set extcomm-list` removes values needed by later policies.
10. Verify the direction in which the route map is applied.
11. Verify community advertisement to downstream neighbors.
12. Reapply policies using an appropriate soft clear when necessary.

---

## Command Summary

| Command | Purpose |
|---|---|
| `ip bgp-community new-format` | Display and configure standard communities in AA:NN format |
| `ip community-list standard <name> permit <values>` | Define a standard community list |
| `ip community-list expanded <name> permit <regex>` | Define an expanded community list |
| `ip extcommunity-list standard <name> permit rt <value>` | Match a specific Route Target |
| `ip extcommunity-list standard <name> permit soo <value>` | Match a specific Site-of-Origin |
| `match community <list>` | Match standard communities |
| `match community <list> exact-match` | Require an exact standard community-set match |
| `match extcommunity <list>` | Match extended communities |
| `set community <values>` | Replace standard communities |
| `set community <values> additive` | Add standard communities |
| `set community none` | Remove all standard communities |
| `set comm-list <list> delete` | Remove selected standard communities |
| `set extcommunity rt <values>` | Replace existing Route Targets |
| `set extcommunity rt <values> additive` | Add Route Targets |
| `set extcommunity soo <value>` | Assign Site-of-Origin |
| `set extcommunity cost ...` | Assign Cost Communities |
| `set extcomm-list <list> delete` | Remove selected extended communities |
| `neighbor <IP> send-community standard` | Advertise standard communities |
| `neighbor <IP> send-community extended` | Advertise extended communities |
| `neighbor <IP> send-community both` | Advertise both types |

---

## Key Points

- Community lists define values to match; they do not modify routes by themselves.
- `match community` matches standard communities.
- `match extcommunity` matches extended communities.
- Multiple community values in a standard community-list entry form a logical AND condition.
- Separate community-list entries provide OR matching with first-match processing.
- `exact-match` requires the complete standard community set to match.
- `set community` assigns or replaces standard communities.
- `set community ... additive` preserves existing standard communities.
- `set community none` removes all standard communities.
- `set comm-list ... delete` selectively removes standard communities.
- When deleting multiple standard communities, configure separate permit entries.
- `set comm-list ... delete` executes before `set community` when both appear in the same route-map sequence.
- `set extcommunity rt` assigns Route Targets.
- `set extcommunity rt ... additive` preserves existing RTs.
- `set extcomm-list ... delete` selectively removes extended communities.
- `set comm-list` and `set extcomm-list` are deletion mechanisms, not community assignment mechanisms.
- Inbound route maps modify attributes of accepted received routes.
- Outbound route maps modify attributes sent to neighbors without directly changing the locally stored BGP path.
- Community advertisement is controlled using `neighbor ... send-community`.
- A community-list deny is not equivalent to a route-map deny.
- Community filtering and modification can be combined in a single route map.

---

## References

- [Cisco: BGP Command Reference, match community and match extcommunity](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_bgp/command/irg-cr-book/bgp-m1.html)
- [Cisco: BGP Command Reference, set community and set comm-list delete](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_bgp/command/irg-cr-book/bgp-n1.html)
- [Cisco: BGP Command Reference, Community Lists](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_bgp/command/irg-cr-book/bgp-c1.html)
- [Cisco: BGP Named Community Lists](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_bgp/configuration/xe-3e/irg-iproute-bgp-xe-3e-book/irg-named-community-list.html)
- [Cisco: MPLS VPN Route Target Rewrite](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/mp_l3_vpns/configuration/xe-3s/mp-l3-vpns-xe-3s-book/mp-vpn-rte-target-rw.html)
- [RFC 1997: BGP Communities Attribute](https://datatracker.ietf.org/doc/html/rfc1997)
- [RFC 4360: BGP Extended Communities Attribute](https://datatracker.ietf.org/doc/html/rfc4360)
