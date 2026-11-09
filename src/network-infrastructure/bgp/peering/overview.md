# 1.5.a IBGP and EBGP Peer Relations

**Internal BGP (IBGP)** and **External BGP (EBGP)** are two types of BGP peer relationships. Both use the same BGP protocol, but they differ in how peers are identified, how routes are advertised, and how routing loops are prevented.

- **IBGP** operates between BGP speakers in the same autonomous system (AS).
- **EBGP** operates between BGP speakers in different autonomous systems.

The distinction is important because it affects **AS_PATH**, **NEXT_HOP**, **route advertisement rules**, **TTL**, and **administrative distance**.

---

## IBGP vs. EBGP

Consider the following topology:

```text
                    AS 65001                         AS 65002
          +---------------------------+     +-------------------+
          |                           |     |                   |
          |   R1 ----------- R2 ------+-----+------ R3          |
          |          IBGP             | EBGP|                   |
          |                           |     |                   |
          +---------------------------+     +-------------------+
```

- R1 and R2 form an **IBGP** session because both are in AS 65001.
- R2 and R3 form an **EBGP** session because they are in different ASes.
- An IBGP peer does not have to be directly connected. An EBGP peer is normally directly connected unless multihop peering is configured.

| Characteristic | IBGP | EBGP |
|---|---|---|
| Peer ASNs | Same | Different |
| Typical use | Distribute BGP routes within an AS | Exchange routes between ASes |
| Default IP TTL for BGP packets | 255 | 1 |
| Directly connected peering required by default | No | Yes, normally |
| Typical peering addresses | Loopback interfaces | Directly connected interface addresses |
| Prepend local ASN to AS_PATH when advertising | No | Yes |
| Default NEXT_HOP behavior | Usually preserves the existing next hop | Usually sets the next hop to self |
| Advertise an IBGP-learned route to another IBGP peer | No, except through mechanisms such as route reflection | Not applicable |
| Default administrative distance of learned routes | 200 | 20 |

These are normal Cisco IOS XE behaviors. Specific features and policies can modify several of them.

> **Important:** IBGP and EBGP identify the type of *BGP session*. They do not indicate whether a route originated from an internal or external routing protocol. For example, a route learned through EBGP can subsequently be advertised to an IBGP neighbor.

---

## How a Peer Relationship Is Identified

On Cisco IOS XE, BGP is configured with a local ASN and an expected ASN for each neighbor.

```text
R1(config)# router bgp 65001
R1(config-router)# neighbor 10.0.12.2 remote-as 65001
```

R1's local ASN and the configured remote ASN match, so this is an **IBGP** neighbor.

```text
R2(config)# router bgp 65001
R2(config-router)# neighbor 10.0.23.3 remote-as 65002
```

R2's local ASN differs from the configured remote ASN, so this is an **EBGP** neighbor.

The two peers must agree on the ASNs presented during session establishment. For a normal EBGP relationship, each side specifies the other router's ASN as its `remote-as`. For IBGP, both routers use the same ASN.

The **BGP router ID** is a separate 32-bit identifier. Two IBGP peers share an ASN, but should have distinct router IDs.

BGP supports **4-byte ASNs**, including values greater than 65535. Their representation and private ASN ranges are covered in [4-Byte AS Numbers](four-byte-asn.md) and [Private AS Numbers](private-asn.md).

---

## BGP Sessions Are Logical Relationships

BGP establishes peer relationships using **TCP port 179**. Unlike OSPF or EIGRP, BGP does not normally discover neighbors dynamically through Hello packets.

The BGP session is independent of the path that carries its TCP packets:

```text
        BGP control-plane session
      <-------------------------->
   R1 --------- R2 --------- R3
          Underlying IP path
```

R1 and R3 could establish an IBGP session even though R2 is between them, provided that:

1. Their peering addresses are reachable through IP routing.
2. Their TCP traffic is permitted in both directions.
3. The configured local and remote ASNs and other session parameters are compatible.
4. Each peer uses the source address that the other expects.

An IGP or static routes commonly provides reachability between IBGP loopback addresses. **The IGP does not establish the BGP session:** it only supplies the IP connectivity needed for the TCP connection.

After TCP establishes, the peers exchange BGP OPEN and KEEPALIVE messages and reach the **Established** state. Only then can they exchange route information for enabled address families.

An **Established** session does not guarantee that any prefixes are being exchanged. Neighbors can be Established with zero received prefixes.

See [BGP Messages and Neighbor States](../messages-states.md) for the full session establishment process.

---

## BGP Route Advertisement Rules

A route's advertisement eligibility depends on how it was learned and on the type of neighbor receiving it.

| Route source | Advertise to IBGP peer? | Advertise to EBGP peer? |
|---|---|---|
| Locally originated BGP route | Yes | Yes |
| Route learned from EBGP | Yes | Yes |
| Route learned from IBGP | **No, by default** | Yes |

This table describes the basic propagation rules, not an unconditional guarantee. BGP normally advertises eligible selected paths, subject to next-hop validity, routing policy, loop prevention, and other advertisement controls.

### EBGP Route Advertisement

When a router advertises a route to an EBGP neighbor, it normally:

- **Prepends its local ASN** to the AS_PATH.
- **Changes NEXT_HOP** to an address on the advertising router, subject to exceptions.
- Can advertise routes learned from either IBGP or EBGP, subject to normal eligibility and policy rules.

For example:

```text
       AS 65001         AS 65002         AS 65003
          R1 ------------- R2 ------------- R3

R3 originates 203.0.113.0/24.

R2 receives AS_PATH: 65003
R1 receives AS_PATH: 65002 65003
```

A router normally rejects a received BGP path if the **local ASN already appears in AS_PATH**. This is an important mechanism for preventing inter-AS routing loops.

### IBGP Route Advertisement

When a router advertises a route to an IBGP neighbor, it normally:

- **Does not prepend** its local ASN to AS_PATH.
- **Preserves the NEXT_HOP** of an EBGP-learned route unless configured otherwise.
- **Does not advertise a route learned from one IBGP neighbor to another IBGP neighbor**.

The last rule is often called the **IBGP split-horizon rule**. It is the reason ordinary IBGP designs need a full mesh of sessions or a mechanism such as route reflection.

---

## IBGP Full-Mesh Requirement

Consider three routers in the same AS:

```text
                      AS 65001

                 R1 ---- R2 ---- R3
                      IBGP   IBGP
```

If R1 advertises a locally originated BGP route to R2:

1. R2 learns the route through IBGP.
2. R2 does **not** readvertise the route to R3 through their IBGP session.
3. R3 cannot learn that route from R2 through ordinary IBGP.

To ensure that all three routers can exchange IBGP-learned routing information without a route reflector, R1 must also peer with R3:

```text
                      AS 65001

                       R1
                      /  \
                   IBGP  IBGP
                    /      \
                   R2------R3
                       IBGP
```

For `N` routers, a full mesh requires:

`Number of IBGP sessions = N × (N - 1) / 2`

| IBGP routers | Full-mesh sessions |
|---|---|
| 3 | 3 |
| 5 | 10 |
| 10 | 45 |
| 100 | 4,950 |

Full-mesh IBGP does **not** require a physical link between every pair of routers. It requires a logical BGP session between every pair of participating BGP speakers.

**Route reflectors** reduce the number of required IBGP sessions by allowing selected IBGP-learned routes to be reflected to other IBGP peers. They use additional loop-prevention mechanisms, including ORIGINATOR_ID and CLUSTER_LIST.

See [Route Reflectors](../scalability/route-reflectors/overview.md) for detailed rules and configuration.

---

## NEXT_HOP Behavior and Reachability

**NEXT_HOP** identifies the address toward which a router must forward traffic to reach an advertised prefix. This is not necessarily the address of the BGP neighbor that sent the advertisement.

Consider this topology:

```text
                  AS 65001                     AS 65002

             R1 ----------- R2 ------------------- R3
                    IBGP              EBGP
                              10.0.23.2      10.0.23.3

                                        203.0.113.0/24
                                          behind R3
```

Suppose R3 advertises `203.0.113.0/24` to R2 through EBGP.

R2 receives the route with a NEXT_HOP of `10.0.23.3`.

When R2 advertises the route to R1 through IBGP, the NEXT_HOP normally remains `10.0.23.3`.

```text
R3 -> R2 (EBGP):
  Prefix:   203.0.113.0/24
  AS_PATH:  65002
  NEXT_HOP: 10.0.23.3

R2 -> R1 (IBGP):
  Prefix:   203.0.113.0/24
  AS_PATH:  65002
  NEXT_HOP: 10.0.23.3
```

R1 must be able to resolve `10.0.23.3` using its routing table. If it cannot, the BGP path is not usable for normal best-path selection and forwarding.

A common solution is to configure R2 to change the next hop when advertising to R1:

```text
R2(config-router)# address-family ipv4 unicast
R2(config-router-af)# neighbor 1.1.1.1 next-hop-self
```

If R2's IBGP session uses Loopback0 (`2.2.2.2`) as its update source, R1 will normally receive R2's advertised route with NEXT_HOP `2.2.2.2`. R1 must still have an IP route to that address.

> **CCIE distinction:** Reachability to the **BGP peer address** is needed to establish and maintain the TCP session. Reachability to a route's **BGP NEXT_HOP** is needed for the route to be usable. These are two separate requirements.

See [BGP Next-Hop Handling](next-hop.md) for more detail, including exceptions to default next-hop processing.

---

## Directly Connected vs. Multihop Peering

### EBGP

By default, EBGP normally uses a **TTL of 1** and requires a directly connected peer.

```text
R1 (AS 65001) -------- R2 (AS 65002)
    10.0.12.1             10.0.12.2
```

EBGP peers can also form sessions across intermediate routers when configured for multihop operation:

```text
R1 (AS 65001) ----- R2 ----- R3 (AS 65002)
   <------ EBGP session ------>
```

For example:

```text
R1(config-router)# neighbor 3.3.3.3 ebgp-multihop 2
```

This allows a larger outgoing TTL and permits non-directly connected EBGP peers. Both peers require appropriate configurations and IP reachability.

For a single-hop EBGP session that uses loopback addresses, IOS XE also supports `neighbor disable-connected-check` in appropriate designs.

### IBGP

IBGP normally uses a **TTL of 255** and does not require directly connected peers. Loopback-to-loopback peering is common:

```text
R1(config-router)# neighbor 2.2.2.2 remote-as 65001
R1(config-router)# neighbor 2.2.2.2 update-source Loopback0
```

The `update-source` command selects the local source interface/address for BGP TCP sessions. The other router must configure the matching neighbor address and have a return route to it.

See [BGP Update Source and Multihop Peering](multihop.md) for examples and TTL security considerations.

---

## Basic IBGP and EBGP Configuration

The following example combines IBGP and EBGP in a simple three-router topology:

```text
                    AS 65001                     AS 65002

      Lo0 1.1.1.1/32    Lo0 2.2.2.2/32
           R1 ---------------- R2 ---------------- R3
           |        IBGP        |       EBGP       |
       E0/0 .1                E0/0 .2         E0/0 .3
           +---10.0.12.0/30---+                 |
                            E0/1 .2             |
                              +--10.0.23.0/30---+

                                     R3 Loopback0:
                                     203.0.113.3/24
```

Assume all interfaces and IP addresses are already configured. R1 and R2 will peer using their loopbacks, and R2 and R3 will peer using their directly connected addresses.

### R1: IBGP Neighbor

```text
R1(config)# ip route 2.2.2.2 255.255.255.255 10.0.12.2
R1(config)# router bgp 65001
R1(config-router)# bgp router-id 1.1.1.1
R1(config-router)# neighbor 2.2.2.2 remote-as 65001
R1(config-router)# neighbor 2.2.2.2 update-source Loopback0
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 2.2.2.2 activate
```

### R2: IBGP and EBGP Neighbors

```text
R2(config)# ip route 1.1.1.1 255.255.255.255 10.0.12.1
R2(config)# router bgp 65001
R2(config-router)# bgp router-id 2.2.2.2
R2(config-router)# neighbor 1.1.1.1 remote-as 65001
R2(config-router)# neighbor 1.1.1.1 update-source Loopback0
R2(config-router)# neighbor 10.0.23.3 remote-as 65002
R2(config-router)# address-family ipv4 unicast
R2(config-router-af)# neighbor 1.1.1.1 activate
R2(config-router-af)# neighbor 1.1.1.1 next-hop-self
R2(config-router-af)# neighbor 10.0.23.3 activate
```

### R3: EBGP Neighbor

```text
R3(config)# router bgp 65002
R3(config-router)# bgp router-id 3.3.3.3
R3(config-router)# neighbor 10.0.23.2 remote-as 65001
R3(config-router)# address-family ipv4 unicast
R3(config-router-af)# neighbor 10.0.23.2 activate
R3(config-router-af)# network 203.0.113.0 mask 255.255.255.0
```

R3's `network` statement requires an exact `203.0.113.0/24` route in its routing table, supplied here by Loopback0.

The example uses `neighbor activate` explicitly for clarity. On IOS XE, IPv4 unicast neighbors are activated by default unless `no bgp default ipv4-unicast` is configured before the neighbors are defined.

The result is:

1. R3 advertises `203.0.113.0/24` through EBGP to R2.
2. R2 learns the route with AS_PATH `65002`.
3. R2 advertises the selected route through IBGP to R1 without adding AS 65001 to AS_PATH.
4. R2 sets itself as NEXT_HOP toward R1 because of `next-hop-self`.
5. R1 can resolve that next hop using its static route toward R2's loopback.

For a complete explanation of the neighbor commands, see [IBGP and EBGP Neighbor Configuration](neighbor-configuration.md).

---

## Administrative Distance

Cisco IOS XE uses different default administrative distances for EBGP and IBGP routes:

| Route type | Default AD |
|---|---|
| EBGP-learned | 20 |
| IBGP-learned | 200 |
| Locally originated BGP | 200 |

Administrative distance is used when the router selects between routes to the **same prefix from different routing sources**. It does not replace BGP's own best-path algorithm when comparing multiple BGP paths.

For example, an OSPF route (AD 110) can be preferred for RIB installation over an IBGP route (AD 200) for the same prefix, even when the IBGP path is marked as BGP's best path.

See [BGP Path Selection and RIB Installation](../path-selection/best-path/rib-installation.md) for details.

---

## Verification and Troubleshooting

### Verify the BGP Sessions

```text
R2# show bgp ipv4 unicast summary
R2# show bgp ipv4 unicast neighbors 1.1.1.1
R2# show bgp ipv4 unicast neighbors 10.0.23.3
```

In the summary output, check the neighbor's AS number and `State/PfxRcd`. An established neighbor normally displays a prefix count in that column, even if the count is `0`.

### Verify Routes and Next Hops

```text
R1# show bgp ipv4 unicast 203.0.113.0/24
R1# show ip route 2.2.2.2
R1# show ip route 203.0.113.0
R1# show ip cef 203.0.113.0
```

On R1, the BGP path should show AS_PATH `65002` and NEXT_HOP `2.2.2.2`. If it is eligible for installation, the installed route should normally appear as an IBGP route with AD 200.

### Verify Received and Advertised Routes

```text
R2# show bgp ipv4 unicast neighbors 10.0.23.3 routes
R2# show bgp ipv4 unicast neighbors 1.1.1.1 advertised-routes
```

The first command shows accepted routes from R3. The second shows paths advertised by R2 to R1. Output and displayed attributes can vary with platform and policy.

### Common Problems

| Symptom | Likely cause or check |
|---|---|
| Session never reaches Established | Peer IP reachability, TCP/179 filtering, incorrect peer address, wrong `remote-as`, missing `update-source`, EBGP TTL/connected check |
| Session Established but no prefixes received | Address-family activation, route origination, outbound filtering, inbound filtering |
| IBGP session works but a route is missing farther inside the AS | IBGP-to-IBGP advertisement restriction, missing full-mesh session or route reflection |
| Route present in BGP table but not usable | Unreachable NEXT_HOP or other path eligibility issue |
| Best BGP route not installed in IP routing table | Another routing source has a lower AD, or another RIB-installation issue |
| EBGP route unexpectedly rejected | Local ASN found in AS_PATH, filtering policy, or other route validation issue |

**Troubleshooting sequence:** First verify **peer reachability and session establishment**, then **address-family exchange**, then **route advertisement and filtering**, and finally **NEXT_HOP resolution and RIB installation**.

---

## Topics in This Section

The following pages explore IBGP and EBGP relationships in greater detail:

- [IBGP and EBGP Neighbor Configuration](neighbor-configuration.md): Local/remote ASN configuration and address-family activation.
- [BGP Next-Hop Handling](next-hop.md): Default NEXT_HOP behavior and next-hop-self.
- [BGP Update Source and Multihop Peering](multihop.md): Loopback peering, multihop EBGP, and TTL handling.
- [Peer Groups and Templates](peer-groups-templates/overview.md): Reusable neighbor configuration and policy inheritance.
- [Active and Passive](active-passive.md): TCP connection initiation and passive neighbors.
- [Timers](timers.md): Keepalive, hold time, and session maintenance.
- [Dynamic Neighbors](dynamic-neighbors.md): Dynamic BGP peer acceptance.
- [4-Byte AS Numbers](four-byte-asn.md): Four-octet ASN support and notation.
- [Private AS Numbers](private-asn.md): Private ASN ranges and their use in BGP.

---

## Cisco Documentation

- [Configuring a Basic BGP Network (Cisco IOS XE 17.x)](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-routing/b-ip-routing/m_irg-basic-net-0.html)
- [Configuring BGP Neighbor Session Options (Cisco IOS XE)](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/ip-routing/b-ip-routing/m_irg-neighbor-0.html)
- [Configuring Internal BGP Features (Cisco IOS XE 17.x)](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-routing/b-ip-routing/m_irg-int-features-0.html)
