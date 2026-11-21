# BGP Route Flap Dampening

**BGP Route Flap Dampening (RFD)** is a mechanism that reduces the propagation of unstable BGP routes.

A **route flap** occurs when a route repeatedly changes state, such as alternating between being advertised and withdrawn.

Frequent route flapping can cause:

- Excessive BGP UPDATE messages.
- Repeated best-path calculations.
- Routing table changes.
- Increased CPU utilization.
- Routing instability across multiple autonomous systems.

Route flap dampening assigns a numerical **penalty** to an unstable route.

As the route continues to flap, its penalty increases.

If the accumulated penalty exceeds a configured threshold, the route is **suppressed**, meaning it is temporarily excluded from BGP best-path selection and advertisement.

The penalty gradually decreases over time, allowing the route to become usable again once it has stabilized.

BGP Route Flap Dampening is defined in **RFC 2439**.

**Important:** BGP Route Flap Dampening is disabled by default on Cisco IOS XE.

---

## Route Flap Dampening Operation

Consider three autonomous systems:

```text
         AS 100            AS 200            AS 300

           R1 -------------- R2 -------------- R3

       192.0.2.0/24
```

R1 advertises `192.0.2.0/24` to R2.

R2 advertises the route to R3.

Suppose the route on R1 repeatedly becomes available and unavailable.

### Without Route Flap Dampening

Each time R1 withdraws the route, R2 propagates the withdrawal to R3.

When R1 readvertises the route, R2 propagates the new advertisement.

```text
R1                       R2                       R3
 |                        |                        |
 |----- Advertisement --->|----- Advertisement --->|
 |                        |                        |
 |----- Withdrawal ------>|----- Withdrawal ------>|
 |                        |                        |
 |----- Advertisement --->|----- Advertisement --->|
 |                        |                        |
 |----- Withdrawal ------>|----- Withdrawal ------>|
 |                        |                        |
```

Repeated route changes generate BGP UPDATE messages throughout the network.

### With Route Flap Dampening

Enable route flap dampening on R2.

R2 tracks the instability of the route using a penalty value.

Initially, R2 continues processing and propagating the route changes.

However, once the penalty exceeds the suppress threshold, R2 suppresses the unstable path.

```text
R1                       R2                       R3
 |                        |                        |
 |----- Advertisement --->|----- Advertisement --->|
 |                        |                        |
 |----- Withdrawal ------>|----- Withdrawal ------>|
 |                        |                        |
 |----- Advertisement --->|----- Advertisement --->|
 |                        |                        |
 |----- Withdrawal ------>|----- Withdrawal ------>|
 |                        |                        |
 |----- Advertisement --->|                        |
 |                        |                        |
 |                  Route Suppressed               |
 |                        |                        |
```

While the path is suppressed, R2 does not select it for forwarding or advertise it to R3.

If R2 previously advertised the path, it withdraws that advertisement as necessary.

The penalty continues decreasing over time.

Once the penalty falls below the reuse threshold, the path can become eligible again.

**Route flap dampening suppresses unstable BGP paths, not the BGP neighbor session itself.**

---

## Route Flap Dampening Parameters

Cisco IOS XE uses four main parameters to control route flap dampening.

| Parameter | Default | Purpose |
|---|---|---|
| Half-life | 15 minutes | Time required for a penalty to decrease by half |
| Reuse | 750 | Threshold below which a suppressed route becomes eligible again |
| Suppress | 2,000 | Threshold above which a route is suppressed |
| Maximum suppress time | 60 minutes | Maximum time a route can remain suppressed |

These parameters control how quickly routes are suppressed and how long suppression lasts.

### Half-Life

The **half-life** specifies how long it takes for a route's penalty to decrease by half.

The default is 15 minutes.

For example, suppose a route has a penalty of 4,000.

If no further flaps occur:

| Time | Penalty |
|---|---:|
| Initially | 4,000 |
| After 15 minutes | 2,000 |
| After 30 minutes | 1,000 |
| After 45 minutes | 500 |
| After 60 minutes | 250 |

The penalty decreases exponentially.

The calculation is:

```text
Penalty(t) = Initial Penalty * 2^(-t / Half-Life)
```

Where:

- `Penalty(t)` is the remaining penalty after time `t`.
- `Initial Penalty` is the penalty at the beginning of the interval.
- `Half-Life` is the configured half-life in minutes.

For example:

```text
Initial Penalty = 4000
Half-Life       = 15 minutes
Elapsed Time    = 30 minutes

Penalty = 4000 * 2^(-30/15)
        = 4000 * 0.25
        = 1000
```

The penalty decreases continuously in principle, although IOS XE processes decay using periodic updates.

Cisco documents penalty decay processing at approximately 5-second intervals.

### Reuse Threshold

The **reuse threshold** determines when a previously suppressed route can become eligible for use again.

The default is **750**.

Once the route's penalty decreases below this threshold, it is unsuppressed.

For example:

```text
Reuse Threshold = 750
```

| Penalty | Result |
|---|---|
| 2001 | Route suppressed |
| 1,000 | Remains suppressed |
| 800 | Remains suppressed |
| 700 | Eligible for reuse |

A route must also be currently advertised and otherwise eligible before it can be selected as a BGP path.

**Important:** A suppressed route is not necessarily reenabled as soon as its penalty drops below the suppress threshold. It must fall below the lower reuse threshold.

### Suppress Threshold

The **suppress threshold** determines when a route becomes dampened.

The default is **2,000**.

When the accumulated penalty exceeds this threshold, the route is suppressed.

For example:

```text
Suppress Threshold = 2000
```

| Penalty | Result |
|---|---|
| 1,000 | Not suppressed |
| 1,500 | Not suppressed |
| 2,000 | At the suppress threshold |
| 2,500 | Suppressed |
| 3,000 | Suppressed |

Cisco's command reference describes suppression as occurring when the penalty **exceeds** the suppress threshold.

### Maximum Suppress Time

The **maximum suppress time** limits how long a route can remain suppressed.

The default is four times the half-life:

```text
Half-Life = 15 minutes

Maximum Suppress Time = 15 * 4
                      = 60 minutes
```

Without this limit, a route accumulating a very large penalty could remain suppressed for an excessive amount of time.

Cisco IOS XE uses the maximum suppress time when calculating the maximum allowable accumulated penalty.

The calculation is:

```text
Maximum Penalty = Reuse * 2^(Max Suppress Time / Half-Life)
```

With the default values:

```text
Maximum Penalty = 750 * 2^(60/15)
                = 750 * 16
                = 12000
```

This means the accumulated penalty is capped at 12,000 under these parameters.

The cap prevents the penalty from growing indefinitely.

**The maximum suppress time is an upper bound, not the minimum amount of time a route must remain suppressed.**

A suppressed route can become usable earlier when its penalty falls below the reuse threshold.

---

## Penalty Accumulation

Cisco IOS XE assigns penalties when BGP routes exhibit instability.

Two important events are:

| Event | Penalty Increase |
|---|---:|
| Route withdrawal | 1,000 |
| Route attribute change | 500 |

Penalties are cumulative.

However, they also decay over time.

### Route Withdrawal

Suppose R2 receives a BGP route from R1.

R1 withdraws the route.

R2 assigns a penalty of 1,000.

If R1 subsequently readvertises the route and then withdraws it again, another penalty is added.

Assuming the events occur rapidly enough that penalty decay is negligible:

| Event | Approximate Penalty |
|---|---:|
| First withdrawal | 1,000 |
| Second withdrawal | 2,000 |
| Third withdrawal | 3,000 |

With the default suppress threshold of 2,000, the third withdrawal causes the penalty to exceed the threshold.

The path becomes suppressed.

**Important:** This table assumes negligible time between events. Actual penalties depend on the amount of decay between flaps.

### Route Attribute Changes

A BGP UPDATE that changes a route's attributes can also contribute to its penalty.

Cisco documents a penalty increase of 500 for an attribute change.

For example:

| Event | Approximate Penalty |
|---|---:|
| First attribute change | 500 |
| Second attribute change | 1,000 |
| Third attribute change | 1,500 |
| Fourth attribute change | 2,000 |
| Fifth attribute change | 2,500 |

Again, this assumes negligible penalty decay.

The fifth attribute change causes the accumulated penalty to exceed the default suppress threshold.

### Penalty Decay

The penalty does not remain constant between route changes.

For example:

1. A withdrawal increases the penalty to 1,000.
2. After 15 minutes without additional flaps, it decreases to approximately 500.
3. Another withdrawal increases it to approximately 1,500.
4. After another 15 minutes, it decreases to approximately 750.

Therefore, the number of route flaps alone does not determine whether suppression occurs.

**Both flap frequency and accumulated penalty determine whether a route is suppressed.**

---

## Suppression and Reuse

The suppress and reuse thresholds create **hysteresis**.

Hysteresis means that different thresholds are used to enter and exit a state.

For BGP dampening:

- A route enters the suppressed state when its penalty exceeds the **suppress threshold**.
- It remains suppressed while its penalty decreases.
- It exits the suppressed state when its penalty falls below the **reuse threshold**.

With the defaults:

```text
Suppress Threshold = 2000
Reuse Threshold    = 750
```

Consider a route whose penalty rises to 3,000.

```text
Penalty
  ^
  |
3000 |           ^ 
     |          / \
2500 |         /   \
     |        /     \
2000 |-------X-------\---------------- Suppress
     |      /         \
1500 |     /           \
     |    /             \
1000 |   /               \
     |  /                 \
 750 |---------------------X---------- Reuse
     |                      \
 500 |                       \
     |
     +--------------------------------> Time
```

The route does not become usable again merely because the penalty decreases from 3,000 to 1,999.

It must fall below 750.

This prevents an unstable route from being repeatedly suppressed and unsuppressed near a single threshold.

### Reuse Calculation Example

Suppose:

```text
Initial Penalty = 3000
Half-Life       = 15 minutes
Reuse Threshold = 750
```

The penalty decays as follows:

| Time | Penalty | State |
|---|---:|---|
| 0 minutes | 3,000 | Suppressed |
| 15 minutes | 1,500 | Suppressed |
| 30 minutes | 750 | At reuse threshold |
| 45 minutes | 375 | Eligible for reuse |

The mathematical penalty reaches 750 after 30 minutes.

It falls below 750 immediately afterward.

IOS XE performs periodic reuse checks, so the actual unsuppression event may occur slightly later than the mathematical crossing time.

---

## History and Dampened States

Cisco IOS XE uses specific BGP status codes to identify routes affected by dampening.

| Status Code | Meaning |
|---|---|
| `h` | History |
| `d` | Dampened |

These codes can be displayed by:

```text
R1# show ip bgp
```

### History State (`h`)

A route enters the **history state** when it has been withdrawn but BGP retains historical information about it for dampening purposes.

The router no longer has a currently advertised path through that neighbor.

However, it retains information such as:

- The route prefix.
- The accumulated penalty.
- Flap statistics.

This allows subsequent flaps to accumulate penalties rather than starting from zero.

**A history entry represents a withdrawn route retained for dampening accounting.**

### Dampened State (`d`)

A route is **dampened** when its accumulated penalty has exceeded the suppress threshold.

If the route is subsequently advertised while still suppressed, BGP does not consider that path eligible for best-path selection.

The suppressed path is not installed in the IP routing table and is not advertised onward.

It remains suppressed until it meets the reuse conditions.

### History vs. Dampened

| State | Meaning |
|---|---|
| History (`h`) | Route is withdrawn, but historical penalty information is retained |
| Dampened (`d`) | Route is suppressed because of excessive accumulated penalty |

A withdrawn route can retain its history even after it has accumulated enough penalty to qualify for suppression.

The distinction is between whether the route is currently withdrawn and whether an advertised path is being suppressed.

### Penalty Cleanup

Cisco documents that dampening history is eventually removed after the penalty decreases sufficiently.

With the default reuse threshold of 750, dampening information can be discarded once the penalty decreases below approximately half the reuse threshold.

```text
750 / 2 = 375
```

At that point, if the route is stable and no longer suppressed, the router no longer needs to maintain its flap history.

---

## Which BGP Routes Are Dampened?

Cisco IOS XE applies traditional BGP route dampening to routes learned through **EBGP**.

Routes learned through IBGP are not independently penalized for their instability by the traditional mechanism.

### Example

```text
              AS 100                AS 200

                R1 ------------------ R2
                                      |
                                      | IBGP
                                      |
                                      R3
```

R1 advertises an unstable prefix to R2 over EBGP.

R2 has route flap dampening enabled.

R2 tracks the penalty associated with the route received from R1.

If the route becomes suppressed:

- R2 does not select the suppressed path for forwarding.
- R2 does not advertise that path to R3.
- The EBGP session between R1 and R2 remains Established.

R3 does not independently add penalties merely because the externally originated route was propagated through IBGP.

### Per-Path Behavior

Dampening operates on BGP routing information associated with particular paths.

Suppose R2 learns the same prefix from two EBGP neighbors:

```text
               AS 100          AS 300
                  R1              R3
                   \              /
                    \            /
                       R2
                     AS 200
```

If the path through R1 becomes dampened, the path through R3 can still be eligible.

R2 can select R3's path as its best route.

**Suppressing one unstable BGP path does not necessarily make the destination prefix unreachable if another valid path exists.**

---

## BGP Session Resets and Dampening

An important exception involves BGP neighbor session resets.

Cisco documents that resetting a BGP peer does **not** add a route flap penalty for the resulting withdrawals.

For example:

```text
R1# clear ip bgp 10.0.12.2
```

This performs a hard reset of the BGP session.

The routes learned through the neighbor are withdrawn as part of the reset.

However, Cisco IOS XE does not assign route dampening penalties for those withdrawals.

Cisco also documents that a peer reset clears the associated route flap statistics.

**Important:** Do not assume that repeatedly executing `clear ip bgp` will produce the same dampening behavior as repeatedly withdrawing and readvertising an individual prefix.

For a dampening lab, flap an advertised prefix without resetting the BGP session.

---

## Basic Route Flap Dampening Configuration

Enable route flap dampening using:

```text
R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# bgp dampening
```

This enables dampening with the default parameters.

| Parameter | Value |
|---|---:|
| Half-life | 15 minutes |
| Reuse | 750 |
| Suppress | 2,000 |
| Maximum suppress time | 60 minutes |

No additional neighbor configuration is required to enable the basic feature for the applicable EBGP-learned routes.

### Custom Parameters

Use:

```text
bgp dampening <HALF-LIFE> <REUSE> <SUPPRESS> <MAX-SUPPRESS>
```

For example:

```text
R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# bgp dampening 30 1500 10000 120
```

This configures:

| Parameter | Value |
|---|---:|
| Half-life | 30 minutes |
| Reuse | 1,500 |
| Suppress | 10,000 |
| Maximum suppress time | 120 minutes |

The route now requires a substantially higher accumulated penalty before being suppressed.

**Important:** The four numerical parameters are positional. If custom values are configured, all four must be supplied.

### Disabling Dampening

```text
R1(config-router-af)# no bgp dampening
```

This disables route flap dampening for the address family.

---

## Selective Route Flap Dampening

Cisco IOS XE supports selective route flap dampening using route maps.

This allows the administrator to apply dampening only to particular BGP prefixes.

### Configuration

Suppose R1 should apply dampening only to `192.0.2.0/24`.

Create a prefix list:

```text
R1(config)# ip prefix-list DAMPEN-PREFIX permit 192.0.2.0/24
```

Create a route map:

```text
R1(config)# route-map DAMPEN-SELECT permit 10
R1(config-route-map)# match ip address prefix-list DAMPEN-PREFIX
```

Apply the route map to BGP dampening:

```text
R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# bgp dampening route-map DAMPEN-SELECT
```

The route map determines which prefixes are subject to dampening.

| Prefix | Matches Route Map? | Dampening Applied? |
|---|---|---|
| `192.0.2.0/24` | Yes | Yes |
| `198.51.100.0/24` | No | No |
| `203.0.113.0/24` | No | No |

The route map used with `bgp dampening route-map` controls the applicability of dampening.

It is not an ordinary neighbor inbound route map for accepting or rejecting BGP routes.

**A prefix excluded from dampening is not automatically rejected by BGP. It simply does not receive dampening treatment from this policy.**

---

## Configuring Dampening Parameters with Route Maps

Cisco IOS XE also supports the `set dampening` command within route maps.

Syntax:

```text
set dampening <HALF-LIFE> <REUSE> <SUPPRESS> <MAX-SUPPRESS>
```

This allows different dampening parameters to be applied to different classes of routes.

### Example

Suppose R1 should apply custom dampening parameters to routes matching an AS-path access list.

Configure:

```text
R1(config)# ip as-path access-list 10 permit _65002$
```

Create a route map:

```text
R1(config)# route-map SET-DAMPEN permit 10
R1(config-route-map)# match as-path 10
R1(config-route-map)# set dampening 30 1500 10000 120
```

Apply the route map inbound:

```text
R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 route-map SET-DAMPEN in
```

This assigns the specified dampening parameters to matching routes on platforms supporting this route-map functionality.

The configured values are:

```text
Half-Life        = 30 minutes
Reuse Threshold  = 1500
Suppress         = 10000
Max Suppress     = 120 minutes
```

### `bgp dampening route-map` vs. `set dampening`

| Command | Purpose |
|---|---|
| `bgp dampening` | Enables dampening using the configured or default parameters |
| `bgp dampening route-map <NAME>` | Controls which routes are subject to dampening |
| `set dampening ...` | Defines dampening parameters through a route map |

These commands serve different purposes.

`bgp dampening route-map` selects where dampening is enabled.

`set dampening` specifies the dampening values used by matching routes.

---

## Verification

### Display Dampening Parameters

```text
R1# show ip bgp dampening parameters
```

This displays the configured dampening parameters.

The output can include fields such as:

```text
Half life time      : 15 min
Maximum suppress time: 60 min
Suppress penalty    : 2000
Reuse penalty       : 750
```

The exact formatting varies by IOS XE release.

### Display Flap Statistics

```text
R1# show ip bgp dampening flap-statistics
```

This displays BGP route flap information.

Useful fields include:

- Destination prefix.
- Number of flaps.
- Current penalty.
- Time since the first flap.
- Expected reuse time.

Some IOS XE platforms also support:

```text
R1# show ip bgp flap-statistics
```

### Display Dampened Routes

```text
R1# show ip bgp dampening dampened-paths
```

This displays routes currently affected by dampening suppression.

The output can include the time remaining until the routes are eligible for reuse.

### Display a Specific Route

```text
R1# show ip bgp 192.0.2.0
```

Check the route's status and detailed path information.

A dampened path may be marked with status code `d`.

A withdrawn route retained for dampening history may be marked with `h`.

### Display the BGP Table

```text
R1# show ip bgp
```

Useful status codes include:

| Code | Meaning |
|---|---|
| `d` | Dampened |
| `h` | History |
| `*` | Valid |
| `>` | Best |

Do not confuse the dampening status code `d` with ordinary BGP path suppression using aggregation.

The status code `s` can identify an aggregation-suppressed route, which is a different mechanism.

---

## Clearing Dampening Information

Cisco IOS XE supports clearing accumulated flap statistics and dampening information.

### Clear Flap Statistics

```text
R1# clear ip bgp flap-statistics
```

This clears accumulated flap statistics.

A specific prefix can also be targeted:

```text
R1# clear ip bgp flap-statistics 192.0.2.0 255.255.255.0
```

Clearing flap statistics resets the accumulated penalty information for the affected routes.

### Clear Dampened Paths

```text
R1# clear ip bgp dampened-paths
```

This clears dampening information and unsuppresses the affected routes.

To target a specific prefix:

```text
R1# clear ip bgp dampened-paths 192.0.2.0 255.255.255.0
```

If the route is currently advertised by the neighbor and otherwise eligible, it can become usable again.

**Important:** Clearing dampening information does not require terminating the BGP neighbor session.

---

## Route Flap Dampening Lab

This lab demonstrates penalty accumulation and suppression.

### Topology

```text
       AS 65001                    AS 65002

          R1 ------------------------ R2
       10.0.12.1                  10.0.12.2

      Dampening                 Loopback0
        Enabled               203.0.113.1/24
```

R2 advertises `203.0.113.0/24` to R1.

R1 has route flap dampening enabled.

### R2 Configuration

Configure a loopback:

```text
R2(config)# interface Loopback0
R2(config-if)# ip address 203.0.113.1 255.255.255.0
```

Configure BGP:

```text
R2(config)# router bgp 65002
R2(config-router)# neighbor 10.0.12.1 remote-as 65001
R2(config-router)# address-family ipv4 unicast
R2(config-router-af)# network 203.0.113.0 mask 255.255.255.0
```

### R1 Configuration

```text
R1(config)# router bgp 65001
R1(config-router)# neighbor 10.0.12.2 remote-as 65002
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# bgp dampening
```

Verify that the BGP session is Established:

```text
R1# show ip bgp summary
```

Verify the received prefix:

```text
R1# show ip bgp 203.0.113.0
```

### Generate Route Flaps

On R2, shut down Loopback0:

```text
R2(config)# interface Loopback0
R2(config-if)# shutdown
```

The connected route to `203.0.113.0/24` disappears from R2's routing table.

BGP withdraws the route from R1.

Restore the loopback:

```text
R2(config-if)# no shutdown
```

R2 readvertises the prefix.

Repeat the shutdown and restoration several times.

Allow each BGP withdrawal and readvertisement to be processed before performing the next flap.

### Observe Penalty Accumulation

On R1:

```text
R1# show ip bgp dampening flap-statistics
```

Observe the accumulated penalty and flap count.

With the default settings, repeated withdrawals cause the penalty to increase toward and eventually exceed the suppress threshold of 2,000.

### Observe Route Suppression

```text
R1# show ip bgp dampening dampened-paths
```

When the route is suppressed:

- R1 no longer considers the affected path for best-path selection.
- The path is not installed in the IP routing table.
- R1 does not advertise the suppressed path to other BGP neighbors.
- The BGP session with R2 remains Established.

### Observe Reuse

Stop flapping the route and leave Loopback0 enabled on R2.

The accumulated penalty gradually decreases.

Once the penalty falls below the reuse threshold of 750, the path can become eligible again.

Verify:

```text
R1# show ip bgp dampening flap-statistics
R1# show ip bgp 203.0.113.0
R1# show ip route 203.0.113.0
```

Alternatively, clear the dampening information:

```text
R1# clear ip bgp dampened-paths 203.0.113.0 255.255.255.0
```

Verify that the route is no longer dampened.

**Lab note:** The timing of flap processing and penalty decay depends on the IOS XE image. A hard BGP session reset is not a substitute for flapping the individual prefix.

---

## Operational Considerations

Route flap dampening was originally introduced to reduce the effects of unstable routes on the global BGP routing system.

However, aggressive dampening can create problems.

For example, a route may flap during a legitimate network convergence event.

If dampening suppresses the route, it can remain unavailable long after the original failure has been resolved.

This can result in:

- Unnecessarily prolonged reachability loss.
- Slower overall BGP convergence.
- Suppression of otherwise healthy routes.
- Unexpected routing behavior following transient failures.

### Modern Recommendations

RFC 7196, *Making Route Flap Damping Usable*, recommends more conservative suppression thresholds than traditional BGP defaults.

In particular:

| Suppress Threshold | Interpretation |
|---|---|
| 2,000 | Traditional Cisco default, considered overly aggressive for general Internet use |
| 6,000 or greater | Less aggressive dampening threshold recommended by RFC 7196 |
| 12,000 or greater | Conservative suppression threshold recommended by RFC 7196 |

These are suppression-threshold recommendations, not complete configurations.

Other parameters, including half-life, reuse, and maximum suppress time, must also be considered.

**Important:** The fact that Cisco IOS XE defaults to a suppress threshold of 2,000 does not mean this is an appropriate production configuration for Internet routing.

Route flap dampening is often avoided or configured conservatively because it can prolong outages.

For CCIE Enterprise Infrastructure, focus on understanding its operation, configuration, default values, and verification.

---

## Route Flap Dampening vs. Other BGP Features

| Feature | Purpose |
|---|---|
| Route Flap Dampening | Suppress unstable BGP paths |
| Maximum Prefix | Limit the number of prefixes accepted from a neighbor |
| Route Refresh | Request routing information again without resetting the BGP session |
| BGP Graceful Restart | Preserve eligible stale routes during control-plane recovery |
| BFD | Detect forwarding-path failures rapidly |
| BGP Next-Hop Tracking | React to changes in BGP NEXT_HOP reachability |

Route flap dampening is a route-stability mechanism.

It is not a replacement for BFD, BGP convergence, or maximum-prefix protection.

In fact, aggressive dampening can conflict with fast-convergence goals by preventing a recovered route from immediately becoming usable.

---

## Command Summary

| Command | Purpose |
|---|---|
| `bgp dampening` | Enable route flap dampening with default parameters |
| `bgp dampening 15 750 2000 60` | Explicitly configure traditional default parameters |
| `bgp dampening 30 1500 10000 120` | Configure custom dampening parameters |
| `bgp dampening route-map <NAME>` | Enable dampening selectively using a route map |
| `set dampening <HALF-LIFE> <REUSE> <SUPPRESS> <MAX-SUPPRESS>` | Configure dampening parameters within a route map |
| `no bgp dampening` | Disable route flap dampening |
| `show ip bgp dampening parameters` | Display dampening parameters |
| `show ip bgp dampening flap-statistics` | Display flap counts and penalties |
| `show ip bgp dampening dampened-paths` | Display suppressed routes |
| `clear ip bgp flap-statistics` | Clear accumulated flap statistics |
| `clear ip bgp dampened-paths` | Clear dampening information and unsuppress routes |

Command availability and output formatting depend on the Cisco IOS XE version and platform.

---

## Key Points

- **BGP Route Flap Dampening** suppresses unstable BGP paths to reduce routing update activity.
- It is defined in **RFC 2439**.
- Route flap dampening is **disabled by default** on Cisco IOS XE.
- A route withdrawal typically adds **1,000** penalty points.
- A route attribute change typically adds **500** penalty points.
- Penalties accumulate but decay exponentially over time.
- The default **half-life** is **15 minutes**.
- The default **reuse threshold** is **750**.
- The default **suppress threshold** is **2,000**.
- The default **maximum suppress time** is **60 minutes**.
- A route is suppressed when its penalty exceeds the suppress threshold.
- A suppressed route becomes eligible for reuse when its penalty falls below the reuse threshold.
- The maximum suppress time limits how long a route can remain suppressed.
- A suppressed path is not used for best-path selection, installed in the RIB, or advertised to other BGP neighbors.
- Status code **`h`** indicates a history entry for a withdrawn route.
- Status code **`d`** indicates a dampened path.
- Traditional Cisco IOS XE dampening applies to EBGP-learned paths, not independently to IBGP-learned routes.
- A BGP neighbor reset does not incur flap penalties for its resulting route withdrawals.
- `bgp dampening route-map` allows selective dampening.
- `set dampening` allows dampening parameters to be configured through a route map.
- Clearing dampening information does not require a BGP session reset.
- **Aggressive dampening can prolong outages**, so modern operational guidance recommends more conservative thresholds.

---

## References

- [Cisco: Configuring Internal BGP Features, IOS XE 17.x](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-routing/b-ip-routing/m_irg-int-features-0.html)
- [Cisco: BGP Command Reference, bgp dampening](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_bgp/command/irg-cr-book/bgp-a1.html)
- [Cisco: BGP Command Reference, set dampening](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_bgp/command/irg-cr-book/bgp-n1.html)
- [RFC 2439: BGP Route Flap Damping](https://www.rfc-editor.org/rfc/rfc2439)
- [RFC 7196: Making Route Flap Damping Usable](https://www.rfc-editor.org/rfc/rfc7196)
