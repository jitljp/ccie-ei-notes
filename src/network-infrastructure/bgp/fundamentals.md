
# BGP Fundamentals and Operation

**Border Gateway Protocol (BGP)** is a **path-vector routing protocol** that exchanges routing information between BGP peers.

BGP is primarily designed for **interdomain routing**, allowing independently administered networks to exchange reachability information while applying their own routing policies.

BGP differs from IGPs such as OSPF and EIGRP in several important ways:

- BGP uses **path attributes** rather than a single routing metric to select routes.
- BGP establishes neighbor relationships using **TCP port 179**.
- BGP does not automatically discover neighbors. Peers are normally configured manually.
- BGP does not exchange a complete network topology.
- BGP advertises routing information incrementally rather than periodically advertising its entire routing table.
- BGP supports extensive policy controls over route acceptance, selection, and advertisement.

BGP version 4 (BGP-4) is the version used in modern networks.

---

## BGP as a Path-Vector Protocol

BGP is classified as a **path-vector protocol** because routes carry information about the autonomous systems through which they have been advertised.

The most important attribute for understanding path-vector operation is **AS_PATH**.

AS_PATH records the autonomous systems traversed by a BGP route.

Consider the following network:

```text
        AS 65001           AS 65002           AS 65003

          R1 --------------- R2 --------------- R3
                                                |
                                           10.3.3.0/24
```

R3 originates `10.3.3.0/24` into BGP.

When R3 advertises the route to R2 through EBGP, it adds its ASN to the AS_PATH.

```text
R2 receives:

Network: 10.3.3.0/24
AS_PATH: 65003
```

When R2 advertises the route to R1 through EBGP, it adds its own ASN to the beginning of the AS_PATH.

```text
R1 receives:

Network: 10.3.3.0/24
AS_PATH: 65002 65003
```

The leftmost ASN is the most recently prepended ASN.

The rightmost ASN generally identifies the originating AS.

### AS_PATH Loop Prevention

The AS_PATH attribute helps BGP prevent routing loops between autonomous systems.

By default, an EBGP router rejects a route if its own ASN appears in the received AS_PATH.

For example, suppose AS 65003 receives an advertisement containing:

```text
AS_PATH: 65002 65001 65003
```

The local ASN, 65003, already appears in the path.

The router rejects the route because accepting it could create an inter-AS routing loop.

This behavior can be modified using features such as `allowas-in`, which are covered later.

> AS_PATH describes the sequence of autonomous systems, not the sequence of individual routers. BGP does not maintain a complete topology of the network.

---

## BGP Routing Domains

BGP operates both between autonomous systems and within an autonomous system.

### External BGP (EBGP)

**EBGP** exchanges routes between routers belonging to different autonomous systems.

```text
       AS 65001              AS 65002

          R1 ------------------ R2
                   EBGP
```

An EBGP relationship is established when the peers use different ASNs.

```text
R1:
Local AS  = 65001
Remote AS = 65002

R2:
Local AS  = 65002
Remote AS = 65001
```

EBGP is commonly used to exchange routes between enterprises, service providers, and other networks.

### Internal BGP (IBGP)

**IBGP** exchanges routes between routers belonging to the same autonomous system.

```text
                 AS 65001

          R1 ------------------ R2
                   IBGP
```

An IBGP relationship is established when both peers use the same ASN.

IBGP is commonly used to distribute externally learned BGP routes throughout an AS.

Unlike EBGP, IBGP does not prepend the local ASN to the AS_PATH when advertising routes to other IBGP peers.

### IBGP Advertisement Restriction

An important rule is that **routes learned from one IBGP peer are not advertised to another IBGP peer by default**.

For example:

```text
                 AS 65001

          R1 -------- R2 -------- R3
               IBGP        IBGP
```

Suppose R1 advertises `10.1.1.0/24` to R2.

R2 learns the route through IBGP but does not advertise it to R3.

Therefore, R3 does not learn the route through R2.

This restriction prevents certain routing loops within an AS.

Two common solutions are:

- **Full-mesh IBGP:** Every IBGP router establishes a session with every other IBGP router.
- **Route reflectors:** Designated routers reflect IBGP-learned routes according to route-reflection rules.

Route reflectors are covered in the convergence and scalability section.

---

## BGP Neighbor Relationships

BGP routers exchange routing information through **BGP sessions** established over TCP.

A BGP neighbor relationship is also called a **peering session**.

Unlike OSPF and EIGRP, BGP does not use multicast Hello messages to discover neighbors.

BGP neighbors are normally configured explicitly.

For example:

```text
R1(config)# router bgp 65001
R1(config-router)# neighbor 10.0.12.2 remote-as 65002
```

The `neighbor` command identifies the peer's IP address and remote ASN.

### TCP Transport

BGP uses:

```text
Transport protocol: TCP
Well-known port:    179
```

Typically, one peer establishes a TCP connection using an ephemeral source port and destination port 179.

After TCP connectivity is established, the routers exchange BGP messages to negotiate and establish the session.

```text
R1                                R2
 |                                 |
 |------ TCP connection ---------->|
 |                                 |
 |<------- BGP OPEN -------------->|
 |                                 |
 |<------- BGP KEEPALIVE --------->|
 |                                 |
 |       BGP Established           |
 |                                 |
 |<-------- BGP UPDATE ----------->|
```

BGP uses TCP for reliable, ordered message delivery.

However, TCP connectivity alone does not guarantee that a BGP session will establish.

The peers must also successfully negotiate BGP parameters, including their ASNs and supported capabilities.

### Directly Connected vs. Multihop Peering

By default:

| Parameter | EBGP | IBGP |
|---|---|---|
| TCP TTL | 1 | 255 |
| Directly connected peers required | Normally, unless configured otherwise | No |
| Common peering address | Physical interface or loopback | Often loopback |
| Remote IP reachability required | Yes | Yes |

IBGP peers can be multiple router hops apart, provided there is IP connectivity between their peering addresses.

EBGP can also operate across multiple hops using additional configuration.

The IP packets carrying BGP messages are forwarded using normal IP routing.

> BGP peering and BGP route forwarding are separate concepts. A router can establish a BGP session through an IGP-learned route without using BGP to reach the neighbor.

---

## BGP Control Plane and Data Plane

BGP exchanges routing information in the **control plane**.

It does not directly forward user traffic.

The forwarding process relies on the router's routing and forwarding tables.

Conceptually:

```text
       BGP Neighbor
             |
             | BGP UPDATE
             v
       BGP Routing Table
             |
             | Best-path selection
             v
       IP Routing Table
             |
             | Route installation
             v
    Forwarding Information Base
             |
             v
       Packet Forwarding
```

These tables have different responsibilities.

| Table | Purpose |
|---|---|
| **BGP table** | Stores BGP paths and associated attributes |
| **IP routing table (RIB)** | Contains selected routes from routing protocols and other sources |
| **Forwarding Information Base (FIB)** | Contains forwarding information used to forward packets |

### BGP Best Path vs. Routing Table Installation

BGP can learn multiple paths to the same destination.

It evaluates their attributes and selects the best BGP path.

However, **the best BGP path is not necessarily installed in the IP routing table**.

For example, suppose R1 learns `192.0.2.0/24` through:

```text
OSPF:
Administrative distance = 110

IBGP:
Administrative distance = 200
```

If both routes are eligible for installation, OSPF is preferred because it has the lower administrative distance.

The IBGP route can remain in the BGP table as the best BGP path without becoming the active route in the IP routing table.

This is an important distinction:

- **BGP best-path selection** compares BGP paths.
- **RIB selection** determines which routing source supplies the route used for forwarding.

---

## BGP Routing Information Bases

BGP defines several logical routing information bases that describe how routes are received, selected, and advertised.

| Routing Information Base | Purpose |
|---|---|
| **Adj-RIB-In** | Routing information received from a BGP peer |
| **Loc-RIB** | Locally selected BGP routing information |
| **Adj-RIB-Out** | Routing information selected for advertisement to a peer |

The conceptual process is:

                  BGP Peer
                      |
                      v
                 Adj-RIB-In
                      |
                      v
                 Import policy
                      |
                      v
             BGP Table (candidates)
                      |
                      v
             Best-path selection
                      |
                      v
              Selected BGP Path
                      |
            +---------+---------+
            |                   |
            v                   v
       IP Routing          Export policy
          Table                 |
                                v
                            Adj-RIB-Out
                                |
                                v
                             BGP Peer

Inbound routing policies can reject routes or modify their attributes before they are considered for best-path selection.

Outbound policies determine which routes are advertised and may modify attributes in those advertisements.

These RIBs are logical components of BGP operation. IOS XE does not necessarily maintain separate, complete physical tables corresponding to every conceptual RIB.

For example, retaining an unmodified copy of all received routes for later policy reprocessing may require inbound soft reconfiguration when route refresh is unavailable.

---

## BGP Route Advertisement

BGP uses **UPDATE messages** to advertise and withdraw routes.

An UPDATE message can carry:

- Network Layer Reachability Information (**NLRI**)
- Path attributes
- Withdrawn routes

**NLRI** identifies the destinations being advertised.

For IPv4 unicast, NLRI commonly represents IP prefixes such as:

```text
10.1.1.0/24
172.16.0.0/16
192.168.1.0/24
```

Path attributes describe properties of the advertised routes.

For example:

```text
NLRI:       192.168.1.0/24
AS_PATH:    65002 65003
NEXT_HOP:   10.0.12.2
ORIGIN:     IGP
```

### Initial Route Advertisement

When a BGP session becomes Established, peers exchange their permitted routing information.

The routes advertised depend on the applicable BGP address family and outbound routing policies.

### Incremental Updates

After the initial exchange, BGP does not periodically resend the entire routing table.

Instead, it sends updates when routing information changes.

For example:

```text
New route becomes available
          |
          v
BGP selects the route
          |
          v
BGP advertises the route
```

If a previously advertised route becomes unavailable:

```text
Route becomes unavailable
          |
          v
BGP selects an alternative
or withdraws the route
          |
          v
Neighbor receives an UPDATE
```

An UPDATE may withdraw the old path, advertise a replacement, or both, depending on the situation.

BGP KEEPALIVE messages are exchanged periodically to maintain the session, but they do not contain routing updates.

---

## BGP Next-Hop Reachability

Every BGP route has a **NEXT_HOP** attribute.

The next hop identifies the IP address that must be reachable to forward traffic toward the advertised destination.

For example:

```text
BGP Prefix: 192.168.1.0/24
NEXT_HOP:   10.0.12.2
```

R1 must have a usable route to `10.0.12.2` for this BGP path to be considered valid.

### Recursive Next-Hop Resolution

A BGP next hop does not have to be directly connected.

Suppose R1 learns:

```text
BGP destination: 192.168.1.0/24
BGP next hop:    10.0.23.3
```

R1's routing table contains:

```text
10.0.23.3/32 via 10.0.12.2, Ethernet0/0
```

R1 can recursively resolve the BGP next hop through the routing table.

```text
Destination: 192.168.1.0/24
           |
           v
BGP next hop: 10.0.23.3
           |
           v
Route to 10.0.23.3
           |
           v
Next hop: 10.0.12.2
           |
           v
Outgoing interface: Ethernet0/0
```

If the BGP next hop becomes unreachable, the BGP path is no longer eligible for best-path selection.

This is one reason an underlying IGP is important in IBGP networks.

The IGP often provides reachability to BGP next hops and loopback addresses used for peering.

---

## BGP Router ID

Each BGP speaker has a **BGP Router ID**, a 32-bit value written in dotted-decimal notation.

For example:

```text
1.1.1.1
```

The Router ID identifies the BGP speaker and is used in protocol operations, including best-path tie-breaking and route-reflector loop prevention.

It should be unique within the BGP routing domain.

### Router ID Selection

On Cisco IOS XE, the BGP Router ID is selected using the following order:

1. Manually configured BGP Router ID
2. Highest IPv4 address on an up loopback interface
3. Highest IPv4 address on an up non-loopback interface

Configure it manually:

```text
R1(config)# router bgp 65001
R1(config-router)# bgp router-id 1.1.1.1
```

Verify the BGP Router ID:

```text
R1# show bgp ipv4 unicast summary
```

The BGP Router ID does not need to be reachable by BGP peers.

It is a protocol identifier, not necessarily the IP address used to establish BGP sessions.

---

## Multiprotocol BGP

BGP-4 was originally designed to exchange IPv4 unicast reachability information.

**Multiprotocol BGP (MP-BGP)** extends BGP to support additional address families.

MP-BGP identifies routing information using:

- **Address Family Identifier (AFI):** Identifies the network-layer protocol.
- **Subsequent Address Family Identifier (SAFI):** Identifies the type of routing information within that address family.

Common examples include:

| Address Family | AFI | SAFI |
|---|---|---|
| IPv4 unicast | 1 | 1 |
| IPv6 unicast | 2 | 1 |
| IPv4 multicast | 1 | 2 |
| IPv4 MPLS VPN (VPNv4) | 1 | 128 |
| IPv6 MPLS VPN (VPNv6) | 2 | 128 |

MP-BGP uses negotiated capabilities and multiprotocol UPDATE attributes to exchange routes for these address families.

### Address Families in IOS XE

A single BGP process can support multiple address families.

For example:

```text
R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# exit-address-family
R1(config-router)# address-family ipv6 unicast
R1(config-router-af)# exit-address-family
```

Each address family has its own routing information and address-family-specific configuration.

A BGP session can exchange routing information for multiple negotiated address families over the same TCP connection.

MP-BGP is also used for MPLS Layer 3 VPNs, which are covered separately in the MPLS section.

---

## BGP and IGP Comparison

BGP is designed for a different purpose from typical interior routing protocols.

| Characteristic | BGP | OSPF | EIGRP |
|---|---|---|---|
| Protocol type | Path vector | Link state | Advanced distance vector |
| Primary use | Interdomain and policy-based routing | Intradomain routing | Intradomain routing |
| Transport | TCP 179 | IP protocol 89 | IP protocol 88 |
| Automatic neighbor discovery | No, except dynamic-neighbor features | Yes | Yes |
| Path selection | BGP attributes | SPF cost | EIGRP metric |
| Complete topology database | No | Yes, within each area | No |
| Loop prevention | AS_PATH, IBGP rules, and other mechanisms | SPF and LSDB consistency | DUAL and feasibility rules |
| Incremental routing updates | Yes | Yes, after initial synchronization | Yes, after initial synchronization |

BGP is particularly useful when route selection must follow administrative policies rather than simply choosing the lowest-cost path.

IGPs and BGP are frequently used together.

For example:

```text
                  AS 65001
        +--------------------------+
        |                          |
        |   R1 ---- R2 ---- R3     |
        |                          |
        |    OSPF for internal     |
        |    reachability          |
        |                          |
        |    IBGP for external     |
        |    routing information   |
        |                          |
        +--------------------------+
              |            |
             EBGP         EBGP
              |            |
            ISP 1        ISP 2
```

In this design:

- OSPF provides internal reachability between routers and BGP next hops.
- IBGP distributes external routing information within AS 65001.
- EBGP exchanges routes with the two ISPs.
- BGP policies determine which ISP is preferred for different destinations.

---

## Verification Commands

The following commands are useful for examining the fundamental operation of BGP.

| Command | Purpose |
|---|---|
| `show bgp ipv4 unicast summary` | Verify BGP Router ID, local ASN, and neighbor states |
| `show bgp ipv4 unicast` | Display the IPv4 BGP table |
| `show bgp ipv4 unicast <prefix>` | Display paths and attributes for a prefix |
| `show bgp ipv4 unicast neighbors` | Examine BGP peering information |
| `show ip route bgp` | Display BGP routes installed in the IP routing table |
| `show ip route <next-hop>` | Verify reachability to a BGP next hop |
| `show bgp ipv6 unicast` | Display the IPv6 unicast BGP table |

### BGP Table Status Codes

The following codes are particularly important:

| Code | Meaning |
|---|---|
| `*` | Valid BGP path |
| `>` | Best BGP path |
| `i` (prefix status) | Route learned through IBGP |
| `i` (origin code) | IGP origin |
| `e` (origin code) | EGP origin |
| `?` (origin code) | Incomplete origin |
| `r` | RIB failure |

For example:

```text
*>  10.1.1.0/24
*   10.1.1.0/24
```

Both paths are valid, but only the first path is selected as the best BGP path.

The best path is the one BGP normally attempts to install in the routing table.

---

## Key Points

- BGP is a path-vector routing protocol that uses TCP port 179.
- BGP is primarily designed for policy-based routing between autonomous systems.
- EBGP exchanges routes between different ASNs, while IBGP exchanges routes within an AS.
- AS_PATH helps prevent inter-AS routing loops.
- IBGP-learned routes are not advertised to other IBGP peers by default.
- BGP does not automatically discover neighbors under normal static peering configuration.
- BGP advertises routes and withdrawals incrementally using UPDATE messages.
- BGP uses path attributes to select the best path.
- A BGP route requires a reachable next hop to be eligible for best-path selection.
- The best BGP path is not necessarily installed in the IP routing table.
- MP-BGP supports multiple address families, including IPv4, IPv6, VPNv4, and VPNv6.
- BGP and IGPs commonly operate together, with the IGP providing internal reachability and BGP controlling routing policies.
