# IPv4 Multicast Addressing

IPv4 multicast uses the address range:

```text
224.0.0.0/4
```

This includes all addresses from 224.0.0.0 through 239.255.255.255.

Historically, this was called the **Class D** address range.

In binary, every IPv4 multicast address begins with:

```text
1110
```

```text
1110xxxx.xxxxxxxx.xxxxxxxx.xxxxxxxx
^^^^
Multicast
```

The first four bits identify the address as multicast.

The remaining 28 bits identify the multicast group.

```text
4 bits  = multicast prefix
28 bits = multicast group address
```

This provides 2^28 possible multicast addresses, although large portions of the space have special purposes or are reserved.

---

## Multicast Addresses Identify Groups

A multicast address identifies a **group**, not a specific host.

For example:

```text
239.1.1.1
```

could represent a multicast group with many receivers.

```text
                  Host A
                    |
                    | joined 239.1.1.1
                    |
Source -------- Multicast Network
                    |
                    | joined 239.1.1.1
                    |
                  Host B
```

Both hosts are members of the same group:

```text
239.1.1.1
```

The multicast address does not identify Host A or Host B individually.

Instead:

```text
Unicast address
    = identifies an interface/device

Multicast address
    = identifies a group of receivers
```

---

## Multicast Addresses Are Destination Addresses

An important rule is that an IPv4 multicast address is used as the **destination address** of a multicast packet.

The source address is a normal **unicast address**.

For example:

```text
Source host:
10.1.1.10

Multicast group:
239.1.1.1
```

The IPv4 packet contains:

```text
Source IP:      10.1.1.10
Destination IP: 239.1.1.1
```

It does **not** use a multicast address as its source address.

```text
Valid:

10.1.1.10 → 239.1.1.1


Not normal multicast operation:

239.1.1.2 → 239.1.1.1
```

> Multicast addresses identify multicast **groups**, not multicast sources.

---

## Major IPv4 Multicast Address Ranges

The `224.0.0.0/4` multicast space is divided into several ranges with different purposes.

The most important ranges for CCIE EI are:

| Range | Purpose |
| --- | --- |
| `224.0.0.0/24` | Local Network Control |
| `224.0.1.0` through `231.255.255.255` | Various globally scoped, assigned, and reserved multicast ranges |
| `232.0.0.0/8` | Source-Specific Multicast (SSM) |
| `233.0.0.0/8` | Primarily GLOP and other assigned multicast space |
| `234.0.0.0/8` | Unicast-prefix-based multicast |
| `235.0.0.0/8` through `238.255.255.255` | Reserved |
| `239.0.0.0/8` | Administratively scoped multicast |

For CCIE EI, the ranges most worth memorizing are:

```text
224.0.0.0/4   = all IPv4 multicast

224.0.0.0/24  = Local Network Control

232.0.0.0/8   = SSM

239.0.0.0/8   = administratively scoped multicast
```

---

## Local Network Control

The range:

```text
224.0.0.0/24
```

is the **Local Network Control Block**.

It contains multicast addresses used by routing protocols and other network control protocols.

Examples include:

```text
224.0.0.1
224.0.0.2
224.0.0.5
224.0.0.6
224.0.0.9
224.0.0.10
224.0.0.13
```

These addresses have **link-local scope**.

Routers do not forward packets destined for this range beyond the local network segment.

```text
LAN 1                           LAN 2

Host ---- R1 -------------------- R2
           ^
           |
     224.0.0.x traffic
     stops at the local link
```

This remains true even if the packet has a TTL greater than 1.

> The individual Local Network Control multicast addresses are covered on the next page.

---

## Globally Scoped Multicast

Multicast addresses outside the link-local and administratively scoped ranges can have broader scope.

Cisco documentation traditionally describes:

```text
224.0.1.0
through
238.255.255.255
```

as the **globally scoped** multicast range.

However, this large block contains several important special-purpose and reserved subranges, including:

```text
232.0.0.0/8 = SSM

233.x.x.x    = GLOP and other assigned space

234.0.0.0/8 = unicast-prefix-based multicast

235.0.0.0/8 - 238.0.0.0/8 = reserved
```

Therefore, it is better to think of the multicast address space as a collection of allocated ranges rather than assuming every address from `224.0.1.0` through `238.255.255.255` is freely usable.

---

## Source-Specific Multicast Range

The IPv4 Source-Specific Multicast range is:

```text
232.0.0.0/8
```

SSM uses a specific **source and group combination**.

For example:

```text
Source:
10.1.1.10

Group:
232.1.1.1
```

The receiver requests:

```text
(10.1.1.10, 232.1.1.1)
```

or:

```text
(S,G)
```

This differs from traditional Any-Source Multicast, where a receiver joins a group without necessarily identifying the source.

```text
ASM:

Join G


SSM:

Join (S,G)
```

Because the receiver already identifies the desired source, SSM does not require an RP for source discovery.

On Cisco IOS XE, the default IPv4 SSM range is:

```text
232.0.0.0/8
```

The SSM range can also be changed using configuration.

> SSM operation is covered in detail in the Source-Specific Multicast section.

---

## Administratively Scoped Multicast

The range:

```text
239.0.0.0/8
```

is reserved for **administratively scoped multicast**.

It is intended for multicast traffic that should remain within a defined administrative domain.

For example, an organization might use:

```text
239.1.1.1
```

for an internal multicast application.

```text
                 Organization

Source ---- R1 ---- R2 ---- R3 ---- Receivers
            <---- 239.1.1.1 ---->

              multicast boundary
                     |
                     X
                     |
                 Outside network
```

Administratively scoped addresses are conceptually similar to private IPv4 unicast addresses.

```text
Private unicast:
10.0.0.0/8
172.16.0.0/12
192.168.0.0/16

Administratively scoped multicast:
239.0.0.0/8
```

However, the mechanisms are different.

A `239.x.x.x` address is still a multicast group address.

Network administrators use multicast boundaries and related mechanisms to control where the multicast traffic is allowed to travel.

Because the scope is locally administered, the same `239.x.x.x` group can be reused in separate multicast domains.

> Multicast boundaries and administratively scoped multicast are covered in more detail later.

---

## GLOP Addressing

A portion of the `233.0.0.0/8` range was historically allocated for **GLOP addressing**.

GLOP provided organizations with multicast addresses based on their public Autonomous System number.

Conceptually:

```text
16-bit AS number
      ↓
Converted into part of a 233.x.x.x multicast prefix
      ↓
Organization receives multicast group space
```

For example, an organization with a particular 16-bit AS number could derive a `/24` multicast range within the GLOP block.

GLOP was designed when 16-bit AS numbers were common.

It is mainly of historical interest today and is not a major CCIE EI multicast topic.

---

## Group Address vs Prefix

When we write:

```text
232.0.0.0/8
```

the `/8` describes an **address allocation range**.

An actual multicast application sends traffic to a particular group address, such as:

```text
232.10.10.10
```

The packet itself simply has:

```text
Destination IP = 232.10.10.10
```

The `/8` is not carried in the multicast packet.

Similarly:

```text
239.0.0.0/8
```

describes the administratively scoped address block.

An application might use:

```text
239.1.1.1
239.10.20.30
239.100.1.1
```

as individual multicast groups within that range.

---

## ASM and SSM Addressing

Two important multicast service models are **ASM** and **SSM**.

### Any-Source Multicast

With **Any-Source Multicast (ASM)**, the receiver requests a group without specifying a source.

```text
Receiver wants:

239.1.1.1
```

Conceptually:

```text
(*,G)
```

Multiple sources could potentially send traffic to the same group.

```text
Source A ----\
              \
               Multicast Group 239.1.1.1 ---- Receiver
              /
Source B ----/
```

ASM commonly uses PIM Sparse Mode with an RP.

### Source-Specific Multicast

With **Source-Specific Multicast (SSM)**, the receiver specifies both the source and group.

```text
Receiver wants:

Source = 10.1.1.10
Group  = 232.1.1.1
```

Conceptually:

```text
(S,G)
```

```text
(10.1.1.10, 232.1.1.1)
```

The standard IPv4 SSM block is:

```text
232.0.0.0/8
```

A useful way to remember the distinction is:

```text
ASM:
"I want traffic for group G."

SSM:
"I want traffic from source S for group G."
```

---

## Multicast Addresses Are Not Assigned to Interfaces

A multicast group address is generally not configured as a normal IP address on an interface.

For example, a router interface might have:

```text
interface GigabitEthernet0/0
 ip address 10.1.1.1 255.255.255.0
```

The interface does not need:

```text
ip address 239.1.1.1 ...
```

for the router to receive or forward traffic for group `239.1.1.1`.

Instead, hosts and routers **join groups**.

```text
Interface unicast address:
10.1.1.1

Multicast group membership:
239.1.1.1
```

These are different concepts.

---

## One Host Can Join Multiple Groups

A single host can be a member of many multicast groups simultaneously.

For example:

```text
Host A

10.1.1.100
    |
    +-- joins 239.1.1.1
    |
    +-- joins 239.1.1.2
    |
    +-- joins 232.10.10.10 from source 10.2.2.2
```

Likewise, many hosts can join the same group.

```text
Host A ----\
Host B -----+---- 239.1.1.1
Host C ----/
```

Group membership is independent of the host's normal unicast IP address.

---

## Multicast Group Address and Ethernet MAC Address

When IPv4 multicast traffic is carried over Ethernet, the IPv4 multicast destination address must be mapped to an Ethernet multicast MAC address.

Conceptually:

```text
IPv4 multicast group
239.1.1.1
      ↓
IPv4 multicast-to-MAC mapping
      ↓
Ethernet multicast MAC address
```

The IPv4-to-Ethernet multicast mapping does not provide a unique MAC address for every multicast IP address.

Multiple multicast IP addresses can map to the same Ethernet multicast MAC address.

> The exact mapping process is covered on the IPv4 Multicast MAC Addresses page.

---

## Example

Suppose a video server has this address:

```text
10.1.1.10
```

and sends a video stream to:

```text
239.10.10.10
```

The packet is:

```text
Source IP:
10.1.1.10

Destination IP:
239.10.10.10
```

Three receivers join the group:

```text
Receiver A:
239.10.10.10

Receiver B:
239.10.10.10

Receiver C:
239.10.10.10
```

The network builds multicast forwarding state for the group.

```text
                     Receiver A
                        ^
                        |
10.1.1.10 ---- R1 ---- R2 ---- Receiver B
                        |
                        v
                     Receiver C

Group = 239.10.10.10
```

The source still sends a single multicast stream.

The routers replicate the traffic where the distribution tree branches.

---

## Important Address Ranges

For CCIE EI, remember these first:

```text
224.0.0.0/4
    All IPv4 multicast addresses

224.0.0.0/24
    Local Network Control
    Not forwarded between network segments

232.0.0.0/8
    Source-Specific Multicast (SSM)

239.0.0.0/8
    Administratively scoped multicast
```

You should also recognize:

```text
233.x.x.x
    Includes GLOP multicast addressing

234.0.0.0/8
    Unicast-prefix-based multicast
```

---

## Key Points

* IPv4 multicast uses `224.0.0.0/4`.
* The IPv4 multicast range is `224.0.0.0` through `239.255.255.255`.
* IPv4 multicast was historically called **Class D** addressing.
* IPv4 multicast addresses begin with the binary bits `1110`.
* The remaining 28 bits identify the multicast group.
* A multicast address identifies a **group of receivers**, not an individual host.
* Multicast addresses are used as destination addresses.
* A multicast packet normally uses a unicast source address and a multicast destination address.
* `224.0.0.0/24` is the Local Network Control block.
* Routers do not forward traffic in `224.0.0.0/24` between network segments.
* `232.0.0.0/8` is the standard IPv4 SSM range.
* SSM identifies traffic using `(S,G)`.
* `239.0.0.0/8` is reserved for administratively scoped multicast.
* Administratively scoped multicast addresses can be reused in separate multicast domains.
* `233.x.x.x` includes the historical GLOP multicast allocation.
* Multicast group addresses are not configured as ordinary unicast addresses on router interfaces.
* A host can join multiple multicast groups.
* Multiple hosts can join the same multicast group.
* IPv4 multicast addresses are mapped to Ethernet multicast MAC addresses when sent over Ethernet.