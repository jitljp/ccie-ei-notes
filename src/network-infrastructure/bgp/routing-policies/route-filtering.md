# BGP Route Filtering

**BGP route filtering** allows administrators to control which routes are accepted from BGP neighbors and which routes are advertised to them.

Route filtering is commonly used to:

- Prevent unwanted routes from entering the local BGP table.
- Restrict which prefixes are advertised to external networks.
- Prevent accidental advertisement of internal or unauthorized prefixes.
- Implement routing policies between autonomous systems.
- Control routing information exchanged with different BGP neighbors.

Cisco IOS XE supports several BGP route-filtering mechanisms, including **prefix lists, distribute lists, filter lists, and route maps**.

---

## Inbound vs. Outbound Route Filtering

BGP supports two primary filtering directions:

| Direction | Purpose | Effect |
|---|---|---|
| Inbound (`in`) | Filter routes received from a neighbor | Controls which received routes are accepted into the local BGP table |
| Outbound (`out`) | Filter routes advertised to a neighbor | Controls which routes are advertised to that neighbor |

### Inbound Filtering

Inbound filtering is applied when BGP receives routes from a neighbor.

```text
       R1                          R2
    AS 65001                    AS 65002

       |                           |
       |<------ BGP UPDATE --------|
       |                           |
 +----------------+
 | Inbound Filter |
 +----------------+
       |
       | Permitted routes
       v
    BGP Table
       |
       v
  Best-Path Selection
```

Routes rejected by an inbound filter are not available for the local BGP best-path selection process.

However, inbound filtering **does not prevent the neighbor from sending the routes**.

The neighbor still advertises them, and the receiving router processes the updates before rejecting the unwanted routes.

To prevent unwanted routes from being sent in the first place, filtering must be applied outbound on the advertising router, or a mechanism such as BGP Outbound Route Filtering (ORF) can be used.

### Outbound Filtering

Outbound filtering controls which routes are advertised to a BGP neighbor.

```text
       R1                          R2
    AS 65001                    AS 65002

    BGP Table
       |
       v
 +-----------------+
 | Outbound Filter |
 +-----------------+
       |
       | Permitted routes
       v
       |------ BGP UPDATE -------->|
```

Routes rejected by an outbound filter are not advertised to that neighbor.

**Outbound filtering does not remove routes from the local BGP table or directly affect local best-path selection.**

Outbound policies are applied separately for each neighbor, allowing a router to advertise different sets of prefixes to different peers.

---

## BGP Route-Filtering Mechanisms

Cisco IOS XE provides several filtering methods.

| Mechanism | Command | Matches |
|---|---|---|
| Prefix list | `neighbor ... prefix-list` | Network prefix and prefix length |
| Distribute list | `neighbor ... distribute-list` | IPv4 prefixes using an ACL |
| Filter list | `neighbor ... filter-list` | BGP AS_PATH |
| Route map | `neighbor ... route-map` | Prefixes, attributes, communities, and other conditions |

All four mechanisms support inbound and outbound filtering.

### Prefix Lists

Prefix lists are commonly used when filtering routes based on their network addresses and prefix lengths.

For example:

```text
R1(config)# ip prefix-list ALLOWED permit 10.10.0.0/16 le 24

R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 prefix-list ALLOWED in
```

This permits routes:

- Within `10.10.0.0/16`.
- With prefix lengths between `/16` and `/24`, inclusive.

Other routes are rejected by the prefix list's implicit deny.

Prefix lists can also be applied in the outbound direction:

```text
R1(config-router-af)# neighbor 10.0.12.2 prefix-list ALLOWED out
```

**Prefix lists are generally preferred over standard IPv4 ACLs for prefix-based BGP filtering**, particularly when matching specific prefix-length ranges.

### Distribute Lists

Distribute lists use IPv4 ACLs to filter BGP routes.

For example:

```text
R1(config)# ip access-list standard ALLOWED
R1(config-std-nacl)# permit 10.10.10.0 0.0.0.255

R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 distribute-list ALLOWED in
```

This permits routes whose network addresses match the ACL.

A standard ACL does not independently match the route's prefix length.

For filtering based on both prefix and prefix length, an extended ACL or prefix list can be used.

> **Important:** `neighbor distribute-list` and `neighbor prefix-list` are mutually exclusive for the same neighbor in the same direction. They cannot both be applied inbound, or both applied outbound, to the same neighbor.

### Filter Lists

BGP filter lists use **AS-path access lists** to filter routes based on their AS_PATH.

For example:

```text
R1(config)# ip as-path access-list 10 permit ^65002$

R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 filter-list 10 in
```

This accepts routes whose AS_PATH consists of exactly AS 65002.

Routes with other AS_PATH values are rejected.

AS-path access lists use regular expressions, allowing filtering based on:

- Originating AS.
- Neighboring AS.
- Specific AS sequences.
- Presence or absence of particular ASNs in the path.

### Route Maps

Route maps provide more flexible filtering by combining match conditions.

For example:

```text
R1(config)# ip prefix-list ALLOWED permit 10.10.10.0/24

R1(config)# route-map FILTER-IN permit 10
R1(config-route-map)# match ip address prefix-list ALLOWED

R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 route-map FILTER-IN in
```

Only routes matching the prefix list are accepted.

A route map can also combine filtering with attribute manipulation:

```text
R1(config)# route-map FILTER-IN permit 10
R1(config-route-map)# match ip address prefix-list ALLOWED
R1(config-route-map)# set local-preference 200
```

Matching routes are accepted with LOCAL_PREF 200.

Nonmatching routes are rejected because of the route map's implicit deny.

See [BGP Prefix Lists, AS-Path ACLs, and Route Maps](filtering-tools.md) for detailed matching and configuration behavior.

---

## Route Map Permit and Deny Behavior

When a route map is used for BGP filtering, its sequence actions determine whether matching routes are accepted or rejected.

| Route-map action | Match result | Effect |
|---|---|---|
| `permit` | Match | Accept the route and apply any `set` actions |
| `permit` | No match | Continue to the next sequence |
| `deny` | Match | Reject the route |
| `deny` | No match | Continue to the next sequence |

If no sequence permits the route, the route is rejected by the **implicit deny**.

### Example: Deny Specific Routes, Permit Everything Else

Suppose R1 should reject `10.10.10.0/24` but accept all other routes from R2.

```text
R1(config)# ip prefix-list BLOCKED permit 10.10.10.0/24

R1(config)# route-map FILTER-IN deny 10
R1(config-route-map)# match ip address prefix-list BLOCKED

R1(config)# route-map FILTER-IN permit 20

R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 route-map FILTER-IN in
```

The result is:

1. Routes matching `10.10.10.0/24` are rejected by sequence 10.
2. Other routes do not match sequence 10.
3. Sequence 20 permits all remaining routes because it has no match conditions.

**Important distinction:** The `permit` statement in the prefix list identifies routes that *match* the route map's condition. The `deny` action of the route map then rejects those matching routes.

A route-map `deny` statement and a prefix-list `deny` statement do not have the same meaning.

---

## Inbound Route Filtering Example

Consider the following topology:

```text
           AS 65001                  AS 65002
             R1 ----------------------- R2
         10.0.12.1                  10.0.12.2

                      EBGP
```

R2 advertises three prefixes to R1:

| Prefix | Requirement |
|---|---|
| `203.0.113.0/24` | Accept |
| `198.51.100.0/24` | Accept |
| `10.10.10.0/24` | Reject |

### R1 Configuration

```text
R1(config)# ip prefix-list FROM-R2 seq 10 permit 203.0.113.0/24
R1(config)# ip prefix-list FROM-R2 seq 20 permit 198.51.100.0/24

R1(config)# router bgp 65001
R1(config-router)# neighbor 10.0.12.2 remote-as 65002
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 prefix-list FROM-R2 in
```

### Result

R1 accepts:

```text
203.0.113.0/24
198.51.100.0/24
```

R1 rejects:

```text
10.10.10.0/24
```

Because the prefix list ends with an implicit deny, any other prefixes advertised by R2 are also rejected.

The filtered routes are not available for BGP best-path selection on R1.

However, R2 continues to advertise them because R2 has no corresponding outbound filtering policy.

---

## Outbound Route Filtering Example

Using the same topology, assume R1 has the following prefixes in its BGP table:

```text
192.0.2.0/24
198.51.100.0/24
203.0.113.0/24
```

R1 should advertise only `192.0.2.0/24` to R2.

### R1 Configuration

```text
R1(config)# ip prefix-list TO-R2 permit 192.0.2.0/24

R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 prefix-list TO-R2 out
```

### Result

Provided `192.0.2.0/24` is eligible for advertisement, R1 advertises it to R2.

The other prefixes are rejected by the outbound filter.

However, they remain in R1's BGP table and can still be used for local path selection and advertised to other eligible neighbors.

This allows different outbound policies to be applied to different BGP neighbors.

---

## Combining Multiple BGP Filters

Cisco IOS XE can apply different filtering mechanisms to the same BGP neighbor.

For example, a neighbor can have both:

- An inbound prefix list.
- An inbound AS-path filter list.

```text
R1(config)# ip prefix-list ALLOWED permit 203.0.113.0/24
R1(config)# ip as-path access-list 10 permit ^65002$

R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 prefix-list ALLOWED in
R1(config-router-af)# neighbor 10.0.12.2 filter-list 10 in
```

For a route to be accepted, it must satisfy **both filtering mechanisms**.

In this example:

- The prefix must be `203.0.113.0/24`.
- The AS_PATH must be exactly `65002`.

A route permitted by one filter but rejected by the other is rejected.

### BGP Filter Processing Order

Cisco documents the following processing order for BGP neighbor filtering:

| Inbound | Outbound |
|---|---|
| 1. Route map | 1. Distribute list |
| 2. Filter list / AS-path filter | 2. Prefix list |
| 3. Prefix list | 3. Filter list / AS-path filter |
| 4. Distribute list | 4. Route map |

This ordering can matter when a route map modifies an attribute that another filter evaluates. Behavior can vary with software and supported policy features.

See [Cisco: BGP 4 Prefix Filter and Inbound Route Maps](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_bgp/configuration/xe-3se/3650/irg-xe-3se-3650-book/irg-prefix-filter.html).

---

## Filtering vs. RIB Installation

**Inbound BGP filtering is different from preventing a BGP route from being installed in the global routing table.**

Consider two scenarios.

### Scenario 1: Inbound Filter Rejects a Route

```text
BGP UPDATE
     |
     v
Inbound Filter
     |
   DENY
```

The route is rejected and is not available in the local BGP table for best-path selection.

### Scenario 2: Route Is Accepted but Not Installed

```text
BGP UPDATE
     |
     v
Inbound Filter
     |
   PERMIT
     |
     v
  BGP Table
     |
     v
Best-Path Selection
     |
     v
RIB Installation
     |
  Not Installed
```

The route is accepted into the BGP table but is not installed in the IP routing table.

For example, an IBGP route with administrative distance 200 may lose RIB installation to an OSPF route with administrative distance 110 for the same prefix.

**A route can be accepted into the BGP table without being installed in the RIB.**

Additionally, a router can retain multiple BGP paths for the same prefix, even though only one is selected as the BGP best path.

See [BGP Path Selection and RIB Installation](../path-selection/best-path/rib-installation.md).

---

## Applying Changes to BGP Filters

Changes to inbound and outbound filters may require existing BGP routing information to be reprocessed.

Cisco IOS XE supports soft clearing to apply policies without tearing down the BGP session.

### Inbound Soft Clear

```text
R1# clear ip bgp 10.0.12.2 soft in
```

If the neighbor supports BGP route refresh, R1 can request updated routing information from the neighbor and reapply its inbound policies.

This allows previously rejected routes to be accepted if the policy now permits them.

### Outbound Soft Clear

```text
R1# clear ip bgp 10.0.12.2 soft out
```

This reapplies outbound routing policies and updates the advertisements sent to the neighbor.

### Soft Reconfiguration Inbound

For peers that do not support route refresh, Cisco IOS XE can retain received routing information locally using:

```text
R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 soft-reconfiguration inbound
```

This allows inbound policies to be reapplied using stored routes.

However, retaining unmodified received routes requires additional memory. Route refresh is generally preferable when supported.

See [BGP Route Refresh](../other-features/route-refresh.md) and [BGP Soft Reconfiguration](../other-features/soft-reconfiguration.md).

---

## Verification and Troubleshooting

### Display Accepted Routes

```text
R1# show bgp ipv4 unicast neighbors 10.0.12.2 routes
```

Displays routes accepted from the specified neighbor.

This is useful for verifying inbound route filtering.

### Display Received Routes Before Filtering

```text
R1# show bgp ipv4 unicast neighbors 10.0.12.2 received-routes
```

Displays routes received from the neighbor before inbound policy filtering, when the router retains the necessary unmodified received-route information.

On common IOS XE implementations, this requires `neighbor ... soft-reconfiguration inbound`.

**Route refresh alone does not necessarily make the pre-policy received routes available for this command.**

This distinction is particularly useful for troubleshooting:

- A route appears in `received-routes` but not in `routes`: an inbound policy or another BGP acceptance check may be rejecting it.
- A route appears in `routes` but is not selected as best: investigate best-path selection.
- A BGP best path is not installed in the IP routing table: investigate RIB installation.

### Display Advertised Routes

```text
R1# show bgp ipv4 unicast neighbors 10.0.12.2 advertised-routes
```

Displays routes advertised to the specified neighbor, allowing verification of outbound filtering.

### Display a Specific Prefix

```text
R1# show bgp ipv4 unicast 203.0.113.0/24
```

Displays available BGP paths and their attributes.

### Display Configured Filters

```text
R1# show ip prefix-list
R1# show ip as-path-access-list
R1# show route-map
R1# show running-config | section router bgp
```

These commands help identify the configured filtering rules and their application to BGP neighbors.

---

## Common Troubleshooting Scenarios

| Symptom | Possible Cause |
|---|---|
| Expected prefix is missing from the BGP table | Inbound filter denies it |
| Prefix is in the BGP table but not advertised | Outbound filter, advertisement eligibility, or BGP propagation rules |
| All routes are unexpectedly filtered | Implicit deny in a prefix list or route map |
| Route map denies the wrong routes | Confusion between prefix-list permit/deny and route-map permit/deny |
| Filter changes do not affect existing routes | Inbound or outbound policy has not been reapplied |
| Prefix is accepted but not installed in the RIB | RIB installation failure, competing route, or next-hop problem |
| Route matches a prefix list but is still rejected | Another filtering mechanism rejects it |

---

## Key Points

- **Inbound filtering** controls which received routes are accepted into the local BGP table.
- **Outbound filtering** controls which routes are advertised to a neighbor.
- Prefix lists filter routes based on network prefixes and prefix lengths.
- Distribute lists use ACLs to filter IPv4 BGP routes.
- Filter lists use AS-path access lists to filter based on AS_PATH.
- Route maps support filtering and attribute manipulation.
- Prefix lists and distribute lists are mutually exclusive for the same neighbor and direction.
- Multiple compatible filtering mechanisms can be applied to the same neighbor.
- Route maps and prefix lists have implicit deny behavior.
- Inbound filtering does not prevent the neighbor from sending unwanted routes.
- Soft clearing allows filtering policies to be reapplied without resetting the BGP session.

---

## References

- [Cisco: Connecting to a Service Provider Using External BGP](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-routing/b-ip-routing/m_irg-external-sp-0.html)
- [Cisco: BGP 4 Prefix Filter and Inbound Route Maps](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_bgp/configuration/xe-3se/3650/irg-xe-3se-3650-book/irg-prefix-filter.html)
- [Cisco: BGP Command Reference, BGP Commands M through N](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_bgp/command/irg-cr-book/bgp-m1.html)
