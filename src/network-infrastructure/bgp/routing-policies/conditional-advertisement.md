# 1.5.c (ii) Conditional Advertisement

**BGP conditional advertisement** allows a router to advertise selected routes to a BGP neighbor depending on whether other routes exist in its local BGP table.

Normally, BGP advertises eligible routes without considering the existence of unrelated prefixes.

Conditional advertisement adds a dependency between routes. For example, a router can advertise a backup prefix to an ISP only when a route learned from another ISP disappears.

Cisco IOS XE supports two conditional advertisement mechanisms:

- **`exist-map`:** Advertise selected routes when a specified condition is present.
- **`non-exist-map`:** Advertise selected routes when a specified condition is absent.

Both mechanisms use an **`advertise-map`** to identify the prefixes that will be conditionally advertised.

---

## Conditional Advertisement Operation

Conditional advertisement uses two route maps:

1. **Advertise map:** Identifies the routes that should be advertised.
2. **Condition map:** Identifies the routes whose existence or absence determines whether advertisement occurs.

The condition map can be either an `exist-map` or a `non-exist-map`.

| Component | Purpose |
|---|---|
| `advertise-map` | Identifies routes to advertise conditionally |
| `exist-map` | Requires matching routes to exist in the local BGP table |
| `non-exist-map` | Requires matching routes to be absent from the local BGP table |

### Decision Process

```text
           Local BGP Table
                  |
                  v
          +----------------+
          | Condition Map  |
          +----------------+
                  |
                  v
          Condition Met?
             /       \
           Yes        No
            |          |
            v          v
        Advertise    Withdraw /
        Selected     Do Not
        Routes       Advertise
```

The condition is evaluated against the **local BGP table**, not simply the global IP routing table.

The routes identified by the advertise map must also exist in the BGP table.

**Conditional advertisement does not create new BGP routes.** It controls the advertisement of existing BGP routes to a specified neighbor.

---

## Conditional Advertisement Syntax

Under the BGP address family:

```text
R1(config-router-af)# neighbor <neighbor> advertise-map <advertise-map> exist-map <exist-map>
```

Or:

```text
R1(config-router-af)# neighbor <neighbor> advertise-map <advertise-map> non-exist-map <non-exist-map>
```

The two options have opposite conditions.

| Condition Route Exists? | `exist-map` | `non-exist-map` |
|---|---|---|
| Yes | Advertise | Do not advertise |
| No | Do not advertise | Advertise |

In both cases, the route identified by the advertise map must be present and eligible for advertisement.

---

## Conditional Advertisement with `exist-map`

The **`exist-map`** option advertises selected routes when a specified condition is present in the local BGP table.

### Example

R1 has two BGP routes:

```text
192.0.2.0/24
203.0.113.0/24
```

R1 should advertise `192.0.2.0/24` to R2 **only when `203.0.113.0/24` exists** in its BGP table.

### Configuration

First, configure prefix lists:

```text
R1(config)# ip prefix-list ADVERTISE permit 192.0.2.0/24
R1(config)# ip prefix-list CONDITION permit 203.0.113.0/24
```

Create the route maps:

```text
R1(config)# route-map ADVERTISE-MAP permit 10
R1(config-route-map)# match ip address prefix-list ADVERTISE

R1(config)# route-map EXIST-MAP permit 10
R1(config-route-map)# match ip address prefix-list CONDITION
```

Apply conditional advertisement:

```text
R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 advertise-map ADVERTISE-MAP exist-map EXIST-MAP
```

### Result

| `203.0.113.0/24` in BGP Table? | Advertise `192.0.2.0/24`? |
|---|---|
| Yes | Yes |
| No | No |

When the condition disappears, R1 withdraws the conditional advertisement from R2.

When the condition becomes true again, R1 can readvertise the route.

The route `192.0.2.0/24` itself does not need to disappear from R1's BGP table when the condition is false.

---

## Conditional Advertisement with `non-exist-map`

The **`non-exist-map`** option performs the opposite function.

It advertises selected routes when a specified condition is **absent** from the local BGP table.

This is particularly useful for backup routes and multihomed networks.

### Example

R1 should advertise `192.0.2.0/24` to R2 only when `203.0.113.0/24` is missing from its BGP table.

The same prefix lists and route maps can be used.

### Configuration

```text
R1(config)# ip prefix-list ADVERTISE permit 192.0.2.0/24
R1(config)# ip prefix-list CONDITION permit 203.0.113.0/24

R1(config)# route-map ADVERTISE-MAP permit 10
R1(config-route-map)# match ip address prefix-list ADVERTISE

R1(config)# route-map NON-EXIST-MAP permit 10
R1(config-route-map)# match ip address prefix-list CONDITION

R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 advertise-map ADVERTISE-MAP non-exist-map NON-EXIST-MAP
```

### Result

| `203.0.113.0/24` in BGP Table? | Advertise `192.0.2.0/24`? |
|---|---|
| Yes | No |
| No | Yes |

**Important:** `non-exist-map` does not mean that the advertise-map route must be absent.

The route identified by the advertise map must still exist in the local BGP table. The non-exist-map condition applies to a separate set of routes.

---

## Complete Conditional Advertisement Lab

The following example demonstrates how conditional advertisement can be used for backup routing.

### Topology

```text
                  R1
                AS 65001
             Lo0: 1.1.1.1
                /     \
               /       \
          E0/0           E0/1
           /               \
          /                 \
         R2                  R3
      AS 65002            AS 65003
   Lo0: 2.2.2.2        Lo0: 3.3.3.3
```

Connections:

| Link | R1 | Neighbor |
|---|---|---|
| R1-R2 | E0/0: 10.0.12.1/30 | R2 E0/0: 10.0.12.2/30 |
| R1-R3 | E0/1: 10.0.13.1/30 | R3 E0/0: 10.0.13.2/30 |

All three routers establish EBGP sessions.

### Objective

R1 should advertise `1.1.1.1/32` to R2 only when the route `3.3.3.3/32`, learned from R3, is absent from R1's BGP table.

This simulates advertising a backup route when the primary path becomes unavailable.

### Step 1: Configure R1

```text
R1(config)# interface Ethernet0/0
R1(config-if)# ip address 10.0.12.1 255.255.255.252
R1(config-if)# no shutdown

R1(config)# interface Ethernet0/1
R1(config-if)# ip address 10.0.13.1 255.255.255.252
R1(config-if)# no shutdown

R1(config)# interface Loopback0
R1(config-if)# ip address 1.1.1.1 255.255.255.255

R1(config)# router bgp 65001
R1(config-router)# neighbor 10.0.12.2 remote-as 65002
R1(config-router)# neighbor 10.0.13.2 remote-as 65003
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 activate
R1(config-router-af)# neighbor 10.0.13.2 activate
R1(config-router-af)# network 1.1.1.1 mask 255.255.255.255
```

### Step 2: Configure R2

```text
R2(config)# interface Ethernet0/0
R2(config-if)# ip address 10.0.12.2 255.255.255.252
R2(config-if)# no shutdown

R2(config)# interface Loopback0
R2(config-if)# ip address 2.2.2.2 255.255.255.255

R2(config)# router bgp 65002
R2(config-router)# neighbor 10.0.12.1 remote-as 65001
R2(config-router)# address-family ipv4 unicast
R2(config-router-af)# neighbor 10.0.12.1 activate
R2(config-router-af)# network 2.2.2.2 mask 255.255.255.255
```

### Step 3: Configure R3

```text
R3(config)# interface Ethernet0/0
R3(config-if)# ip address 10.0.13.2 255.255.255.252
R3(config-if)# no shutdown

R3(config)# interface Loopback0
R3(config-if)# ip address 3.3.3.3 255.255.255.255

R3(config)# router bgp 65003
R3(config-router)# neighbor 10.0.13.1 remote-as 65001
R3(config-router)# address-family ipv4 unicast
R3(config-router-af)# neighbor 10.0.13.1 activate
R3(config-router-af)# network 3.3.3.3 mask 255.255.255.255
```

### Step 4: Configure Conditional Advertisement on R1

Create the prefix lists:

```text
R1(config)# ip prefix-list ADVERTISE permit 1.1.1.1/32
R1(config)# ip prefix-list TRACK-R3 permit 3.3.3.3/32
```

Create the route maps:

```text
R1(config)# route-map ADVERTISE-MAP permit 10
R1(config-route-map)# match ip address prefix-list ADVERTISE

R1(config)# route-map TRACK-R3-MAP permit 10
R1(config-route-map)# match ip address prefix-list TRACK-R3
```

Apply the conditional advertisement to R2:

```text
R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 advertise-map ADVERTISE-MAP non-exist-map TRACK-R3-MAP
```

### Step 5: Verify Normal Operation

With R3 operational, R1 receives `3.3.3.3/32` through EBGP.

Verify:

```text
R1# show bgp ipv4 unicast 3.3.3.3/32
```

The route should be present in R1's BGP table.

Check advertisements toward R2:

```text
R1# show bgp ipv4 unicast neighbors 10.0.12.2 advertised-routes
```

Because `3.3.3.3/32` exists in R1's BGP table, the non-exist-map condition is not satisfied.

Therefore, R1 should **not advertise `1.1.1.1/32` to R2**.

Verify on R2:

```text
R2# show bgp ipv4 unicast 1.1.1.1/32
```

The prefix should be absent.

### Step 6: Simulate Primary Route Failure

On R3, shut down Loopback0:

```text
R3(config)# interface Loopback0
R3(config-if)# shutdown
```

This removes the connected route required by R3's BGP `network` statement.

R3 withdraws `3.3.3.3/32` from R1.

Verify:

```text
R1# show bgp ipv4 unicast 3.3.3.3/32
```

Once R1 no longer has the route in its BGP table, the `non-exist-map` condition is satisfied.

R1 can now advertise `1.1.1.1/32` to R2.

Verify:

```text
R1# show bgp ipv4 unicast neighbors 10.0.12.2 advertised-routes
R2# show bgp ipv4 unicast 1.1.1.1/32
```

On R2, the prefix should appear with:

```text
Network: 1.1.1.1/32
AS_PATH: 65001
```

**Note:** Conditional advertisement is not necessarily immediate. BGP periodically reevaluates the conditions, so a delay is expected before the advertisement changes.

### Step 7: Restore the Primary Route

On R3:

```text
R3(config)# interface Loopback0
R3(config-if)# no shutdown
```

R3 reoriginates `3.3.3.3/32`.

R1 receives the route again, causing the non-exist-map condition to become false.

After reevaluation, R1 withdraws the conditional advertisement of `1.1.1.1/32` from R2.

Verify that the route is no longer present in R2's BGP table.

---

## Conditional Advertisement and BGP Route Attributes

Conditional advertisement can also use route maps that modify BGP attributes.

For example:

```text
R1(config)# route-map ADVERTISE-MAP permit 10
R1(config-route-map)# match ip address prefix-list ADVERTISE
R1(config-route-map)# set metric 200
```

When the conditional advertisement is activated, the matching route is advertised with MED 200.

Cisco recommends configuring the intended BGP attribute modifications within the **advertise map** rather than relying on a separate route map for those conditional advertisements.

This allows conditional advertisement and attribute manipulation to be implemented together.

---

## Matching Additional Attributes in Condition Maps

Condition maps can match more than just destination prefixes.

For example, a condition map can match both:

- A destination prefix.
- Its AS_PATH.

This allows conditional advertisement to depend on learning a particular prefix through a particular autonomous system.

### Example

R1 should advertise `192.0.2.0/24` to a neighbor only when `203.0.113.0/24` is learned with AS_PATH beginning with AS 65002.

Configure the prefix list:

```text
R1(config)# ip prefix-list CONDITION permit 203.0.113.0/24
```

Configure an AS-path access list:

```text
R1(config)# ip as-path access-list 10 permit ^65002_
```

Create the condition map:

```text
R1(config)# route-map EXIST-MAP permit 10
R1(config-route-map)# match ip address prefix-list CONDITION
R1(config-route-map)# match as-path 10
```

Both conditions must match:

1. The route's prefix is `203.0.113.0/24`.
2. The AS_PATH begins with AS 65002.

The advertise map can then be applied using:

```text
R1(config-router-af)# neighbor 10.0.12.2 advertise-map ADVERTISE-MAP exist-map EXIST-MAP
```

**Important:** For conditional advertisement, an AS_PATH match supplements the prefix condition. Do not rely on an AS-path-only condition map to test for the existence of arbitrary routes from an AS.

---

## The `check-all-paths` Option

By default, conditional advertisement generally evaluates the selected BGP path when checking route attributes against a condition map.

However, a prefix can have multiple BGP paths.

Consider:

| Prefix | AS_PATH | Best Path? |
|---|---|---|
| `203.0.113.0/24` | `65002` | No |
| `203.0.113.0/24` | `65003` | Yes |

Suppose the condition map matches:

```text
R1(config)# ip as-path access-list 10 permit ^65002$
```

Without additional configuration, the condition may not match because the selected best path has AS_PATH `65003`.

Cisco IOS XE supports the optional **`check-all-paths`** keyword:

```text
R1(config-router-af)# neighbor 10.0.12.2 advertise-map ADVERTISE-MAP exist-map EXIST-MAP check-all-paths
```

This enables evaluation of all available BGP paths for the relevant prefix, rather than relying only on the selected path.

This is useful when the condition map matches path-specific information such as AS_PATH.

The option is particularly important when the condition should be satisfied by a non-best BGP path.

---

## Conditional Advertisement Timing

BGP conditional advertisement does not necessarily react immediately when the condition changes.

On traditional Cisco IOS/IOS XE implementations, the **BGP scanner** periodically evaluates conditional advertisement policies.

The default scanner interval is generally **60 seconds**.

For example:

1. R1 learns that a tracked BGP prefix has disappeared.
2. The BGP table is updated.
3. The conditional advertisement policy is reevaluated.
4. R1 sends the appropriate UPDATE or withdrawal to the affected neighbor.

As a result, conditional advertisement may take approximately one scanner interval to react, in addition to any BGP convergence and update-processing delays.

**Conditional advertisement is not an instantaneous failover mechanism.**

Even when BFD detects a failed BGP neighbor quickly, conditional advertisements can still be delayed by the reevaluation process.

---

## Conditional Advertisement vs. Conditional Route Injection

**Conditional advertisement** and **conditional route injection** are separate BGP features.

| Feature | Conditional Advertisement | Conditional Route Injection |
|---|---|---|
| Primary command | `neighbor ... advertise-map` | `bgp inject-map` |
| Purpose | Control advertisements to a neighbor | Create more-specific BGP routes conditionally |
| Creates new BGP prefixes? | No | Yes |
| Requires advertised route in BGP table? | Yes | Injection generates a new route when conditions are met |
| Typical use | Backup advertisements and multihoming | More-specific route injection and traffic engineering |

Conditional route injection uses commands such as:

```text
R1(config-router)# bgp inject-map INJECT exist-map EXIST
```

This can create more-specific BGP routes based on existing routing information.

It should not be confused with `neighbor advertise-map`, which controls advertisement of routes already present in the BGP table.

---

## VRF-Aware Conditional Advertisement

Cisco IOS XE also supports conditional advertisement within VRF address families on supported platforms.

For example:

```text
R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 vrf CUSTOMER
R1(config-router-af)# neighbor 10.0.12.2 remote-as 65002
R1(config-router-af)# neighbor 10.0.12.2 activate
R1(config-router-af)# neighbor 10.0.12.2 advertise-map ADVERTISE-MAP non-exist-map NON-EXIST-MAP
```

In this case:

- The condition is evaluated using the **CUSTOMER VRF BGP table**.
- The advertised prefixes must exist in the same relevant BGP address family.
- The conditional advertisement is applied to the specified VRF neighbor.

A route existing only in the global BGP table does not automatically satisfy a condition evaluated within a VRF.

VRF-aware conditional advertisement is also supported for IPv6 unicast address families on applicable IOS XE platforms.

---

## Verification and Troubleshooting

### Verify the Tracked Route

```text
R1# show bgp ipv4 unicast 203.0.113.0/24
```

This confirms whether the tracked prefix exists in the BGP table.

### Verify the Advertised Route

```text
R1# show bgp ipv4 unicast 192.0.2.0/24
```

The route identified by the advertise map must be present in the BGP table.

### Verify Neighbor Advertisements

```text
R1# show bgp ipv4 unicast neighbors 10.0.12.2 advertised-routes
```

Check whether the conditionally controlled prefix is currently advertised.

### Verify Conditional Advertisement Status

```text
R1# show bgp ipv4 unicast neighbors 10.0.12.2
```

On supported IOS XE versions, the detailed neighbor output includes conditional advertisement information.

For example:

```text
Condition-map NON-EXIST-MAP, Advertise-map ADVERTISE-MAP, status: Advertise
```

Or:

```text
Condition-map NON-EXIST-MAP, Advertise-map ADVERTISE-MAP, status: Withdraw
```

| Status | Meaning |
|---|---|
| `Advertise` | Condition is satisfied; conditional advertisement is active |
| `Withdraw` | Condition is not satisfied; conditional advertisement is inactive |

These statuses describe the conditional advertisement state. Normal BGP advertisement requirements still apply.

### Verify Route Maps

```text
R1# show route-map
R1# show route-map ADVERTISE-MAP
R1# show route-map EXIST-MAP
```

### Verify Prefix Lists

```text
R1# show ip prefix-list
```

### Verify the BGP Configuration

```text
R1# show running-config | section router bgp
```

### Debug Conditional Advertisements

On supported IOS XE versions:

```text
R1# debug ip bgp updates
```

This can reveal BGP UPDATE processing and conditional route advertisement decisions.

Use BGP debugging cautiously in production networks, particularly when large numbers of prefixes are being processed.

---

## Common Troubleshooting Scenarios

| Symptom | Possible Cause |
|---|---|
| Advertise-map route is never advertised | Route is missing from the local BGP table |
| `exist-map` never activates | Tracked route is absent or does not match the condition map |
| `non-exist-map` never activates | A matching tracked route still exists in the BGP table |
| Condition seems correct but advertisement is delayed | BGP scanner has not reevaluated the condition |
| AS_PATH-based condition fails unexpectedly | The selected path does not match; `check-all-paths` may be required |
| Condition works in global BGP but not a VRF | The tracked prefix is absent from the VRF BGP table |
| Route is not received despite `Advertise` status | Other BGP advertisement rules or outbound filtering may prevent it |
| Route is advertised to the wrong peers | Conditional advertisement was configured on the wrong neighbor |

---

## Key Points

- **Conditional advertisement** controls which BGP routes are advertised based on the existence of other BGP routes.
- **`advertise-map`** identifies the routes to advertise.
- **`exist-map`** activates advertisement when matching routes exist in the BGP table.
- **`non-exist-map`** activates advertisement when matching routes are absent from the BGP table.
- The advertised routes must already exist in the BGP table.
- Conditional advertisement does not create new BGP prefixes.
- Conditions are evaluated using the relevant BGP table, not simply the global RIB.
- Condition maps can match both prefixes and BGP attributes.
- **`check-all-paths`** allows the condition to consider non-best paths.
- Conditional advertisements can be delayed by periodic BGP scanner processing.
- Conditional advertisement is particularly useful in multihomed networks and backup routing designs.
- **Conditional route injection** is a separate feature that creates new BGP routes.

---

## References

- [Cisco: Configure and Verify the BGP Conditional Advertisement Feature](https://www.cisco.com/c/en/us/support/docs/ip/border-gateway-protocol-bgp/16137-cond-adv.html)
- [Cisco: BGP Conditional Advertisement Feature](https://www.cisco.com/c/en/us/support/docs/ip/border-gateway-protocol-bgp/215634-bgp-conditional-advertisement-feature.html)
- [Cisco: IOS XE 17.x BGP 4 Configuration](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-routing/b-ip-routing/m_irg-bgp4.html)
- [Cisco: BGP-VRF-Aware Conditional Advertisement](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-routing/b-ip-routing/m_irg-vrf-conditional-adv.html)
- [Cisco: BGP Command Reference, M through N](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_bgp/command/irg-cr-book/bgp-m1.html)
