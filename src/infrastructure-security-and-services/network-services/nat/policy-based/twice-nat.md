# Twice NAT and Overlapping Networks

**Twice NAT** translates both the source and destination IP addresses of a packet as it crosses a NAT boundary. On Cisco IOS XE, this can be accomplished by combining **inside source NAT** and **outside source NAT** on the same router.

A common use case is connecting two networks that use overlapping IPv4 address spaces. For example, two companies that both use `10.1.1.0/24` might need to communicate after a merger without immediately renumbering either network.

Each network can be represented by a different translated prefix when viewed from the other network:

| Network | Actual prefix | Prefix seen by the other network |
|---|---|---|
| Inside | `10.1.1.0/24` | `172.16.1.0/24` |
| Outside | `10.1.1.0/24` | `172.16.2.0/24` |

Inside hosts use `172.16.2.0/24` to reach outside hosts. Outside hosts use `172.16.1.0/24` to reach inside hosts.

**Twice NAT does not require two NAT routers.** Both address translations can be performed on Local_Router. It also does not inherently require PAT, route maps, or VRFs.

---

## Why Overlapping Networks Cannot Communicate Normally

Suppose LH5 and RH7 are on separate LANs, but both LANs use `10.1.1.0/24`:

```text
LH5: 10.1.1.5/24                         RH7: 10.1.1.7/24
       |                                         |
  Inside LAN ---- Local_Router ---- ISP1 ---- Outside LAN
```

If LH5 attempts to communicate with `10.1.1.7`, it considers the destination directly connected because the address belongs to its own subnet.

Instead of forwarding the packet to its default gateway, LH5 sends an ARP Request for `10.1.1.7` on the inside LAN.

RH7 is on a different LAN, so it does not receive the ARP Request. If an inside host also uses `10.1.1.7`, LH5 might communicate with that host instead.

There is also a routing problem at Local_Router. A normal routing lookup for `10.1.1.7` selects the connected inside subnet, not the outside LAN behind ISP1.

Translating only one network is insufficient for this design:

| Translation | Remaining problem |
|---|---|
| Inside source NAT only | Inside hosts still address outside hosts using an overlapping prefix. |
| Outside source NAT only | Outside hosts still see inside hosts using an overlapping prefix and can attempt to send replies on their own LAN. |
| Inside source NAT and outside source NAT | Both sides communicate using non-overlapping representations of the remote hosts. |

The translated remote addresses must be outside the hosts' own on-link prefixes, and routing must deliver traffic for those addresses to the NAT router.

---

## NAT Address Terminology

Recall the four NAT address types introduced in [NAT Address Terminology and Translation Types](../terminology.md).

For communication between LH5 and RH7:

| Address type | Address | Meaning |
|---|---|---|
| Inside local (IL) | `10.1.1.5` | LH5's actual address |
| Inside global (IG) | `172.16.1.5` | LH5 as seen by outside hosts |
| Outside local (OL) | `172.16.2.7` | RH7 as seen by inside hosts |
| Outside global (OG) | `10.1.1.7` | RH7's actual address |

```text
                  Inside view                Outside view

  LH5 (inside):    10.1.1.5    <---------->    172.16.1.5
                       IL                         IG

  RH7 (outside):   172.16.2.7  <---------->    10.1.1.7
                       OL                         OG
```

The terms **local** and **global** refer to the address representations on each side of the NAT boundary. A global address does not have to be a public IPv4 address.

In this design, all four addresses are private IPv4 addresses.

### Which Fields Are Translated?

| Direction | Inside source NAT | Outside source NAT |
|---|---|---|
| Inside to outside | Source: IL to IG | Destination: OL to OG |
| Outside to inside | Destination: IG to IL | Source: OG to OL |

Inside source NAT always translates the inside host's address. Outside source NAT always translates the outside host's address. Which packet field is modified depends on the direction of the packet.

---

## Lab Topology

We will return to the single-ISP topology used earlier in the NAT section, with one change: the remote LAN now uses `10.1.1.0/24` instead of `3.3.3.0/24`.

This exercise is adapted from Case 12 of [Jedadiah Casey's NAT article](https://neckercube.com/portfolio/posts/2019-03-18-ya-nat-yet-another-network-address-translation-post/), using the translated prefixes introduced in the [Policy-Based NAT Overview](overview.md).

```text
       INSIDE NETWORK                         OUTSIDE NETWORK

  LH5: 10.1.1.5/24
          |
          |                   Local_Router                    ISP1
          +---- LAN ---------- E0/0    E0/1 ---------------- E0/1
          |                10.1.1.1   7.7.7.2              7.7.7.1
          |                NAT inside NAT outside              |
  LH6: 10.1.1.6/24                                  E0/0: 10.1.1.1/24
                                                               |
                                                          Outside LAN
                                                               |
                                                        +------+------+
                                                        |             |
                                                  RH7 10.1.1.7   RH8 10.1.1.8
```

| Device | Interface | IPv4 address | Default gateway |
|---|---|---|---|
| LH5 | LAN | `10.1.1.5/24` | `10.1.1.1` |
| LH6 | LAN | `10.1.1.6/24` | `10.1.1.1` |
| Local_Router | Ethernet0/0 | `10.1.1.1/24` | N/A |
| Local_Router | Ethernet0/1 | `7.7.7.2/24` | N/A |
| ISP1 | Ethernet0/1 | `7.7.7.1/24` | N/A |
| ISP1 | Ethernet0/0 | `10.1.1.1/24` | N/A |
| RH7 | LAN | `10.1.1.7/24` | `10.1.1.1` |
| RH8 | LAN | `10.1.1.8/24` | `10.1.1.1` |

The two LANs must be separate Layer 2 domains. Local_Router and ISP1 can both use `10.1.1.1/24` on their respective LAN-facing interfaces because those interfaces are in different broadcast domains.

**Do not configure both overlapping LAN interfaces in the same routing table on Local_Router.** ISP1 provides Layer 3 separation between Local_Router and the outside LAN, while the transit link uses a unique subnet.

Before beginning, remove the previous NAT mappings, PBR policies, static routes (including default routes), and tracking configuration that would affect this exercise. ISP2 is not used in this lab.

---

## Lab 1: Static Host-to-Host Twice NAT

### Requirements

- Allow LH5 and RH7 to initiate connections to each other.
- LH5 must appear as `172.16.1.5` to RH7.
- RH7 must appear as `172.16.2.7` to LH5.
- Do not change the actual IP addresses of either host.
- Do not configure PAT.

### Step 1: Configure Interfaces

On Local_Router:

```text
Local_Router(config)# interface Ethernet0/0
Local_Router(config-if)# ip address 10.1.1.1 255.255.255.0
Local_Router(config-if)# ip nat inside
Local_Router(config-if)# no shutdown
Local_Router(config-if)# exit
!
Local_Router(config)# interface Ethernet0/1
Local_Router(config-if)# ip address 7.7.7.2 255.255.255.0
Local_Router(config-if)# ip nat outside
Local_Router(config-if)# no shutdown
Local_Router(config-if)# exit
```

On ISP1:

```text
ISP1(config)# interface Ethernet0/1
ISP1(config-if)# ip address 7.7.7.1 255.255.255.0
ISP1(config-if)# no shutdown
ISP1(config-if)# exit
!
ISP1(config)# interface Ethernet0/0
ISP1(config-if)# ip address 10.1.1.1 255.255.255.0
ISP1(config-if)# no shutdown
ISP1(config-if)# exit
```

ISP1 does not perform NAT. The inside and outside interface roles are defined only on Local_Router.

### Step 2: Configure Both Static Mappings

```text
Local_Router(config)# ip nat inside source static 10.1.1.5 172.16.1.5
Local_Router(config)# ip nat outside source static 10.1.1.7 172.16.2.7
```

The first command translates LH5's address. The second command translates RH7's address.

Pay attention to the argument order:

```text
ip nat inside source static  <inside-local>   <inside-global>
ip nat outside source static <outside-global> <outside-local>
```

In both cases, the host's actual address is specified first, followed by its translated address.

### Step 3: Configure Routing on Local_Router

```text
Local_Router(config)# ip route 172.16.2.0 255.255.255.0 7.7.7.1
```

This route is for RH7's **outside local** address, not its outside global address.

When LH5 sends traffic to `172.16.2.7`, Local_Router performs its initial routing lookup using that destination. The route selects ISP1 as the next hop and Ethernet0/1 as the outgoing interface. NAT then changes the destination to `10.1.1.7`.

A route to `10.1.1.7` through ISP1 is not required on Local_Router for this design. In fact, adding such a route could incorrectly divert traffic destined for an inside host using that address.

### Step 4: Configure Routing on ISP1

```text
ISP1(config)# ip route 172.16.1.0 255.255.255.0 7.7.7.2
```

This route delivers packets addressed to LH5's **inside global** address to Local_Router.

After receiving a packet destined for `172.16.1.5`, Local_Router translates the destination to `10.1.1.5` and routes it toward the inside LAN.

### Routing Summary

| Device | Destination prefix or host | Forwarding path |
|---|---|---|
| Local_Router | `10.1.1.0/24` | Connected inside LAN |
| Local_Router | `7.7.7.0/24` | Connected transit link |
| Local_Router | `172.16.2.7/32` | ISP1 (`7.7.7.1`) |
| ISP1 | `10.1.1.0/24` | Connected outside LAN |
| ISP1 | `7.7.7.0/24` | Connected transit link |
| ISP1 | `172.16.1.5/32` | Local_Router (`7.7.7.2`) |

Neither router requires a default route for this exercise. Neither translated prefix needs to be assigned to an interface.

---

## Packet Flow: Inside to Outside

LH5 initiates a ping to RH7 using its outside local address:

```text
LH5# ping 172.16.2.7
```

### Step 1: LH5 Sends the Packet to Local_Router

The destination `172.16.2.7` is not on LH5's LAN. LH5 sends the packet to its default gateway, `10.1.1.1`.

```text
Source:      10.1.1.5       (IL)
Destination: 172.16.2.7     (OL)
```

### Step 2: Local_Router Selects the Outside Path

For inside-to-outside traffic, **routing is performed before NAT**.

Local_Router looks up `172.16.2.7` and selects the next hop `7.7.7.1` through its NAT outside interface.

### Step 3: Local_Router Translates Both Addresses

- Inside source NAT changes the source from `10.1.1.5` to `172.16.1.5`.
- Outside source NAT changes the destination from `172.16.2.7` to `10.1.1.7`.

```text
Source:      172.16.1.5     (IG)
Destination: 10.1.1.7       (OG)
```

### Step 4: ISP1 Delivers the Packet to RH7

ISP1 receives the translated packet from Local_Router. Its connected route to `10.1.1.0/24` allows it to forward the packet to RH7.

RH7 sees an Echo Request from `172.16.1.5`, not `10.1.1.5`.

```text
 LH5                 Local_Router                 ISP1               RH7
  |                        |                       |                   |
  |  10.1.1.5 ->           |                       |                   |
  |       172.16.2.7       |                       |                   |
  +----------------------->|                       |                   |
                           | Route to 172.16.2.7   |                   |
                           | Translate both IPs    |                   |
                           |                       |                   |
                           |  172.16.1.5 ->        |                   |
                           |        10.1.1.7       |                   |
                           +---------------------->+------------------>|
```

**The initial routing lookup on Local_Router uses the outside local address. ISP1's routing lookup uses the outside global address.**

If Local_Router performed an ordinary routing lookup for the outside global address before selecting the outside path, the connected inside route would point in the wrong direction.

---

## Packet Flow: Outside to Inside

RH7 replies to the Echo Request. The Echo Reply is destined for `172.16.1.5`, which RH7 considers remote.

### Step 1: RH7 Sends the Reply to ISP1

```text
Source:      10.1.1.7       (OG)
Destination: 172.16.1.5     (IG)
```

RH7 sends the packet to its default gateway, `10.1.1.1` (ISP1).

ISP1's static route to `172.16.1.5` forwards the packet to Local_Router via `7.7.7.2`.

### Step 2: Local_Router Translates Both Addresses

For outside-to-inside traffic, **NAT is performed before routing**.

- Outside source NAT changes the source from `10.1.1.7` to `172.16.2.7`.
- Inside source NAT changes the destination from `172.16.1.5` to `10.1.1.5`.

```text
Source:      172.16.2.7     (OL)
Destination: 10.1.1.5       (IL)
```

### Step 3: Local_Router Delivers the Reply to LH5

Local_Router routes the packet using the translated destination, `10.1.1.5`, and forwards it through Ethernet0/0.

LH5 receives an Echo Reply from `172.16.2.7`, the address it originally contacted.

```text
 LH5                 Local_Router                 ISP1               RH7
  |                        |                       |                   |
  |                        |                       |  10.1.1.7 ->      |
  |                        |                       |       172.16.1.5  |
  |                        |<----------------------+<------------------+
  |                        | Translate both IPs    |                   |
  |                        | Route to 10.1.1.5     |                   |
  |  172.16.2.7 ->         |                       |                   |
  |        10.1.1.5        |                       |                   |
  |<-----------------------+                       |                   |
```

### Complete Packet Flow Summary

| Packet | Source before NAT | Destination before NAT | Source after NAT | Destination after NAT |
|---|---|---|---|---|
| Echo Request (inside to outside) | `10.1.1.5` | `172.16.2.7` | `172.16.1.5` | `10.1.1.7` |
| Echo Reply (outside to inside) | `10.1.1.7` | `172.16.1.5` | `172.16.2.7` | `10.1.1.5` |

A useful way to remember twice NAT is to think about the two address spaces:

- On the inside, packets contain an IL address and an OL address.
- On the outside, packets contain an IG address and an OG address.

See [NAT Packet Processing and Order of Operations](../order-of-operations.md) for the general routing/NAT processing model.

---

## Verification

### Verify Bidirectional Connectivity

```text
LH5# ping 172.16.2.7
RH7# ping 172.16.1.5
```

Both hosts should be able to initiate traffic using the translated address of the other host. Single-address static mappings do not require a prior connection to allocate their addresses.

For a TCP test, enable a TCP listener on the destination host, then initiate a connection to its translated address. For example, if RH7 has a Telnet server listening on TCP/23:

```text
LH5# telnet 172.16.2.7
```

Repeat in the opposite direction with a listener on LH5.

### Verify the Translation Table

```text
Local_Router# show ip nat translations
Local_Router# show ip nat translations verbose
```

Before traffic is generated, the two static address mappings can be represented as follows:

```text
Pro  Inside global      Inside local       Outside local      Outside global
---  172.16.1.5         10.1.1.5           ---                ---
---  ---                ---                172.16.2.7         10.1.1.7
```

After establishing a TCP connection from LH5 to RH7, look for an extended entry containing all four address representations. For example, assuming LH5 uses source port 51000:

```text
Pro  Inside global       Inside local        Outside local      Outside global
tcp  172.16.1.5:51000    10.1.1.5:51000      172.16.2.7:23      10.1.1.7:23
```

This is illustrative output. The actual port number and displayed parent/child entries depend on the traffic and NAT implementation.

All four addresses differ in this example, even though IL and OG belong to the same prefix.

The ports are unchanged because the rules perform address-only NAT. An extended entry containing port numbers does not necessarily indicate PAT.

### Verify Routing

```text
Local_Router# show ip route 172.16.2.7
Local_Router# show ip route 10.1.1.5
Local_Router# show ip cef 172.16.2.7
ISP1# show ip route 172.16.1.5
ISP1# show ip route 10.1.1.7
```

Notice that `show ip route 10.1.1.7` on Local_Router identifies the inside LAN. That is expected in this topology. It is not evidence that the translated outbound packet will be sent toward the inside LAN.

### Verify the Actual Packets

Capture the Echo Request and Echo Reply on the inside LAN and the transit link.

| Capture location | Echo Request | Echo Reply |
|---|---|---|
| Inside LAN | `10.1.1.5` to `172.16.2.7` | `172.16.2.7` to `10.1.1.5` |
| Transit link | `172.16.1.5` to `10.1.1.7` | `10.1.1.7` to `172.16.1.5` |
| Outside LAN | `172.16.1.5` to `10.1.1.7` | `10.1.1.7` to `172.16.1.5` |

Alternatively, enable NAT debugging on Local_Router:

```text
Local_Router# debug ip nat
```

Look for both source and destination changes for each direction. Disable debugging after the test:

```text
Local_Router# undebug all
```

---

## Lab 2: Static Network-to-Network Twice NAT

The previous exercise translates only LH5 and RH7. To allow all hosts in the two overlapping /24 networks to communicate using predictable translated addresses, configure two static **network mappings** instead.

### Step 1: Remove the Host Mappings

Stop active traffic and clear dynamic flow entries before replacing the mappings.

```text
Local_Router# clear ip nat translation *
Local_Router# configure terminal
Local_Router(config)# no ip nat inside source static 10.1.1.5 172.16.1.5
Local_Router(config)# no ip nat outside source static 10.1.1.7 172.16.2.7
```

### Step 2: Configure Network Mappings

```text
Local_Router(config)# ip nat inside source static network 10.1.1.0 172.16.1.0 /24
Local_Router(config)# ip nat outside source static network 10.1.1.0 172.16.2.0 /24
```

The host bits are preserved in both mappings.

| Host | Actual address | Translated address |
|---|---|---|
| LH5 | `10.1.1.5` | `172.16.1.5` |
| LH6 | `10.1.1.6` | `172.16.1.6` |
| RH7 | `10.1.1.7` | `172.16.2.7` |
| RH8 | `10.1.1.8` | `172.16.2.8` |

Although both commands reference the same actual prefix (`10.1.1.0/24`), they represent hosts on different sides of the NAT boundary.

### Step 3: Test the Mappings

```text
LH5# ping 172.16.2.7
LH5# ping 172.16.2.8
LH6# ping 172.16.2.7
LH6# ping 172.16.2.8
RH7# ping 172.16.1.5
RH8# ping 172.16.1.6
```

Also test new connections initiated from each side before the other host has generated traffic. In particular, distinguish a configured static network mapping from its individual host entries.

For ordinary static network translation, the translated address is predetermined even if an individual host entry has not yet been created. Inspect whether the target IOS XE image can create the required host entries on the initial packet in both directions. If cold-start initiation through the outside source network mapping fails, use the explicit host mappings from Lab 1 for services that require reliable bidirectional initiation, and investigate the network-entry lookup behavior separately.

### Step 4: Inspect the Translation Table

```text
Local_Router# show ip nat translations 
Pro  Inside global         Inside local          Outside local         Outside global
---  172.16.1.0            10.1.1.0              ---                   ---
---  ---                   ---                   172.16.2.0            10.1.1.0              
---  ---                   ---                   172.16.2.7            10.1.1.7              
---  ---                   ---                   172.16.2.8            10.1.1.8              
---  172.16.1.6            10.1.1.6              ---                   ---
---  172.16.1.5            10.1.1.5              ---                   ---
icmp 172.16.1.5:0          10.1.1.5:0            172.16.2.7:0          10.1.1.7:0
icmp 172.16.1.5:3          10.1.1.5:3            172.16.2.7:3          10.1.1.7:3
icmp 172.16.1.6:0          10.1.1.6:0            172.16.2.7:0          10.1.1.7:0
icmp 172.16.1.5:4          10.1.1.5:4            172.16.2.8:4          10.1.1.8:4
icmp 172.16.1.6:0          10.1.1.6:0            172.16.2.8:0          10.1.1.8:0
icmp 172.16.1.6:1          10.1.1.6:1            172.16.2.8:1          10.1.1.8:1
Total number of translations: 12
```

There are three conceptual levels of translation state:

| Entry | Purpose |
|---|---|
| Static network entry | Defines the prefix-to-prefix mapping |
| Individual host entry | Represents a specific address pair derived from the network mapping |
| Extended entry | Represents a specific protocol/flow using both address mappings |

The configured network mappings remain in place even when individual flow entries age out.

Because the translated prefixes are explicitly routed to the NAT router rather than treated as directly connected networks, the design does not depend on ARP aliases for each translated host address. See [Static Network-to-Network NAT](../static/network-mapping.md) for the difference between a static prefix mapping and its derived host entries.

---

## Identical Host Addresses on Both Sides

Twice NAT can also resolve an overlap in which the communicating hosts use the exact same IPv4 address.

Suppose LH5 uses `10.1.1.5` and a different outside host also uses `10.1.1.5`. With the network mappings from Lab 2, the two hosts have different translated representations:

| Address type | Address |
|---|---|
| IL | `10.1.1.5` |
| IG | `172.16.1.5` |
| OL | `172.16.2.5` |
| OG | `10.1.1.5` |

Inside-to-outside traffic:

```text
Before NAT: 10.1.1.5   -> 172.16.2.5
After NAT:  172.16.1.5 -> 10.1.1.5
```

Outside-to-inside traffic:

```text
Before NAT: 10.1.1.5   -> 172.16.1.5
After NAT:  172.16.2.5 -> 10.1.1.5
```

The same numeric address identifies different hosts in different network domains. The NAT interface roles and distinct translated prefixes make the two identities distinguishable.

To test this, readdress RH8 to `10.1.1.5/24` (assuming no other host on the outside LAN uses that address), retaining its default gateway. LH5 should contact it using `172.16.2.5`, while RH8 should contact LH5 using `172.16.1.5`.

---

## The add-route Keyword and Overlapping Networks

Recall that outside source NAT can use `add-route` to install routes associated with outside local addresses.

In many non-overlapping designs, routing toward the outside global address is sufficient to determine an appropriate next hop for the outside local route. However, in this topology, the outside global address belongs to Local_Router's connected inside subnet.

Therefore, do not rely on automatic route generation to determine the correct path through ISP1. Configure the outside local route explicitly:

```text
Local_Router(config)# ip route 172.16.2.0 255.255.255.0 7.7.7.1
```

This identifies the intended outside forwarding path without relying on resolution through the overlapping outside global prefix.

If experimenting with `add-route`, inspect the actual installed route and its resolved interface. A route to the correct alias with the wrong outgoing interface is still incorrect.

For example, `add-route` can cause communication to fail:

```text
Local_Router# show ip route
...
S        172.16.2.5/32 [1/0] via 10.1.1.5
```

`add-route` installed a host route to the outside local address `172.16.2.5`, using the outside global address `10.1.1.5` as the next hop.

However, because of the overlapping address spaces, Local_Router resolves the next hop `10.1.1.5` via its inside interface, where LH5 is located.

As a result, Local_Router forwards LH5's Echo Requests back to LH5 itself instead of forwarding them through the NAT outside interface. The packets do not undergo NAT.

See [Static Outside Source NAT and PAT](../static/outside-source.md) and [Static NAT Options and ARP Behavior](../static/options.md) for more about `add-route`.

---

## Dynamic NAT, PAT, and Twice NAT

Twice NAT describes which addresses are translated, not how the mappings are allocated.

It is possible to combine different translation mechanisms. For example, consider an alternative requirement:

- Inside hosts must be able to initiate connections to RH7.
- RH7 must be represented by a fixed outside local address.
- Inside hosts do not need individual fixed inside global addresses or unsolicited inbound reachability.

Remove the two static network mappings and configure a static outside source mapping for RH7 with dynamic inside source PAT:

```text
Local_Router(config)# ip nat outside source static 10.1.1.7 172.16.2.7
!
Local_Router(config)# ip access-list extended TO-RH7
Local_Router(config-ext-nacl)# permit ip 10.1.1.0 0.0.0.255 host 172.16.2.7
Local_Router(config-ext-nacl)# exit
!
Local_Router(config)# ip nat pool INSIDE-PAT 172.16.1.100 172.16.1.100 prefix-length 24
Local_Router(config)# ip nat inside source list TO-RH7 pool INSIDE-PAT overload
```

Retain the two network routes from Lab 2. The `172.16.1.100` PAT address is covered by ISP1's route to `172.16.1.0/24`.

The extended ACL matches the outside **local** destination (`172.16.2.7`) presented by the inside host. Generate a new flow and verify the selected rule and both translations.

For example, a connection from LH5 to RH7 could be represented as follows (assuming PAT allocates port 40001):

| Field | Before NAT | After NAT |
|---|---|---|
| Source IP | `10.1.1.5` | `172.16.1.100` |
| Source TCP port | `51000` | `40001` |
| Destination IP | `172.16.2.7` | `10.1.1.7` |
| Destination TCP port | `23` | `23` |

This design reduces inside-global address consumption, but RH7 cannot initiate an arbitrary new connection to a specific inside host using the shared PAT address. The inside source PAT state determines which inside host receives a reply.

A dynamic outside source pool is another possibility, but it introduces an allocation dependency: the outside-local-to-outside-global mapping must exist before an inside host can use the allocated outside local address. For predictable access to known remote services, static outside source mappings are simpler.

---

## DNS and Address Planning

Applications must use the correct representation of remote hosts. Configuring twice NAT does not automatically change application configuration or DNS records.

For example, DNS can be configured to provide the following answers in each network:

| DNS query location | Remote hostname | Address returned |
|---|---|---|
| Inside | RH7's hostname | `172.16.2.7` |
| Outside | LH5's hostname | `172.16.1.5` |

Split DNS, separate DNS zones, or appropriate DNS record management can provide the translated addresses. Some NAT implementations also support DNS payload translation, but it should not be assumed to work for all DNS transports or application protocols.

When choosing translated prefixes:

- Avoid overlap with existing reachable networks on the side where the translated prefix is used.
- Reserve enough addresses for the required mappings.
- Advertise or statically route the translated prefixes toward the NAT boundary.
- Avoid redirecting unrelated destinations by selecting excessively broad alias prefixes.

For large overlaps, it is not always necessary to translate the entire original address space. For example, if only a few servers need connectivity between two organizations that both use `10.0.0.0/8`, individual static mappings or smaller prefix mappings may be sufficient. PAT can also reduce the number of translated source addresses where individual inbound reachability is unnecessary.

Twice NAT solves the address-representation problem, while VRFs can solve the separate problem of maintaining identical real prefixes in independent routing tables. VRF-aware NAT is covered in [4.5.d (iv) VRF-Aware NAT, PAT](../vrf/overview.md).

---

## Troubleshooting

### Test 1: Remove the Outside Local Route

Using the static network configuration from Lab 2, temporarily remove the route on Local_Router:

```text
Local_Router(config)# no ip route 172.16.2.0 255.255.255.0 7.7.7.1
```

Attempt to ping `172.16.2.7` from LH5.

With no default route or other matching route, the ping should fail because Local_Router cannot select an outside forwarding path for the outside local destination before NAT.

Restore the route:

```text
Local_Router(config)# ip route 172.16.2.0 255.255.255.0 7.7.7.1
```

### Test 2: Remove the Inside Global Route

Temporarily remove the route on ISP1:

```text
ISP1(config)# no ip route 172.16.1.0 255.255.255.0 7.7.7.2
```

The outbound Echo Request from LH5 can still reach RH7 with both addresses translated. However, ISP1 cannot forward the Echo Reply to `172.16.1.5` without a matching route.

Restore the route:

```text
ISP1(config)# ip route 172.16.1.0 255.255.255.0 7.7.7.2
```

These two tests demonstrate that the outside local route and inside global route are independently necessary in this topology.

### Common Problems

| Symptom | Likely cause or verification step |
|---|---|
| LH5 ARPs for RH7's actual address | LH5 is using the outside global address instead of the outside local address. |
| LH5's packet reaches Local_Router but does not leave toward ISP1 | Missing or incorrect route to the outside local prefix, incorrect NAT interface roles, or an interfering PBR policy |
| RH7 receives traffic with a source in `10.1.1.0/24` | Inside source NAT did not translate the source. RH7 may ARP for the source locally. |
| ISP1 receives a packet destined for `172.16.2.7` | Outside source NAT did not translate the destination to its outside global address. |
| RH7 replies, but the reply never reaches Local_Router | Missing route to the inside global prefix on ISP1, or incorrect host gateway |
| LH5 receives a reply sourced from `10.1.1.7` | Outside source NAT did not translate the reply's source. |
| Existing flows work but a new destination fails | Check host/flow entries, network mapping lookup, or dynamic address allocation. |
| IP connectivity works but hostname-based access fails | Check DNS answers and application use of translated addresses. |

Useful commands on Local_Router:

```text
Local_Router# show ip nat translations
Local_Router# show ip nat translations verbose
Local_Router# show ip nat statistics
Local_Router# show ip route 172.16.2.7
Local_Router# show ip cef 172.16.2.7
Local_Router# show running-config | include ip nat
Local_Router# show ip policy
```

On ISP1:

```text
ISP1# show ip route 172.16.1.5
ISP1# show ip route 10.1.1.7
ISP1# show ip arp
```

When testing with interface ACLs, remember which version of the packet is visible at each attachment point. For example, an inside input ACL sees IL as the source and OL as the destination, while an outside output ACL sees IG as the source and OG as the destination for the same packet.

---

## Key Points

- Twice NAT translates both the source and destination addresses of a packet.
- Traditional IOS XE twice NAT can be implemented by combining inside source NAT and outside source NAT on one router.
- Overlapping networks require distinct representations of remote addresses so hosts do not treat remote destinations as on-link.
- Inside hosts use outside local addresses to reach outside hosts; outside hosts use inside global addresses to reach inside hosts.
- In the inside-to-outside direction, the initial routing lookup uses the outside local address. The next router receives a packet destined for the outside global address.
- In the outside-to-inside direction, NAT changes the destination from inside global to inside local before the routing lookup.
- Explicit routes to the translated prefixes provide reachability without relying on ARP aliases or automatic route generation.
- Static network-to-network NAT preserves host bits, making address mappings predictable in both directions.
- An extended twice NAT entry can contain four distinct IP addresses without PAT.
- Static host mappings, static network mappings, and dynamic translation mechanisms have different allocation and initiation behavior.
- Twice NAT can resolve overlaps even when the two communicating hosts use identical actual IP addresses.
- DNS and application configuration must account for the translated addresses.

---

## References

- [Cisco IOS XE 17.x: Configuring NAT for IP Address Conservation](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-addressing/b-ip-addressing/m_iadnat-addr-consv-xe.html)
- [Cisco: Using NAT in Overlapping Networks](https://www.cisco.com/c/en/us/support/docs/ip/network-address-translation-nat/13774-3.html)
- [Cisco: Configure NAT to Enable Communication Between Overlapping Networks](https://www.cisco.com/c/en/us/support/docs/ip/network-address-translation-nat/200726-Configure-NAT-to-Enable-Communication-Be.html) (different routing topology and an outside-initiated design; note its documented initiation limitation)
- [Jedadiah Casey: YA-NAT: Yet Another Network Address Translation Post (Case 12)](https://neckercube.com/portfolio/posts/2019-03-18-ya-nat-yet-another-network-address-translation-post/)
