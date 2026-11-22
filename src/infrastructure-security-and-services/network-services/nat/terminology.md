# NAT Address Terminology and Translation Types

**Network Address Translation (NAT)** changes IP addresses, and optionally transport-layer port numbers, as packets pass through a router. To understand NAT configuration and troubleshooting, it is essential to distinguish **which host is being represented**, **where an address is used**, and **which packet fields are translated**.

Cisco IOS XE uses four address terms:

- **Inside local**
- **Inside global**
- **Outside local**
- **Outside global**

It also distinguishes **inside source**, **outside source**, and **inside destination** translation. These are separate concepts from whether a translation is **static**, **dynamic**, or uses **Port Address Translation (PAT)**.

---

## Inside and Outside NAT Domains

In IOS XE NAT, interfaces are assigned NAT roles:

```text
                 R1 (NAT Router)

   Inside network                     Outside network
  10.10.10.0/24                        198.51.100.0/24
         |                                     |
         | G1                              G2 |
         +------------[  R1  ]----------------+
            ip nat inside    ip nat outside
```

The `inside` and `outside` terms describe **NAT domains**, not the direction in which a particular packet happens to be traveling.

```text
R1(config)# interface GigabitEthernet1
R1(config-if)# ip nat inside

R1(config)# interface GigabitEthernet2
R1(config-if)# ip nat outside
```

An inside host can initiate a connection to an outside host, or an outside host can initiate a connection to an inside host when the NAT configuration permits it. The NAT interface roles remain the same in both cases.

**Inside does not necessarily mean private, and outside does not necessarily mean the Internet.** For example, NAT can connect two organizations with overlapping address spaces or operate between a VRF and a shared network.

---

## The Four NAT Address Types

The words **inside** and **outside** identify *which host* an address represents. The words **local** and **global** identify *how that host is addressed from the respective side of the NAT boundary*.

| Term | Meaning |
|---|---|
| **Inside local** | Address of an inside host as it appears on the inside network, commonly the host's actual configured address |
| **Inside global** | Address that represents the inside host to the outside network |
| **Outside local** | Address that represents an outside host to the inside network |
| **Outside global** | Address of the outside host as it appears on the outside network, commonly its actual configured address |

In the most common NAT deployment, **only inside addresses are translated**. Consequently, the outside local and outside global addresses are identical.

### Example: Inside Source NAT

Suppose an inside PC uses `10.10.10.10`, and R1 translates it to `203.0.113.10` when the PC communicates with a server at `198.51.100.20`.

```text
      Inside network                    Outside network

     PC                  R1                   Server
  10.10.10.10       NAT Router           198.51.100.20
       |                 |                      |
       +-----------------+----------------------+

  Inside local       Inside global       Outside host
  10.10.10.10        203.0.113.10        198.51.100.20
```

The four addresses are:

| Address type | Address | Explanation |
|---|---|---|
| Inside local | `10.10.10.10` | PC's address on the inside network |
| Inside global | `203.0.113.10` | Address used to represent the PC outside |
| Outside local | `198.51.100.20` | Server's address as known inside |
| Outside global | `198.51.100.20` | Server's address on the outside network |

The outside local and outside global addresses are the same because **the server's address is not being translated**.

> **Important:** `local` and `global` do not literally mean *private* and *public*. An inside local address can be globally routable, and an outside local address can be from private address space. These terms describe address representations, not whether an address belongs to RFC 1918 space.

The addresses in the `192.0.2.0/24`, `198.51.100.0/24`, and `203.0.113.0/24` ranges are reserved for documentation examples.

---

## Inside Source Translation

**Inside source NAT** maps an **inside local** address to an **inside global** address.

For example:

```text
R1(config)# ip nat inside source static 10.10.10.10 203.0.113.10
```

In this command:

- `inside` identifies the host whose address is mapped.
- `source` identifies the source-address mapping used for traffic originating from that inside host.
- `10.10.10.10` is the **inside local** address.
- `203.0.113.10` is the **inside global** address.

### Packet Traveling Inside to Outside

The PC connects to the server using TCP destination port 443 and source port 50000:

| Location | Source | Destination |
|---|---|---|
| Before NAT, on the inside | `10.10.10.10:50000` | `198.51.100.20:443` |
| After NAT, on the outside | `203.0.113.10:50000` | `198.51.100.20:443` |

The router changes the **source IP address**. This is ordinary static address translation, so the TCP ports are unchanged.

### Return Packet Traveling Outside to Inside

The server replies:

| Location | Source | Destination |
|---|---|---|
| Before NAT, on the outside | `198.51.100.20:443` | `203.0.113.10:50000` |
| After NAT, on the inside | `198.51.100.20:443` | `10.10.10.10:50000` |

For the return packet, the router changes the **destination IP address**.

**Key point:** The term *inside source NAT* describes the address mapping. It does **not** mean that NAT can only modify a packet's source field. The reverse direction uses the same mapping to translate the destination.

---

## Outside Source Translation

**Outside source NAT** maps an **outside global** address to an **outside local** address.

This is useful when outside hosts need to appear under different addresses to the inside network, particularly when networks use overlapping address ranges.

Suppose the outside server's real address is `198.51.100.20`, but inside devices should access it as `10.10.20.20`.

```text
   Inside PC                 R1                Outside server
  10.10.10.10            NAT Router           198.51.100.20
       |                      |                       |
       |          Outside local: 10.10.20.20          |
       +----------------------+-----------------------+
```

Configure a static outside source mapping:

```text
R1(config)# ip nat outside source static 198.51.100.20 10.10.20.20 
```

**Notice the argument order:** For `ip nat outside source static`, the command specifies the **outside local address first**, then the **outside global address**.

### Packet Traveling Outside to Inside

If the server sends a packet to the PC:

| Location | Source | Destination |
|---|---|---|
| Before NAT, on the outside | `198.51.100.20` | `10.10.10.10` |
| After NAT, on the inside | `10.10.20.20` | `10.10.10.10` |

The router translates the **source** from outside global to outside local.

### Packet Traveling Inside to Outside

If the PC sends a packet toward the server's translated address:

| Location | Source | Destination |
|---|---|---|
| Before NAT, on the inside | `10.10.10.10` | `10.10.20.20` |
| After NAT, on the outside | `10.10.10.10` | `198.51.100.20` |

The router translates the **destination** from outside local to outside global.

**Important:** Outside source NAT requires the routing configuration to steer packets correctly toward the translated outside host. In some scenarios, the `add-route` option or a manually configured route is required. The detailed routing behavior is covered in [Static Outside Source NAT and PAT](static/outside-source.md).

---

## Inside Source vs. Outside Source NAT

Both types support translation in **both directions**. The difference is **which host's address is mapped**.

| Feature | Inside Source NAT | Outside Source NAT |
|---|---|---|
| Host being represented | Inside host | Outside host |
| Maps | Inside local ↔ inside global | Outside global ↔ outside local |
| Inside-to-outside packet | Translates source | Translates destination |
| Outside-to-inside packet | Translates destination | Translates source |
| Typical purpose | Represent inside hosts using different outside-facing addresses | Represent outside hosts using different inside-facing addresses |
| Example command | `ip nat inside source static ...` | `ip nat outside source static ...` |

### Translating Both Sides

The same router can map **both the inside and outside hosts**.

For example:

```text
R1(config)# ip nat inside source static 10.10.10.10 203.0.113.10
R1(config)# ip nat outside source static 10.10.20.20 198.51.100.20
```

An inside-to-outside packet can appear as follows:

```text
Inside network                             Outside network

Source: 10.10.10.10                       Source: 203.0.113.10
Destination: 10.10.20.20                  Destination: 198.51.100.20
                       ---- R1 ---->
```

This example changes **both the source and destination IP addresses** as the packet crosses the NAT router. The return traffic undergoes the corresponding reverse translations. Routing must be configured so that both directions traverse the NAT device.

This is useful when connecting networks that have overlapping or incompatible address plans. See [Twice NAT and Overlapping Networks](policy-based/twice-nat.md) for more advanced cases.

---

## Inside Destination Translation

**Inside destination NAT** selects an inside host by translating the **destination** of an incoming connection.

A classic Cisco IOS XE use case is **TCP server load balancing**: clients connect to one virtual IP address, and the router translates individual connections to different real servers in a rotary address pool.

```text
      Outside client
      198.51.100.20
             |
             v
     Virtual destination
       203.0.113.100
             |
          NAT R1
           /   \
          /     \
         v       v
  10.10.10.11  10.10.10.12
    Server A     Server B
```

The corresponding NAT configuration uses the `ip nat inside destination list ... pool ...` command and a NAT pool of type `rotary`.

**Do not confuse this with the reverse direction of inside source NAT.** A static inside source mapping also changes the *destination* of incoming return traffic, but it is still an **inside source mapping**. Inside destination NAT is a distinct configuration mechanism whose primary function is to select the inside destination, such as a backend server.

See [Inside Destination NAT and Load Balancing](dynamic/destination-nat.md) for configuration and limitations.

---

## Static vs. Dynamic Translation

The words **static** and **dynamic** describe **how a NAT mapping is established**, not which side of the packet is translated.

| Type | Description | Typical application |
|---|---|---|
| **Static NAT** | Explicitly configured address mapping | Server with a predictable translated address |
| **Dynamic NAT** | Mapping selected from an address pool when matching traffic creates a translation | Hosts sharing a pool of available addresses |
| **Static PAT** | Explicitly configured mapping that includes a TCP or UDP port | Port forwarding to an internal service |
| **Dynamic PAT** | Dynamically translates addresses and differentiates sessions using ports or similar identifiers | Many inside hosts sharing an outside address |

Dynamic translations are subject to allocation and timeout behavior. A static mapping is configured persistently, although associated session-specific entries can still appear and expire.

### NAT vs. PAT

Ordinary address NAT changes the IP address. **PAT (Port Address Translation)** can also change TCP/UDP port numbers, allowing many sessions to share an IP address.

For example, two PCs connect to the same web server:

| Inside packet source | Translated packet source | Destination |
|---|---|---|
| `10.10.10.10:50000` | `203.0.113.10:30001` | `198.51.100.20:443` |
| `10.10.10.11:50000` | `203.0.113.10:30002` | `198.51.100.20:443` |

Both inside hosts use **the same inside global IP address**, but NAT distinguishes their sessions using translated source ports.

PAT does not always need to change the port number. It can preserve the original port when there is no conflict. For ICMP, routers can use the ICMP identifier rather than a TCP/UDP port.

In Cisco IOS XE, the `overload` keyword enables the familiar dynamic PAT behavior.

---

## Other Ways to Classify NAT

A translation can belong to several categories at once. For example, a configuration might use **dynamic**, **inside source**, **route-map-based**, **VRF-aware PAT**.

| Dimension | Question answered | Examples |
|---|---|---|
| Host and translation direction | Which host's address is represented, and which mapping is used? | Inside source, outside source, inside destination |
| Mapping allocation | How is the mapping established? | Static, dynamic |
| Fields changed | Does NAT translate addresses only or also ports? | NAT, PAT |
| Match conditions | Which traffic should be translated? | ACL-based, route-map-based (policy-based) |
| Routing context | In which routing table does NAT operate? | Global routing table, VRF-aware NAT |
| Inter-VRF forwarding | How does traffic cross routing contexts? | VASI combined with NAT |

**VRF-aware NAT** considers the packet's VRF context when applying translation rules. **VASI NAT** uses paired virtual interfaces to steer packets between routing contexts, allowing NAT policies to be applied along that path. These are additional deployment mechanisms, not replacements for the four address terms.

---

## Interpreting `show ip nat translations`

Cisco IOS XE displays translations using the same four address labels:

```text
R1# show ip nat translations
Pro  Inside global       Inside local         Outside local        Outside global
```

A conceptual dynamic PAT entry might look like this:

```text
tcp  203.0.113.10:30001  10.10.10.10:50000   198.51.100.20:443  198.51.100.20:443
```

Reading the entry:

1. **Inside local:** The original inside endpoint is `10.10.10.10:50000`.
2. **Inside global:** It appears outside as `203.0.113.10:30001`.
3. **Outside local:** The destination appears inside as `198.51.100.20:443`.
4. **Outside global:** The server is also `198.51.100.20:443` outside.

The outside addresses are identical because the outside host has not been translated.

A static address mapping may show dashes in the outside columns because it describes an address mapping rather than one particular conversation. Individual flows can produce more specific entries.

For more on interpreting translation entries, timers, and counters, see [NAT Translation Tables and Verification](verification.md).

---

## NAT Terminology: Common Mistakes

- **Inside/local does not mean private.** NAT's local/global terms describe address representations, not RFC 1918 status.
- **Inside/outside does not identify the direction of every packet.** Both request and reply packets may traverse the same NAT interface pair in opposite directions.
- **`source` does not mean only the source IP field is ever changed.** The reverse-direction packet typically has its destination translated.
- **Static and dynamic do not mean inbound and outbound.** They refer to how mappings are created.
- **PAT is not necessarily dynamic.** Both static PAT (port forwarding) and dynamic PAT (`overload`) exist.
- **Inside destination NAT is not synonymous with all destination-address changes.** Cisco uses it for a specific NAT configuration mechanism.
- **NAT does not replace routing.** Packets still need valid forwarding and return paths, including routes for translated addresses where necessary.

---

## Related Pages

- [NAT Packet Processing and Order of Operations](order-of-operations.md)
- [NAT Translation Tables and Verification](verification.md)
- [Static Inside Source NAT](static/inside-source.md)
- [Static Outside Source NAT and PAT](static/outside-source.md)
- [Dynamic PAT and NAT Overload](dynamic/pat.md)
- [Inside Destination NAT and Load Balancing](dynamic/destination-nat.md)
- [Twice NAT and Overlapping Networks](policy-based/twice-nat.md)
- [VRF-Aware NAT, PAT](vrf/overview.md)
- [VASI NAT](vasi/overview.md)

## References

- [Cisco: IP Addressing Configuration Guide, Cisco IOS XE 17.x, Configuring NAT for IP Address Conservation](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-addressing/b-ip-addressing/m_iadnat-addr-consv-xe.html)
- [Cisco: Use Local and Global NAT Terms](https://www.cisco.com/c/en/us/support/docs/ip/network-address-translation-nat/4606-8.html)
- [Cisco: Understand the NAT Order of Operation](https://www.cisco.com/c/en/us/support/docs/ip/network-address-translation-nat/6209-5.html)
