# 1.5.f Other BGP Features

Cisco IOS XE supports several additional **BGP features** that improve routing policy management, operational stability, scalability, and convergence.

These features address different aspects of BGP operation:

- **Soft reconfiguration:** Reevaluate received BGP routes using updated inbound policies without resetting the BGP session.
- **Route refresh:** Request that a neighbor readvertise routing information without resetting the BGP session.
- **Session clearing:** Reset BGP sessions or reprocess routing information when configuration changes require it.
- **Maximum prefix:** Limit the number of prefixes accepted from a BGP neighbor to protect against excessive routing information.
- **Fast convergence and BFD:** Detect failures and react to changes more quickly.
- **Additional Paths (ADD-PATH):** Allow multiple paths for the same prefix to be advertised over a BGP session.
- **Route flap dampening:** Suppress unstable routes that repeatedly become available and unavailable.

These mechanisms are largely independent of the basic BGP best-path selection algorithm, although some influence the paths available for selection.

---

## BGP Soft Reconfiguration

**BGP soft reconfiguration** allows a router to apply updated routing policies without terminating an established BGP session.

Normally, BGP evaluates received routes against inbound policies before accepting them into the local BGP table.

If an inbound policy changes, the router may need to reevaluate routes that were previously received, including routes rejected by the original policy.

### Inbound Soft Reconfiguration

With traditional inbound soft reconfiguration, the router stores an unmodified copy of the routing information received from a neighbor.

For example:

```text
R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 soft-reconfiguration inbound
```

The router maintains the original received routes separately from the routes processed by inbound policy.

When the inbound policy changes, the stored information can be reprocessed without requesting the routes again from the neighbor.

```text
R1# clear ip bgp 10.0.12.2 soft in
```

This applies the updated inbound policy without terminating the BGP session.

The disadvantage is **increased memory consumption**, because the router must maintain additional routing information.

Modern BGP deployments generally prefer Route Refresh when both neighbors support it.

See [BGP Soft Reconfiguration](soft-reconfiguration.md) for configuration and operation.

---

## BGP Route Refresh

**BGP Route Refresh** allows a router to request that a BGP neighbor readvertise routing information for a particular address family.

Like inbound soft reconfiguration, Route Refresh allows updated inbound policies to be applied without resetting the BGP session.

However, the mechanisms are different.

### Route Refresh Operation

Consider two EBGP neighbors:

```text
        AS 65001                     AS 65002

           R1 ------------------------- R2
                  EBGP Session

           R1 receives routes from R2
```

Suppose R1 modifies its inbound route map.

To apply the updated policy:

1. R1 initiates an inbound soft reset.
2. R1 sends a ROUTE-REFRESH message to R2.
3. R2 readvertises the appropriate routing information.
4. R1 processes the received routes using its updated inbound policy.
5. The BGP session remains Established throughout the process.

For example:

```text
R1# clear ip bgp 10.0.12.2 soft in
```

If Route Refresh is supported and traditional inbound soft reconfiguration is not being used, this command dynamically requests the routing information from R2.

### Soft Reconfiguration vs. Route Refresh

| Feature | Traditional Inbound Soft Reconfiguration | Route Refresh |
|---|---|---|
| Requires storing unmodified received routes | Yes | No |
| Requires neighbor support | No special capability required | Route Refresh capability |
| Requires additional local memory | Yes | No additional stored copy required |
| Requires route readvertisement from neighbor | No | Yes |
| Resets the BGP session | No | No |
| Typical modern preference | Used for specific requirements | Generally preferred |

**Important:** Route Refresh and inbound soft reconfiguration accomplish a similar operational goal, but they use different mechanisms.

See [BGP Route Refresh](route-refresh.md) for details.

---

## BGP Session Reset and Clearing

Cisco IOS XE provides several methods to clear BGP routing information or reset BGP sessions.

The appropriate method depends on the type of configuration change and whether the BGP session must be reestablished.

### Hard Reset

A **hard reset** terminates the BGP session and causes the neighbors to establish a new session.

For example:

```text
R1# clear ip bgp 10.0.12.2
```

This causes:

1. The existing BGP session to be terminated.
2. Routes learned through that session to be withdrawn from the local BGP process.
3. The neighbors to establish a new TCP connection and BGP session.
4. Routing information to be exchanged again.

A hard reset can temporarily disrupt routing and should be avoided when a soft reset is sufficient.

### Soft Reset

A **soft reset** reprocesses routing information without terminating the BGP session.

Two common operations are:

```text
R1# clear ip bgp 10.0.12.2 soft in
```

Reapply inbound routing policies using stored routes or Route Refresh, as appropriate.

```text
R1# clear ip bgp 10.0.12.2 soft out
```

Reevaluate and readvertise eligible routes using the current outbound routing policies.

### Comparison

| Operation | Session Reset? | Purpose |
|---|---|---|
| Hard reset | Yes | Reestablish BGP peering |
| Inbound soft reset | No | Reapply inbound policies |
| Outbound soft reset | No | Reapply outbound policies |
| Route Refresh | No | Obtain routing information again from a neighbor |

**CCIE consideration:** Changing a route map does not normally require a hard reset. Use the appropriate soft reset when updated policy must be applied to existing routing information.

See [BGP Session Reset and Clearing](session-reset.md) for detailed clearing commands and behavior.

---

## BGP Maximum Prefix

The **BGP Maximum Prefix** feature protects a router from receiving an excessive number of prefixes from a BGP neighbor.

This is especially useful when establishing EBGP sessions with external networks, where the number of received prefixes might unexpectedly increase.

For example, a neighbor might accidentally advertise a full Internet routing table instead of a small set of customer routes.

Without appropriate protection, this can consume significant memory and CPU resources.

### Configuration

Configure a maximum of 1,000 prefixes, with a warning threshold of 80%:

```text
R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 maximum-prefix 1000 80
```

The router generates a warning when the prefix count reaches the configured threshold.

By default, exceeding the maximum prefix limit causes the BGP session to be terminated.

### Maximum Prefix Options

Cisco IOS XE supports additional options that modify this behavior.

| Option | Purpose |
|---|---|
| `maximum-prefix` | Configure a maximum prefix count |
| Warning threshold | Generate a warning before the maximum is exceeded |
| `warning-only` | Generate warnings without terminating the session |
| `restart` | Attempt to reestablish the session after a configured interval |
| `discard-extra` | Discard additional prefixes instead of resetting the session, where supported |

For example:

```text
R1(config-router-af)# neighbor 10.0.12.2 maximum-prefix 1000 80 warning-only
```

This generates warnings without terminating the session when the limit is exceeded.

**Important:** `warning-only` provides monitoring but does not enforce a hard limit on the number of received prefixes.

See [BGP Maximum Prefix](maximum-prefix.md) for configuration, prefix counting, and session recovery.

---

## BGP Fast Convergence and BFD

**BGP convergence** is the process of detecting routing changes, selecting alternative paths, and updating forwarding information.

BGP convergence time can be affected by several factors:

- How quickly a failed neighbor or path is detected.
- How quickly unavailable BGP paths are invalidated.
- How quickly an alternative path is selected.
- How quickly the routing and forwarding tables are updated.

Cisco IOS XE supports several mechanisms that improve convergence.

### BGP Hold Timer

BGP neighbors exchange KEEPALIVE messages to maintain their sessions.

If a router does not receive a KEEPALIVE or UPDATE message before the negotiated Hold Timer expires, it considers the BGP session failed.

The default Cisco BGP timers are generally:

| Timer | Default |
|---|---|
| Keepalive | 60 seconds |
| Hold | 180 seconds |

Waiting for Hold Timer expiration can delay failure detection.

### BGP Fast External Fallover

**BGP Fast External Fallover** allows a directly connected EBGP session to be reset when the interface used to reach the neighbor goes down.

It is enabled by default on supported Cisco IOS XE platforms.

```text
R1(config)# router bgp 65001
R1(config-router)# bgp fast-external-fallover
```

This avoids waiting for the BGP Hold Timer when the relevant directly connected interface fails.

However, it does not necessarily detect failures beyond a directly connected interface.

### Bidirectional Forwarding Detection

**Bidirectional Forwarding Detection (BFD)** provides rapid forwarding-path failure detection independently of BGP's KEEPALIVE and Hold Timer mechanisms.

BFD peers exchange control packets at configurable intervals.

When BFD detects a failure, it can notify BGP so the affected BGP session can be brought down quickly.

For example:

```text
R1(config)# interface GigabitEthernet0/0
R1(config-if)# bfd interval 100 min_rx 100 multiplier 3
```

The example configures BFD timing parameters on the interface.

BFD must also be associated with the BGP neighbor using the appropriate configuration, and both devices must support compatible BFD operation.

### BGP Next-Hop Tracking

**BGP Next-Hop Tracking (NHT)** allows BGP to respond to changes in next-hop reachability without waiting for the periodic BGP scanner.

Unlike BFD, which detects forwarding-path failures, NHT monitors changes affecting the resolution of BGP next-hop addresses.

These mechanisms can work together to improve convergence.

See [BGP Fast Convergence and BFD](fast-convergence.md) for configuration and verification.

Next-Hop Tracking is also covered in [BGP Next-Hop Handling](../peering/next-hop.md).

---

## BGP Additional Paths (ADD-PATH)

By default, BGP normally advertises only its selected best path for a destination prefix to a particular neighbor.

**BGP Additional Paths (ADD-PATH)** allows multiple paths for the same destination prefix to be advertised over a single BGP session.

This feature is especially useful in networks that use BGP route reflectors.

### Route Reflector Path Hiding

Consider a route reflector receiving two paths to the same prefix:

```text
                 R2
                  |
                  | Path A
                  |
                 RR
                  |
                  | Path B
                  |
                 R3

             RR Client
                 R4
```

Suppose the RR learns Path A from R2 and Path B from R3.

By default:

1. The RR evaluates both paths.
2. The RR selects one best path.
3. The RR normally advertises only that best path to R4.

R4 may not learn about the alternative path until the RR selects and advertises it.

This is one example of **BGP path hiding**.

### ADD-PATH Operation

With ADD-PATH, the RR can advertise multiple eligible paths for the same prefix to R4.

For example:

```text
192.0.2.0/24 via Path A
192.0.2.0/24 via Path B
```

ADD-PATH introduces a **Path Identifier** that allows multiple advertisements for the same Network Layer Reachability Information (NLRI) to be distinguished.

The receiving router can maintain multiple paths for the prefix and perform its own path selection.

### Important Distinction

ADD-PATH and BGP multipath are different features.

| Feature | ADD-PATH | BGP Multipath |
|---|---|---|
| Primary purpose | Advertise multiple paths | Install multiple eligible paths |
| Main function | BGP path advertisement | Routing and forwarding |
| Requires multiple paths to be advertised | Yes | Not necessarily |
| Automatically enables ECMP | No | Can enable ECMP |
| Common use | Route reflector path diversity | Load sharing |

ADD-PATH does not automatically cause the receiving router to install every advertised path in the routing table.

The receiving router still performs BGP best-path selection and applies its normal RIB installation rules.

See [BGP Additional Paths (ADD-PATH)](add-path.md) for configuration, capability negotiation, and path selection.

---

## BGP Route Flap Dampening

**BGP route flap dampening** reduces the impact of unstable routes that repeatedly become reachable and unreachable.

A route that frequently changes state can generate repeated BGP UPDATE messages and cause unnecessary routing recalculations.

Route flap dampening assigns penalties to route instability and can temporarily suppress routes whose accumulated penalties exceed a configured threshold.

### Route Dampening Operation

Route flap dampening uses several parameters:

| Parameter | Purpose |
|---|---|
| Penalty | Accumulates when a route flaps |
| Suppress limit | Penalty threshold at which the route is suppressed |
| Reuse limit | Penalty threshold below which a suppressed route can be reused |
| Half-life | Time required for the accumulated penalty to decay by half |
| Maximum suppress time | Maximum duration for which a route can remain suppressed |

For example:

1. A route is withdrawn.
2. BGP increases the route's accumulated penalty.
3. The route becomes available again.
4. If its penalty remains below the suppress limit, it can be used normally.
5. Further flaps increase the penalty.
6. If the penalty exceeds the suppress limit, BGP suppresses the route.
7. The penalty gradually decays over time.
8. When the penalty falls below the reuse limit, a currently available route can be reused.

### Configuration

Enable BGP route flap dampening using default parameters:

```text
R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# bgp dampening
```

The feature also supports configurable half-life, reuse, suppress, and maximum suppress-time parameters.

**Important:** Route flap dampening is not universally recommended. Aggressive dampening can suppress otherwise usable routes for an extended period, potentially making convergence worse.

It should be configured carefully, based on network requirements and routing stability.

See [BGP Route Flap Dampening](route-flap-dampening.md) for configuration and penalty calculations.

---

## Comparing BGP Operational Features

The following table summarizes the primary purposes of the features covered in this section.

| Feature | Primary Purpose | Main Benefit |
|---|---|---|
| Soft Reconfiguration | Reapply inbound policies | Avoid session resets |
| Route Refresh | Request routes again | Reapply inbound policy without storing all received routes |
| Session Clearing | Reset sessions or routing information | Apply changes and troubleshoot |
| Maximum Prefix | Control excessive prefix reception | Protect router resources |
| Fast Convergence and BFD | Detect and react to failures | Reduce convergence time |
| ADD-PATH | Advertise multiple paths | Improve path visibility |
| Route Flap Dampening | Suppress unstable routes | Reduce repeated routing updates |

These features address different problems and can be combined when appropriate.

For example, an enterprise router might use:

- **Maximum Prefix** to protect against unexpected route advertisements from an ISP.
- **Route Refresh** to apply updated inbound routing policies.
- **BFD** to detect forwarding-path failures quickly.
- **ADD-PATH** within a route reflector deployment to improve path visibility.

---

## Verification and Troubleshooting

Several commands are useful when working with these features.

### BGP Neighbor Status and Capabilities

```text
R1# show ip bgp summary
R1# show ip bgp neighbors
```

Verify:

- Neighbor states.
- Received prefix counts.
- Negotiated capabilities, including Route Refresh and ADD-PATH where supported.
- Keepalive and Hold Timer values.
- Session reset information.

### Received Routes

```text
R1# show ip bgp neighbors 10.0.12.2 routes
R1# show ip bgp neighbors 10.0.12.2 received-routes
```

The `routes` command displays accepted routes from the neighbor.

The `received-routes` command is useful when traditional inbound soft reconfiguration is configured and the original received routing information has been stored.

### Advertised Routes

```text
R1# show ip bgp neighbors 10.0.12.2 advertised-routes
```

Use this command to verify which routes are advertised to a BGP neighbor.

### BFD

```text
R1# show bfd neighbors
R1# show bfd neighbors details
```

These commands display BFD session status and negotiated timing information.

### Route Flap Dampening

```text
R1# show ip bgp dampening parameters
R1# show ip bgp dampening flap-statistics
R1# show ip bgp dampening dampened-paths
```

These commands help verify dampening parameters, route instability, and suppressed paths.

---

## Topics Covered

This section covers the following BGP operational features:

1. [BGP Soft Reconfiguration](soft-reconfiguration.md)
   - Traditional inbound soft reconfiguration.
   - Stored received routing information.
   - Reapplying inbound routing policies.

2. [BGP Route Refresh](route-refresh.md)
   - Route Refresh capability.
   - Dynamic inbound soft resets.
   - Differences from traditional soft reconfiguration.

3. [BGP Session Reset and Clearing](session-reset.md)
   - Hard resets.
   - Inbound and outbound soft resets.
   - BGP clearing commands and operational impact.

4. [BGP Maximum Prefix](maximum-prefix.md)
   - Maximum prefix limits and warning thresholds.
   - Session termination and automatic restart.
   - Warning-only operation and prefix-limit enforcement.

5. [BGP Fast Convergence and BFD](fast-convergence.md)
   - BGP failure detection.
   - Fast external fallover.
   - BFD operation and configuration.
   - BGP convergence optimization.

6. [BGP Additional Paths (ADD-PATH)](add-path.md)
   - ADD-PATH capability negotiation.
   - Multiple path advertisements.
   - Route reflector path diversity.
   - Differences from BGP multipath.

7. [BGP Route Flap Dampening](route-flap-dampening.md)
   - Route flap penalties.
   - Half-life, suppress, and reuse thresholds.
   - Dampening configuration and verification.

---

## References

- [Cisco: CCIE Enterprise Infrastructure v1.1 Practical Exam Topics](https://learningnetwork.cisco.com/s/ccie-enterpr-infrastructure-exam-topics)
- [Cisco: BGP 4 Soft Configuration](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-routing/b-ip-routing/m_irg-soft-config.html)
- [Cisco: Configuring a Basic BGP Network](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-routing/b-ip-routing/m_irg-basic-net-0.html)
- [Cisco: BGP Maximum-Prefix on IOS XE](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-routing/b-ip-routing/m_irg-max-prefix.html)
- [Cisco: BGP Additional Paths](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-routing/b-ip-routing/m_irg-additional-paths.html)
- [Cisco: Configuring Internal BGP Features](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-routing/b-ip-routing/m_irg-int-features-0.html)
