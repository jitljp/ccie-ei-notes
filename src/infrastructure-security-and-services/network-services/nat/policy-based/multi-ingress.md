# Multi-Ingress NAT

**Multi-ingress NAT** allows outside hosts to initiate connections to the same inside host through multiple outside addresses. For example, an enterprise connected to two ISPs can publish an inside server using an inside global address from each ISP's address space.

In the preceding page, [Multi-ISP NAT and Failover](multi-egress.md), the inside host initiated connections and NAT selected an inside global address according to the outgoing interface. In a multi-ingress design, the outside client initiates the connection by selecting one of the server's inside global addresses.

Two independent requirements must be satisfied:

- **Translation consistency:** Reply packets must use the inside global address to which the client originally connected.
- **Routing consistency:** Reply packets must follow a path that can carry traffic sourced from that inside global address.

These requirements are related, but NAT does not automatically enforce both.

---

## Multiple Static Mappings for One Inside Host

Ordinary static inside source NAT establishes a fixed mapping between an inside local address and an inside global address:

```text
Local_Router(config)# ip nat inside source static 10.1.1.5 7.7.7.50
```

To make LH5 reachable via a second inside global address, configure both mappings with the `extendable` keyword:

```text
Local_Router(config)# ip nat inside source static 10.1.1.5 7.7.7.50 extendable
Local_Router(config)# ip nat inside source static 10.1.1.5 8.8.8.50 extendable
```

The `extendable` keyword allows otherwise conflicting static mappings to coexist, such as multiple inside global addresses associated with the same inside local address.

| Inside local | Inside global | Intended outside path |
|---|---|---|
| `10.1.1.5` | `7.7.7.50` | ISP1 |
| `10.1.1.5` | `8.8.8.50` | ISP2 |

Both mappings are installed when configured. Neither requires LH5 to initiate traffic before an outside client can connect.

The initial inbound translation is unambiguous because the destination inside global address identifies the mapping. When an outside client establishes a connection, NAT can create an extended translation that identifies that particular flow. Replies belonging to that flow can then use the correct inside global address.

---

## Lab Topology

Continue using the topology from the previous page. The multi-ingress exercise is adapted from Case 15 of [Jedadiah Casey's NAT article](https://wax-trax.medium.com/understanding-and-configuring-network-address-translation-nat-for-cisco-ccna-ccnp-and-ccie-7fef92fa654c).

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

LH5's inside global addresses: 7.7.7.50 and 8.8.8.50
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

Remove the NAT rules, PBR policies, IP SLA tracking of default routes, and destination-specific routes from previous exercises before starting. Stop active traffic and clear dynamic translations before removing in-use rules.

### NAT Interface Roles

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
```

### Outside Routing

```text
ISP1(config)# ip route 3.3.3.0 255.255.255.0 192.0.2.2
ISP1(config)# ip route 7.7.7.50 255.255.255.255 7.7.7.2
!
ISP2(config)# ip route 3.3.3.0 255.255.255.0 192.0.2.6
ISP2(config)# ip route 8.8.8.50 255.255.255.255 8.8.8.2
!
Remote_Router(config)# ip route 7.7.7.0 255.255.255.0 192.0.2.1
Remote_Router(config)# ip route 8.8.8.0 255.255.255.0 192.0.2.5
```

The host routes on ISP1 and ISP2 direct traffic for the published addresses toward Local_Router's actual interface addresses. The remote hosts do not require routes to `10.1.1.0/24`.

Initially, configure only one default route on Local_Router:

```text
Local_Router(config)# ip route 0.0.0.0 0.0.0.0 Ethernet0/1 7.7.7.1
```

All traffic from Local_Router to the remote LAN will initially use ISP1. This is intentional: it allows us to observe the difference between the inbound path and the return path.

---

## Lab 1: Multi-Ingress Static NAT with extendable

### Requirements

- Make LH5 reachable at `7.7.7.50` through ISP1.
- Make LH5 reachable at `8.8.8.50` through ISP2.
- Allow connections to both inside global addresses without requiring inside-initiated traffic.

### Configuration

```text
Local_Router(config)# ip nat inside source static 10.1.1.5 7.7.7.50 extendable
Local_Router(config)# ip nat inside source static 10.1.1.5 8.8.8.50 extendable
```

Check the NAT table before generating traffic:

```text
Local_Router# show ip nat translations
```

Expected simple entries:

```text
Pro  Inside global    Inside local    Outside local    Outside global
---  7.7.7.50         10.1.1.5        ---              ---
---  8.8.8.50         10.1.1.5        ---              ---
```

The mappings have different inside global addresses but the same inside local address.

### Verification: Outside-Initiated Connections

From RH7, ping both inside global addresses:

```text
RH7# ping 7.7.7.50
RH7# ping 8.8.8.50
```

For a TCP test, enable a listening service on LH5. If LH5 is an IOS router acting as a host, the TCP echo service can be used:

```text
LH5(config)# service tcp-small-servers
```

Initiate connections from RH7 to TCP port 7 on both global addresses (use separate sessions to keep both connections active):

```text
RH7# telnet 7.7.7.50 7
RH7# telnet 8.8.8.50 7
```

On Local_Router:

```text
Local_Router# show ip nat translations
Local_Router# show ip nat translations verbose
```

Illustrative output, with example client source ports:

```text
Pro  Inside global    Inside local    Outside local    Outside global
tcp  7.7.7.50:7       10.1.1.5:7      3.3.3.7:51000    3.3.3.7:51000
tcp  8.8.8.50:7       10.1.1.5:7      3.3.3.7:51001    3.3.3.7:51001
---  7.7.7.50         10.1.1.5        ---              ---
---  8.8.8.50         10.1.1.5        ---              ---
```

The extended entries identify which inside global address belongs to each connection. The original static entries remain in the table even after the individual connections expire.

These are address translations, not overloaded PAT. The TCP port numbers are used to identify the flows; the static rules do not change the server's port number.

### Packet Flow

For the second TCP connection, consider the following packet headers:

| Packet | Source | Destination |
|---|---|---|
| Client request before NAT | `3.3.3.7:51001` | `8.8.8.50:7` |
| Client request after NAT | `3.3.3.7:51001` | `10.1.1.5:7` |
| Server reply before NAT | `10.1.1.5:7` | `3.3.3.7:51001` |
| Server reply after NAT | `8.8.8.50:7` | `3.3.3.7:51001` |

The client expects a reply from `8.8.8.50`, not from `7.7.7.50` or `10.1.1.5`. An existing extended translation supplies the correct inside global address for the reply.

The two connections are distinguishable on the inside because the client source ports differ. If a client deliberately reuses the same source IP and source port when connecting to both global addresses at the same server port, the resulting inside tuples would be identical. Do not assume such simultaneous connections can always be distinguished without additional translation.

---

## Translation Consistency vs. Path Symmetry

In Lab 1, the initial packet destined to `8.8.8.50` enters Local_Router through ISP2. However, Local_Router's only route to RH7 is the default route through ISP1.

```text
                       ISP2
                      /    \
             request /      \ request
                    v        \
             Local_Router     Remote_Router --- RH7
                    \        ^
               reply \      / reply
                      v    /
                       ISP1
```

The reply can be correctly translated to a source address of `8.8.8.50` but forwarded through ISP1.

This happens because the inside-to-outside routing decision is made before source NAT. A translation entry identifies how to translate a packet; it does not act as a route to the remote client.

In this topology, asymmetric traffic can succeed because both ISP paths converge at Remote_Router and both directions traverse the same NAT router. However, ISP1 might reject packets sourced from ISP2's address space, for example because of source address validation using an ACL or unicast RPF (uRPF).

To see the actual path, capture the traffic on Local_Router's E0/1 and E0/2 links. For a connection from RH7 to `8.8.8.50`, look for the following:

| Direction | Interface | Source | Destination |
|---|---|---|---|
| Inbound request | E0/2 | `3.3.3.7` | `8.8.8.50` |
| Outbound reply | E0/1 | `8.8.8.50` | `3.3.3.7` |

A successful ping or TCP connection is not proof of symmetric routing.

### Demonstrating Upstream Source Filtering

On ISP1, simulate a provider that accepts customer traffic only from its own address block:

```text
ISP1(config)# ip access-list extended CUSTOMER-SOURCES
ISP1(config-ext-nacl)# permit ip 7.7.7.0 0.0.0.255 any
ISP1(config-ext-nacl)# exit
ISP1(config)# interface Ethernet0/1
ISP1(config-if)# ip access-group CUSTOMER-SOURCES in
```

Repeat the tests from RH7. Connections to `7.7.7.50` should still work. Connections to `8.8.8.50` should fail because the correctly translated replies are exiting via ISP1 with a source address of `8.8.8.50`.

```text
ISP1# show access-lists CUSTOMER-SOURCES
Local_Router# show ip route 3.3.3.7
Local_Router# show ip nat translations
```

The NAT configuration is not the cause of this failure. The routing policy and the provider's source address policy are incompatible.

---

## Lab 2: Conditional Static NAT and Symmetric Routing

Instead of using `extendable`, associate each static mapping with a route map. This also allows deterministic selection of the inside global address for new inside-initiated traffic.

For this exercise, RH7 will use ISP1's published address and RH8 will use ISP2's published address. Since the remote clients have distinct IP addresses, their return paths can be selected using ordinary destination-based routing.

### Step 1: Configure the Return Paths

```text
Local_Router(config)# ip route 3.3.3.7 255.255.255.255 Ethernet0/1 7.7.7.1
Local_Router(config)# ip route 3.3.3.8 255.255.255.255 Ethernet0/2 8.8.8.1
```

These host routes take precedence over the default route because they are more specific. The static routes are independent of NAT.

### Step 2: Replace the Static Mappings

Close existing TCP sessions and clear dynamic translations before changing the configuration:

```text
Local_Router# clear ip nat translation *
Local_Router# configure terminal
Local_Router(config)# no ip nat inside source static 10.1.1.5 7.7.7.50 extendable
Local_Router(config)# no ip nat inside source static 10.1.1.5 8.8.8.50 extendable
```

Configure the following route maps and conditional static NAT statements:

```text
Local_Router(config)# route-map STATIC-ISP1 permit 10
Local_Router(config-route-map)# match interface Ethernet0/1
Local_Router(config-route-map)# exit
!
Local_Router(config)# route-map STATIC-ISP2 permit 10
Local_Router(config-route-map)# match interface Ethernet0/2
Local_Router(config-route-map)# exit
!
Local_Router(config)# ip nat inside source static 10.1.1.5 7.7.7.50 route-map STATIC-ISP1
Local_Router(config)# ip nat inside source static 10.1.1.5 8.8.8.50 route-map STATIC-ISP2
```

The static NAT statements identify the inside local address, so a source ACL matching LH5 is not needed. The route maps distinguish the selected outgoing interface. The `route-map` keyword allows the multiple conditional static mappings to coexist without an explicit `extendable` keyword.

### Step 3: Verify Inside-Initiated Traffic

```text
LH5# ping 3.3.3.7
LH5# ping 3.3.3.8
Local_Router# show ip nat translations
```

| Destination | Selected outgoing interface | Expected inside global |
|---|---|---|
| RH7 (`3.3.3.7`) | E0/1 | `7.7.7.50` |
| RH8 (`3.3.3.8`) | E0/2 | `8.8.8.50` |

The route map selects the mapping for a new inside-initiated flow according to the path selected by routing.

### Step 4: Verify Outside-Initiated Traffic

Stop the inside-initiated test traffic and clear dynamic entries so that the next test starts with the configured static mappings alone.

```text
Local_Router# clear ip nat translation *
RH7# telnet 7.7.7.50 7
RH8# telnet 8.8.8.50 7
Local_Router# show ip nat translations
```

| Client | Inside global used | Incoming ISP | Returning ISP |
|---|---|---|---|
| RH7 | `7.7.7.50` | ISP1 | ISP1 |
| RH8 | `8.8.8.50` | ISP2 | ISP2 |

Both clients should be able to connect to LH5, and the return paths are symmetric. The source filter on ISP1 no longer interferes with the intended connections.

### Route Maps on Static NAT and Inbound Traffic

Without `reversible`, the route map on a static inside source mapping is intended to select the mapping for new inside-initiated traffic. The configured inside global address can still identify the mapping for a new outside-initiated connection. The resulting extended entry can identify the correct translation for replies.

Test a new connection from RH7 to `8.8.8.50` and inspect the traffic on both ISP links. The incoming packet can be translated to LH5 even though the routing decision for new outbound traffic to RH7 would select `STATIC-ISP1`. The reply may be translated using `8.8.8.50` but routed via ISP1, where the source filter drops it.

The route map is not an inbound interface ACL. `match interface Ethernet0/2` does not mean that only packets entering Ethernet0/2 can use the mapping.

The static `reversible` option can be used with a supported address-based route map to impose reverse policy conditions. This is different from merely associating an egress-based route map with a static mapping. See [Route-Map-Based NAT and Conditional Translation](route-maps.md) for the syntax and restrictions.

### Limitation: One Client Using Both Global Addresses

The host routes in this exercise work because RH7 and RH8 use different addresses. They cannot ensure symmetry if RH7 connects to both global addresses at the same time.

Before source NAT, replies from LH5 to both RH7 connections have the same source and destination IP addresses: `10.1.1.5` and `3.3.3.7`. A destination-based route cannot select different ISPs according to which global address RH7 originally contacted. A source-based PBR policy matching only LH5 cannot do so either. A NAT route map matching the outgoing interface does not provide this missing routing information.

A design that requires strict symmetry for arbitrary clients needs an additional mechanism that preserves or distinguishes the ingress context when routing replies. Examples include separating the service paths into different routing domains or using a stateful edge device that supports connection-aware return-path selection. If the client populations are known and distinct, static routes or PBR matching those populations may be sufficient, as demonstrated in this exercise.

---

## Static PAT for Multi-Ingress Services

If only a particular service needs to be accessible, use static PAT instead of exposing an address-wide mapping.

For example, suppose LH5 runs an HTTPS server on TCP port 443. Replace the address-only mappings with:

```text
Local_Router(config)# ip nat inside source static tcp 10.1.1.5 443 7.7.7.50 443 extendable
Local_Router(config)# ip nat inside source static tcp 10.1.1.5 443 8.8.8.50 443 extendable
```

| Outside endpoint | Inside endpoint |
|---|---|
| `7.7.7.50:443` | `10.1.1.5:443` |
| `8.8.8.50:443` | `10.1.1.5:443` |

The same inside service is accessible at two outside endpoints. Other ports are not published by these static PAT rules. The `extendable` keyword may be added automatically to static PAT statements by IOS XE.

The routing considerations are the same as with address-only NAT. Static PAT does not ensure that a reply exits through the ISP that received the request.

See [Static PAT and Port Forwarding](../static/pat.md) for the underlying configuration and port translation behavior.

---

## Combining Multi-Ingress NAT with Outbound PAT

An enterprise may need both a published server and ordinary Internet access for its remaining inside hosts.

Using the address-only conditional static mappings from Lab 2, configure outbound PAT for the other hosts. Explicitly exclude LH5 from the PAT classifier so that the static and dynamic policies do not overlap:

```text
Local_Router(config)# ip access-list standard PAT-HOSTS
Local_Router(config-std-nacl)# deny host 10.1.1.5
Local_Router(config-std-nacl)# permit 10.1.1.0 0.0.0.255
Local_Router(config-std-nacl)# exit
!
Local_Router(config)# route-map PAT-ISP1 permit 10
Local_Router(config-route-map)# match ip address PAT-HOSTS
Local_Router(config-route-map)# match interface Ethernet0/1
Local_Router(config-route-map)# exit
!
Local_Router(config)# route-map PAT-ISP2 permit 10
Local_Router(config-route-map)# match ip address PAT-HOSTS
Local_Router(config-route-map)# match interface Ethernet0/2
Local_Router(config-route-map)# exit
!
Local_Router(config)# ip nat inside source route-map PAT-ISP1 interface Ethernet0/1 overload
Local_Router(config)# ip nat inside source route-map PAT-ISP2 interface Ethernet0/2 overload
```

| Inside host | Egress via ISP1 | Egress via ISP2 |
|---|---|---|
| LH5 (conditional static NAT) | `7.7.7.50` | `8.8.8.50` |
| LH6 (dynamic PAT) | `7.7.7.2` | `8.8.8.2` |

Verify by generating new inside-initiated traffic from LH5 and LH6 to both remote hosts. The table describes new outbound flows; replies to existing inbound flows use their associated translations.

The ACL deny excludes LH5 from these PAT rules. It does not drop LH5's packets.

---

## Multi-Ingress NAT and Failover

Publishing the server at two global addresses does not mean that connections to either address automatically survive an ISP failure.

If ISP1 fails:

- Clients using `7.7.7.50` may no longer be able to reach that address.
- Clients that can reach `8.8.8.50` can establish new connections through ISP2, provided return routing and the service are available.
- Existing connections to `7.7.7.50` cannot simply switch to `8.8.8.50` while retaining the same TCP connection.

The outside clients must discover and use the alternate address, for example through DNS failover, an application-level retry mechanism, or explicit configuration. Multiple DNS A records alone do not guarantee health-based failover.

IP SLA tracking on Local_Router can change its outbound routes, but it cannot change the address to which a remote client sends traffic or make an ISP-specific prefix reachable through another ISP. Advertising a common globally routable prefix through both ISPs is a different design.

See [Multi-ISP NAT and Failover](multi-egress.md) for outbound route tracking and the effect of routing changes on existing NAT state.

---

## Verification and Troubleshooting

```text
Local_Router# show ip nat translations
Local_Router# show ip nat translations verbose
Local_Router# show ip nat statistics
Local_Router# show ip route 3.3.3.7
Local_Router# show ip route 3.3.3.8
Local_Router# show ip cef exact-route 10.1.1.5 3.3.3.7
Local_Router# show route-map
Local_Router# show ip policy
Local_Router# show ip aliases
Local_Router# show running-config | include ip nat
ISP1# show access-lists CUSTOMER-SOURCES
```

When PBR is configured, a CEF lookup alone does not prove the actual forwarding path of a policy-routed packet. Use packet captures to confirm the selected interfaces and translated addresses.

| Symptom | What to check |
|---|---|
| Second static mapping cannot be configured | `extendable`, conditional route map syntax, conflicting existing mappings |
| One global address is unreachable | Outside routes, ARP, interface status, incoming ACLs |
| Inbound packet reaches the NAT router but not the server | Static mapping, NAT interface roles, route to the inside local address |
| Both global addresses reach LH5, but only one TCP connection succeeds | Existing extended entries, server listener, return path, upstream source filtering |
| Replies have the correct source address but are sent through the wrong ISP | Routing/PBR rather than the NAT mapping |
| New inside-initiated traffic uses the undesired global address | Conditional static route map, selected egress, competing PAT policy, existing translations |
| Traffic to an old global address fails after ISP failover | Outside reachability of that address and client selection of the alternate address |

In a lab, use `debug ip nat` to observe translations and `undebug all` to stop debugging. Clear dynamic translations between independent tests if needed. Clearing dynamic state does not remove the configured static mappings.

After completing the exercises, remove the temporary ISP1 input ACL and disable the TCP small servers on LH5 if enabled for this lab:

```text
ISP1(config)# interface Ethernet0/1
ISP1(config-if)# no ip access-group CUSTOMER-SOURCES in
!
LH5(config)# no service tcp-small-servers
```

---

## Key Points

- Multi-ingress NAT allows the same inside host or service to be reached using multiple inside global addresses.
- `extendable` allows multiple otherwise conflicting static mappings to coexist.
- Static mappings exist before traffic is generated. Extended entries can identify individual connections and their associated inside global addresses.
- A conditional static NAT route map can select an inside global address for a new inside-initiated flow.
- `match interface` in a NAT route map refers to the selected outgoing interface, not the incoming interface.
- Correct reverse translation does not guarantee symmetric routing. Routing selects the return path before inside source NAT.
- Static PAT can publish the same TCP or UDP service at multiple outside endpoints.
- A change of ISP does not automatically preserve existing sessions or make the failed ISP's global addresses reachable through the surviving ISP.

---

## References

- [Cisco IOS XE 17.x: Configuring NAT for IP Address Conservation](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-addressing/b-ip-addressing/m_iadnat-addr-consv-xe.html)
- [Cisco IOS IP Addressing Services Command Reference: NAT Commands](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/ipaddr/command/ipaddr-cr-book/ipaddr-i3.html)
- [Cisco IOS XE Release 2: Configuring NAT for IP Address Conservation (Static NAT Multihoming)](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/ipaddr_nat/configuration/xe-2/nat-xe-2-book/iadnat-addr-consv.html)
- [Cisco: Understand the NAT Order of Operation](https://www.cisco.com/c/en/us/support/docs/ip/network-address-translation-nat/6209-5.html)
- [Cisco: Network Address Translation (NAT) Frequently Asked Questions](https://www.cisco.com/c/en/us/support/docs/ip/network-address-translation-nat/26704-nat-faq-00.html)
- [Jedadiah Casey: Understanding and Configuring Network Address Translation (NAT) for Cisco CCNA, CCNP and CCIE (Case 15)](https://wax-trax.medium.com/understanding-and-configuring-network-address-translation-nat-for-cisco-ccna-ccnp-and-ccie-7fef92fa654c)
