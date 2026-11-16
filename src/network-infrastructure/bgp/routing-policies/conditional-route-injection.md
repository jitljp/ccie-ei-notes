# BGP Conditional Route Injection

**BGP conditional route injection** allows a router to generate new BGP prefixes based on the existence of another BGP route.

It is configured using:

```text
R1(config-router-af)# bgp inject-map <inject-map> exist-map <exist-map> [copy-attributes]
```

Unlike conditional advertisement, which controls the advertisement of existing routes, **conditional route injection creates new routes in the local BGP table**.

The feature is primarily used to generate more-specific prefixes for traffic engineering.

---

## Conditional Route Injection Operation

Conditional route injection uses two route maps:

| Route Map | Purpose |
|---|---|
| `exist-map` | Identifies the existing parent route and its source |
| `inject-map` | Identifies the prefixes to inject into BGP |

Consider a router that learns the following BGP route:

```text
10.10.0.0/16
```

Conditional route injection can generate:

```text
10.10.1.0/24
10.10.2.0/24
```

These more-specific routes do not need to exist independently in the IP routing table or BGP table before injection.

However, a valid parent route must exist in the BGP table.

### Requirements

- The parent route must exist in the local BGP table.
- The `exist-map` must identify the parent prefix and its route source.
- Injected prefixes must be contained within the parent prefix.
- The injected routes are added to the local BGP table when the conditions are satisfied.

**Important:** Creating a more-specific BGP route does not automatically create actual network connectivity to that destination.

---

## Configuration Example

### Topology

```text
      AS 65001                       AS 65002

         R1 ---------------------------- R2
       E0/0 .1                       E0/0 .2
                10.0.12.0/30

      Advertises:                  Receives:
      10.10.0.0/16                 10.10.0.0/16
```

R1 advertises `10.10.0.0/16` to R2.

R2 should inject the following more-specific routes whenever it learns the parent route from R1:

```text
10.10.1.0/24
10.10.2.0/24
```

### Step 1: Configure R1

Assume R1 has IP connectivity to R2.

```text
R1(config)# ip route 10.10.0.0 255.255.0.0 Null0

R1(config)# router bgp 65001
R1(config-router)# neighbor 10.0.12.2 remote-as 65002
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# network 10.10.0.0 mask 255.255.0.0
```

The static Null0 route provides the exact RIB entry required by the BGP `network` statement.

This is a control-plane demonstration. Actual traffic to the Null0 route would be discarded unless a more-specific forwarding route exists.

### Step 2: Configure R2's `exist-map`

First, identify the parent prefix:

```text
R2(config)# ip prefix-list PARENT permit 10.10.0.0/16
```

Identify the BGP route source:

```text
R2(config)# ip prefix-list SOURCE permit 10.0.12.1/32
```

Create the existence route map:

```text
R2(config)# route-map EXIST permit 10
R2(config-route-map)# match ip address prefix-list PARENT
R2(config-route-map)# match ip route-source prefix-list SOURCE
```

The condition requires:

1. The BGP route `10.10.0.0/16` to exist.
2. The route to have been learned from BGP neighbor `10.0.12.1`.

For conditional route injection, **`match ip route-source` matches the BGP neighbor address from which the parent route was learned**.

It is not necessarily the same as the route's NEXT_HOP attribute.

### Step 3: Configure R2's `inject-map`

Create a prefix list identifying the routes to inject:

```text
R2(config)# ip prefix-list INJECT permit 10.10.1.0/24
R2(config)# ip prefix-list INJECT permit 10.10.2.0/24
```

Create the injection route map:

```text
R2(config)# route-map INJECT-MAP permit 10
R2(config-route-map)# set ip address prefix-list INJECT
```

**Important distinction:** The `inject-map` uses `set ip address prefix-list`, not `match ip address prefix-list`, to specify the new prefixes.

### Step 4: Enable Conditional Route Injection

On R2:

```text
R2(config)# router bgp 65002
R2(config-router)# neighbor 10.0.12.1 remote-as 65001
R2(config-router)# address-family ipv4 unicast
R2(config-router-af)# bgp inject-map INJECT-MAP exist-map EXIST
```

When the parent route matches the existence condition, R2 injects the two more-specific prefixes into its local BGP table.

### Step 5: Verify

On R2:

```text
R2# show ip bgp injected-paths
R2# show bgp ipv4 unicast 10.10.0.0/16
R2# show bgp ipv4 unicast 10.10.1.0/24
R2# show bgp ipv4 unicast 10.10.2.0/24
```

The expected injected prefixes are:

```text
10.10.1.0/24
10.10.2.0/24
```

Check the IP routing table separately:

```text
R2# show ip route 10.10.1.0
R2# show ip route 10.10.2.0
```

**BGP table presence and IP RIB installation are separate decisions.** Verify both before relying on the routes for forwarding.

### Step 6: Remove the Parent Route

On R1:

```text
R1(config)# no ip route 10.10.0.0 255.255.0.0 Null0
```

Assuming no other exact route exists, R1 withdraws its BGP advertisement for `10.10.0.0/16`.

Once R2 no longer has a qualifying parent path, conditional route injection is no longer active and the injected routes are withdrawn.

---

## The `copy-attributes` Option

By default, injected routes use the default attributes for locally originated BGP routes.

To inherit attributes from the parent route, configure:

```text
R2(config-router-af)# bgp inject-map INJECT-MAP exist-map EXIST copy-attributes
```

This causes the injected routes to inherit BGP attributes from the parent route.

For example, inherited attributes can include AS_PATH and other applicable path attributes.

This is useful when injected more-specific prefixes should retain the routing policy information associated with their parent route.

---

## Important Limitations

### Injected Prefixes Must Be Within the Parent Prefix

Suppose the parent route is:

```text
10.10.0.0/16
```

Valid more-specific injected prefixes include:

```text
10.10.1.0/24
10.10.2.0/24
10.10.128.0/17
```

However, the following cannot be injected using this parent:

```text
10.11.0.0/16
192.0.2.0/24
```

They are not contained within `10.10.0.0/16`.

### Route-Map Clause Limitations

Cisco documents that inject maps and existence maps operate on a single matching prefix per route-map clause.

If multiple parent prefixes must be tracked, separate route-map clauses may be required.

An injection map can reference a prefix list containing multiple prefixes to generate multiple injected routes.

### Route Injection vs. Forwarding

Conditional route injection modifies BGP routing information.

It does not guarantee that the injecting router has working forwarding connectivity to every injected destination.

An injected more-specific route can attract traffic that would otherwise follow a less-specific route.

Therefore, the underlying forwarding path must be verified to avoid blackholing traffic.

---

## `advertise-map` vs. `inject-map`

Although both features use route maps and route-existence conditions, they operate differently.

| Characteristic | `advertise-map` | `inject-map` |
|---|---|---|
| Feature | Conditional advertisement | Conditional route injection |
| Main purpose | Control route advertisement | Create new BGP routes |
| Requires advertised/injected prefix to exist beforehand? | Yes | No |
| Requires parent prefix? | No | Yes |
| Conditional existence map | `exist-map` or `non-exist-map` | `exist-map` |
| Scope | Neighbor-specific | Local BGP process/address family |
| Generates new NLRI? | No | Yes |
| Supports `copy-attributes` | No | Yes |
| Common use | Conditional backup advertisements | More-specific traffic engineering |

### Configuration Comparison

**Conditional advertisement:**

```text
R1(config-router-af)# neighbor 10.0.12.2 advertise-map ADV exist-map EXIST
```

This controls whether routes identified by `ADV` are advertised to neighbor `10.0.12.2`.

**Conditional route injection:**

```text
R1(config-router-af)# bgp inject-map INJECT exist-map EXIST
```

This creates new BGP prefixes identified by `INJECT` when the parent route identified by `EXIST` is present.

The injected routes can subsequently participate in normal BGP path selection and advertisement, subject to BGP policies and eligibility rules.

---

## Key Points

- `neighbor advertise-map` conditionally advertises **existing BGP routes**.
- `bgp inject-map` conditionally **creates new BGP routes**.
- Conditional route injection requires an existing parent route in the BGP table.
- The `exist-map` identifies both the parent prefix and its BGP route source.
- The `inject-map` uses `set ip address prefix-list` to identify the injected prefixes.
- Injected prefixes must be contained within the parent route.
- `copy-attributes` makes injected routes inherit attributes from the parent route.
- `show ip bgp injected-paths` displays injected BGP paths.
- Injected routes require valid forwarding reachability to avoid blackholing traffic.

---

## References

- [Cisco: BGP Command Reference, bgp inject-map](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_bgp/command/irg-cr-book/bgp-a1.html)
- [Cisco: BGP Configuration Guide, Understanding BGP](https://www.cisco.com/c/en/us/td/docs/switches/lan/c9000/lyr3-fwd/bgp/bgp-configuration-guide/routing-bgp.html)
