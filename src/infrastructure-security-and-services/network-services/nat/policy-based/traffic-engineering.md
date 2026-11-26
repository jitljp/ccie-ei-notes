# Destination-Based NAT and Traffic Engineering

**Destination-based NAT** allows the same inside local address to be translated to different inside global addresses depending on the destination of the traffic. An extended ACL referenced by a NAT route map can also distinguish traffic by protocol and transport-layer port.

Destination-based NAT is particularly useful when combined with **traffic engineering**, in which routing policies control the path taken by traffic through the network.

For example, an enterprise connected to two ISPs might have the following requirements:

- Traffic to a partner network must use ISP2 and an address assigned by ISP2.
- Traffic to other destinations must use ISP1 and an address assigned by ISP1.
- Selected applications must use ISP2, even when other traffic to the same destination uses ISP1.

Meeting these requirements involves two separate decisions: selecting the forwarding path and selecting the appropriate translation.

---

## Destination-Based Translation vs. Destination NAT

Do not confuse destination-based NAT with **inside destination NAT**, which was covered in the dynamic NAT section.

| Feature | Purpose |
|---|---|
| Destination-based inside source NAT | Select an inside global source address based on the packet's destination |
| Inside destination NAT | Translate an incoming destination address to an inside server address, such as with a rotary pool |
| Policy-based routing (PBR) | Select a forwarding path based on configured criteria |

For destination-based inside source NAT, the destination is a **match criterion**, not the address being translated.

For example:

| Original packet | Translated packet |
|---|---|
| `10.1.1.5 -> 3.3.3.7` | `7.7.7.50 -> 3.3.3.7` |
| `10.1.1.5 -> 3.3.3.8` | `7.7.7.60 -> 3.3.3.8` |

Both packets retain their original destination addresses. They can even exit through the same interface.

The previous page, [Route-Map-Based NAT and Conditional Translation](route-maps.md), demonstrates how extended translations allow these mappings to coexist without `overload`. This page focuses on the interaction between translation policy and forwarding policy.

---

## Separating Routing and NAT Policies

For traditional inside-to-outside NAT, PBR and the routing decision precede address translation.

```text
                   Original inside packet
                            |
                            v
                 PBR (if configured)
                            |
                            v
                Select next hop / egress
                            |
                            v
                  NAT translation lookup
                            |
                            v
             Select NAT rule if necessary
                            |
                            v
                    Translate packet
                            |
                            v
                 Forward toward next hop
```

The NAT rule can classify the packet using its original source and destination addresses, or it can match the selected outgoing interface and next hop.

| Requirement | Feature or command |
|---|---|
| Select a translation based on the destination | NAT route map with `match ip address` and an extended ACL |
| Select a translation based on the outgoing interface | NAT route map with `match interface` |
| Select a translation based on the selected next hop | NAT route map with `match ip next-hop` |
| Select an ISP based on the destination prefix | Static or dynamic routing |
| Select an ISP based on the source, destination, protocol, or port | PBR with `match ip address` and `set ip next-hop` |

For example, the following NAT statement uses the IP address of `Ethernet0/2` as the inside global address:

```text
ip nat inside source route-map NAT-ISP2 interface Ethernet0/2 overload
```

**The `interface Ethernet0/2` argument does not instruct the router to forward packets out of Ethernet0/2.** It identifies the address to use for PAT.

Similarly, a `match interface Ethernet0/2` condition in a NAT route map matches a forwarding decision that has already been made. It does not make that decision.

To send traffic through a particular ISP, configure routing or PBR separately. See [NAT Packet Processing and Order of Operations](../order-of-operations.md).

---

## Lab Topology

Extend the topology from the preceding NAT exercises by adding ISP2 and a remote router. Both ISPs can reach RH7 and RH8, allowing us to choose either path to the same destination.

```text
                             ISP1
                       E0/1       E0/0
                     7.7.7.1     192.0.2.1
                        /            \
                       /              \
             E0/1: 7.7.7.2       E0/1: 192.0.2.2
                       /              \
LH5 ---- LAN ---- Local_Router       Remote_Router ---- LAN ---- RH7
LH6               E0/0: 10.1.1.1      E0/0: 3.3.3.1            RH8
                       \              /
             E0/2: 8.8.8.2       E0/2: 192.0.2.6
                       \              /
                        \            /
                     8.8.8.1     192.0.2.5
                       E0/1       E0/0
                             ISP2
```

| Device | Interface | IPv4 address | NAT role |
|---|---|---|---|
| Local_Router | E0/0 | `10.1.1.1/24` | Inside |
| Local_Router | E0/1 | `7.7.7.2/24` | Outside |
| Local_Router | E0/2 | `8.8.8.2/24` | Outside |
| ISP1 | E0/1 | `7.7.7.1/24` | None |
| ISP1 | E0/0 | `192.0.2.1/30` | None |
| ISP2 | E0/1 | `8.8.8.1/24` | None |
| ISP2 | E0/0 | `192.0.2.5/30` | None |
| Remote_Router | E0/1 | `192.0.2.2/30` | None |
| Remote_Router | E0/2 | `192.0.2.6/30` | None |
| Remote_Router | E0/0 | `3.3.3.1/24` | None |

| Host | IPv4 address | Default gateway |
|---|---|---|
| LH5 | `10.1.1.5/24` | `10.1.1.1` |
| LH6 | `10.1.1.6/24` | `10.1.1.1` |
| RH7 | `3.3.3.7/24` | `3.3.3.1` |
| RH8 | `3.3.3.8/24` | `3.3.3.1` |

### Outside Routing

Configure both ISPs with a route to the remote LAN:

```text
ISP1(config)# ip route 3.3.3.0 255.255.255.0 192.0.2.2
ISP2(config)# ip route 3.3.3.0 255.255.255.0 192.0.2.6
```

On Remote_Router, configure routes to the two ISP-facing networks:

```text
Remote_Router(config)# ip route 7.7.7.0 255.255.255.0 192.0.2.1
Remote_Router(config)# ip route 8.8.8.0 255.255.255.0 192.0.2.5
```

These routes ensure that replies to `7.7.7.2` return through ISP1, and replies to `8.8.8.2` return through ISP2.

No outside router needs a route to `10.1.1.0/24` for correctly translated traffic. Only Local_Router performs NAT.

---

## Lab 1: Destination-Based Traffic Engineering with Static Routes

### Requirements

- Traffic from the inside LAN to RH8 must be forwarded through ISP2 and translated to `8.8.8.2`.
- Traffic from the inside LAN to RH7 must be forwarded through ISP1 and translated to `7.7.7.2`.
- Both LH5 and LH6 must be able to share the translated addresses using PAT.

### Step 1: Configure NAT Interface Roles and Routing

```text
Local_Router(config)# interface Ethernet0/0
Local_Router(config-if)# ip nat inside
Local_Router(config-if)# exit
!
Local_Router(config)# interface Ethernet0/1
Local_Router(config-if)# ip nat outside
Local_Router(config-if)# exit
!
Local_Router(config)# interface Ethernet0/2
Local_Router(config-if)# ip nat outside
Local_Router(config-if)# exit
!
Local_Router(config)# ip route 0.0.0.0 0.0.0.0 7.7.7.1
Local_Router(config)# ip route 3.3.3.8 255.255.255.255 8.8.8.1
```

The default route selects ISP1 for RH7. The more specific `/32` route selects ISP2 for RH8.

### Step 2: Configure Egress-Based PAT

Rather than duplicate the destination-based routing policy in two NAT ACLs, configure NAT to select the appropriate global address according to the actual outgoing interface.

```text
Local_Router(config)# ip access-list standard NAT-SOURCES
Local_Router(config-std-nacl)# permit 10.1.1.0 0.0.0.255
Local_Router(config-std-nacl)# exit
!
Local_Router(config)# route-map NAT-ISP1 permit 10
Local_Router(config-route-map)# match ip address NAT-SOURCES
Local_Router(config-route-map)# match interface Ethernet0/1
Local_Router(config-route-map)# exit
!
Local_Router(config)# route-map NAT-ISP2 permit 10
Local_Router(config-route-map)# match ip address NAT-SOURCES
Local_Router(config-route-map)# match interface Ethernet0/2
Local_Router(config-route-map)# exit
!
Local_Router(config)# ip nat inside source route-map NAT-ISP1 interface Ethernet0/1 overload
Local_Router(config)# ip nat inside source route-map NAT-ISP2 interface Ethernet0/2 overload
```

Both route maps match the same source network, but they match different outgoing interfaces. The rules are mutually exclusive for a packet with a single selected egress interface.

### Step 3: Verify

From LH5 and LH6, ping RH7 and RH8:

```text
LH5# ping 3.3.3.7
LH5# ping 3.3.3.8
LH6# ping 3.3.3.7
LH6# ping 3.3.3.8
```

All four pings should succeed. The expected results are:

| Source | Destination | Outgoing interface | Inside global |
|---|---|---|---|
| LH5 | RH7 | E0/1 | `7.7.7.2` |
| LH5 | RH8 | E0/2 | `8.8.8.2` |
| LH6 | RH7 | E0/1 | `7.7.7.2` |
| LH6 | RH8 | E0/2 | `8.8.8.2` |

Check translations:

```text
Local_Router# show ip nat translations
Pro  Inside global         Inside local          Outside local         Outside global
icmp 8.8.8.2:2             10.1.1.6:1            3.3.3.8:1             3.3.3.8:2
icmp 8.8.8.2:1             10.1.1.5:1            3.3.3.8:1             3.3.3.8:1
icmp 7.7.7.2:1             10.1.1.5:0            3.3.3.7:0             3.3.3.7:1
icmp 7.7.7.2:2             10.1.1.6:0            3.3.3.7:0             3.3.3.7:2
Total number of translations: 4
```

Check the NAT table immediately after generating the traffic because ICMP translations have a short timeout. A packet capture on each outside link confirms the actual forwarding path and translated source address.

### Step 4: Change the Routing Policy

Stop the test traffic and clear the dynamic translations. Remove the more specific route to RH8:

```text
Local_Router# clear ip nat translation *
Local_Router# configure terminal
Local_Router(config)# no ip route 3.3.3.8 255.255.255.255 8.8.8.1
```

Repeat the pings. Both destinations now use ISP1, and new translations for both destinations use `7.7.7.2`.

```
Local_Router# show ip nat translations 
Pro  Inside global         Inside local          Outside local         Outside global
icmp 7.7.7.2:2             10.1.1.5:2            3.3.3.8:2             3.3.3.8:2
icmp 7.7.7.2:4             10.1.1.6:3            3.3.3.8:3             3.3.3.8:4
icmp 7.7.7.2:1             10.1.1.6:2            3.3.3.7:2             3.3.3.7:1
icmp 7.7.7.2:3             10.1.1.5:3            3.3.3.7:3             3.3.3.7:3
Total number of translations: 4
```

**The NAT configuration did not change.** The egress-based route maps select translations appropriate for the new forwarding decision. This is an advantage of matching the outgoing interface instead of hard-coding a destination-to-ISP relationship in the NAT rules.

Restore the `/32` route to RH8 before proceeding to Lab 2. Stop traffic and clear dynamic translations again.

---

## Lab 2: Application-Based Traffic Engineering with PBR

Destination-based routing alone cannot distinguish applications that communicate with the same destination IP address. PBR can make forwarding decisions using additional fields, such as the source IP address, protocol, and destination port.

### Requirements

In addition to the requirements from Lab 1:

- Telnet traffic (TCP destination port 23) from the inside LAN to RH7 must use ISP2.
- Other traffic to RH7 must continue using ISP1.
- Traffic to RH8 must continue using ISP2.
- Each new flow must be translated using the address of its selected outgoing interface.

Telnet is used only as a convenient TCP test in this isolated lab. Configure a listening Telnet service on RH7, or substitute another available TCP service and change the ACL accordingly.

### Step 1: Configure PBR

```text
Local_Router(config)# ip access-list extended RH7-TELNET
Local_Router(config-ext-nacl)# permit tcp 10.1.1.0 0.0.0.255 host 3.3.3.7 eq 23
Local_Router(config-ext-nacl)# exit
!
Local_Router(config)# route-map PBR-INSIDE permit 10
Local_Router(config-route-map)# match ip address RH7-TELNET
Local_Router(config-route-map)# set ip next-hop 8.8.8.1
Local_Router(config-route-map)# exit
!
Local_Router(config)# interface Ethernet0/0
Local_Router(config-if)# ip policy route-map PBR-INSIDE
```

The PBR route map is applied **inbound on the inside interface**. It matches the original, untranslated packet and sends matching traffic to ISP2. Traffic not matched by the PBR policy follows ordinary destination-based routing.

The NAT route maps from Lab 1 remain unchanged. They match the outgoing interface selected by routing or PBR.

`set ip next-hop` can override an existing destination route. Do not substitute `set ip default next-hop` here: the latter is intended to affect traffic for which ordinary routing does not have an explicit destination route.

### Step 2: Verify Different Paths to the Same Destination

From LH5, initiate a ping and a Telnet connection to RH7:

```text
LH5# ping 3.3.3.7
LH5# telnet 3.3.3.7
```

Keep the Telnet connection open while inspecting the NAT table. If using an IOS router as LH5, the Telnet escape sequence (`Ctrl-Shift-6`, then `x`) can suspend the session without closing it.

| Traffic | Forwarding decision | Outgoing interface | Inside global |
|---|---|---|---|
| ICMP to RH7 | Default route | E0/1 | `7.7.7.2` |
| TCP/23 to RH7 | PBR sequence 10 | E0/2 | `8.8.8.2` |
| ICMP to RH8 | `/32` static route | E0/2 | `8.8.8.2` |

NAT translation table:

```text
Pro  Inside global         Inside local          Outside local         Outside global
icmp 7.7.7.2:4             10.1.1.5:4            3.3.3.7:4             3.3.3.7:4
tcp  8.8.8.2:5062          10.1.1.5:37438        3.3.3.7:23            3.3.3.7:23
Total number of translations: 2
```

The destination address is identical, but the flows use different global source addresses. The source ports or ICMP identifiers may be preserved or changed by PAT.

### Packet Walk: Telnet to RH7

1. Local_Router receives a TCP packet from `10.1.1.5` to `3.3.3.7`, destination port 23, on E0/0.
2. The `PBR-INSIDE` route map selects next hop `8.8.8.1` through E0/2, overriding the default route through ISP1.
3. For a new flow, `NAT-ISP2` selects the PAT rule using E0/2's address.
4. Local_Router forwards the packet toward ISP2 with source address `8.8.8.2`. The destination is still `3.3.3.7`.
5. RH7 replies to `8.8.8.2`. Remote_Router routes the reply through ISP2.
6. Local_Router receives the reply on E0/2, translates the destination back to `10.1.1.5` (and restores the original port if necessary), then routes the packet toward LH5.

The ordinary route to RH7 on Local_Router still points toward ISP1. PBR does not modify the routing table.

### Alternative: Match the Forwarding Next Hop

Instead of matching the outgoing interface, a NAT route map can match the selected next-hop address. For example, the equivalent NAT-ISP2 classifier in this topology is:

```text
ip access-list standard ISP2-NEXT-HOP
 permit host 8.8.8.1
!
route-map NAT-ISP2 permit 10
 match ip address NAT-SOURCES
 no match interface Ethernet0/2
 match ip next-hop ISP2-NEXT-HOP
```

Stop the test traffic and clear dynamic translations before testing the change. The PAT statement is unchanged.

This is useful when multiple possible next hops share an outgoing interface. An interface match cannot distinguish those next hops, but a next-hop match can. The translated address or pool must still be specified in the NAT statement.

---

## Troubleshooting: NAT Address Selection Does Not Select the Path

Suppose an administrator replaces the two egress-based PAT rules with the following destination-based rules:

```text
ip access-list extended TO-RH7
 permit ip 10.1.1.0 0.0.0.255 host 3.3.3.7
!
ip access-list extended TO-RH8
 permit ip 10.1.1.0 0.0.0.255 host 3.3.3.8
!
route-map NAT-RH7 permit 10
 match ip address TO-RH7
!
route-map NAT-RH8 permit 10
 match ip address TO-RH8
!
ip nat inside source route-map NAT-RH7 interface Ethernet0/1 overload
ip nat inside source route-map NAT-RH8 interface Ethernet0/2 overload
```

Without PBR, this policy agrees with the routing policy from Lab 1. However, with the PBR policy from Lab 2, a new Telnet flow to RH7 is forwarded through ISP2 while the NAT rule selects the address of E0/1 (`7.7.7.2`).

| Decision | Result |
|---|---|
| PBR | Next hop `8.8.8.1` (ISP2) |
| NAT classification | `NAT-RH7` |
| Translated source | `7.7.7.2` (ISP1 address) |

The result is a mismatch between the forwarding path and the translated source address.

A real ISP may discard traffic sourced from a prefix not assigned to that connection (source address validation). Even if it does not, the remote network may return traffic via ISP1 because the source address belongs to ISP1's prefix.

In this lab, the latter asymmetric path can still work because both ISP links terminate on the same NAT router and no upstream source filter is configured. A successful ping or TCP connection alone does not prove that the intended forwarding policy has been implemented.

For this design, restore the egress-based rules from Lab 1. The NAT classifier should follow the actual path selected by PBR or routing rather than independently assuming a destination always uses a particular ISP.

If a policy genuinely requires a particular global address for a destination regardless of the outgoing interface, destination-based NAT is appropriate, but the forwarding and upstream return routing must support that address on the selected path.

---

## Verification and Troubleshooting

```text
Local_Router# show ip policy
Local_Router# show route-map PBR-INSIDE
Local_Router# show access-lists
Local_Router# show ip route 3.3.3.7
Local_Router# show ip route 3.3.3.8
Local_Router# show ip cef exact-route 10.1.1.5 3.3.3.7
Local_Router# show route-map NAT-ISP1
Local_Router# show route-map NAT-ISP2
Local_Router# show ip nat translations
Local_Router# show ip nat translations verbose
Local_Router# show ip nat statistics
```

`show ip cef exact-route` shows the ordinary CEF forwarding decision for the specified addresses. It does not demonstrate the effect of an inbound PBR policy that matches additional packet fields such as a TCP destination port. Use PBR policy counters and packet captures on the outside links to confirm actual forwarding.

| Symptom | Possible cause |
|---|---|
| All traffic follows ISP1 | PBR is not applied to the inside interface, the ACL does not match, or the specific route is missing |
| PBR works, but the source address belongs to the other ISP | NAT policy selects an address independently of the actual egress interface, or existing translation state is reused |
| Ping and Telnet use the same ISP | The PBR ACL is too broad or does not match the intended protocol/port |
| Traffic exits through the correct ISP but is untranslated | Incorrect NAT interface roles, nonmatching NAT route maps, or missing NAT rules |
| Translated traffic leaves, but replies do not arrive | Missing return routes to the inside global address, upstream filtering, or a host/service problem |
| Old flows continue using a previous translation | Applicable NAT state already exists |

When testing, generate traffic from LH5 or LH6, not from Local_Router itself. Locally generated traffic does not enter E0/0 and therefore is not subject to that interface's inbound PBR policy.

Stop test traffic and clear dynamic translations between routing or NAT policy changes when you need a clean comparison:

```text
Local_Router# clear ip nat translation *
```

Existing flows are not guaranteed to acquire a new translation automatically after a routing change. Changing a translated address can also break the transport-layer session. Resilient routing and NAT behavior during an ISP outage are covered separately in [Multi-ISP NAT and Failover](multi-egress.md).

---

## Key Points

- Destination-based inside source NAT uses the destination as a classification criterion while translating the inside source address.
- NAT selects a translation; routing and PBR select a forwarding path.
- Inside-to-outside routing (including PBR) occurs before NAT.
- `match interface` in a NAT route map matches the selected outgoing interface.
- The interface specified in a NAT overload statement provides the inside global address; it does not select the forwarding path.
- Destination-based routing can select different ISPs for different destination prefixes without PBR.
- PBR can select different ISPs for traffic to the same destination, based on criteria such as protocol and destination port.
- Egress-based NAT allows the translation policy to follow the actual forwarding decision.
- Existing translations, upstream source validation, and return reachability must be considered when combining NAT with traffic engineering.

---

## References

- [Cisco: NAT Support for Multiple Pools Using Route Maps](https://www.cisco.com/c/en/us/support/docs/ip/network-address-translation-nat/13739-nat-routemap.html)
- [Cisco: Understand the NAT Order of Operation](https://www.cisco.com/c/en/us/support/docs/ip/network-address-translation-nat/6209-5.html)
- [Cisco: IOS NAT Load-Balancing for Two ISP Connections](https://www.cisco.com/c/en/us/support/docs/ip/network-address-translation-nat/100658-ios-nat-load-balancing-2isp.html)
- [Cisco IOS XE 17.x: Configuring NAT for IP Address Conservation](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-addressing/b-ip-addressing/m_iadnat-addr-consv-xe.html)
- [Cisco IOS XE 17.x: Policy-Based Routing](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-routing/b-ip-routing/m_iri-pbr.html)
