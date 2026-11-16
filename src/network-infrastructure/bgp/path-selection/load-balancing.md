# 1.5.b (iii) Load Balancing

**BGP load balancing** allows traffic to the same destination prefix to be distributed across multiple forwarding paths.

By default, BGP selects a single best path for each destination prefix. However, Cisco IOS XE supports **BGP multipath**, which allows multiple eligible BGP paths to be installed in the routing table.

These paths can then be used for load balancing by **Cisco Express Forwarding (CEF)**.

---

## Load Balancing vs. Multipathing

Although the terms are often used together, **multipathing** and **load balancing** describe different functions.

| Feature | Multipathing | Load Balancing |
|---|---|---|
| Function | Selects multiple eligible paths | Distributes traffic across paths |
| Plane | Control plane | Data plane |
| BGP role | Determines which BGP paths can be installed together | Does not directly forward packets |
| CEF role | Uses the installed forwarding information | Selects the forwarding path for packets |
| Configuration | `maximum-paths` | CEF load-sharing behavior |

The distinction is:

- **Multipathing** determines which paths are available for forwarding.
- **Load balancing** determines how traffic is distributed among those paths.

For example:

```text
              BGP Table
                  |
                  v
           Best Path Selection
                  |
                  v
        BGP Multipath Evaluation
                  |
          +-------+-------+
          |               |
          v               v
        Path A          Path B
          |               |
          +-------+-------+
                  |
                  v
             RIB / FIB
                  |
                  v
          CEF Load Balancing
                  |
          +-------+-------+
          |               |
          v               v
       Next Hop A      Next Hop B
```

**Important:** BGP multipathing is not the only way to achieve load balancing. A single BGP best path can resolve through multiple equal-cost IGP paths, allowing CEF to load balance without BGP multipath.

---

## BGP Multipath Operation

By default, BGP selects one best path for each prefix.

Consider:

```text
                  AS 65002
                 R2       R3
                  \       /
                   \     /
                    \   /
                     R1
                  AS 65001
```

R2 and R3 both advertise `192.0.2.0/24` to R1.

Assume:

| Attribute | Path via R2 | Path via R3 |
|---|---|---|
| Weight | 0 | 0 |
| LOCAL_PREF | 100 | 100 |
| AS_PATH | 65002 | 65002 |
| ORIGIN | IGP | IGP |
| MED | 0 | 0 |
| IGP cost to NEXT_HOP | 0 | 0 |

All attributes required for normal EBGP multipath are equal.

### Default Behavior

Without BGP multipath enabled:

1. R1 receives both paths.
2. BGP selects one path as best.
3. Only that BGP path is installed in the RIB.
4. The other path remains available in the BGP table.

BGP might select R2 or R3 based on later best-path criteria.

### With Multipath Enabled

Configure:

```text
R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# maximum-paths 2
```

R1 can now install both paths into the RIB.

However, **BGP still designates only one path as its overall best path**.

Both paths can be used for forwarding, but BGP normally advertises only the selected best path to its neighbors.

---

## BGP Multipath Eligibility

BGP does not install every available path simply because `maximum-paths` is configured.

Paths must satisfy the relevant **multipath eligibility requirements**.

In normal equal-cost BGP multipath, the following attributes must match:

| Attribute | Requirement |
|---|---|
| Weight | Equal |
| LOCAL_PREF | Equal |
| Local origination preference | Compatible |
| AS_PATH length | Equal |
| AS_PATH content / neighboring AS | Must satisfy the applicable multipath rules |
| ORIGIN | Equal |
| MED | Equal under applicable comparison rules |
| IGP metric to NEXT_HOP | Equal |
| NEXT_HOP | Distinct usable forwarding next hops |

Additional attributes and platform-specific selection rules can also affect eligibility.

For IBGP multipath, the entire AS_PATH must normally match, not just its length.

For standard EBGP multipath, paths through the same neighboring AS can qualify without enabling AS_PATH relaxation.

### Example: Different LOCAL_PREF

R1 receives:

| Attribute | Path A | Path B |
|---|---|---|
| LOCAL_PREF | 200 | 100 |
| AS_PATH | 65002 | 65002 |
| ORIGIN | IGP | IGP |

Even with:

```text
R1(config-router-af)# maximum-paths 2
```

Only Path A is selected.

Path B does not qualify because its LOCAL_PREF is lower.

**`maximum-paths` sets the maximum number of eligible paths. It does not override BGP path-selection policies.**

---

## EBGP Multipath Configuration

Configure the maximum number of EBGP paths that BGP can install:

```text
R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# maximum-paths 4
```

This permits up to four eligible EBGP paths for a destination prefix.

The actual number installed depends on:

- The available BGP paths.
- Multipath eligibility.
- Next-hop reachability.
- RIB and platform limitations.

The default is one path.

### Example

```text
R1(config)# router bgp 65001
R1(config-router)# neighbor 10.0.12.2 remote-as 65002
R1(config-router)# neighbor 10.0.13.3 remote-as 65002
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# maximum-paths 2
```

If R2 and R3 advertise eligible paths to `192.0.2.0/24`, both can be installed.

---

## IBGP Multipath Configuration

IBGP multipath allows multiple eligible IBGP-learned paths to be installed in the RIB.

Configure:

```text
R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# maximum-paths ibgp 4
```

This permits up to four eligible IBGP paths.

For example:

```text
                    R1
                  /    \
                 /      \
               R2        R3
                 AS 65001
```

R1 learns `192.0.2.0/24` through both R2 and R3.

For normal IBGP multipath, the BGP attributes must match, including the entire AS_PATH and the IGP metric to each BGP NEXT_HOP.

The next hops must be distinct.

If eligible, both paths can be installed and used for forwarding.

**Route reflector consideration:** A route reflector normally advertises only its selected best path for a prefix. IBGP multipath on a downstream router does not automatically cause the route reflector to advertise all available paths.

---

## Combined EBGP and IBGP Multipath

Cisco IOS XE also supports **eiBGP multipath**, which allows eligible EBGP and IBGP paths to participate in the same multipath group.

This is useful in networks where a router has both external and internal paths to a destination.

Configure:

```text
R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# maximum-paths eibgp 4
```

Unlike ordinary `maximum-paths` or `maximum-paths ibgp`, this feature permits combined EBGP/IBGP multipath when its eligibility requirements are satisfied.

It is supported on applicable IOS XE platforms in both VRF and non-VRF address families.

The exact supported address families and path limits depend on the platform and software release.

---

## Multipath Across Different Autonomous Systems

By default, BGP multipath normally cannot load share across paths from different neighboring autonomous systems unless the applicable AS_PATH comparison requirements are satisfied.

Consider:

```text
               AS 65002       AS 65003
                  R2             R3
                   \             /
                    \           /
                     \         /
                         R1
                      AS 65001
```

Both R2 and R3 advertise `192.0.2.0/24`.

| Attribute | Path via R2 | Path via R3 |
|---|---|---|
| AS_PATH | 65002 | 65003 |
| AS_PATH length | 1 | 1 |
| Weight | 0 | 0 |
| LOCAL_PREF | 100 | 100 |
| ORIGIN | IGP | IGP |
| MED | 0 | 0 |

Although the AS_PATH lengths are equal, the neighboring AS numbers are different.

On platforms supporting AS_PATH multipath relaxation, configure:

```text
R1(config)# router bgp 65001
R1(config-router)# bgp bestpath as-path multipath-relax
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# maximum-paths 2
```

The `bgp bestpath as-path multipath-relax` command relaxes the AS_PATH matching requirements for multipath selection.

Paths with different AS_PATH contents can qualify if their lengths and other required attributes match.

### Important

- `bgp bestpath as-path multipath-relax` does not independently enable multipath installation.
- A suitable `maximum-paths` command is still required.
- The AS_PATH lengths must still match under normal multipath relaxation.
- Other eligibility requirements continue to apply.
- It does not disable AS_PATH loop prevention.

This command is different from `bgp bestpath as-path ignore`, which changes the ordinary best-path selection algorithm.

**Platform note:** AS_PATH multipath relaxation is platform and release dependent. Verify CLI support on the target IOS XE image.

---

## CEF Load Balancing

After multipath routes are installed, **CEF** determines how packets are distributed across the available forwarding paths.

Two important methods are:

1. Per-destination (flow-based) load balancing.
2. Per-packet load balancing.

### Per-Destination Load Balancing

This is the normal default CEF behavior.

CEF uses a hash based on packet header information to select a forwarding path.

Depending on platform and configuration, hash inputs can include:

- Source IP address.
- Destination IP address.
- Layer 4 port numbers.

Packets belonging to the same flow normally follow the same path.

For example:

```text
Flow A -------- R1 -------- R2
                 |
Flow B ---------+
                 |
                 +---------- R3

R2 and R3 provide paths to the same destination.
```

CEF might select:

| Flow | Selected Next Hop |
|---|---|
| 10.1.1.1 to 192.0.2.10 | R2 |
| 10.1.1.2 to 192.0.2.10 | R3 |
| 10.1.1.3 to 192.0.2.20 | R2 |

The exact distribution depends on the hash algorithm.

Advantages:

- Preserves packet ordering within a flow.
- Distributes many flows over available paths.
- Generally preferred for normal production traffic.

**Per-destination load balancing does not guarantee equal link utilization.**

For example, a single large flow might use R2 while R3 remains mostly idle.

### Per-Packet Load Balancing

With per-packet load balancing, successive packets can be forwarded over different paths.

For example:

```text
Packet 1 -> R2
Packet 2 -> R3
Packet 3 -> R2
Packet 4 -> R3
```

On platforms supporting the feature, configure it on the relevant outgoing interfaces:

```text
R1(config)# interface Ethernet0/0
R1(config-if)# ip load-sharing per-packet
```

Restore per-destination behavior:

```text
R1(config-if)# ip load-sharing per-destination
```

However, per-packet load balancing has disadvantages:

- Packets can arrive out of order.
- TCP performance can be affected.
- Delay variation can affect real-time applications.
- Some IOS XE platforms do not support configurable per-packet CEF load balancing.

Per-flow load balancing is generally preferred.

---

## Load Balancing Without BGP Multipath

An important distinction is that **CEF can load balance traffic to a BGP destination even when BGP selects only one path**.

This occurs when the BGP NEXT_HOP resolves through multiple equal-cost routes.

Consider:

```text
                        R4
                    192.0.2.0/24
                         |
                         |
                         R3
                     3.3.3.3/32
                       /   \
                      /     \
                    R2       R5
                      \     /
                       \   /
                         R1
```

R1 learns one BGP path:

```text
192.0.2.0/24
NEXT_HOP: 3.3.3.3
```

However, its IGP provides two equal-cost paths to `3.3.3.3/32`:

```text
3.3.3.3/32 via R2
3.3.3.3/32 via R5
```

BGP selects only one path to `192.0.2.0/24`.

But recursive next-hop resolution can produce two forwarding paths:

```text
              BGP Best Path
              192.0.2.0/24
                     |
                     v
             NEXT_HOP 3.3.3.3
                     |
                     v
             IGP ECMP Resolution
                  /     \
                 /       \
                R2       R5
```

CEF can distribute traffic across both R2 and R5.

**BGP multipath is not required in this situation.**

This is IGP-based ECMP used to resolve a single BGP NEXT_HOP, not multiple BGP paths installed for the destination.

---

## Unequal-Cost Load Sharing with BGP Link Bandwidth

By default, BGP multipath is used with paths that meet equal-cost eligibility requirements.

However, **BGP Link Bandwidth**, also called **DMZ Link Bandwidth**, allows traffic to be distributed proportionally across links with different bandwidths.

For example:

```text
                  20 Mbps
             +---------------- R2
             |
             |
             R1
             |
             +---------------- R3
                  50 Mbps
```

Both R2 and R3 provide eligible paths to the same destination.

Without bandwidth-aware load sharing, CEF may distribute flows approximately equally across the two paths.

However, R3 has more capacity.

Using BGP Link Bandwidth, the intended traffic-share ratio becomes:

```text
R2: 20 / 70 = 28.6%
R3: 50 / 70 = 71.4%
```

Actual traffic utilization depends on flow distribution and CEF implementation.

### Configuration

On supported IOS XE platforms:

```text
R1(config)# interface Ethernet0/0
R1(config-if)# bandwidth 20000
R1(config-if)# exit
R1(config)# interface Ethernet0/1
R1(config-if)# bandwidth 50000
R1(config-if)# exit

R1(config)# router bgp 65001
R1(config-router)# neighbor 10.0.12.2 remote-as 65002
R1(config-router)# neighbor 10.0.13.3 remote-as 65002
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# maximum-paths 2
R1(config-router-af)# bgp dmzlink-bw
R1(config-router-af)# neighbor 10.0.12.2 dmzlink-bw
R1(config-router-af)# neighbor 10.0.13.3 dmzlink-bw
```

The commands have different purposes:

| Command | Purpose |
|---|---|
| `maximum-paths 2` | Enables multipath installation |
| `bgp dmzlink-bw` | Enables bandwidth-proportional load sharing |
| `neighbor ... dmzlink-bw` | Associates link bandwidth with routes learned from that EBGP neighbor |
| `bandwidth` | Supplies the interface bandwidth value used by the feature |

The interface `bandwidth` command does not physically limit or change the interface's transmission speed.

### Propagating Link Bandwidth Through IBGP

Link bandwidth information is carried in a **BGP extended community**.

To advertise the community to an IBGP peer:

```text
R1(config-router-af)# neighbor 10.0.14.4 send-community extended
```

The receiving router must also support the feature and have the appropriate multipath and bandwidth-aware forwarding configuration.

**Important:** BGP Link Bandwidth changes how traffic is shared across eligible multipaths. It does not simply permit arbitrary BGP paths with different LOCAL_PREF, AS_PATH lengths, or other incompatible attributes to be installed together.

See [Extended BGP Communities](../routing-policies/communities/extended.md).

---

## Verifying BGP Multipath and Load Balancing

### BGP Table

Display all BGP paths for a prefix:

```text
R1# show bgp ipv4 unicast 192.0.2.0/24
```

Look for:

- The path marked `best`.
- Additional paths marked `multipath`.
- The BGP NEXT_HOP of each path.
- Any differences in BGP attributes.

A path can be marked `multipath` without being the single selected best path.

### IP Routing Table

Verify RIB installation:

```text
R1# show ip route 192.0.2.0
```

With two installed paths, output might include:

```text
Routing entry for 192.0.2.0/24
  Known via "bgp 65001", distance 20
  Routing Descriptor Blocks:
    10.0.12.2, via Ethernet0/0
    10.0.13.3, via Ethernet0/1
```

Both next hops are available for forwarding.

### CEF Table

Verify forwarding information:

```text
R1# show ip cef 192.0.2.0/24 detail
```

This can identify the resolved forwarding paths.

To check which path CEF selects for a particular source/destination pair:

```text
R1# show ip cef exact-route 10.1.1.10 192.0.2.10
```

Test different source and destination addresses to observe how flows are distributed.

The displayed forwarding information and available verification options depend on the platform.

### Link Bandwidth Verification

```text
R1# show bgp ipv4 unicast 192.0.2.0/24
R1# show ip route 192.0.2.0
```

With BGP Link Bandwidth enabled, detailed BGP output can display the `DMZ-Link Bw` extended-community information.

Routing and forwarding output can also show relative traffic-share values.

---

## Troubleshooting BGP Load Balancing

| Problem | Possible Cause |
|---|---|
| Only one BGP path installed | `maximum-paths` not configured |
| Second path not eligible | Different Weight, LOCAL_PREF, ORIGIN, MED, AS_PATH, or next-hop metric |
| Paths through different providers not shared | AS_PATH multipath comparison requirements not satisfied |
| IBGP paths not shared | `maximum-paths ibgp` missing or IGP next-hop metrics differ |
| EBGP and IBGP paths not shared | `maximum-paths eibgp` not configured or unsupported |
| Multiple paths in BGP but not RIB | RIB installation failure or platform restriction |
| Multiple next hops installed but uneven usage | CEF flow hashing or unequal traffic flows |
| DMZ bandwidth ratio not applied | Missing `bgp dmzlink-bw`, neighbor configuration, extended community, or platform support |
| IBGP router sees only one path | Route reflector advertises only its best path |

---

## Key Points

- **BGP multipathing** selects multiple eligible BGP paths for RIB installation.
- **CEF load balancing** distributes traffic over available forwarding paths.
- BGP selects and normally advertises **one best path**, even when multiple paths are installed.
- `maximum-paths` enables EBGP multipath.
- `maximum-paths ibgp` enables IBGP multipath.
- `maximum-paths eibgp` enables combined EBGP/IBGP multipath on supported platforms.
- `bgp bestpath as-path multipath-relax` relaxes AS_PATH matching requirements where supported.
- CEF normally uses per-flow hashing rather than round-robin per-packet forwarding.
- A single BGP path can achieve load balancing through recursive IGP ECMP resolution.
- **BGP Link Bandwidth** allows bandwidth-proportional traffic sharing across eligible multipaths.

---

## References

- [Cisco: Select BGP Best-path Algorithm](https://www.cisco.com/c/en/us/support/docs/ip/border-gateway-protocol-bgp/13753-25.html)
- [Cisco: iBGP Multipath Load Sharing](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/ip-routing/b-ip-routing/m_irg-multi-load-0.html)
- [Cisco: eiBGP Multipath for Non-VRF Interfaces](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-routing/b-ip-routing/m_irg-eibgp-multipath-for-nonvrf-interfaces.html)
- [Cisco: BGP Link Bandwidth](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/ip-routing/b-ip-routing/m_irg-link-band-0.html)
- [Cisco: Troubleshooting CEF Load Balancing](https://www.cisco.com/c/en/us/support/docs/ip/express-forwarding-cef/18285-loadbal-cef.html)
