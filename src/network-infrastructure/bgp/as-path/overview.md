# 1.5.d AS Path Manipulations

**BGP AS_PATH manipulation** allows administrators to modify, filter, or control how autonomous system numbers (ASNs) are handled in BGP routing updates.

The AS_PATH attribute is a **well-known mandatory, transitive** BGP path attribute that records the autonomous systems a route has traversed.

AS_PATH manipulation is commonly used for:

- **Inbound traffic engineering:** Influencing which paths external autonomous systems prefer when sending traffic toward the local AS.
- **AS migration:** Allowing BGP neighbors to continue using an old ASN while transitioning to a new ASN.
- **Loop prevention exceptions:** Allowing routes containing the local ASN to be accepted in specific network designs.
- **Private ASN removal:** Preventing private ASNs from being advertised to the global Internet.
- **Route filtering:** Accepting or rejecting BGP routes based on the ASNs in their AS_PATH.

Cisco IOS XE supports these capabilities through neighbor configuration commands, route maps, and AS-path access lists.

---

## AS_PATH Fundamentals

By default, a BGP router **prepends its local ASN** to the AS_PATH when advertising a route to an EBGP neighbor.

The AS_PATH is normally preserved when advertising routes to IBGP neighbors.

Consider the following topology:

```text
        AS 65001            AS 65002            AS 65003

           R1 ---------------- R2 ---------------- R3
                  EBGP                EBGP

     192.0.2.0/24
```

R1 originates `192.0.2.0/24`.

As the route travels between autonomous systems, its AS_PATH changes:

| Router | AS_PATH |
|---|---|
| R1 | Empty |
| R2 | `65001` |
| R3 | `65002 65001` |

The most recently added ASN appears on the **left**.

For example:

```text
65003 65002 65001
```

- `65003` is the most recently traversed AS.
- `65001` is the originating AS.
- The AS_PATH contains three ASN occurrences.

### AS_PATH and Best-Path Selection

By default, BGP prefers the **shortest AS_PATH**, provided earlier best-path selection criteria are equal.

For example:

```text
Path A: 65002 65001
Path B: 65003 65004 65001
```

Path A has an AS_PATH length of 2.

Path B has an AS_PATH length of 3.

If the best-path selection process reaches AS_PATH length, **Path A is preferred**.

However, attributes such as Weight and LOCAL_PREF are normally evaluated before AS_PATH length.

Therefore, manipulating AS_PATH does not guarantee that another BGP router will select the desired path.

For a detailed explanation, see [AS_PATH](../path-selection/attributes/as-path.md) and [BGP Best Path Selection Steps](../path-selection/best-path/selection-steps.md).

---

## AS_PATH Manipulation Mechanisms

The CCIE Enterprise Infrastructure blueprint identifies three major areas of AS_PATH manipulation.

| Topic | Mechanism | Purpose |
|---|---|---|
| 1.5.d (i) | `neighbor local-as` | Present an alternative ASN to a BGP neighbor |
| 1.5.d (i) | `neighbor allowas-in` | Accept routes containing the local ASN |
| 1.5.d (i) | `neighbor remove-private-as` | Remove or replace private ASNs in outbound advertisements |
| 1.5.d (ii) | `set as-path prepend` | Artificially increase AS_PATH length |
| 1.5.d (iii) | AS-path regular expressions | Match routes according to AS_PATH contents |

These mechanisms have different effects on BGP operation.

Some modify AS_PATH attributes in outgoing advertisements, while others change how incoming routes are accepted or filtered.

---

## 1.5.d (i) Local-AS, Allowas-in, and Remove-Private-AS

These features modify how BGP handles autonomous system numbers during peering, route reception, and route advertisement.

### Local-AS

The `neighbor local-as` command allows a BGP router to use an **alternative ASN for a specific EBGP neighbor**.

This is especially useful during autonomous system migrations.

For example, suppose R1 belongs to AS 65001 but must continue peering with R2, which expects R1 to belong to AS 65010.

```text
        Actual AS: 65001                   AS 65002
        Local-AS:  65010

              R1 ----------------------------- R2
          10.0.12.1                        10.0.12.2
```

R1 can configure:

```text
R1(config)# router bgp 65001
R1(config-router)# neighbor 10.0.12.2 remote-as 65002
R1(config-router)# neighbor 10.0.12.2 local-as 65010
```

R1 uses AS 65010 as its local ASN for the EBGP session with R2, rather than its globally configured BGP ASN of 65001.

By default, this feature can also affect the AS_PATH of exchanged routes.

Several options control its behavior.

| Option | Purpose |
|---|---|
| `local-as <ASN>` | Configure an alternative ASN for the neighbor |
| `no-prepend` | Prevent the alternative local ASN from being added to AS_PATHs received from the neighbor |
| `replace-as` | Use the alternative local ASN instead of the global BGP ASN in outgoing AS_PATHs toward the neighbor |
| `dual-as` | Allow the neighbor to establish the EBGP session using either the alternative or global ASN |

For example:

```text
R1(config-router)# neighbor 10.0.12.2 local-as 65010 no-prepend replace-as dual-as
```

The options are particularly useful when migrating between ASNs without requiring every neighboring router to be reconfigured simultaneously.

**Important:** `local-as` modifies the ASN used for a particular peering relationship. It does not change the ASN configured under `router bgp`.

See [Local-AS](as-number-manipulation/local-as.md) for detailed behavior, AS_PATH examples, and migration configurations.

### Allowas-in

BGP uses the AS_PATH attribute for **inter-AS loop prevention**.

By default, a BGP router rejects a received route if its own ASN appears in the AS_PATH.

For example, R1 belongs to AS 65001.

It receives:

```text
AS_PATH: 65002 65003 65001
```

R1 normally rejects the route because its own ASN, `65001`, appears in the path.

However, some network designs intentionally reuse an ASN at multiple locations.

Cisco IOS XE supports `neighbor allowas-in` to permit these routes.

```text
R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 allowas-in 2
```

This permits R1 to accept routes from the specified neighbor containing its own ASN **up to two times**.

For example:

| Received AS_PATH | Result |
|---|---|
| `65002 65003` | Accepted, assuming other checks pass |
| `65002 65001` | Accepted |
| `65002 65001 65003 65001` | Accepted |
| `65002 65001 65003 65001 65001` | Rejected by the occurrence limit |

**Important:** `allowas-in` weakens normal AS_PATH loop prevention. It should only be used when the network design requires it.

It does not remove the local ASN from the received AS_PATH.

See [Allowas-in](as-number-manipulation/allowas-in.md).

### Remove-Private-AS

Private ASNs are commonly used within enterprise networks and for customer-to-provider EBGP peering.

The private ASN ranges are:

| Type | Range |
|---|---|
| 2-byte private ASNs | 64512-65534 |
| 4-byte private ASNs | 4200000000-4294967294 |

Private ASNs are not globally unique and should not be advertised in the global Internet BGP routing table.

Cisco IOS XE supports removing private ASNs from outgoing EBGP advertisements.

For example:

```text
             Customer              ISP             Internet
              AS 65001            AS 100            AS 200

                 R1 --------------- R2 --------------- R3
```

R2 learns a route from R1 with:

```text
AS_PATH: 65001
```

Normally, R2 advertises the route to R3 with:

```text
AS_PATH: 100 65001
```

R2 can remove the private ASN:

```text
R2(config)# router bgp 100
R2(config-router)# address-family ipv4 unicast
R2(config-router-af)# neighbor 10.0.23.3 remove-private-as
```

R3 now receives:

```text
AS_PATH: 100
```

Cisco IOS XE also supports:

```text
R2(config-router-af)# neighbor 10.0.23.3 remove-private-as all
R2(config-router-af)# neighbor 10.0.23.3 remove-private-as all replace-as
```

The `all` option supports removing private ASNs from paths containing both public and private ASNs.

The `replace-as` option replaces private ASNs with the advertising router's ASN rather than simply deleting them.

**Important distinctions:**

- Private ASN removal affects outbound EBGP advertisements.
- It does not remove private ASNs from the local BGP table.
- It does not change the BGP ASN used to establish the neighbor session.
- It is different from the `replace-as` option of `neighbor local-as`.

See [Remove-Private-AS](as-number-manipulation/remove-private-as.md).

---

## 1.5.d (ii) AS Path Prepending

**AS_PATH prepending** intentionally adds extra ASN occurrences to the AS_PATH attribute.

Its primary purpose is **inbound traffic engineering**.

Because BGP normally prefers shorter AS_PATHs, prepending can make one route less attractive than another.

### Example

Consider an enterprise connected to two ISPs:

```text
                       Internet
                          |
                    +-----+-----+
                    |           |
                   ISP1        ISP2
                 AS 65100    AS 65200
                    |           |
                    |           |
                    R1----------R2
                       AS 65000
                          |
                   203.0.113.0/24
```

The enterprise advertises `203.0.113.0/24` through both ISPs.

By default, each ISP receives an AS_PATH containing:

```text
65000
```

The enterprise wants to make ISP2 less attractive for inbound traffic.

It can prepend AS 65000 twice when advertising the prefix toward ISP2.

### Configuration

On R2, assuming ISP2 is the EBGP neighbor at `10.0.23.3`:

```text
R2(config)# ip prefix-list ENTERPRISE permit 203.0.113.0/24

R2(config)# route-map PREPEND-ISP2 permit 10
R2(config-route-map)# match ip address prefix-list ENTERPRISE
R2(config-route-map)# set as-path prepend 65000 65000

R2(config)# route-map PREPEND-ISP2 permit 20

R2(config)# router bgp 65000
R2(config-router)# address-family ipv4 unicast
R2(config-router-af)# neighbor 10.0.23.3 route-map PREPEND-ISP2 out
```

The additional permit sequence allows other eligible routes to be advertised without the extra prepending.

After the policy is applied, ISP2 receives:

```text
65000 65000 65000
```

This contains:

- One normal EBGP prepend of AS 65000.
- Two additional occurrences configured by the route map.

ISP1 continues to receive:

```text
65000
```

External networks may therefore prefer the path through ISP1.

However, **AS_PATH prepending does not guarantee inbound path selection**.

Other autonomous systems can use LOCAL_PREF, routing policies, or other attributes to prefer a longer AS_PATH.

Also, prepending changes the **AS_PATH length**, not the actual number of router hops or the underlying link cost.

See [AS Path Prepending](prepending.md) and [Inbound Traffic Engineering](../routing-policies/multihoming/inbound.md).

---

## 1.5.d (iii) Regular Expressions

BGP supports **AS-path regular expressions** to match routes based on the ASN sequences contained in their AS_PATH attributes.

Unlike AS_PATH prepending, regular expressions do not directly modify the AS_PATH.

Instead, they are used to identify routes for filtering or routing-policy actions.

Cisco IOS XE supports regular expressions through:

- `ip as-path access-list`
- `neighbor filter-list`
- `match as-path` in route maps
- BGP verification commands

### Common AS_PATH Regular Expressions

| Expression | Meaning |
|---|---|
| `^$` | Empty AS_PATH |
| `^65001$` | AS_PATH containing only AS 65001 |
| `^65001_` | AS_PATH beginning with AS 65001 |
| `_65001$` | AS_PATH ending with AS 65001 |
| `_65001_` | AS 65001 appears anywhere in the AS_PATH |
| `.*` | Match any AS_PATH |

Several special characters are especially important:

| Character | Meaning |
|---|---|
| `^` | Beginning of the AS_PATH string |
| `$` | End of the AS_PATH string |
| `_` | Cisco AS_PATH delimiter matching |
| `.` | Match any single character |
| `*` | Match zero or more occurrences of the preceding expression |
| `+` | Match one or more occurrences of the preceding expression |
| `[]` | Match a character from a specified set |
| `()` | Group expressions |

The underscore (`_`) is particularly useful.

It matches AS_PATH boundaries, including separators between ASNs and the beginning or end of the AS_PATH.

For example:

```text
_65001_
```

Matches:

```text
65001
65002 65001
65001 65002
65002 65001 65003
```

It does not match ASN 650010 simply because that ASN contains the digits `65001`.

### Filtering with AS-Path Access Lists

Suppose R1 must reject all BGP routes whose AS_PATH contains AS 65050.

Configure:

```text
R1(config)# ip as-path access-list 10 deny _65050_
R1(config)# ip as-path access-list 10 permit .*
```

Apply the access list to incoming updates from R2:

```text
R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 filter-list 10 in
```

R1 rejects routes containing AS 65050 in their AS_PATH.

All other routes are permitted by the second access-list entry.

The following configuration demonstrates using the same AS-path access list in a route map:

```text
R1(config)# route-map FILTER-AS permit 10
R1(config-route-map)# match as-path 10
```

**Important:** AS-path access lists have an implicit deny. When used in a route-map `match as-path` statement, the ACL's permit entries identify matches; the route-map sequence determines what happens to those matching routes.

See [Regular Expressions](regular-expressions.md) for detailed syntax, pattern matching, filtering, and verification.

---

## AS_PATH Manipulation and Loop Prevention

AS_PATH manipulation can affect BGP's normal inter-AS loop-prevention mechanisms.

Two important mechanisms are:

1. **Local-AS detection:** Rejecting routes containing the local ASN.
2. **First-AS enforcement:** Verifying that the leftmost ASN in a received EBGP AS_PATH matches the neighbor's ASN.

Cisco IOS XE normally enables first-AS enforcement by default.

```text
R1(config)# router bgp 65001
R1(config-router)# bgp enforce-first-as
```

It can be disabled using:

```text
R1(config-router)# no bgp enforce-first-as
```

These mechanisms are independent.

| Feature | Effect |
|---|---|
| `neighbor allowas-in` | Permits received routes containing the local ASN |
| `no bgp enforce-first-as` | Disables the check that the first ASN matches the EBGP neighbor's ASN |
| `neighbor local-as` | Allows an alternative ASN to be used for a specific peering relationship |
| `neighbor remove-private-as` | Removes or replaces private ASNs in outbound updates |

**Disabling first-AS enforcement does not disable normal AS_PATH loop detection.**

Likewise, `allowas-in` does not automatically disable first-AS enforcement.

These distinctions are important when troubleshooting unusual EBGP designs.

---

## Verification and Troubleshooting

Several commands are useful when verifying AS_PATH manipulation.

### Display BGP Routes and AS_PATHs

```text
R1# show ip bgp
R1# show ip bgp 203.0.113.0
```

These commands display the AS_PATH information associated with BGP routes.

### Inspect Neighbor Advertisements

```text
R1# show ip bgp neighbors 10.0.12.2 advertised-routes
```

This command helps identify routes advertised toward a neighbor.

However, the displayed output may not show every final outbound AS_PATH transformation on all IOS XE implementations.

**The most reliable way to verify the final AS_PATH is to inspect the route on the receiving BGP neighbor.**

### Display AS-Path Access Lists

```text
R1# show ip as-path-access-list
```

### Display Routes Matching a Regular Expression

```text
R1# show ip bgp regexp _65001_
```

This displays routes with AS_PATHs matching the specified regular expression.

### Display Route Maps

```text
R1# show route-map
```

### Display Detailed Neighbor Information

```text
R1# show ip bgp neighbors 10.0.12.2
```

### Reapply Routing Policies

After modifying an inbound BGP routing policy, perform an inbound soft clear if necessary:

```text
R1# clear ip bgp 10.0.12.2 soft in
```

For outbound policy changes:

```text
R1# clear ip bgp 10.0.12.2 soft out
```

These operations normally allow policies to be reprocessed without resetting the established TCP session.

---

## Key Points

- AS_PATH is a **well-known mandatory, transitive** BGP attribute.
- EBGP normally prepends the advertising router's ASN to the AS_PATH.
- IBGP normally preserves the AS_PATH.
- The most recently traversed ASN appears on the **left**.
- BGP normally prefers the shortest AS_PATH when earlier best-path criteria are equal.
- `neighbor local-as` allows an alternative ASN to be used for an EBGP neighbor.
- `neighbor allowas-in` allows a router to accept routes containing its own ASN.
- `neighbor remove-private-as` removes private ASNs from outbound EBGP advertisements.
- `set as-path prepend` adds ASN occurrences to influence inbound traffic engineering.
- AS-path regular expressions allow routes to be matched according to their ASN sequences.
- `neighbor filter-list` applies AS-path access lists directly to BGP neighbors.
- `match as-path` allows AS-path access lists to be used within route maps.
- AS_PATH manipulation can affect routing policy, path selection, and loop prevention.

---

## Topics Covered

This section includes:

1. **[Local-AS, Allowas-in, and Remove-Private-AS](as-number-manipulation/overview.md)**
   - [Local-AS](as-number-manipulation/local-as.md): Alternative ASNs, AS migrations, and `no-prepend`, `replace-as`, and `dual-as`.
   - [Allowas-in](as-number-manipulation/allowas-in.md): Accepting routes containing the local ASN and controlling occurrence limits.
   - [Remove-Private-AS](as-number-manipulation/remove-private-as.md): Removing and replacing private ASNs in outbound EBGP advertisements.
2. **[AS Path Prepending](prepending.md):** Manipulating AS_PATH length through route maps to influence inbound traffic.
3. **[Regular Expressions](regular-expressions.md):** AS_PATH pattern matching, filtering, and regular-expression syntax.

---

## References

- [Cisco: Connecting to a Service Provider Using External BGP](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-routing/b-ip-routing/m_irg-external-sp-0.html)
- [Cisco: BGP Support for Dual AS Configuration for Network AS Migrations](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/ip-routing/b-ip-routing/m_irg-dual-as.html)
- [Cisco: Removing Private AS Numbers from the AS Path in BGP](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/ip-routing/b-ip-routing/m_irg-remove-as-0.html)
- [Cisco: BGP Command Reference](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_bgp/command/irg-cr-book/bgp-m1.html)
- [Cisco: Examine Border Gateway Protocol Case Studies](https://www.cisco.com/c/en/us/support/docs/ip/border-gateway-protocol-bgp/26634-bgp-toc.html)
