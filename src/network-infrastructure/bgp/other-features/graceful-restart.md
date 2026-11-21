# BGP Graceful Restart

**BGP Graceful Restart (GR)** allows a router to temporarily preserve routes learned from a BGP neighbor while that neighbor's control plane restarts.

Normally, when a BGP session goes down, the router removes the routes learned through that neighbor and recalculates its best paths.

However, a control-plane restart does not necessarily mean the router's forwarding plane has stopped working.

If the restarting router can continue forwarding packets, immediately removing its BGP routes may cause unnecessary traffic disruption.

Graceful Restart allows neighboring routers to retain eligible routes temporarily while the BGP session is reestablished.

**The goal of BGP Graceful Restart is to minimize forwarding disruption during a control-plane restart.**

---

## Graceful Restart vs. Nonstop Forwarding

Graceful Restart is closely associated with **Nonstop Forwarding (NSF)**.

| Feature | Purpose |
|---|---|
| Graceful Restart (GR) | Allows BGP neighbors to retain stale routes during a control-plane restart |
| Nonstop Forwarding (NSF) | Allows a router to continue forwarding packets while its control plane restarts |
| Stateful Switchover (SSO) | Allows a redundant control plane to take over when the active control plane fails |

Two important roles exist in a BGP Graceful Restart operation.

### Restarting Router (NSF-Capable)

The restarting router experiences a BGP control-plane restart.

If the platform supports NSF, its forwarding plane can continue forwarding packets using previously installed forwarding information.

The router reestablishes its BGP sessions and exchanges routing information with its neighbors.

### Helper Router (NSF-Aware)

The helper router detects that its BGP neighbor is restarting.

Instead of immediately removing the neighbor's routes, it temporarily marks eligible routes as **stale** and retains them.

The helper continues using these routes while the restarting router recovers, subject to Graceful Restart conditions and timers.

**Important:** Supporting Graceful Restart as a helper does not necessarily mean a router can preserve its own forwarding state during a control-plane restart.

---

## Graceful Restart Operation

Consider two EBGP neighbors:

```text
       AS 65001                    AS 65002

          R1 ------------------------ R2
       GR Helper                Restarting Router
```

Both routers have negotiated Graceful Restart capability.

R2 experiences a control-plane restart, but its forwarding plane continues operating.

The process is:

1. **Session loss:** R1 detects that its BGP session with R2 has gone down.
2. **Stale routes:** If the failure is eligible for Graceful Restart, R1 marks routes learned from R2 as stale instead of immediately removing them.
3. **Forwarding continues:** R1 can continue forwarding traffic using the stale routes while R2 restarts.
4. **Session reestablishment:** R2 reestablishes its BGP session with R1.
5. **Route readvertisement:** R2 advertises its current BGP routing information.
6. **End-of-RIB:** R2 sends an End-of-RIB marker to indicate that its initial routing updates for the address family are complete.
7. **Stale route removal:** R1 removes any remaining stale routes that were not refreshed.

If R2 fails to recover within the applicable Graceful Restart timers, R1 removes the stale routes.

### End-of-RIB (EoR) Marker

The **End-of-RIB (EoR)** marker signals that a restarting router has completed its initial routing information exchange for an address family.

The helper uses this information to identify stale routes that are no longer valid.

For example:

- R1 previously learned `192.0.2.0/24` and `198.51.100.0/24` from R2.
- R2 restarts.
- R1 temporarily marks both routes as stale.
- R2 reestablishes BGP but readvertises only `192.0.2.0/24`.
- R2 sends the End-of-RIB marker.
- R1 removes the remaining stale route to `198.51.100.0/24`.

**Important:** Graceful Restart does not mean stale routes are retained indefinitely.

---

## Graceful Restart Configuration

On Cisco IOS XE, configure BGP Graceful Restart using:

```text
R1(config)# router bgp 65001
R1(config-router)# bgp graceful-restart
```

Configure the corresponding capability on the neighbor:

```text
R2(config)# router bgp 65002
R2(config-router)# bgp graceful-restart
```

This enables the Graceful Restart capability and BGP NSF awareness.

Cisco IOS XE documents BGP Graceful Restart as **disabled by default** in its BGP NSF Awareness configuration guide.

Enabling the command does not automatically make a router capable of preserving its own forwarding state through a control-plane restart. That depends on the platform's high-availability capabilities.

### Per-Neighbor Configuration

Graceful Restart can also be enabled for an individual neighbor.

For example:

```text
R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 ha-mode graceful-restart
```

Disable Graceful Restart for a specific neighbor:

```text
R1(config-router-af)# neighbor 10.0.12.2 ha-mode graceful-restart disable
```

The neighbor-specific command allows different Graceful Restart settings for different BGP peers.

### Capability Negotiation

Graceful Restart capability is negotiated through BGP OPEN messages.

Therefore, if Graceful Restart is enabled after a session has already been established, the session must be reestablished for the new capability to be negotiated.

For example:

```text
R1# clear ip bgp 10.0.12.2
```

This performs a hard reset.

**A BGP soft reset does not renegotiate Graceful Restart capability.**

---

## Graceful Restart Timers

Cisco IOS XE provides two important Graceful Restart timers.

| Timer | Default | Purpose |
|---|---|---|
| Restart Timer | 120 seconds | Limits how long the helper waits for the restarting router to reestablish BGP |
| Stale Path Timer | 360 seconds | Limits retention of stale routing information during Graceful Restart recovery |

Configure the timers under the BGP process:

```text
R1(config)# router bgp 65001
R1(config-router)# bgp graceful-restart restart-time 120
R1(config-router)# bgp graceful-restart stalepath-time 360
```

Both timers use seconds.

The Restart Timer relates to the time allowed for the restarting BGP speaker to return.

The Stale Path Timer controls how long stale routes may be retained during the recovery process.

Stale routes can be removed earlier when Graceful Restart processing determines they are no longer valid.

**The default timer values are generally appropriate and do not need to be modified simply to enable Graceful Restart.**

---

## Graceful Restart and BFD

BGP Graceful Restart and **Bidirectional Forwarding Detection (BFD)** can conflict.

- Graceful Restart attempts to preserve routes when a control plane restarts.
- BFD detects failures quickly and normally causes BGP to remove routes when the BFD session goes Down.

However, BFD may itself be affected by a control-plane restart.

Cisco IOS XE supports:

```text
R1(config-router)# neighbor 10.0.12.2 fall-over bfd check-control-plane-failure
```

This allows BGP to consider the BFD **Control Plane Independent (C-bit)** information.

| BFD Control-Plane Independent? | `check-control-plane-failure` | BGP Action After BFD Down |
|---|---|---|
| Yes | Disabled | Purge routes |
| No | Disabled | Purge routes |
| Yes | Enabled | Purge routes |
| No | Enabled | Preserve eligible stale routes through NSF/GR |

This table assumes BGP Graceful Restart/NSF is applicable.

### Why Does This Matter?

If BFD is **control-plane independent**, a control-plane restart should not interrupt BFD.

Therefore, a BFD failure indicates a likely forwarding-path failure, so BGP removes the affected routes.

If BFD is **control-plane dependent**, a control-plane restart might interrupt BFD even while forwarding remains operational.

With `check-control-plane-failure` enabled, BGP can preserve stale routes rather than immediately removing them.

However, this carries a risk: a control-plane-dependent BFD failure might actually indicate a forwarding failure.

**The BFD C-bit describes the independence of the BFD implementation, not the actual cause of an individual failure.**

See [BGP Fast Convergence and BFD](fast-convergence.md) for further details.

---

## Verification

### Verify Graceful Restart Capability

```text
R1# show ip bgp neighbors 10.0.12.2
```

Look for the Graceful Restart capability in the neighbor details:

```text
Graceful Restart Capability: advertised and received
```

Also look for the configured timers:

```text
Graceful-Restart is enabled,
restart-time 120 seconds, stalepath-time 360 secs
```

The precise output varies by IOS XE version.

### Verify Stale Routes

```text
R1# show ip bgp
```

During Graceful Restart, Cisco IOS XE can mark stale BGP paths using the status code `S`.

### Useful Debug Commands

```text
R1# debug ip bgp
R1# debug ip bgp events
R1# debug ip bgp updates
```

These can help examine capability negotiation, Graceful Restart timers, and End-of-RIB messages.

Use debug commands carefully on production routers.

---

## Key Points

- **BGP Graceful Restart** allows eligible stale routes to be retained temporarily during a BGP control-plane restart.
- **NSF** allows forwarding to continue while a router's control plane restarts, if the platform supports it.
- The **restarting router** experiences the control-plane restart.
- The **helper router** retains stale routes learned from the restarting neighbor.
- Graceful Restart capability is negotiated through BGP OPEN messages.
- Configure Graceful Restart globally with `bgp graceful-restart`.
- Configure it per neighbor with `neighbor ... ha-mode graceful-restart`.
- The default Restart Timer is **120 seconds**.
- The default Stale Path Timer is **360 seconds**.
- The **End-of-RIB marker** helps the helper identify stale routes that were not readvertised.
- Graceful Restart does not guarantee that forwarding continues after every BGP session failure.
- By default, BFD failure causes BGP to purge the neighbor's routes.
- `check-control-plane-failure` allows BGP to consider the BFD C-bit before deciding whether to retain eligible stale routes during Graceful Restart.

---

## References

- [Cisco: BGP NSF Awareness, IOS XE 17.x](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-routing/b-ip-routing/m_irg-nsf-awareness.html)
- [Cisco: BGP Graceful Restart per Neighbor, IOS XE 17.x](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-routing/b-ip-routing/m_irg-grace-restart-neighbor.html)
- [Cisco: BFD, BGP Multihop Client Support, cBit, and Strict Mode, IOS XE 17.x](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-routing/b-ip-routing/m_irg-multihop-cbit.html)
- [RFC 4724: Graceful Restart Mechanism for BGP](https://www.rfc-editor.org/rfc/rfc4724)
- [RFC 5882: Generic Application of BFD](https://www.rfc-editor.org/rfc/rfc5882)
