# 1.5.a (iii) Timers

BGP uses several **timers** to maintain neighbor sessions, detect failures, and control route advertisements.

The two primary session timers are:

- **Keepalive Timer:** Controls how frequently a router sends BGP KEEPALIVE messages.
- **Hold Timer:** Determines how long a router waits without receiving BGP messages before declaring the neighbor down.

Cisco IOS XE uses the following defaults:

| Timer | Default |
|---|---|
| Keepalive | 60 seconds |
| Hold | 180 seconds |

The keepalive interval is normally approximately **one-third of the hold time**.

---

## BGP Keepalive Timer

The **Keepalive Timer** determines how frequently BGP sends KEEPALIVE messages to maintain a neighbor relationship.

KEEPALIVE messages contain no routing information. They consist of only the **19-byte BGP message header**.

By default, Cisco IOS XE sends KEEPALIVE messages every 60 seconds when no other BGP messages are being exchanged.

```text
          R1                           R2
           |                            |
           |-------- KEEPALIVE -------->|
           |                            |
           |         60 seconds         |
           |                            |
           |-------- KEEPALIVE -------->|
           |                            |
           |         60 seconds         |
           |                            |
           |-------- KEEPALIVE -------->|
```

BGP does not need to send a separate KEEPALIVE if other BGP messages are already providing the necessary session activity.

---

## BGP Hold Timer

The **Hold Timer** determines how long a router can go without receiving a KEEPALIVE or UPDATE message from its neighbor.

By default, the hold time is **180 seconds**.

Every time a router receives a valid KEEPALIVE or UPDATE message, its Hold Timer is reset.

For example:

```text
            R1                           R2
             |                            |
             |<------- KEEPALIVE ---------|
             |                            |
             |    Hold Timer = 180 sec    |
             |                            |
             |      No more messages      |
             |                            |
             |       180 sec elapsed      |
             |                            |
             |    Hold Timer expires      |
             |                            |
             |------- NOTIFICATION ------>|
             |                            |
             |     TCP connection closed  |
```

When the Hold Timer expires, the router:

1. Sends a BGP NOTIFICATION message with the **Hold Timer Expired** error.
2. Terminates the BGP session.
3. Removes the neighbor's routes from normal BGP consideration, subject to features such as Graceful Restart.
4. Attempts to reestablish the session according to its BGP state machine.

**The Hold Timer is reset by received messages, not by messages transmitted by the local router.**

---

## BGP Hold Time Negotiation

When two BGP peers establish a session, they exchange their configured hold times in **BGP OPEN messages**.

The negotiated hold time is the **lower of the two advertised values**.

```text
Negotiated Hold Time = MIN(Local Hold Time, Remote Hold Time)
```

For example:

```text
              R1                           R2
       Hold Time: 90 sec            Hold Time: 180 sec
               |                           |
               |-----==--- OPEN ---------->|
               |        Hold = 90          |
               |                           |
               |<--------- OPEN -----------|
               |        Hold = 180         |
               |                           |
               |    Negotiated Hold = 90   |
```

Both routers use a negotiated hold time of **90 seconds**.

The configured keepalive intervals do not need to match.

Each router determines its effective keepalive interval using its local configuration and the negotiated hold time.

On Cisco IOS XE, the keepalive interval is adjusted when necessary so it does not exceed approximately one-third of the negotiated hold time.

For example:

| Parameter | R1 | R2 |
|---|---:|---:|
| Configured Keepalive | 10 sec | 60 sec |
| Configured Hold Time | 30 sec | 180 sec |
| Negotiated Hold Time | 30 sec | 30 sec |
| Effective Keepalive | 10 sec | 10 sec |

R2 reduces its keepalive interval because the negotiated hold time is shorter than its configured hold time.

**BGP peers do not need identical timer configurations to establish a session.**

---

## Configuring BGP Timers

### Process-Wide Timers

Use `timers bgp` to configure the default keepalive and hold times for the BGP process.

Syntax:

```text
R1(config-router)# timers bgp <keepalive> <holdtime> [min-holdtime]
```

For example:

```text
R1(config)# router bgp 65001
R1(config-router)# timers bgp 10 30
```

This configures:

- Keepalive: 10 seconds
- Hold Time: 30 seconds

These values apply to neighbors without more specific timer configurations.

Restore the defaults:

```text
R1(config-router)# no timers bgp
```

### Neighbor-Specific Timers

Use `neighbor timers` to configure timers for an individual BGP neighbor.

```text
R1(config)# router bgp 65001
R1(config-router)# neighbor 10.0.12.2 remote-as 65002
R1(config-router)# neighbor 10.0.12.2 timers 5 15
```

This configures the following timers for neighbor `10.0.12.2`:

- Keepalive: 5 seconds
- Hold Time: 15 seconds

**Neighbor-specific timers override process-wide timers.**

For example:

```text
R1(config)# router bgp 65001
R1(config-router)# timers bgp 30 90
R1(config-router)# neighbor 10.0.12.2 timers 5 15
```

| Neighbor | Keepalive | Hold Time |
|---|---:|---:|
| 10.0.12.2 | 5 sec | 15 sec |
| Other neighbors without overrides | 30 sec | 90 sec |

These are the locally configured timer values. Actual hold times are negotiated separately with each peer.

Remove a neighbor-specific timer configuration:

```text
R1(config-router)# no neighbor 10.0.12.2 timers
```

The neighbor then uses the applicable process-wide timer configuration.

---

## Minimum Acceptable Hold Time

Cisco IOS XE supports an optional third argument:

```text
R1(config-router)# neighbor <IP> timers <keepalive> <holdtime> <min-holdtime>
```

The **minimum acceptable hold time** specifies the lowest hold time that the remote neighbor is permitted to advertise.

For example:

```text
R1(config)# router bgp 65001
R1(config-router)# neighbor 10.0.12.2 timers 30 90 45
```

This configures:

| Parameter | Value |
|---|---:|
| Keepalive | 30 sec |
| Local advertised Hold Time | 90 sec |
| Minimum acceptable remote Hold Time | 45 sec |

R1 rejects a BGP session if R2 advertises a hold time lower than 45 seconds.

For example:

| R2 advertised Hold Time | Result |
|---|---|
| 180 sec | Accepted, negotiated to 90 sec |
| 90 sec | Accepted, negotiated to 90 sec |
| 60 sec | Accepted, negotiated to 60 sec |
| 45 sec | Accepted, negotiated to 45 sec |
| 30 sec | Rejected |

If the remote hold time is below the minimum, the local router sends a NOTIFICATION indicating an unacceptable hold time.

**The minimum acceptable hold time is not the negotiated hold time.** It is a lower bound used to decide whether to accept the remote peer's proposed hold time.

The minimum acceptable value must not exceed the locally configured hold time.

---

## Hold Time of Zero

BGP supports a negotiated hold time of **0 seconds**.

A hold time of zero disables the normal BGP Hold Timer mechanism.

According to RFC 4271:

- The negotiated hold time must be **0 or at least 3 seconds**.
- Hold times of 1 or 2 seconds are invalid.
- When the negotiated hold time is zero, periodic KEEPALIVE messages are disabled.

For example:

```text
R1(config)# router bgp 65001
R1(config-router)# neighbor 10.0.12.2 timers 0 0
```

This configures zero keepalive and hold-time values for the neighbor.

**Disabling the Hold Timer removes BGP's normal message-based peer failure detection.**

The session may still terminate because of TCP failures, administrative actions, or other mechanisms.

A hold time of zero is generally undesirable for ordinary BGP peering.

---

## BGP Advertisement Interval

BGP also uses a **Minimum Route Advertisement Interval (MRAI)** to limit how frequently routing information is advertised to a neighbor.

Unlike the Keepalive and Hold Timers, this controls **UPDATE advertisements**, not peer liveliness.

Configure:

```text
R1(config-router)# neighbor <IP> advertisement-interval <seconds>
```

For example:

```text
R1(config)# router bgp 65001
R1(config-router)# neighbor 10.0.12.2 advertisement-interval 10
```

This configures a 10-second minimum advertisement interval.

The purpose is to reduce excessive routing updates when routes change repeatedly.

The advertisement interval is not a periodic full-table refresh timer. BGP sends UPDATEs when routing information changes, subject to advertisement scheduling.

### Default Advertisement Intervals

Cisco IOS XE command references document the following common defaults:

| Neighbor type | Default |
|---|---:|
| EBGP (global routing table) | 30 seconds |
| EBGP (VRF) | 0 seconds |
| IBGP | 0 seconds |

Defaults can vary by software release and platform. Some Cisco documentation lists 5 seconds for IBGP.

An interval of **0 seconds** means updates are eligible to be sent immediately rather than being held by the minimum advertisement interval. Other BGP processing delays can still apply.

The advertisement interval does not control BGP KEEPALIVE messages.

> See [BGP Route Advertisement and Network Statements](../route-advertisement.md) for general route advertisement behavior.

---

## Other BGP Timers

BGP has additional timers that affect session establishment and convergence.

| Timer | Purpose |
|---|---|
| ConnectRetry | Controls retries after unsuccessful TCP connection attempts |
| Advertisement Interval (MRAI) | Limits repeated route advertisements |
| BGP Scanner | Periodically examines BGP routing information, including next-hop validity |
| Next-Hop Tracking Delay | Controls event-driven processing following changes to BGP next-hop reachability |

### ConnectRetry Timer

The **ConnectRetry Timer** is part of the BGP finite state machine.

It controls when BGP retries connection establishment after unsuccessful attempts.

RFC 4271 suggests a default of **120 seconds**, although actual retry behavior depends on the implementation.

The ConnectRetry Timer should not be confused with the Hold Timer:

- **ConnectRetry:** Used while trying to establish a BGP session.
- **Hold Timer:** Used to detect missing BGP messages in an established session.

See [BGP Messages and Neighbor States](../messages-states.md) for FSM operation.

### BGP Scanner

Cisco IOS XE uses a periodic BGP scanner, historically with a default interval of **60 seconds**.

On supported platforms, the scanner interval can be configured with:

```text
R1(config-router)# bgp scan-time <seconds>
```

However, modern IOS XE platforms also use **BGP Next-Hop Tracking (NHT)** to react to changes in next-hop reachability.

NHT can provide faster event-driven processing without waiting for the next periodic scan.

The scanner's behavior and configuration restrictions depend on the IOS XE release and whether NHT is enabled.

See [BGP Next-Hop Handling](next-hop.md).

---

## BGP Timers and Fast Failure Detection

Reducing the BGP Hold Timer can improve failure detection when a neighbor stops sending messages.

However, aggressive timers can also cause unnecessary session resets during:

- CPU congestion
- Control-plane overload
- Temporary packet loss
- Network instability

Cisco IOS XE warns when configuring a hold time below 20 seconds because of the increased risk of session flapping.

Other mechanisms can detect failures without relying exclusively on the Hold Timer.

| Mechanism | Purpose |
|---|---|
| BGP Hold Timer | Detects loss of received BGP messages |
| Fast External Fallover | Resets directly connected EBGP sessions when the relevant interface fails |
| BGP Next-Hop Tracking | Reacts to changes in route or next-hop reachability |
| BFD | Provides rapid failure detection independent of normal BGP KEEPALIVE intervals |

For example, IOS XE enables **BGP fast external fallover** by default on supported platforms:

```text
R1(config)# router bgp 65001
R1(config-router)# bgp fast-external-fallover
```

This allows directly connected EBGP sessions to be reset when the underlying interface fails, without waiting for the Hold Timer.

See [BGP Fast Convergence and BFD](../other-features/fast-convergence.md) for additional details.

---

## Verification

### Display BGP Neighbor Timers

```text
R1# show ip bgp neighbors 10.0.12.2
```

Look for output similar to:

```text
BGP neighbor is 10.0.12.2, remote AS 65002, external link
  BGP state = Established
  Last read 00:00:04
  hold time is 30, keepalive interval is 10 seconds
```

The important fields are:

| Field | Meaning |
|---|---|
| `BGP state` | Current neighbor state |
| `Last read` | Time since the last received BGP message |
| `hold time` | Negotiated Hold Timer |
| `keepalive interval` | Effective local KEEPALIVE transmission interval |

### Display Configured Timers

```text
R1# show running-config | section router bgp
```

Example:

```text
router bgp 65001
 timers bgp 30 90
 neighbor 10.0.12.2 remote-as 65002
 neighbor 10.0.12.2 timers 10 30
```

The neighbor-specific configuration overrides the process-wide timers.

### Troubleshoot Timer Expiration

```text
R1# show ip bgp summary
R1# show ip bgp neighbors 10.0.12.2
R1# debug ip bgp events
```

If the Hold Timer expires, examine:

1. Whether KEEPALIVE or UPDATE messages are arriving.
2. Whether the negotiated hold time is unexpectedly short.
3. Whether there is packet loss or control-plane congestion.
4. Whether BGP sessions are repeatedly resetting.

Disable debugging afterward:

```text
R1# undebug all
```

---

## Command Summary

| Command | Purpose |
|---|---|
| `timers bgp <keepalive> <holdtime>` | Configure process-wide BGP timers |
| `neighbor <IP> timers <keepalive> <holdtime>` | Configure neighbor-specific timers |
| `neighbor <IP> timers <keepalive> <holdtime> <min-holdtime>` | Configure minimum acceptable remote hold time |
| `no timers bgp` | Restore process-wide timer defaults |
| `no neighbor <IP> timers` | Remove neighbor-specific timer configuration |
| `neighbor <IP> advertisement-interval <seconds>` | Configure minimum route advertisement interval |
| `bgp scan-time <seconds>` | Configure periodic BGP scanning on supported platforms |
| `show ip bgp neighbors <IP>` | Verify negotiated hold and effective keepalive timers |
| `show ip bgp summary` | Display BGP session status |

---

## Key Points

- Default BGP Keepalive Timer: **60 seconds**.
- Default BGP Hold Timer: **180 seconds**.
- The Hold Timer is reset when a router receives valid KEEPALIVE or UPDATE messages.
- BGP negotiates the Hold Timer using the **lower of the two advertised hold times**.
- Keepalive intervals are adjusted based on the negotiated hold time and local configuration.
- **Neighbor-specific timers override process-wide timers.**
- The optional `min-holdtime` rejects peers advertising a hold time below the configured minimum.
- A negotiated hold time of **0 disables the normal Hold Timer and periodic KEEPALIVE messages**.
- The **Advertisement Interval** regulates route UPDATE advertisements, not KEEPALIVE transmission.
- BFD, fast external fallover, and NHT can provide faster convergence than waiting for Hold Timer expiration.

---

## Cisco Documentation

- [Cisco IOS XE 17.x: Configuring BGP Neighbor Session Options](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-routing/b-ip-routing/m_irg-neighbor-0.html)
- [Cisco IOS XE: Configuring Internal BGP Features](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-routing/b-ip-routing/m_irg-int-features-0.html)
- [Cisco BGP Command Reference: neighbor timers and advertisement-interval](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_bgp/command/irg-cr-book/bgp-m1.html)
- [Cisco BGP Command Reference: timers bgp](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_bgp/command/irg-cr-book/bgp-s1.html)
- [RFC 4271: BGP-4](https://www.rfc-editor.org/rfc/rfc4271.html)
