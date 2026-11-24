# Static Outside Source NAT and PAT

**Static outside source NAT** creates a fixed mapping between an **outside global** IPv4 address and an **outside local** IPv4 address.

Whereas inside source NAT changes how an inside host appears to outside devices, outside source NAT changes how an outside host appears to inside devices.

Outside source NAT is useful when:

- Inside and outside networks use overlapping IP address spaces.
- Outside servers must be represented using a different addressing scheme within the inside network.
- Inside hosts must access outside services using alternative IP addresses or TCP/UDP ports.
- Both inside and outside addresses need to be translated as packets cross a NAT boundary.

The name **outside source** identifies which host's address is being translated:

- **Outside to inside:** The packet's source address changes from outside global to outside local.
- **Inside to outside:** The packet's destination address changes from outside local to outside global.

As with static inside source NAT, the mapping is bidirectional and can support connections initiated from either side.

However, outside source NAT introduces important routing considerations because of the order in which routing and NAT occur.

---

## Configuration Syntax

The basic command is:

```text
Router(config)# ip nat outside source static <outside-global> <outside-local>
```

For example:

```text
Local_Router(config)# ip nat outside source static 3.3.3.7 172.16.1.7
```

This creates the following mapping:

```text
Outside global                       Outside local
    3.3.3.7        <-------------->    172.16.1.7
```

| Argument | Example | Meaning |
|---|---|---|
| Outside global | `3.3.3.7` | The outside host's actual IP address |
| Outside local | `172.16.1.7` | The address representing the outside host to inside devices |

**Outside global comes first, followed by outside local.**

Compare this with static inside source NAT:

```text
ip nat inside source static <inside-local> <inside-global>
ip nat outside source static <outside-global> <outside-local>
```

In both commands, the original address of the host being translated comes first, and its translated address comes second.

### Inside Source vs. Outside Source NAT

| Feature | Inside Source NAT | Outside Source NAT |
|---|---|---|
| Host being represented | Inside host | Outside host |
| First address argument | Inside local | Outside global |
| Second address argument | Inside global | Outside local |
| Inside-to-outside translation | Source | Destination |
| Outside-to-inside translation | Destination | Source |
| Common use | Represent inside hosts externally | Represent outside hosts internally |

Neither command restricts communication to sessions initiated from the named side.

---

## Lab Topology

We'll continue using the topology from the previous static NAT lessons.

```text
       INSIDE NETWORK                         OUTSIDE NETWORK

  Local_Host_5 (LH5)
     10.1.1.5/24
          |
          |                   Local_Router                    ISP1
          +---- LAN ---------- E0/0    E0/1 ---------------- E0/1
          |                10.1.1.1   7.7.7.2              7.7.7.1
          |                NAT inside NAT outside              |
  Local_Host_6 (LH6)                                           |
     10.1.1.6/24                                   E0/0: 3.3.3.1/24
                                                               |
                                                          Outside LAN
                                                               |
                                                        +------+------+
                                                        |             |
                                                Remote_Host_7  Remote_Host_8
                                                 3.3.3.7/24    3.3.3.8/24
```

| Device | Interface or role | Address |
|---|---|---|
| Local_Host_5 | Inside host | `10.1.1.5/24` |
| Local_Host_6 | Inside host | `10.1.1.6/24` |
| Local_Router | `E0/0`, NAT inside | `10.1.1.1/24` |
| Local_Router | `E0/1`, NAT outside | `7.7.7.2/24` |
| ISP1 | Toward Local_Router | `7.7.7.1/24` |
| ISP1 | Toward remote hosts | `3.3.3.1/24` |
| Remote_Host_7 | Outside host to translate | `3.3.3.7/24` |
| Remote_Host_8 | Additional outside host | `3.3.3.8/24` |

Our objective is to make RH7 appear to inside hosts as `172.16.1.7` rather than `3.3.3.7`.

```text
Actual address of RH7:                 3.3.3.7
Address used by inside hosts:        172.16.1.7
```

RH7 continues using `3.3.3.7` on its interface. The address `172.16.1.7` exists as its translated representation on the inside network.

### Routing Assumptions

For this exercise:

- LH5 and LH6 use `10.1.1.1` as their default gateway.
- RH7 and RH8 use `3.3.3.1` as their default gateway.
- Local_Router has a default route toward ISP1.
- ISP1 knows the two directly connected outside networks.
- ISP1 has a route to the inside network via Local_Router.

The final assumption is different from the earlier inside source NAT exercises.

Because we are translating the **outside host**, the inside hosts' addresses remain unchanged. The outside network must therefore have a return path to those addresses.

Remove the NAT mappings from the previous exercises before beginning this lab.

---

## Configure Static Outside Source NAT

### Step 1: Configure the Inside Interface

```text
Local_Router(config)# interface Ethernet0/0
Local_Router(config-if)# ip address 10.1.1.1 255.255.255.0
Local_Router(config-if)# ip nat inside
Local_Router(config-if)# no shutdown
```

### Step 2: Configure the Outside Interface

```text
Local_Router(config)# interface Ethernet0/1
Local_Router(config-if)# ip address 7.7.7.2 255.255.255.0
Local_Router(config-if)# ip nat outside
Local_Router(config-if)# no shutdown
```

The interface roles are identical to those used for inside source NAT.

We do **not** reverse the `ip nat inside` and `ip nat outside` interface commands when configuring outside source NAT.

### Step 3: Configure Static Outside Source NAT

```text
Local_Router(config)# ip nat outside source static 3.3.3.7 172.16.1.7
```

The configured mapping is:

| Address type | Value |
|---|---|
| Outside global | `3.3.3.7` |
| Outside local | `172.16.1.7` |

No inside source NAT mapping has been configured.

### Step 4: Configure Routing on Local_Router

```text
Local_Router(config)# ip route 0.0.0.0 0.0.0.0 7.7.7.1
```

In this simple single-egress topology, the default route provides an outside forwarding path for both the outside global and outside local addresses.

However, relying on the default route is not appropriate for every outside source NAT design.

To explicitly provide reachability to the outside local address, configure:

```text
Local_Router(config)# ip route 172.16.1.7 255.255.255.255 7.7.7.1
```

Alternatively, the `add-route` keyword can automatically create a route associated with the outside source mapping. This is covered later in this page.

### Step 5: Configure Return Routing on ISP1

```text
ISP1(config)# ip route 10.1.1.0 255.255.255.0 7.7.7.2
```

This route is necessary in our lab because LH5 and LH6 are not being translated.

For example, when LH5 communicates with RH7, RH7 sees the source IP address `10.1.1.5`.

ISP1 must know how to reach that address.

**Outside source NAT does not hide the inside hosts' addresses.**

---

## Verify the Static Mapping

Before generating any traffic:

```text
Local_Router# show ip nat translations
Pro  Inside global      Inside local       Outside local      Outside global
---  ---                ---                172.16.1.7         3.3.3.7
```

This is a representative **simple static outside source NAT entry**.

Notice that:

- The inside global and inside local columns are empty.
- The outside local column contains the translated address.
- The outside global column contains RH7's actual address.
- The protocol column contains `---`, indicating an address-only mapping.

The static mapping exists before either host initiates communication.

**The order of addresses in the NAT translation table differs from the argument order of the configuration command.**

```text
Configuration command:  Outside global, Outside local
Translation table:       Outside local, Outside global
```

---

## Outside-to-Inside Packet Flow

Suppose RH7 initiates a ping to LH5:

```text
Remote_Host_7# ping 10.1.1.5
```

The original ICMP Echo Request contains:

```text
Source:       3.3.3.7
Destination:  10.1.1.5
```

ISP1 forwards the packet toward Local_Router using its static route to `10.1.1.0/24`.

When the packet arrives at Local_Router's outside interface, it matches the outside source NAT mapping.

Local_Router changes the packet's **source address**:

```text
3.3.3.7 -> 172.16.1.7
```

The translated packet contains:

```text
Source:       172.16.1.7
Destination:  10.1.1.5
```

### Packet Translation

| Field | Before NAT | After NAT |
|---|---|---|
| Source IP | `3.3.3.7` | `172.16.1.7` |
| Destination IP | `10.1.1.5` | `10.1.1.5` |

The destination address remains unchanged.

### Packet Flow Diagram

```text
RH7                         ISP1                 Local_Router                LH5
 |                            |                        |                      |
 | Src 3.3.3.7                |                        |                      |
 | Dst 10.1.1.5               |                        |                      |
 +--------------------------->+----------------------->|                      |
                                                       | Outside source NAT   |
                                                       | 3.3.3.7              |
                                                       |    -> 172.16.1.7     |
                                                       |                      |
                                                       | Src 172.16.1.7       |
                                                       | Dst 10.1.1.5         |
                                                       +--------------------->|
```

LH5 receives an ICMP Echo Request that appears to originate from `172.16.1.7`.

LH5 does not need to know RH7's actual address.

For traditional outside-to-inside NAT, translation occurs before the routing decision that forwards the packet to the inside host.

---

## Inside-to-Outside Return Traffic

LH5 sends an ICMP Echo Reply to the source address of the received Echo Request:

```text
Source:       10.1.1.5
Destination:  172.16.1.7
```

LH5 forwards the packet to its default gateway, Local_Router.

Local_Router must first perform a routing lookup for:

```text
172.16.1.7
```

The destination is still the **outside local address** at this point.

After selecting the outside forwarding path, Local_Router applies the static outside source mapping in reverse:

```text
Destination: 172.16.1.7 -> 3.3.3.7
```

The translated packet contains:

```text
Source:       10.1.1.5
Destination:  3.3.3.7
```

### Packet Translation

| Field | Before NAT | After NAT |
|---|---|---|
| Source IP | `10.1.1.5` | `10.1.1.5` |
| Destination IP | `172.16.1.7` | `3.3.3.7` |

### Packet Flow Diagram

```text
LH5                      Local_Router                   ISP1                  RH7
 |                             |                          |                     |
 | Src 10.1.1.5                |                          |                     |
 | Dst 172.16.1.7              |                          |                     |
 +---------------------------->|                          |                     |
                               | Route toward outside     |                     |
                               | using 172.16.1.7         |                     |
                               |                          |                     |
                               | Destination NAT          |                     |
                               | 172.16.1.7 -> 3.3.3.7    |                     |
                               |                          |                     |
                               | Src 10.1.1.5             |                     |
                               | Dst 3.3.3.7              |                     |
                               +------------------------->+-------------------->|
```

RH7 receives a reply from `10.1.1.5`, the address it originally contacted.

**Key point:** The `ip nat outside source static` command translates the destination address of inside-to-outside traffic.

---

## Inside-Initiated Traffic

Outside source NAT is not limited to connections initiated by outside hosts.

LH5 can initiate communication to RH7 using the outside local address:

```text
Local_Host_5# ping 172.16.1.7
```

The packet is translated as follows:

| Packet | Source | Destination |
|---|---|---|
| Echo Request before NAT | `10.1.1.5` | `172.16.1.7` |
| Echo Request after NAT | `10.1.1.5` | `3.3.3.7` |
| Echo Reply before NAT | `3.3.3.7` | `10.1.1.5` |
| Echo Reply after NAT | `172.16.1.7` | `10.1.1.5` |

The complete communication uses two different representations of RH7:

- LH5 communicates with `172.16.1.7`.
- RH7 communicates using its actual address, `3.3.3.7`.

Because the mapping is static, RH7 does not need to initiate traffic before LH5 can use the outside local address.

### Should LH5 Ping the Outside Global Address?

The intended test is:

```text
LH5# ping 172.16.1.7
```

Not:

```text
LH5# ping 3.3.3.7
```

If LH5 sends traffic directly to `3.3.3.7`, the destination does not match the configured outside-local-to-outside-global translation.

However, replies sourced from `3.3.3.7` can still undergo outside source NAT.

This can produce an address mismatch between the destination that LH5 originally contacted and the source address of the replies.

For example:

```text
Echo Request destination:  3.3.3.7
Echo Reply source:         172.16.1.7
```

This can cause communication to fail, particularly for connection-oriented protocols such as TCP.

Applications should use the outside local address when communicating with a host represented by outside source NAT.

---

## Outside Source NAT and the Order of Operations

This is one of the most important aspects of outside source NAT for the CCIE EI exam.

Recall the traditional IOS NAT processing order:

| Direction | Simplified order |
|---|---|
| Inside to outside | Routing, then NAT |
| Outside to inside | NAT, then routing |

With ordinary inside source NAT, the destination address of an inside-to-outside packet does not change.

However, outside source NAT changes that destination address.

Consider this configuration:

```text
R1(config)# ip nat outside source static 3.3.3.7 172.16.1.7
```

An inside host sends:

```text
Source:       10.1.1.5
Destination:  172.16.1.7
```

Before the destination is translated to `3.3.3.7`, R1 must determine the forwarding path using `172.16.1.7`.

```text
Packet enters NAT inside interface
                 |
                 v
       Routing lookup for
          172.16.1.7
                 |
                 v
      Select outside interface
                 |
                 v
       Translate destination
    172.16.1.7 -> 3.3.3.7
                 |
                 v
         Forward outside
```

Therefore, a route to the outside global address alone is not necessarily sufficient.

The NAT router must have an appropriate forwarding path for the **outside local address** before translation.

### Example of a Routing Failure

Suppose Local_Router has the following route:

```text
S 3.3.3.0/24 [1/0] via 7.7.7.1
```

However, there is no default route and no route to `172.16.1.7`.

An inside host sends traffic to `172.16.1.7`.

The routing lookup fails before destination NAT can provide the intended outside forwarding path.

Even though Local_Router knows how to reach the actual outside network (`3.3.3.0/24`), it cannot successfully process the packet using the required NAT forwarding sequence.

### Incorrect Egress Interface

A route to the outside local address must also point toward the correct NAT domain.

For example:

```text
S 172.16.1.0/24 [1/0] via 10.1.1.254
```

If `10.1.1.254` is reachable through a NAT inside interface, the routing decision selects an inside forwarding path instead of an outside forwarding path.

The packet may therefore fail to undergo the intended inside-to-outside NAT processing.

**The requirement is not merely a matching route. The route must provide the appropriate outside forwarding path.**

For a detailed explanation of packet processing, see [NAT Packet Processing and Order of Operations](../order-of-operations.md).

---

## The `add-route` Keyword

Cisco IOS XE supports the `add-route` keyword with outside source NAT.

The syntax is:

```text
Router(config)# ip nat outside source static <outside-global> <outside-local> add-route
```

For example:

```text
Local_Router(config)# ip nat outside source static 3.3.3.7 172.16.1.7 add-route
```

The `add-route` keyword instructs NAT to automatically install a route for the **outside local address**, using the **outside global address** as the next hop.

The generated route is logically equivalent to:

```text
ip route 172.16.1.7 255.255.255.255 3.3.3.7
```

Note that the next hop is RH7's actual address (`3.3.3.7`), not necessarily the address of the directly connected ISP router.

Recursive routing resolves the outside global address to the appropriate forwarding path.

### Example Configuration

Configure reachability to the actual outside network:

```text
Local_Router(config)# ip route 3.3.3.0 255.255.255.0 7.7.7.1
```

Then configure the NAT mapping:

```text
Local_Router(config)# ip nat outside source static 3.3.3.7 172.16.1.7 add-route
```

For an isolated test of `add-route`, remove any manually configured route to `172.16.1.7` from the previous example.

### Verify the Automatically Installed Route

```text
Local_Router# show ip route 172.16.1.7
```

Representative output:

```text
Routing entry for 172.16.1.7/32
  Known via "static", distance 1, metric 0
  Routing Descriptor Blocks:
  * 3.3.3.7
      Route metric is 0, traffic share count is 1
```

The routing table contains the equivalent of:

```text
S 172.16.1.7/32 [1/0] via 3.3.3.7
S 3.3.3.0/24 [1/0] via 7.7.7.1
C 7.7.7.0/24 is directly connected, Ethernet0/1
```

### Recursive Resolution

The outside local address is resolved as follows:

```text
172.16.1.7
     |
     | Static route installed by NAT
     v
3.3.3.7
     |
     | Route to 3.3.3.0/24
     v
7.7.7.1
     |
     | Connected network
     v
Ethernet0/1 (NAT outside)
```

This provides the appropriate outside forwarding path for packets destined for `172.16.1.7`.

The NAT mapping subsequently translates the packet's destination to `3.3.3.7`.

The route installed by NAT and the NAT translation itself perform different functions:

- The route selects the forwarding path.
- NAT modifies the destination IP address.

### Does `add-route` Advertise the Outside Local Address?

No.

The `add-route` keyword installs a local routing table entry on the NAT router. It does not automatically advertise that route to neighboring routers through OSPF, EIGRP, BGP, or other routing protocols.

If other inside routers need reachability to the outside local address, they must learn an appropriate route through normal routing mechanisms.

### Does `add-route` Configure Return Routing?

No.

For example:

```text
ip nat outside source static 3.3.3.7 172.16.1.7 add-route
```

This creates a route associated with `172.16.1.7` on Local_Router.

It does not configure ISP1's route to `10.1.1.0/24`.

The outside network must still have a return path to the inside hosts, unless additional inside source NAT is configured.

### Is `add-route` Mandatory?

Not necessarily.

If an existing route to the outside local address already provides the correct forwarding path, an automatically generated route may not be necessary.

For example, a default route pointing toward the outside interface may be sufficient in a simple single-egress topology.

However, explicitly configuring the outside-local route or using `add-route` is more predictable, particularly in networks with:

- Multiple outside interfaces.
- Multiple ISP connections.
- Overlapping address spaces.
- More-specific routes pointing toward inside interfaces.

Cisco also documents potential packet loss associated with outside-local route installation timing and recommends configuring an explicit route or using `add-route`.

### Manual Route vs. `add-route`

| Method | Example |
|---|---|
| Manual route via the adjacent ISP | `ip route 172.16.1.7 255.255.255.255 7.7.7.1` |
| Manual route via the outside global address | `ip route 172.16.1.7 255.255.255.255 3.3.3.7` |
| Automatically generated route | `ip nat outside source static 3.3.3.7 172.16.1.7 add-route` |

All three can provide the required forwarding path when the relevant next hop is reachable.

The automatically generated route is associated with the NAT mapping. Verify its installation and removal using `show ip route`.

---

## Outside Local Addresses on the Inside Subnet

The previous examples used an outside local address (`172.16.1.7`) that belongs to a different subnet from the inside hosts.

Consider an alternative mapping:

```text
Local_Router(config)# ip nat outside source static 3.3.3.7 10.1.1.7 add-route
```

Now RH7 is represented to inside hosts as `10.1.1.7`.

```text
LH5 actual address:       10.1.1.5/24
LH6 actual address:       10.1.1.6/24
RH7 outside local:        10.1.1.7
RH7 outside global:       3.3.3.7
```

LH5 considers `10.1.1.7` directly connected because it belongs to LH5's own subnet.

When LH5 attempts to communicate with `10.1.1.7`, it does not initially send the packet to its default gateway.

Instead, it sends an ARP Request:

```text
Who has 10.1.1.7?
```

However, RH7 is not physically connected to the inside LAN.

### NAT Alias and ARP Behavior

Static outside source NAT creates an IP alias associated with the outside local address **if it is on-link**.

This can allow the NAT router to answer ARP Requests for the translated address.

```text
LH5                            Local_Router
 |                                   |
 | ARP Request: Who has 10.1.1.7?    |
 +---------------------------------->|
 |                                   |
 | ARP Reply with Local_Router MAC   |
 |<----------------------------------+
 |                                   |
 | IP destination: 10.1.1.7          |
 | Ethernet destination: Router MAC  |
 +---------------------------------->|
```

Local_Router can then forward the packet toward RH7 after applying the outside source NAT mapping in reverse.

Verify the alias and ARP behavior rather than assuming it is identical across all IOS XE platforms.

```text
Local_Router# show ip aliases
Local_Router# show ip route 10.1.1.7
```

On LH5:

```text
Local_Host_5# show ip arp 10.1.1.7
```

The `no-alias` keyword can affect NAT-generated alias behavior and is covered in [Static NAT Options and ARP Behavior](options.md).

### Why `add-route` Is Especially Important Here

Without a more-specific route, Local_Router already has this connected route:

```text
C 10.1.1.0/24 is directly connected, Ethernet0/0
```

A packet destined for `10.1.1.7` matches that connected route.

However, Ethernet0/0 is a NAT inside interface.

The intended destination is an outside host, so the routing decision must instead select the NAT outside forwarding path.

The `add-route` keyword can install:

```text
S 10.1.1.7/32 [1/0] via 3.3.3.7
```

The `/32` static route is more specific than the connected `/24` route and therefore takes precedence in the longest prefix match.

```text
C 10.1.1.0/24    via Ethernet0/0 (inside)
S 10.1.1.7/32    via 3.3.3.7   (outside)
```

This is an important example of why having a default route is not always sufficient for outside source NAT.

The connected `/24` would take precedence over the default route, whereas the NAT-generated `/32` takes precedence over the connected `/24`.

### Two Separate Requirements

When the outside local address belongs to the inside subnet:

1. The inside host must deliver the packet to the NAT router, potentially through a NAT-generated ARP alias.
2. The NAT router must select the appropriate outside forwarding path, potentially through a route installed by `add-route`.

**ARP resolution and routing are separate requirements.**

A static route on the NAT router does not automatically make an inside host send traffic to the router.

Likewise, a successful ARP response does not guarantee that the NAT router will select the correct outside forwarding path.

A separate routed outside-local prefix, such as `172.16.1.0/24`, avoids the need for inside hosts to resolve individual outside-local addresses through ARP.

---

## Multiple Static Outside Source Mappings

Multiple outside hosts can be represented by different outside local addresses.

For example:

```text
Local_Router(config)# ip nat outside source static 3.3.3.7 172.16.1.7 add-route
Local_Router(config)# ip nat outside source static 3.3.3.8 172.16.1.8 add-route
```

The resulting mappings are:

| Outside host | Outside global | Outside local |
|---|---|---|
| RH7 | `3.3.3.7` | `172.16.1.7` |
| RH8 | `3.3.3.8` | `172.16.1.8` |

Representative translation table:

```text
Local_Router# show ip nat translations
Pro  Inside global      Inside local       Outside local      Outside global
---  ---                ---                172.16.1.7         3.3.3.7
---  ---                ---                172.16.1.8         3.3.3.8
```

Representative routes:

```text
S 172.16.1.7/32 [1/0] via 3.3.3.7
S 172.16.1.8/32 [1/0] via 3.3.3.8
```

LH5 can use:

```text
Local_Host_5# ping 172.16.1.7
Local_Host_5# ping 172.16.1.8
```

The two outside hosts remain distinguishable because each has a unique outside local address.

---

## Static Outside Source Network Mapping

On supported IOS XE platforms, an entire outside network can be mapped to another network.

The syntax follows the same global-first argument order:

```text
ip nat outside source static network <global-network> <local-network> <mask>
```

For example, on an implementation accepting CIDR-style mask notation:

```text
Local_Router(config)# ip nat outside source static network 3.3.3.0 172.16.1.0 /24 add-route
```

Check the target IOS XE image for its supported mask format and optional keywords.

The resulting relationship preserves host bits:

| Outside global | Outside local |
|---|---|
| `3.3.3.7` | `172.16.1.7` |
| `3.3.3.8` | `172.16.1.8` |
| `3.3.3.50` | `172.16.1.50` |
| `3.3.3.100` | `172.16.1.100` |

This is the outside source equivalent of the feature covered in [Static Network-to-Network NAT](network-mapping.md).

As with individual outside source mappings, appropriate routing toward the outside local network is required.

If the entire outside local prefix needs to be forwarded toward ISP1, a manually configured route could be:

```text
Local_Router(config)# ip route 172.16.1.0 255.255.255.0 7.7.7.1
```

Verify how the specific IOS XE implementation handles network-wide mappings, per-host translation entries, ARP aliases, and route installation before relying on automatic behavior.


### `add-route` with Static Network Mappings

Unlike individual static outside source mappings, static network mappings do not immediately install routes when configured with `add-route`.

For example:

```text
R1(config)# ip nat outside source static network 3.3.3.0 172.16.1.0 /24 add-route
```

In my lab:

- The static network translation is installed immediately.
- No route to `172.16.1.0/24` is automatically installed.
- When RH7 initiates communication, an individual host translation is created.
- NAT then installs a `/32` route to `172.16.1.7` via `3.3.3.7`.

This creates a potential chicken-and-egg problem for inside-initiated traffic: a route is needed before NAT can occur, but NAT creates the route only when the individual host translation is created.

To allow inside hosts to initiate communication without depending on prior outside-initiated traffic, manually configure a route for the entire outside local prefix:

```text
R1(config)# ip route 172.16.1.0 255.255.255.0 7.7.7.1
```

**The static network mapping is persistent, but the individual host translations and associated routes are created on demand.**

---

## Static Outside Source PAT

**Static outside source PAT** extends the outside source NAT concept to include TCP or UDP port numbers.

It maps an outside host's actual IP address and port to a different IP address and port used by inside devices.

The syntax is:

```text
Router(config)# ip nat outside source static {tcp | udp} <global-ip> <global-port> <local-ip> <local-port>
```

As with address-only outside source NAT:

**Outside global comes first, followed by outside local.**

For example:

```text
Local_Router(config)# ip nat outside source static tcp 3.3.3.7 443 172.16.1.7 8443 add-route
```

This creates the following mapping:

```text
Outside global                      Outside local
  3.3.3.7:443     <-------------->   172.16.1.7:8443
```

Assume RH7 runs an HTTPS server on TCP port `443`.

Inside clients connect to:

```text
172.16.1.7:8443
```

Local_Router translates those connections to:

```text
3.3.3.7:443
```

Unlike the previous static inside source PAT examples, the service being represented is located on the **outside network**.

Remove any overlapping address-only outside source mappings before testing this example in isolation.

### Verify the Static PAT Mapping

Representative output:

```text
Local_Router# show ip nat translations
Pro  Inside global      Inside local       Outside local       Outside global
tcp  ---                ---                172.16.1.7:8443     3.3.3.7:443
```

The static entry specifies:

- The transport protocol.
- The outside local address and port.
- The outside global address and port.

No inside host is specified in the static mapping.

### Inside-to-Outside Packet Flow

LH5 initiates an HTTPS connection to RH7's translated endpoint.

Before NAT:

```text
Source:       10.1.1.5:51000
Destination:  172.16.1.7:8443
Protocol:     TCP
```

Local_Router first selects the outside forwarding path using the outside local destination address.

It then translates both the destination IP address and destination TCP port.

After NAT:

```text
Source:       10.1.1.5:51000
Destination:  3.3.3.7:443
Protocol:     TCP
```

| Field | Before NAT | After NAT |
|---|---|---|
| Source IP | `10.1.1.5` | `10.1.1.5` |
| Source TCP port | `51000` | `51000` |
| Destination IP | `172.16.1.7` | `3.3.3.7` |
| Destination TCP port | `8443` | `443` |

The inside host's source address and port remain unchanged.

### Outside-to-Inside Return Traffic

RH7 responds:

```text
Source:       3.3.3.7:443
Destination:  10.1.1.5:51000
Protocol:     TCP
```

Local_Router translates the source IP address and TCP port.

After NAT:

```text
Source:       172.16.1.7:8443
Destination:  10.1.1.5:51000
Protocol:     TCP
```

| Field | Before NAT | After NAT |
|---|---|---|
| Source IP | `3.3.3.7` | `172.16.1.7` |
| Source TCP port | `443` | `8443` |
| Destination IP | `10.1.1.5` | `10.1.1.5` |
| Destination TCP port | `51000` | `51000` |

LH5 receives a reply from the endpoint it originally contacted.

### Complete TCP Exchange

```text
LH5                      Local_Router                    RH7
10.1.1.5                                                3.3.3.7
   |                           |                           |
   | Src 10.1.1.5:51000        |                           |
   | Dst 172.16.1.7:8443       |                           |
   +-------------------------->|                           |
                               | Destination NAT           |
                               | 172.16.1.7:8443           |
                               |    -> 3.3.3.7:443         |
                               |                           |
                               | Src 10.1.1.5:51000        |
                               | Dst 3.3.3.7:443           |
                               +-------------------------->|
                               |                           |
                               | Src 3.3.3.7:443           |
                               | Dst 10.1.1.5:51000        |
                               |<--------------------------+
                               |                           |
                               | Source NAT                |
                               | 3.3.3.7:443               |
                               |    -> 172.16.1.7:8443     |
                               |                           |
   | Src 172.16.1.7:8443       |                           |
   | Dst 10.1.1.5:51000        |                           |
   |<--------------------------+                           |
```

---

## Outside Source PAT Matching

Consider:

```text
ip nat outside source static tcp 3.3.3.7 443 172.16.1.7 8443
```

The matching criteria depend on the direction in which the packet traverses the NAT router.

| Direction | Fields matched |
|---|---|
| Inside to outside | TCP destination `172.16.1.7:8443` |
| Outside to inside | TCP source `3.3.3.7:443` |

For example:

| Packet | Matches? |
|---|---|
| Inside-to-outside TCP destined for `172.16.1.7:8443` | Yes |
| Inside-to-outside TCP destined for `172.16.1.7:443` | No |
| Inside-to-outside UDP destined for `172.16.1.7:8443` | No |
| Outside-to-inside TCP sourced from `3.3.3.7:443` | Yes |
| Outside-to-inside TCP sourced from `3.3.3.7:51000` | No |
| Outside-to-inside ICMP sourced from `3.3.3.7` | No |

These examples consider the specified static PAT mapping in isolation.

Other NAT rules or existing translations may handle packets that do not match it.

**The protocol and port of the outside host must match the mapping.**

### Port Preservation

Outside source PAT does not require the global and local port numbers to differ.

For example:

```text
ip nat outside source static tcp 3.3.3.7 443 172.16.1.7 443 add-route
```

The IP address is translated, but TCP port `443` is preserved.

This is still static PAT because the mapping is specific to a protocol and port.

### TCP and UDP Mappings

TCP and UDP must be configured separately.

For example, to represent an outside DNS server using the address `172.16.1.7`:

```text
ip nat outside source static tcp 3.3.3.7 53 172.16.1.7 53 add-route
ip nat outside source static udp 3.3.3.7 53 172.16.1.7 53 add-route
```

The two rules represent different transport-layer endpoints.

Both use the same outside local address and numeric port, but they match different protocols.

---

## Multiple Outside Servers Sharing One Outside Local Address

Static outside source PAT allows different outside hosts to be represented by the same outside local IP address using distinguishable protocol/port combinations.

Suppose both RH7 and RH8 run HTTPS servers on TCP port `443`.

We want inside hosts to use:

```text
172.16.1.100:8443 -> RH7
172.16.1.100:9443 -> RH8
```

Configure:

```text
Local_Router(config)# ip nat outside source static tcp 3.3.3.7 443 172.16.1.100 8443
Local_Router(config)# ip nat outside source static tcp 3.3.3.8 443 172.16.1.100 9443
```

Provide an explicit route for the shared outside local address:

```text
Local_Router(config)# ip route 172.16.1.100 255.255.255.255 7.7.7.1
```

The mappings are:

| Outside local | Outside global |
|---|---|
| `172.16.1.100:8443/TCP` | `3.3.3.7:443/TCP` |
| `172.16.1.100:9443/TCP` | `3.3.3.8:443/TCP` |

Both outside servers are reachable through the same ISP next hop in this topology.

### Routing Consideration

IP routing selects a forwarding path based on the destination IP address, not the TCP/UDP destination port.

Therefore, both connections initially undergo a routing lookup for `172.16.1.100`.

If the two actual outside servers require different outside interfaces or next hops, an ordinary route to their shared outside local address may not provide the correct forwarding path for both mappings.

Such designs may require additional forwarding policies or separate translated addresses.

Do not assume that `add-route` can distinguish routes based on TCP/UDP port numbers.

In this example, a manually configured route toward the common ISP next hop avoids ambiguity between the two outside global addresses.

---

## Simultaneous Inside Source and Outside Source NAT

Inside source and outside source NAT can be configured together.

This allows the NAT router to translate both the source and destination addresses of a packet.

Consider the original lab topology:

```text
LH5 actual address:       10.1.1.5
RH7 actual address:        3.3.3.7
```

Configure the following mappings:

```text
Local_Router(config)# ip nat inside source static 10.1.1.5 7.7.7.50
Local_Router(config)# ip nat outside source static 3.3.3.7 172.16.1.7 add-route
```

The resulting address representations are:

| NAT address type | Address |
|---|---|
| Inside local | `10.1.1.5` |
| Inside global | `7.7.7.50` |
| Outside local | `172.16.1.7` |
| Outside global | `3.3.3.7` |

LH5 represents itself using `10.1.1.5` and communicates with RH7 using `172.16.1.7`.

RH7 represents itself using `3.3.3.7` and communicates with LH5 using `7.7.7.50`.

### Inside-to-Outside Traffic

LH5 initiates a ping to RH7's outside local address:

```text
Local_Host_5# ping 172.16.1.7
```

Before NAT:

```text
Source:       10.1.1.5
Destination:  172.16.1.7
```

Local_Router applies both translations:

```text
Inside source NAT:
10.1.1.5 -> 7.7.7.50

Outside source NAT (reverse direction):
172.16.1.7 -> 3.3.3.7
```

After NAT:

```text
Source:       7.7.7.50
Destination:  3.3.3.7
```

Both the source and destination addresses have changed.

### Outside-to-Inside Return Traffic

RH7 sends a reply:

```text
Source:       3.3.3.7
Destination:  7.7.7.50
```

Local_Router applies both translations in the reverse direction:

```text
Outside source NAT:
3.3.3.7 -> 172.16.1.7

Inside source NAT (reverse direction):
7.7.7.50 -> 10.1.1.5
```

After NAT:

```text
Source:       172.16.1.7
Destination:  10.1.1.5
```

### Translation Summary

| Packet | Source | Destination |
|---|---|---|
| Inside-to-outside before NAT | Inside local | Outside local |
| Inside-to-outside after NAT | Inside global | Outside global |
| Outside-to-inside before NAT | Outside global | Inside global |
| Outside-to-inside after NAT | Outside local | Inside local |

Using the actual lab addresses:

| Packet | Source | Destination |
|---|---|---|
| Echo Request before NAT | `10.1.1.5` | `172.16.1.7` |
| Echo Request after NAT | `7.7.7.50` | `3.3.3.7` |
| Echo Reply before NAT | `3.3.3.7` | `7.7.7.50` |
| Echo Reply after NAT | `172.16.1.7` | `10.1.1.5` |

### Conceptual Diagram

```text
        Inside view                          Outside view

 LH5                 RH7              LH5                 RH7
10.1.1.5          172.16.1.7        7.7.7.50           3.3.3.7
   |                  |                 |                  |
   +------------------+                 +------------------+
            |                                    |
            |          NAT boundary              |
            +--------[ Local_Router ]------------+

 Inside local:    10.1.1.5
 Inside global:   7.7.7.50
 Outside local:   172.16.1.7
 Outside global:  3.3.3.7
```

This illustrates why Cisco NAT terminology includes four different address types.

All four addresses can be relevant to a single translated connection.

### Translation Table

Before traffic is generated, the static mappings can appear as separate simple entries:

```text
Pro  Inside global      Inside local       Outside local      Outside global
---  7.7.7.50           10.1.1.5           ---                ---
---  ---                ---                172.16.1.7         3.3.3.7
```

After communication occurs, an extended entry containing all four addresses can appear:

```text
Pro  Inside global      Inside local       Outside local      Outside global
icmp 7.7.7.50:1         10.1.1.5:1         172.16.1.7:1       3.3.3.7:1
---  7.7.7.50           10.1.1.5           ---                ---
---  ---                ---                172.16.1.7         3.3.3.7
```

The ICMP identifier and precise table presentation are illustrative.

The two static mappings are persistent. Any dynamically created extended flow entries can be subject to timeouts.

### Routing Changes

With simultaneous inside and outside source NAT, ISP1 no longer needs a route to LH5's actual address.

RH7 sees LH5 as `7.7.7.50`, not `10.1.1.5`.

For this specific translated communication, we can remove the route configured earlier:

```text
ISP1(config)# no ip route 10.1.1.0 255.255.255.0 7.7.7.2
```

ISP1 must still be able to deliver packets addressed to `7.7.7.50` to Local_Router.

Because `7.7.7.50` belongs to the directly connected outside subnet in this example, NAT alias and ARP behavior are relevant.

The routing requirements associated with outside source NAT also remain in effect. Local_Router still requires the appropriate outside forwarding path for `172.16.1.7`.

---

## Outside Source NAT and Overlapping Networks

One of the most important use cases for outside source NAT is communication between networks with overlapping address spaces.

Suppose two organizations both use `10.1.1.0/24`.

```text
Organization A                         Organization B
10.1.1.0/24                           10.1.1.0/24

Host A                                Host B
10.1.1.5                              10.1.1.7
    |                                    |
    +----------[ NAT boundary ]----------+
```

Host A cannot ordinarily reach Host B using `10.1.1.7`.

From Host A's perspective, `10.1.1.7` belongs to its own directly connected network.

It sends an ARP Request locally instead of forwarding traffic toward the remote organization.

Outside source NAT can provide an alternative representation of Host B:

```text
Outside global: 10.1.1.7
Outside local:  172.16.1.7
```

Host A can now attempt to communicate with Host B using `172.16.1.7`.

However, outside source NAT alone does not necessarily solve the entire overlapping-address problem.

The remote network may also consider Host A's address (`10.1.1.5`) locally connected.

Therefore, inside source NAT may be required to represent Host A using a different address on the outside network.

For example:

```text
Inside local:    10.1.1.5
Inside global:   172.16.2.5
Outside global:  10.1.1.7
Outside local:   172.16.1.7
```

The resulting communication would use:

| Location | Source | Destination |
|---|---|---|
| Organization A | `10.1.1.5` | `172.16.1.7` |
| Organization B | `172.16.2.5` | `10.1.1.7` |

Both organizations communicate with a non-overlapping representation of the remote host.

> **Important:** This is a conceptual address-translation example, not a complete overlapping-network configuration. When the actual inside and outside prefixes overlap, the NAT router itself must also have an unambiguous forwarding path. VRFs, policy-based routing, or other design mechanisms may be necessary.

More advanced overlapping-address scenarios are covered in:

- [Twice NAT and Overlapping Networks](../policy-based/twice-nat.md)
- [Overlapping Addresses Across VRFs](../vrf/overlapping-addresses.md)

---

## Outside Source NAT and ACLs

As with inside source NAT, the placement of an ACL determines whether it evaluates the original or translated addresses.

Consider:

```text
ip nat outside source static 3.3.3.7 172.16.1.7
```

### Outside-to-Inside Traffic

An outside host sends:

```text
Source:       3.3.3.7
Destination:  10.1.1.5
```

After NAT:

```text
Source:       172.16.1.7
Destination:  10.1.1.5
```

| ACL location | Source address evaluated |
|---|---|
| Outside interface inbound | `3.3.3.7` (outside global) |
| Inside interface outbound | `172.16.1.7` (outside local) |

For example, an outside inbound ACL could match the packet with:

```text
permit ip host 3.3.3.7 host 10.1.1.5
```

An inside outbound ACL could match the translated packet with:

```text
permit ip host 172.16.1.7 host 10.1.1.5
```

### Inside-to-Outside Traffic

An inside host sends:

```text
Source:       10.1.1.5
Destination:  172.16.1.7
```

After NAT:

```text
Source:       10.1.1.5
Destination:  3.3.3.7
```

| ACL location | Destination address evaluated |
|---|---|
| Inside interface inbound | `172.16.1.7` (outside local) |
| Outside interface outbound | `3.3.3.7` (outside global) |

These are the traditional IOS interface ACL and NAT processing relationships.

Additional forwarding features may introduce further considerations.

See [NAT Packet Processing and Order of Operations](../order-of-operations.md).

---

## Verification and Troubleshooting

### Verify the Configured Mapping

```text
Local_Router# show running-config | include ip nat
```

For example:

```text
ip nat inside
ip nat outside
ip nat outside source static 3.3.3.7 172.16.1.7 add-route
```

Confirm the argument order:

```text
Outside global: 3.3.3.7
Outside local:  172.16.1.7
```

Reversing these addresses creates a mapping with the wrong matching criteria.

### Verify the Translation Table

```text
Local_Router# show ip nat translations
```

Representative output:

```text
Pro  Inside global      Inside local       Outside local      Outside global
---  ---                ---                172.16.1.7         3.3.3.7
```

For additional information:

```text
Local_Router# show ip nat translations verbose
```

Check the configured static entry and any dynamically generated extended entries.

### Verify Routing to Both Outside Addresses

```text
Local_Router# show ip route 172.16.1.7
Local_Router# show ip route 3.3.3.7
```

The first command checks the route used for the pre-translation destination.

The second checks reachability to the outside global address.

Both are particularly relevant when `add-route` creates a recursively resolved static route.

Also inspect the CEF forwarding entries:

```text
Local_Router# show ip cef 172.16.1.7 detail
Local_Router# show ip cef 3.3.3.7 detail
```

Confirm that the outside local address resolves toward the intended outside forwarding path.

### Verify Return Routing

If only outside source NAT is configured, the inside host's address is not translated.

On ISP1:

```text
ISP1# show ip route 10.1.1.5
```

Verify that the outside network can deliver traffic to the inside host.

If inside source NAT is also configured, instead verify reachability to the relevant inside global address.

### Verify ARP When Necessary

If the outside local address belongs to the inside subnet:

```text
Local_Router# show ip aliases
```

On the inside host:

```text
Local_Host_5# show ip arp
```

Confirm that the inside host can resolve the outside local address to the appropriate NAT router MAC address.

Remember that the NAT router must receive the packet before it can translate it.

### Test Inside-Initiated Traffic

For an address-only mapping:

```text
Local_Host_5# ping 172.16.1.7
```

For static outside source PAT:

```text
Local_Host_5# telnet 172.16.1.7 8443
```

Alternatively, from a Linux host:

```bash
curl -k https://172.16.1.7:8443/
```

The outside server must actually be listening on the corresponding outside global port.

An ICMP ping does not test a TCP-specific static PAT mapping.

### Test Outside-Initiated Traffic

For address-only outside source NAT without inside source NAT:

```text
Remote_Host_7# ping 10.1.1.5
```

Verify that LH5 receives the packet with source IP address `172.16.1.7`.

If simultaneous inside source NAT is configured, use the inside global address instead:

```text
Remote_Host_7# ping 7.7.7.50
```

### Debug NAT

In a controlled lab:

```text
Local_Router# debug ip nat
```

Generate traffic and observe which addresses are translated.

For outside source NAT, expect:

- Source translation for outside-to-inside packets.
- Destination translation for inside-to-outside packets.

Disable debugging when finished:

```text
Local_Router# undebug all
```

Debug visibility can vary depending on the platform and forwarding implementation.

### Packet Captures

A packet capture on each side of Local_Router provides a direct way to verify outside source NAT.

For an inside-initiated ping:

| Capture location | Source | Destination |
|---|---|---|
| Inside link, Echo Request | `10.1.1.5` | `172.16.1.7` |
| Outside link, Echo Request | `10.1.1.5` | `3.3.3.7` |
| Outside link, Echo Reply | `3.3.3.7` | `10.1.1.5` |
| Inside link, Echo Reply | `172.16.1.7` | `10.1.1.5` |

This allows you to distinguish translation problems from routing, ARP, and application problems.

---

## Common Troubleshooting Scenarios

| Symptom | Possible cause |
|---|---|
| Inside host cannot reach outside local address | Missing route or incorrect outside forwarding path |
| NAT entry exists but no traffic is translated | Packet does not traverse the correct NAT interfaces or match the rule |
| Outside-initiated request reaches inside host, but reply fails | Missing or incorrect route to outside local address |
| Inside-initiated request reaches outside host, but reply fails | Outside network lacks a route to inside local address |
| Inside host ARPs for outside local address without success | Outside local address is on-link, but the required NAT alias or ARP handling is missing |
| Packet is forwarded toward an inside interface instead of outside | A connected or more-specific route overrides the intended outside route |
| TCP connection uses unexpected ports | Incorrect outside global/local port order |
| TCP works but UDP does not | Only a TCP static PAT mapping was configured |
| Ping fails while static PAT works | ICMP does not match a TCP/UDP-specific mapping |
| Shared outside local address reaches the wrong network | Multiple PAT mappings require incompatible pre-NAT routing paths |
| Inside host receives replies from an unexpected IP | Inside host contacted the outside global address instead of outside local |

---

## Command Summary

| Task | Command |
|---|---|
| Configure static outside source NAT | `ip nat outside source static <global-ip> <local-ip>` |
| Automatically install outside-local route | `ip nat outside source static <global-ip> <local-ip> add-route` |
| Configure static outside source TCP PAT | `ip nat outside source static tcp <global-ip> <global-port> <local-ip> <local-port>` |
| Configure static outside source UDP PAT | `ip nat outside source static udp <global-ip> <global-port> <local-ip> <local-port>` |
| Configure static outside source network NAT | `ip nat outside source static network <global-network> <local-network> <mask>` |
| Verify NAT translations | `show ip nat translations` |
| Verify detailed translations | `show ip nat translations verbose` |
| Verify outside-local routing | `show ip route <outside-local>` |
| Verify outside-global routing | `show ip route <outside-global>` |
| Verify NAT-generated aliases | `show ip aliases` |
| Debug NAT | `debug ip nat` |

---

## Key Takeaways

- **Static outside source NAT** maps an outside global address to an outside local address.
- The outside global address is the outside host's actual address.
- The outside local address is the address representing that host to inside devices.
- The configuration command specifies **outside global first, followed by outside local**.
- Outside-to-inside packets undergo **source translation**.
- Inside-to-outside packets undergo **destination translation**.
- Static outside source NAT supports both inside-initiated and outside-initiated communication.
- Outside source NAT does not translate the inside host's address unless an additional inside source NAT mapping is configured.
- Because routing precedes inside-to-outside NAT, the NAT router needs an appropriate route to the **outside local address**.
- The `add-route` keyword automatically installs a route to the outside local address, using the outside global address as the next hop for an ordinary host mapping.
- `add-route` does not automatically advertise routes or configure return routing on other routers.
- If the outside local address belongs to the inside subnet, ARP alias behavior and longest prefix matching become particularly important.
- Static outside source PAT identifies specific TCP/UDP endpoints and can translate both addresses and ports.
- Multiple outside services can share one outside local IP address using distinguishable protocol/port combinations.
- Simultaneous inside source and outside source NAT can translate both source and destination addresses in the same packet.
- Outside source NAT is particularly useful in overlapping-address scenarios.

---

## References

- Cisco, [Configure IP NAT Outside Source Static Command](https://www.cisco.com/c/en/us/support/docs/ip/ip-addressing-services/218383-use-ip-nat-command-for-configuration.html)
- Cisco, [IP Addressing Configuration Guide, Cisco IOS XE 17.x: Configuring NAT for IP Address Conservation](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-addressing/b-ip-addressing/m_iadnat-addr-consv-xe.html)
- Cisco, [IP Addressing Services Command Reference: ip nat outside source](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/ipaddr/command/ipaddr-cr-book/ipaddr-i3.html)
- Cisco, [Cisco IOS XE 17.8.x: IP Addressing Services Commands](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9600/software/release/17-8/command_reference/b_178_9600_cr/ip_addressing_services_commands.html)
- Cisco, [Understand the NAT Order of Operation](https://www.cisco.com/c/en/us/support/docs/ip/network-address-translation-nat/6209-5.html)
- Cisco, [Configure IP NAT Outside Source List Command](https://www.cisco.com/c/en/us/support/docs/ip/network-address-translation-nat/13770-1.html)
- Jedadiah Casey, [Understanding and Configuring Network Address Translation (NAT) for Cisco CCNA, CCNP and CCIE](https://wax-trax.medium.com/understanding-and-configuring-network-address-translation-nat-for-cisco-ccna-ccnp-and-ccie-7fef92fa654c)
