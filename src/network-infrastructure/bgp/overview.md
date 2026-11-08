
# BGP Overview

**Border Gateway Protocol (BGP)** is a **path-vector routing protocol** used to exchange routing information between autonomous systems (ASes) and within an autonomous system.

BGP is the primary routing protocol used to exchange routes between networks on the Internet. It is also widely used in enterprise networks, service provider networks, and MPLS VPN environments.

Unlike IGPs such as OSPF and EIGRP, BGP focuses on **policy-based routing decisions** rather than simply selecting the path with the lowest metric.

BGP uses attributes such as **AS_PATH, LOCAL_PREF, MED, and NEXT_HOP** to determine which routes to advertise and which paths to prefer.

---

## Autonomous Systems

An **Autonomous System (AS)** is a collection of networks operating under a common routing administration.

Each AS is identified by an **Autonomous System Number (ASN)**.

For example:

```text
            AS 65001
         +-------------+
         |             |
         |  R1 --- R2  |
         |          |  |
         +----------|--+
                    |
                    | BGP
                    |
         +----------|--+
         |          |  |
         |  R3 --- R4  |
         |             |
         +-------------+
            AS 65002
```

R1 and R2 belong to AS 65001.

R3 and R4 belong to AS 65002.

BGP allows these autonomous systems to exchange routing information while maintaining independent routing policies.

BGP supports both **2-byte and 4-byte ASNs**.

| ASN Type | Range |
|---|---|
| 2-byte | 0 to 65535 |
| 4-byte | 0 to 4294967295 |
| 16-bit private ASNs | 64512 to 65534 |
| 32-bit private ASNs | 4200000000 to 4294967294 |

Not every ASN in these numerical ranges is available for general assignment. Some values are reserved for special purposes.

Private ASNs are commonly used in enterprise networks and lab environments.

---

## EBGP and IBGP

BGP supports two types of peer relationships:

- **External BGP (EBGP):** BGP peering between routers in different autonomous systems.
- **Internal BGP (IBGP):** BGP peering between routers in the same autonomous system.

```text
             AS 65001                    AS 65002

          R1 -------- R2 ------------- R3 -------- R4
                IBGP         EBGP             IBGP
```

### EBGP

**EBGP** exchanges routing information between different autonomous systems.

For example:

```text
R2 (AS 65001) -------- R3 (AS 65002)
```

EBGP is commonly used for:

- Internet connectivity between ISPs
- Enterprise connections to ISPs
- Multihomed enterprise networks
- Inter-AS routing

EBGP normally adds the local ASN to the AS_PATH when advertising routes to another AS.

This helps prevent inter-AS routing loops.

### IBGP

**IBGP** exchanges BGP routing information between routers within the same autonomous system.

For example:

```text
R1 (AS 65001) -------- R2 (AS 65001)
```

IBGP allows routes learned through EBGP to be distributed within an AS.

An important IBGP rule is:

**A route learned from an IBGP neighbor is not advertised to another IBGP neighbor by default.**

This rule prevents certain routing loops but creates a scalability problem.

Without additional mechanisms, IBGP routers generally require a **full mesh of IBGP sessions** to distribute all IBGP-learned routes.

**Route reflectors** solve this problem by allowing selected IBGP routes to be reflected between IBGP peers.

### EBGP vs. IBGP

| Characteristic | EBGP | IBGP |
|---|---|---|
| Peer ASNs | Different | Same |
| Default TTL | 1 | 255 |
| Default administrative distance | 20 | 200 |
| Adds local AS to AS_PATH | Yes | No |
| Changes NEXT_HOP by default | Typically yes | Typically no |
| Advertises IBGP-learned routes | Yes, subject to policy | No, by default |

The exact behavior of NEXT_HOP and route advertisement can be modified through BGP configuration.

---

## BGP Transport and Neighbor Relationships

BGP uses **TCP port 179** to establish reliable sessions between neighbors.

Unlike OSPF and EIGRP, BGP does not require its own transport reliability mechanisms because TCP provides reliable, ordered delivery.

BGP neighbors are also called **BGP peers**.

A BGP session must be established before the routers can exchange routing information.

```text
R1                                  R2
 |                                   |
 |-------- TCP connection ---------->|
 |                                   |
 |<------- BGP OPEN exchange ------->|
 |                                   |
 |<------- BGP KEEPALIVE ----------->|
 |                                   |
 |         Established               |
 |                                   |
 |<--------- BGP UPDATE ------------>|
 |                                   |
```

BGP uses four fundamental message types:

| Message | Purpose |
|---|---|
| **OPEN** | Establishes a BGP session and negotiates parameters |
| **UPDATE** | Advertises or withdraws routing information |
| **KEEPALIVE** | Maintains the BGP session |
| **NOTIFICATION** | Reports an error or session termination |

BGP also uses a finite state machine to manage neighbor sessions.

The six BGP states are:

1. Idle
2. Connect
3. Active
4. OpenSent
5. OpenConfirm
6. Established

Only after reaching the **Established** state can neighbors exchange routing information.

---

## BGP Path Attributes

BGP uses **path attributes** to describe routes and influence routing decisions.

Unlike OSPF and EIGRP, which primarily calculate route metrics, BGP evaluates a sequence of attributes when selecting its preferred path.

Important BGP attributes include:

| Attribute | Purpose |
|---|---|
| **Weight** | Cisco-specific value used to influence local path selection |
| **LOCAL_PREF** | Determines preferred exit paths within an AS |
| **AS_PATH** | Records the sequence of autonomous systems traversed |
| **ORIGIN** | Identifies how a route originated into BGP |
| **MED** | Suggests a preferred entry point into an AS |
| **NEXT_HOP** | Identifies the next-hop address used to reach a destination |
| **COMMUNITY** | Tags routes to support routing policies |

For example:

```text
                  AS 65001
                 /        \
                /          \
             AS 65002     AS 65003
                \          /
                 \        /
                  AS 65004
```

AS 65001 might learn two routes to a destination in AS 65004.

```text
Path 1: 65002 65004
Path 2: 65003 65004
```

Both paths have the same AS_PATH length.

BGP must evaluate other attributes to determine the best path.

**BGP does not automatically select the path with the fewest router hops or lowest latency.**

Its best path selection process follows a defined sequence of attribute comparisons.

---

## BGP Routing Policies

One of BGP's most important capabilities is **routing policy control**.

A router can control:

- Which routes it accepts from neighbors
- Which routes it advertises to neighbors
- Which paths it prefers for outgoing traffic
- How it influences incoming traffic
- Which attributes it attaches to advertised routes

Cisco IOS XE supports several tools for implementing BGP routing policies.

| Tool | Purpose |
|---|---|
| Prefix lists | Match routes based on their prefixes and prefix lengths |
| AS-path access lists | Match routes based on their AS_PATH |
| Community lists | Match routes based on BGP communities |
| Route maps | Match routes and apply filtering or attribute modifications |
| Filter lists | Filter BGP routes using AS-path access lists |
| Distribute lists | Filter BGP routes using access lists |
| Conditional advertisement | Advertise routes based on the presence or absence of other routes |

These mechanisms are essential for controlling routing behavior in multihomed networks.

For example, an enterprise connected to two ISPs might use **LOCAL_PREF** to prefer one ISP for outbound traffic and **AS_PATH prepending** to influence inbound traffic.

---

## BGP Convergence and Scalability

BGP is designed to support large routing tables and complex routing policies.

However, large BGP networks introduce scalability challenges.

Important mechanisms include:

### Route Reflectors

Route reflectors reduce the requirement for a full mesh of IBGP sessions.

A route reflector can advertise routes learned from one IBGP client to other IBGP clients.

```text
             RR
           / | \
          /  |  \
        R1   R2  R3
```

This significantly reduces the number of BGP sessions required in large autonomous systems.

### Route Aggregation

BGP supports **route aggregation**, allowing multiple specific prefixes to be represented by a less-specific prefix.

For example:

```text
10.0.0.0/24
10.0.1.0/24
10.0.2.0/24
10.0.3.0/24
       |
       v
  10.0.0.0/22
```

Aggregation reduces routing table size and the number of routes advertised to neighbors.

BGP provides additional options to control which component routes are suppressed and how aggregate attributes are constructed.

### Route Refresh

BGP supports **route refresh**, allowing a router to request that a neighbor resend routing information.

This is particularly useful after modifying inbound routing policies.

Route refresh allows new policies to be applied without tearing down the BGP session.

---

## Basic BGP Configuration

The following example demonstrates a basic EBGP relationship between two routers.

### Topology

```text
        AS 65001                    AS 65002

    Lo0: 1.1.1.1/32               Lo0: 2.2.2.2/32
          R1                            R2
          |                             |
        E0/0                          E0/0
     10.0.12.1/24                  10.0.12.2/24
          |                             |
          +-----------------------------+
                      EBGP
```

Both routers will advertise their Loopback0 networks through BGP.

### R1 Configuration

```text
R1(config)# interface Loopback0
R1(config-if)# ip address 1.1.1.1 255.255.255.255

R1(config)# interface Ethernet0/0
R1(config-if)# ip address 10.0.12.1 255.255.255.0
R1(config-if)# no shutdown

R1(config)# router bgp 65001
R1(config-router)# neighbor 10.0.12.2 remote-as 65002
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 activate
R1(config-router-af)# network 1.1.1.1 mask 255.255.255.255
```

### R2 Configuration

```text
R2(config)# interface Loopback0
R2(config-if)# ip address 2.2.2.2 255.255.255.255

R2(config)# interface Ethernet0/0
R2(config-if)# ip address 10.0.12.2 255.255.255.0
R2(config-if)# no shutdown

R2(config)# router bgp 65002
R2(config-router)# neighbor 10.0.12.1 remote-as 65001
R2(config-router)# address-family ipv4 unicast
R2(config-router-af)# neighbor 10.0.12.1 activate
R2(config-router-af)# network 2.2.2.2 mask 255.255.255.255
```

The commands perform three important functions:

1. `router bgp` creates the BGP process and specifies the local ASN.
2. `neighbor remote-as` identifies the BGP peer and its ASN.
3. `network` originates an existing route into BGP.

The `neighbor activate` command explicitly enables the neighbor for the IPv4 unicast address family. IPv4 unicast neighbors are activated by default on IOS XE unless this behavior is disabled with `no bgp default ipv4-unicast`.

**The BGP `network` command requires an exact matching route in the local routing table.**

For example, `network 1.1.1.1 mask 255.255.255.255` requires a route to `1.1.1.1/32` in R1's routing table.

It does not automatically create a route or activate BGP on an interface.

---

## BGP Verification

Several commands are useful for verifying BGP operation.

### Verify Neighbor Relationships

```text
R1# show bgp ipv4 unicast summary
```

This displays BGP neighbor information, including:

- Neighbor IP addresses
- Neighbor ASNs
- BGP session state
- Uptime
- Received prefix counts

For an established session, the `State/PfxRcd` column normally displays a prefix count rather than a BGP state name.

### Verify Detailed Neighbor Information

```text
R1# show bgp ipv4 unicast neighbors 10.0.12.2
```

This displays additional information about the neighbor, including session state, timers, capabilities, and message statistics.

### Verify the BGP Table

```text
R1# show bgp ipv4 unicast
```

This displays routes in the BGP table.

BGP uses several status codes to indicate route status.

For example:

```text
* = Valid
> = Best
i = Learned from IBGP
```

A route marked with `*>` is valid and selected as the best BGP path.

### Verify BGP Routes in the Routing Table

```text
R1# show ip route bgp
```

This displays BGP routes installed in the IPv4 routing table.

A route can be present in the BGP table without being installed in the routing table, for example, if another routing protocol has a route with a better administrative distance.

### Additional Verification Commands

| Command | Purpose |
|---|---|
| `show bgp all summary` | Display BGP neighbor summaries across address families |
| `show bgp ipv4 unicast` | Display the IPv4 unicast BGP table |
| `show bgp ipv4 unicast <prefix>` | Display BGP paths and attributes for a prefix |
| `show bgp ipv4 unicast neighbors` | Display detailed neighbor information |
| `show bgp ipv4 unicast neighbors <IP> advertised-routes` | Display routes advertised to a neighbor |
| `show bgp ipv4 unicast neighbors <IP> routes` | Display accepted routes from a neighbor |
| `show ip route bgp` | Display installed BGP routes |
| `show ip bgp summary` | Display an IPv4 BGP neighbor summary |

---

## Topics Covered in This Section

This section covers the BGP technologies in the CCIE Enterprise Infrastructure blueprint.

| Section | Topic |
|---|---|
| **1.5.a** | IBGP and EBGP peer relations, peer groups, templates, timers, dynamic neighbors, and ASNs |
| **1.5.b** | Path attributes, best path selection, and load balancing |
| **1.5.c** | Routing policies, attribute manipulation, conditional advertisement, ORF, communities, and multihoming |
| **1.5.d** | AS path manipulation, local-as, allowas-in, remove-private-as, prepending, and regular expressions |
| **1.5.e** | Convergence and scalability, route reflectors, aggregation, and AS-SET |
| **1.5.f** | Other BGP features, including soft reconfiguration and route refresh |

The focus is on configuring BGP on Cisco IOS XE, understanding how BGP selects and advertises routes, manipulating routing behavior through policies, and troubleshooting BGP neighbor relationships and route propagation.
