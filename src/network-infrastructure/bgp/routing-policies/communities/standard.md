# Standard BGP Communities

**Standard BGP communities** are 32-bit values used to tag routes so that routing policies can identify and process groups of routes.

They are carried in the **COMMUNITIES path attribute (Type 8)**, which is an optional transitive BGP attribute.

A route can carry multiple standard communities.

Standard communities are used for:

- Route classification and filtering.
- Manipulating attributes such as LOCAL_PREF and MED.
- Controlling route advertisement.
- Influencing inbound traffic through service-provider policies.
- Identifying routes learned from particular customers or locations.

Two important categories are:

1. **Well-known communities:** Have predefined meanings recognized by BGP implementations.
2. **Administratively defined communities:** Have meanings assigned by network administrators.

---

## Standard Community Format

A standard BGP community is a **32-bit value**.

Communities are commonly represented using the following format:

```text
    16 bits          16 bits
+---------------+---------------+
|      AA       |      NN       |
+---------------+---------------+
```

The format is written as:

```text
AA:NN
```

| Field | Size | Description |
|---|---|---|
| AA | 16 bits | Typically identifies the AS defining the community |
| NN | 16 bits | Identifies a specific route category or policy |

Each field has a range of 0 to 65535.

For example:

```text
65001:100
65001:200
65002:300
```

The meaning of these values is administratively defined.

For example, AS 65001 might use:

| Community | Meaning |
|---|---|
| `65001:100` | Customer routes |
| `65001:200` | Peer routes |
| `65001:300` | Backup routes |

BGP does not inherently understand these meanings. A routing policy must be configured to interpret the communities.

### Decimal and AA:NN Formats

Standard communities can be represented as a single 32-bit decimal number or as two 16-bit values.

The conversion is:

```text
Community = (AA × 65536) + NN
```

For example:

```text
65001:100 = 4259905636
```

Configure IOS XE to display and accept communities in the AA:NN format:

```text
R1(config)# ip bgp-community new-format
```

This changes the representation of communities, not their values or BGP wire format.

**Important:** Because the AA field is only 16 bits, standard communities cannot directly encode a 4-byte ASN greater than 65535 using the conventional ASN:NN representation.

BGP Large Communities address this limitation using a 96-bit format with a 32-bit Global Administrator field.

---

## Well-Known Standard Communities

Well-known communities have predefined meanings recognized by BGP implementations.

The most important communities are:

| Community | Hexadecimal Value | AA:NN Equivalent |
|---|---|---|
| `internet` | `0x00000000` | `0:0` |
| `no-export` | `0xFFFFFF01` | `65535:65281` |
| `no-advertise` | `0xFFFFFF02` | `65535:65282` |
| `local-as` | `0xFFFFFF03` | `65535:65283` |

The `local-as` community is also called **NO_EXPORT_SUBCONFED**.

These communities are particularly important because `no-export`, `no-advertise`, and `local-as` impose route advertisement restrictions without requiring an explicit community-matching route map.

### INTERNET Community

The `internet` community identifies the general Internet community.

All BGP speakers logically belong to this community by default.

```text
R1(config-route-map)# set community internet
```

Unlike the other well-known communities, `internet` does not impose a route advertisement restriction.

It also does not override other restrictions or normal BGP advertisement rules.

**Important:** A route does not need to explicitly carry community `0:0` to be eligible for normal BGP advertisement. By default, routes may have no COMMUNITIES attribute at all.

### NO_EXPORT Community

The `no-export` community prevents a route from being advertised **outside the local AS or BGP confederation**.

A router receiving a route with `no-export` can advertise it to:

- IBGP neighbors.
- Confederation EBGP neighbors.

However, it cannot advertise the route to ordinary EBGP neighbors outside the AS or confederation.

#### Example

```text
                  AS 65001
           +-------------------+
           |                   |
          R1 ---------------- R2 ---------------- R3
           |                   |                    |
           |      IBGP         |       EBGP         |
           |                   |                    |
           +-------------------+                AS 65002
```

R1 advertises `192.0.2.0/24` to R2 with the `no-export` community.

R2:

1. Receives the route.
2. Can select and install the route normally.
3. Can advertise the route to other eligible IBGP neighbors.
4. **Cannot advertise the route to R3**, because R3 belongs to another AS.

The community does not prevent R2 from using the route locally.

#### Configuration

On R1:

```text
R1(config)# route-map SET-NO-EXPORT permit 10
R1(config-route-map)# set community no-export additive

R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 route-map SET-NO-EXPORT out
R1(config-router-af)# neighbor 10.0.12.2 send-community standard
```

This tags routes advertised to R2 with `no-export`, while preserving their existing communities.

**Important:** A common use of `no-export` is for a customer to advertise a route to an ISP while requesting that the ISP not advertise it beyond the ISP's AS. The customer may attach the community in its outbound policy; the receiving ISP then enforces the restriction on subsequent advertisements.

### NO_ADVERTISE Community

The `no-advertise` community prevents a route from being advertised to **any BGP neighbor**.

This includes:

- IBGP neighbors.
- Ordinary EBGP neighbors.
- Confederation EBGP neighbors.

#### Example

```text
                 AS 65001
          +------------------+
          |                  |
         R1 ---------------- R2 ---------------- R3
          |                  |                    |
          |      IBGP        |        EBGP        |
          |                  |                    |
          +------------------+                AS 65002
```

R1 advertises `192.0.2.0/24` to R2 with `no-advertise`.

R2:

1. Receives the route.
2. Can select and install the route.
3. **Cannot advertise the route to any other BGP neighbor.**

The restriction is stronger than `no-export`.

#### Configuration

```text
R1(config)# route-map SET-NO-ADVERTISE permit 10
R1(config-route-map)# set community no-advertise additive
```

**Important:** `no-advertise` prevents further BGP advertisement. It does not prevent the receiving router from using the route for forwarding, nor does it automatically remove the route from the routing table.

### LOCAL_AS Community

The `local-as` community is primarily used in **BGP confederations**.

It prevents a route from being advertised outside the local member AS (sub-AS) of the confederation.

Unlike `no-export`, it also prevents advertisement to other member ASes of the same confederation.

#### Example

Consider a BGP confederation with identifier 65000 and three member ASes:

```text
                  Confederation 65000

     +-------------------------------------------+
     |                                           |
     |   Sub-AS 65001        Sub-AS 65002        |
     |                                           |
     |      R1 ---------------- R2               |
     |             Confed EBGP                   |
     |                                           |
     +-------------------------------------------+
                                             
                          Ordinary EBGP
                                |
                                |
                               R3
                            AS 65003
```

Suppose R1 has a route tagged with `local-as`.

R1 can advertise it to eligible IBGP neighbors in sub-AS 65001.

However, R1 cannot advertise it to R2 because R2 belongs to a different member AS.

The route also cannot cross the confederation boundary.

#### Configuration

```text
R1(config)# route-map SET-LOCAL-AS permit 10
R1(config-route-map)# set community local-as additive
```

**Important:** `local-as` is a standard BGP community and is unrelated to the `neighbor ... local-as` command used for AS-number manipulation.

---

## Community Propagation Rules

The differences between the well-known communities can be summarized as follows.

| Community | IBGP | Confederation EBGP | Ordinary EBGP |
|---|---|---|---|
| `internet` | Allowed | Allowed | Allowed |
| `no-export` | Allowed | Allowed | Blocked |
| `no-advertise` | Blocked | Blocked | Blocked |
| `local-as` | Allowed | Blocked | Blocked |

These are community-specific advertisement restrictions. Other BGP rules and policies still apply.

**Key distinction:**

- `no-export`: Do not advertise outside the AS or confederation.
- `local-as`: Do not advertise outside the local member AS.
- `no-advertise`: Do not advertise to any BGP neighbor.

### Multiple Well-Known Communities

A BGP route can carry multiple communities simultaneously.

For example:

```text
Community: 65001:100 no-export no-advertise
```

In this case, the most restrictive applicable behavior determines whether the route can be advertised.

The presence of `no-advertise` prevents advertisement to all BGP peers, even though `no-export` would otherwise permit advertisement to IBGP peers.

Communities are not mutually exclusive.

---

## Additional Well-Known Communities

Other standardized communities provide functionality beyond the basic advertisement restrictions.

### GRACEFUL_SHUTDOWN

The **GRACEFUL_SHUTDOWN (GSHUT)** community is defined in RFC 8326.

Its value is:

```text
65535:0
```

It is used to prepare for planned BGP session shutdowns.

The objective is to allow neighboring routers to prefer alternative paths **before** a BGP session is terminated, reducing packet loss during convergence.

The general operation is:

1. A router preparing for maintenance advertises the GRACEFUL_SHUTDOWN community.
2. Receiving routers recognize the community.
3. A configured or supported policy reduces the preference of the affected paths.
4. Traffic shifts toward alternative paths, if available.
5. The BGP session is shut down after convergence.

A receiving router might use:

```text
R1(config)# ip community-list standard GSHUT permit 65535:0

R1(config)# route-map GSHUT-IN permit 10
R1(config-route-map)# match community GSHUT
R1(config-route-map)# set local-preference 0

R1(config)# route-map GSHUT-IN permit 20

R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 route-map GSHUT-IN in
```

Routes carrying the GSHUT community receive LOCAL_PREF 0.

Other routes are permitted unchanged.

**Important:** The community itself is a signal. The receiving router must recognize and act on it, either through configured policy or supported automatic behavior.

Cisco IOS XE also supports BGP graceful-shutdown features that automate parts of this procedure.

### BLACKHOLE

The **BLACKHOLE** community is defined in RFC 7999.

Its value is:

```text
65535:666
```

It is commonly used for remotely triggered blackholing (RTBH).

For example, a customer experiencing a DDoS attack might advertise a more-specific prefix with the BLACKHOLE community to its service provider.

The provider can then discard traffic destined for that prefix.

However, **the BLACKHOLE community does not automatically cause traffic to be discarded**.

The receiving network must explicitly support and authorize the corresponding blackholing policy.

Service providers may also use their own community values for blackholing.

---

## Administratively Defined Communities

Unlike well-known communities, administratively defined communities have no universally defined behavior.

For example, an ISP might publish the following community policy:

| Community | Provider Action |
|---|---|
| `65000:100` | Set LOCAL_PREF 100 |
| `65000:200` | Set LOCAL_PREF 200 |
| `65000:300` | Prepend the provider ASN toward selected upstreams |
| `65000:400` | Suppress advertisement toward selected peers |

These values are examples, not standardized community assignments.

A customer can attach the provider's documented communities to its routes to request particular routing policies.

This can provide more control over inbound traffic than AS_PATH prepending alone.

However, the provider determines which community values it supports and how they are interpreted.

**A community has no effect unless the receiving BGP implementation recognizes it or a routing policy acts on it.**

---

## Assigning Standard Communities

Cisco IOS XE uses the `set community` route-map command to assign standard communities.

### Setting a Community

```text
R1(config)# route-map TAG-ROUTES permit 10
R1(config-route-map)# set community 65001:100
```

This replaces the route's existing standard communities with `65001:100`.

### Adding a Community

Use the `additive` keyword to preserve existing communities.

```text
R1(config)# route-map TAG-ROUTES permit 10
R1(config-route-map)# set community 65001:100 additive
```

For example, suppose a route initially carries:

```text
65002:200
```

After applying the route map:

```text
65002:200 65001:100
```

Both communities are retained.

Without `additive`, the original `65002:200` community would be removed.

### Setting Multiple Communities

Multiple communities can be assigned in a single command.

```text
R1(config-route-map)# set community 65001:100 65001:200 additive
```

This adds both communities while preserving existing standard communities.

### Removing All Standard Communities

Use:

```text
R1(config-route-map)# set community none
```

This removes all standard communities from routes processed by the route-map sequence.

It does not remove extended communities, which are carried in a separate BGP attribute.

### Removing Specific Communities

Use `set comm-list ... delete` to remove only communities matching a community list.

For example:

```text
R1(config)# ip community-list standard REMOVE permit 65001:100

R1(config)# route-map DELETE-COMMUNITY permit 10
R1(config-route-map)# set comm-list REMOVE delete
```

This removes community `65001:100`, if present, while preserving other standard communities.

When using a standard community list for deletion, define each community to be removed in a separate permit entry.

Detailed configuration and filtering are covered in [BGP Community Configuration and Filtering](configuration.md).

---

## Matching Standard Communities

Community lists allow routing policies to match standard community values.

Cisco IOS XE supports:

- **Standard community lists:** Match explicitly specified community values.
- **Expanded community lists:** Match communities using regular expressions.

### Standard Community Lists

For example:

```text
R1(config)# ip community-list standard CUSTOMER permit 65001:100
```

This matches routes containing community `65001:100`.

A route map can then apply an action:

```text
R1(config)# route-map PREFER-CUSTOMER permit 10
R1(config-route-map)# match community CUSTOMER
R1(config-route-map)# set local-preference 200

R1(config)# route-map PREFER-CUSTOMER permit 20
```

Routes carrying `65001:100` receive LOCAL_PREF 200.

Other routes are permitted by sequence 20.

### Matching Multiple Communities

Consider:

```text
R1(config)# ip community-list standard BOTH permit 65001:100 65001:200
```

The two values in the same entry form a **logical AND** condition.

A route must carry both communities to match.

For example:

| Route Communities | Match? |
|---|---|
| `65001:100` | No |
| `65001:200` | No |
| `65001:100 65001:200` | Yes |
| `65001:100 65001:200 65001:300` | Yes |

To match either community, configure separate permit entries:

```text
R1(config)# ip community-list standard EITHER permit 65001:100
R1(config)# ip community-list standard EITHER permit 65001:200
```

These entries form a **logical OR** condition.

### Exact Community Matching

By default, a standard community list can match a route that also carries additional communities.

For example, a list permitting `65001:100` matches:

```text
65001:100 65001:200
```

To require an exact match, use the `exact-match` keyword in the route map:

```text
R1(config)# route-map EXACT-MATCH permit 10
R1(config-route-map)# match community CUSTOMER exact-match
```

This matches only routes whose complete standard community set matches the permitted community set.

For a community list permitting only `65001:100`:

| Route Communities | Normal Match | Exact Match |
|---|---|---|
| `65001:100` | Yes | Yes |
| `65001:100 65001:200` | Yes | No |
| `65001:200` | No | No |

**Important:** The `exact-match` keyword applies to the complete standard community set, not just the existence of a particular value.


### Expanded Community Lists

**Expanded community lists** use regular expressions to match standard BGP communities.

Unlike standard community lists, which match explicitly specified community values, expanded community lists can match patterns of values.

For example, an expanded community list can match:

- All communities assigned by a particular AS.
- A range of community values.
- Multiple communities using a single regular expression.

Cisco IOS XE supports:

| Type | Syntax |
|---|---|
| Numbered expanded list | `ip community-list <100-500> permit <regex>` |
| Named expanded list | `ip community-list expanded <name> permit <regex>` |

#### Example 1: Matching a Range of Communities

Match communities `65001:100` through `65001:199`:

```text
R1(config)# ip community-list expanded RANGE permit 65001:1[0-9][0-9]_
```

The regular expression means:

- `65001:` matches the first part of the community.
- `1` matches the first digit of the second part.
- `[0-9][0-9]` matches two additional digits.
- `_` matches a delimiter or boundary, preventing a match against a longer value such as `65001:1000`.

Examples:

| Community | Match? |
|---|---|
| `65001:100` | Yes |
| `65001:150` | Yes |
| `65001:199` | Yes |
| `65001:200` | No |
| `65001:1000` | No |
| `65002:150` | No |

#### Example 2: Matching Communities from a Particular AS

Match communities whose first field is `65001`:

```text
R1(config)# ip community-list expanded AS65001 permit 65001:[0-9]+_
```

The expression `[0-9]+` matches one or more decimal digits.

Examples:

| Community | Match? |
|---|---|
| `65001:100` | Yes |
| `65001:200` | Yes |
| `65001:3000` | Yes |
| `65002:100` | No |

This is useful when an AS defines numerous communities and the policy should match all of them.

#### Example 3: Using an Expanded Community List in a Route Map

Match communities `65001:100` through `65001:199` and assign LOCAL_PREF 200.

```text
R1(config)# ip community-list expanded PREFERRED permit 65001:1[0-9][0-9]_

R1(config)# route-map SET-LOCAL-PREF permit 10
R1(config-route-map)# match community PREFERRED
R1(config-route-map)# set local-preference 200

R1(config)# route-map SET-LOCAL-PREF permit 20

R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 route-map SET-LOCAL-PREF in
```

Routes carrying a matching community receive LOCAL_PREF 200.

Other routes are permitted unchanged by sequence 20.

#### Regular Expression Characters

Expanded community lists support Cisco IOS regular expressions.

| Character | Meaning |
|---|---|
| `.` | Match any single character |
| `*` | Match zero or more occurrences |
| `+` | Match one or more occurrences |
| `?` | Match zero or one occurrence |
| `[0-9]` | Match one decimal digit |
| `[1-5]` | Match a digit from 1 to 5 |
| `^` | Match the beginning of the input string |
| `$` | Match the end of the input string |
| `_` | Match a boundary or delimiter |
| `\|` | Logical OR (alternation) |
| `( )` | Group expressions |

**Important:** Expanded community-list regular expressions are matched against the community attribute's string representation. A route may carry multiple communities, so be careful when using `^` and `$` to anchor expressions.

For matching exact community values, standard community lists are generally simpler and easier to maintain.

#### Permit and Deny Behavior

Like standard community lists, expanded community lists support `permit` and `deny` statements.

For example:

```text
R1(config)# ip community-list expanded FILTER deny 65001:150_
R1(config)# ip community-list expanded FILTER permit 65001:1[0-9][0-9]_
```

A route carrying `65001:150` fails the community-list match, while routes carrying other values in the specified range can match.

**Important:** A `deny` statement in a community list means the community-list match fails. It does not independently deny the BGP route.

The final action depends on the route-map configuration.

#### Verification

Display the configured expanded community list:

```text
R1# show ip community-list FILTER
```

Display BGP routes matching the list:

```text
R1# show ip bgp community-list FILTER
```

Expanded community lists are particularly useful when a routing policy needs to match a large range of community values without individually listing each value.

---

## Advertising Communities

Assigning a community locally does not automatically cause it to be included in outgoing BGP UPDATE messages.

On Cisco IOS XE, standard community advertisement must be enabled for the neighbor.

```text
R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 send-community standard
```

The following abbreviated form also enables standard community advertisement:

```text
R1(config-router-af)# neighbor 10.0.12.2 send-community
```

To advertise both standard and extended communities:

```text
R1(config-router-af)# neighbor 10.0.12.2 send-community both
```

**Important distinctions:**

- `set community` assigns or changes communities.
- `send-community` permits their transmission to a neighbor.
- `match community` identifies routes based on communities.
- `ip community-list` defines which community values to match.

The `send-community` command does not override well-known advertisement restrictions such as `no-export` or `no-advertise`.

---

## Example: Controlling Route Advertisement with NO_EXPORT

Consider the following topology:

```text
      AS 65001             AS 65002             AS 65003

         R1 ------------------ R2 ------------------ R3
            10.0.12.0/30          10.0.23.0/30
```

R1 originates two prefixes:

- `192.0.2.0/24`
- `198.51.100.0/24`

The requirements are:

1. Both routes must be advertised to R2.
2. R2 must not advertise `192.0.2.0/24` to R3.
3. R2 can advertise `198.51.100.0/24` to R3.

### R1 Configuration

Assume both prefixes exist in R1's routing table.

Configure a prefix list matching the restricted route:

```text
R1(config)# ip prefix-list RESTRICTED permit 192.0.2.0/24
```

Configure an outbound route map:

```text
R1(config)# route-map SET-COMMUNITY permit 10
R1(config-route-map)# match ip address prefix-list RESTRICTED
R1(config-route-map)# set community no-export additive

R1(config)# route-map SET-COMMUNITY permit 20
```

Sequence 10 adds `no-export` to `192.0.2.0/24`.

Sequence 20 permits all remaining eligible routes without modifying their communities.

Configure BGP:

```text
R1(config)# router bgp 65001
R1(config-router)# neighbor 10.0.12.2 remote-as 65002
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 activate
R1(config-router-af)# network 192.0.2.0 mask 255.255.255.0
R1(config-router-af)# network 198.51.100.0 mask 255.255.255.0
R1(config-router-af)# neighbor 10.0.12.2 route-map SET-COMMUNITY out
R1(config-router-af)# neighbor 10.0.12.2 send-community standard
```

### R2 Configuration

Configure R2's EBGP neighbors normally:

```text
R2(config)# router bgp 65002
R2(config-router)# neighbor 10.0.12.1 remote-as 65001
R2(config-router)# neighbor 10.0.23.3 remote-as 65003
R2(config-router)# address-family ipv4 unicast
R2(config-router-af)# neighbor 10.0.12.1 activate
R2(config-router-af)# neighbor 10.0.23.3 activate
```

No special community-filtering route map is required on R2.

R2 automatically recognizes the well-known `no-export` community and enforces its advertisement restriction.

### Expected Result

On R2:

```text
R2# show bgp ipv4 unicast 192.0.2.0/24
```

The route should carry `no-export`.

R2 can still select and install the route.

However, when checking routes advertised to R3:

```text
R2# show bgp ipv4 unicast neighbors 10.0.23.3 advertised-routes
```

The expected result is:

| Prefix | Community | Advertised to R3? |
|---|---|---|
| `192.0.2.0/24` | `no-export` | No |
| `198.51.100.0/24` | None | Yes |

This assumes both routes are otherwise eligible for BGP advertisement.

**The key point is that the `no-export` community controls advertisement without requiring an explicit outbound filtering policy on R2.**

---

## Communities and Route Aggregation

BGP aggregation can affect the communities carried by an aggregate route.

On Cisco IOS XE, the `aggregate-address` command supports the `as-set` keyword.

When `as-set` is used, BGP can preserve information from contributing routes, including their communities, in the aggregate.

For example:

```text
R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# aggregate-address 192.0.2.0 255.255.254.0 as-set
```

If component routes carry different communities, the aggregate can inherit communities from those routes.

This can be important when component routes contain well-known communities such as `no-export`.

An aggregate that inherits a restrictive community may be advertised less widely than intended.

Community manipulation using an `attribute-map` can be used to control aggregate attributes.

See [BGP Aggregation Configuration](../../scalability/aggregation/configuration.md).

---

## Verification

### Display Communities on a Route

```text
R1# show bgp ipv4 unicast 192.0.2.0/24
```

Example output:

```text
BGP routing table entry for 192.0.2.0/24
Paths: (1 available, best #1)

  65002
    10.0.12.2 from 10.0.12.2
      Origin IGP, localpref 100, valid, external, best
      Community: 65001:100 no-export
```

The `Community:` field identifies the communities carried by the route.

### Display Routes with a Specific Community

```text
R1# show bgp ipv4 unicast community 65001:100
```

To display routes carrying `no-export`:

```text
R1# show ip bgp community no-export
```

To display routes carrying `no-advertise`:

```text
R1# show ip bgp community no-advertise
```

### Display Routes Matching a Community List

```text
R1# show ip bgp community-list CUSTOMER
```

### Display Community Lists

```text
R1# show ip community-list
R1# show ip community-list CUSTOMER
```

### Verify Route Maps

```text
R1# show route-map
R1# show route-map SET-COMMUNITY
```

### Verify Advertisements

```text
R1# show bgp ipv4 unicast neighbors 10.0.12.2 advertised-routes
```

This displays the routes selected for advertisement toward the neighbor.

To verify whether standard communities were actually received, inspect the detailed route on the neighboring router.

### Reapply Policies

After changing an inbound community-based policy:

```text
R1# clear ip bgp 10.0.12.2 soft in
```

After changing an outbound community-based policy:

```text
R1# clear ip bgp 10.0.12.2 soft out
```

These commands can reapply policies without resetting the BGP session.

---

## Troubleshooting

Common problems involving standard BGP communities include:

| Problem | Possible Cause |
|---|---|
| Community missing on receiving router | `send-community` not configured on advertising router |
| Existing communities disappear | `set community` configured without `additive` |
| Route not advertised to an EBGP neighbor | Route carries `no-export`, `local-as`, or `no-advertise` |
| Route not advertised to any neighbor | Route carries `no-advertise` |
| Community policy has no effect | Missing or incorrect community list, route map, or policy attachment |
| Community match unexpectedly succeeds | Route contains the matched community plus additional communities |
| Community match unexpectedly fails | `exact-match` requires an exact community-set match |
| Community value displayed as a large decimal number | `ip bgp-community new-format` not configured |

When troubleshooting, distinguish between:

1. **Community assignment:** Does the route actually carry the expected community?
2. **Community propagation:** Was the community transmitted to the next BGP router?
3. **Community interpretation:** Does the receiving router recognize the community or have a policy matching it?
4. **BGP advertisement eligibility:** Is the route permitted to be advertised under normal BGP rules and the community-specific restrictions?

A route can have the correct community but still fail to produce the expected result because another routing policy or BGP rule takes precedence.

---

## Key Points

- Standard BGP communities are 32-bit values carried in the optional transitive COMMUNITIES attribute.
- They are commonly represented as two 16-bit fields in AA:NN format.
- A route can carry multiple communities.
- `internet` identifies the general Internet community and does not override other BGP policies.
- `no-export` prevents advertisement outside an AS or confederation.
- `no-advertise` prevents advertisement to every BGP neighbor.
- `local-as` prevents advertisement outside the local member AS of a confederation.
- `no-export` and `local-as` differ when advertising routes between member ASes of a confederation.
- `additive` preserves existing communities when adding new ones.
- Without `additive`, `set community` replaces the existing standard communities.
- `set community none` removes all standard communities.
- A standard community-list entry containing multiple values requires all those values to match.
- Separate permit entries allow matching any of the specified communities.
- `match community ... exact-match` requires the complete community set to match.
- `send-community` enables community advertisement to a neighbor.
- Administratively defined communities have no inherent meaning until a routing policy acts on them.
- Well-known communities such as GSHUT and BLACKHOLE support specialized operational procedures.

---

## References

- [Cisco: BGP Named Community Lists](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_bgp/configuration/xe-3e/irg-iproute-bgp-xe-3e-book/irg-named-community-list.html)
- [Cisco: BGP Command Reference, Community Lists](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_bgp/command/irg-cr-book/bgp-c1.html)
- [Cisco: BGP Command Reference, Match Community](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_bgp/command/irg-cr-book/bgp-m1.html)
- [Cisco: BGP Frequently Asked Questions](https://www.cisco.com/c/en/us/support/docs/ip/border-gateway-protocol-bgp/5816-bgpfaq-5816.html)
- [RFC 1997: BGP Communities Attribute](https://datatracker.ietf.org/doc/html/rfc1997)
- [RFC 7999: BLACKHOLE Community](https://datatracker.ietf.org/doc/html/rfc7999)
- [RFC 8326: Graceful BGP Session Shutdown](https://datatracker.ietf.org/doc/html/rfc8326)
