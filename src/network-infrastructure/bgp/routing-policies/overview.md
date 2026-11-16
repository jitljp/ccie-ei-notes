# 1.5.c Routing Policies

**BGP routing policies** allow administrators to control which routes are accepted, which routes are advertised, and how BGP path attributes are manipulated.

Unlike IGPs such as OSPF and EIGRP, BGP is designed to support extensive policy-based control over routing decisions.

Routing policies are commonly used to:

- **Filter routes** received from or advertised to BGP neighbors.
- **Manipulate path attributes** to influence best-path selection.
- **Control route advertisements** based on specified conditions.
- **Tag routes with communities** to simplify policy application.
- **Implement traffic engineering** in multihomed networks.

On Cisco IOS XE, BGP routing policies are primarily implemented using **prefix lists, AS-path access lists, community lists, and route maps**.

---

## Inbound and Outbound Routing Policies

BGP routing policies can be applied in two directions:

- **Inbound policies** control how routes received from a neighbor are processed.
- **Outbound policies** control which routes are advertised to a neighbor and how their attributes are modified.

### Inbound Routing Policies

Inbound policies are applied when a router receives BGP UPDATE messages from a neighbor.

They can:

- Accept or reject received routes.
- Modify attributes such as Weight, LOCAL_PREF, MED, and communities.
- Influence which paths are selected as best.

```text
    BGP Neighbor
         |
         | BGP UPDATE
         v
  +----------------+
  | Inbound Policy |
  +----------------+
         |
         v
    BGP Table
         |
         v
  Best-Path Selection
         |
         v
   RIB Installation
```

For example, an inbound route map can assign a higher LOCAL_PREF to routes received from a preferred ISP.

**Inbound filtering occurs before best-path selection.** A route rejected by an inbound policy is not considered by the local router's BGP best-path algorithm.

### Outbound Routing Policies

Outbound policies control routing information advertised to BGP neighbors.

They can:

- Permit or deny advertisement of specific prefixes.
- Modify attributes such as AS_PATH, MED, and communities.
- Influence routing decisions made by neighboring autonomous systems.

```text
    BGP Table
         |
         v
  Advertisement
    Selection
         |
         v
 +-----------------+
 | Outbound Policy |
 +-----------------+
         |
         | BGP UPDATE
         v
    BGP Neighbor
```

For example, an outbound route map can prepend the local ASN to selected routes to make a particular path less attractive to external networks.

**Outbound policies do not directly influence the local router's best-path selection.** They modify or filter information sent to other BGP speakers.

By default, BGP normally advertises its selected best path for a prefix to eligible neighbors, subject to BGP advertisement rules and configured policies.

---

## BGP Routing Policy Tools

Cisco IOS XE supports several mechanisms for implementing BGP policies.

| Tool | Purpose |
|---|---|
| Prefix list | Match or filter routes based on network prefixes and prefix lengths |
| Standard or extended ACL | Match IPv4 prefixes for route filtering |
| AS-path access list | Match routes based on AS_PATH using regular expressions |
| Community list | Match routes based on BGP community values |
| Route map | Combine match conditions with filtering or attribute modifications |
| Filter list | Apply AS-path access lists to BGP neighbors |
| Distribute list | Apply ACL-based route filtering to BGP neighbors |
| Conditional advertisement | Advertise routes based on the presence or absence of other BGP routes |
| Outbound Route Filtering (ORF) | Allow a BGP peer to request that its neighbor filter outgoing advertisements |

### Prefix Lists

Prefix lists match routes based on their prefixes and prefix lengths.

For example:

```text
R1(config)# ip prefix-list FILTER permit 10.10.0.0/16 le 24
```

This permits routes within `10.10.0.0/16` with prefix lengths between `/16` and `/24`, inclusive.

Prefix lists are commonly used for both inbound and outbound BGP filtering.

### AS-Path Access Lists

AS-path access lists use regular expressions to match a route's AS_PATH.

For example:

```text
R1(config)# ip as-path access-list 10 permit ^65002$
```

This matches routes whose AS_PATH consists of exactly AS 65002.

AS-path access lists can be used directly with BGP filter lists or as match conditions in route maps.

### Community Lists

Community lists match routes based on their BGP communities.

For example:

```text
R1(config)# ip community-list standard CUSTOMER permit 65001:100
```

This permits routes that carry community `65001:100`.

Community lists are frequently used with route maps to apply common policies to groups of prefixes.

### Route Maps

**Route maps** are among the most flexible BGP policy tools.

They allow administrators to:

1. Match routes based on specified conditions.
2. Permit or deny matching routes.
3. Modify attributes using `set` commands.

For example:

```text
R1(config)# route-map PREFER-ISP permit 10
R1(config-route-map)# match ip address prefix-list CUSTOMER
R1(config-route-map)# set local-preference 200
```

This route map assigns LOCAL_PREF 200 to routes matching the prefix list `CUSTOMER`.

When used for BGP route filtering, a route map has an **implicit deny** at the end. Routes that do not match a permitting sequence are rejected.

The interaction between prefix lists, AS-path ACLs, and route maps is covered in [BGP Prefix Lists, AS-Path ACLs, and Route Maps](filtering-tools.md).

---

## Basic Routing Policy Configuration

Consider two EBGP neighbors:

```text
           AS 65001                 AS 65002

              R1 --------------------- R2
        10.0.12.1                    10.0.12.2
```

R1 should:

1. Accept only `203.0.113.0/24` from R2.
2. Set LOCAL_PREF to 200 for that route.
3. Advertise only `192.0.2.0/24` to R2.

### Configuration

```text
R1(config)# ip prefix-list INBOUND permit 203.0.113.0/24
R1(config)# ip prefix-list OUTBOUND permit 192.0.2.0/24

R1(config)# route-map FROM-R2 permit 10
R1(config-route-map)# match ip address prefix-list INBOUND
R1(config-route-map)# set local-preference 200

R1(config)# router bgp 65001
R1(config-router)# neighbor 10.0.12.2 remote-as 65002
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 route-map FROM-R2 in
R1(config-router-af)# neighbor 10.0.12.2 prefix-list OUTBOUND out
```

### Result

**Inbound:**

- R1 accepts `203.0.113.0/24` from R2.
- The route is assigned LOCAL_PREF 200.
- Other prefixes received from R2 are denied by the route map's implicit deny.

**Outbound:**

- R1 is permitted to advertise `192.0.2.0/24` to R2, provided that it is eligible for advertisement.
- Other prefixes are denied by the outbound prefix list.

The inbound and outbound policies operate independently.

> **Important:** A routing policy does not necessarily cause a route to be advertised or installed. BGP advertisement eligibility, best-path selection, next-hop reachability, and RIB installation rules still apply.

---

## Routing Policies and Traffic Engineering

BGP routing policies are frequently used to implement **traffic engineering**.

Traffic engineering involves influencing the paths used to forward traffic through a network.

Two important distinctions are:

| Direction | Objective | Common Techniques |
|---|---|---|
| Outbound traffic engineering | Control how traffic leaves the local AS | Weight, LOCAL_PREF, filtering |
| Inbound traffic engineering | Influence how external networks send traffic into the local AS | AS_PATH prepending, MED, communities, selective advertisement |

For example, an enterprise connected to two ISPs might:

- Set a higher LOCAL_PREF on routes learned from ISP1 to prefer it for outbound traffic.
- Prepend its ASN when advertising selected prefixes to ISP2 to make ISP1 more attractive for inbound traffic.
- Advertise communities to request specific routing policies from its providers.

**Outbound traffic engineering is generally more deterministic** because the local AS controls its own BGP best-path selection.

Inbound traffic engineering is less predictable because external autonomous systems independently select their preferred routes.

See [BGP Multihoming](multihoming/overview.md) for detailed examples.

---

## Advanced BGP Routing Policies

### Attribute Manipulation

BGP attributes can be modified using route maps to influence path selection.

Common examples include:

- `set weight`
- `set local-preference`
- `set metric`
- `set as-path prepend`
- `set community`
- `set extcommunity`

See [BGP Attribute Manipulation](attribute-manipulation.md).

### Conditional Advertisement

Conditional advertisement allows a router to advertise selected routes depending on whether another BGP route exists.

It uses:

- `advertise-map`: Identifies routes to advertise conditionally.
- `exist-map`: Advertises the selected routes when a matching route exists.
- `non-exist-map`: Advertises the selected routes when a matching route does not exist.

This is useful for controlling route advertisements during primary-link failures.

See [BGP Conditional Advertisement](conditional-advertisement.md).

### Outbound Route Filtering (ORF)

**BGP Outbound Route Filtering (ORF)** allows a BGP router to communicate filtering requirements to a neighbor.

Instead of receiving unwanted routes and filtering them locally, the router can request that its neighbor filter those routes before advertising them.

This can reduce:

- Unnecessary BGP UPDATE messages.
- Control-plane processing.
- Resource consumption associated with unwanted routing information.

ORF requires compatible capabilities to be negotiated between the BGP peers.

See [BGP Outbound Route Filtering](orf.md).

### BGP Communities

**BGP communities** provide a way to tag routes with additional information that can be used by routing policies.

Two important categories are:

- **Standard communities:** 32-bit values commonly used to group routes and control their advertisement or preference.
- **Extended communities:** 64-bit values supporting additional applications such as route targets, site-of-origin, and other routing policies.

Communities do not inherently change BGP best-path selection. They can, however, be matched by routing policies that modify path attributes or control route advertisement.

See [Standard and Extended Communities](communities/overview.md).

---

## Verification and Troubleshooting

Useful commands for verifying BGP policies include:

```text
R1# show bgp ipv4 unicast
R1# show bgp ipv4 unicast 203.0.113.0/24
R1# show bgp ipv4 unicast neighbors 10.0.12.2 routes
R1# show bgp ipv4 unicast neighbors 10.0.12.2 advertised-routes
R1# show ip prefix-list
R1# show ip as-path-access-list
R1# show ip community-list
R1# show route-map
```

| Command | Purpose |
|---|---|
| `show bgp ipv4 unicast` | Display BGP routes and their selected best paths |
| `show bgp ipv4 unicast <prefix>` | Examine the attributes of a particular route |
| `show bgp ipv4 unicast neighbors <IP> routes` | Display routes accepted from a neighbor |
| `show bgp ipv4 unicast neighbors <IP> advertised-routes` | Display routes advertised to a neighbor |
| `show ip prefix-list` | Display configured prefix lists and match statistics |
| `show ip as-path-access-list` | Display AS-path access lists |
| `show ip community-list` | Display configured community lists |
| `show route-map` | Display route maps and available match statistics |

### Applying Policy Changes

After changing an inbound routing policy, previously received routes may need to be reprocessed.

A soft inbound clear can trigger route refresh when the neighbor supports it:

```text
R1# clear ip bgp 10.0.12.2 soft in
```

To reapply outbound routing policies:

```text
R1# clear ip bgp 10.0.12.2 soft out
```

Soft clears normally allow policies to be reapplied without tearing down the established BGP session.

See [BGP Route Refresh](../other-features/route-refresh.md) for details.

---

## Topics Covered

This section covers the following BGP routing-policy mechanisms:

1. [BGP Route Filtering](route-filtering.md): Inbound and outbound filtering, filtering behavior, and policy application.
2. [BGP Prefix Lists, AS-Path ACLs, and Route Maps](filtering-tools.md): Matching routes and constructing routing policies.
3. [BGP Attribute Manipulation](attribute-manipulation.md): Modifying BGP attributes using routing policies.
4. [BGP Conditional Advertisement](conditional-advertisement.md): Controlling advertisements based on route existence.
5. [BGP Outbound Route Filtering](orf.md): Exchanging prefix-based filtering requirements between BGP neighbors.
6. [Standard and Extended Communities](communities/overview.md): Tagging routes and applying community-based policies.
7. [BGP Multihoming](multihoming/overview.md): Inbound and outbound traffic engineering with multiple external connections.

AS_PATH-specific manipulation, including `local-as`, `allowas-in`, `remove-private-as`, AS_PATH prepending, and regular expressions, is covered separately in [1.5.d AS Path Manipulations](../as-path/overview.md).

---

## References

- [Cisco: Connecting to a Service Provider Using External BGP](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-routing/b-ip-routing/m_irg-external-sp-0.html)
- [Cisco: BGP Frequently Asked Questions](https://www.cisco.com/c/en/us/support/docs/ip/border-gateway-protocol-bgp/5816-bgpfaq-5816.html)
- [Cisco: Configure an Upstream Provider Network with BGP Community Values](https://www.cisco.com/c/en/us/support/docs/ip/border-gateway-protocol-bgp/28784-bgp-community.html)
