# Inbound Traffic Engineering

**BGP inbound traffic engineering** attempts to influence which external connections other autonomous systems use when sending traffic toward the local AS.

In a multihomed network, an enterprise may advertise the same IP prefixes through multiple ISPs. External networks then select their preferred paths using BGP best-path selection.

Common objectives include:

- Prefer one ISP for inbound traffic.
- Use one ISP as primary and another as backup.
- Attract traffic for different prefixes through different ISPs.
- Reduce inbound traffic over expensive or congested connections.
- Influence inbound traffic using service-provider BGP communities.
- Maintain reachability when a primary ISP connection fails.

Unlike outbound traffic engineering, **inbound traffic engineering is not fully controlled by the destination AS**.

External autonomous systems independently select their preferred routes, potentially overriding the preferences suggested by the enterprise.

---

## Inbound Traffic Engineering Overview

Consider the following network:

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
                        IBGP
                      AS 65000
                           |
                           |
                    203.0.113.0/24
```

AS 65000 advertises `203.0.113.0/24` through both ISPs.

An external router might learn:

```text
203.0.113.0/24 via ISP1
203.0.113.0/24 via ISP2
```

The external router performs its own BGP best-path selection.

The enterprise can manipulate the attributes included in its outgoing BGP advertisements to influence that selection.

For example:

- Advertise a longer AS_PATH through ISP2.
- Advertise a lower MED through ISP1 when both links connect to the same neighboring AS.
- Attach a community requesting that ISP2 lower its LOCAL_PREF.
- Advertise more-specific prefixes through selected ISPs.

**Important:** The direction of the route map and the direction of user traffic are opposite in many inbound traffic-engineering configurations.

An **outbound BGP route map** modifies route advertisements sent to an ISP, which can influence **inbound user traffic** received through that ISP.

---

## Inbound Traffic Engineering Mechanisms

The primary mechanisms are:

| Mechanism | Primary Effect | Scope |
|---|---|---|
| AS_PATH prepending | Makes a path appear longer | Potentially influences multiple external ASes |
| MED | Suggests a preferred entry point | Primarily one neighboring AS |
| BGP communities | Requests provider-specific routing policies | Depends on provider support |
| Selective prefix advertisement | Controls which paths are available | Depends on advertisement propagation |
| More-specific advertisements | Influences forwarding through longest-prefix matching | Networks receiving the more-specific route |
| Conditional advertisement | Controls route advertisement based on BGP conditions | Configured neighbor and downstream propagation |

These mechanisms can be combined.

For example, an enterprise might use:

- AS_PATH prepending to make ISP2 less attractive.
- Provider communities to further reduce preference through ISP2.
- Conditional advertisement to make ISP2 a backup connection.

### Why Inbound Traffic Engineering Is Less Predictable

BGP evaluates multiple attributes before AS_PATH length and MED.

For example, the beginning of Cisco's normal best-path selection process includes:

```text
1. Weight
2. LOCAL_PREF
3. Locally originated routes
4. AIGP (when applicable)
5. AS_PATH length
6. ORIGIN
7. MED (when compared)
...
```

Cost Communities and other features can affect the algorithm.

An external AS can assign LOCAL_PREF values that override the enterprise's AS_PATH prepending or MED.

For example:

| Attribute | Path via ISP1 | Path via ISP2 |
|---|---|---|
| LOCAL_PREF | 100 | 200 |
| AS_PATH length | 2 | 5 |

The path through ISP2 is preferred because LOCAL_PREF is evaluated before AS_PATH length.

**The destination AS cannot force an independent external AS to prefer a particular path simply by manipulating AS_PATH or MED.**

---

## AS_PATH Prepending

**AS_PATH prepending** adds additional occurrences of an ASN to the AS_PATH advertised to a BGP neighbor.

BGP normally prefers a shorter AS_PATH when earlier best-path criteria are equal.

Therefore, prepending can make one connection appear less attractive to external networks.

### Example Topology

```text
                       Remote AS
                       AS 65300
                        /    \
                       /      \
                    ISP1      ISP2
                  AS 65100  AS 65200
                      |       |
                      |       |
                      R1------R2
                        IBGP
                      AS 65000

                   203.0.113.0/24
```

AS 65000 advertises `203.0.113.0/24` through both ISPs.

Without prepending, a remote router might receive:

| Path | AS_PATH |
|---|---|
| Through ISP1 | `65100 65000` |
| Through ISP2 | `65200 65000` |

Both paths have an AS_PATH length of 2.

The preferred path depends on the other BGP attributes.

### Prepending Toward ISP2

The enterprise wants to make ISP1 more attractive.

Configure the router connected to ISP2 to prepend AS 65000 three additional times.

```text
R2(config)# route-map PREPEND-ISP2 permit 10
R2(config-route-map)# set as-path prepend 65000 65000 65000

R2(config)# router bgp 65000
R2(config-router)# address-family ipv4 unicast
R2(config-router-af)# neighbor 10.0.23.3 route-map PREPEND-ISP2 out
```

When R2 advertises a locally originated route through EBGP, its own ASN is also normally added to the AS_PATH.

Therefore, the remote AS might observe:

| Path | AS_PATH |
|---|---|
| Through ISP1 | `65100 65000` |
| Through ISP2 | `65200 65000 65000 65000 65000` |

ISP2's path is now significantly longer.

Assuming earlier best-path criteria are equal, the external AS prefers ISP1.

### Prepending Selected Prefixes

Prepending can be applied selectively.

For example, prepend only `203.0.113.0/24`:

```text
R2(config)# ip prefix-list PREPEND-PFX permit 203.0.113.0/24

R2(config)# route-map PREPEND-ISP2 permit 10
R2(config-route-map)# match ip address prefix-list PREPEND-PFX
R2(config-route-map)# set as-path prepend 65000 65000 65000

R2(config)# route-map PREPEND-ISP2 permit 20
```

Apply the route map outbound toward ISP2:

```text
R2(config)# router bgp 65000
R2(config-router)# address-family ipv4 unicast
R2(config-router-af)# neighbor 10.0.23.3 route-map PREPEND-ISP2 out
```

The result:

- `203.0.113.0/24` receives the additional AS_PATH entries.
- Other eligible routes are advertised normally through sequence 20.

**Important:** Without sequence 20, other routes would be denied by the route map's implicit deny.

### Prepending Different Amounts

An enterprise can use different prepend counts for different providers or prefixes.

For example:

| Connection | Additional Prepends | Intended Preference |
|---|---|---|
| ISP1 | 0 | Most preferred |
| ISP2 | 2 | Less preferred |
| ISP3 | 4 | Least preferred |

However, AS_PATH length does not directly represent bandwidth, latency, reliability, or geographic distance.

It is simply one of the BGP path-selection criteria.

### Limitations of AS_PATH Prepending

AS_PATH prepending is not guaranteed to influence all external networks.

For example, an upstream provider may assign:

```text
Path via ISP1: LOCAL_PREF 100
Path via ISP2: LOCAL_PREF 200
```

The path through ISP2 may still be preferred, regardless of its longer AS_PATH.

Other limitations include:

- Some networks apply policies that override AS_PATH length.
- Some networks may alter or remove prepended ASNs according to their policies.
- Multiple upstream paths may have different attributes.
- An external AS may receive only one of the available advertisements.
- Route changes take time to propagate.

**AS_PATH prepending influences inbound traffic but cannot guarantee its distribution.**

---

## AS_PATH Prepending and Primary/Backup Links

A common use of prepending is to prefer one ISP while keeping another available as backup.

Consider:

```text
       ISP1                      ISP2
         |                         |
         |                         |
         R1------------------------R2
                 AS 65000
```

The enterprise advertises the same prefix through both ISPs.

The requirements are:

1. Prefer ISP1 for normal inbound traffic.
2. Retain ISP2 as an alternative path.
3. Use ISP2 when the ISP1 path becomes unavailable.

Configure AS_PATH prepending toward ISP2.

During normal operation:

```text
Via ISP1: 65100 65000
Via ISP2: 65200 65000 65000 65000
```

If the path through ISP1 is withdrawn, the external network can select the remaining path through ISP2.

This provides a relatively simple primary/backup policy without suppressing backup advertisements.

### Prepending vs. Withdrawing the Backup Route

| Approach | Backup Route Normally Advertised? | Main Characteristic |
|---|---|---|
| AS_PATH prepending | Yes | Backup remains visible, but less attractive |
| Conditional advertisement | Depends on condition | Backup advertisement can be suppressed until needed |
| Selective advertisement | Depends on policy | Backup visibility determined by configured prefix advertisements |

Prepending is generally simpler because the backup path remains advertised and available.

However, the external AS may still prefer the prepended path because of its own routing policies.

---

## Multi-Exit Discriminator (MED)

The **Multi-Exit Discriminator (MED)** is an optional nontransitive BGP path attribute used to suggest a preferred entry point into an AS.

Important characteristics:

- Lower MED is preferred.
- MED is a 32-bit unsigned value.
- Its range is 0 through 4294967295.
- MED is normally preserved through IBGP.
- By default, Cisco normally compares MED only between paths learned from the same neighboring AS.
- MED is evaluated after AS_PATH length and ORIGIN.
- MED is commonly used when two ASes have multiple interconnections.

Unlike AS_PATH prepending, MED is primarily intended to influence the directly neighboring AS.

### MED Example Topology

```text
             Customer AS 65000

              R1----------R2
               |          |
               | EBGP     | EBGP
               |          |
              R3----------R4
                   IBGP

             Provider AS 65100

            203.0.113.0/24
            behind R1/R2
```

AS 65000 advertises `203.0.113.0/24` to AS 65100 through two connections.

The desired behavior is:

- Prefer the connection through R1.
- Use the connection through R2 as backup.

### Configuring MED

On R1:

```text
R1(config)# route-map SET-MED permit 10
R1(config-route-map)# set metric 50

R1(config)# router bgp 65000
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.13.3 route-map SET-MED out
```

On R2:

```text
R2(config)# route-map SET-MED permit 10
R2(config-route-map)# set metric 200

R2(config)# router bgp 65000
R2(config-router)# address-family ipv4 unicast
R2(config-router-af)# neighbor 10.0.24.4 route-map SET-MED out
```

The provider receives:

| Path | MED |
|---|---|
| Through R1 | 50 |
| Through R2 | 200 |

Assuming both paths are available for comparison and earlier best-path criteria are equal, the provider prefers the path through R1.

**Important:** Even when both paths are learned from the same neighboring AS, a provider can override MED using LOCAL_PREF, Weight, or other routing policies.

---

## MED Comparison Rules

MED has several important limitations.

### Comparing Paths from the Same Neighboring AS

By default, Cisco compares MED values only when the paths have the same neighboring AS.

For example:

| Path | Neighboring AS | MED |
|---|---|---|
| Path A | 65000 | 100 |
| Path B | 65000 | 200 |

MED can be compared.

Path A is preferred if earlier criteria are equal.

### Comparing Paths from Different Neighboring ASes

Consider:

| Path | Neighboring AS | MED |
|---|---|---|
| Path A | 65100 | 100 |
| Path B | 65200 | 50 |

By default, Cisco does not normally compare MED between these paths.

Therefore, Path B is not automatically preferred simply because its MED is lower.

The receiving router can enable MED comparison across different neighboring ASes:

```text
R1(config)# router bgp 65000
R1(config-router)# bgp always-compare-med
```

However, **this command must be configured on the router performing best-path selection**.

An enterprise cannot force an external ISP to compare MED values across different ASes.

### Deterministic MED

Cisco also supports:

```text
R1(config-router)# bgp deterministic-med
```

This command makes MED comparisons deterministic by grouping paths from the same neighboring AS before comparing the resulting candidates.

It does not cause MED to be compared across different neighboring ASes.

| Command | Effect |
|---|---|
| `bgp always-compare-med` | Compare MED across different neighboring ASes |
| `bgp deterministic-med` | Make MED comparisons among paths from the same neighboring AS deterministic |

These commands solve different problems.

See [Origin and MED](../../path-selection/attributes/origin-med.md) for detailed MED behavior.

---

## AS_PATH Prepending vs. MED

Both mechanisms can influence inbound traffic, but they operate differently.

| Characteristic | AS_PATH Prepending | MED |
|---|---|---|
| Modified attribute | AS_PATH | MED |
| Preferred value | Shortest | Lowest |
| Typical scope | Multiple external ASes | Direct neighboring AS |
| Works across different ISPs? | Can influence paths across ISPs | Limited by MED comparison rules |
| Evaluated first? | Yes | No |
| Configuration | `set as-path prepend` | `set metric` |
| Can be overridden by external LOCAL_PREF? | Yes | Yes |
| Guaranteed traffic engineering? | No | No |

**General guideline:**

- Use AS_PATH prepending to influence inbound traffic across different ISPs.
- Use MED when controlling which of multiple connections to the same neighboring AS should be preferred.
- Consider provider communities when finer control over upstream policies is required.

---

## Inbound Traffic Engineering with BGP Communities

BGP communities allow an enterprise to request specific routing behavior from a service provider.

Unlike AS_PATH prepending, provider communities can influence attributes such as LOCAL_PREF inside the ISP's own AS.

This can be especially powerful because LOCAL_PREF is evaluated before AS_PATH length.

### Example Provider Community Policy

Suppose ISP1, AS 65100, publishes the following community policy:

| Community | ISP1 Action |
|---|---|
| `65100:50` | Set LOCAL_PREF 50 |
| `65100:100` | Set LOCAL_PREF 100 |
| `65100:200` | Set LOCAL_PREF 200 |
| `65100:300` | Prepend ISP1's ASN toward selected external peers |

These community values are illustrative. Actual providers define their own community values and behavior.

An enterprise can attach these communities to its advertisements.

### Example: Requesting Lower LOCAL_PREF

The enterprise wants ISP1 to treat its route as a less-preferred path.

Configure:

```text
R1(config)# ip bgp-community new-format

R1(config)# route-map ISP1-OUT permit 10
R1(config-route-map)# set community 65100:50 additive
```

Apply the route map outbound toward ISP1:

```text
R1(config)# router bgp 65000
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 route-map ISP1-OUT out
R1(config-router-af)# neighbor 10.0.12.2 send-community standard
```

If ISP1 honors the community, it sets the route's LOCAL_PREF according to its policy.

The route can then become less attractive within ISP1's AS.

### Why Communities Can Be More Effective

Consider two paths within an ISP:

| Attribute | Path A | Path B |
|---|---|---|
| LOCAL_PREF | 100 | 200 |
| AS_PATH length | 2 | 5 |

The ISP prefers Path B because LOCAL_PREF is evaluated before AS_PATH length.

Therefore, a provider community that changes LOCAL_PREF may influence path selection even when AS_PATH prepending would not.

**Important:** The enterprise does not directly advertise LOCAL_PREF to an ordinary EBGP neighbor.

Instead, it advertises a community that the ISP may interpret as a request to modify LOCAL_PREF locally.

### Geographic or Peer-Specific Communities

Some providers support communities that influence route advertisements toward particular:

- Upstream providers.
- Peering networks.
- Geographic regions.
- Customer groups.

For example, a provider might support communities requesting ASN prepending toward particular peers.

This can allow more granular inbound traffic engineering than prepending the route toward the provider itself.

However, each provider has its own community conventions.

Always consult the provider's published community policy.

See [BGP Community Configuration and Filtering](../communities/configuration.md).

---

## More-Specific Prefix Advertisement

Another inbound traffic-engineering mechanism is to advertise more-specific prefixes through selected ISPs.

This relies on **longest-prefix matching**.

Unlike AS_PATH prepending and MED, longest-prefix matching occurs during forwarding-table lookup and takes precedence over BGP attribute comparisons between different prefixes.

### Example

Suppose an enterprise owns:

```text
10.10.0.0/23
```

This contains:

```text
10.10.0.0/24
10.10.1.0/24
```

The enterprise advertises the following routes:

| Prefix | ISP1 | ISP2 |
|---|---|---|
| `10.10.0.0/23` | Yes | Yes |
| `10.10.0.0/24` | Yes | No |
| `10.10.1.0/24` | No | Yes |

*These private prefixes are used only to demonstrate the behavior in a lab. Public Internet deployments require authorized globally routable address space.*

### Expected Traffic Flow

Traffic toward `10.10.0.10` matches:

```text
10.10.0.0/24 via ISP1
10.10.0.0/23 via ISP1 or ISP2
```

The /24 route is more specific, so ISP1 is used.

Traffic toward `10.10.1.10` matches:

```text
10.10.1.0/24 via ISP2
10.10.0.0/23 via ISP1 or ISP2
```

The /24 route through ISP2 is preferred.

This allows the enterprise to attract traffic for different destination ranges through different ISPs.

### Configuration Example

Assume R1 has valid BGP routes for:

```text
10.10.0.0/23
10.10.0.0/24
10.10.1.0/24
```

Create the outbound prefix list for ISP1:

```text
R1(config)# ip prefix-list TO-ISP1 permit 10.10.0.0/23
R1(config)# ip prefix-list TO-ISP1 permit 10.10.0.0/24
```

Create the outbound prefix list for ISP2:

```text
R1(config)# ip prefix-list TO-ISP2 permit 10.10.0.0/23
R1(config)# ip prefix-list TO-ISP2 permit 10.10.1.0/24
```

Apply the prefix lists:

```text
R1(config)# router bgp 65000
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 prefix-list TO-ISP1 out
R1(config-router-af)# neighbor 10.0.13.3 prefix-list TO-ISP2 out
```

The result is:

- ISP1 receives the aggregate and the first /24.
- ISP2 receives the aggregate and the second /24.
- The aggregate provides an alternative route if a more-specific advertisement disappears.

**Important:** Prefix lists do not originate BGP routes. They only filter existing eligible advertisements.

The aggregate and component prefixes must be originated into BGP using appropriate mechanisms.

### Generating More-Specific Routes

A BGP `network` statement requires an exact matching prefix in the routing table.

For example:

```text
R1(config)# router bgp 65000
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# network 10.10.0.0 mask 255.255.254.0
R1(config-router-af)# network 10.10.0.0 mask 255.255.255.0
R1(config-router-af)# network 10.10.1.0 mask 255.255.255.0
```

Each `network` statement requires an exact matching route in the appropriate RIB.

Alternatively, BGP aggregation or conditional route injection can be used where appropriate.

See [BGP Conditional Route Injection](../conditional-route-injection.md).

---

## Limitations of More-Specific Advertisements

More-specific prefix advertisement can be powerful, but it has several limitations.

### Internet Prefix-Length Filtering

Many Internet networks filter IPv4 prefixes longer than /24.

Therefore, advertising a /25 or /26 may not provide reliable global reachability.

For IPv6, many providers commonly apply filtering policies around /48, although acceptance varies.

### Prefix Authorization

The enterprise must be authorized to originate the advertised prefixes.

For example:

- The prefix must be allocated or delegated appropriately.
- Upstream prefix filters must permit the advertisement.
- RPKI Route Origin Authorizations (ROAs), where used, must authorize the originating ASN and prefix length.

A more-specific advertisement can be rejected if its prefix length exceeds the relevant ROA's maximum length.

### Routing-Table Growth

Advertising more-specific prefixes increases the number of routes carried by external networks.

Excessive deaggregation contributes to global routing-table growth.

### Failover Behavior

Suppose an enterprise advertises:

```text
10.10.0.0/23 via ISP1 and ISP2
10.10.0.0/24 via ISP1 only
```

During normal operation, traffic to `10.10.0.10` generally follows the /24 through ISP1.

If ISP1 withdraws the /24, remote networks can use the remaining /23 through ISP2.

However, this requires the aggregate to remain available and accepted through ISP2.

**Important:** More-specific advertisements influence forwarding only in networks that actually receive and install those advertisements.

---

## Conditional Advertisement for Backup Connections

**BGP conditional advertisement** allows a router to advertise selected routes based on the presence or absence of other routes in its local BGP table.

This is useful when an enterprise wants a backup advertisement to become active only under specified conditions.

For example:

```text
                   Internet
                      |
                 +----+----+
                 |         |
                ISP1      ISP2
                 |         |
                 R1--------R2
                    IBGP
                  AS 65000
```

The enterprise wants to advertise `203.0.113.0/24` through ISP1 during normal operation.

ISP2 should receive a conditional advertisement when a monitored primary-path route disappears.

### Conditional Advertisement Commands

The general syntax is:

```text
neighbor <IP> advertise-map <ADV> exist-map <CONDITION>
```

Or:

```text
neighbor <IP> advertise-map <ADV> non-exist-map <CONDITION>
```

The components are:

| Component | Purpose |
|---|---|
| `advertise-map` | Identifies the routes to advertise conditionally |
| `exist-map` | Requires a matching BGP route to exist |
| `non-exist-map` | Requires a matching BGP route to be absent |

### Example: Backup Advertisement

Suppose R2 is the backup edge router.

The enterprise prefix is:

```text
203.0.113.0/24
```

R2 also learns a monitoring prefix from the primary ISP path:

```text
198.51.100.0/24
```

The objective is to advertise `203.0.113.0/24` to ISP2 when the monitoring prefix is absent from R2's BGP table.

Configure the prefix lists:

```text
R2(config)# ip prefix-list BACKUP-PFX permit 203.0.113.0/24
R2(config)# ip prefix-list PRIMARY-CHECK permit 198.51.100.0/24
```

Configure the route maps:

```text
R2(config)# route-map ADVERTISE-BACKUP permit 10
R2(config-route-map)# match ip address prefix-list BACKUP-PFX

R2(config)# route-map PRIMARY-CONDITION permit 10
R2(config-route-map)# match ip address prefix-list PRIMARY-CHECK
```

Apply conditional advertisement toward ISP2:

```text
R2(config)# router bgp 65000
R2(config-router)# address-family ipv4 unicast
R2(config-router-af)# neighbor 10.0.23.3 advertise-map ADVERTISE-BACKUP non-exist-map PRIMARY-CONDITION
```

### Expected Behavior

| Monitoring Prefix Present? | Backup Prefix Advertised? |
|---|---|
| Yes | No |
| No | Yes |

The advertise-map prefix must itself exist in R2's BGP table and be eligible for advertisement.

**Important:** The monitored route must accurately represent the state of the primary path.

If the monitoring prefix remains in the BGP table through another path, the `non-exist-map` condition may remain false even after the primary connection fails.

Additionally, the absence of a particular BGP route does not necessarily mean that the physical connection or upstream Internet connectivity has failed.

Conditional advertisement is based on BGP routing information, not a direct end-to-end reachability test.

### Conditional Advertisement vs. Prepending

| Feature | AS_PATH Prepending | Conditional Advertisement |
|---|---|---|
| Backup route normally visible? | Yes | Can be suppressed |
| Influences best-path selection? | Yes | Controls route availability |
| Depends on BGP route conditions? | No | Yes |
| Can completely avoid backup advertisements during normal operation? | No | Yes |
| External AS still controls path selection? | Yes | Yes, among available paths |

Conditional advertisement offers more explicit advertisement control but introduces an additional dependency on the monitored condition.

See [BGP Conditional Advertisement](../conditional-advertisement.md).

---

## Complete Example: AS_PATH Prepending Toward a Backup ISP

The following lab demonstrates inbound traffic engineering using a router connected to two ISPs.

### Topology

```text
              ISP1                      ISP2
            AS 65100                  AS 65200
          10.0.12.2                 10.0.13.3
               |                         |
               |                         |
          E0/0 |                         | E0/1
               +----------- R1 ----------+
                          AS 65000
                              |
                              |
                        Loopback0
                     203.0.113.1/24
```

R1 originates `203.0.113.0/24` into BGP.

Requirements:

1. Advertise the prefix to both ISPs.
2. Prefer ISP1 for inbound traffic.
3. Make ISP2 less attractive using AS_PATH prepending.
4. Preserve ISP2 as a backup path.

### Step 1: Configure R1 Interfaces

```text
R1(config)# interface Ethernet0/0
R1(config-if)# ip address 10.0.12.1 255.255.255.252
R1(config-if)# no shutdown

R1(config)# interface Ethernet0/1
R1(config-if)# ip address 10.0.13.1 255.255.255.252
R1(config-if)# no shutdown

R1(config)# interface Loopback0
R1(config-if)# ip address 203.0.113.1 255.255.255.0
```

The connected Loopback0 network provides the exact route required by the BGP `network` statement.

### Step 2: Configure the Prefix List

```text
R1(config)# ip prefix-list LOCAL-PFX permit 203.0.113.0/24
```

This prefix list identifies the route that R1 is authorized to advertise in the lab.

### Step 3: Configure the Route Map

Configure a route map that prepends the local ASN three times:

```text
R1(config)# route-map TO-ISP2 permit 10
R1(config-route-map)# match ip address prefix-list LOCAL-PFX
R1(config-route-map)# set as-path prepend 65000 65000 65000
```

Only routes matching `LOCAL-PFX` are permitted by this route map.

Other routes are denied by the implicit deny.

### Step 4: Configure BGP

```text
R1(config)# router bgp 65000
R1(config-router)# bgp router-id 1.1.1.1
R1(config-router)# neighbor 10.0.12.2 remote-as 65100
R1(config-router)# neighbor 10.0.13.3 remote-as 65200

R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 activate
R1(config-router-af)# neighbor 10.0.13.3 activate
R1(config-router-af)# network 203.0.113.0 mask 255.255.255.0
R1(config-router-af)# neighbor 10.0.12.2 prefix-list LOCAL-PFX out
R1(config-router-af)# neighbor 10.0.13.3 route-map TO-ISP2 out
```

The outbound prefix list toward ISP1 permits only the authorized prefix.

The route map toward ISP2 permits the same prefix and adds the AS_PATH prepends.

### Step 5: Configure ISP1

On R2, representing ISP1:

```text
R2(config)# interface Ethernet0/0
R2(config-if)# ip address 10.0.12.2 255.255.255.252
R2(config-if)# no shutdown

R2(config)# router bgp 65100
R2(config-router)# bgp router-id 2.2.2.2
R2(config-router)# neighbor 10.0.12.1 remote-as 65000
R2(config-router)# address-family ipv4 unicast
R2(config-router-af)# neighbor 10.0.12.1 activate
```

### Step 6: Configure ISP2

On R3, representing ISP2:

```text
R3(config)# interface Ethernet0/0
R3(config-if)# ip address 10.0.13.3 255.255.255.252
R3(config-if)# no shutdown

R3(config)# router bgp 65200
R3(config-router)# bgp router-id 3.3.3.3
R3(config-router)# neighbor 10.0.13.1 remote-as 65000
R3(config-router)# address-family ipv4 unicast
R3(config-router-af)# neighbor 10.0.13.1 activate
```

### Step 7: Verify on ISP1

```text
R2# show bgp ipv4 unicast 203.0.113.0/24
```

The expected AS_PATH is:

```text
65000
```

### Step 8: Verify on ISP2

```text
R3# show bgp ipv4 unicast 203.0.113.0/24
```

The expected AS_PATH is:

```text
65000 65000 65000 65000
```

The AS_PATH includes:

- Three additional occurrences configured by `set as-path prepend`.
- One occurrence added by normal EBGP AS_PATH processing.

### Step 9: Verify External Path Selection

To observe the actual inbound traffic-engineering effect, connect ISP1 and ISP2 to a common external AS.

That external router should receive both advertisements.

For example:

```text
Via ISP1: 65100 65000
Via ISP2: 65200 65000 65000 65000 65000
```

If the external AS assigns equal preference to both paths and earlier best-path criteria do not differ, it should prefer ISP1 because of the shorter AS_PATH.

**Important:** Without a common external router receiving both paths, the lab verifies AS_PATH manipulation but not the resulting external best-path decision.

---

## Selective Advertisement for Inbound Traffic Distribution

An enterprise can combine more-specific advertisements with AS_PATH prepending or communities.

For example, suppose an enterprise has two public /24 prefixes:

```text
203.0.113.0/24
192.0.2.0/24
```

The enterprise wants:

- `203.0.113.0/24` to prefer ISP1.
- `192.0.2.0/24` to prefer ISP2.

Both prefixes should remain reachable through either ISP during a failure.

One option is to advertise both prefixes to both ISPs but apply different prepend policies.

### ISP1 Policy

Prepend only `192.0.2.0/24`:

```text
R1(config)# ip prefix-list VIA-ISP2 permit 192.0.2.0/24

R1(config)# route-map TO-ISP1 permit 10
R1(config-route-map)# match ip address prefix-list VIA-ISP2
R1(config-route-map)# set as-path prepend 65000 65000 65000

R1(config)# route-map TO-ISP1 permit 20
```

### ISP2 Policy

Prepend only `203.0.113.0/24`:

```text
R2(config)# ip prefix-list VIA-ISP1 permit 203.0.113.0/24

R2(config)# route-map TO-ISP2 permit 10
R2(config-route-map)# match ip address prefix-list VIA-ISP1
R2(config-route-map)# set as-path prepend 65000 65000 65000

R2(config)# route-map TO-ISP2 permit 20
```

Apply each route map outbound toward the corresponding ISP.

### Expected Advertisement Attributes

| Prefix | ISP1 Advertisement | ISP2 Advertisement |
|---|---|---|
| `203.0.113.0/24` | Normal AS_PATH | Prepended AS_PATH |
| `192.0.2.0/24` | Prepended AS_PATH | Normal AS_PATH |

This encourages remote networks to prefer different ISPs for the two prefixes.

Both paths remain available as alternatives.

**Important:** This example uses documentation prefixes for lab purposes. Production deployments must use authorized public prefixes and provider-approved advertisement policies.

---

## Controlling Inbound Traffic with Route Filtering

Outbound BGP filtering can control which routes are advertised through each ISP.

For example:

```text
R1(config)# ip prefix-list TO-ISP1 permit 203.0.113.0/24

R1(config)# router bgp 65000
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 prefix-list TO-ISP1 out
```

This permits the selected prefix to be advertised to ISP1, provided the route is eligible.

Other prefixes are denied.

### Risks of Selective Advertisement

Suppose a prefix is advertised only through ISP1.

If ISP1 fails, the prefix may become unreachable from external networks because ISP2 has no route to advertise.

Therefore, selective advertisement must account for failover requirements.

Possible solutions include:

- Advertising aggregates through both ISPs.
- Advertising backup routes with less-preferred attributes.
- Conditional advertisement.
- Provider-supported backup-routing policies.

**Important:** Filtering a prefix from a backup ISP is not equivalent to making its route less preferred.

Filtering removes the advertisement entirely, potentially eliminating the backup path.

---

## Inbound Traffic Engineering and Asymmetric Routing

Inbound and outbound traffic engineering are independent.

An enterprise might configure:

```text
Outbound traffic: Prefer ISP1
Inbound traffic:  Prefer ISP2
```

This produces asymmetric routing:

```text
Enterprise -----> ISP1 -----> Internet
Enterprise <----- ISP2 <----- Internet
```

Asymmetric routing is normally valid at the IP forwarding level.

However, it can cause problems with:

- Stateful firewalls.
- NAT.
- Security policies.
- Traffic monitoring.
- Troubleshooting.

For example, if traffic leaves through a firewall on ISP1 but returns through a different stateful firewall on ISP2, the return traffic might be rejected because the second firewall does not have the corresponding connection state.

NAT can introduce additional complications when different ISP connections use different public addresses.

**Inbound traffic engineering must be coordinated with the network's security and NAT architecture.**

---

## Verifying Inbound Traffic Engineering

Inbound traffic engineering should be verified from both the advertising and receiving perspectives.

### Verify Local BGP Routes

```text
R1# show bgp ipv4 unicast
```

This confirms that the prefixes exist in the local BGP table.

However, it does not necessarily show the attributes that will be sent to each neighbor after outbound route-map processing.

### Verify Advertised Routes

```text
R1# show bgp ipv4 unicast neighbors 10.0.12.2 advertised-routes
```

This helps confirm which prefixes are eligible for advertisement to the neighbor.

For AS_PATH and MED manipulation, inspect the detailed route on the **receiving ISP router** to verify the transmitted attributes.

### Verify AS_PATH on the ISP

```text
R2# show bgp ipv4 unicast 203.0.113.0/24
```

Check:

- AS_PATH length.
- Number of prepended ASNs.
- Neighboring AS.
- Best-path selection.

### Verify MED

```text
R2# show bgp ipv4 unicast 203.0.113.0/24
```

Look for the MED value, normally displayed as the route metric.

Verify that the provider sees the expected MED on each path.

### Verify Communities

```text
R2# show bgp ipv4 unicast 203.0.113.0/24
```

Check the Community attribute.

Verify that the ISP actually received the configured community.

If the community requests an ISP-specific policy, confirm whether the provider applied the expected policy.

### Verify Conditional Advertisement

```text
R1# show bgp ipv4 unicast neighbors 10.0.12.2 advertised-routes
```

Check which routes are advertised when the condition is present.

Then remove the monitored route in a controlled lab and verify that the conditional advertisement changes.

### Verify External Best-Path Selection

The most meaningful verification is performed on an external router that receives advertisements from both ISPs.

For example:

```text
R4# show bgp ipv4 unicast 203.0.113.0/24
```

Compare:

- LOCAL_PREF.
- AS_PATH.
- ORIGIN.
- MED, if applicable.
- Best-path status.

This reveals whether the traffic-engineering policy actually influenced the external router's path selection.

### Verify with Traceroute

For a realistic inbound-path test, initiate traceroute from an external network toward an address inside the enterprise prefix.

For example:

```text
R4# traceroute 203.0.113.1
```

The observed path can help confirm the selected ISP connection.

However, traceroute results may be incomplete because intermediate routers can filter, rate-limit, or handle traceroute packets differently.

---

## Applying Policy Changes

After changing an outbound BGP route map, the advertising router may need to reprocess its outbound advertisements.

Use:

```text
R1# clear ip bgp 10.0.12.2 soft out
```

This reapplies the outbound routing policy without resetting the established BGP session.

For example, after changing:

```text
set as-path prepend 65000 65000 65000
```

to:

```text
set as-path prepend 65000 65000
```

an outbound soft clear can be used to regenerate the advertisements with the updated AS_PATH.

However, external networks may not immediately change their best paths.

Their convergence depends on:

- When the updated BGP advertisements arrive.
- External route policies.
- BGP best-path selection.
- Route propagation.
- Forwarding-table updates.

**Changing a BGP advertisement does not guarantee immediate inbound traffic movement.**

---

## Troubleshooting

| Problem | Possible Cause |
|---|---|
| AS_PATH prepending has no visible effect | External LOCAL_PREF or other earlier BGP criteria override it |
| Prepending not visible on ISP | Outbound route map not applied, match failed, or update not refreshed |
| MED does not influence routing | Paths from different neighboring ASes, earlier criteria differ, or ISP ignores MED |
| Provider community has no effect | Unsupported community value or provider policy |
| Community missing on ISP | `send-community` not configured |
| More-specific prefix not visible externally | Provider prefix-length filter or route authorization issue |
| Backup prefix disappears during failure | Backup ISP does not receive an eligible advertisement |
| Conditional advertisement does not activate | Condition route remains in the BGP table |
| Conditional advertisement activates unexpectedly | Monitored BGP route disappeared despite usable primary connectivity |
| Route unexpectedly filtered | Outbound prefix list or route-map implicit deny |
| Inbound traffic remains asymmetric | Independent inbound and outbound routing decisions |
| Local advertisement looks correct but traffic uses another ISP | Remote AS best-path selection differs from intended policy |

### Troubleshooting Checklist

1. Verify that the enterprise originates the expected BGP prefixes.
2. Confirm that the outbound policy matches the intended routes.
3. Verify the correct route-map direction.
4. Check the AS_PATH, MED, or communities received by each ISP.
5. Confirm that `send-community` is configured where needed.
6. Verify the external AS's best-path selection.
7. Check for provider policies that override the intended preference.
8. Verify prefix-length filters and route origin authorization.
9. Confirm that backup advertisements remain available.
10. Test failover by removing the primary connection in a controlled lab.
11. Verify inbound forwarding from an external source.
12. Reapply outbound policies with a soft clear when necessary.

---

## Key Points

- **Inbound traffic engineering influences how external networks reach the local AS.**
- It is generally less deterministic than outbound traffic engineering.
- Inbound traffic engineering commonly uses outbound BGP route maps.
- AS_PATH prepending makes a route appear to traverse a longer AS path.
- A shorter AS_PATH is preferred only when earlier BGP best-path criteria do not determine the result.
- AS_PATH prepending can influence external ASes but does not guarantee traffic distribution.
- MED suggests a preferred entry point into an AS.
- Lower MED is preferred.
- By default, Cisco normally compares MED only between paths from the same neighboring AS.
- `bgp always-compare-med` changes comparison behavior on the router where it is configured.
- `bgp deterministic-med` does not enable cross-AS MED comparison.
- Provider communities can request LOCAL_PREF changes and other routing policies.
- Community behavior depends on the receiving ISP's configuration.
- More-specific prefix advertisements can influence inbound traffic through longest-prefix matching.
- Prefix deaggregation is subject to filtering and route origin authorization.
- Conditional advertisement can control backup advertisements based on BGP route existence.
- Filtering a prefix from a backup ISP can eliminate the backup path entirely.
- Inbound and outbound traffic engineering can produce asymmetric routing.
- Verification must include the attributes and best-path decisions observed by external BGP routers.
- Changing advertised attributes does not guarantee immediate traffic movement.

---

## References

- [Cisco: Connecting to a Service Provider Using External BGP](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-routing/b-ip-routing/m_irg-external-sp-0.html)
- [Cisco: Configure BGP with Two Different Service Providers](https://www.cisco.com/c/en/us/support/docs/ip/border-gateway-protocol-bgp/217964-configure-sample-for-bgp-with-two-differ.html)
- [Cisco: Select BGP Best-path Algorithm](https://www.cisco.com/c/en/us/support/docs/ip/border-gateway-protocol-bgp/13753-25.html)
- [Cisco: Use MultiExit Discriminator to Influence BGP Best Path Selection](https://www.cisco.com/c/en/us/support/docs/ip/border-gateway-protocol-bgp/13759-37.html)
- [Cisco: Configure an Upstream Provider Network with BGP Community Values](https://www.cisco.com/c/en/us/support/docs/ip/border-gateway-protocol-bgp/28784-bgp-community.html)
- [Cisco: Configure and Verify BGP Conditional Advertisement](https://www.cisco.com/c/en/us/support/docs/ip/border-gateway-protocol-bgp/16137-cond-adv.html)
- [Cisco: Understand Load Sharing with BGP in Single/Multihomed Environments](https://www.cisco.com/c/en/us/support/docs/ip/border-gateway-protocol-bgp/13762-40.html)
- [Cisco: BGP Frequently Asked Questions](https://www.cisco.com/c/en/us/support/docs/ip/border-gateway-protocol-bgp/5816-bgpfaq-5816.html)
