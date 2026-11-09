# BGP Update Source and Multihop Peering

BGP neighbors establish a **TCP session** between two IP addresses. A BGP session does not have to use the physical interface addresses on the link between the routers.

Cisco IOS XE provides several commands that control how BGP sessions are established:

| Command | Purpose |
|---|---|
| `neighbor <address> update-source <interface>` | Select the local interface address used as the source of the BGP TCP connection |
| `neighbor <address> ebgp-multihop <ttl>` | Permit EBGP peering with a non-directly connected neighbor and set the outgoing TTL |
| `neighbor <address> disable-connected-check` | Bypass the directly connected neighbor check for single-hop EBGP, typically when peering with loopbacks |
| `neighbor <address> ttl-security hops <count>` | Enable EBGP TTL security and permit peers up to the specified hop count away |

These commands control different aspects of **BGP transport**. They do not, by themselves, originate prefixes into BGP or make the BGP NEXT_HOP attribute reachable.

---

## BGP Peering Addresses and TCP Source Addresses

When configuring a BGP neighbor, the neighbor IP address identifies the **destination address** used to establish the TCP session.

For example:

```text
R1(config)# router bgp 65001
R1(config-router)# neighbor 10.0.12.2 remote-as 65002
```

R1 tries to establish a TCP session with destination `10.0.12.2`, using TCP port 179.

Without `update-source`, IOS XE normally chooses the address of the outgoing interface used to reach the neighbor as its local TCP source address.

```text
         AS 65001                             AS 65002

           R1 ---------------------------------- R2
         E0/0                                  E0/0
     10.0.12.1/30                          10.0.12.2/30

            TCP source: 10.0.12.1
       TCP destination: 10.0.12.2
```

R2 typically has a corresponding neighbor definition:

```text
R2(config)# router bgp 65002
R2(config-router)# neighbor 10.0.12.1 remote-as 65001
```

The TCP source and destination addresses must match the peer addresses expected on the opposite router. A mismatch can prevent the session from reaching **Established**, even if the routers can ping each other.

### `neighbor update-source`

The `update-source` command changes the local source interface used for the BGP TCP connection:

```text
R1(config-router)# neighbor 2.2.2.2 update-source Loopback0
```

If R1's Loopback0 address is `1.1.1.1`, packets that R1 initiates toward neighbor `2.2.2.2` are sourced from `1.1.1.1`.

The command does **not**:

- Change the remote neighbor address in `neighbor <address> remote-as`.
- Create a route to the remote neighbor.
- Increase the EBGP TTL.
- Automatically change the NEXT_HOP attribute of every route advertised over the session.

**The BGP TCP source address and the BGP NEXT_HOP path attribute are separate concepts.** 

However, when updating the NEXT_HOP of paths advertised to a neighbor, the TCP source address (peering address) is typically used. See [BGP Next-Hop Handling](next-hop.md).

---

## IBGP Peering Using Loopback Interfaces

IBGP neighbors frequently establish sessions using **loopback addresses** instead of physical interface addresses.

A loopback is useful because it can remain operational when one physical link fails, provided the underlay network still has a route to the loopback through another path.

### Topology

```text
                        AS 65001

        R1 ---------------- R2 ---------------- R3
        E0/0              E0/0 E0/1             E0/0
      10.0.12.1/30     .2       .2          10.0.23.3/30
                       10.0.12.0/30        10.0.23.0/30

       Lo0: 1.1.1.1/32                    Lo0: 3.3.3.3/32

             <-------- IBGP over loopbacks -------->
                 (OSPF provides IP reachability)
```

R1 and R3 are **not directly connected**. R2 routes between them, but R2 does not need to participate in their BGP session.

### Underlay Configuration

Configure the interface addresses and enable OSPF to advertise the transit networks and loopbacks.

**R1:**

```text
R1(config)# interface Ethernet0/0
R1(config-if)# ip address 10.0.12.1 255.255.255.252
R1(config-if)# ip ospf 1 area 0
R1(config-if)# no shutdown
R1(config)# interface Loopback0
R1(config-if)# ip address 1.1.1.1 255.255.255.255
R1(config-if)# ip ospf 1 area 0
```

**R2:**

```text
R2(config)# interface Ethernet0/0
R2(config-if)# ip address 10.0.12.2 255.255.255.252
R2(config-if)# ip ospf 1 area 0
R2(config-if)# no shutdown
R2(config)# interface Ethernet0/1
R2(config-if)# ip address 10.0.23.2 255.255.255.252
R2(config-if)# ip ospf 1 area 0
R2(config-if)# no shutdown
```

**R3:**

```text
R3(config)# interface Ethernet0/0
R3(config-if)# ip address 10.0.23.3 255.255.255.252
R3(config-if)# ip ospf 1 area 0
R3(config-if)# no shutdown
R3(config)# interface Loopback0
R3(config-if)# ip address 3.3.3.3 255.255.255.255
R3(config-if)# ip ospf 1 area 0
```

Verify the remote loopback routes before configuring BGP:

```text
R1# show ip route 3.3.3.3
R1# ping 3.3.3.3 source Loopback0
R3# show ip route 1.1.1.1
R3# ping 1.1.1.1 source Loopback0
```

### BGP Configuration

**R1:**

```text
R1(config)# router bgp 65001
R1(config-router)# bgp router-id 1.1.1.1
R1(config-router)# neighbor 3.3.3.3 remote-as 65001
R1(config-router)# neighbor 3.3.3.3 update-source Loopback0
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 3.3.3.3 activate
```

**R3:**

```text
R3(config)# router bgp 65001
R3(config-router)# bgp router-id 3.3.3.3
R3(config-router)# neighbor 1.1.1.1 remote-as 65001
R3(config-router)# neighbor 1.1.1.1 update-source Loopback0
R3(config-router)# address-family ipv4 unicast
R3(config-router-af)# neighbor 1.1.1.1 activate
```

The TCP connection uses:

| Parameter | R1 to R3 | R3 to R1 |
|---|---|---|
| Neighbor address | `3.3.3.3` | `1.1.1.1` |
| TCP source address | `1.1.1.1` | `3.3.3.3` |
| TCP destination address | `3.3.3.3` | `1.1.1.1` |
| Default outgoing TTL | 255 | 255 |
| EBGP multihop required? | No | No |

**IBGP uses an outgoing TTL of 255 by default**, so `ebgp-multihop` is neither required nor appropriate for this IBGP session.

### Why `update-source` Is Needed

Without `update-source`, R1 would normally select `10.0.12.1` as the TCP source when connecting to `3.3.3.3`.

However, R3 has configured `1.1.1.1` as its neighbor, not `10.0.12.1`.

```text
R1's attempted session without update-source:

Source:      10.0.12.1
Destination: 3.3.3.3

R3's configured neighbor: 1.1.1.1

Result: The source does not match the expected neighbor.
```

Configuring `update-source Loopback0` on both peers ensures the TCP endpoints use the expected addresses.

> Configuring `bgp router-id 1.1.1.1` does **not** automatically make BGP use `1.1.1.1` as the TCP source. The router ID and update source are independent settings.

---

## EBGP Default Single-Hop Behavior

EBGP is designed to peer with **directly connected neighbors** by default.

IOS XE normally applies two relevant restrictions:

1. The EBGP neighbor must pass a directly connected neighbor check.
2. The outgoing IP TTL is **1**.

These are related, but distinct.

A TTL of 1 is sufficient to reach a BGP peer on the directly attached router because the packet does not need to be forwarded by an intermediate router.

For example:

```text
        AS 65001                       AS 65002

          R1 ---------------------------- R2
        E0/0                            E0/0
     10.0.12.1/30                    10.0.12.2/30
```

```text
R1(config)# router bgp 65001
R1(config-router)# neighbor 10.0.12.2 remote-as 65002

R2(config)# router bgp 65002
R2(config-router)# neighbor 10.0.12.1 remote-as 65001
```

Neither `update-source` nor `ebgp-multihop` is needed when using these directly connected interface addresses.

---

## Single-Hop EBGP Using Loopbacks

It is possible for two **physically adjacent EBGP routers** to establish their BGP session using loopbacks instead of the connected interface addresses.

This is still a **single-hop** topology, although the neighbor loopback address does not belong to the directly connected subnet.

### Topology

```text
        AS 65001                       AS 65002

          R1 ---------------------------- R2
        E0/0                            E0/0
     10.0.12.1/30                    10.0.12.2/30

     Lo0: 1.1.1.1/32                 Lo0: 2.2.2.2/32

               EBGP over loopbacks
```

The routers need routes to each other's loopback addresses.

**R1:**

```text
R1(config)# ip route 2.2.2.2 255.255.255.255 10.0.12.2
R1(config)# router bgp 65001
R1(config-router)# neighbor 2.2.2.2 remote-as 65002
R1(config-router)# neighbor 2.2.2.2 update-source Loopback0
R1(config-router)# neighbor 2.2.2.2 disable-connected-check
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 2.2.2.2 activate
```

**R2:**

```text
R2(config)# ip route 1.1.1.1 255.255.255.255 10.0.12.1
R2(config)# router bgp 65002
R2(config-router)# neighbor 1.1.1.1 remote-as 65001
R2(config-router)# neighbor 1.1.1.1 update-source Loopback0
R2(config-router)# neighbor 1.1.1.1 disable-connected-check
R2(config-router)# address-family ipv4 unicast
R2(config-router-af)# neighbor 1.1.1.1 activate
```

Assume both physical interfaces and loopbacks are configured as shown.

### `neighbor disable-connected-check`

The command:

```text
R1(config-router)# neighbor 2.2.2.2 disable-connected-check
```

**Disables the direct-connection verification** for that EBGP neighbor. It does not increase the TTL.

With a single physical hop, the default EBGP TTL of 1 can still reach the other router's loopback.

This is a narrower alternative to changing the session's TTL to 2 or more simply to bypass the connected-neighbor check.

An alternative is:

```text
R1(config-router)# neighbor 2.2.2.2 ebgp-multihop 2
```

With this alternative, the EBGP neighbor may be more than one IP hop away and the outgoing TTL is 2. For an ordinary single-hop loopback EBGP session, `disable-connected-check` avoids increasing the TTL unnecessarily.

> `disable-connected-check` does **not** make a default TTL-1 EBGP packet traverse an intermediate router. If an actual transit router forwards the packet, the TTL must be increased or TTL security must be configured instead.

---

## Multihop EBGP Peering

**Multihop EBGP** allows external BGP neighbors to establish a session even when one or more intermediate routers forward packets between them.

### Topology

```text
     AS 65001               Transit                AS 65003

        R1 ------------------- R2 -------------------- R3
       E0/0                 E0/0 E0/1                 E0/0
    10.0.12.1/30       10.0.12.2 10.0.23.2      10.0.23.3/30

    Lo0: 1.1.1.1/32                            Lo0: 3.3.3.3/32

       <----------- Multihop EBGP session ----------->
                   Two IP hops (R1-R2-R3)
```

R2 acts only as an IP transit router; it does not need to run BGP.

### Provide IP Reachability First

This example uses static routes so that both BGP peers can reach the other loopback before BGP is established.

**R1:**

```text
R1(config)# ip route 3.3.3.3 255.255.255.255 10.0.12.2
```

**R2:**

```text
R2(config)# ip route 1.1.1.1 255.255.255.255 10.0.12.1
R2(config)# ip route 3.3.3.3 255.255.255.255 10.0.23.3
```

**R3:**

```text
R3(config)# ip route 1.1.1.1 255.255.255.255 10.0.23.2
```

The interfaces and loopbacks must also be configured with the addresses in the topology.

Test with sourced pings:

```text
R1# ping 3.3.3.3 source Loopback0
R3# ping 1.1.1.1 source Loopback0
```

An IGP can provide this reachability instead of static routes.

### Configure `ebgp-multihop`

**R1:**

```text
R1(config)# router bgp 65001
R1(config-router)# neighbor 3.3.3.3 remote-as 65003
R1(config-router)# neighbor 3.3.3.3 update-source Loopback0
R1(config-router)# neighbor 3.3.3.3 ebgp-multihop 2
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 3.3.3.3 activate
```

**R3:**

```text
R3(config)# router bgp 65003
R3(config-router)# neighbor 1.1.1.1 remote-as 65001
R3(config-router)# neighbor 1.1.1.1 update-source Loopback0
R3(config-router)# neighbor 1.1.1.1 ebgp-multihop 2
R3(config-router)# address-family ipv4 unicast
R3(config-router-af)# neighbor 1.1.1.1 activate
```

The command:

```text
R1(config-router)# neighbor 3.3.3.3 ebgp-multihop 2
```

Permits a multihop EBGP session and sets the outgoing BGP packet TTL to 2.

A packet sent from R1 toward R3 travels as follows:

```text
R1 sends:               TTL = 2
R2 forwards packet:     TTL = 1
R3 receives packet:     TTL = 1
```

The TTL needs to remain greater than zero while the packet is being forwarded. With an initial TTL of 1, R2 would discard the packet instead of forwarding it to R3.

**Configure multihop settings in both directions** when both peers need to initiate connections over the multihop path.

### Important TTL Details

| Neighbor relationship | Default outgoing TTL | Directly connected check |
|---|---:|---|
| IBGP | 255 | Not required |
| EBGP | 1 | Enabled |
| EBGP with `ebgp-multihop 2` | 2 | Bypassed for multihop peering |
| EBGP with `ebgp-multihop 5` | 5 | Bypassed for multihop peering |

The TTL is a limit on the path that the TCP/IP packets carrying BGP messages can traverse. It is **not** a BGP hop-count metric and does not control how many ASNs may appear in `AS_PATH`.

When specifying a TTL explicitly, choose a value sufficient for the longest expected forwarding path between the peer addresses. It is generally better not to set a needlessly high TTL.

---

## EBGP TTL Security (GTSM)

**Generalized TTL Security Mechanism (GTSM)** provides additional protection for EBGP sessions by checking the TTL of incoming packets.

Instead of transmitting EBGP packets with TTL 1 or a small multihop value, TTL security sends packets with an initial TTL of **255** and rejects incoming packets with a TTL below a configured threshold.

The assumption is that packets from a legitimate nearby peer arrive with a high TTL. Packets sent from a distant attacker normally arrive with a lower TTL because routers decrement TTL as they forward packets.

Configure:

```text
R1(config)# router bgp 65001
R1(config-router)# neighbor 3.3.3.3 ttl-security hops 2
```

On a two-hop topology, the other peer uses:

```text
R3(config)# router bgp 65003
R3(config-router)# neighbor 1.1.1.1 ttl-security hops 2
```

Keep the `remote-as`, `update-source`, routing, and address-family activation configuration from the preceding multihop lab. Replace `ebgp-multihop 2` with `ttl-security hops 2` on each peer.

### TTL Security Calculation

Cisco IOS XE calculates the minimum acceptable incoming TTL as:

```text
Minimum incoming TTL = 255 - configured hop count
```

| Command | Initial outgoing TTL | Minimum accepted incoming TTL |
|---|---:|---:|
| `ttl-security hops 1` | 255 | 254 |
| `ttl-security hops 2` | 255 | 253 |
| `ttl-security hops 3` | 255 | 252 |
| `ttl-security hops 5` | 255 | 250 |

For the R1-R2-R3 example, a packet sent with TTL 255 arrives at R3 with TTL 254, so `ttl-security hops 2` accepts it.

The threshold is inclusive: an incoming packet is accepted by the TTL check if its TTL is **greater than or equal to** the configured minimum. Other BGP and TCP checks must also succeed.

### `ttl-security` vs. `ebgp-multihop`

**The commands are mutually exclusive for the same EBGP neighbor.** On IOS XE, `ttl-security hops` supplies the multihop allowance itself; it cannot be combined with `ebgp-multihop` on that session.

```text
R1(config)# router bgp 65001
R1(config-router)# no neighbor 3.3.3.3 ebgp-multihop
R1(config-router)# neighbor 3.3.3.3 ttl-security hops 2
```

| Feature | `ebgp-multihop` | `ttl-security hops` |
|---|---|---|
| Primary purpose | Permit non-direct EBGP peering | Restrict EBGP peer distance using incoming TTL validation |
| Outgoing TTL | Configured value | 255 |
| Incoming TTL security check | No special GTSM validation | Yes |
| Allows multihop EBGP | Yes | Yes |
| Can use both on one neighbor | No | No |

TTL security is not supported for IBGP peers in the documented IOS XE implementation. It should normally be configured on **both EBGP peers**. If only one side uses TTL security, the other side must still transmit packets with a sufficiently high TTL, and interoperability requires careful verification.

> TTL security helps reject off-path spoofed packets with unexpectedly low TTLs. It does not authenticate a neighbor and does not replace TCP authentication or appropriate control-plane filtering.

---

## IPv6 Update Source and Multihop Peering

The same principles apply to BGP peering with IPv6 addresses.

For example, to establish an **IPv6-addressed multihop EBGP session** between loopbacks, configure a reachable IPv6 address on each loopback and ensure IPv6 routing works in both directions.

**R1 (AS 65001):**

```text
R1(config)# interface Loopback0
R1(config-if)# ipv6 address 2001:DB8:1::1/128
R1(config)# router bgp 65001
R1(config-router)# neighbor 2001:DB8:3::3 remote-as 65003
R1(config-router)# neighbor 2001:DB8:3::3 update-source Loopback0
R1(config-router)# neighbor 2001:DB8:3::3 ebgp-multihop 2
R1(config-router)# address-family ipv6 unicast
R1(config-router-af)# neighbor 2001:DB8:3::3 activate
```

**R3 (AS 65003):**

```text
R3(config)# interface Loopback0
R3(config-if)# ipv6 address 2001:DB8:3::3/128
R3(config)# router bgp 65003
R3(config-router)# neighbor 2001:DB8:1::1 remote-as 65001
R3(config-router)# neighbor 2001:DB8:1::1 update-source Loopback0
R3(config-router)# neighbor 2001:DB8:1::1 ebgp-multihop 2
R3(config-router)# address-family ipv6 unicast
R3(config-router-af)# neighbor 2001:DB8:1::1 activate
```

This example assumes IPv6 forwarding is enabled as required and the IPv6 transit network already provides reachability between the loopbacks. The peering addresses are IPv6, and the peers exchange IPv6-unicast NLRI.

A BGP session can also use IPv4 transport to carry IPv6 NLRI when the platform and address-family configuration support it. **The address family of the routes and the IP version of the BGP TCP session are separate concepts.**

IPv6 link-local BGP neighbors have additional interface-scoping and update-source requirements. Their configuration differs from global-address loopback peering.

---

## Verification

### Check Neighbor State

```text
R1# show bgp ipv4 unicast summary
R1# show bgp ipv4 unicast neighbors 3.3.3.3
```

For an IPv6 unicast neighbor:

```text
R1# show bgp ipv6 unicast summary
R1# show bgp ipv6 unicast neighbors 2001:DB8:3::3
```

The detailed neighbor output can show:

- Whether the neighbor is internal or external.
- Local and remote TCP endpoint addresses and ports.
- Outgoing TTL and, when configured, minimum incoming TTL.
- BGP session state, uptime, and negotiated capabilities.

Look for `Local host`, `Foreign host`, and TCP TTL information in the detailed output. The exact display format depends on the IOS XE release.

### Verify Local and Remote Address Reachability

```text
R1# show ip route 3.3.3.3
R1# show ip cef 3.3.3.3
R1# ping 3.3.3.3 source Loopback0
R1# traceroute 3.3.3.3 source Loopback0
```

For IPv6:

```text
R1# show ipv6 route 2001:DB8:3::3
R1# ping ipv6 2001:DB8:3::3 source Loopback0
```

**Always verify return-path routing.** A one-way route toward the remote loopback is insufficient for a TCP session.

### Verify the TCP Connection

Depending on IOS XE platform and release:

```text
R1# show tcp brief all
R1# show bgp ipv4 unicast neighbors 3.3.3.3
```

An established session has a TCP connection whose local and foreign addresses correspond to the configured source and neighbor addresses.

### Verify the Configuration

```text
R1# show running-config | section router bgp
R1# show running-config interface Loopback0
```

Check that the neighbor IP, remote ASN, update source, multihop or TTL-security configuration, and address-family activation are consistent.

---

## Troubleshooting

| Symptom | Possible cause | Check or fix |
|---|---|---|
| Loopback peering fails, but interface-address peering works | Incorrect TCP source address | Configure `neighbor update-source Loopback0` on both peers |
| Peering loopback is unreachable | Missing underlay route | Verify forward and return routes with sourced pings |
| Direct EBGP works, but loopback EBGP does not | Connected-neighbor check | For single-hop loopback EBGP, use `disable-connected-check` or appropriate multihop configuration |
| EBGP peer is beyond an intermediate router | Outgoing TTL too small | Configure `ebgp-multihop` with sufficient TTL or use TTL security |
| TTL-security session fails | Incoming TTL below threshold | Verify actual hop count and configured `ttl-security hops` |
| CLI rejects `ttl-security` configuration | `ebgp-multihop` already configured | Remove `ebgp-multihop` for that neighbor first |
| Neighbor remains in Active/Connect | TCP connection cannot complete | Check source address, TCP/179 filtering, reachability, and TTL |
| Session establishes but no expected routes appear | Missing address-family activation, origination, or routing policy | Verify the address-family and advertisements; peering configuration alone does not originate routes |
| Session is established but advertised routes are unusable | BGP NEXT_HOP not reachable | Inspect the route's NEXT_HOP separately from session reachability |

A reliable troubleshooting order is:

1. Confirm that both endpoints use the expected local and remote ASNs.
2. Confirm that each router has a route to the other's configured peering address.
3. Ping the peer **using the configured local update-source address**.
4. Verify that the incoming TCP source address matches the neighbor definition on the other router.
5. For EBGP, verify the direct-connection check, outgoing TTL, or incoming TTL-security threshold.
6. Verify TCP port 179 is not blocked and inspect detailed BGP neighbor output.
7. Once Established, troubleshoot address-family activation and route advertisement separately.

---

## Command Summary

| Command | Purpose |
|---|---|
| `neighbor <IP> remote-as <ASN>` | Define the peer and determine whether the relationship is IBGP or EBGP |
| `neighbor <IP> update-source Loopback0` | Source the BGP TCP session from Loopback0 |
| `no neighbor <IP> update-source Loopback0` | Remove the explicit source interface |
| `neighbor <IP> ebgp-multihop 2` | Permit EBGP peering across a path requiring TTL 2 |
| `no neighbor <IP> ebgp-multihop` | Remove the multihop setting |
| `neighbor <IP> disable-connected-check` | Bypass the directly connected verification for single-hop EBGP |
| `no neighbor <IP> disable-connected-check` | Reenable the direct-connection verification |
| `neighbor <IP> ttl-security hops 2` | Send EBGP packets with TTL 255 and require a sufficiently high incoming TTL |
| `neighbor <IP> activate` | Activate a neighbor under the current address family |
| `show bgp ipv4 unicast neighbors <IP>` | Inspect session and transport parameters |
| `show tcp brief all` | Inspect local and remote TCP endpoints |
| `ping <peer-IP> source Loopback0` | Test underlay connectivity using the BGP source interface |
| `traceroute <peer-IP> source Loopback0` | Check the routed path and intermediate hops |

---

## Cisco Documentation

- [Troubleshoot Border Gateway Protocol Basic Issues](https://www.cisco.com/c/en/us/support/docs/ip/border-gateway-protocol-bgp/218027-troubleshoot-border-gateway-protocol-bas.html)
- [Cisco IOS XE IP Routing Configuration Guide: Configuring BGP Neighbor Session Options](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/ip-routing/b-ip-routing/m_irg-neighbor-0.html)
- [Cisco IOS BGP Command Reference: `neighbor update-source`, `neighbor ebgp-multihop`, and `neighbor disable-connected-check`](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_bgp/command/irg-cr-book/bgp-m1.html)
- [Protecting Border Gateway Protocol for the Enterprise](https://sec.cloudapps.cisco.com/security/center/resources/protecting_border_gateway_protocol)
