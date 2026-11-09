# IBGP and EBGP Neighbor Configuration

BGP neighbors are normally configured explicitly. Unlike OSPF or EIGRP, BGP does not automatically discover peers on connected interfaces.

Configuring a BGP neighbor involves three main tasks:

1. Start the BGP process and identify the **local ASN**.
2. Identify the **neighbor address** and **remote ASN**.
3. Enable the neighbor for the required **address family**, such as IPv4 or IPv6 unicast.

The local and remote ASNs determine whether the relationship is **IBGP** or **EBGP**.

---

## Basic Neighbor Configuration

The basic IOS XE syntax is:

```text
R1(config)# router bgp <local-asn>
R1(config-router)# neighbor <peer-address> remote-as <remote-asn>
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor <peer-address> activate
```

| Command | Purpose |
|---|---|
| `router bgp <asn>` | Starts the BGP process and sets the local ASN |
| `neighbor <address> remote-as <asn>` | Defines a neighbor and the ASN expected from that neighbor |
| `address-family ipv4 unicast` | Enters the IPv4 unicast address family |
| `neighbor <address> activate` | Enables the neighbor to exchange IPv4 unicast routes |

**The neighbor address is the remote router's peering address**, not a network prefix or the remote router's BGP router ID.

The router ID is a separate 32-bit identifier exchanged in BGP OPEN messages. It does not identify the destination of the TCP session.

### IBGP vs. EBGP

| Configuration | Relationship |
|---|---|
| Local ASN = remote ASN | IBGP |
| Local ASN != remote ASN | EBGP |

For example, on a router in AS 65001:

```text
R1(config)# router bgp 65001
R1(config-router)# neighbor 10.0.12.2 remote-as 65001
R1(config-router)# neighbor 10.0.13.3 remote-as 65002
```

- `10.0.12.2` is an **IBGP** neighbor.
- `10.0.13.3` is an **EBGP** neighbor.

Both sides must configure a compatible relationship. For example, if R1 (AS 65001) expects R2 to be in AS 65002, R2 must identify itself as AS 65002 and expect R1 to be in AS 65001.

BGP uses **TCP port 179**. One peer typically initiates the TCP connection to port 179 on the other peer using an ephemeral source port. The session cannot reach **Established** until TCP connectivity and BGP OPEN negotiation both succeed.

---

## Address-Family Activation

BGP supports multiple **Address Family Identifier (AFI)** and **Subsequent Address Family Identifier (SAFI)** combinations, including IPv4 unicast and IPv6 unicast.

A neighbor can be defined at the BGP process level but enabled for only selected address families.

### Default IPv4 Unicast Activation

On IOS XE, **IPv4 unicast is enabled by default for neighbors created with `neighbor remote-as`** unless this default behavior has been disabled.

For example, with the default setting:

```text
R1(config)# router bgp 65001
R1(config-router)# neighbor 10.0.12.2 remote-as 65002
```

The IPv4 unicast neighbor is activated automatically. An explicit `neighbor 10.0.12.2 activate` is not required in this case.

To disable **automatic IPv4 unicast activation for newly configured neighbors**:

```text
R1(config)# router bgp 65001
R1(config-router)# no bgp default ipv4-unicast
```

Then explicitly activate each IPv4 unicast neighbor:

```text
R1(config-router)# neighbor 10.0.12.2 remote-as 65002
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 activate
```

**Important:** Configure `no bgp default ipv4-unicast` **before** adding neighbors if you want all new neighbors to require explicit IPv4 activation. Applying it afterward does not automatically deactivate previously configured neighbors.

Explicit activation is useful in MP-BGP configurations because it makes the enabled address families unambiguous.

### Deactivating a Neighbor for One Address Family

To stop exchanging IPv4 unicast routes with a specific neighbor:

```text
R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# no neighbor 10.0.12.2 activate
```

This deactivates that neighbor for **IPv4 unicast**, not necessarily for other address families. If no address families remain active, the BGP session may no longer be maintained.

Do not confuse `no neighbor <address> activate` with `neighbor <address> shutdown`, which administratively disables the entire neighbor relationship,
not just route exchange for the specified address family.

---

## Complete IBGP and EBGP Configuration Example

This example uses one IBGP session and one EBGP session.

### Topology

```text
                AS 65001                           AS 65002

    R1                         R2                      R3
  E0/0                       E0/0 E0/1                E0/0
   .1                         .2    .1                  .2
    +--------------------------+     +-------------------+
           10.0.12.0/30                  10.0.23.0/30
               IBGP                          EBGP

                                                  Loopback0:
                                                  192.0.2.1/24
```

| Router | Interface | Address | BGP ASN |
|---|---|---|---|
| R1 | Ethernet0/0 | 10.0.12.1/30 | 65001 |
| R2 | Ethernet0/0 | 10.0.12.2/30 | 65001 |
| R2 | Ethernet0/1 | 10.0.23.1/30 | 65001 |
| R3 | Ethernet0/0 | 10.0.23.2/30 | 65002 |
| R3 | Loopback0 | 192.0.2.1/24 | 65002 |

R3 originates `192.0.2.0/24` into BGP. R2 learns the route through EBGP and advertises it to R1 through IBGP.

The example deliberately disables automatic IPv4 activation, so each neighbor must be activated under `address-family ipv4 unicast`.

### R1 Configuration

```text
R1(config)# interface Ethernet0/0
R1(config-if)# ip address 10.0.12.1 255.255.255.252
R1(config-if)# no shutdown

R1(config)# router bgp 65001
R1(config-router)# bgp router-id 1.1.1.1
R1(config-router)# no bgp default ipv4-unicast
R1(config-router)# neighbor 10.0.12.2 remote-as 65001
R1(config-router)# neighbor 10.0.12.2 description IBGP-to-R2
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 activate
```

R1 and R2 use the same ASN, so their relationship is IBGP.

### R2 Configuration

```text
R2(config)# interface Ethernet0/0
R2(config-if)# ip address 10.0.12.2 255.255.255.252
R2(config-if)# no shutdown

R2(config)# interface Ethernet0/1
R2(config-if)# ip address 10.0.23.1 255.255.255.252
R2(config-if)# no shutdown

R2(config)# router bgp 65001
R2(config-router)# bgp router-id 2.2.2.2
R2(config-router)# no bgp default ipv4-unicast
R2(config-router)# neighbor 10.0.12.1 remote-as 65001
R2(config-router)# neighbor 10.0.12.1 description IBGP-to-R1
R2(config-router)# neighbor 10.0.23.2 remote-as 65002
R2(config-router)# neighbor 10.0.23.2 description EBGP-to-R3
R2(config-router)# address-family ipv4 unicast
R2(config-router-af)# neighbor 10.0.12.1 activate
R2(config-router-af)# neighbor 10.0.12.1 next-hop-self
R2(config-router-af)# neighbor 10.0.23.2 activate
```

R2 has two BGP neighbors:

- `10.0.12.1`: IBGP neighbor in AS 65001.
- `10.0.23.2`: EBGP neighbor in AS 65002.

By default, when R2 advertises an EBGP-learned route to R1 through IBGP, the original BGP NEXT_HOP is normally preserved. Here, `next-hop-self` makes R2 advertise itself (`10.0.12.2`) as the next hop so R1 can reach the destination without an additional route toward the R2-R3 subnet.

See [BGP Next-Hop Handling](next-hop.md) for detailed NEXT_HOP behavior.

### R3 Configuration

```text
R3(config)# interface Ethernet0/0
R3(config-if)# ip address 10.0.23.2 255.255.255.252
R3(config-if)# no shutdown

R3(config)# interface Loopback0
R3(config-if)# ip address 192.0.2.1 255.255.255.0

R3(config)# router bgp 65002
R3(config-router)# bgp router-id 3.3.3.3
R3(config-router)# no bgp default ipv4-unicast
R3(config-router)# neighbor 10.0.23.1 remote-as 65001
R3(config-router)# neighbor 10.0.23.1 description EBGP-to-R2
R3(config-router)# address-family ipv4 unicast
R3(config-router-af)# neighbor 10.0.23.1 activate
R3(config-router-af)# network 192.0.2.0 mask 255.255.255.0
```

The `network` statement originates the exact connected route `192.0.2.0/24` into BGP. It does **not** establish the neighbor relationship.

### Expected Results

After the sessions establish:

1. R3 originates `192.0.2.0/24` into BGP.
2. R3 advertises the route to R2 through EBGP with AS_PATH `65002`.
3. R2 selects the route and can advertise it to R1 through IBGP.
4. R2 advertises NEXT_HOP `10.0.12.2` to R1 because `next-hop-self` is configured.
5. R1 can install the route as an IBGP route if it has no more-preferred route and the next hop resolves successfully.

On R1, the BGP table should identify the route as **IBGP-learned** (`i` in the status column), with AS_PATH `65002` and NEXT_HOP `10.0.12.2`.

The routing table should contain an entry similar to:

```text
B 192.0.2.0/24 [200/0] via 10.0.12.2, ...
```

The administrative distance is **200** because R1 learned the route through IBGP. On R2, the corresponding EBGP route normally has an administrative distance of **20**.

---

## Configuring IPv6 BGP Neighbors

IPv6 unicast is **not** activated by default just because an IPv6 neighbor is configured. Activate the neighbor under `address-family ipv6 unicast`.

The example below assumes a directly connected IPv6 link:

```text
R4 (AS 65004)                     R5 (AS 65005)
2001:DB8:45::4/64                 2001:DB8:45::5/64
     E0/0 -------------------------- E0/0
```

### R4

```text
R4(config)# ipv6 unicast-routing
R4(config)# interface Ethernet0/0
R4(config-if)# ipv6 address 2001:DB8:45::4/64
R4(config-if)# no shutdown

R4(config)# router bgp 65004
R4(config-router)# bgp router-id 4.4.4.4
R4(config-router)# no bgp default ipv4-unicast
R4(config-router)# neighbor 2001:DB8:45::5 remote-as 65005
R4(config-router)# address-family ipv6 unicast
R4(config-router-af)# neighbor 2001:DB8:45::5 activate
```

### R5

```text
R5(config)# ipv6 unicast-routing
R5(config)# interface Ethernet0/0
R5(config-if)# ipv6 address 2001:DB8:45::5/64
R5(config-if)# no shutdown

R5(config)# router bgp 65005
R5(config-router)# bgp router-id 5.5.5.5
R5(config-router)# no bgp default ipv4-unicast
R5(config-router)# neighbor 2001:DB8:45::4 remote-as 65004
R5(config-router)# address-family ipv6 unicast
R5(config-router-af)# neighbor 2001:DB8:45::4 activate
```

Verify the IPv6 BGP sessions:

```text
R4# show bgp ipv6 unicast summary
R4# show bgp ipv6 unicast neighbors 2001:DB8:45::5
```

**BGP router IDs remain 32-bit values written in dotted-decimal notation**, even when the session carries only IPv6 routes.

BGP can also carry IPv6 NLRI over an IPv4 TCP session when the required multiprotocol capabilities and next-hop handling are configured. The TCP transport address family and the NLRI address family are distinct concepts.

---

## Neighbor-Level Configuration Options

Several useful commands apply to individual neighbors at the BGP process level.

### Description and Logging

```text
R1(config)# router bgp 65001
R1(config-router)# neighbor 10.0.12.2 description IBGP-to-R2
R1(config-router)# bgp log-neighbor-changes
```

- `neighbor ... description` adds an operator-defined description.
- `bgp log-neighbor-changes` enables logging of BGP neighbor state changes.

These commands help identify failed sessions during troubleshooting.

### TCP MD5 Authentication

BGP supports TCP MD5 authentication using `neighbor password`:

```text
R1(config-router)# neighbor 10.0.12.2 password LabKey123
```

The other peer must configure the **same password** for the corresponding neighbor. A mismatch prevents the TCP connection from establishing successfully.

This command protects the BGP TCP connection with the TCP MD5 signature option. It is **not** BGP message encryption.

The password is illustrative; use an appropriate unique secret in production.

### Administratively Shutting Down a Neighbor

To disable a BGP neighbor without deleting its configuration:

```text
R1(config-router)# neighbor 10.0.12.2 shutdown
```

To reenable it:

```text
R1(config-router)# no neighbor 10.0.12.2 shutdown
```

This is different from deactivating a single address family. A neighbor-level shutdown prevents the BGP session from being established.

---

## Peering Reachability and Source Addresses

**BGP does not create IP reachability to its neighbors.** The peer address must be reachable through the existing routing table, and the peer must be able to return traffic to the source address used for the TCP connection.

For a directly connected session, the router normally uses the outgoing interface's IP address as the TCP source address.

```text
R1# show ip route 10.0.12.2
R1# ping 10.0.12.2 source 10.0.12.1
```

IBGP neighbors do not need to be directly connected. Loopback-based peering is common because it makes the BGP session less dependent on a single physical interface, provided the underlying network can route between the loopback addresses.

For loopback-based sessions, configure `neighbor ... update-source` on the appropriate routers. For EBGP sessions that span multiple IP hops, additional configuration is generally required because of the default EBGP TTL and directly connected peer checks.

See [BGP Update Source and Multihop Peering](multihop.md) for configuration and detailed behavior.

---

## Verifying Neighbor Relationships

### Neighbor Summary

```text
R2# show bgp ipv4 unicast summary
```

Important fields include:

| Field | Meaning |
|---|---|
| `Neighbor` | Remote peering address |
| `V` | BGP protocol version, normally 4 |
| `AS` | Neighbor's ASN |
| `MsgRcvd` / `MsgSent` | BGP messages received and sent |
| `Up/Down` | Session uptime or duration of the current state |
| `State/PfxRcd` | Session state or number of accepted prefixes |

When a session is **Established**, `State/PfxRcd` normally shows a numeric prefix count rather than the word `Established`.

**A value of `0` does not mean the BGP session is down.** It means no prefixes are currently counted as received for that address family.

### Detailed Neighbor Information

```text
R2# show bgp ipv4 unicast neighbors 10.0.12.1
R2# show bgp ipv4 unicast neighbors 10.0.23.2
```

Use this output to verify:

- **BGP state** and how long the session has been Established.
- **Local and remote ASNs**.
- **Local and remote BGP router IDs**.
- **Local and foreign TCP addresses and ports**.
- **Negotiated hold and keepalive timers**.
- **Negotiated address-family capabilities**.
- **Last reset reason**, if the neighbor has recently gone down.

### BGP Routes and Advertisements

```text
R2# show bgp ipv4 unicast
R2# show bgp ipv4 unicast 192.0.2.0/24
R2# show bgp ipv4 unicast neighbors 10.0.23.2 routes
R2# show bgp ipv4 unicast neighbors 10.0.12.1 advertised-routes
R1# show ip route 192.0.2.0
```

| Command | Purpose |
|---|---|
| `show bgp ipv4 unicast` | Display paths in the local BGP table |
| `show bgp ipv4 unicast <prefix>` | Inspect the paths and attributes for one prefix |
| `show bgp ipv4 unicast neighbors <peer> routes` | Display routes accepted from the neighbor |
| `show bgp ipv4 unicast neighbors <peer> advertised-routes` | Display routes advertised to the neighbor |
| `show ip route <prefix>` | Check whether the prefix was installed in the IP routing table |

**An Established session does not guarantee that any routes will be exchanged or installed.** Address-family activation, route origination, policy, best-path selection, next-hop reachability, and RIB selection are separate checks.

---

## Common Neighbor Configuration Problems

| Symptom | Likely cause | What to check |
|---|---|---|
| Neighbor remains in `Idle`, `Connect`, or `Active` | Missing IP reachability, TCP filtering, session disabled, or source-address mismatch | `show ip route`, sourced ping, neighbor configuration, TCP/179 filtering |
| TCP reaches the peer but BGP resets | ASN mismatch, unacceptable OPEN parameters, or another negotiation error | `show bgp ... neighbors`, logs and last reset reason |
| Neighbor is `Idle (Admin)` | Administratively shut down | `neighbor ... shutdown` |
| Neighbor exists but is missing from an AF summary | Address family not activated | `neighbor ... activate` under the intended AF |
| IPv6 unicast routes are not exchanged | IPv6 AF not activated or required capability unavailable | `show bgp ipv6 unicast summary`, neighbor capabilities |
| Session is Established but `State/PfxRcd` is `0` | No routes advertised, routes filtered, or route origination issue | BGP tables, advertised routes, inbound policy |
| BGP route is present but not valid | NEXT_HOP cannot be resolved, or another validity condition fails | `show bgp ipv4 unicast <prefix>`, `show ip route <next-hop>` |
| BGP best path is valid but absent from the IP RIB | Another route is preferred for installation or RIB-related restriction | `show ip route <prefix>`, detailed BGP output |

For session establishment details and state transitions, see [BGP Messages and Neighbor States](../messages-states.md).

---

## Command Summary

| Command | Purpose |
|---|---|
| `router bgp <asn>` | Configure the local BGP process |
| `bgp router-id <router-id>` | Set the BGP router ID |
| `no bgp default ipv4-unicast` | Disable automatic IPv4 activation for new peers |
| `neighbor <peer> remote-as <asn>` | Define an IBGP or EBGP neighbor |
| `neighbor <peer> description <text>` | Label a neighbor |
| `neighbor <peer> password <secret>` | Enable TCP MD5 authentication |
| `neighbor <peer> shutdown` | Administratively disable a neighbor |
| `no neighbor <peer> shutdown` | Reenable a neighbor |
| `address-family ipv4 unicast` | Enter IPv4 unicast AF configuration |
| `address-family ipv6 unicast` | Enter IPv6 unicast AF configuration |
| `neighbor <peer> activate` | Activate a neighbor for the selected AF |
| `no neighbor <peer> activate` | Deactivate a neighbor for the selected AF |
| `neighbor <peer> next-hop-self` | Advertise a local next hop to that peer for applicable routes |
| `show bgp ipv4 unicast summary` | Verify IPv4 unicast sessions |
| `show bgp ipv6 unicast summary` | Verify IPv6 unicast sessions |
| `show bgp ipv4 unicast neighbors <peer>` | Inspect detailed session information |
| `show bgp ipv4 unicast neighbors <peer> advertised-routes` | Inspect advertised routes |

---

## Related Topics

- [BGP Next-Hop Handling](next-hop.md)
- [BGP Update Source and Multihop Peering](multihop.md)
- [Peer Groups and Templates](peer-groups-templates/overview.md)
- [BGP Active and Passive](active-passive.md)
- [BGP Timers](timers.md)
- [BGP Route Advertisement and Network Statements](../route-advertisement.md)

## Cisco Documentation

- [Configuring a Basic BGP Network, Cisco IOS XE 17.x](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-routing/b-ip-routing/m_irg-basic-net-0.html)
- [Configuring BGP Neighbor Session Options, Cisco IOS XE 17.x](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-routing/b-ip-routing/m_irg-neighbor-0.html)
- [IPv6 Routing: Multiprotocol BGP Extensions for IPv6, Cisco IOS XE 17.x](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-routing/b-ip-routing/m_ip6-mbgp-ext-xe.html)
- [Cisco IOS IP Routing BGP Command Reference: Neighbor Commands](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_bgp/command/irg-cr-book/bgp-m1.html)
