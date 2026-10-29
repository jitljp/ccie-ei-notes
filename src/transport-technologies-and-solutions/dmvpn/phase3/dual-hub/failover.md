# Dual-Hub Failover and Convergence

**Dual-hub DMVPN Phase 3** provides alternate hub paths, but the presence of two NHRP Next Hop Servers (NHSs) does not by itself guarantee uninterrupted forwarding. Recovery requires the **underlay**, **NHRP**, **overlay routing protocol**, and **CEF** to reach a usable state. Existing spoke-to-spoke shortcuts add another forwarding path whose behavior must be considered separately.

This page uses the **single-cloud** and **dual-cloud** EIGRP labs from [Dual-Hub Phase 3 Configuration](configuration.md). It focuses on the order of failure detection and recovery, rather than repeating the complete configurations. The fundamental designs are compared in [Dual-Hub DMVPN Topologies](topologies.md), and NHRP shortcut invalidation is covered in [NHRP Redirects and Shortcuts](../nhrp/redirect-shortcut.md). IPsec/IKEv2 recovery is addressed in the subsequent IPsec section.

---

## Reference Topology

The same five DMVPN routers connect through the ISP underlay:

```text
                       Hub1               Hub2
                        |                   |
                        +---------+---------+
                                  |
                          DMVPN connectivity
                            /     |     \
                        Spoke1 Spoke2 Spoke3
```

The diagram represents logical DMVPN connectivity. Each router has its own physical ISP-facing connection; the spokes do not connect to a single shared physical link between the hubs.

| Router | WAN / NBMA address | Site LAN / Loopback0 |
| --- | --- | --- |
| Hub1 | `198.51.100.1` | `10.1.1.0/24` |
| Hub2 | `198.51.100.5` | `10.2.2.0/24` |
| Spoke1 | `198.51.100.9` | `10.11.11.0/24` |
| Spoke2 | `198.51.100.13` | `10.12.12.0/24` |
| Spoke3 | `198.51.100.17` | `10.13.13.0/24` |

### Single-Cloud Baseline

```text
                    DMVPN cloud 100
                     172.16.0.0/24

             Hub1                         Hub2
          Tunnel0 .1                   Tunnel0 .2
                \                         /
                 \                       /
                  +------ Cloud 100 -----+
                         /    |    \
                      Spoke1 Spoke2 Spoke3
                       .11    .12    .13
```

- All five devices use `Tunnel0`, NHRP network ID `100`, and GRE key `100`.
- Every spoke has **two NHSs**: Hub1 (`172.16.0.1`) and Hub2 (`172.16.0.2`).
- Hub2 registers with Hub1 to obtain hub-to-hub mGRE reachability, but **Hub2 is independently an NHS for the spokes**.
- The hubs can form an EIGRP adjacency with one another over the shared tunnel.
- Both hub adjacencies on a spoke use the **same interface**, which limits per-hub metric tuning through `bandwidth` and `delay`.

### Dual-Cloud Baseline

```text
               Cloud 100                    Cloud 200
             172.16.0.0/24                172.16.1.0/24

                Hub1                          Hub2
            Tunnel0 .1                    Tunnel1 .2
               /|\                           /|\
              / | \                         / | \
         Spoke1 Spoke2 Spoke3         Spoke1 Spoke2 Spoke3
          T0 .11 T0 .12 T0 .13        T1 .11 T1 .12 T1 .13

             Each spoke participates in both clouds.
```

| Property | Cloud 100 | Cloud 200 |
| --- | --- | --- |
| NHS | Hub1 `172.16.0.1` | Hub2 `172.16.1.2` |
| Spoke interface | `Tunnel0` | `Tunnel1` |
| NHRP network ID | `100` | `200` |
| GRE key | `100` | `200` |
| Spoke tunnel delay | `1000` | `5000` |
| Intended preference | Primary, when eligible | Backup, when eligible |

The lab uses EIGRP AS `100` on both tunnels. **The configured delay only influences EIGRP's metric; it does not measure actual network latency or constitute a failure detector.** Verify that the resulting EIGRP route really prefers Cloud 100 before starting a failover test.

---

## Understand What Must Converge

A failed hub can affect several different pieces of state:

| Plane or component | What can change | Useful verification |
| --- | --- | --- |
| Underlay | Reachability to a hub's NBMA address | `ping` with WAN source, `show ip route` |
| GRE/mGRE | Local tunnel interface state; outer encapsulation reachability | `show interfaces Tunnel0`, `show ip interface brief` |
| NHRP NHS | Registration and hub reachability state | `show ip nhrp nhs`, `show dmvpn` |
| EIGRP | Neighbor state, topology entries, successors | `show ip eigrp neighbors`, `show ip eigrp topology` |
| RIB | Installed route and its ordinary next hop | `show ip route <destination>` |
| NHRP shortcut | Cached direct peer mapping, `H` route, or next-hop override | `show ip nhrp shortcut`, `show ip route next-hop-override` |
| CEF | Actual outgoing tunnel and forwarding next hop | `show ip cef <destination> detail` |
| Data plane | Reachability and path used by real packets | Extended `ping`, `traceroute` |

**NHRP registration, EIGRP adjacency, and working forwarding are separate conditions.** An NHS can be reachable while an EIGRP neighbor is down; EIGRP can be up while a redirect or shortcut is not working; a local mGRE interface can remain up while the remote hub is unreachable.

### Failure Detection Is Not the Same as Recovery

A useful convergence model is:

1. **Detect failure.** A directly attached interface, NHRP peer, EIGRP neighbor, or security session changes state.
2. **Select a valid route.** EIGRP removes an unusable successor and selects another feasible path or performs a DUAL computation.
3. **Update forwarding.** The RIB and CEF reflect the new route; NHRP shortcut state is retained, revalidated, or removed as appropriate.
4. **Resolve a direct path again if needed.** When traffic initially follows a hub path, Phase 3 can generate a new Redirect and Resolution exchange.
5. **Restore return-path connectivity.** The destination side must have a valid path back; routing and shortcuts can be asymmetric.

The total interruption depends on which of these steps are actually necessary. **There is no single universal DMVPN failover timer.**

---

## Failure Detection and Relevant Timers

### Physical Interface Failure

If a router detects that its own `Ethernet0/0` has gone down, its GRE source is no longer usable. Local routing and tunnel state can react promptly to that directly detected condition.

If **Hub1's** Ethernet0/0 goes down, however, **Spoke1's** Ethernet0/0 remains up. Spoke1 must detect the remote failure through control-plane or reachability mechanisms. Its tunnel interface should not be assumed to drop immediately.

### EIGRP Hello and Hold Timers

EIGRP uses Hellos to maintain neighbors. If a remote hub stops sending valid Hellos and the neighbor's hold timer expires, the adjacency is removed and its routes must be reconsidered.

On a spoke:

```cisco
show ip eigrp neighbors
show ip protocols
show ip eigrp interfaces detail
```

Configured timers are interface-specific. In a **single cloud**, changing timers on `Tunnel0` affects the hub relationships using that interface, rather than selectively changing Hub1 without affecting Hub2.

### NHRP Registration and NHS State

Each spoke uses NHRP to register its tunnel/NBMA information with the configured NHSs. When a hub becomes unreachable, NHRP's attempts to communicate and refresh registration may fail, and the NHS state can change.

```cisco
show ip nhrp nhs
show dmvpn
show ip nhrp traffic
```

In the single-cloud lab, the loss of Hub1 need not remove Hub2 from Spoke1's NHS list. In the dual-cloud lab, NHS reachability is maintained independently on `Tunnel0` and `Tunnel1`.

### Why an mGRE Tunnel Can Stay Up

mGRE does not have one configured `tunnel destination` whose reachability determines the status of all its peers. A tunnel can report up/up while some or all remote NHSs are unusable.

**Do not use `show ip interface brief` as the sole hub-failure test.** Check the actual NHS states and overlay neighbors.

### Optional: NHRP Interface State Control

Cisco provides an optional feature that ties a spoke tunnel's protocol state to NHS reachability:

```cisco
interface Tunnel0
 if-state nhrp
```

With this feature, a spoke tunnel can be declared down when **all configured NHSs on that interface are down**, and brought up when at least one NHS is available. The resulting interface-state change can trigger routing-protocol updates. In the baseline single-cloud design, the loss of **one** of two working NHSs does not meet the all-NHSs-down condition.

In the dual-cloud design, each spoke tunnel has its **own NHS**. If Cloud 100's NHS is lost, `if-state nhrp` on `Tunnel0` can influence that tunnel's operational state without intentionally taking down `Tunnel1`.

Apply this only when the feature is supported and its effect is understood. The examples in [Dual-Hub Phase 3 Configuration](configuration.md) do **not** require it for EIGRP to detect a failed adjacency.

---

## Single-Cloud Dual-Hub Failover

### Normal Operation

Both hub interfaces and all spokes share `172.16.0.0/24` on `Tunnel0`. Spoke1 has an NHS registration toward Hub1 and Hub2, and normally has EIGRP adjacencies with both hubs.

```text
                            Hub1        Hub2
                        172.16.0.1   172.16.0.2
                               \       /
                                Spoke1
                              172.16.0.11
```

The two adjacencies do **not** guarantee that every destination has two equal-cost routes. EIGRP chooses successors based on the complete computed metric and available topology. Hub1 and Hub2 can advertise different metrics or even different reachable prefixes.

For the destination LAN `10.12.12.0/24`, first determine whether Hub1, Hub2, or both are selected before any NHRP shortcut is created.

### Test 1: Hub1 Stops Participating in DMVPN

A controlled lab test is to shut down Hub1's tunnel interface:

```cisco
Hub1(config)# interface Tunnel0
Hub1(config-if)# shutdown
```

Observe Spoke1:

```cisco
show ip nhrp nhs
show ip eigrp neighbors
show ip eigrp topology
show ip route 10.12.12.12
show ip nhrp shortcut
show ip cef 10.12.12.12 detail
```

Expected control-plane sequence:

1. Spoke1 eventually determines that its Hub1 EIGRP adjacency is no longer usable.
2. EIGRP removes Hub1-dependent paths and selects another eligible route. Routes from Hub2 remain available if its adjacency and advertisements are intact.
3. Hub2 continues acting as an NHS for spokes that register with it.
4. If the new overlay route forwards traffic through Hub2, Hub2 can send Phase 3 Redirects for traffic it forwards between spokes on its tunnel interface.
5. Spoke1 can install a direct spoke-to-spoke shortcut after a successful Resolution exchange.

The point of the test is to **verify** those transitions. If the destination already had a direct shortcut before Hub1 failed, its continued validity depends on route tracking and the remaining forwarding state; it is not guaranteed to persist or disappear solely because Hub1 went down.

Restore Hub1:

```cisco
Hub1(config)# interface Tunnel0
Hub1(config-if)# no shutdown
```

### Important: Hub2 Registering with Hub1 Does Not Make Hub2 a Standby-Only NHS

In the preceding configuration chapter, Hub2 includes:

```cisco
interface Tunnel0
 ip nhrp nhs 172.16.0.1 nbma 198.51.100.1 multicast
```

This makes Hub2 an **NHRP client of Hub1**, primarily to establish hub-to-hub mGRE reachability. Hub2 also accepts registrations from the spokes as their NHS.

If Hub1 fails, Hub2 may lose its own registration to Hub1, but **Hub2 does not stop serving its registered spokes merely because its upstream NHS is down**. Do not confuse Hub2's client relationship with its server role.

### Test 2: Partial Hub1 Reachability Failure

A more realistic failure may affect only one spoke's ability to reach Hub1's NBMA address, while other spokes still reach Hub1.

For example, use the ISP to deny traffic between Spoke1's WAN address `198.51.100.9` and Hub1's `198.51.100.1`, while permitting Spoke1-to-Hub2 connectivity. Apply the restriction in the **underlay**, not with an overlay routing-protocol filter.

Possible outcomes:

- Spoke1 loses its Hub1 EIGRP adjacency and/or registration while other spokes retain theirs.
- Spoke1 selects a Hub2 route if one is eligible.
- Spoke2 may continue using a route through Hub1, so traffic in the two directions can initially take different paths.
- Direct Spoke1-to-Spoke2 GRE forwarding may still succeed if their NBMA addresses remain mutually reachable.

This proves why testing only a complete hub shutdown is insufficient. **Failover must handle partial and asymmetric failures as well.**

### Single-Cloud Routing Preference Limitations

Both EIGRP hub peers are reached through **Spoke1's same `Tunnel0` interface**. The following commands cannot make one hub more preferred than the other on that spoke:

```cisco
interface Tunnel0
 bandwidth 100000
 delay 1000
```

They change the spoke interface's metric contribution for both peers.

To influence hub preference in a single cloud, use appropriate **routing-protocol mechanisms**, such as metrics advertised by each hub or route filtering, and verify the actual installed successor. Do not equate **NHRP NHS order or priority** with EIGRP's chosen data-plane next hop.

### Hub-to-Hub Adjacency Failure

The single-cloud hubs form an overlay EIGRP adjacency because Hub2 registers with Hub1. Losing that adjacency does not necessarily mean spokes have lost their individual hub adjacencies.

If the hubs provide distinct LANs that are exchanged only through their tunnel adjacency, losing it can isolate those hub networks even while the spokes remain registered with both NHSs. This is a **route-exchange failure**, not automatically a failure of all spoke-to-hub tunnels.

---

## Dual-Cloud Dual-Hub Failover

### Normal Active/Standby Routing

Each spoke is simultaneously attached to the two independent clouds:

```text
                      Primary                      Backup
                    Cloud 100                    Cloud 200
                       Hub1                         Hub2
                  172.16.0.1                   172.16.1.2
                       |                            |
                    Tunnel0                      Tunnel1
                         \                         /
                          \                       /
                                Spoke1
                        172.16.0.11 / 172.16.1.11
```

The baseline configuration uses:

```cisco
interface Tunnel0
 bandwidth 100000
 delay 1000
!
interface Tunnel1
 bandwidth 100000
 delay 5000
```

Assuming the learned EIGRP routes remain comparable, Spoke1 should prefer **Hub1 over Tunnel0** for remote LAN traffic while retaining a path through Hub2 over `Tunnel1`.

Inspect the actual successor and backup eligibility:

```cisco
Spoke1# show ip eigrp neighbors
Spoke1# show ip eigrp topology
Spoke1# show ip route 10.12.12.12
Spoke1# show ip cef 10.12.12.12 detail
```

An alternate entry in the EIGRP topology table is **not necessarily a feasible successor**. If a valid feasible successor exists, EIGRP can select it quickly. If not, EIGRP may need to query and calculate a new successor. EIGRP stub configuration helps limit unnecessary query propagation but does not eliminate every possible convergence delay.

### Test 3: Hub1 Tunnel Failure

On Hub1:

```cisco
Hub1(config)# interface Tunnel0
Hub1(config-if)# shutdown
```

Spoke1's `Tunnel0` may still report up/up because it is mGRE, but the EIGRP relationship with Hub1 should eventually fail. Cloud 200 should remain operational on `Tunnel1`.

The desired progression is:

```text
Before failure:
  Route toward 10.12.12.0/24 -> Hub1 -> Tunnel0

After Hub1 is lost and EIGRP converges:
  Route toward 10.12.12.0/24 -> Hub2 -> Tunnel1

After a new Phase 3 shortcut is formed in Cloud 200:
  Forward directly to Spoke2 via Tunnel1
```

Once a valid Cloud 200 shortcut exists, CEF should resolve the destination through **`Tunnel1` and Spoke2's Cloud 200 tunnel address `172.16.1.12`**, rather than using the failed Cloud 100 path.

### Test 4: Spoke1 Loses Only Cloud 100

On Spoke1, locally shut down the primary tunnel:

```cisco
Spoke1(config)# interface Tunnel0
Spoke1(config-if)# shutdown
```

This removes **Spoke1's** primary overlay interface immediately while leaving the shared WAN underlay and `Tunnel1` intact. Spoke1 can continue registering with Hub2 and exchange EIGRP routes through Cloud 200.

Restore it:

```cisco
Spoke1(config)# interface Tunnel0
Spoke1(config-if)# no shutdown
```

This is an intentionally simple test of local interface failover. Compare it with a remote NHS outage: shutting down the local tunnel gives routing an immediate interface-state event, while a remote outage may require a neighbor hold timer or other detection mechanism.

### Test 5: Spoke2 Loses Cloud 100 While Spoke1 Does Not

Shut down Spoke2's primary tunnel:

```cisco
Spoke2(config)# interface Tunnel0
Spoke2(config-if)# shutdown
```

Spoke1 still reaches Hub1, but **Hub1 can no longer use Cloud 100 to reach Spoke2**. Spoke2 continues advertising its loopback through Cloud 200 if its EIGRP adjacency to Hub2 is intact.

This is different from the failure of Hub1 itself:

- Cloud 100 may remain operational for Spoke1 and Spoke3.
- Spoke1 must obtain a valid alternate route to Spoke2's prefix through Cloud 200 or another appropriately routed path.
- Existing direct Cloud 100 shortcuts to Spoke2 may need to be invalidated or replaced.
- **Do not assume** the remaining reachability to Hub1 implies reachability to every spoke behind Hub1.

A separate inter-hub core link, if configured, can change which routes Hub1 learns and forwards; verify whether a path through both hubs is intended. Avoid using a spoke as an accidental transit router between the clouds.

Restore Spoke2's Tunnel0 after the test.

### Test 6: Spoke WAN Interface Failure

If Spoke1's single `Ethernet0/0` WAN connection fails, **both `Tunnel0` and `Tunnel1` lose their common underlay source**. Two DMVPN clouds over one physical WAN link do not protect against failure of that link.

A true dual-transport design requires independent underlay paths (for example, different WAN interfaces and service providers) or other underlay redundancy. Multiple logical GRE tunnels alone provide **hub and overlay redundancy**, not physical transport redundancy.

---

## NHRP Shortcuts During Failover

### Base Routing and Shortcut Forwarding Are Different

In Phase 3, a spoke initially has an ordinary EIGRP route toward the selected hub. A successful NHRP Resolution can then install:

- A **more-specific `H` route** that refines a broader EIGRP covering route; or
- A **next-hop override (`[NHO]`)** associated with an existing matching route prefix.

For example, before a direct shortcut:

```text
D 10.12.12.0/24 via 172.16.0.1, Tunnel0
```

After a matching-prefix shortcut:

```text
D   % 10.12.12.0/24 via 172.16.0.1, Tunnel0
        [NHO] via 172.16.0.12, Tunnel0
```

The EIGRP route still names Hub1, but the **actual CEF next hop** is Spoke2. A traceroute may show only Spoke2 even though the ordinary routing entry still lists Hub1.

### Route Watch and Changes to Covering Routes

**NHRP Route Watch** tracks routing information that justifies installed shortcuts. If the relevant covering route disappears, or a newly selected covering route changes the underlying forwarding path, NHRP may invalidate the existing shortcut. This prevents stale NHRP forwarding information from indefinitely overriding changed routing topology.

For a dual-cloud failure, distinguish:

1. A route moving from `Tunnel0` to `Tunnel1`.
2. The Cloud 100 NHRP mapping stored against `Tunnel0`.
3. A new Cloud 200 Resolution that can create a separate mapping against `Tunnel1`.

Even when **both clouds lead to the same Spoke2 NBMA address** `198.51.100.13`, they have different overlay next hops and GRE keys. An old Tunnel0 shortcut is not automatically a Tunnel1 shortcut.

### Existing Shortcuts Can Mask a Hub Failure

Consider established Spoke1-to-Spoke2 forwarding over the underlay:

```text
Spoke1 -> direct GRE -> Spoke2
```

If Hub1 fails but direct spoke NBMA reachability remains intact, current traffic **may continue temporarily or uninterrupted** over its existing shortcut, depending on the route-watch and forwarding state. When that shortcut becomes invalid or expires, the router must still have a usable route and a way to resolve future paths.

Therefore, testing only `ping 10.12.12.12` during a hub failure can be misleading: the ping might be using an already established direct shortcut and never testing hub fallback.

### Test With and Without Cached Shortcuts

**Cold-path test:**

1. Establish the overlay routing adjacencies.
2. Avoid sending inter-spoke traffic before the test, or clear a relevant dynamic shortcut cache in the lab.
3. Fail the preferred hub or cloud.
4. Generate new traffic and observe the new routing path, Redirect, Resolution, and eventual shortcut.

**Warm-path test:**

1. Generate inter-spoke traffic until a direct shortcut exists.
2. Verify the shortcut and CEF state.
3. Fail the preferred hub or cloud.
4. Observe whether the shortcut survives, is purged, or is re-established through the alternate cloud.

When deliberately clearing NHRP cache entries, remember that this is a diagnostic reset that itself changes state; it is not part of normal automatic convergence.

---

## Failback When the Preferred Hub Returns

Failover and failback are different events. Restoring a hub does not necessarily mean every route instantly changes back to that hub.

### Single Cloud

When Hub1 returns:

1. Hub2 can renew or restore its registration to Hub1.
2. Spokes can reestablish their Hub1 NHRP registrations and EIGRP adjacencies.
3. EIGRP evaluates the now-available paths according to actual metrics and topology.
4. NHRP watches routing changes and can invalidate shortcuts whose underlying paths are no longer appropriate.

**There is no inherent primary-hub concept** merely because Hub1 appears first in a spoke's `ip nhrp nhs` configuration. If Hub1 and Hub2 advertise equal EIGRP metrics, the resulting route selection or ECMP behavior can differ from a strictly active/standby design.

### Dual Cloud

In the baseline design, `Tunnel0` has a lower EIGRP delay than `Tunnel1`. If the primary path becomes eligible again and remains the best route, EIGRP can prefer Cloud 100 again.

This return can cause an additional transition:

```text
Normal operation:     Cloud 100 / Tunnel0
Hub1 unavailable:     Cloud 200 / Tunnel1
Hub1 restored:        Cloud 100 / Tunnel0 (if preferred by routing)
```

Changing the best route may also affect existing NHRP shortcuts. A new direct shortcut may be learned in Cloud 100 after new traffic triggers a Redirect.

### Avoiding Repeated Path Flaps

If a hub or WAN path is intermittently reachable, paths may oscillate. Possible controls include:

- Stable underlay routing and failure detection.
- Deliberate EIGRP metric design rather than equal-cost paths everywhere.
- Careful Hello/Hold timer adjustments when needed.
- Routing policy or dampening mechanisms appropriate to the actual problem.
- NHS recovery and fallback controls, when using the optional Backup NHS feature.

**Do not treat an NHS fallback timer as a routing-protocol failback timer.** They govern different control-plane decisions.

---

## Optional Backup NHS Features

Cisco's **DMVPN Tunnel Health Monitoring and Recovery (Backup NHS)** feature allows spokes to select preferred **NHRP Next Hop Servers (NHSs)** and automatically switch to backup NHSs when their preferred servers become unavailable.

The feature provides four main capabilities:

| Feature | Purpose |
| --- | --- |
| **NHS priority** | Determines which NHSs the spoke prefers to use. |
| **Maximum NHS connections** | Limits how many NHSs the spoke actively maintains NHRP registrations with. |
| **NHS clusters** | Organizes NHSs into groups, each with its own connection limit. |
| **NHS fallback timer** | Delays returning to a higher-priority NHS after it recovers. |

These features are optional and are not required for basic dual-hub DMVPN operation.

### NHS Priority

NHS priority determines **which hubs a spoke prefers to maintain NHRP registrations with**.

An **active NHS association** means that the spoke maintains NHRP registration with that NHS and uses it for NHRP services. In a single-cloud DMVPN topology, multiple NHS associations can use the same mGRE tunnel interface. They do not require separate tunnel interfaces.

NHS priority is configured **on the spoke**, as part of the `ip nhrp nhs` command.

- The priority range is **0–255**.
- **0** is the highest priority.
- **255** is the lowest priority.
- The default priority is **0**.
- Multiple NHSs can have the same priority.

For example, configure Spoke1 to prefer Hub1 over Hub2:

```cisco
interface Tunnel0
 ip nhrp nhs 172.16.0.1 nbma 198.51.100.1 multicast priority 1
 ip nhrp nhs 172.16.0.2 nbma 198.51.100.5 multicast priority 10
```

Hub1 has the higher priority because its numerical value (`1`) is lower than Hub2's (`10`).

However, **priority alone does not limit the spoke to using only Hub1**. The spoke can maintain registrations with both hubs unless a connection limit prevents it.

### Maximum NHS Connections

The **maximum NHS connections** setting controls how many NHSs the spoke actively maintains NHRP registrations with.

It is configured using `ip nhrp nhs cluster ... max-connections`.

For example, limit Spoke1 to one active NHS:

```cisco
interface Tunnel0
 ip nhrp nhs 172.16.0.1 nbma 198.51.100.1 multicast priority 1
 ip nhrp nhs 172.16.0.2 nbma 198.51.100.5 multicast priority 10
 ip nhrp nhs cluster 0 max-connections 1
```

**Cluster 0** is the default cluster to which NHSs belong unless explicitly assigned to another cluster.

With this configuration:

1. Spoke1 maintains an active NHRP association with Hub1 because it has the higher priority.
2. Hub2 is configured as a backup NHS but is not selected for an active association.
3. If Hub1 becomes unavailable, Spoke1 can establish an active NHRP association with Hub2.
4. The spoke continues using Hub2 until Hub1 recovers and is selected again.

Changing the maximum connection count changes the behavior:

| Configuration | Normal operation |
| --- | --- |
| `max-connections 1` | Spoke1 actively uses Hub1. Hub2 is a backup. |
| `max-connections 2` | Spoke1 can actively use both Hub1 and Hub2. |

With `max-connections 2`, both hubs can remain registered simultaneously even though they have different priorities.

**NHS priority determines which hubs are preferred. Maximum connections determines how many can be active within the cluster.**

### NHS States

The Backup NHS feature tracks the state of each configured NHS.

| State | Description |
| --- | --- |
| **UP** | The NHS has an active association with the spoke. |
| **PROBE** | The NHS is not currently active, but the spoke is probing it to determine whether it can become active. |
| **DOWN** | The NHS is not currently active and is waiting to be scheduled for probing or selection. |

An NHS in the DOWN state is not necessarily unreachable. It may simply be an unselected backup that is waiting to be scheduled.

A higher-priority NHS that has failed may enter the PROBE state while the spoke continues operating through a lower-priority NHS.

### NHS Failover and Recovery

Consider the following configuration on Spoke1:

```cisco
interface Tunnel0
 ip nhrp nhs 172.16.0.1 nbma 198.51.100.1 multicast priority 1
 ip nhrp nhs 172.16.0.2 nbma 198.51.100.5 multicast priority 10
 ip nhrp nhs cluster 0 max-connections 1
```

#### 1. Normal Operation

- Hub1 is the preferred NHS because it has priority `1`.
- Spoke1 maintains its active NHRP registration with Hub1.
- Hub2 is available as a backup but is not actively selected.
- Only one NHS is active because `max-connections 1` is configured.

#### 2. Hub1 Fails

1. Spoke1 detects that Hub1 is no longer responding to NHRP registration requests.
2. Hub1 loses its active association and enters the recovery/probing process.
3. Spoke1 selects Hub2, the next preferred available NHS.
4. Spoke1 establishes an active NHRP registration with Hub2.
5. Spoke1 continues probing Hub1 for recovery.

**Failover occurs because the preferred NHS becomes unavailable, not because its priority changes.**

#### 3. Hub1 Recovers

1. Spoke1 detects that Hub1 is available again.
2. Since Hub1 has a higher priority than Hub2, Spoke1 prefers to return to Hub1.
3. Spoke1 replaces the active Hub2 association with Hub1 when fallback is permitted.
4. Hub2 returns to a backup role.

This automatic return to the preferred NHS is called **fallback**.

### NHS Fallback Timer

The **NHS fallback timer** controls how long the spoke waits before replacing an active, lower-priority NHS with a recovered, higher-priority NHS.

Configure it with:

```cisco
interface Tunnel0
 ip nhrp nhs fallback 30
```

This configures a **30-second fallback period**.

For example:

1. Spoke1 is using Hub2 because Hub1 previously failed.
2. Hub1 becomes reachable again.
3. Spoke1 detects Hub1's recovery, but continues using Hub2 during the fallback period.
4. After the fallback period expires, Spoke1 can switch back to Hub1.

This prevents the spoke from repeatedly switching between NHSs if the preferred hub experiences intermittent connectivity.

**The fallback timer delays recovery to a higher-priority NHS. It does not control how quickly the spoke detects an NHS failure or selects a backup after failure.**

### NHS Clusters

An **NHS cluster** is a group of configured NHSs that share a maximum active-connection limit.

By default, NHSs belong to **cluster 0**. The previous examples place Hub1 and Hub2 in the same default cluster.

The `cluster` keyword can also be configured explicitly:

```cisco
interface Tunnel0
 ip nhrp nhs 172.16.0.1 nbma 198.51.100.1 multicast priority 1 cluster 1
 ip nhrp nhs 172.16.0.2 nbma 198.51.100.5 multicast priority 10 cluster 1
 ip nhrp nhs cluster 1 max-connections 1
```

Here, Hub1 and Hub2 belong to cluster `1`, and Spoke1 maintains at most one active NHS association in that cluster.

#### Why Use Multiple Clusters?

Clusters become useful when a spoke has several groups of NHSs, such as two hubs at each data center.

For example:

- **Cluster 1:** Two NHSs at Data Center A.
- **Cluster 2:** Two NHSs at Data Center B.

If each cluster is configured with `max-connections 1`, the spoke maintains **one active NHS from each data center**.

If the preferred NHS at Data Center A fails, the spoke can select the other NHS in cluster 1 without affecting its active NHS association in cluster 2.

Without separate clusters, the spoke selects NHSs from a single common priority list. This could result in both active NHSs being selected from the same data center.

**Clusters provide independent NHS selection and connection limits for groups of hubs.** They do not create separate DMVPN clouds or tunnel interfaces.

### NHS Priority Versus EIGRP Route Selection

NHS priority controls **NHRP server selection**, while EIGRP independently controls **IP routing path selection**.

| Feature | Controls |
| --- | --- |
| **NHS priority** | Which hubs the spoke prefers for active NHRP registrations |
| **Maximum NHS connections** | How many NHS registrations can be active in each cluster |
| **NHS fallback timer** | When the spoke can return to a recovered, higher-priority NHS |
| **EIGRP metric** | Which route and routing next hop are preferred |
| **EIGRP feasible successor** | Whether a loop-free alternate route is immediately available |
| **NHRP Redirect/Shortcut** | Whether traffic can bypass the hub using direct spoke-to-spoke forwarding |

For example, suppose Spoke1 has:

- Hub1 NHS priority `1`.
- Hub2 NHS priority `10`.
- EIGRP adjacencies with both hubs.
- A better EIGRP metric to `10.12.12.0/24` through Hub2.

**EIGRP can prefer the route through Hub2 even though Hub1 has the higher NHS priority.**

However, NHS selection can indirectly influence routing. If Backup NHS limits Spoke1 to one active NHS association, the routing-protocol adjacency with the unselected hub may not be maintained.

Therefore, the spoke might have only one available EIGRP path until the active NHS fails and another NHS is selected.

In a **dual-cloud DMVPN topology**, each tunnel has its own NHRP configuration. NHS priorities configured on different tunnel interfaces do not directly determine which tunnel EIGRP prefers.

### Verification

Use these commands on a spoke:

```cisco
show ip nhrp nhs
show ip nhrp nhs redundancy
show dmvpn
show ip eigrp neighbors
show ip route
```

- `show ip nhrp nhs` displays configured NHSs, including priority and cluster information.
- `show ip nhrp nhs redundancy` displays Backup NHS selection and recovery information, including cluster connection limits and NHS states.
- `show dmvpn` displays DMVPN peer state.
- `show ip eigrp neighbors` verifies which routing adjacencies are established.
- `show ip route` verifies which hub is selected as the routing next hop.

Compare the NHRP results with the EIGRP neighbor and routing tables to distinguish **NHS selection** from **routing path selection**.

### Key Points

- The **Backup NHS** feature allows spokes to prioritize NHSs, limit active registrations, and switch to backups after failures.
- **NHS priority** determines which hubs are preferred. The range is **0–255**, with **0 being the highest priority**.
- The default NHS priority is **0**.
- **Maximum NHS connections** determines how many NHS associations can be active within a cluster.
- **Cluster 0** is the default cluster.
- **Multiple clusters** allow independent connection limits for different groups of NHSs.
- With `max-connections 1`, the spoke normally maintains an active registration with the highest-priority available NHS in that cluster.
- **NHS fallback** delays returning to a recovered, higher-priority NHS.
- NHS selection and routing path selection are separate mechanisms. **NHS priority does not directly influence EIGRP metrics.**

---

## What About Faster Failure Detection?

Convergence time can be reduced only if the actual failure is detected and handled earlier. Choose mechanisms appropriate to the failure domain:

| Mechanism | Detects or influences | Does not guarantee |
| --- | --- | --- |
| Physical link state | Failure of the local physical interface | Immediate detection of a remote hub's failure |
| EIGRP Hellos/Hold | Loss of an EIGRP neighbor | NHRP registration or IPsec session health |
| NHRP NHS state | NHS registration/reachability | Which EIGRP route wins |
| `if-state nhrp` | Spoke tunnel interface state based on NHS availability | Individual NHS failover when another NHS on the same interface is healthy |
| Routing metrics | Primary/backup route preference | Faster failure detection |
| Route Watch | Validity of existing shortcuts relative to route changes | Replacement of a lost routing protocol neighbor |
| Backup NHS priority/fallback | NHS connection preference and recovery | EIGRP active/standby routing policy |
| IPsec/IKEv2 liveness | Security peer/session problems when IPsec is deployed | EIGRP and NHRP state automatically matching every security change |

For this lab, start by measuring EIGRP neighbor loss and route installation, then measure the extra time required for NHRP shortcut reestablishment. Do not infer convergence from a configured timer alone.

---

## Failure Scenarios and Expected Outcomes

| Failure | Single cloud | Dual cloud |
| --- | --- | --- |
| Hub1 fails entirely | Hub2 can remain an NHS and routing neighbor on the shared Tunnel0 | Tunnel1/Cloud 200 can provide backup overlay reachability |
| Hub2 fails entirely | Hub1 can continue serving the spokes | Tunnel0/Cloud 100 remains available |
| One spoke loses reachability to Hub1 | Its Hub2 registration and adjacency can remain usable | Only its Cloud 100 path is affected, assuming independent Hub2 reachability |
| Spoke's only physical WAN fails | Both hub relationships fail | Both logical clouds fail if they share that WAN source |
| EIGRP process fails on Hub1 | NHS registration may remain even though Hub1 routes are absent | Cloud 100 NHRP may remain while Cloud 100 EIGRP routes are unusable |
| NHRP registration fails but EIGRP is temporarily up | Registration and routing states can diverge | One cloud can have failed NHRP state while the other remains healthy |
| Spoke2 loses primary cloud | EIGRP routes/shortcut reachability to Spoke2 must be reevaluated | Cloud 200 may provide an alternate spoke-to-spoke forwarding domain |
| Hub1 recovers | Routing preference depends on EIGRP metrics | Primary Cloud 100 can become preferred again if its metric is best |

A useful exercise is to predict **which exact neighbor, route, NHRP entry, and CEF adjacency should change** for each row before applying the failure.

---

## Key Points

- **Two configured NHSs provide NHRP redundancy; EIGRP determines the overlay route chosen for forwarding.** These decisions are related but separate.
- **A remote hub can fail while a local mGRE tunnel remains up/up.** Inspect NHRP state and routing neighbors, not just tunnel line protocol.
- In a **single cloud**, all routers use one tunnel subnet; surviving Hub2 can continue serving spokes even if Hub1, to which Hub2 registers, fails.
- A single-cloud spoke cannot use different interface delay or bandwidth values to select between two hubs reached over its **same tunnel interface**.
- In a **dual cloud**, each spoke has two tunnels and independent NHS relationships. Different interface metrics can influence active/standby routing.
- **EIGRP alternate paths must be eligible.** A second hub or a second tunnel does not guarantee that a particular prefix has an immediately usable backup successor.
- **NHRP shortcuts are interface/cloud-specific.** A shortcut on Tunnel0 does not automatically become a Tunnel1 shortcut after failover.
- **Route Watch** monitors shortcuts relative to the underlying routing information and can remove shortcuts that are no longer valid.
- **Existing direct spoke-to-spoke shortcuts may conceal hub failure** during testing. Test both cold and established shortcut states.
- **`if-state nhrp` is optional** and can tie a spoke tunnel's protocol state to NHS reachability; its effect depends on all NHSs configured on that interface.
- **Backup NHS priority and fallback govern NHS association choices, not EIGRP route preference.**
- Two clouds using the **same physical WAN interface** are not redundant against failure of that interface or upstream transport.
- Always verify **NHS registration, EIGRP successor, NHRP shortcut, CEF, and bidirectional forwarding** when measuring convergence.

## References

- [Cisco: Dynamic Multipoint IPsec VPNs (Using Multipoint GRE/NHRP to Scale IPsec VPNs)](https://www.cisco.com/c/en/us/support/docs/security-vpn/ipsec-negotiation-ike-protocols/41940-dmvpn.html)
- [Cisco: DMVPN Tunnel Health Monitoring and Recovery](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-multicast/b-ip-multicast/m_sec-conn-dmvpn-tun-mon.html)
- [Cisco: DMVPN Tunnel Health Monitoring and Recovery Backup NHS](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/security-vpn/security-vpn/m_sec-conn-dmvpn-backup-nhs.html)
- [Cisco: Shortcut Switching Enhancements for NHRP in DMVPN Networks](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/ip-addressing/ip-addressing/m_nhrp-switch-enhancemts-dmvpn-0.html)
- [Cisco: Troubleshoot DMVPN Issues](https://www.cisco.com/c/en/us/support/docs/security/dynamic-multipoint-vpn-dmvpn/111976-dmvpn-troubleshoot-00.html)
