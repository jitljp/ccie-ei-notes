# BGP Fast Convergence and BFD

**BGP fast convergence** refers to mechanisms that reduce the time required for BGP to detect failures, select alternative paths, and update routing and forwarding information.

By default, BGP relies partly on KEEPALIVE messages and the Hold Timer to detect neighbor failures.

However, waiting for the Hold Timer to expire can result in significant convergence delays.

Cisco IOS XE supports several mechanisms for improving BGP convergence:

- **Fast External Fallover:** Immediately resets directly connected EBGP sessions when the underlying interface goes down.
- **BGP Neighbor Fall-Over:** Monitors reachability to a BGP neighbor and resets the session when the route to that neighbor becomes unavailable.
- **Selective Address Tracking:** Uses a route map to determine whether a route to a BGP neighbor is acceptable for maintaining the session.
- **Bidirectional Forwarding Detection (BFD):** Provides rapid forwarding-path failure detection independently of BGP KEEPALIVE messages.
- **BGP Next-Hop Tracking (NHT):** Reacts to changes in the reachability of BGP NEXT_HOP addresses.

These mechanisms address different types of failures and can be used together.

**Important:** Fast failure detection is only one part of BGP convergence. The router must also select alternative paths and update its routing and forwarding tables.

---

## BGP Convergence Process

Consider a router with two possible paths to a destination:

```text
                  AS 65002
                  R2
                 /  \
                /    \
               /      \
              R1      R3
               \      /
                \    /
                  R4
              AS 65001
```

Suppose R4 has two BGP paths to a destination, one through R1 and another through R3.

If the preferred path through R1 fails, the network must converge.

The process generally involves:

1. **Failure detection:** Identify that the current forwarding path or BGP neighbor has failed.
2. **Path invalidation:** Mark the affected BGP path as unavailable.
3. **Best-path selection:** Select an alternative valid BGP path.
4. **RIB installation:** Install the selected route in the routing table, subject to normal RIB selection rules.
5. **FIB update:** Update forwarding information.
6. **Route advertisement:** Notify applicable BGP neighbors of routing changes.

The total convergence time depends on how quickly these operations complete.

### Factors Affecting Convergence

| Factor | Effect |
|---|---|
| Failure detection | Determines how quickly the router recognizes a failure |
| BGP table size | Affects the amount of routing information that may need processing |
| Available alternative paths | Determines whether another route can immediately replace the failed path |
| Best-path calculation | Determines which path becomes preferred |
| RIB/FIB programming | Determines when forwarding changes take effect |
| Routing policies | Can affect path eligibility and route advertisements |
| Advertisement intervals | Can delay the propagation of certain BGP updates |

A router can detect a failure quickly but still experience slower convergence because of subsequent routing calculations or forwarding updates.

---

## BGP Hold Timer

The **BGP Hold Timer** is the basic mechanism for detecting neighbors that stop sending BGP messages.

By default, Cisco IOS XE commonly uses:

| Timer | Default |
|---|---|
| Keepalive | 60 seconds |
| Hold | 180 seconds |

If a router does not receive a KEEPALIVE or UPDATE message within the negotiated Hold Time, it terminates the BGP session.

For example:

```text
          R1                           R2
           |                            |
           |<-------- KEEPALIVE --------|
           |                            |
           |      No more messages      |
           |                            |
           |       180 seconds          |
           |                            |
           |     Hold Timer Expires     |
           |                            |
           |------- NOTIFICATION ------>|
           |                            |
           |      Session Down          |
```

A failure that prevents BGP messages from arriving may therefore take up to the negotiated Hold Time to detect if no other failure-detection mechanism intervenes.

### Reducing BGP Timers

Configure shorter timers:

```text
R1(config)# router bgp 65001
R1(config-router)# neighbor 10.0.12.2 timers 5 15
```

This configures:

- Keepalive: 5 seconds
- Local advertised Hold Time: 15 seconds

The actual Hold Time is negotiated with the neighbor.

Shorter timers can improve failure detection, but aggressive values increase the risk of session resets during temporary packet loss or control-plane congestion.

**BFD is generally preferred when rapid forwarding-path failure detection is required.**

See [BGP Timers](../peering/timers.md) for further details.

---

## BGP Fast External Fallover

**BGP Fast External Fallover** allows a router to immediately reset a directly connected EBGP session when the interface used to reach the neighbor goes down.

This eliminates the need to wait for the BGP Hold Timer to expire when a physical or logical interface failure is detected.

Fast External Fallover is **enabled by default** on Cisco IOS XE.

### Example

Consider:

```text
       AS 65001                    AS 65002

          R1 ------------------------ R2
      10.0.12.1                  10.0.12.2
          E0/0                      E0/0

                  Direct EBGP
```

R1 establishes an EBGP session with R2.

Suppose R1's Ethernet0/0 interface goes down.

With Fast External Fallover enabled:

1. R1 detects that Ethernet0/0 is down.
2. BGP immediately resets the directly connected EBGP session.
3. Routes learned through R2 become unavailable for normal path selection.
4. BGP selects alternative paths if available.

R1 does not need to wait for the BGP Hold Timer.

### Configuration

Fast External Fallover is enabled by default, so no configuration is normally required.

Explicitly enable it:

```text
R1(config)# router bgp 65001
R1(config-router)# bgp fast-external-fallover
```

Disable it:

```text
R1(config-router)# no bgp fast-external-fallover
```

With Fast External Fallover disabled, BGP no longer uses this particular mechanism to reset the session when the interface fails.

Other mechanisms, such as TCP failure detection, BFD, or Hold Timer expiration, may still cause the session to terminate.

### Per-Interface Configuration

On supported Cisco IOS XE platforms, Fast External Fallover can also be controlled at the interface level.

```text
R1(config)# interface Ethernet0/0
R1(config-if)# ip bgp fast-external-fallover
```

This provides interface-specific control and can override the global setting.

### Limitations

Fast External Fallover has several important limitations:

- It applies to **directly connected EBGP sessions**.
- It is primarily triggered by the failure of the interface used to reach the neighbor.
- It does not provide equivalent failure detection for ordinary multihop BGP sessions.
- It does not necessarily detect a forwarding-path failure when the local interface remains up.

For example:

```text
       R1 -------- SW1 -------- R2
```

If SW1 stops forwarding packets but R1's interface remains up, Fast External Fallover may not detect the failure.

BFD can detect many failures of this type.

**Fast External Fallover monitors the directly connected interface state. BFD monitors forwarding-path liveliness.**

---

## BGP Neighbor Fall-Over Without BFD

Cisco IOS XE also supports **BGP Neighbor Fall-Over**, which provides event-driven monitoring of reachability to individual BGP neighbors.

Unlike Fast External Fallover, this feature can be useful for IBGP and multihop EBGP sessions.

Configure it using:

```text
R1(config-router)# neighbor <IP> fall-over
```

Without the `bfd` keyword, BGP uses routing information to determine whether the neighbor remains reachable.

### Example: IBGP over Loopbacks

Consider:

```text
                AS 65001

         R1 -------- R2 -------- R3
         Lo0                     Lo0
      1.1.1.1/32              3.3.3.3/32

         <------ IBGP -------->
```

R1 and R3 establish an IBGP session using their loopback addresses.

An IGP provides reachability between the loopbacks.

For example, R1's routing table contains:

```text
O 3.3.3.3/32 [110/20] via 10.0.12.2, Ethernet0/0
```

Configure R1:

```text
R1(config)# router bgp 65001
R1(config-router)# neighbor 3.3.3.3 remote-as 65001
R1(config-router)# neighbor 3.3.3.3 update-source Loopback0
R1(config-router)# neighbor 3.3.3.3 fall-over
```

R1 now monitors the route to the BGP peer address `3.3.3.3`.

### Failure Detection

Suppose R2 fails and R1 loses its route to `3.3.3.3`.

The process is:

1. The IGP withdraws the route to `3.3.3.3`.
2. R1's routing table reflects the reachability change.
3. BGP receives an event-driven notification.
4. Because the peer address is no longer reachable, BGP resets the session with R3.

The session can be terminated without waiting for the BGP Hold Timer.

### Important Distinction

`neighbor ... fall-over` tracks **reachability to the BGP peer address**, not necessarily the NEXT_HOP attributes of routes learned through that peer.

For example:

```text
BGP peer:      3.3.3.3
BGP NEXT_HOP:  10.10.10.10
```

These are different addresses.

- Neighbor fall-over monitors reachability to `3.3.3.3`.
- BGP Next-Hop Tracking monitors reachability to `10.10.10.10`.

Both mechanisms can be active simultaneously.

### Limitation: Alternative Routes

Suppose R1 has two routes to R3's loopback:

```text
             R2
            /  \
           /    \
         R1      R3
           \    /
            \  /
             R4
```

If the path through R2 fails but R1 retains a valid route through R4, the BGP peer address remains reachable.

Plain neighbor fall-over does not need to reset the BGP session merely because the forwarding path changed.

This is an important advantage of using loopback peering in redundant networks.

---

## Selective Neighbor Fall-Over Using a Route Map

Cisco IOS XE supports **BGP Selective Address Tracking**, which extends neighbor fall-over with route-map-based conditions.

Configure:

```text
R1(config-router)# neighbor <IP> fall-over route-map <NAME>
```

The route map evaluates the routing-table route used to reach the BGP neighbor.

The feature allows BGP to reset the session when the route to the peer changes to one that is not acceptable.

### Why Is Selective Tracking Useful?

Consider an IBGP session between R1 and R3:

```text
         R1 -------- R2 -------- R3
         Lo0                     Lo0
      1.1.1.1/32              3.3.3.3/32
```

R1 initially reaches R3 through OSPF:

```text
O 3.3.3.3/32 [110/20] via 10.0.12.2
```

R1 also has a default route:

```text
S* 0.0.0.0/0 [1/0] via 10.0.14.4
```

Suppose the OSPF route disappears.

The default route might still provide a routing-table match for `3.3.3.3`.

Plain neighbor fall-over may therefore consider the peer address reachable.

However, the administrator wants the BGP session to remain established **only while R1 has a specific /32 route to the peer**.

### Configuration

Create a prefix list that matches host routes:

```text
R1(config)# ip prefix-list PEER-HOST permit 0.0.0.0/0 ge 32
```

Create a route map:

```text
R1(config)# route-map TRACK-PEER permit 10
R1(config-route-map)# match ip address prefix-list PEER-HOST
```

Apply it to the BGP neighbor:

```text
R1(config)# router bgp 65001
R1(config-router)# neighbor 3.3.3.3 fall-over route-map TRACK-PEER
```

### Operation

The route map evaluates the routing-table route used to reach the peer.

| Resolving Route | Route-Map Result | Session Behavior |
|---|---|---|
| `3.3.3.3/32` via OSPF | Permit | No fall-over reset required |
| `3.3.3.3/32` via static route | Permit | No fall-over reset required |
| `3.3.3.0/24` via OSPF | Deny | Reset |
| `0.0.0.0/0` via static route | Deny | Reset |
| No route to peer | Unreachable | Reset |

If the new resolving route is denied by the route map, BGP resets the session.

**Important:** The route map is evaluated against the routing-table route used to reach the BGP peer. It is not applied to BGP prefixes received from the neighbor.

### Matching the Route Source

Instead of matching prefix length, the route map can match the routing protocol that installed the resolving route.

For example:

```text
R1(config)# route-map TRACK-OSPF permit 10
R1(config-route-map)# match source-protocol ospf
```

Apply it:

```text
R1(config)# router bgp 65001
R1(config-router)# neighbor 3.3.3.3 fall-over route-map TRACK-OSPF
```

This allows the BGP session to remain active while the route to the peer is resolved through OSPF.

If the resolving route changes to an unacceptable source, such as a static default route, the route map denies it and BGP resets the session.

### Supported Route-Map Commands

Cisco documents two supported match commands for selective neighbor fall-over:

- `match ip address`
- `match source-protocol`

Other match commands and `set` commands are not supported for this purpose.

**The route map is used for monitoring changes to the peer route, not for initial BGP session establishment.**

---

## Bidirectional Forwarding Detection (BFD)

**Bidirectional Forwarding Detection (BFD)** is a protocol designed to detect forwarding-path failures rapidly.

BFD operates independently of the BGP KEEPALIVE and Hold Timer mechanisms.

It can be used by multiple routing protocols, including:

- BGP
- OSPF
- EIGRP
- IS-IS

BFD provides a common failure-detection mechanism with configurable detection intervals.

### How BFD Works

Two neighboring routers establish a BFD session.

They exchange BFD control packets at negotiated intervals.

```text
          R1                            R2
           |                             |
           |-------- BFD Control ------->|
           |<------- BFD Control --------|
           |                             |
           |-------- BFD Control ------->|
           |<------- BFD Control --------|
           |                             |
           |-------- BFD Control ------->|
           |                             |
```

If a router stops receiving the expected BFD packets for the negotiated detection period, it declares the BFD session Down.

The BFD process notifies the routing protocol registered as a client.

For BGP:

1. BFD detects the forwarding-path failure.
2. BFD reports the failure to BGP.
3. BGP resets the affected session.
4. Routes learned from that neighbor become unavailable for normal path selection.
5. BGP selects alternative routes when available.

This can occur significantly faster than BGP Hold Timer expiration.

### BFD Session States

BFD defines four session states.

| State | Meaning |
|---|---|
| AdminDown | Session has been administratively disabled |
| Down | BFD session is not operational |
| Init | Router is attempting to establish the BFD session |
| Up | BFD session is operational |

BFD uses its own state machine, independently of BGP.

A BGP session being Established does not automatically prove that BFD is Up.

### BFD Operating Modes

BFD supports two principal operating modes:

| Mode | Description |
|---|---|
| Asynchronous | Peers periodically exchange BFD control packets |
| Demand | Periodic control packets can be suppressed after the session is established, under the applicable conditions |

Cisco IOS XE's traditional BGP BFD implementation primarily uses **asynchronous mode**.

BFD can also support an optional Echo function, subject to platform capabilities.

---

## BFD Timers

BFD supports configurable transmission intervals and detection multipliers.

On Cisco IOS XE, traditional interface-level BFD configuration uses:

```text
R1(config-if)# bfd interval <TX-MS> min_rx <RX-MS> multiplier <MULTIPLIER>
```

### Example

```text
R1(config)# interface Ethernet0/0
R1(config-if)# bfd interval 100 min_rx 100 multiplier 3
```

This configures:

| Parameter | Value | Meaning |
|---|---|---|
| `interval 100` | 100 ms | Desired minimum BFD transmission interval |
| `min_rx 100` | 100 ms | Minimum acceptable BFD receive interval |
| `multiplier 3` | 3 | Detection multiplier advertised to the peer |

### Detection Time

In a simple symmetrical configuration, where both peers use 100 ms intervals and a multiplier of 3:

```text
Detection Time = 100 ms * 3

Detection Time = 300 ms
```

A router can detect the failure in approximately 300 milliseconds, assuming the negotiated intervals support that result.

### BFD Timer Negotiation

BFD peers do not necessarily transmit at their locally configured desired intervals.

The actual transmission interval must satisfy both:

- The sender's desired minimum transmission interval.
- The receiver's required minimum receive interval.

For example:

| Parameter | R1 | R2 |
|---|---:|---:|
| Desired minimum TX | 100 ms | 200 ms |
| Required minimum RX | 200 ms | 100 ms |
| Detection multiplier | 3 | 3 |

R1 must transmit no faster than 200 ms because R2 requires at least 200 ms between received packets.

R2 also transmits no faster than 200 ms because its desired minimum TX is 200 ms.

The resulting detection times are approximately 600 ms in both directions, assuming stable negotiated intervals.

**BFD timers are negotiated independently of BGP KEEPALIVE and Hold Timers.**

### Aggressive BFD Timers

Very short BFD intervals can provide faster detection but increase processing overhead.

Potential problems include:

- CPU overload.
- Packet loss.
- False failure detection.
- Routing protocol flapping.
- Increased control-plane processing.

Supported minimum intervals depend on the platform, forwarding architecture, and whether BFD hardware offload is available.

Use timer values appropriate for the devices and network conditions.

---

## BGP Neighbor Fall-Over with BFD

Cisco IOS XE integrates BFD with BGP using:

```text
R1(config-router)# neighbor <IP> fall-over bfd
```

This enables BFD-based failure detection for the specified BGP neighbor.

Unlike plain `neighbor ... fall-over`, BFD-based fall-over monitors forwarding-path liveliness using BFD packets.

### Configuration Requirements

For traditional single-hop BFD:

1. IP connectivity must exist between the BGP neighbors.
2. CEF must be enabled.
3. BFD must be configured on the applicable interfaces.
4. BGP must register with BFD using `neighbor ... fall-over bfd`.
5. Both routers must support compatible BFD operation.

Configuring BFD timers on an interface does not necessarily create an active BFD session.

A routing protocol or another application must register as a BFD client.

### Example: Single-Hop EBGP with BFD

```text
       AS 65001                    AS 65002

          R1 ------------------------ R2
      10.0.12.1                  10.0.12.2
          E0/0                      E0/0

                    EBGP
                    BFD
```

Assume the interfaces already have their IP addresses configured.

#### R1 Configuration

Configure BFD:

```text
R1(config)# interface Ethernet0/0
R1(config-if)# bfd interval 200 min_rx 200 multiplier 3
```

Configure BGP:

```text
R1(config)# router bgp 65001
R1(config-router)# neighbor 10.0.12.2 remote-as 65002
R1(config-router)# neighbor 10.0.12.2 fall-over bfd
```

#### R2 Configuration

Configure BFD:

```text
R2(config)# interface Ethernet0/0
R2(config-if)# bfd interval 200 min_rx 200 multiplier 3
```

Configure BGP:

```text
R2(config)# router bgp 65002
R2(config-router)# neighbor 10.0.12.1 remote-as 65001
R2(config-router)# neighbor 10.0.12.1 fall-over bfd
```

### Expected Operation

Once the BFD session is established:

```text
          R1                          R2
           |                           |
           |       BGP Established     |
           |<------------------------->|
           |                           |
           |         BFD Up            |
           |<------------------------->|
           |                           |
           |     Forwarding Failure    |
           |                           |
           |      BFD Detects Down     |
           |                           |
           |      BGP Session Down     |
```

BFD notifies BGP when it detects the failure.

With symmetrical 200 ms intervals and a multiplier of 3, the nominal BFD detection time is approximately 600 ms.

This is much faster than the default 180-second BGP Hold Timer.

### Verification

```text
R1# show bfd neighbors
R1# show bfd neighbors details
R1# show ip bgp neighbors 10.0.12.2
```

The detailed BGP neighbor output can include:

```text
Fall over configured for session
BFD is configured. BFD peer is Up.
Using BFD to detect fast fallover (single-hop).
```

The exact output varies by IOS XE version.

**Important:** `neighbor ... fall-over bfd` does not change the negotiated BGP Hold Timer. It provides an additional, faster failure-detection mechanism.

---

## Single-Hop BFD vs. Fast External Fallover

Both BFD and Fast External Fallover can detect failures affecting directly connected EBGP neighbors.

However, they detect different events.

Consider:

```text
       R1 -------- SW1 -------- R2
```

### Scenario 1: R1's Interface Goes Down

If R1's interface goes down:

- Fast External Fallover can immediately reset the EBGP session.
- BFD can also detect the failure if the BFD session is affected.
- The BGP Hold Timer is not required for failure detection.

### Scenario 2: Intermediate Forwarding Failure

Suppose SW1 stops forwarding traffic but R1's Ethernet interface remains up.

- Fast External Fallover may not detect the failure.
- BFD can detect that packets are no longer being received.
- BGP can reset the session when BFD reports the failure.

### Comparison

| Feature | Fast External Fallover | BFD-Based Fall-Over |
|---|---|---|
| Detects local interface-down events | Yes | Yes, if BFD is affected |
| Detects failures while interface remains up | Not necessarily | Yes, if BFD packets are lost |
| Uses BGP Hold Timer | No | No |
| Requires BFD configuration | No | Yes |
| Supports directly connected EBGP | Yes | Yes |
| Supports IBGP | No | Yes |
| Supports multihop BGP | No | Yes, with multihop BFD support |
| Enabled by default | Yes | No |

**BFD provides forwarding-path failure detection, while Fast External Fallover primarily reacts to directly connected interface failures.**

---

## Multihop BFD for BGP

Cisco IOS XE also supports **multihop BFD**, which allows BFD to monitor BGP sessions between routers that are not directly connected.

This is useful for:

- IBGP sessions between loopback addresses.
- Multihop EBGP sessions.
- Redundant routed networks.
- BGP sessions carried across an IGP underlay.

### Example Topology

```text
                   AS 65001

           R1 -------- R2 -------- R3
           Lo0                     Lo0
        1.1.1.1/32              3.3.3.3/32

           <------ IBGP -------->
           <------ BFD --------->
```

R1 and R3 are IBGP neighbors using their loopbacks.

Assume OSPF provides reachability between the loopbacks.

A multihop BFD session can monitor forwarding connectivity between `1.1.1.1` and `3.3.3.3`.

### BFD Template

Cisco IOS XE supports multihop BFD templates.

For example, on R1:

```text
R1(config)# bfd-template multi-hop MH-BFD
R1(config-bfd)# interval min-tx 200 min-rx 200 multiplier 3
```

Create a BFD map:

```text
R1(config)# bfd map ipv4 3.3.3.3/32 1.1.1.1/32 MH-BFD
```

The map associates the remote and local BFD endpoints with the multihop BFD template.

Configure BGP:

```text
R1(config)# router bgp 65001
R1(config-router)# neighbor 3.3.3.3 remote-as 65001
R1(config-router)# neighbor 3.3.3.3 update-source Loopback0
R1(config-router)# neighbor 3.3.3.3 fall-over bfd multi-hop
```

On R3, configure the corresponding template:

```text
R3(config)# bfd-template multi-hop MH-BFD
R3(config-bfd)# interval min-tx 200 min-rx 200 multiplier 3
```

Configure the reverse BFD map:

```text
R3(config)# bfd map ipv4 1.1.1.1/32 3.3.3.3/32 MH-BFD
```

Configure BGP:

```text
R3(config)# router bgp 65001
R3(config-router)# neighbor 1.1.1.1 remote-as 65001
R3(config-router)# neighbor 1.1.1.1 update-source Loopback0
R3(config-router)# neighbor 1.1.1.1 fall-over bfd multi-hop
```

The BFD session now monitors connectivity between the two loopbacks.

### Important Multihop Consideration

Multihop BFD timers must account for IGP reconvergence.

For example:

```text
             R2
            /  \
           /    \
         R1      R3
           \    /
            \  /
             R4
```

If the path through R2 fails, OSPF may converge to the alternative path through R4.

If BFD detects a failure before OSPF completes reconvergence, the BGP session could be reset unnecessarily.

Cisco recommends that the multihop BFD detection time be longer than the expected IGP reconvergence time for such designs.

**A very short BFD detection time is not always desirable for multihop BGP sessions.**

Multihop BFD support and restrictions depend on the IOS XE platform and software release.

---

## BFD Strict Mode

Normally, configuring BFD-based fall-over does not necessarily prevent a BGP session from becoming Established while the corresponding BFD session is Down.

Cisco IOS XE supports an optional **BFD Strict Mode** to change this behavior.

### Without Strict Mode

With:

```text
R1(config-router)# neighbor 10.0.12.2 fall-over bfd
```

BGP session establishment and BFD state are not strictly coupled.

On supported implementations, the BGP session can become Established even when the BFD session has not reached Up.

However, an operational BFD session that subsequently detects a failure can trigger BGP session termination.

### With Strict Mode

Configure:

```text
R1(config-router)# neighbor 10.0.12.2 fall-over bfd single-hop strict-mode
```

With Strict Mode, BGP is not allowed to become Established while the corresponding BFD session is Down.

| BFD State | BGP Without Strict Mode | BGP With Strict Mode |
|---|---|---|
| Up | Can establish | Can establish |
| Down | May establish | Cannot become Established |
| Dampened | May establish | Cannot become Established |

Strict Mode requires compatible BFD configuration on both neighbors.

It is useful when the administrator wants BGP session establishment to depend on verified forwarding connectivity.

However, Strict Mode can prevent a BGP session from establishing if BFD is misconfigured or unsupported.


### BFD Control-Plane Independence

Cisco IOS XE supports the optional **`check-control-plane-failure`** keyword, which allows BGP to consider BFD's **Control Plane Independent (C-bit)** information when responding to a BFD failure.

For example:

```text
R1(config)# router bgp 65001
R1(config-router)# neighbor 3.3.3.3 fall-over bfd check-control-plane-failure
```

For multihop BFD:

```text
R1(config-router)# neighbor 3.3.3.3 fall-over bfd multi-hop check-control-plane-failure
```

This feature is particularly useful when BFD is combined with **BGP Graceful Restart (GR)** and **Nonstop Forwarding (NSF)**.

The central problem is that a BFD session can go Down for different reasons:

1. **Forwarding-plane failure:** The network can no longer forward traffic to the neighbor.
2. **Control-plane failure:** A router's control plane restarts, potentially interrupting BFD even though its forwarding plane continues operating.

These failures should not necessarily produce the same routing behavior.

#### BFD and BGP Graceful Restart

BGP Graceful Restart allows a router to preserve forwarding during a BGP control-plane restart.

Consider two BGP neighbors:

```text
          R1                           R2
           |                            |
           |        BGP Session         |
           |<-------------------------->|
           |                            |
           |        BFD Session         |
           |<-------------------------->|
           |                            |
```

Suppose R2's BGP control plane restarts, but its forwarding plane continues operating.

With BGP Graceful Restart, R1 may retain routes previously learned from R2 as **stale routes** while R2 restarts its control plane.

This allows traffic to continue using R2, provided its forwarding plane is still operational.

However, what happens if the BFD session also goes Down?

By default, BGP treats a BFD Down notification as evidence that the forwarding path has failed and removes the routes learned through the neighbor.

This can defeat the purpose of Graceful Restart if BFD went Down only because the neighbor's control plane restarted.

The `check-control-plane-failure` option allows BGP to distinguish between these situations when deciding whether to preserve stale routes.

#### The BFD C-Bit

BFD control packets contain a **Control Plane Independent (C)** bit.

The C-bit indicates whether a router's BFD implementation operates independently of its control plane.

| C-Bit | Meaning |
|---|---|
| `1` (Set) | BFD operates independently of the control plane |
| `0` (Clear) | BFD is not independent of the control plane |

The distinction is important.

**C-bit = 1**

If BFD is independent of the control plane, a control-plane restart should not interrupt the BFD session.

Therefore, if BFD goes Down, it strongly indicates that the forwarding path has failed.

Continuing to use stale routes through that neighbor could result in traffic being blackholed.

**C-bit = 0**

If BFD is dependent on the control plane, a control-plane restart can also interrupt BFD.

Therefore, a BFD failure does not necessarily mean the forwarding plane has failed.

BGP may be able to preserve stale routes while Graceful Restart completes.

**Important:** A cleared C-bit does not prove that the failure was caused by a control-plane restart. It indicates that BFD cannot reliably distinguish that possibility from a forwarding failure.

For BFD to be considered fully control-plane independent under RFC 5882, the C-bit must be set by **both BFD peers**.

If either peer clears the C-bit, the BFD session may be affected by a control-plane restart.

#### Behavior Without `check-control-plane-failure`

By default, BGP does not use the C-bit information when deciding whether to preserve routes after a BFD Down notification.

Configure:

```text
R1(config-router)# neighbor 3.3.3.3 fall-over bfd
```

Suppose BFD reports that the session with R3 has gone Down.

Regardless of whether BFD is control-plane independent:

1. BFD notifies BGP that the session has failed.
2. BGP resets the affected peering session.
3. BGP purges the routes learned through that neighbor.
4. BGP selects alternative paths, if available.

Even if BGP Graceful Restart is configured, the BFD failure causes the routes to be purged under this default behavior.

| BFD Control-Plane Independent? | Action When BFD Goes Down |
|---|---|
| Yes | Purge routes |
| No | Purge routes |

This prioritizes immediate route removal following BFD failure.

However, if BFD was interrupted by a control-plane restart while forwarding remained operational, the route removal may be unnecessary.

#### Behavior With `check-control-plane-failure`

Configure:

```text
R1(config-router)# neighbor 3.3.3.3 fall-over bfd check-control-plane-failure
```

Now, when BFD reports a failure, BGP considers the C-bit information before deciding whether to purge routes or continue Graceful Restart.

**Scenario 1: BFD Is Control-Plane Independent**

Assume both BFD peers advertise C-bit = 1.

The BFD session goes Down.

Because BFD operates independently of the control plane, its failure indicates that the forwarding path is no longer operational.

BGP therefore:

1. Processes the BFD Down notification.
2. Resets the affected BGP session.
3. Purges the routes learned through the neighbor.
4. Selects alternative paths, if available.

Graceful Restart route retention is not continued for this BFD failure.

This avoids forwarding traffic through a failed path.

**Scenario 2: BFD Is Control-Plane Dependent**

Assume at least one BFD peer advertises C-bit = 0.

The BFD session goes Down.

Because BFD is not independent of the control plane, the failure might have been caused by a control-plane restart.

If BGP Graceful Restart is applicable, BGP can:

1. Process the BFD Down notification.
2. Allow the affected BGP session to go Down.
3. Continue Graceful Restart/NSF procedures.
4. Preserve eligible stale routes in the RIB.
5. Allow forwarding to continue using those routes while the neighbor restarts.

The stale routes are retained subject to normal Graceful Restart conditions and timers.

**Important:** `check-control-plane-failure` does not keep the BGP TCP session Established when BFD fails. Its purpose is to influence whether routes are purged or retained during Graceful Restart.

#### Behavior Comparison

Cisco documents the following behavior when BFD reports a failure and BGP Graceful Restart/NSF is enabled:

| `check-control-plane-failure` | BFD Control-Plane Independent? | BGP Action |
|---|---|---|
| Disabled (default) | Yes | Purge routes |
| Disabled (default) | No | Purge routes |
| Enabled | Yes | Purge routes |
| Enabled | No | Continue NSF and retain eligible stale routes |

The only situation in which the command changes the default behavior is when:

- `check-control-plane-failure` is enabled.
- BFD is not control-plane independent.
- BGP Graceful Restart/NSF is applicable.

In that situation, BGP can preserve stale routes rather than immediately removing them.

#### Example: Control-Plane Restart

Consider R1 and R2 running BGP, BFD, and Graceful Restart.

```text
          R1                           R2
           |                            |
           |        BGP Established     |
           |<-------------------------->|
           |                            |
           |           BFD Up           |
           |<-------------------------->|
           |                            |
           |                     Control Plane
           |                       Restarts
           |                            |
           |         BFD Down           |
           |                            |
```

Assume:

- R2's forwarding plane continues operating.
- BFD depends on R2's control plane.
- R1 is acting as a Graceful Restart helper.

**Without `check-control-plane-failure`:**

R1 receives the BFD Down notification and removes R2's routes.

Traffic must use alternative paths, even though R2 might still be capable of forwarding packets.

**With `check-control-plane-failure`:**

R1 recognizes that BFD is control-plane dependent.

Instead of immediately purging R2's routes, R1 can retain eligible stale routes while Graceful Restart completes.

This can help prevent unnecessary traffic disruption during the control-plane restart.

#### Example: Forwarding-Plane Failure

Now suppose R2's forwarding plane actually fails.

If BFD is control-plane independent, its session goes Down independently of any control-plane restart.

With `check-control-plane-failure` enabled, R1 interprets this as a forwarding-path failure and removes the affected routes.

This prevents stale routes from continuing to direct traffic toward a failed forwarding path.

However, if BFD is control-plane dependent, the failure is ambiguous.

BGP may preserve stale routes even though the forwarding plane has actually failed.

This can temporarily blackhole traffic.

Therefore, **`check-control-plane-failure` is not a guarantee against traffic loss**. It allows BGP to make a more informed decision based on the capabilities of the BFD implementation.

#### Important Considerations

- The command does not enable BGP Graceful Restart. GR must be separately supported and configured as appropriate.
- The command does not enable NSF or make a forwarding plane capable of surviving a control-plane restart.
- The command does not force the BFD C-bit to 1. The bit reflects the capabilities of the BFD implementation.
- If BGP Graceful Restart is not applicable, the command does not provide stale-route preservation.
- If BFD is control-plane independent, a BFD failure results in route removal even when `check-control-plane-failure` is configured.
- If BFD is control-plane dependent, route preservation carries some risk because a genuine forwarding failure cannot always be distinguished from a control-plane failure.
- Support for control-plane-independent BFD varies by platform and implementation.

#### Difference from BFD Strict Mode

`check-control-plane-failure` and `strict-mode` serve different purposes.

| Feature | Purpose |
|---|---|
| `check-control-plane-failure` | Determines whether routes should be purged or preserved during GR/NSF after a BFD failure |
| `strict-mode` | Requires the BFD session to be Up before BGP can become Established |

For example:

```text
R1(config-router)# neighbor 3.3.3.3 fall-over bfd multi-hop check-control-plane-failure strict-mode
```

This combines two independent behaviors:

1. **Strict Mode:** BGP cannot become Established until BFD is Up.
2. **Control-Plane Failure Checking:** BGP considers the C-bit when deciding whether to preserve stale routes after a BFD failure.

Strict Mode controls **session establishment**, while `check-control-plane-failure` controls **route retention during Graceful Restart**.

#### Key Points

- BFD's **C-bit** indicates whether BFD operates independently of the control plane.
- **C-bit = 1:** BFD is control-plane independent.
- **C-bit = 0:** BFD is control-plane dependent.
- Without `check-control-plane-failure`, BGP purges the neighbor's routes when BFD reports Down.
- With `check-control-plane-failure`, BGP considers whether BFD is control-plane independent.
- If BFD is independent, BGP purges routes to avoid forwarding through a failed path.
- If BFD is dependent, BGP can preserve eligible stale routes through Graceful Restart.
- This feature requires applicable BGP Graceful Restart/NSF behavior to provide a benefit.
- **`check-control-plane-failure` does not change the BFD C-bit or require BFD to be Up before BGP can establish.**

#### References

- [Cisco: BFD, BGP Multihop Client Support, cBit, and Strict Mode](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-routing/b-ip-routing/m_irg-multihop-cbit.html)
- [RFC 5880: Bidirectional Forwarding Detection](https://www.rfc-editor.org/rfc/rfc5880)
- [RFC 5882: Generic Application of BFD, Section 4.3](https://www.rfc-editor.org/rfc/rfc5882)

---

## BGP Next-Hop Tracking

**BGP Next-Hop Tracking (NHT)** provides event-driven monitoring of BGP NEXT_HOP reachability.

It allows BGP to react to routing-table changes without waiting for the traditional periodic BGP scanner.

NHT is enabled by default for IPv4 unicast on supported Cisco IOS XE platforms.

### Example

Suppose R1 learns:

```text
BGP Prefix: 203.0.113.0/24
NEXT_HOP:   3.3.3.3
```

R1's routing table contains:

```text
O 3.3.3.3/32 [110/20] via 10.0.12.2
```

BGP uses the OSPF route to resolve the BGP NEXT_HOP.

If the OSPF route disappears:

1. The routing table detects the loss of reachability.
2. NHT notifies BGP.
3. BGP reevaluates paths using that next hop.
4. If the next hop is unreachable, the affected BGP path becomes unusable.
5. BGP selects another valid path, if available.

### NHT Trigger Delay

Cisco IOS XE commonly uses a default **5-second NHT trigger delay** for the documented IPv4 implementation.

Configure:

```text
R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# bgp nexthop trigger delay 1
```

This configures a 1-second trigger delay.

The trigger delay controls when BGP processes next-hop change notifications.

A shorter delay can improve convergence but can also cause more frequent best-path recalculations during IGP instability.

For complete NHT configuration, see [BGP Next-Hop Handling](../peering/next-hop.md).

---

## Neighbor Fall-Over vs. Next-Hop Tracking

Although both mechanisms use routing information, they monitor different addresses.

Consider:

```text
              R1 -------- R2 -------- R3

        BGP Peer:     2.2.2.2
        Route Prefix: 203.0.113.0/24
        BGP NEXT_HOP: 3.3.3.3
```

R1 has an IBGP session with R2 and receives a route whose NEXT_HOP is `3.3.3.3`.

### Neighbor Fall-Over

```text
R1(config-router)# neighbor 2.2.2.2 fall-over
```

This monitors reachability to the **BGP peer address**, `2.2.2.2`.

If the peer address becomes unreachable, BGP can reset the session.

### Next-Hop Tracking

NHT monitors reachability to the **BGP NEXT_HOP**, `3.3.3.3`.

If `3.3.3.3` becomes unreachable:

- The affected route can become invalid.
- The BGP session with `2.2.2.2` can remain Established.

### Comparison

| Feature | Monitored Address | Typical Action |
|---|---|---|
| `neighbor ... fall-over` | BGP peer address | Reset BGP session |
| `neighbor ... fall-over route-map` | RIB route resolving the peer address | Reset session when route becomes unacceptable |
| `neighbor ... fall-over bfd` | BFD forwarding path to peer | Reset session when BFD reports failure |
| BGP NHT | NEXT_HOP attribute of BGP paths | Reevaluate affected BGP paths |

**Important:** A BGP route can become unusable because its NEXT_HOP is unreachable even though the BGP session that advertised the route remains Established.

---

## Comparing Fast-Convergence Mechanisms

| Mechanism | Trigger | Primary Effect | Requires BFD? |
|---|---|---|---|
| BGP Hold Timer | Missing BGP messages for negotiated Hold Time | Reset session | No |
| Fast External Fallover | Directly connected EBGP interface goes down | Reset session | No |
| Neighbor Fall-Over | Peer address becomes unreachable | Reset session | No |
| Selective Neighbor Fall-Over | Resolving route becomes unacceptable | Reset session | No |
| BFD Fall-Over | BFD detects forwarding-path failure | Reset session | Yes |
| BGP NHT | BGP NEXT_HOP reachability changes | Reevaluate BGP paths | No |

These mechanisms are not mutually exclusive.

For example, a directly connected EBGP session might have:

- Fast External Fallover enabled.
- BFD configured.
- Normal BGP KEEPALIVE and Hold Timers.

The mechanism that detects the failure first can cause BGP to reset the session.

---

## Configuration Lab

This lab compares BGP failure detection with and without BFD.

### Topology

```text
       AS 65001                    AS 65002

          R1 ------------------------ R2
      10.0.12.1                  10.0.12.2
          E0/0                      E0/0

                     EBGP
```

Assume the interfaces are already configured and the EBGP session is Established.

### Test 1: Fast External Fallover

On R1:

```text
R1(config)# router bgp 65001
R1(config-router)# bgp fast-external-fallover
```

On R2, shut down the interface:

```text
R2(config)# interface Ethernet0/0
R2(config-if)# shutdown
```

Observe the BGP session on R2 and R1.

R2 should detect the local interface failure, and R1 should also detect it if the link state goes down.

The session can terminate without waiting for the BGP Hold Timer.

Restore the interface:

```text
R2(config-if)# no shutdown
```

### Test 2: Neighbor Fall-Over Without BFD

Use a three-router topology with loopback-based IBGP peering:

```text
            R1 -------- R2 -------- R3
             <------- IBGP -------->
```

Configure on R1:

```text
R1(config)# router bgp 65001
R1(config-router)# neighbor 3.3.3.3 fall-over
```

Ensure the IGP provides a route to `3.3.3.3/32`.

Verify:

```text
R1# show ip route 3.3.3.3
R1# show ip bgp neighbors 3.3.3.3
```

Then remove the IGP route to R3 by shutting down the transit interface or modifying the lab topology.

If R1 loses all acceptable reachability to `3.3.3.3`, neighbor fall-over should reset the BGP session without waiting for the Hold Timer.

Restore the route and verify that BGP reestablishes.

### Test 3: Single-Hop BFD

Configure BFD on R1:

```text
R1(config)# interface Ethernet0/0
R1(config-if)# bfd interval 200 min_rx 200 multiplier 3

R1(config)# router bgp 65001
R1(config-router)# neighbor 10.0.12.2 fall-over bfd
```

Configure the corresponding settings on R2:

```text
R2(config)# interface Ethernet0/0
R2(config-if)# bfd interval 200 min_rx 200 multiplier 3

R2(config)# router bgp 65002
R2(config-router)# neighbor 10.0.12.1 fall-over bfd
```

Verify:

```text
R1# show bfd neighbors
R1# show bfd neighbors details
R1# show ip bgp neighbors 10.0.12.2
```

Confirm the BFD session is Up.

Test failure detection by interrupting BFD packet delivery while leaving the BGP TCP session's basic reachability intact.

In a lab, this can be simulated using an appropriate interface ACL or another controlled packet-filtering mechanism.

Observe how quickly BFD detects the failure and notifies BGP.

**Note:** Shutting down the local interface is not a good way to isolate BFD behavior because Fast External Fallover may detect the interface failure first.

### Test 4: Selective Fall-Over

Using loopback-based peering, configure:

```text
R1(config)# ip prefix-list PEER-HOST permit 3.3.3.3/32

R1(config)# route-map TRACK-PEER permit 10
R1(config-route-map)# match ip address prefix-list PEER-HOST

R1(config)# router bgp 65001
R1(config-router)# neighbor 3.3.3.3 fall-over route-map TRACK-PEER
```

Ensure R1 has:

- A specific route to `3.3.3.3/32`.
- A less-specific backup route, such as `0.0.0.0/0`.

Remove the specific route.

Even if the backup route still matches the peer address, the route map should reject it and cause BGP to reset the session.

This demonstrates the difference between ordinary neighbor fall-over and selective address tracking.

---

## Verification

### BGP Neighbor Status

```text
R1# show ip bgp summary
R1# show ip bgp neighbors 10.0.12.2
```

Verify:

- Neighbor state.
- Session uptime.
- Last reset reason.
- BGP KEEPALIVE and Hold Timers.
- Fall-over configuration.
- BFD status, if configured.

### BFD Session Status

```text
R1# show bfd neighbors
R1# show bfd neighbors details
```

Verify:

- BFD peer address.
- BFD session state.
- Negotiated transmission and receive intervals.
- Detection multiplier.
- Session uptime and diagnostic information.

### IP Reachability

```text
R1# show ip route 3.3.3.3
R1# show ip cef 3.3.3.3
```

These commands help determine whether the BGP peer address remains reachable.

They are particularly useful when troubleshooting plain neighbor fall-over and selective address tracking.

### Routing Table Changes

```text
R1# show ip bgp
R1# show ip route
R1# show ip cef
```

Verify whether alternate routes were selected and installed after a failure.

### Syslog Messages

Depending on the failure mechanism, IOS XE can generate BGP neighbor reset and BFD session state-change messages.

Examine these messages to determine whether the session went down because of:

- Hold Timer expiration.
- Direct interface failure.
- Neighbor reachability loss.
- BFD failure.
- Administrative reset.

The BGP session uptime can also help verify whether a hard reset occurred.

---

## Troubleshooting

| Symptom | Possible Cause | Solution |
|---|---|---|
| EBGP session does not immediately reset after forwarding failure | Local interface remains up, so Fast External Fallover is not triggered | Consider BFD |
| IBGP session stays Established after losing a preferred IGP path | An alternative route to the peer still exists | Verify peer reachability and expected fall-over behavior |
| Neighbor fall-over does not reset after losing a specific route | Less-specific route or default route still resolves the peer | Use selective address tracking if required |
| BGP session resets during IGP reconvergence | Neighbor reachability temporarily disappears | Review fall-over configuration and IGP stability |
| BFD session remains Down | Missing BFD configuration, reachability problem, or incompatible settings | Verify BFD on both routers |
| BGP session is Established but BFD is Down | Strict Mode is not configured, or BFD has not become operational | Verify BFD and determine whether Strict Mode is needed |
| BGP does not reset when BFD fails | BFD is not associated with the BGP neighbor | Check `neighbor ... fall-over bfd` |
| BFD causes frequent BGP resets | Aggressive timers, packet loss, or CPU congestion | Adjust BFD timers and investigate packet loss |
| Multihop BFD flaps during IGP convergence | Detection time shorter than IGP reconvergence | Increase BFD detection time |
| BGP session remains Established but routes become invalid | BGP NEXT_HOP is unreachable | Examine NHT and next-hop resolution |

---

## Command Summary

| Command | Purpose |
|---|---|
| `bgp fast-external-fallover` | Enable Fast External Fallover |
| `no bgp fast-external-fallover` | Disable global Fast External Fallover |
| `ip bgp fast-external-fallover` | Configure interface-specific Fast External Fallover |
| `neighbor <IP> fall-over` | Enable route-based fast session deactivation |
| `neighbor <IP> fall-over route-map <NAME>` | Enable selective neighbor address tracking |
| `neighbor <IP> fall-over bfd` | Enable BFD-based fast fall-over |
| `neighbor <IP> fall-over bfd multi-hop` | Enable multihop BFD-based fall-over |
| `neighbor <IP> fall-over bfd single-hop strict-mode` | Require BFD to be Up for BGP session establishment |
| `neighbor <IP> fall-over bfd check-control-plane-failure` | Use BFD C-bit information for control-plane failure handling |
| `bfd interval <TX> min_rx <RX> multiplier <N>` | Configure interface BFD timing |
| `bfd-template multi-hop <NAME>` | Configure multihop BFD template |
| `bfd map ipv4 <REMOTE> <LOCAL> <TEMPLATE>` | Associate multihop BFD endpoints with a template |
| `bgp nexthop trigger delay <SECONDS>` | Configure BGP NHT processing delay |
| `show bfd neighbors` | Display BFD session status |
| `show bfd neighbors details` | Display detailed BFD information |
| `show ip bgp neighbors <IP>` | Verify BGP neighbor and fall-over status |

Feature availability and configuration modes vary by platform and IOS XE release.

---

## Key Points

- BGP convergence involves failure detection, path selection, RIB installation, and FIB updates.
- The default BGP Hold Timer is generally 180 seconds on Cisco IOS XE.
- **Fast External Fallover** is enabled by default and resets directly connected EBGP sessions when the underlying interface goes down.
- **`neighbor ... fall-over`** uses routing information to monitor reachability to the BGP peer address.
- Plain neighbor fall-over does not require BFD.
- **`neighbor ... fall-over route-map`** applies selective address tracking to the route used to reach the peer.
- Selective address tracking supports `match ip address` and `match source-protocol`.
- **`neighbor ... fall-over bfd`** uses BFD forwarding-path failure detection to trigger a BGP session reset.
- BFD is independent of BGP KEEPALIVE and Hold Timers.
- BFD timers are negotiated between peers.
- Configuring interface BFD parameters alone does not necessarily establish a BFD session. BGP must register as a BFD client.
- **Multihop BFD** supports monitoring BGP sessions between non-directly connected routers.
- Multihop BFD detection times should account for expected IGP reconvergence.
- **BFD Strict Mode** prevents a BGP session from becoming Established while its BFD session is Down.
- BGP NHT monitors reachability to the NEXT_HOP attributes of BGP routes, which may differ from BGP peer addresses.
- A BGP session can remain Established even when an individual BGP route becomes unusable because its NEXT_HOP is unreachable.
- **Fast failure detection does not automatically guarantee fast end-to-end convergence.**

---

## References

- [Cisco: BGP Support for BFD (IOS XE 17.x)](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-routing/b-ip-routing/m_irg-sup-for-bfd.html)
- [Cisco: Configuring BGP Neighbor Session Options (IOS XE 17.x)](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-routing/b-ip-routing/m_irg-neighbor-0.html)
- [Cisco: BGP Support for Next-Hop Address Tracking (IOS XE 17.x)](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-routing/b-ip-routing/m_irg-nexthop-track.html)
- [Cisco: BFD-BGP Multihop Client Support, C-bit, and Strict Mode (IOS XE 17.x)](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-routing/b-ip-routing/m_irg-multihop-cbit.html)
- [Cisco: BGP Command Reference, neighbor fall-over](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_bgp/command/irg-cr-book/bgp-m1.html)
- [Cisco: BGP Command Reference, bgp fast-external-fallover](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_bgp/command/irg-cr-book/bgp-a1.html)
- [RFC 5880: Bidirectional Forwarding Detection (BFD)](https://www.rfc-editor.org/rfc/rfc5880)
- [RFC 5881: BFD for IPv4 and IPv6 (Single Hop)](https://www.rfc-editor.org/rfc/rfc5881)
- [RFC 5882: Generic Application of BFD](https://www.rfc-editor.org/rfc/rfc5882)
- [RFC 5883: BFD for Multihop Paths](https://www.rfc-editor.org/rfc/rfc5883)
