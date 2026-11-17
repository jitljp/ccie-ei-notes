# 1.5.c (v) Multihoming

**BGP multihoming** refers to connecting an autonomous system to multiple external network connections to improve redundancy, availability, and traffic engineering.

A multihomed enterprise can use BGP routing policies to control:

- **Outbound traffic:** Which external connection is preferred when sending traffic to other networks.
- **Inbound traffic:** Which external connection other networks prefer when sending traffic toward the enterprise.
- **Failover:** How traffic is redirected when a connection or routing path becomes unavailable.
- **Load sharing:** How traffic can be distributed across multiple connections.
- **Route advertisement:** Which prefixes are advertised to each provider.

BGP multihoming is commonly implemented with two or more Internet Service Providers (ISPs), although similar designs are used in private WANs and MPLS VPN environments.

The primary objectives are usually **redundancy and traffic engineering**.

---

## Multihoming Terminology

BGP multihoming designs can be classified according to the number of connections, providers, and edge routers.

### Single-Homed

A single-homed network has one connection to one ISP.

```text
       Enterprise                ISP

          R1 -------------------- R2
               Single Connection
```

This is the simplest design but has no redundant external connection.

If the link or ISP becomes unavailable, the enterprise loses connectivity through that connection.

### Dual-Homed to One ISP

An enterprise has two connections to the same ISP.

```text
        Enterprise                ISP
                                +------+
          R1 -------------------|      |
             \                  |      |
              \-----------------|      |
                                +------+
```

This provides redundancy against an individual connection failure.

However, it does not necessarily protect against a failure affecting the entire ISP.

The connections can be configured for:

- Primary/backup operation.
- Load sharing.
- Different traffic preferences.

### Multihomed to Multiple ISPs

An enterprise connects to two or more different ISPs.

```text
                         ISP1
                       AS 65002
                          |
                          |
                          R1 AS 65001
                          |
                          |
                       AS 65003
                         ISP2
```

This design provides provider diversity.

If one ISP becomes unavailable, traffic can potentially use the other ISP.

It also allows the enterprise to influence inbound and outbound traffic independently.

### Multiple Edge Routers

For additional redundancy, enterprises often deploy separate edge routers.

```text
        ISP1                         ISP2
      AS 65002                     AS 65003
         |                            |
         |                            |
         R1 ------------------------- R2
         |           IBGP             |
         |                            |
         +------------+---------------+
                      |
                Internal Network
                   AS 65001
```

In this design:

- R1 maintains an EBGP session with ISP1.
- R2 maintains an EBGP session with ISP2.
- R1 and R2 exchange routing information through IBGP.
- An internal routing mechanism provides reachability between the edge routers and the rest of the enterprise.

This provides redundancy against both link and edge-router failures.

**Important:** Separate edge routers do not guarantee end-to-end redundancy. Internal routing, next-hop reachability, physical paths, and other shared infrastructure must also support failover.

---

## Outbound vs. Inbound Traffic Engineering

One of the most important BGP multihoming concepts is the distinction between **outbound** and **inbound traffic engineering**.

These terms describe the direction of user traffic relative to the enterprise AS.

They do not describe the direction of a BGP route map.

Consider:

```text
                    Internet
                       |
                +------+------+
                |             |
                |             |
              ISP1           ISP2
                |             |
                |             |
                R1            R2
                 \            /
                  \          /
                   Enterprise
                    AS 65001
```

### Outbound Traffic Engineering

Outbound traffic engineering determines which path the enterprise uses to send traffic toward external destinations.

For example:

```text
Enterprise -----> ISP1 -----> Internet
```

The enterprise controls its own BGP best-path selection.

Therefore, outbound traffic engineering is generally more predictable.

Common mechanisms include:

- Weight.
- LOCAL_PREF.
- Inbound route filtering.
- Selective acceptance of more-specific routes.
- BGP multipath.
- Policy-Based Routing (PBR), when appropriate.

**LOCAL_PREF is generally the preferred BGP attribute for controlling outbound traffic across an entire AS.**

### Inbound Traffic Engineering

Inbound traffic engineering attempts to influence how external networks send traffic toward the enterprise.

For example:

```text
Internet -----> ISP2 -----> Enterprise
```

The enterprise does not control the best-path selection algorithms or routing policies of external autonomous systems.

Instead, it influences their decisions by modifying the routing information advertised to them.

Common mechanisms include:

- AS_PATH prepending.
- MED.
- BGP communities.
- Selective prefix advertisement.
- More-specific route advertisements.
- Conditional advertisement.

**Inbound traffic engineering is inherently less deterministic because external ASes make their own routing decisions.**

---

## Traffic Engineering Mechanisms

The main BGP multihoming mechanisms are summarized below.

| Mechanism | Main Direction | Purpose |
|---|---|---|
| Weight | Outbound | Prefer a path on one local Cisco router |
| LOCAL_PREF | Outbound | Prefer an exit path throughout the local AS |
| Inbound route filtering | Outbound | Control which received paths are available for selection |
| AS_PATH prepending | Inbound | Make an advertised path appear less attractive |
| MED | Inbound | Suggest a preferred entry point, typically to the same neighboring AS |
| BGP communities | Inbound or outbound | Request or apply specific routing policies |
| Selective advertisement | Inbound | Control which external networks learn particular routes |
| More-specific advertisement | Inbound | Influence traffic using longest-prefix matching |
| Conditional advertisement | Inbound/failover | Advertise selected routes only when conditions are satisfied |
| BGP multipath | Outbound | Install multiple eligible BGP paths for forwarding |

These mechanisms are not mutually exclusive.

For example, an enterprise might use:

- LOCAL_PREF to prefer ISP1 for outbound traffic.
- AS_PATH prepending to make ISP2 less attractive for inbound traffic.
- Conditional advertisement to provide a backup advertisement when ISP1 fails.

---

## Receiving Routes from Multiple ISPs

Multihomed enterprises must decide how much routing information to receive from each ISP.

Three common approaches are:

1. Default routes only.
2. Default routes with selected external prefixes.
3. Full Internet routing tables.

### Default Routes Only

The enterprise receives `0.0.0.0/0` from each ISP.

```text
                  AS 65001

          ISP1               ISP2
            |                  |
            | 0.0.0.0/0        | 0.0.0.0/0
            |                  |
            +--------R1--------+
```

R1 receives two candidate default routes and selects a preferred path using BGP best-path selection.

For example:

| Default Route | LOCAL_PREF |
|---|---|
| Via ISP1 | 200 |
| Via ISP2 | 100 |

R1 prefers ISP1.

If the preferred BGP path is withdrawn and the backup remains valid, R1 can select ISP2.

Advantages:

- Small BGP routing table.
- Lower memory and control-plane requirements.
- Simple configuration and maintenance.

Disadvantages:

- Limited destination-specific traffic engineering.
- The router cannot make detailed routing decisions based on individual Internet prefixes.

**Important:** Receiving two default routes does not automatically result in load balancing. BGP normally selects one best path unless the relevant multipath requirements are satisfied.

### Default Routes with Selected Prefixes

The enterprise receives default routes plus selected more-specific prefixes.

For example:

```text
0.0.0.0/0        via ISP1
0.0.0.0/0        via ISP2
203.0.113.0/24   via ISP2
```

Traffic toward `203.0.113.0/24` uses ISP2 because the more-specific route wins during the routing-table lookup.

Other destinations use the selected default route.

This approach provides targeted traffic engineering without requiring full Internet routing tables.

**Important:** Longest-prefix matching in the forwarding table takes precedence over comparing BGP attributes between routes of different prefix lengths.

BGP best-path selection compares alternative paths to the same prefix.

### Full Internet Routing Tables

The enterprise receives full Internet routing information from its ISPs.

This provides more detailed control over outbound path selection.

For example, the enterprise can prefer ISP1 for one set of destinations and ISP2 for another set.

Advantages:

- Greater visibility into external routes.
- More granular path selection.
- More extensive traffic engineering opportunities.

Disadvantages:

- Higher memory requirements.
- More BGP control-plane processing.
- Larger routing tables.
- Greater operational complexity.

**Important:** Receiving full routes from multiple ISPs does not automatically load balance traffic between them.

BGP best-path selection and configured policies determine which routes are preferred.

---

## Outbound Traffic Engineering with LOCAL_PREF

Consider an enterprise connected to two ISPs.

```text
       AS 65002                   AS 65003
         ISP1                       ISP2
           |                          |
           |                          |
           R1 ----------------------- R2
                       IBGP

                    AS 65001
```

The enterprise wants to prefer ISP1 for outbound traffic.

R1 can assign a higher LOCAL_PREF to routes received from ISP1.

For example:

```text
R1(config)# route-map PREFER-ISP1 permit 10
R1(config-route-map)# set local-preference 200

R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 route-map PREFER-ISP1 in
```

Routes learned from ISP1 receive LOCAL_PREF 200.

Assuming ISP2's routes retain the default LOCAL_PREF of 100, routes through ISP1 are preferred when both ISPs advertise the same prefixes and the relevant paths are valid.

LOCAL_PREF is propagated within the enterprise AS through IBGP.

It is not advertised to ordinary EBGP neighbors.

### Weight vs. LOCAL_PREF

| Feature | Weight | LOCAL_PREF |
|---|---|---|
| Scope | Local router | Entire AS |
| Cisco-specific? | Yes | No |
| Advertised through IBGP? | No | Yes |
| Higher value preferred? | Yes | Yes |
| Best for | Local router-specific decisions | AS-wide outbound path preference |

For multihoming with multiple edge routers, LOCAL_PREF is usually more useful because it can provide consistent exit preferences throughout the AS.

See [Outbound Traffic Engineering](outbound.md) for detailed configuration.

---

## Inbound Traffic Engineering with AS_PATH Prepending

AS_PATH prepending adds additional occurrences of an ASN to the AS_PATH advertised to a neighbor.

This makes the route appear to have traversed a longer AS path.

Consider:

```text
                 Internet
                    |
             +------+------+
             |             |
            ISP1          ISP2
             |             |
             R1------------R2
                  AS 65001
```

The enterprise advertises `203.0.113.0/24` through both ISPs.

It wants ISP1 to be the preferred inbound path.

One option is to prepend AS 65001 when advertising the prefix to ISP2.

For example:

```text
R2(config)# route-map PREPEND permit 10
R2(config-route-map)# set as-path prepend 65001 65001 65001

R2(config)# router bgp 65001
R2(config-router)# address-family ipv4 unicast
R2(config-router-af)# neighbor 10.0.23.3 route-map PREPEND out
```

The advertised AS_PATH through ISP2 becomes longer.

For an external router that sees the two paths, the AS_PATHs might look like:

| Path | AS_PATH |
|---|---|
| Via ISP1 | `65002 65001` |
| Via ISP2 | `65003 65001 65001 65001 65001` |

If earlier BGP best-path criteria do not determine the result, the shorter AS_PATH through ISP1 is preferred.

However, **AS_PATH prepending does not guarantee that external networks will use ISP1**.

External networks may apply their own LOCAL_PREF, routing policies, or other path-selection criteria.

See [Inbound Traffic Engineering](inbound.md) for detailed configuration.

---

## Inbound Traffic Engineering with MED

The **Multi-Exit Discriminator (MED)** is a BGP attribute used to suggest a preferred entry point into an AS.

A lower MED is preferred.

MED is particularly useful when an enterprise has multiple connections to the **same neighboring AS**.

For example:

```text
                  AS 65002
                    ISP1
                  /      \
                 /        \
                R1--------R2
                   IBGP
                  AS 65001
```

The enterprise can advertise the same prefix through both connections with different MED values.

For example:

```text
Via R1: MED 50
Via R2: MED 200
```

ISP1 can prefer the connection through R1, assuming earlier path-selection criteria are equal.

### Configuration Example

On R1:

```text
R1(config)# route-map SET-MED permit 10
R1(config-route-map)# set metric 50
```

On R2:

```text
R2(config)# route-map SET-MED permit 10
R2(config-route-map)# set metric 200
```

Apply these route maps outbound toward the appropriate ISP neighbors.

### MED Limitations

By default, Cisco BGP normally compares MED values only for paths learned from the same neighboring AS.

MED is therefore less reliable for controlling inbound traffic across different ISPs.

Even when MED is compared, other BGP attributes and provider policies may take precedence.

MED is commonly preserved when an EBGP-learned route is advertised to an IBGP neighbor, but is not normally propagated unchanged across successive ordinary EBGP AS boundaries.

**Important:** Do not assume that setting a lower MED guarantees preferred inbound routing.

---

## Inbound Traffic Engineering with Communities

BGP communities allow customers to request specific routing policies from their providers.

For example, an ISP might define:

| Community | Provider Policy |
|---|---|
| `65002:100` | Set LOCAL_PREF 100 |
| `65002:200` | Set LOCAL_PREF 200 |
| `65002:300` | Prepend the provider ASN toward selected peers |

These values are illustrative. Each ISP defines its own supported community values.

An enterprise can tag advertisements with the provider's documented communities.

For example:

```text
R1(config)# route-map SET-COMMUNITY permit 10
R1(config-route-map)# set community 65002:200 additive

R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 route-map SET-COMMUNITY out
R1(config-router-af)# neighbor 10.0.12.2 send-community standard
```

The provider can then apply a routing policy based on the received community.

Communities can be more effective than AS_PATH prepending because they allow a customer to influence the provider's LOCAL_PREF or other internal policies.

However, their effectiveness depends entirely on the provider's documented behavior.

See [BGP Community Configuration and Filtering](../communities/configuration.md).

---

## Selective Prefix Advertisement

An enterprise can influence inbound traffic by advertising different prefixes to different providers.

Consider an enterprise that owns:

```text
203.0.112.0/23
```

The address block contains two /24 prefixes:

```text
203.0.112.0/24
203.0.113.0/24
```

The enterprise might advertise:

| Advertisement | ISP1 | ISP2 |
|---|---|---|
| `203.0.112.0/23` | Yes | Yes |
| `203.0.112.0/24` | Yes | No |
| `203.0.113.0/24` | No | Yes |

The more-specific /24 advertisements generally attract traffic toward the corresponding ISP because routers use longest-prefix matching.

This is different from AS_PATH prepending.

**Longest-prefix matching occurs before BGP best-path selection between routes to different prefixes.**

For example, an external router with both:

```text
203.0.112.0/23 via ISP2
203.0.112.0/24 via ISP1
```

forwards traffic toward `203.0.112.10` through ISP1.

The route through ISP1 is more specific.

### Limitations

Selective advertisement has several important constraints:

- The enterprise must be authorized to originate the prefixes.
- ISPs may filter overly specific prefixes.
- Internet routing policies commonly filter IPv4 prefixes longer than /24.
- Route Origin Authorization (ROA) records and other routing-security policies must permit the advertised prefix lengths.
- Advertising additional prefixes increases routing-table entries.
- External routing policies can still affect whether the advertisements propagate.

Therefore, prefix deaggregation is not always a practical traffic-engineering method.

---

## Conditional Advertisement for Failover

**BGP conditional advertisement** allows a router to advertise selected routes based on the presence or absence of other routes in its BGP table.

This can be useful in primary/backup multihoming designs.

For example:

```text
                    Internet
                       |
                  +----+----+
                  |         |
                 ISP1      ISP2
                  |         |
                  R1--------R2
                    AS 65001
```

Suppose the enterprise wants:

- ISP1 to be the preferred inbound connection.
- ISP2 to act as a backup.
- A selected prefix to be advertised to ISP2 only when a monitored route is absent.

Conditional advertisement can be configured using:

```text
neighbor <IP> advertise-map <ADV> {exist-map | non-exist-map} <CONDITION>
```

The components are:

| Component | Purpose |
|---|---|
| `advertise-map` | Identifies the routes to advertise conditionally |
| `exist-map` | Condition is satisfied when a matching BGP route exists |
| `non-exist-map` | Condition is satisfied when no matching BGP route exists |

The router periodically evaluates the configured condition.

When the condition changes, the conditional advertisements can be updated accordingly.

**Important:** Conditional advertisement evaluates BGP routing information. It does not inherently test physical link status or end-to-end reachability.

The monitored route must therefore be selected carefully.

Also, conditional advertisement is not instantaneous. The route condition must be evaluated and the resulting advertisements must propagate before external traffic shifts.

See [BGP Conditional Advertisement](../conditional-advertisement.md).

---

## Preventing Unintended Transit Routing

An enterprise connected to multiple ISPs should normally avoid unintentionally providing transit connectivity between them.

Consider:

```text
         ISP1                  ISP2
           |                     |
           |                     |
           +---------R1----------+
                  AS 65001
```

R1 learns Internet routes from both ISPs.

Without appropriate outbound controls, it might advertise routes learned from ISP1 to ISP2, or vice versa.

This could make the enterprise an unintended transit AS.

### Recommended Policy

Use outbound prefix lists or route maps to advertise only the enterprise's authorized prefixes.

For example:

```text
R1(config)# ip prefix-list CUSTOMER-PFX permit 203.0.113.0/24
```

Apply the prefix list outbound toward both ISPs:

```text
R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 prefix-list CUSTOMER-PFX out
R1(config-router-af)# neighbor 10.0.13.3 prefix-list CUSTOMER-PFX out
```

This limits advertisements to the authorized customer prefix, assuming the prefix is otherwise eligible for advertisement.

Other prefixes are denied by the prefix list's implicit deny.

Additional safeguards can include:

- Maximum-prefix limits.
- AS_PATH filters.
- Route-policy validation.
- Prefix and origin authorization controls.

**Important:** BGP's normal AS_PATH loop prevention does not prevent a multihomed enterprise from accidentally advertising routes between two different upstream providers.

Explicit outbound routing policies are essential.

---

## Failover and BGP Convergence

BGP multihoming provides alternative routing paths, but the presence of redundant connections does not guarantee immediate failover.

The convergence process typically involves:

1. Detecting a connection or path failure.
2. Invalidating or withdrawing affected routes.
3. Running BGP best-path selection.
4. Installing an alternative route in the routing table.
5. Updating the forwarding table.
6. Advertising routing changes to relevant neighbors.

### Failure Detection Mechanisms

Cisco IOS XE supports several mechanisms that can improve failure detection or path convergence.

| Mechanism | Purpose |
|---|---|
| BGP Hold Timer | Detect loss of a BGP peer when keepalives are no longer received |
| `fast-external-fallover` | Quickly terminate directly connected EBGP sessions when the connected interface goes down |
| BFD | Detect forwarding-path failures using rapid bidirectional liveliness checks |
| Next-Hop Tracking (NHT) | React to changes affecting BGP next-hop reachability |
| BGP route refresh / policy reevaluation | Reprocess routes after policy changes without unnecessarily resetting sessions |
| BGP PIC, on supported platforms | Reduce forwarding reconvergence time after certain path failures |

### Physical vs. Logical Failures

A directly connected interface going down is generally straightforward to detect.

However, an ISP connection can remain physically up while upstream connectivity is unavailable.

For example:

```text
Enterprise -------- ISP Edge -------- ISP Core -------- Internet
       Link Up                     Failure
```

The enterprise's local interface and EBGP session may remain established.

Therefore, a physical interface failure-detection mechanism alone may not detect every upstream failure.

BGP routing changes, BFD, next-hop reachability, and monitoring policies can provide additional information, depending on where the failure occurs.

**Important:** A default route received from an ISP indicates that the ISP is advertising reachability. It does not prove that every destination on the Internet is currently reachable through that ISP.

---

## Multihoming and Load Balancing

Multihoming does not necessarily mean load balancing.

An enterprise may have two active BGP sessions but use only one connection for a particular destination.

### Active/Standby

One ISP is preferred, while the other provides backup connectivity.

For example:

```text
ISP1: LOCAL_PREF 200
ISP2: LOCAL_PREF 100
```

For prefixes available through both ISPs, the ISP1 path is normally preferred, assuming earlier BGP preference criteria do not decide otherwise.

If the ISP1 path becomes unavailable, the ISP2 path can become best.

### Active/Active

Both ISPs carry traffic.

For example:

- ISP1 is preferred for some destination prefixes.
- ISP2 is preferred for other destination prefixes.

This can be implemented with prefix-specific LOCAL_PREF policies.

However, both links carrying traffic does not necessarily mean that BGP multipath is configured.

### BGP Multipath

BGP multipath allows multiple eligible BGP paths to be installed in the RIB.

For example:

```text
R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# maximum-paths 2
```

The paths must satisfy the applicable multipath eligibility requirements.

For EBGP paths from different neighboring ASes, supported IOS XE configurations can use:

```text
R1(config-router-af)# bgp bestpath as-path multipath-relax
```

This relaxes certain AS_PATH matching requirements for multipath.

It does not override all other multipath conditions.

**Important:** BGP multipath determines which paths can be installed. CEF determines how traffic is distributed across the installed forwarding paths.

See [BGP Load Balancing](../../path-selection/load-balancing.md).

---

## Asymmetric Routing

Multihomed networks frequently experience **asymmetric routing**.

Asymmetric routing occurs when traffic takes different paths in the forward and return directions.

For example:

```text
Outbound: Enterprise -> ISP1 -> Internet

Inbound:  Internet -> ISP2 -> Enterprise
```

This can occur because:

- The enterprise controls its outbound routing policy.
- External networks independently control their routing toward the enterprise.

Asymmetric routing is not inherently a problem for IP forwarding.

However, it can cause issues with:

- Stateful firewalls.
- NAT.
- Path-dependent security policies.
- Traffic monitoring.
- Troubleshooting and performance analysis.

For example, a stateful firewall may observe the outbound half of a connection through ISP1 but not the return traffic arriving through ISP2.

The firewall might then reject the return traffic because it has no matching session state.

**Important:** Traffic engineering must account for the placement and state synchronization of firewalls, NAT devices, and other stateful services.

---

## Multihoming Design Considerations

When designing BGP multihoming, consider the following questions.

### 1. What Level of Redundancy Is Required?

Possible designs include:

- Two links to one ISP.
- Two ISPs connected to one edge router.
- Two ISPs connected to separate edge routers.
- Multiple providers and multiple geographic locations.

Each design protects against different failure scenarios.

### 2. What Routing Information Should Be Received?

Choose between:

- Default routes only.
- Default routes plus selected prefixes.
- Full Internet routing tables.

The decision affects memory requirements, routing complexity, and traffic-engineering flexibility.

### 3. How Should Outbound Traffic Be Controlled?

Typical options include:

- LOCAL_PREF.
- Weight.
- Prefix-based policies.
- BGP multipath.

LOCAL_PREF is usually the preferred mechanism for AS-wide outbound path preference.

### 4. How Should Inbound Traffic Be Controlled?

Typical options include:

- AS_PATH prepending.
- MED.
- Provider communities.
- Selective prefix advertisement.
- Conditional advertisement.

No mechanism provides complete control over routing decisions made by independent external ASes.

### 5. How Will Failures Be Detected?

Consider:

- Physical link failures.
- BGP session failures.
- Next-hop failures.
- Upstream connectivity failures.
- Internal network failures.

Different failure types may require different detection mechanisms.

### 6. How Will Unintended Transit Be Prevented?

Outbound filtering should restrict advertisements to authorized enterprise prefixes.

Receiving Internet routes from multiple ISPs does not imply that the enterprise should redistribute those routes between providers.

---

## Verification

Useful commands for verifying a multihomed BGP deployment include:

```text
R1# show bgp ipv4 unicast summary
R1# show bgp ipv4 unicast
R1# show bgp ipv4 unicast 0.0.0.0/0
R1# show ip route bgp
R1# show ip route 0.0.0.0
R1# show bgp ipv4 unicast neighbors 10.0.12.2 routes
R1# show bgp ipv4 unicast neighbors 10.0.12.2 advertised-routes
R1# show route-map
R1# show ip prefix-list
```

### Verify BGP Neighbor Status

```text
R1# show bgp ipv4 unicast summary
```

Confirm that the expected EBGP and IBGP sessions are established.

### Verify Best-Path Selection

```text
R1# show bgp ipv4 unicast 0.0.0.0/0
```

If default routes are received from multiple providers, examine:

- Weight.
- LOCAL_PREF.
- AS_PATH.
- MED.
- NEXT_HOP.
- Best-path status.

### Verify Routing-Table Installation

```text
R1# show ip route 0.0.0.0
```

Confirm that the expected default route is installed.

A BGP route being present in the BGP table does not guarantee that it is installed in the global RIB.

### Verify Advertised Routes

```text
R1# show bgp ipv4 unicast neighbors 10.0.12.2 advertised-routes
```

Confirm that only authorized prefixes are advertised to the ISP.

### Verify Forwarding

```text
R1# show ip cef 203.0.113.10 detail
R1# traceroute 203.0.113.10
```

The CEF table identifies the forwarding path selected for the destination.

Traceroute can help verify the path taken by test traffic, although intermediate devices may not respond or may process traceroute traffic differently.

---

## Key Points

- BGP multihoming connects an AS to multiple external connections for redundancy and traffic engineering.
- Multihoming can use one or multiple ISPs and one or multiple edge routers.
- **Outbound traffic engineering** controls how traffic leaves the enterprise.
- **Inbound traffic engineering** attempts to influence how external networks reach the enterprise.
- LOCAL_PREF is generally preferred for AS-wide outbound path selection.
- Weight influences path selection on only one Cisco router.
- AS_PATH prepending influences inbound traffic by increasing the advertised AS_PATH length.
- MED is most useful when advertising routes over multiple links to the same neighboring AS.
- BGP communities can request provider-specific routing policies.
- More-specific prefixes influence forwarding through longest-prefix matching.
- Conditional advertisement can control backup advertisements based on BGP route existence.
- Default routes provide simple redundancy but limited destination-specific traffic engineering.
- Full routing tables provide greater path-selection flexibility at the cost of additional resources.
- Redundant BGP sessions do not automatically provide load balancing.
- BGP multipath requires eligible paths and appropriate configuration.
- Physical link failures and upstream reachability failures are not equivalent.
- Multihoming can produce asymmetric routing.
- Outbound filtering is essential to prevent unintended transit routing between ISPs.

---

## Topics Covered

1. [Outbound Traffic Engineering](outbound.md): Weight, LOCAL_PREF, route filtering, path preference, active/standby designs, and load sharing.
2. [Inbound Traffic Engineering](inbound.md): AS_PATH prepending, MED, provider communities, selective advertisement, and inbound traffic distribution.

Related topics:

- [BGP Conditional Advertisement](../conditional-advertisement.md)
- [BGP Community Configuration and Filtering](../communities/configuration.md)
- [BGP Load Balancing](../../path-selection/load-balancing.md)
- [BGP Best Path Selection Configuration](../../path-selection/best-path/configuration.md)
- [BGP Fast Convergence and BFD](../../other-features/fast-convergence.md)

---

## References

- [Cisco: Configure BGP with Two Different Service Providers](https://www.cisco.com/c/en/us/support/docs/ip/border-gateway-protocol-bgp/217964-configure-sample-for-bgp-with-two-differ.html)
- [Cisco: Connecting to a Service Provider Using External BGP](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-routing/b-ip-routing/m_irg-external-sp-0.html)
- [Cisco: Configure an Upstream Provider Network with BGP Community Values](https://www.cisco.com/c/en/us/support/docs/ip/border-gateway-protocol-bgp/28784-bgp-community.html)
- [Cisco: Configure and Verify BGP Conditional Advertisement](https://www.cisco.com/c/en/us/support/docs/ip/border-gateway-protocol-bgp/16137-cond-adv.html)
- [Cisco: BGP Frequently Asked Questions](https://www.cisco.com/c/en/us/support/docs/ip/border-gateway-protocol-bgp/5816-bgpfaq-5816.html)
