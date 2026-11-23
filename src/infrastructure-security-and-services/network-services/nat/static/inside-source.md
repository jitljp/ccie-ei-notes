# Static Inside Source NAT

**Static inside source NAT** creates a permanent, one-to-one mapping between an **inside local** IPv4 address and an **inside global** IPv4 address.

It is useful when an inside host needs to appear under a predictable address on an outside network, especially when outside hosts must be able to initiate traffic to that host. The mapping exists independently of any particular TCP, UDP, or ICMP session.

The name **inside source** identifies which host's address is being mapped. It does not mean that only the source field can change:

- **Inside to outside:** The inside host's **source** address changes from inside local to inside global.
- **Outside to inside:** The packet's **destination** address changes from inside global to inside local.

This page demonstrates ordinary address-only static NAT. Static PAT, network-to-network translation, and other options are covered separately.

The lab topology and addressing are adapted from **Case 1** of [Jedadiah Casey's NAT article](https://wax-trax.medium.com/understanding-and-configuring-network-address-translation-nat-for-cisco-ccna-ccnp-and-ccie-7fef92fa654c). The same topology can be reused in subsequent NAT lessons.

---

## Syntax and Address Order

The basic command is:

```text
Router(config)# ip nat inside source static <inside-local> <inside-global>
```

For this lab:

```text
Local_Router(config)# ip nat inside source static 10.1.1.5 7.7.7.50
```

| Argument | Address | Meaning |
|---|---|---|
| First | `10.1.1.5` | **Inside local**, the address configured on the inside host |
| Second | `7.7.7.50` | **Inside global**, the address that represents the host to the outside network |

The address order is important. In particular, `ip nat outside source static` uses a different argument order: **outside global first, outside local second**.

### Characteristics of Static Inside Source NAT

- The mapping is configured manually and does not require an ACL or address pool.
- The same inside local address is represented by the same inside global address for the ordinary address-only mapping.
- It is not restricted to one TCP or UDP port.
- Both inside-initiated and outside-initiated traffic can use the mapping.
- The configured mapping does not age out like a dynamically allocated NAT address.
- NAT does **not** automatically allow traffic through an ACL or firewall.

**Static NAT is not PAT.** With ordinary address-only static NAT, the router changes the IP address without deliberately remapping transport-layer ports to let multiple hosts share that global address.

---

## Lab Topology

We will use **Local_Router** as the NAT device, **ISP1** as the upstream router, and two hosts on each side. Only **Local_Host_5 (LH5)** needs NAT for the first exercise.

```text
      INSIDE                         OUTSIDE
    10.1.1.0/24                   7.7.7.0/24                3.3.3.0/24

 LH5 10.1.1.5 ---+
                  |             E0/0             E0/1                 E0/1      E0/0        +--- RH7 3.3.3.7
                  +--SW-- 10.1.1.1 [ Local_Router ] 7.7.7.2 --- 7.7.7.1 [ ISP1 ] 3.3.3.1 --SW
                  |            ip nat inside    ip nat outside                              +--- RH8 3.3.3.8
 LH6 10.1.1.6 ---+

                  Static mapping on Local_Router:
                  10.1.1.5 (inside local) <--> 7.7.7.50 (inside global)
```

| Device | Interface | IPv4 address | Role |
|---|---|---|---|
| Local_Host_5 (LH5) | Host interface | `10.1.1.5/24` | Inside host to translate |
| Local_Host_6 (LH6) | Host interface | `10.1.1.6/24` | Additional inside host, initially not translated |
| Local_Router | `Ethernet0/0` | `10.1.1.1/24` | `ip nat inside` |
| Local_Router | `Ethernet0/1` | `7.7.7.2/24` | `ip nat outside` |
| ISP1 | `Ethernet0/1` | `7.7.7.1/24` | Toward Local_Router |
| ISP1 | `Ethernet0/0` | `3.3.3.1/24` | Toward remote hosts |
| Remote_Host_7 (RH7) | Host interface | `3.3.3.7/24` | Outside test host |
| Remote_Host_8 (RH8) | Host interface | `3.3.3.8/24` | Additional outside host |
| Static mapping | N/A | `10.1.1.5` ↔ `7.7.7.50` | Inside local ↔ inside global |

**Interface naming:** The article uses `Ethernet0/0` and `Ethernet0/1`, which are common in IOS lab images. The configuration below follows those names for consistency. On an IOS XE virtual router such as CSR1000v or Catalyst 8000V, substitute the actual interfaces (for example, `GigabitEthernet1` and `GigabitEthernet2`). The NAT behavior is the same.

**Addressing note:** `7.7.7.0/24` and `3.3.3.0/24` reproduce the article's lab addressing. They are **not** RFC 5737 documentation prefixes and should not be used as arbitrary public addressing in a production network.

### Routing Assumptions

For this lab:

- LH5 and LH6 use `10.1.1.1` as their default gateway.
- RH7 and RH8 use `3.3.3.1` as their default gateway.
- Local_Router has a default route toward ISP1 at `7.7.7.1`.
- ISP1 already knows `7.7.7.0/24` and `3.3.3.0/24` because they are directly connected.
- We do **not** add a route for the private inside subnet (`10.1.1.0/24`) on ISP1.

The last point is intentional. Outside devices should reach LH5 using its **inside global** address, `7.7.7.50`, not its private inside local address.

---

## Configure Static Inside Source NAT

### Step 1: Configure Local_Router's Inside Interface

```text
Local_Router(config)# interface Ethernet0/0
Local_Router(config-if)# ip address 10.1.1.1 255.255.255.0
Local_Router(config-if)# ip nat inside
Local_Router(config-if)# no shutdown
```

`ip nat inside` identifies the interface through which traffic from LH5 enters the NAT router.

### Step 2: Configure Local_Router's Outside Interface

```text
Local_Router(config)# interface Ethernet0/1
Local_Router(config-if)# ip address 7.7.7.2 255.255.255.0
Local_Router(config-if)# ip nat outside
Local_Router(config-if)# no shutdown
```

`ip nat outside` identifies the outside NAT domain. The interface address `7.7.7.2` is **not** the inside global address used for LH5 in this exercise.

### Step 3: Configure the Static Mapping

```text
Local_Router(config)# ip nat inside source static 10.1.1.5 7.7.7.50
```

The configured address mapping is:

```text
Inside local                       Inside global
  10.1.1.5       <------------->       7.7.7.50
```

LH5 continues using `10.1.1.5` on its interface. Do **not** configure `7.7.7.50` on LH5.

### Step 4: Configure Local_Router's Default Route

```text
Local_Router(config)# ip route 0.0.0.0 0.0.0.0 7.7.7.1
```

For ordinary inside-to-outside NAT, Local_Router performs the routing decision before source translation. It needs a route toward the original outside destination, such as `3.3.3.7`.

The connected `10.1.1.0/24` route already provides reachability to LH5 after destination translation in the reverse direction.

### Step 5: Configure ISP1

```text
ISP1(config)# interface Ethernet0/1
ISP1(config-if)# ip address 7.7.7.1 255.255.255.0
ISP1(config-if)# no shutdown

ISP1(config)# interface Ethernet0/0
ISP1(config-if)# ip address 3.3.3.1 255.255.255.0
ISP1(config-if)# no shutdown
```

ISP1 needs no additional route for `7.7.7.50`, because it considers the entire `7.7.7.0/24` network directly connected. It must, however, be able to resolve `7.7.7.50` to a Layer 2 address. The NAT-generated alias is discussed below.

### Step 6: Configure the End Hosts

Assign these addresses and gateways to the endpoints:

```text
LH5:  10.1.1.5/24   gateway 10.1.1.1
LH6:  10.1.1.6/24   gateway 10.1.1.1
RH7:  3.3.3.7/24   gateway 3.3.3.1
RH8:  3.3.3.8/24   gateway 3.3.3.1
```

The endpoints can be ordinary hosts or IOS routers acting as hosts. If using routers to simulate hosts, configure their default route or gateway as appropriate to whether IP routing is enabled.

---

## Verify the Static Mapping Before Sending Traffic

On Local_Router:

```text
Local_Router# show ip nat translations
Pro  Inside global      Inside local       Outside local      Outside global
---  7.7.7.50           10.1.1.5           ---                ---
```

This is a **simple static NAT entry**.

- `---` under `Pro` indicates an address-only translation rather than a mapping restricted to a particular protocol.
- `7.7.7.50` is the inside global address.
- `10.1.1.5` is the inside local address.
- No outside host is specified in the static rule, so the outside address columns are blank.

**The static mapping exists even before LH5 or RH7 sends any traffic.** This differs from a dynamically allocated NAT binding.

---

## Inside-to-Outside Packet Flow

Suppose LH5 pings RH7:

```text
Local_Host_5# ping 3.3.3.7
```

The ICMP Echo Request begins with these IP addresses:

```text
Source:      10.1.1.5
Destination: 3.3.3.7
```

Local_Router processes it as follows:

1. LH5 forwards the packet to its default gateway, `10.1.1.1`.
2. The packet enters Local_Router's NAT inside interface, `Ethernet0/0`.
3. Local_Router looks up the destination `3.3.3.7` and selects its default route toward ISP1 through `Ethernet0/1`.
4. Static NAT matches the packet's source address `10.1.1.5`.
5. The router changes the **source** address to `7.7.7.50` and forwards the packet to ISP1.
6. ISP1 forwards the packet to RH7 on its connected `3.3.3.0/24` network.

| Field | Before NAT | After NAT |
|---|---|---|
| Source IP | `10.1.1.5` | `7.7.7.50` |
| Destination IP | `3.3.3.7` | `3.3.3.7` |
| ICMP type | Echo Request | Echo Request |
| ICMP identifier | Unchanged | Unchanged |

```text
LH5                         Local_Router                  ISP1                RH7
 |                                |                         |                  |
 | Src 10.1.1.5                   |                         |                  |
 | Dst 3.3.3.7                    |                         |                  |
 +------------------------------->|                         |                  |
                                  | Source NAT              |                  |
                                  | 10.1.1.5 -> 7.7.7.50    |                  |
                                  | Src 7.7.7.50            |                  |
                                  | Dst 3.3.3.7             |                  |
                                  +------------------------>+----------------->|
```

RH7 sees the ping as coming from `7.7.7.50`. It does not need a route to `10.1.1.5`.

### Return Packet

RH7 sends the ICMP Echo Reply to `7.7.7.50`:

```text
Source:      3.3.3.7
Destination: 7.7.7.50
```

1. RH7 sends the reply to its default gateway, ISP1 (`3.3.3.1`).
2. ISP1 considers `7.7.7.50` directly connected on `7.7.7.0/24` and delivers it toward Local_Router using ARP resolution.
3. The packet enters Local_Router on `Ethernet0/1`, its NAT outside interface.
4. NAT translates the **destination** address from `7.7.7.50` to `10.1.1.5`.
5. Local_Router routes the translated packet onto the connected `10.1.1.0/24` network and delivers it to LH5.

| Field | Before NAT | After NAT |
|---|---|---|
| Source IP | `3.3.3.7` | `3.3.3.7` |
| Destination IP | `7.7.7.50` | `10.1.1.5` |
| ICMP type | Echo Reply | Echo Reply |
| ICMP identifier | Unchanged | Unchanged |

```text
LH5                         Local_Router                  ISP1                RH7
 |                                |                         |                  |
 |                                |                         |  Src 3.3.3.7     |
 |                                |                         |  Dst 7.7.7.50    |
 |                                |<------------------------+<-----------------+
 |                                | Destination NAT         |
 |                                | 7.7.7.50 -> 10.1.1.5    |
 |  Src 3.3.3.7                   |                         |
 |  Dst 10.1.1.5                  |                         |
 |<-------------------------------+                         |
```

**Key point:** Although the command is called `inside source`, it changes the destination address on return traffic. The command names the host whose address is being mapped, not the packet field that must change in every direction.

---

## Outside-Initiated Traffic

Static inside source NAT also supports traffic initiated from the outside. LH5 does not need to initiate a connection first.

For example, RH7 can ping LH5 using LH5's **inside global** address:

```text
Remote_Host_7# ping 7.7.7.50
```

The forwarding sequence is:

```text
RH7 (3.3.3.7)
  |
  | Destination 7.7.7.50
  v
ISP1 (3.3.3.1 / 7.7.7.1)
  |
  | Destination 7.7.7.50
  v
Local_Router (7.7.7.2)
  |
  | NAT: destination 7.7.7.50 -> 10.1.1.5
  v
LH5 (10.1.1.5)
```

The reply from LH5 travels back through Local_Router, where its source address is changed from `10.1.1.5` to `7.7.7.50`.

This is one of the main differences between a static address mapping and an ordinary dynamic NAT pool assignment: the static mapping is available before a connection is initiated by the inside host.

**Important:** Static NAT is not a firewall policy. An inbound connection also requires appropriate routing, ACL permissions, and a responding application on LH5. A static mapping does not make a closed TCP port reachable.

---

## Why the Outside Global Address Works Without an ISP1 Static Route

The first article topology uses a convenient arrangement:

```text
Local_Router E0/1:  7.7.7.2/24
ISP1 E0/1:          7.7.7.1/24
Inside global:      7.7.7.50
```

Because `7.7.7.50` belongs to ISP1's directly connected `7.7.7.0/24` network, ISP1 does not look for a separate next-hop route to that address. It tries to resolve the address directly using **ARP**.

### NAT-Generated ARP Alias

With a traditional static NAT mapping, Cisco NAT can create an **IP alias** for an inside global address on an attached outside subnet. This allows Local_Router to answer ARP requests for `7.7.7.50`, even though `7.7.7.50` is not configured as the interface's primary IP address.

The Layer 2 delivery is therefore:

```text
ISP1: Who has 7.7.7.50?
                  |
                  v
Local_Router: 7.7.7.50 is reachable at my outside-interface MAC
                  |
                  v
ISP1 sends the packet to Local_Router
                  |
                  v
NAT translates 7.7.7.50 -> 10.1.1.5
```

You can view the dynamiac alias with:

```text
R1# show ip aliases 
Address Type             IP Address      Port
Interface                7.7.7.2 
Dynamic                  7.7.7.50  <--- CREATED DUE TO STATIC NAT
Interface                10.1.1.1 
```

This allows Local_Router to respond to ISP1's ARP Request for `7.7.7.50`.

**Platform note:** NAT alias behavior and its interaction with ARP can vary by IOS XE device and forwarding implementation. Verify it on the specific virtual or physical platform used in the lab. The optional `no-alias` keyword suppresses automatic alias behavior where supported and is covered in [Static NAT Options and ARP Behavior](options.md).

### Contrast: A Routed Inside Global Address

You can also use an inside global address **outside** the subnet of the NAT router's outside interface. In that design, the upstream router needs a route for the global address or prefix via Local_Router.

For example, if the global address were `198.51.100.50` instead of `7.7.7.50`, the upstream router could be configured with:

```text
ISP1(config)# ip route 198.51.100.50 255.255.255.255 7.7.7.2
```

In that case, ISP1 resolves the **next-hop** address `7.7.7.2` with ARP, not the translated address `198.51.100.50`.

Both approaches are valid. The article's topology uses the **on-link global address and IP alias** approach, so no extra route is needed on ISP1 for `7.7.7.50`.

> When the **inside global** address is **off-link**, the router does NOT create an IP alias. This is because ISP1 will not ARP for the inside global address,
> but rather for the next hop of its route (`7.7.7.2`, Local_Router's interface IP).

---

## Translation Table Entries After Traffic

When RH7 pings `7.7.7.50`, Local_Router may display a protocol-specific entry in addition to the simple static mapping:

```text
Local_Router# show ip nat translations
Pro  Inside global      Inside local       Outside local      Outside global
icmp 7.7.7.50:4        10.1.1.5:4         3.3.3.7:4          3.3.3.7:4
---  7.7.7.50           10.1.1.5           ---                ---
```

This sample follows the article's example, but protocol identifiers and the exact table contents depend on the traffic and platform.

The two rows have different functions:

| Entry | Purpose |
|---|---|
| `--- 7.7.7.50 10.1.1.5 ...` | **Static simple entry** defining the address mapping |
| `icmp ...` | **Extended entry** associated with the ICMP exchange |

The extended entry does **not** mean this is PAT. An ordinary address-only NAT mapping can still generate protocol-specific flow entries.

The permanent static mapping remains after the ICMP flow has aged out. For details on simple and extended entries and their timers, see [NAT Translation Tables and Verification](../verification.md).

### Useful Verification Commands

On Local_Router:

```text
Local_Router# show running-config | include ip nat
Local_Router# show ip nat translations
Local_Router# show ip nat translations verbose
Local_Router# show ip nat statistics
Local_Router# show ip route 3.3.3.7
Local_Router# show ip route 10.1.1.5
Local_Router# show ip interface Ethernet0/0
Local_Router# show ip interface Ethernet0/1
```

On ISP1:

```text
ISP1# show ip route 7.7.7.50
ISP1# show ip arp 7.7.7.50
ISP1# show ip route 3.3.3.7
```

For the functional test:

```text
Remote_Host_7# ping 7.7.7.50
Local_Host_5# ping 3.3.3.7
```

On a dedicated lab device, `debug ip nat` can help observe translation decisions. Use caution on production devices, and note that software debugging can differ on hardware-accelerated IOS XE forwarding platforms.

---

## What About Local_Host_6?

LH6 is connected to the same inside subnet as LH5, but **there is no static NAT rule for LH6**.

Only the configured address is mapped:

```text
Local_Router(config)# ip nat inside source static 10.1.1.5 7.7.7.50
```

If LH6 sends traffic toward `3.3.3.7`, the router may still forward it according to the routing table, but **LH6's source IP is not translated by this rule**. With no return route to `10.1.1.0/24` on ISP1, LH6 should not expect normal bidirectional connectivity through this lab topology.

To make LH6 reachable under its own fixed outside address, an additional static mapping could be configured:

```text
Local_Router(config)# ip nat inside source static 10.1.1.6 7.7.7.60
```

This is a separate one-to-one translation, not PAT or a dynamically allocated pool entry. Remove the extra mapping if you are following only Case 1 of the article.

---

## Troubleshooting Static Inside Source NAT

| Symptom | Possible cause | What to check |
|---|---|---|
| RH7 cannot reach `7.7.7.50` | ISP1 cannot resolve the NAT global address on the connected subnet | `show ip arp 7.7.7.50` on ISP1; NAT alias behavior |
| Packet reaches Local_Router, but no translation occurs | Missing static mapping, incorrect inside/outside interface roles, or platform restrictions | NAT configuration and translation table |
| Packet translates to `10.1.1.5`, but LH5 never receives it | Inside interface, subnet, ARP, or routing issue | Connected route to `10.1.1.0/24`; LH5 connectivity |
| LH5 receives traffic but its reply does not return | LH5 default gateway or return path is incorrect | Gateway `10.1.1.1`, routing and filtering |
| LH5-initiated ping fails | Local_Router has no usable route toward RH7, or the return traffic cannot reach the NAT global address | Default route `7.7.7.1`, ISP1 ARP |
| RH7 tries to reach `10.1.1.5` directly and fails | Private inside local address is not reachable in the outside routing domain | Use `7.7.7.50` instead |
| RH7 ping succeeds, but a TCP service fails | Service not listening or security policy blocks the flow | Host services and ACLs |
| LH6 cannot communicate like LH5 | Static mapping applies only to `10.1.1.5` | Add a separate mapping or appropriate dynamic NAT policy |

### Suggested Troubleshooting Order

For outside-initiated traffic from RH7:

```text
1. RH7 sends the packet to its gateway, 3.3.3.1.
2. ISP1 routes toward 7.7.7.50 on connected 7.7.7.0/24.
3. ISP1 resolves 7.7.7.50 to Local_Router through ARP.
4. Local_Router receives the packet on its NAT outside interface.
5. NAT translates destination 7.7.7.50 to 10.1.1.5.
6. Local_Router forwards to LH5 on connected 10.1.1.0/24.
7. LH5 replies through Local_Router.
8. Local_Router translates the reply source to 7.7.7.50.
9. ISP1 forwards the reply to RH7.
```

Check these independently: **delivery to the NAT router**, **the translation itself**, and **forwarding after translation**. A correct static NAT command does not guarantee the whole packet path works.

---

## Static Address NAT vs. Static PAT

The mapping in this page covers an entire address:

```text
Local_Router(config)# ip nat inside source static 10.1.1.5 7.7.7.50
```

A static PAT mapping, by contrast, identifies a particular protocol and port:

```text
Local_Router(config)# ip nat inside source static tcp 10.1.1.5 23 7.7.7.50 230
```

This would direct incoming TCP traffic for `7.7.7.50:230` to `10.1.1.5:23`, subject to route and policy requirements. **Treat this as an alternative configuration, not an additional command to apply alongside the address-only mapping:** remove the original static address mapping before testing a different mapping that uses the same addresses, unless the particular configuration is explicitly supported. Static PAT is covered in [Static PAT and Port Forwarding](pat.md).

---

## Key Points

- Static inside source NAT uses the argument order **inside local, inside global**.
- This lab maps **`10.1.1.5` to `7.7.7.50`**, matching the first topology in the referenced article.
- **Inside to outside:** source `10.1.1.5` becomes `7.7.7.50`.
- **Outside to inside:** destination `7.7.7.50` becomes `10.1.1.5`.
- An outside host can initiate communication to the inside global address even before the inside host has sent traffic.
- In this lab, ISP1 regards `7.7.7.50` as on-link. NAT alias and ARP behavior allow packets to reach Local_Router without an ISP1 static route to a separate global `/32`.
- Local_Router needs a route toward the inside local host after destination translation and toward outside destinations when forwarding inside-originated traffic.
- A static simple mapping is present independently of traffic; extended flow entries may appear when communication occurs.
- NAT is not a substitute for routing, access control, or application availability.

---

## Related Pages

- [Static NAT, PAT Overview](overview.md)
- [Static Network-to-Network NAT](network-mapping.md)
- [Static PAT and Port Forwarding](pat.md)
- [Static Outside Source NAT and PAT](outside-source.md)
- [Static NAT Options and ARP Behavior](options.md)
- [NAT Address Terminology and Translation Types](../terminology.md)
- [NAT Packet Processing and Order of Operations](../order-of-operations.md)
- [NAT Translation Tables and Verification](../verification.md)

## References

- [Jedadiah Casey, Understanding and Configuring NAT, Case 1](https://wax-trax.medium.com/understanding-and-configuring-network-address-translation-nat-for-cisco-ccna-ccnp-and-ccie-7fef92fa654c)
- [Cisco IOS XE 17.x, Configuring NAT for IP Address Conservation](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-addressing/b-ip-addressing/m_iadnat-addr-consv-xe.html)
- [Cisco, Understanding the NAT Order of Operation](https://www.cisco.com/c/en/us/support/docs/ip/network-address-translation-nat/6209-5.html)
- [Cisco IOS IP Addressing Services Command Reference: `ip nat inside source`](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/ipaddr/command/ipaddr-cr-book/ipaddr-i3.html)
