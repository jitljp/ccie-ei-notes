# Static Network-to-Network NAT

**Static network-to-network NAT** maps an entire inside IPv4 subnet to another subnet using a single `ip nat inside source static network` command.

Instead of manually configuring a one-to-one static NAT statement for each host, the router preserves the **host portion** of the IPv4 address and replaces the **network portion**. The resulting relationship is predictable and bidirectional.

For example, when `10.1.1.0/24` is mapped to `7.7.7.0/24`:

| Inside local | Inside global |
|---|---|
| `10.1.1.5` | `7.7.7.5` |
| `10.1.1.6` | `7.7.7.6` |
| `10.1.1.50` | `7.7.7.50` |
| `10.1.1.100` | `7.7.7.100` |

This is **not Port Address Translation (PAT)**: each inside host receives its own corresponding inside global address, and the translation does not inherently alter TCP or UDP port numbers.

Unlike dynamic NAT with a pool, the translated address is **not allocated arbitrarily** when a host starts communicating. The mapping is fixed by the configured prefixes.

> **Scope:** This page covers traditional *inside source* network-to-network NAT on Cisco IOS XE. Outside source network translation, dynamic NAT pools, policy-based NAT, and stateless static NAT are related but distinct features. Feature support and the precise translation-table presentation can vary by platform and release.

---

## Configuration Syntax

```text
R1(config)# ip nat inside source static network <inside-local-network> <inside-global-network> <mask>
```

On platforms accepting CIDR-style mask notation, for example:

```text
Local_Router(config)# ip nat inside source static network 10.1.1.0 7.7.7.0 /24
```

The arguments are:

| Argument | Meaning | Example |
|---|---|---|
| `inside-local-network` | Original prefix used by inside hosts | `10.1.1.0` |
| `inside-global-network` | Translated prefix seen by outside hosts | `7.7.7.0` |
| `mask` | Network mask defining the translated prefix and host bits | `/24` |

**Inside local comes first; inside global comes second.** Both prefixes use the same mask, so the host bits are preserved.

The command reference also documents mask-based syntax, and some Catalyst IOS XE NAT implementations use the `prefix_len` or `subnet` form. Check the exact syntax supported by the image with `ip nat inside source static network ?`.

Cisco references:

- [IOS XE IP Addressing Services Command Reference](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9400/software/release/26-x/command_reference/b_26x_9400_cr/ip_addressing__services_commands.html)
- [IOS XE 17.x NAT Configuration Guide](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-addressing/b-ip-addressing/m_iadnat-addr-consv-xe.html)

### How the Host Bits Are Preserved

With a `/24` mask, the first three octets identify the network and the last octet identifies the host:

```text
Inside local:   10.1.1.5
                  |    |
           10.1.1.0    +-- Host bits: 5

Inside global:   7.7.7.5
                  |    |
            7.7.7.0    +-- Same host bits: 5
```

The general principle is:

```text
Inside global = Global network prefix + Original host bits
```

This also works with masks other than `/24`. For example, mapping `10.1.1.0/27` to `198.51.100.32/27` preserves the five host bits:

| Inside local | Inside global |
|---|---|
| `10.1.1.1` | `198.51.100.33` |
| `10.1.1.5` | `198.51.100.37` |
| `10.1.1.30` | `198.51.100.62` |

The addresses at the beginning and end of a conventional subnet are its network and broadcast addresses, not ordinary host addresses.

---

## Lab Topology

We'll use the same devices and IP addressing as **Case 6** of Jedadiah Casey's NAT article, which builds on the topology used in the preceding [Static Inside Source NAT](inside-source.md) page.

```text
      INSIDE NETWORK                 OUTSIDE NETWORK

  Local_Host_5 (LH5)
     10.1.1.5/24
          |
          |                  Local_Router                   ISP1
          +---- LAN --------- E0/0    E0/1 ---------------- E0/0
          |               10.1.1.1   7.7.7.2             7.7.7.1
          |               NAT inside  NAT outside            |
  Local_Host_6 (LH6)                                         |
     10.1.1.6/24                                  E0/1: 3.3.3.1/24
                                                             |
                                                        Outside LAN
                                                             |
                                                      +------+------+
                                                      |             |
                                             Remote_Host_7   Remote_Host_8
                                              3.3.3.7/24     3.3.3.8/24
```

| Device | Interface or role | Address |
|---|---|---|
| Local_Host_5 | Host | `10.1.1.5/24` |
| Local_Host_6 | Host | `10.1.1.6/24` |
| Local_Router | `E0/0`, NAT inside | `10.1.1.1/24` |
| Local_Router | `E0/1`, NAT outside | `7.7.7.2/24` |
| ISP1 | Toward Local_Router | `7.7.7.1/24` |
| ISP1 | Toward remote hosts | `3.3.3.1/24` |
| Remote_Host_7 | Host | `3.3.3.7/24` |
| Remote_Host_8 | Host | `3.3.3.8/24` |

LH5 and LH6 use `10.1.1.1` as their default gateway. RH7 and RH8 use `3.3.3.1` as their default gateway. Local_Router has a default route toward ISP1.

**Lab caveat:** In this exercise, `7.7.7.0/24` is both the outside link subnet *and* the translated network. That reproduces the article and makes ARP behavior interesting, but it also means `7.7.7.1` and `7.7.7.2` are already assigned to network devices. Do not give an inside host an address that would map to either occupied outside address. A separate, routed inside-global prefix is a cleaner real-world design, as demonstrated later.

### Step 1: Configure NAT Inside and Outside Interfaces

On Local_Router:

```text
Local_Router(config)# interface Ethernet0/0
Local_Router(config-if)# ip address 10.1.1.1 255.255.255.0
Local_Router(config-if)# ip nat inside
Local_Router(config-if)# no shutdown

Local_Router(config)# interface Ethernet0/1
Local_Router(config-if)# ip address 7.7.7.2 255.255.255.0
Local_Router(config-if)# ip nat outside
Local_Router(config-if)# no shutdown
```

The important distinction is the *NAT interface role*, not whether an address is private or public. Packets from the `10.1.1.0/24` LAN toward ISP1 cross from `ip nat inside` to `ip nat outside`.

### Step 2: Configure a Default Route

```text
Local_Router(config)# ip route 0.0.0.0 0.0.0.0 7.7.7.1
```

For inside-to-outside traffic, the route to the **original destination** must lead out a NAT outside interface. NAT is not a replacement for routing.

Ensure ISP1 has reachability to the remote network and that the remote hosts have the correct default gateway. Since ISP1 is directly connected to both `7.7.7.0/24` and `3.3.3.0/24`, it does not require a separate static route to either directly connected subnet.

### Step 3: Configure the Network Mapping

Remove any earlier, potentially overlapping static or dynamic NAT statements from previous labs before applying the new mapping.

```text
Local_Router(config)# ip nat inside source static network 10.1.1.0 7.7.7.0 /24
```

This one command conceptually establishes the following one-to-one relationships:

```text
10.1.1.5   <--> 7.7.7.5
10.1.1.6   <--> 7.7.7.6
10.1.1.50  <--> 7.7.7.50
10.1.1.100 <--> 7.7.7.100
```

No NAT ACL, address pool, or `overload` keyword is required. A static *network mapping* differs from dynamic translation of an ACL-selected set of hosts.

---

## NAT Table: Network Entry and Individual Host Entries

A particularly important point is that a **static network mapping does not necessarily produce one permanent, individual table row for every host in the subnet**.

The `show ip nat translations` output may initially contain a **network-wide entry**:

```text
Local_Router# show ip nat translations
Pro  Inside global      Inside local       Outside local      Outside global
---  7.7.7.0            10.1.1.0           ---                ---
```

`show ip nat translations verbose` identifies that entry with `static, network` flags. It represents the *configured rule*, not a flow to the `.0` host.

```
Local_router# show ip nat translations verbose 
Pro Inside global      Inside local       Outside local      Outside global
--- 7.7.7.0            10.1.1.0           ---                ---
    create 00:07:51, use 00:07:04, 
    flags: 
static, network, use_count: 0, entry-id: 6, lc_entries: 0, Entry type : 3, flags: 81
```

After LH5 initiates a ping to RH7, the router shows **individual host and protocol-specific entries**, in addition to the network mapping:

```text
Local_Router# show ip nat translations
Pro  Inside global      Inside local       Outside local      Outside global
icmp 7.7.7.5:1          10.1.1.5:1         3.3.3.7:1          3.3.3.7:1
---  7.7.7.5            10.1.1.5           ---                ---
---  7.7.7.0            10.1.1.0           ---                ---
```

| Entry | Meaning |
|---|---|
| Network entry: `7.7.7.0` to `10.1.1.0` | Persistent, configured static prefix mapping |
| Host entry: `7.7.7.5` to `10.1.1.5` | Individual host binding derived from the network mapping |
| ICMP entry | The actual flow using that host binding |

The fact that an individual host row appears as traffic starts **does not mean the address mapping is dynamically chosen**. The translated IP address is always determined by the static prefix relationship.

The network-wide rule remains configured while host/flow entries can age independently. The `ip nat translation timeout` setting can affect those individual address bindings, subject to active child entries. Do not confuse the **permanent rule** with the **lifetime of its derived entries**.

For more on parent/child entries and timers, see [NAT Translation Tables and Verification](../verification.md).

---

## Packet Flow: Inside to Outside

LH5 sends an ICMP Echo Request to RH7 (`3.3.3.7`).

The original packet is:

```text
Source:      10.1.1.5
Destination: 3.3.3.7
Protocol:    ICMP
```

Local_Router processes it as follows:

1. The packet enters `Ethernet0/0`, which is configured as `ip nat inside`.
2. Local_Router performs a routing lookup for `3.3.3.7`. Its default route points out `Ethernet0/1` toward ISP1.
3. The static network mapping matches the source address `10.1.1.5`.
4. NAT preserves the host value `.5`, replacing the network prefix `10.1.1.0/24` with `7.7.7.0/24`.
5. Local_Router forwards the packet to ISP1 with source address `7.7.7.5`.

| Field | Before NAT | After NAT |
|---|---|---|
| Source IP | `10.1.1.5` | `7.7.7.5` |
| Destination IP | `3.3.3.7` | `3.3.3.7` |
| Protocol | ICMP | ICMP |

```text
LH5                      Local_Router                    RH7
10.1.1.5                     NAT                       3.3.3.7
   |                           |                           |
   |  10.1.1.5 -> 3.3.3.7      |                           |
   +-------------------------->|                           |
   |                           |  7.7.7.5 -> 3.3.3.7       |
   |                           +-------------------------->|
```

The destination address is not modified. This is **inside source NAT**, so the relevant inside address is changed on the outbound packet.

### Return Traffic: Outside to Inside

RH7 sends its Echo Reply to `7.7.7.5`:

```text
Source:      3.3.3.7
Destination: 7.7.7.5
```

When the reply reaches Local_Router's NAT outside interface:

1. NAT recognizes `7.7.7.5` as the inside global representation of `10.1.1.5`.
2. NAT changes the **destination** IP address to `10.1.1.5`.
3. The router forwards the packet through the inside interface to LH5.

| Field | Before NAT | After NAT |
|---|---|---|
| Source IP | `3.3.3.7` | `3.3.3.7` |
| Destination IP | `7.7.7.5` | `10.1.1.5` |

**Key point:** The same static mapping is used in both directions. Inside-to-outside traffic has its *source* translated; outside-to-inside traffic has its *destination* translated.

---

## Why Outside-Initiated Traffic Can Fail: ARP

In this lab, the inside global prefix `7.7.7.0/24` overlaps ISP1's directly connected outside link.

If RH7 initiates a ping to `7.7.7.5`, ISP1 decides that the destination is **on-link** and must learn a MAC address for `7.7.7.5`:

```text
RH7 -> ISP1
         |
         | Destination 7.7.7.5 is on 7.7.7.0/24
         v
     ARP for 7.7.7.5
         |
         v
  Local_Router must answer
  or ISP1 needs another
  usable Layer 2 resolution
```

For a **single-host static NAT** statement, IOS XE creates an **IP alias** for an on-link inside global address, allowing the NAT router to answer ARP for that address.

For a **static network mapping**, however, IOS XE create the individual host entries and associated aliases **when matching traffic is processed**, rather than installing an ARP alias for every possible host immediately.

An initial outside-initiated ping to `7.7.7.5` fails, while a host-specific ARP entry on ISP1 enables the packet to reach Local_Router and be translated. Once a host generates matching outbound traffic, a corresponding alias and ARP resolution can appear.

> It does **not** fail because Local_Router hasn't created the dynamic NAT mapping yet.
> It fails because Local_Router doesn't have an IP alias for `7.7.7.5` yet, so ISP1's ARP Request for `7.7.7.5` fails.

### Verify ARP and NAT Separately

On ISP1:

```text
ISP1# show ip arp 7.7.7.5
```

On Local_Router:

```text
Local_Router# show ip aliases
Local_Router# show ip nat translations
Local_Router# show ip nat translations verbose
```

These commands answer different questions:

- **ARP:** Can ISP1 encapsulate a packet for `7.7.7.5` in an Ethernet frame toward Local_Router?
- **Aliases:** Does Local_Router currently claim the relevant inside global address on its attached network?
- **NAT entries:** Is there a network mapping and, if applicable, a derived host/flow entry?

An ARP failure can prevent the packet from ever reaching NAT. An apparently correct NAT table does not prove Layer 2 reachability.

### Lab Workaround: A Static ARP Entry

To isolate the ARP issue in a controlled lab, you can temporarily add an entry on ISP1 pointing `7.7.7.5` at the **actual MAC address of Local_Router's outside interface**:

```text
ISP1(config)# arp 7.7.7.5 <Local_Router-E0/1-MAC> arpa
```

This is a *diagnostic exercise*, not a scalable production design. An entry for one address does not solve reachability for every address in the mapped subnet.

---

## Recommended Alternative: Route a Separate Translated Prefix

A cleaner approach is to use an inside global prefix that **does not overlap the NAT router's outside link**.

Keep the existing lab topology, but map the inside network to `7.7.8.0/24` instead:

```text
Inside LAN               Local_Router              ISP1
10.1.1.0/24              7.7.7.2/24               7.7.7.1/24
    |                         |                        |
    +-------------------------+------------------------+
                                   |
                         Translated prefix:
                             7.7.8.0/24
                         (routed via 7.7.7.2)
```

Remove the previous mapping and configure the new one:

```text
Local_Router(config)# no ip nat inside source static network 10.1.1.0 7.7.7.0 /24
Local_Router(config)# ip nat inside source static network 10.1.1.0 7.7.8.0 /24
```

On ISP1, install a route toward the translated prefix:

```text
ISP1(config)# ip route 7.7.8.0 255.255.255.0 7.7.7.2
```

Now, when RH7 sends a packet to `7.7.8.5`:

1. RH7 forwards the packet to ISP1.
2. ISP1 routes `7.7.8.5` toward next hop `7.7.7.2`.
3. ISP1 ARPs for **`7.7.7.2`**, not `7.7.8.5`.
4. Local_Router receives the packet and can translate `7.7.8.5` to `10.1.1.5`.

This avoids the need to install per-host ARP aliases on the outside link. The packet must still be routable to Local_Router, and the NAT implementation must recognize the mapped destination, but the on-link ARP dependency is gone.

**CCIE distinction:** An **on-link** translated address requires appropriate Layer 2 resolution; an **off-link** translated address requires appropriate Layer 3 routing toward the NAT router. NAT configuration alone does not guarantee either.

---

## Static Network Mapping vs. Other NAT Mechanisms

| Feature | Static network mapping | Individual static NAT | Dynamic NAT pool | Dynamic PAT |
|---|---|---|---|---|
| Example command | `static network` | `static <local> <global>` | `list ... pool` | `... overload` |
| Address relationship | Fixed host-bit-preserving mapping | Fixed configured host mapping | Allocated from available pool addresses | Multiple hosts can share a global IP |
| Port translation needed? | No | No | No | Generally yes for TCP/UDP |
| Requires NAT ACL? | No | No | Yes, or a supported route map | Yes, or a supported route map |
| Fixed translated IP for a given host? | Yes | Yes | Not inherently | Not inherently |
| Typical use | Map an entire subnet one-to-one | Map a few specific hosts | Temporary one-to-one assignments | IPv4 address conservation |

Static network NAT is particularly useful when every inside host needs a **predictable translated address**. It is not an efficient way to conserve public IPv4 addresses because each host still needs its own distinct inside global address.

The key difference from an equivalent dynamic pool with `type match-host` is **how the mapping is established and maintained**. A static network rule defines a permanent prefix relationship. A dynamic pool still allocates and ages its host bindings according to dynamic NAT behavior.

See [Dynamic Inside Source NAT and Address Pools](../dynamic/inside-source.md) and [NAT Pool Allocation and Host Matching](../dynamic/pool-allocation.md).

---

## Troubleshooting and Verification

When a static network NAT lab does not work, verify the following in order.

### 1. Check the NAT Configuration

```text
Local_Router# show running-config | include ip nat
Local_Router# show ip nat statistics
```

Confirm that:

- The intended interfaces are `ip nat inside` and `ip nat outside`.
- The local/global **network addresses are in the correct order**.
- The configured mask covers the intended hosts.
- No previous static mapping or other NAT rule creates an address conflict.

### 2. Check Routes and Interface Roles

```text
Local_Router# show ip route 3.3.3.7
Local_Router# show ip route 10.1.1.5
ISP1# show ip route 7.7.7.5
```

If using the routed-prefix alternative, check `7.7.8.5` on ISP1 instead.

The inside host needs a default route, Local_Router needs a route toward the outside, and ISP1 needs a valid path back to the inside global address or prefix.

### 3. Examine Network and Host Entries

```text
Local_Router# show ip nat translations
Local_Router# show ip nat translations verbose
Local_Router# show ip nat statistics
```

A network-wide static entry may exist even before any host traffic appears. Generate traffic from LH5 and LH6, then look for the corresponding host and protocol entries.

### 4. Check ARP When Using On-Link Global Addresses

```text
ISP1# show ip arp 7.7.7.5
Local_Router# show ip aliases
```

If ISP1 cannot resolve `7.7.7.5`, the packet never arrives at Local_Router, regardless of the configured NAT rule.

### 5. Verify Both Directions

Test from the inside:

```text
Local_Host_5# ping 3.3.3.7
Local_Host_6# ping 3.3.3.8
```

Then test from the outside to the **inside global** addresses:

```text
Remote_Host_7# ping 7.7.7.5
Remote_Host_8# ping 7.7.7.6
```

For the routed-prefix alternative, use `7.7.8.5` and `7.7.8.6`.

### Common Failure Causes

| Symptom | Likely cause |
|---|---|
| Inside host cannot reach RH7 | Missing default route, incorrect NAT interface role, wrong mapping, or other forwarding problem |
| Inside traffic works but remote host cannot initiate traffic | Missing/expired on-link ARP resolution, missing route toward global prefix, or an ACL blocking the traffic |
| Host maps to an unexpected global IP | Incorrect mask, conflicting NAT rule, or confusion with a dynamic pool |
| Only the network entry appears | No matching host traffic yet, or platform-specific translation-table behavior |
| Outside router ARPs for every translated host | Translated prefix is directly connected/on-link from its perspective |
| Translation overlaps a router interface address | Address plan conflicts with occupied outside IP addresses |

---

## Key Takeaways

- `ip nat inside source static network` creates a **fixed, one-to-one network mapping** that preserves host bits.
- **Inside local comes first**, followed by **inside global** and the mask.
- It does **not** require an ACL, NAT pool, or PAT overloading.
- A network-wide static entry can exist before individual host/flow entries are created. **Static mapping** does not necessarily mean *all per-host entries are permanently installed*.
- For an **on-link translated network**, ARP and NAT alias behavior can affect initial outside-to-inside connectivity. Verify it on the actual IOS XE platform.
- For an **off-link translated network**, route the entire global prefix toward the NAT router so upstream devices ARP only for the NAT router's next-hop address.
- Keep translated addresses distinct from existing router/interface addresses. The article's `7.7.7.0/24` example is useful for observing ARP behavior, not ideal production address planning.

## References

- Cisco, [IP Addressing Configuration Guide, Cisco IOS XE 17.x: Configuring NAT for IP Address Conservation](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-addressing/b-ip-addressing/m_iadnat-addr-consv-xe.html)
- Cisco, [IP Addressing Services Command Reference: `ip nat inside source`](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9400/software/release/26-x/command_reference/b_26x_9400_cr/ip_addressing__services_commands.html)
- Jedadiah Casey, [Understanding and Configuring Network Address Translation (NAT) for Cisco CCNA, CCNP and CCIE, Case 6](https://wax-trax.medium.com/understanding-and-configuring-network-address-translation-nat-for-cisco-ccna-ccnp-and-ccie-7fef92fa654c)
