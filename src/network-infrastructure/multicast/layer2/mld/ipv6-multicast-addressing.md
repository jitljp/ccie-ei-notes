# IPv6 Multicast Addressing

Although IPv6 multicast addressing itself is a separate topic from MLD, let's review it here.

IPv6 multicast addresses use the range:

```text
ff00::/8
```

All IPv6 multicast addresses begin with the hexadecimal value `ff`.

Like IPv4 multicast, an IPv6 multicast address identifies a **group of receivers** rather than a single device.

The destination address of a multicast packet is a multicast address, while the source address must be a unicast address.

Unlike IPv4, **IPv6 does not support broadcast**. Multicast is used for many functions that traditionally relied on broadcast in IPv4.

## IPv6 Multicast Address Format

An IPv6 multicast address is 128 bits long and consists of four fields:

```text
| 8 bits  | 4 bits | 4 bits |       112 bits       |
+---------+--------+--------+----------------------+
|11111111 | Flags  | Scope  |       Group ID       |
+---------+--------+--------+----------------------+
```

| Field | Size | Purpose |
|---|---|---|
| Prefix | 8 bits | Always `11111111` (`ff`) |
| Flags | 4 bits | Identifies the multicast address allocation type |
| Scope | 4 bits | Defines the multicast distribution scope |
| Group ID | 112 bits | Identifies the multicast group; some address formats subdivide this field |

The first 16 bits determine the multicast address's flags and scope.

For example:

```text
ff02::1
```

Breaking down the first 16 bits:

```text
ff      0      2
│       │      │
Prefix  Flags  Scope
```

This is a **permanently assigned** multicast address (`Flags = 0`) with **link-local scope** (`Scope = 2`).

## Multicast Flags

The four flag bits are defined as:

```text
0 R P T
```

| Flag | Meaning |
|---|---|
| `0` | Reserved; must be zero |
| `R` | Rendezvous Point information embedded in the address |
| `P` | Multicast address derived from a unicast network prefix |
| `T` | Transient (non-permanently assigned) multicast address |

The **T bit** distinguishes two important address types:

- `T = 0`: Permanently assigned multicast group.
- `T = 1`: Transient or dynamically assigned multicast group.

If `P = 1`, the `T` bit must also be 1.

If `R = 1`, both `P` and `T` must be 1.

Flag values include:

| Flags | Address Prefix | Meaning |
|---|---|---|
| `0000` | `ff0x` | Permanently assigned multicast |
| `0001` | `ff1x` | Transient multicast |
| `0011` | `ff3x` | Unicast-prefix-based multicast, including SSM |
| `0111` | `ff7x` | Embedded-RP multicast |

Here, `x` represents the scope value.

## IPv6 Multicast Scopes

The **scope field** determines how far multicast traffic is permitted to travel.

Unlike IPv4, where multicast scope is often controlled through address blocks and administrative boundaries, IPv6 includes scope information directly in the multicast address.

| Scope Value | Name | Description |
|---|---|---|
| `1` | Interface-Local | Restricted to a single interface on a node |
| `2` | Link-Local | Restricted to a single link |
| `3` | Realm-Local | Restricted to an automatically defined realm |
| `4` | Admin-Local | Restricted to an administratively defined region |
| `5` | Site-Local | Intended for a single site |
| `8` | Organization-Local | Intended for multiple sites within an organization |
| `E` | Global | No scope boundary; potentially routable globally |

Values `0` and `F` are reserved. Values `6`, `7`, and `9` through `D` are unassigned.

> **Realm-Local (`3`)** was introduced by RFC 7346, which updated the original scope definitions in RFC 4291.

### Link-Local Scope — ff02::/16

The most commonly encountered IPv6 multicast scope is **link-local**:

```text
ff02::/16
```

Multicast packets with link-local scope must not be forwarded between network links.

For example:

```text
LAN 1                         LAN 2

Host ---- SW1 ---- R1 -------- SW2 ---- Host
|------------------> X
              ff02::/16
```

The scope restriction is determined by the **destination IPv6 address**, not simply the IPv6 Hop Limit.

Even if the packet's Hop Limit is greater than 1, a router must not forward link-local multicast traffic beyond the originating link.

### Site-Local, Organization-Local, and Global Scopes

Other scopes allow multicast traffic to span larger networks.

For example:

```text
ff15::/16   Transient, Site-Local
ff18::/16   Transient, Organization-Local
ff1e::/16   Transient, Global
```

These prefixes all have the same flags (`0001`) but different scope values.

Administrative boundaries determine the permitted forwarding regions for scopes such as Site-Local and Organization-Local.

> Do not confuse IPv6 **Site-Local multicast scope (`5`)** with the deprecated IPv6 Site-Local *unicast* address range `fec0::/10`. Site-Local multicast scope remains valid.

## Important IPv6 Multicast Addresses

Several IPv6 multicast addresses are reserved for local network control protocols.

| Address | Purpose |
|---|---|
| `ff02::1` | All Nodes |
| `ff02::2` | All Routers |
| `ff02::5` | OSPFv3 AllSPFRouters |
| `ff02::6` | OSPFv3 AllDRouters |
| `ff02::a` | EIGRP for IPv6 |
| `ff02::d` | All PIM Routers |
| `ff02::16` | All MLDv2-capable Routers |
| `ff02::1:2` | DHCPv6 Servers and Relay Agents |

### ff02::1 — All Nodes

`ff02::1` identifies all IPv6 nodes on the local link.

All IPv6-enabled interfaces listen for this multicast address.

It is used by protocols such as Neighbor Discovery to communicate with all nodes on the link.

MLD membership reports are **not sent for `ff02::1`**, because listening to the group is permanently enabled.

### ff02::2 — All Routers

`ff02::2` identifies all IPv6 routers on the local link.

For example, an IPv6 host sends a Router Solicitation to:

```text
ff02::2
```

to solicit Router Advertisements from routers on the link.

### ff02::16 — MLDv2 Routers

`ff02::16` is used as the destination address of **MLDv2 Multicast Listener Reports**.

All MLDv2-capable multicast routers on the link listen for this group.

This is similar to how IGMPv3 Reports use `224.0.0.22` in IPv4.

> 0x16 is equivalent to 0d22

## Source-Specific Multicast — ff3x::/32

IPv6 reserves the following address space for **Source-Specific Multicast (SSM)**:

```text
ff3x:0000::/32
```

Here, `x` is the multicast scope.

Common examples of SSM scope prefixes are:

```text
ff32:0000::/32   Link-Local SSM
ff35:0000::/32   Site-Local SSM
ff3e:0000::/32   Global SSM
```

SSM receivers specify both a source address and a multicast group:

```text
(S,G)
```

For example:

```text
Source: 2001:db8:10::10
Group:  ff3e::fd00:1234

Channel:
(2001:db8:10::10, ff3e::fd00:1234)
```

The multicast group uses global scope, while its group ID is within the private-use allocation.

With SSM, the receiver explicitly identifies the desired source, so an RP is not required for multicast source discovery.

**MLDv2** provides the source-specific membership signaling required for native IPv6 SSM.

### SSM Address Allocation

The defined SSM address format uses:

```text
ff3x:0000:0000:0000:0000:0000:GGGG:GGGG
```

The final 32 bits contain the group ID.

Although the entire `ff3x:0000::/32` block is classified as SSM, **actual group allocation uses `ff3x::/96`**, and not every group ID is available for arbitrary use.

RFC 10028 defines the following dynamic group ID allocations:

| Group ID Range | Purpose |
|---|---|
| `f0000000–fcffffff` | Host allocation of SSM group addresses |
| `fd000000–fdffffff` | Private Use |
| `fe000000–feffffff` | Experimental Use |
| `ff000000–ffffffff` | Reserved for Solicited-Node multicast Group IDs |

> The `ff000000–ffffffff` range refers to the **lowest 32 bits (Group ID)**, not a complete IPv6 multicast address. Actual Solicited-Node multicast addresses use `ff02::1:ff00:/104`, ranging from `ff02::1:ff00:0` through `ff02::1:ffff:ffff`.
>
> Reserving these Group IDs prevents dynamic allocation schemes from using the same low 32 bits, avoiding multicast Ethernet MAC address collisions with Solicited-Node groups (`33:33` + the lowest 32 bits).

For lab configurations, the **private-use group ID range** is particularly useful.

For example:

```text
ff35::fd00:1234
```

is a site-local SSM group using a private-use group ID.

## Unicast-Prefix-Based Multicast

RFC 3306 defines multicast addresses that incorporate an IPv6 unicast prefix.

These addresses also use flags `0011` (`ff3x`), but include additional fields:

```text
| Prefix | Flags | Scope | Reserved | Plen | Network Prefix | Group ID |
| 8 bits | 4 bits| 4 bits|  8 bits  |8 bits|    64 bits     | 32 bits  |
```

The embedded unicast prefix can identify the network responsible for allocating the multicast group.

For example, a multicast address allocated using:

```text
2001:db8:1::/48
```

could have the following form:

```text
ff3e:0030:2001:0db8:0001:0000:0000:0001
```

Here:

- `ff3e` identifies unicast-prefix-based multicast with global scope.
- `0030` contains a reserved byte (`00`) and a prefix length of `0x30` (48 decimal).
- `2001:0db8:0001:0000` contains the 64-bit network-prefix field.
- `0000:0001` identifies the multicast group.

Although both use `ff3x` flags, an ordinary unicast-prefix-based multicast address is **not necessarily an SSM address**.

SSM uses a zero prefix length and zero network-prefix field.

### Embedded-RP Addresses

RFC 3956 also defines **Embedded-RP multicast addresses**, using:

```text
ff7x::/16
```

These encode information that allows multicast routers to derive an RP address from the multicast group address.

This is an IPv6 PIM-SM mechanism and is distinct from SSM, which does not require an RP.

## Solicited-Node Multicast Addresses

IPv6 uses **Solicited-Node multicast addresses** for Neighbor Discovery.

The address range is:

```text
ff02::1:ff00:0/104
```

A Solicited-Node multicast address is calculated by combining:

```text
ff02::1:ff00:0/104
+
Lowest 24 bits of the IPv6 unicast/anycast address
```

### Example

Suppose a device has this IPv6 unicast address:

```text
2001:db8:1::1234:5678
```

The lowest 24 bits are:

```text
34:5678
```

Therefore, its Solicited-Node multicast address is:

```text
ff02::1:ff34:5678
```

A device joins the corresponding Solicited-Node multicast group for each configured unicast or anycast address, although multiple addresses can map to the same group.

### Neighbor Discovery

Solicited-Node multicast addresses are used by **Neighbor Solicitation** messages.

Instead of broadcasting an address-resolution request to every device, as ARP does in IPv4, IPv6 sends a Neighbor Solicitation to a Solicited-Node multicast group.

For example:

```text
Target IPv6:
2001:db8:1::1234:5678

Neighbor Solicitation destination:
ff02::1:ff34:5678
```

Only devices listening to that multicast group need to process the message at the IP layer.

This reduces unnecessary processing compared to IPv4 ARP broadcasts.

## IPv6 Multicast MAC Addresses

When IPv6 multicast packets are transmitted over Ethernet, their destination IPv6 addresses are mapped to Ethernet multicast MAC addresses.

IPv6 multicast Ethernet MAC addresses use the prefix:

```text
33:33
```

The remaining 32 bits are copied from the **lowest 32 bits of the IPv6 multicast address**.

```text
IPv6 multicast address:
ff02:0000:0000:0000:0000:0000:0000:0001
                              └───────┘
                          Lowest 32 bits = 00000001

Ethernet destination MAC:
33:33:00:00:00:01
```

Examples:

| IPv6 Multicast Address | Ethernet MAC Address |
|---|---|
| `ff02::1` | `33:33:00:00:00:01` |
| `ff02::2` | `33:33:00:00:00:02` |
| `ff02::5` | `33:33:00:00:00:05` |
| `ff02::16` | `33:33:00:00:00:16` |
| `ff02::1:ff34:5678` | `33:33:FF:34:56:78` |
| `ff35::fd00:1234` | `33:33:FD:00:12:34` |

### Multicast MAC Address Collisions

The Ethernet multicast MAC address contains only **32 bits from the IPv6 multicast address**.

Therefore, multiple IPv6 multicast groups can map to the same Ethernet MAC address.

For example:

```text
ff02::2
ff05::2
```

Both map to:

```text
33:33:00:00:00:02
```

These addresses have different IPv6 scopes but the same lower 32 bits.

As with IPv4 multicast MAC mapping, **IPv6-to-Ethernet multicast mapping is not one-to-one**.

A device may therefore receive a frame whose Ethernet multicast destination matches an address it listens for, even though the IPv6 multicast destination is different.

The IPv6 layer can discard traffic for groups the device has not joined.

## Key Points

- IPv6 multicast addresses use `ff00::/8`.
- The address format includes an 8-bit prefix, 4-bit flags, 4-bit scope, and 112-bit group ID field.
- The **scope field** determines how far multicast traffic may travel.
- `ff02::/16` identifies permanently assigned link-local multicast addresses.
- `ff02::1` is All Nodes, `ff02::2` is All Routers, and `ff02::16` is used for MLDv2 Reports.
- IPv6 SSM uses `ff3x:0000::/32`, with group allocation defined within `ff3x::/96`.
- **Solicited-Node multicast addresses** use `ff02::1:ff00:0/104` and the lowest 24 bits of an IPv6 unicast or anycast address.
- IPv6 multicast Ethernet MAC addresses use **`33:33` + the lowest 32 bits** of the IPv6 multicast address.
- Multiple IPv6 multicast groups can map to the same Ethernet multicast MAC address.