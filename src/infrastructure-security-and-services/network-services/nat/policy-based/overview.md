# 4.5.d (iii) Policy-Based NAT, PAT

**Policy-based NAT** uses configured conditions to determine whether traffic should be translated and which translation should be applied. Rather than selecting a translation solely according to the inside local address, the NAT router can also consider information such as the destination address and outgoing interface.

On Cisco IOS XE routers, policy-based NAT is commonly implemented by referencing a **route map** in a NAT statement.

Policy-based NAT and PAT are useful for:

- Translating the same inside host to different inside global addresses depending on the destination.
- Exempting selected traffic from translation, such as traffic destined for a private network over a VPN.
- Selecting the appropriate translated address for traffic forwarded through different ISPs.
- Providing multiple outside address representations of the same inside server.
- Combining NAT with routing policies to implement more complex forwarding designs.

**Policy-based NAT is not a separate type of address translation.** It describes how a NAT rule is selected. Policy-based translation can use static NAT, dynamic NAT, or PAT.

---

## ACL-Based NAT vs. Route-Map-Based NAT

In the previous section, dynamic inside source NAT was configured by referencing an ACL directly in the NAT statement:

```text
Local_Router(config)# access-list 10 permit 10.1.1.0 0.0.0.255
Local_Router(config)# ip nat inside source list 10 pool POOL-A
```

The ACL identifies traffic eligible for translation, and `POOL-A` supplies the inside global addresses.

A route map can instead be used to identify the traffic:

```text
Local_Router(config)# route-map NAT-A permit 10
Local_Router(config-route-map)# match ip address 10
Local_Router(config-route-map)# exit
Local_Router(config)# ip nat inside source route-map NAT-A pool POOL-A
```

The route map references the ACL, and the NAT statement references the route map.

| Feature | ACL-Based NAT | Route-Map-Based NAT |
|---|---|---|
| NAT statement references | An ACL (`list`) | A route map (`route-map`) |
| Source address matching | Yes | Yes, via an ACL |
| Destination address matching | Yes, with an extended ACL | Yes, via an extended ACL |
| Outgoing interface matching | No | Yes |
| Next-hop matching | No | Yes |
| Dynamic NAT without overload | Can create simple address mappings | Creates extended translations that distinguish flows |
| Dynamic PAT | Supported | Supported |
| Conditional static inside source NAT | Not via the `list` form | Supported |

The route-map approach is particularly useful when the same inside local address needs to use different inside global addresses concurrently.

**An extended ACL by itself is not always sufficient for destination-dependent NAT.** An existing simple translation can be reused for traffic to a different destination, bypassing the ACL selection that would otherwise apply when creating a new translation.

---

## Route Map Match Criteria

The main route map match commands used for NAT on IOS XE routers are:

| Command | Purpose |
|---|---|
| `match ip address <acl>` | Match packet fields permitted by the referenced ACL |
| `match interface <interface>` | Match the interface through which the packet is being forwarded |
| `match ip next-hop <acl>` | Match the selected next-hop IP address against an ACL |

For example:

```text
Local_Router(config)# ip access-list standard INSIDE-HOSTS
Local_Router(config-std-nacl)# permit 10.1.1.0 0.0.0.255
Local_Router(config-std-nacl)# exit
!
Local_Router(config)# route-map NAT-ISP1 permit 10
Local_Router(config-route-map)# match ip address INSIDE-HOSTS
Local_Router(config-route-map)# match interface Ethernet0/1
```

This route map matches traffic from `10.1.1.0/24` that is being forwarded through `Ethernet0/1`.

When different types of `match` commands are configured in the same route map sequence, all of those conditions must be satisfied.

### Permit and Deny

An ACL referenced by a NAT route map is used for **classification**, not packet filtering.

- An ACL `permit` entry indicates a match for the `match ip address` condition.
- An ACL `deny` entry indicates that the condition does not match; later route map sequences may still be evaluated.
- A matching `permit` route map sequence makes the traffic eligible for the associated NAT rule.
- A matching `deny` sequence rejects the traffic for that NAT rule.

If no permit sequence matches, the route map does not select the associated NAT rule.

**Traffic that does not match a NAT rule is not automatically dropped.** Another NAT rule may apply, or the packet may be forwarded untranslated. To explicitly filter packets, configure a separate security policy.

The NAT statement specifies the translation itself. A `set` command is not required in a NAT route map.

For detailed route map evaluation and conditional translations, see [Route-Map-Based NAT and Conditional Translation](route-maps.md).

---

## Policy-Based NAT Configuration Syntax

### Dynamic NAT Using a Pool

```text
ip nat inside source route-map <map-name> pool <pool-name>
```

The NAT rule selects addresses from the specified pool when traffic matches the route map.

### Dynamic PAT Using a Pool

```text
ip nat inside source route-map <map-name> pool <pool-name> overload
```

The `overload` keyword allows multiple hosts and flows to share the pool's addresses.

### Dynamic PAT Using an Interface

```text
ip nat inside source route-map <map-name> interface <interface> overload
```

The IP address of the specified interface is used as the inside global address.

**The `interface` keyword in the NAT statement does not specify a forwarding path or restrict the rule to packets exiting that interface.** It specifies the source of the translated address. To match the actual outgoing interface, use `match interface` in the route map.

### Conditional Static NAT

```text
ip nat inside source static <inside-local> <inside-global> route-map <map-name>
```

The route map associates conditions with a fixed address mapping. This can be used to configure different mappings for an inside host according to the path or destination. The `extendable` option is useful when configuring multiple static mappings that would otherwise conflict.

Static route-map behavior for outside-initiated traffic must also be considered when designing inbound connectivity.

---

## Example: Destination-Based NAT

Consider the topology used in the dynamic NAT section:

```text
      INSIDE NETWORK                                OUTSIDE NETWORK

  LH5: 10.1.1.5/24
          |
          |              Local_Router              ISP1
          +--- LAN ------ E0/0     E0/1 ---------- E0/1
          |            10.1.1.1   7.7.7.2         7.7.7.1
          |            NAT inside NAT outside        |
  LH6: 10.1.1.6/24                             E0/0: 3.3.3.1/24
                                                      |
                                             +--------+--------+
                                             |                 |
                                       RH7: 3.3.3.7      RH8: 3.3.3.8
```

The requirements are:

- Traffic from the inside network to RH7 must use addresses from `POOL-A`.
- Traffic from the inside network to RH8 must use addresses from `POOL-B`.
- The same inside host must be able to communicate with both outside hosts simultaneously using different inside global addresses.
- Traffic to other destinations must not be translated by these rules.

Both destinations are reached through the **same outside interface**. The policy depends on the destination IP address, not the egress path.

### Step 1: Configure Interface Roles and Routing

Assuming the interface IP addresses and host default gateways are already configured:

```text
Local_Router(config)# interface Ethernet0/0
Local_Router(config-if)# ip nat inside
Local_Router(config-if)# exit
!
Local_Router(config)# interface Ethernet0/1
Local_Router(config-if)# ip nat outside
Local_Router(config-if)# exit
!
Local_Router(config)# ip route 0.0.0.0 0.0.0.0 7.7.7.1
```

Remove any previous NAT rules that would interfere with this exercise.

### Step 2: Classify Traffic

```text
Local_Router(config)# ip access-list extended TO-RH7
Local_Router(config-ext-nacl)# permit ip 10.1.1.0 0.0.0.255 host 3.3.3.7
Local_Router(config-ext-nacl)# exit
!
Local_Router(config)# ip access-list extended TO-RH8
Local_Router(config-ext-nacl)# permit ip 10.1.1.0 0.0.0.255 host 3.3.3.8
Local_Router(config-ext-nacl)# exit
!
Local_Router(config)# route-map NAT-RH7 permit 10
Local_Router(config-route-map)# match ip address TO-RH7
Local_Router(config-route-map)# exit
!
Local_Router(config)# route-map NAT-RH8 permit 10
Local_Router(config-route-map)# match ip address TO-RH8
Local_Router(config-route-map)# exit
```

Each route map matches the same inside source network, but a different outside destination.

### Step 3: Configure the NAT Pools and Rules

```text
Local_Router(config)# ip nat pool POOL-A 7.7.7.50 7.7.7.50 prefix-length 24
Local_Router(config)# ip nat pool POOL-B 7.7.7.60 7.7.7.60 prefix-length 24
!
Local_Router(config)# ip nat inside source route-map NAT-RH7 pool POOL-A
Local_Router(config)# ip nat inside source route-map NAT-RH8 pool POOL-B
```

Each pool contains one address. There is no `overload` keyword, so this example uses dynamic NAT rather than dynamic PAT. Test with LH5 first, since the pools have limited capacity.

### Step 4: Ensure Return Reachability

On ISP1, configure explicit host routes toward the NAT router for the two pool addresses:

```text
ISP1(config)# ip route 7.7.7.50 255.255.255.255 7.7.7.2
ISP1(config)# ip route 7.7.7.60 255.255.255.255 7.7.7.2
```

These routes make ISP1 use the NAT router's actual interface address (`7.7.7.2`) as the next hop, avoiding a dependency on ARP resolution for the individual pool addresses.

### Step 5: Test and Verify

Initiate traffic from LH5 to both RH7 and RH8. For example, ping both destinations and check the NAT translation table before the ICMP translations expire.

The expected translations are:

| Inside local | Outside destination | Inside global |
|---|---|---|
| `10.1.1.5` | `3.3.3.7` | `7.7.7.50` |
| `10.1.1.5` | `3.3.3.8` | `7.7.7.60` |

The route-map-based rules allow these different translations to coexist. For ICMP, the extended entries also identify the ICMP identifiers.

```text
Local_Router# show ip nat translations
Local_Router# show ip nat translations verbose
Local_Router# show ip nat statistics
```

### Convert the Example to PAT

To allow LH5 and LH6 to share each of the single-address pools, replace the two NAT statements with overloaded versions.

First, clear the active dynamic translations in the lab if necessary:

```text
Local_Router# clear ip nat translation *
```

Then modify the rules:

```text
Local_Router(config)# no ip nat inside source route-map NAT-RH7 pool POOL-A
Local_Router(config)# no ip nat inside source route-map NAT-RH8 pool POOL-B
!
Local_Router(config)# ip nat inside source route-map NAT-RH7 pool POOL-A overload
Local_Router(config)# ip nat inside source route-map NAT-RH8 pool POOL-B overload
```

Both inside hosts can now share `7.7.7.50` for traffic to RH7 and `7.7.7.60` for traffic to RH8. The selection policy has not changed; only the translation mechanism has changed from NAT to PAT.

For more destination-based policies and their interaction with routing, see [Destination-Based NAT and Traffic Engineering](traffic-engineering.md).

---

## Policy-Based NAT and the Translation Table

An important distinction between ACL-based dynamic NAT and route-map-based dynamic NAT is the kind of translation entry created.

### Simple Translation

A simple inside source NAT entry associates an inside local address with an inside global address. It is not specific to a particular outside endpoint.

For example, an existing simple translation from `10.1.1.5` to `7.7.7.50` can be reused even when LH5 communicates with a different outside destination.

### Extended Translation

An extended translation contains information identifying the inside and outside endpoints, as well as the protocol and relevant protocol-specific identifiers.

Unlike ACL-based dynamic NAT, route-map-based dynamic NAT does not create a general-purpose simple translation entry unless the `reversible` keyword is configured. This allows different flows from the same inside local address to use different inside global addresses, depending on the configured policy.

**An extended translation does not necessarily mean PAT is being performed.** Route-map-based dynamic NAT can maintain flow-specific state without using port translation to share an address among inside hosts.

The NAT router looks for applicable existing translation state when processing packets. Changing the ACL or route map does not necessarily cause existing translations to be replaced immediately.

When testing a policy change, initiate a new flow or clear the relevant dynamic translations and test again. Clearing all dynamic translations with `clear ip nat translation *` can disrupt active sessions, so use it carefully outside a lab.

See [NAT Translation Tables and Verification](../verification.md) for more information about simple and extended entries.

---

## Policy-Based NAT vs. Policy-Based Routing

Both features can use route maps, but they have different purposes.

| Feature | Policy-Based NAT | Policy-Based Routing (PBR) |
|---|---|---|
| Primary purpose | Select a translation | Select a forwarding path |
| Configuration reference | `ip nat inside source route-map ...` | `ip policy route-map ...` |
| Typical `match` criteria | Source/destination, egress interface, next hop | Source/destination and other packet characteristics |
| Action | Defined in the NAT statement | Defined by a route map `set` command |
| Changes IP addresses or ports | Yes | No |
| Overrides normal routing decisions | No | Can do so |

For traditional inside-to-outside NAT, the routing decision (including applicable PBR) occurs before translation. This makes it possible for NAT to match the selected egress interface.

The logical processing sequence is:

```text
Inside interface
       |
       v
Ingress processing (including applicable PBR)
       |
       v
Routing / egress interface selection
       |
       v
Inside-to-outside NAT
       |
       v
Outside interface
```

For outside-to-inside traffic, translation occurs before the normal routing lookup, allowing the router to forward the packet toward the translated inside local address.

**PBR decides where a packet goes; NAT decides how its addresses and ports are represented.**

See [NAT Packet Processing and Order of Operations](../order-of-operations.md) for the complete explanation.

---

## Multi-ISP NAT and Failover

A common use of policy-based PAT is to connect an inside network to two ISPs, each providing a different outside IPv4 address.

```text
                             +----- ISP1
                             |      E0/1 address: 7.7.7.2
Inside LAN ---- Local_Router-+
                             |      E0/2 address: 8.8.8.2
                             +----- ISP2
```

Both ISP-facing interfaces are configured as `ip nat outside`. A separate route map matches each outgoing interface:

```text
Local_Router(config)# access-list 10 permit 10.1.1.0 0.0.0.255
!
Local_Router(config)# route-map NAT-ISP1 permit 10
Local_Router(config-route-map)# match ip address 10
Local_Router(config-route-map)# match interface Ethernet0/1
Local_Router(config-route-map)# exit
!
Local_Router(config)# route-map NAT-ISP2 permit 10
Local_Router(config-route-map)# match ip address 10
Local_Router(config-route-map)# match interface Ethernet0/2
Local_Router(config-route-map)# exit
!
Local_Router(config)# ip nat inside source route-map NAT-ISP1 interface Ethernet0/1 overload
Local_Router(config)# ip nat inside source route-map NAT-ISP2 interface Ethernet0/2 overload
```

This configuration assumes that the interfaces, addresses, and routing are already configured.

When new traffic is forwarded toward ISP1, it can be translated using ISP1's interface address. When new traffic is forwarded toward ISP2, it can instead use ISP2's interface address.

The forwarding decision may be made by static routes, dynamic routing, or PBR. IP SLA and object tracking can be used to detect a failure and change the active route.

**NAT does not detect an ISP failure or change the routing table to select the backup ISP.** A separate routing or tracking mechanism must do so.

Existing sessions using the failed ISP's address may stop working. A routing change alone does not guarantee that existing NAT state is recreated or that transport-layer sessions survive the change of global address.

See [Multi-ISP NAT and Failover](multi-egress.md) for a complete failover lab.

---

## Multi-Ingress NAT

Multi-ingress designs involve traffic arriving through different outside interfaces, such as two ISP links providing access to the same inside server.

For example, a server might be represented by one global address through ISP1 and another global address through ISP2. Inbound packets must be delivered to the correct inside server, and replies must use the outside representation expected by the original client.

Conditional static mappings, existing NAT state, routing, and the return path all need to be considered.

Do not interpret `match interface` in a NAT route map as an ingress-interface match. It refers to the outgoing interface selected for forwarding.

Dynamic NAT configured with a route map also has different outside-initiated connection behavior from ordinary simple dynamic NAT. Existing extended entries support return traffic for the flows they identify, rather than creating a general inbound mapping for arbitrary outside hosts. The `reversible` keyword on supported dynamic NAT rules enables certain outside-initiated connections using reverse route map matching and an existing inside-to-global allocation.

See [Multi-Ingress NAT](multi-ingress.md) for the configuration and inbound/return-path considerations.

---

## Twice NAT and Overlapping Networks

**Twice NAT** translates both the source and destination IP addresses of a packet.

For example, two networks might both use `10.1.1.0/24`. A host on one network cannot distinguish a remote host from a local host by its actual address alone.

A NAT router can give each network a distinct translated representation:

| Address type | Example |
|---|---|
| Inside local | `10.1.1.5` |
| Inside global | `172.16.1.5` |
| Outside local | `172.16.2.7` |
| Outside global | `10.1.1.7` |

For traffic from the inside host to the outside host:

| Packet | Source | Destination |
|---|---|---|
| Before NAT | `10.1.1.5` | `172.16.2.7` |
| After NAT | `172.16.1.5` | `10.1.1.7` |

The inside host communicates with the outside local address, while the outside host sees the inside global address.

On IOS XE, inside source and outside source NAT rules can be combined to achieve this behavior. The forwarding design must ensure that traffic reaches the NAT router instead of being incorrectly treated as local traffic or routed back toward the wrong network.

See [Twice NAT and Overlapping Networks](twice-nat.md) for the complete topology, mappings, and routing requirements.

---

## NAT-on-a-Stick with PBR

Traditional inside/outside NAT normally requires traffic to traverse interfaces with different NAT roles.

In a NAT-on-a-stick design, traffic may enter and leave the NAT router through the same physical Layer 3 interface. A loopback interface and PBR can be used to create a logical path through both NAT domains in the classic NAT-on-a-stick design.

This is an example of PBR being used to steer traffic through the translation process, rather than to select a different inside global address.

The policy must distinguish the traffic that needs to be redirected from traffic that has already traversed the logical NAT path, to avoid forwarding loops.

See [NAT-on-a-Stick with PBR](nat-on-a-stick.md) for the classic loopback-based design. IOS XE also has a dedicated NAT Stick feature on supported platforms, which is distinct from the classic PBR approach.

---

## Verification and Troubleshooting

The primary verification commands for policy-based NAT are:

```text
Local_Router# show ip nat translations
Local_Router# show ip nat translations verbose
Local_Router# show ip nat statistics
Local_Router# show route-map
Local_Router# show access-lists
Local_Router# show ip route
Local_Router# show ip cef exact-route <source> <destination>
Local_Router# show ip policy
Local_Router# show running-config | include ip nat
```

For a multi-ISP design using IP SLA and tracking, also verify the tracking state and active routes.

When troubleshooting, check the following:

1. **NAT direction:** Are the ingress and egress interfaces assigned the appropriate NAT roles?
2. **Routing:** Which outgoing interface and next hop are selected for the packet? If PBR is configured, check its policy as well; a CEF lookup alone does not prove the PBR result.
3. **Classification:** Do the original packet fields and selected forwarding path satisfy the NAT route map?
4. **Rule selection:** Are there overlapping NAT rules or unconditional mappings that could take precedence over the intended rule?
5. **Translation state:** Does an existing translation match the packet? Check the actual inside global address and outside endpoint.
6. **Resources:** Does the selected pool or PAT rule have available translation resources?
7. **Return reachability:** Can the outside network return traffic to the selected inside global address through the NAT router?
8. **Reverse translation:** Is the return packet consistent with the existing translation or configured static mapping?

`show ip nat translations` is especially important because the route map configuration alone does not prove which translation a particular flow is using.

---

## Key Points

- **Policy-based NAT** applies translations according to configured conditions.
- A NAT route map can match packet addresses, outgoing interfaces, and selected next hops on supporting IOS XE routers.
- Route-map-based NAT can be used with both static and dynamic translation rules, including PAT.
- `overload` enables dynamic PAT; it is not required for policy-based NAT.
- Route-map-based dynamic NAT creates extended translation entries even without overload.
- The same inside local address can use different inside global addresses for different destinations.
- A NAT ACL or route map does not itself filter packets.
- NAT route maps select translations; PBR route maps select forwarding paths.
- Inside-to-outside routing occurs before NAT, allowing a NAT route map to match the selected outgoing interface.
- In multi-ISP designs, NAT and routing policies must agree on which outside addresses and paths are used.
- Existing NAT translations can affect the results of policy changes and failover.
- Twice NAT and NAT-on-a-stick illustrate how translation can be combined with specialized forwarding designs.

---

## References

- [Cisco IOS XE 17.x: Configuring NAT for IP Address Conservation](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-addressing/b-ip-addressing/m_iadnat-addr-consv-xe.html)
- [Cisco: NAT Support for Multiple Pools Using Route Maps](https://www.cisco.com/c/en/us/support/docs/ip/network-address-translation-nat/13739-nat-routemap.html)
- [Cisco IOS XE Release 2: Configuring NAT for IP Address Conservation](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/ipaddr_nat/configuration/xe-2/nat-xe-2-book/iadnat-addr-consv.html)
- [Cisco: NAT Routemaps Outside-to-Inside Support](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/ipaddr_nat/configuration/15-mt/nat-15-mt-book/iadnat-rmap-outin.html)
- [Cisco: IOS NAT Load-Balancing with Optimized Edge Routing for Two Internet Connections](https://www.cisco.com/c/en/us/support/docs/ip/network-address-translation-nat/99427-ios-nat-2isp.html)
- [Cisco: Network Address Translation on a Stick](https://www.cisco.com/c/en/us/support/docs/ip/network-address-translation-nat/6505-nat-on-stick.html)
- [Cisco IOS XE: NAT Stick Overview](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/ip-addressing/ip-addressing/m_nat-on-stick.html)
