# Static NAT Options and ARP Behavior

Cisco IOS XE supports several optional keywords that modify the behavior of static NAT and PAT mappings.

Some options affect how translations are identified, while others affect ARP, routing, application-layer translation, or the creation of flow-specific NAT entries.

| Keyword | Purpose |
|---|---|
| `extendable` | Allows multiple static mappings involving the same local or global address |
| `no-alias` | Suppresses automatic creation of an IP alias for the translated address |
| `no-payload` | Disables translation of embedded IP addresses and ports in application payloads |
| `add-route` | Automatically installs a route to an outside local address |
| `stateless` | Prevents creation of flow-specific NAT sessions for a static mapping |
| `route-map` | Conditionally applies a static mapping based on a route map |
| `reversible` | Supports reverse-direction initiation for applicable route-map-based mappings |
| `vrf` | Associates the NAT mapping with a VRF |
| `match-in-vrf` | Supports NAT between interfaces in the same VRF |
| `redundancy` | Associates the mapping with a NAT redundancy configuration |
| `mapping-id` | Identifies a mapping for NAT high availability |
| `forced` | Supports forced removal of applicable static mappings with active translations |

Not all keywords are available with every NAT command, platform, or IOS XE release.

This page focuses primarily on `extendable`, `no-alias`, `no-payload`, `add-route`, and `stateless`.

---

## Lab Topology

The examples on this page use the same topology as the previous static NAT lessons.

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
| LH5 | Inside host | `10.1.1.5/24` |
| LH6 | Inside host | `10.1.1.6/24` |
| Local_Router | E0/0, NAT inside | `10.1.1.1/24` |
| Local_Router | E0/1, NAT outside | `7.7.7.2/24` |
| ISP1 | Toward Local_Router | `7.7.7.1/24` |
| ISP1 | Toward remote hosts | `3.3.3.1/24` |
| RH7 | Outside host | `3.3.3.7/24` |
| RH8 | Outside host | `3.3.3.8/24` |

The examples are independent unless otherwise stated. Remove previous NAT mappings and associated dynamic translations when changing between exercises.

---

## The `extendable` Keyword

The `extendable` keyword allows multiple static NAT mappings to involve the same local and/or global IP address.

For example:

```text
R1(config)# ip nat inside source static 10.1.1.5 7.7.7.50 extendable
R1(config)# ip nat inside source static 10.1.1.5 7.7.7.60 extendable
```

Both mappings use the same inside local address:

```text
                     +--> 7.7.7.50
                     |
10.1.1.5 ------------+
                     |
                     +--> 7.7.7.60
```

Without `extendable`, overlapping address-only static mappings can be rejected as conflicting translations.

With `extendable`, supported overlapping mappings can coexist in the configuration.

However, the keyword does **not** itself determine which mapping should be used when multiple mappings match the same packet.

A route map or another supported distinguishing mechanism may be required.

### Why Extendable Static NAT Is Useful

Consider a router connected to two ISPs.

```text
                            ISP1
                         7.7.7.0/24
                              |
                              |
Inside host ------------- NAT router
10.1.1.5                      |
                              |
                             ISP2
                         8.8.8.0/24
```

The administrator might want the same inside host to use different inside global addresses depending on the selected ISP.

| Egress path | Inside global |
|---|---|
| ISP1 | `7.7.7.50` |
| ISP2 | `8.8.8.50` |

The `extendable` keyword allows the mappings to coexist.

Route maps can be used to distinguish when each mapping should be applied.

See [Route-Map-Based NAT and Conditional Translation](../policy-based/route-maps.md) and [Multi-ISP NAT and Failover](../policy-based/multi-egress.md).

---

## `extendable` and Static PAT

Static PAT mappings already include additional identifying information:

- Transport protocol.
- Local IP address.
- Local port.
- Global IP address.
- Global port.

Therefore, multiple static PAT mappings can share an inside local or inside global IP address while remaining distinguishable.

For example:

```text
R1(config)# ip nat inside source static tcp 10.1.1.5 443 7.7.7.50 8443 extendable
R1(config)# ip nat inside source static tcp 10.1.1.6 443 7.7.7.50 9443 extendable
```

The mappings share the same inside global IP address, but use different global TCP ports.

| Protocol | Inside local | Inside global |
|---|---|---|
| TCP | `10.1.1.5:443` | `7.7.7.50:8443` |
| TCP | `10.1.1.6:443` | `7.7.7.50:9443` |

Similarly, multiple static PAT mappings can share the same inside local IP address:

```text
R1(config)# ip nat inside source static tcp 10.1.1.5 22 7.7.7.50 2222 extendable
R1(config)# ip nat inside source static tcp 10.1.1.5 443 7.7.7.60 8443 extendable
```

The local TCP port distinguishes the mappings.

### Automatically Added `extendable` Keyword

On modern IOS XE, the router automatically appends `extendable` when a static PAT mapping is configured.

For example, enter:

```text
R1(config)# ip nat inside source static tcp 10.1.1.5 440 7.7.7.50 9440
```

Then examine the running configuration:

```text
R1(config)# do show running-config | include nat
 ip nat inside
 ip nat outside
ip nat inside source static tcp 10.1.1.5 440 7.7.7.50 9440 extendable
```

Although `extendable` was omitted when entering the command, IOS XE added it automatically.

Therefore, on this implementation, there is no need to explicitly configure `extendable` for ordinary static PAT mappings.

### Limitations of `extendable`

The keyword does not make every combination of mappings unambiguous.

For example:

```text
ip nat inside source static tcp 10.1.1.5 443 7.7.7.50 8443 extendable
ip nat inside source static tcp 10.1.1.6 443 7.7.7.50 8443 extendable
```

Both mappings attempt to use the same global TCP endpoint:

```text
7.7.7.50:8443
```

An incoming TCP connection addressed to that endpoint cannot be uniquely associated with one of the two inside hosts based on the global endpoint alone.

The `extendable` keyword is not a load-balancing mechanism and does not eliminate the need for unambiguous translation selection.

### Extended vs. Extendable

These terms are related but are not synonymous.

| Term | Meaning |
|---|---|
| Extended translation | A translation containing additional identifying information, such as protocol, ports, and/or outside addresses |
| Extendable mapping | A static mapping configured to permit supported overlapping translation relationships |

An extended translation can exist without the `extendable` configuration keyword.

For example, dynamically created PAT entries are extended translations.

See [NAT Translation Tables and Verification](../verification.md).

---

## NAT IP Aliases

An **IP alias** allows a router to recognize an additional IP address without configuring that address as an ordinary interface IP address.

In traditional Cisco NAT, IP aliases can be automatically created to allow packets addressed to translated IP addresses to reach the NAT router.

This is particularly important when a translated address belongs to a directly connected subnet.

Consider the following static inside source NAT mapping:

```text
R1(config)# ip nat inside source static 10.1.1.5 7.7.7.50
```

R1's outside interface uses `7.7.7.2/24`.

ISP1's adjacent interface uses `7.7.7.1/24`.

The inside global address, `7.7.7.50`, belongs to the same subnet.

```text
                   7.7.7.0/24

           R1 -------------------- ISP1
        7.7.7.2                   7.7.7.1
           |
           | NAT mapping:
           | 7.7.7.50 <-> 10.1.1.5
           |
          LH5
       10.1.1.5
```

Although `7.7.7.50` is not configured as an ordinary interface IP address on R1, R1 needs to receive packets addressed to it.

### Why an IP Alias Is Needed

Suppose RH7 sends a packet to `7.7.7.50`.

ISP1 performs a routing lookup and finds:

```text
C 7.7.7.0/24 is directly connected
```

Because `7.7.7.50` is on-link, ISP1 sends an ARP Request for that address.

```text
Who has 7.7.7.50?
```

Without a mechanism to respond to this ARP Request, ISP1 cannot deliver the packet to R1.

When the NAT mapping creates an IP alias for `7.7.7.50`, R1 can respond to the ARP Request using its own MAC address.

```text
ISP1                                  R1
  |                                    |
  | ARP Request: Who has 7.7.7.50?     |
  +----------------------------------->|
  |                                    |
  | ARP Reply: 7.7.7.50 is at R1 MAC   |
  |<-----------------------------------+
  |                                    |
  | IP destination: 7.7.7.50           |
  | Ethernet destination: R1 MAC       |
  +----------------------------------->|
                                       |
                                       | NAT:
                                       | 7.7.7.50 -> 10.1.1.5
                                       |
                                       v
                                      LH5
```

R1 receives the packet and applies the static NAT mapping in reverse.

### Verify IP Aliases

```text
R1# show ip aliases
```

Examine whether the translated address appears as an alias.

The exact output format and alias type depend on the IOS XE implementation.

On ISP1:

```text
ISP1# show ip arp 7.7.7.50
```

ISP1 should associate `7.7.7.50` with the MAC address of R1's outside interface.

### IP Aliases vs. Interface Addresses

An IP alias is not the same as a primary or secondary interface address.

For example, configuring:

```text
ip nat inside source static 10.1.1.5 7.7.7.50
```

does not configure the following interface command:

```text
ip address 7.7.7.50 255.255.255.0 secondary
```

The NAT-generated alias is a mechanism associated with address translation.

Do not expect the translated address to appear as an ordinary interface address in `show ip interface brief`.

---

## When Are IP Aliases Created?

For ordinary single-host static mappings, the relevant translated address is:

- **Inside global** for inside source NAT.
- **Outside local** for outside source NAT.

An IP alias is created when the translated address is on-link in the corresponding NAT domain.

| NAT type | Address for which an alias can be created | Relevant connected subnet |
|---|---|---|
| Inside source NAT | Inside global | NAT outside subnet |
| Outside source NAT | Outside local | NAT inside subnet |

The purpose is to allow the translated address to be resolved through ARP when neighboring devices consider it directly connected.

### Inside Source NAT Example

```text
R1(config)# ip nat inside source static 10.1.1.5 7.7.7.50
```

Because `7.7.7.50` belongs to the directly connected outside subnet, an IP alias can be created.

ISP1 can then resolve the translated address to R1's MAC address.

### Outside Source NAT Example

```text
R1(config)# ip nat outside source static 3.3.3.7 10.1.1.7 add-route
```

The outside local address, `10.1.1.7`, belongs to the directly connected inside subnet.

LH5 considers `10.1.1.7` directly connected and sends an ARP Request for it.

An IP alias can allow R1 to respond to the ARP Request on behalf of RH7.

The `add-route` keyword serves a separate purpose: it installs an appropriate route toward the outside local address so that R1 selects the outside forwarding path.

Both Layer 2 delivery and Layer 3 routing must work.

---

## On-Link vs. Off-Link Translated Addresses

A NAT-generated IP alias is not always necessary.

### On-Link Inside Global Address

Consider:

```text
R1(config)# ip nat inside source static 10.1.1.5 7.7.7.50
```

The inside global address belongs to the outside transit subnet.

ISP1 considers the translated address directly connected and ARPs for:

```text
7.7.7.50
```

R1 must receive packets addressed to that translated address.

An IP alias provides the necessary ARP behavior.

### Off-Link Inside Global Address

Now consider:

```text
R1(config)# ip nat inside source static 10.1.1.5 203.0.113.50
```

The inside global address does not belong to the outside transit subnet.

Configure a route on ISP1:

```text
ISP1(config)# ip route 203.0.113.50 255.255.255.255 7.7.7.2
```

When ISP1 sends a packet to `203.0.113.50`, it performs a routing lookup and selects `7.7.7.2` as the next hop.

ISP1 therefore ARPs for:

```text
7.7.7.2
```

Not:

```text
203.0.113.50
```

Because the next hop is R1's actual interface address, there is no need for an alias for `203.0.113.50` on the transit link.

An off-link translated address does not result in an IP alias being created.

### Comparison

| Characteristic | On-link global address | Off-link global address |
|---|---|---|
| Inside global | `7.7.7.50` | `203.0.113.50` |
| ISP1 route | Connected route | Route via R1 |
| Address resolved through ARP | `7.7.7.50` | `7.7.7.2` |
| NAT alias needed | Normally yes | No |
| Additional route on ISP1 | Not normally | Yes, unless already learned |

A separate routed NAT prefix avoids the need for neighboring routers to maintain ARP entries for individual translated addresses.

This is often a cleaner design when many addresses are being translated.

---

## The `no-alias` Keyword

The `no-alias` keyword suppresses automatic creation of an IP alias associated with a NAT mapping.

For example:

```text
R1(config)# ip nat inside source static 10.1.1.5 7.7.7.50 no-alias
```

The NAT mapping still exists.

However, NAT does not automatically create the IP alias for `7.7.7.50`.

### What `no-alias` Does Not Do

The `no-alias` keyword does not:

- Disable the NAT mapping.
- Prevent source or destination translation.
- Prevent creation of all dynamic extended translations.
- Disable ARP on the interface.
- Disable ordinary proxy ARP globally.
- Automatically create a route to the translated address.

It only suppresses the automatic alias behavior associated with the mapping.

### Lab Exercise: Without `no-alias`

Configure:

```text
R1(config)# ip nat inside source static 10.1.1.5 7.7.7.50
```

Verify the alias:

```text
R1# show ip aliases
```

From RH7:

```text
RH7# ping 7.7.7.50
```

On ISP1:

```text
ISP1# show ip arp 7.7.7.50
```

The expected behavior is:

1. ISP1 sends an ARP Request for `7.7.7.50`.
2. R1 responds using its outside interface MAC address.
3. ISP1 forwards the packet to R1.
4. R1 translates the destination to `10.1.1.5`.
5. LH5 receives the packet.

### Lab Exercise: With `no-alias`

Remove the previous mapping:

```text
R1(config)# no ip nat inside source static 10.1.1.5 7.7.7.50
```

Configure:

```text
R1(config)# ip nat inside source static 10.1.1.5 7.7.7.50 no-alias
```

Verify the NAT table:

```text
R1# show ip nat translations
```

Representative output:

```text
Pro  Inside global      Inside local       Outside local      Outside global
---  7.7.7.50           10.1.1.5           ---                ---
```

The static NAT entry still exists.

However, the NAT-generated alias should not be created:

```text
R1# show ip aliases
```

Before testing, clear the existing ARP entry for `7.7.7.50` on ISP1.

For example, where supported:

```text
ISP1# clear ip arp 7.7.7.50
```

Then initiate another ping from RH7:

```text
RH7# ping 7.7.7.50
```

If no other ARP-response mechanism is available, ISP1 cannot resolve the translated address.

The packet never reaches R1, so NAT cannot process it.

**The failure is caused by Layer 2 reachability, not by the absence of a NAT mapping.**

---

## Making `no-alias` Work with On-Link Addresses

Suppressing NAT alias creation does not necessarily make an on-link translated address unreachable.

Other mechanisms can deliver the packet to the NAT router.

### Method 1: Configure a Static Route on ISP1

Configure a more-specific route for the translated address:

```text
ISP1(config)# ip route 7.7.7.50 255.255.255.255 7.7.7.2
```

ISP1 now has:

```text
C 7.7.7.0/24 is directly connected
S 7.7.7.50/32 [1/0] via 7.7.7.2
```

The static `/32` route takes precedence over the connected `/24` route due to longest prefix matching.

Instead of directly resolving `7.7.7.50`, ISP1 forwards the packet through the next hop `7.7.7.2`.

Therefore, the packet reaches R1 without requiring R1 to create an IP alias for `7.7.7.50`.

The NAT configuration can remain:

```text
R1(config)# ip nat inside source static 10.1.1.5 7.7.7.50 no-alias
```

### Method 2: Configure a Static ARP Entry on ISP1

A static ARP entry can associate the translated address with R1's actual outside interface MAC address.

The syntax is:

```text
ISP1(config)# arp <translated-ip> <nat-router-mac> arpa
```

For example, assuming R1's outside interface MAC address is `aaaa.bbbb.cccc`:

```text
ISP1(config)# arp 7.7.7.50 aaaa.bbbb.cccc arpa
```

ISP1 can now encapsulate packets addressed to `7.7.7.50` in Ethernet frames destined for R1.

This is useful for isolating ARP-related issues in a lab.

However, manually maintaining static ARP entries for many translated addresses is generally less convenient than routing a translated prefix toward the NAT router.

### Comparison

| Method | How packets reach the NAT router |
|---|---|
| Default NAT alias behavior | NAT router answers ARP for the translated address |
| More-specific static route | Upstream router resolves the NAT router's actual next-hop IP |
| Static ARP entry | Upstream router already knows the required MAC address |

All three methods address packet delivery to the NAT router.

The NAT mapping itself still performs the address translation.

---

## NAT Aliases vs. Ordinary Proxy ARP

NAT-generated IP aliases and ordinary proxy ARP can produce a similar visible result:

A router responds to an ARP Request for an address that is not configured as its ordinary interface IP address.

However, the mechanisms are different.

### Ordinary Proxy ARP

With ordinary proxy ARP, a router can answer an ARP Request on behalf of a destination that it knows how to reach through another interface.

Proxy ARP is controlled through interface configuration:

```text
R1(config-if)# ip proxy-arp
```

Or:

```text
R1(config-if)# no ip proxy-arp
```

### NAT-Generated Alias

With NAT, the router creates an alias associated with a translated address.

The alias is controlled through the NAT mapping:

```text
R1(config)# ip nat inside source static 10.1.1.5 7.7.7.50 no-alias
```

### Comparison

| Feature | Ordinary proxy ARP | NAT-generated alias |
|---|---|---|
| Associated with | Routing and interface behavior | NAT mapping |
| Primary purpose | Enable Layer 2 delivery for destinations reachable through the router | Enable Layer 2 delivery for translated addresses |
| Configuration control | `ip proxy-arp` / `no ip proxy-arp` | `no-alias` on the NAT mapping |
| NAT mapping required | No | Yes |

Do not assume that disabling ordinary proxy ARP necessarily suppresses NAT-generated alias responses.

Similarly, `no-alias` does not disable ordinary proxy ARP.

Other ARP-response mechanisms may still allow the packet to reach the NAT router.

Verify the behavior using `show ip aliases`, the neighboring device's ARP table, and packet captures.

---

## ARP Behavior with Static Network NAT

A single-host static NAT mapping and a static network-to-network NAT mapping behave differently with respect to alias creation.

### Single-Host Static Mapping

```text
R1(config)# ip nat inside source static 10.1.1.5 7.7.7.5
```

An IP alias for the on-link inside global address is created when the static mapping is configured.

As a result, ISP1 can resolve `7.7.7.5` through ARP before LH5 initiates communication.

### Static Network Mapping

```text
R1(config)# ip nat inside source static network 10.1.1.0 7.7.7.0 /24
```

The configured mapping covers the entire network.

However, IOS XE does not immediately create an IP alias for every individual translated host address.

Instead, individual host translations and associated aliases can be created on demand when matching traffic is processed.

Consider LH5:

```text
Inside local:  10.1.1.5
Inside global: 7.7.7.5
```

Before LH5 has initiated any traffic, R1 may not have an alias for `7.7.7.5`.

If RH7 initiates a ping to `7.7.7.5`, ISP1 sends an ARP Request.

Without an alias or another ARP-response mechanism, ISP1 cannot deliver the packet to R1.

```text
RH7 initiates ping to 7.7.7.5
              |
              v
ISP1 sends ARP Request for 7.7.7.5
              |
              v
R1 has no corresponding IP alias yet
              |
              v
ARP resolution fails
              |
              v
Packet never reaches NAT router
```

### Inside-Initiated Traffic

If LH5 initiates traffic toward RH7:

```text
LH5# ping 3.3.3.7
```

R1 processes the packet using the static network mapping.

The corresponding host translation and IP alias can then be created.

ISP1 can subsequently resolve `7.7.7.5` through ARP.

### Important Distinction

The problem is not that the static network mapping only supports inside-initiated traffic.

The static network mapping is bidirectional.

The problem is that outside-initiated traffic may not reach the NAT router due to missing ARP resolution.

If a static ARP entry or an appropriate route delivers the initial outside packet to R1, the static network mapping can translate it without requiring previous inside-initiated traffic.

### Comparison

| Mapping type | Static mapping installed | On-link host aliases |
|---|---|---|
| Single-host static NAT | Immediately | Normally created immediately |
| Static PAT | Immediately | Normally created immediately for an applicable translated IP |
| Static network NAT | Immediately | Can be created individually on demand |

These observations reflect the tested IOS XE implementation.

See [Static Network-to-Network NAT](network-mapping.md).

---

## Be Careful with Cached ARP Entries

When testing NAT alias behavior, an existing ARP entry can conceal the effect of changing a NAT mapping.

For example:

1. Configure an ordinary single-host static NAT mapping.
2. Generate traffic so ISP1 learns an ARP entry for the inside global address.
3. Remove the single-host mapping.
4. Configure a static network mapping or a mapping using `no-alias`.
5. Initiate traffic from the outside again.

The outside-initiated traffic may succeed because ISP1 still has a valid ARP entry from the previous configuration.

This does not prove that the new NAT mapping creates an IP alias.

The upstream router may simply be using cached Layer 2 information.

### Recommended Testing Procedure

Before testing alias creation:

1. Remove conflicting NAT mappings.
2. Clear relevant dynamic NAT translations if necessary.
3. Verify the configured static NAT entry.
4. Check `show ip aliases` on the NAT router.
5. Clear the relevant ARP entry on the neighboring device.
6. Initiate traffic from the intended side.
7. Observe ARP resolution, NAT translation creation, and packet forwarding.

Useful commands include:

```text
R1# show ip aliases
R1# show ip nat translations
R1# show ip nat translations verbose
ISP1# show ip arp
```

Where supported:

```text
ISP1# clear ip arp 7.7.7.5
```

In an isolated lab, it is also possible to clear dynamic NAT translations:

```text
R1# clear ip nat translation *
```

Be aware that clearing NAT translations or ARP entries can disrupt active connections.

**A successful test is not necessarily testing the intended feature if state from a previous test is still present.**

---

## The `no-payload` Keyword

NAT primarily translates information in packet headers.

For example:

- IPv4 source and destination addresses.
- TCP and UDP source and destination ports.
- Protocol-specific identifiers, where applicable.

However, certain application protocols also include IP addresses or port numbers inside their application-layer payloads.

Simply translating the packet headers may not be sufficient for these protocols to function correctly through NAT.

Cisco NAT supports **Application Layer Gateways (ALGs)** for certain protocols. These can inspect and modify embedded addressing information and, where necessary, support related connections.

The `no-payload` keyword prevents NAT from translating embedded addresses or ports in the payload for the applicable mapping.

### Configuration

```text
R1(config)# ip nat inside source static 10.1.1.5 7.7.7.50 no-payload
```

The IP address translation still occurs.

For example:

```text
Source IP before NAT: 10.1.1.5
Source IP after NAT:   7.7.7.50
```

However, NAT does not rewrite embedded addressing information in the application payload for this mapping.

### Conceptual Example: FTP

Traditional FTP can carry IP address and port information in its control connection.

For example, an active-mode FTP client might send:

```text
PORT 10,1,1,5,195,80
```

This command advertises:

```text
IP address: 10.1.1.5
TCP port:   50000
```

The port is calculated as:

```text
195 * 256 + 80 = 50000
```

Suppose the client is translated using:

```text
ip nat inside source static 10.1.1.5 7.7.7.50
```

If the relevant FTP ALG is active, NAT can modify the embedded addressing information to reflect the translated endpoint.

For example:

```text
PORT 7,7,7,50,195,80
```

The exact resulting port depends on the translation and ALG behavior.

Without this modification, the outside FTP server may attempt to establish a data connection to the client's private address.

### FTP with `no-payload`

Now configure:

```text
ip nat inside source static 10.1.1.5 7.7.7.50 no-payload
```

The packet's IPv4 source address is still translated to `7.7.7.50`.

However, the embedded FTP addressing information is not translated by NAT for that mapping.

```text
IPv4 header source: 7.7.7.50

FTP payload:
PORT 10,1,1,5,195,80
```

This may cause the application to fail if it depends on NAT rewriting its embedded addressing information.

### Normal NAT vs. `no-payload`

| Field | Normal NAT | NAT with `no-payload` |
|---|---|---|
| IPv4 header address | Translated | Translated |
| TCP/UDP header port | Translated if required | Translated if required |
| Supported embedded payload address | Can be translated by an ALG | Not translated |
| Supported embedded payload port | Can be translated by an ALG | Not translated |

The `no-payload` keyword does **not** disable static PAT.

For example:

```text
R1(config)# ip nat inside source static tcp 10.1.1.5 443 7.7.7.50 8443 no-payload
```

The TCP header port is still translated between `443` and `8443`.

The keyword concerns embedded information in the application payload, not the TCP or UDP header itself.

### Why Use `no-payload`?

Possible reasons include:

- The application already handles NAT traversal.
- Embedded addressing information must remain unchanged.
- An ALG's modifications interfere with the application.
- The translated traffic should undergo header translation without supported payload rewriting.

Do not enable `no-payload` indiscriminately.

Some protocols depend on application-layer NAT handling to function properly.

---

## The `add-route` Keyword

The `add-route` keyword is used with **outside source NAT**, including supported static and dynamic configurations.

Its purpose is to automatically install a route associated with the outside local address.

Consider:

```text
R1(config)# ip nat outside source static 3.3.3.7 172.16.1.7 add-route
```

The mapping is:

```text
Outside global: 3.3.3.7
Outside local:  172.16.1.7
```

For an ordinary single-host static mapping, the generated route is logically equivalent to:

```text
ip route 172.16.1.7 255.255.255.255 3.3.3.7
```

The route uses the outside global address as its next hop.

### Why Is `add-route` Needed?

Recall the traditional IOS NAT order of operations:

| Direction | Order |
|---|---|
| Inside to outside | Routing, then NAT |
| Outside to inside | NAT, then routing |

When LH5 sends a packet to `172.16.1.7`, R1 must perform a routing lookup before translating the destination address.

```text
Original destination: 172.16.1.7
Translated destination: 3.3.3.7
```

The route to `172.16.1.7` must select the appropriate outside forwarding path.

A route to `3.3.3.7` alone does not necessarily satisfy the initial lookup.

### Recursive Route Resolution

Assume the following route is configured:

```text
R1(config)# ip route 3.3.3.0 255.255.255.0 7.7.7.1
```

With `add-route`, the forwarding path can resolve as follows:

```text
172.16.1.7
     |
     | NAT-generated /32 route
     v
3.3.3.7
     |
     | Route to 3.3.3.0/24
     v
7.7.7.1
     |
     | Connected route
     v
Ethernet0/1 (NAT outside)
```

The destination is then translated to `3.3.3.7` as the packet passes through NAT.

### Verification

```text
R1# show ip route 172.16.1.7
R1# show ip route 3.3.3.7
R1# show ip cef 172.16.1.7 detail
```

Confirm that:

- The outside local route exists.
- The outside global next hop can be resolved.
- The resulting forwarding path uses the intended NAT outside interface.

---

## `add-route` with Static Network NAT

The behavior of `add-route` differs between ordinary single-host static outside source NAT and static outside source network NAT.

Consider:

```text
R1(config)# ip nat outside source static network 3.3.3.0 172.16.1.0 /24 add-route
```

Immediately after configuring the mapping:

```text
R1# show ip nat translations
Pro  Inside global      Inside local       Outside local      Outside global
---  ---                ---                172.16.1.0         3.3.3.0
```

The static network mapping exists.

However, on the tested IOS XE image, R1 does not automatically install a route for the entire outside local prefix.

There is no automatically generated route for:

```text
172.16.1.0/24
```

### Host Routes Are Installed on Demand

When RH7 initiates communication toward an inside host, the corresponding individual outside source translation is created.

NAT then installs a host route associated with the outside local address.

The generated route is logically equivalent to:

```text
ip route 172.16.1.7 255.255.255.255 3.3.3.7
```

This route is associated with RH7's individual translation.

If RH8 subsequently generates matching traffic, another host route can be created:

```text
ip route 172.16.1.8 255.255.255.255 3.3.3.8
```

The network mapping does not cause a single aggregate route to be installed.

### The Inside-Initiation Problem

Suppose R1 has:

- A route to `3.3.3.0/24`.
- No default route.
- No route to `172.16.1.0/24`.
- No individual outside-local host routes.

LH5 attempts to initiate communication:

```text
LH5# ping 172.16.1.7
```

R1 performs a routing lookup for `172.16.1.7`.

No matching route exists, so the packet cannot undergo the intended inside-to-outside forwarding process.

```text
Inside host sends packet to outside local address
                       |
                       v
                Routing lookup
                       |
                       v
                No matching route
                       |
                       v
           Intended NAT processing fails
```

The problem is that the individual host route is created on demand, but the inside-initiated packet requires a route before it can trigger the intended NAT translation.

### Recommended Solution

Manually configure a route toward the entire outside local prefix:

```text
R1(config)# ip route 172.16.1.0 255.255.255.0 7.7.7.1
```

This allows inside hosts to initiate traffic toward any mapped outside-local address without requiring prior outside-initiated traffic.

If `add-route` is also configured, any subsequently created `/32` routes take precedence over the manually configured `/24` route.

### Comparison

| Mapping type | Observed `add-route` behavior |
|---|---|
| Static outside source host NAT | Installs a host route when the mapping is configured |
| Static outside source network NAT | Installs individual host routes as the corresponding host translations are created |
| Dynamic outside source NAT | Installs host routes as outside source translations are allocated |

See [Static Outside Source NAT and PAT](outside-source.md).

---

## `no-alias` vs. `add-route`

These keywords affect different parts of packet delivery.

| Keyword | Affects | Purpose |
|---|---|---|
| `no-alias` | ARP and IP alias behavior | Suppresses automatic creation of a NAT alias |
| `add-route` | Routing table | Automatically installs a route associated with an outside local address |

Consider an outside source mapping whose outside local address belongs to the inside subnet:

```text
R1(config)# ip nat outside source static 3.3.3.7 10.1.1.7 add-route
```

LH5 considers `10.1.1.7` directly connected.

Two separate mechanisms are needed:

1. LH5 must deliver the packet to R1 instead of attempting to communicate directly with a nonexistent local host.
2. R1 must select the outside forwarding path instead of forwarding the packet back toward its connected inside network.

The NAT alias can solve the first problem.

The route installed by `add-route` can solve the second problem.

### Combining `no-alias` and `add-route`

On implementations supporting both keywords together:

```text
R1(config)# ip nat outside source static 3.3.3.7 10.1.1.7 add-route no-alias
```

The configuration requests:

- Automatic route installation.
- Suppression of the automatic NAT alias.

The route to `10.1.1.7` may exist on R1, but this alone does not guarantee that LH5 can deliver the packet to R1.

Ordinary proxy ARP or another delivery mechanism may still be needed.

**A valid route on the NAT router does not automatically guarantee Layer 2 reachability from an adjacent host.**

---

## The `stateless` Keyword

Traditional static NAT mappings are persistent, but traffic matching those mappings can create additional dynamic extended translation entries.

These entries maintain flow-specific NAT state.

Cisco introduced the `stateless` keyword for static NAT in IOS XE Bengaluru 17.4.1a.

When a supported static mapping is configured as stateless, matching traffic is translated without creating the usual flow-specific NAT sessions.

### Ordinary Stateful Static NAT

Consider:

```text
R1(config)# ip nat inside source static 10.1.1.5 7.7.7.50
```

Before traffic is generated:

```text
R1# show ip nat translations
Pro  Inside global      Inside local       Outside local      Outside global
---  7.7.7.50           10.1.1.5           ---                ---
```

When LH5 communicates with RH7, an additional extended translation can be created.

For example:

```text
Pro  Inside global      Inside local       Outside local      Outside global
tcp  7.7.7.50:51000     10.1.1.5:51000     3.3.3.7:443        3.3.3.7:443
---  7.7.7.50           10.1.1.5           ---                ---
```

The simple static mapping is persistent.

The additional flow-specific entry is subject to NAT timeouts.

### Stateless Static NAT

Instead, configure:

```text
R1(config)# ip nat inside source static 10.1.1.5 7.7.7.50 stateless
```

The fixed mapping determines how each matching packet should be translated.

A separate NAT session does not need to be created for each matching flow.

Conceptually:

```text
Before traffic:

---  7.7.7.50  10.1.1.5  ---  ---

After traffic:

---  7.7.7.50  10.1.1.5  ---  ---
```

The display is conceptual. The exact translation-table presentation depends on the platform.

The key distinction is the absence of ordinary flow-specific NAT session creation for traffic matching only the stateless mapping.

### Why Use Stateless Static NAT?

Stateless static NAT can be useful when:

- A large number of flows use predictable static translations.
- Per-flow NAT state is unnecessary.
- Reducing NAT session-table resource consumption is desirable.
- A redundant NAT design uses deterministic static mappings.

Because the mapping is fixed, the router can determine the necessary translation directly from the packet and the configured mapping.

### Stateless Static PAT

Stateless operation is also supported with applicable static PAT mappings.

For example:

```text
R1(config)# ip nat inside source static tcp 10.1.1.5 443 7.7.7.50 8443 extendable stateless
```

The mapping remains:

```text
TCP 10.1.1.5:443 <-> 7.7.7.50:8443
```

Outside clients can initiate connections using the inside global endpoint.

The mapping itself is sufficient to determine the required translation without creating an ordinary NAT session for every connection matching only this rule.

### Stateless Network NAT

Cisco also documents stateless operation for static network mappings.

For example:

```text
R1(config)# ip nat inside source static network 10.1.1.0 203.0.113.0 /24 stateless
```

Each host is represented by its corresponding address in the translated network.

```text
10.1.1.5 -> 203.0.113.5
10.1.1.6 -> 203.0.113.6
```

Using a separate routed global prefix avoids dependence on per-host on-link ARP alias creation.

For example, on ISP1:

```text
ISP1(config)# ip route 203.0.113.0 255.255.255.0 7.7.7.2
```

Do not assume that stateless network NAT creates the same on-demand aliases or individual translation entries as ordinary stateful network NAT.

Verify the behavior on the target platform.

### Stateless Outside Source NAT

Outside source static mappings can also support stateless operation:

```text
R1(config)# ip nat outside source static 3.3.3.7 172.16.1.7 stateless
```

The routing requirements remain.

For example:

```text
R1(config)# ip route 172.16.1.7 255.255.255.255 7.7.7.1
```

Stateless operation does not eliminate the need for the NAT router to perform the initial routing lookup for the outside local address.

### Stateful and Stateless Mappings Together

Cisco documents that when a flow matches both a stateless mapping and a stateful mapping, a NAT session can still be created.

For example, consider:

```text
ip nat inside source static 10.1.1.5 7.7.7.50 stateless
ip nat outside source static 3.3.3.7 172.16.1.7
```

Traffic between LH5 and RH7 matches both mappings.

Although the inside source mapping is stateless, the outside source mapping is stateful.

Therefore, the presence of a stateless rule does not guarantee that the complete flow will be processed without NAT session state.

### Stateful vs. Stateless Static NAT

| Feature | Stateful static NAT | Stateless static NAT |
|---|---|---|
| Fixed configured mapping | Yes | Yes |
| Supports static address translation | Yes | Yes |
| Supports applicable static PAT | Yes | Yes |
| Flow-specific NAT session creation | Yes, where applicable | No, for traffic matching only stateless mappings |
| Flow-specific NAT timeouts | Applicable to created sessions | No corresponding ordinary NAT session to expire |
| Requires routing | Yes | Yes |
| Automatically provides firewall protection | No | No |

**Static and stateless are not synonyms.**

- Static describes how the mapping is configured.
- Stateless describes whether flow-specific NAT session state is created.

---

## The `route-map` Keyword

Static NAT can be combined with a route map to conditionally select a translation.

For example:

```text
R1(config)# ip nat inside source static 10.1.1.5 7.7.7.50 route-map ISP1 extendable
```

The route map can provide additional conditions that determine when the mapping applies.

This is particularly useful when the same inside local address must be translated differently depending on the selected outside path.

For example:

```text
ip nat inside source static 10.1.1.5 7.7.7.50 route-map ISP1 extendable
ip nat inside source static 10.1.1.5 8.8.8.50 route-map ISP2 extendable
```

The route maps can distinguish between applicable traffic flows or forwarding paths.

The `extendable` keyword allows the overlapping mappings to coexist, whereas the route maps provide translation-selection conditions.

See [Route-Map-Based NAT and Conditional Translation](../policy-based/route-maps.md).

---

## The `reversible` Keyword

Ordinary static NAT is bidirectional.

However, adding route-map-based conditions to a NAT mapping can change whether new connections can be initiated in the reverse direction.

The `reversible` keyword supports reverse-direction initiation for applicable route-map-based NAT configurations.

For example:

```text
R1(config)# ip nat inside source static 10.1.1.5 7.7.7.50 route-map NAT-POLICY reversible
```

The exact matching behavior and supported combinations depend on the NAT feature and IOS XE release.

Do not confuse `reversible` with the ordinary bidirectional behavior of static NAT.

An ordinary static mapping does not require `reversible` to translate packets traveling in both directions.

The keyword is particularly relevant to policy-based NAT and is covered in [Route-Map-Based NAT and Conditional Translation](../policy-based/route-maps.md).

---

## VRF-Aware Static NAT Options

The `vrf` and `match-in-vrf` keywords are used with VRF-aware NAT.

### The `vrf` Keyword

The `vrf` keyword associates a supported NAT mapping with a particular VRF.

For example:

```text
R1(config)# ip nat inside source static 10.1.1.5 7.7.7.50 vrf CUSTOMER-A
```

This allows NAT to distinguish translations associated with different routing contexts.

It is particularly useful when multiple VRFs use overlapping inside address spaces.

### The `match-in-vrf` Keyword

The `match-in-vrf` keyword supports NAT when the inside and outside interfaces belong to the same VRF.

For example:

```text
R1(config)# ip nat inside source static 10.1.1.5 7.7.7.50 vrf CUSTOMER-A match-in-vrf
```

These keywords affect NAT's routing-context association and lookup behavior.

They do not independently establish routing between VRFs.

Detailed configuration examples are covered in:

- [VRF-to-Global NAT and PAT](../vrf/vrf-to-global.md)
- [Intra-VRF NAT and match-in-vrf](../vrf/match-in-vrf.md)
- [Overlapping Addresses Across VRFs](../vrf/overlapping-addresses.md)

---

## NAT Redundancy Options

Cisco supports additional static NAT options for certain high-availability designs.

### The `redundancy` Keyword

The `redundancy` keyword associates a NAT mapping with a supported redundancy group.

Depending on the redundancy mechanism and IOS XE release, the group can be identified by a name or numeric ID.

The purpose is to coordinate NAT behavior between redundant devices.

### The `mapping-id` Keyword

The `mapping-id` keyword identifies a NAT mapping in an applicable high-availability configuration.

These keywords are part of a larger NAT redundancy design.

Configuring them alone does not establish NAT state synchronization or complete a redundant NAT deployment.

### The `garp-interface` Keyword

Some static NAT command forms also support the `garp-interface` keyword.

It is associated with gratuitous ARP behavior in supported NAT configurations, particularly where neighboring devices need updated Layer 2 information following changes in address ownership.

Exact syntax, requirements, and behavior should be verified for the specific IOS XE platform and redundancy feature.

These redundancy-related options are less fundamental than `extendable`, `no-alias`, `no-payload`, and `add-route` for ordinary static NAT configuration.

---

## Removing Static NAT Mappings with Active Translations

A static NAT mapping may have dynamically created extended translations associated with it.

For example:

```text
R1# show ip nat translations
Pro  Inside global      Inside local       Outside local      Outside global
tcp  7.7.7.50:51000     10.1.1.5:51000     3.3.3.7:443        3.3.3.7:443
---  7.7.7.50           10.1.1.5           ---                ---
```

The simple static mapping is associated with an active extended translation.

Depending on the implementation, attempting to remove the static mapping can fail while associated translations remain active.

For example:

```text
R1(config)# no ip nat inside source static 10.1.1.5 7.7.7.50
```

The router may report that the mapping is in use.

### Clearing Dynamic Translations

In a controlled lab, the dynamic translations can be cleared first:

```text
R1# clear ip nat translation *
```

Then remove the static mapping:

```text
R1(config)# no ip nat inside source static 10.1.1.5 7.7.7.50
```

Be careful: clearing all dynamic NAT translations can interrupt unrelated connections.

### The `forced` Keyword

Supported static NAT command forms include the `forced` keyword, which can be used to force removal of a mapping despite associated active translations.

For example, where supported:

```text
R1(config)# no ip nat inside source static 10.1.1.5 7.7.7.50 forced
```

Check the available syntax using CLI context-sensitive help.

The important distinction is that removing a configured static mapping and clearing its associated dynamic translations are normally separate operations.

Forced removal can disrupt active flows using that mapping.

---

## Verification Commands

Several commands are particularly useful when troubleshooting static NAT options.

### Verify the Configured Keywords

```text
R1# show running-config | include ip nat
```

Look for:

- Explicitly configured keywords.
- Automatically appended keywords such as `extendable`.
- The referenced interface in interface-based mappings.
- VRF and route-map associations.

### Verify Translation Types and Flags

```text
R1# show ip nat translations verbose
```

For example, a static PAT entry might display:

```text
flags:
extended, extendable, static
```

A simple static address mapping might display:

```text
flags:
static
```

A dynamically created child entry might display:

```text
flags:
extended
```

Additional fields can include:

- Creation time.
- Last use time.
- Remaining lifetime.
- Use count.
- Entry ID.
- Internal flags.

The exact output depends on the implementation.

Do not confuse the lifetime of a dynamic child entry with the lifetime of its configured static parent mapping.

### Verify NAT Aliases

```text
R1# show ip aliases
```

Check whether an alias exists for the relevant translated address.

Remember:

- Inside source NAT can create aliases for on-link inside global addresses.
- Outside source NAT can create aliases for on-link outside local addresses.
- `no-alias` suppresses automatic NAT alias creation.
- Network mappings can create individual aliases on demand.

### Verify Neighbor ARP State

On ISP1:

```text
ISP1# show ip arp 7.7.7.50
```

On LH5, for an on-link outside local address:

```text
LH5# show ip arp 10.1.1.7
```

Confirm that the neighboring device associates the translated address with the expected MAC address.

### Verify Routing

```text
R1# show ip route 172.16.1.7
R1# show ip route 3.3.3.7
R1# show ip cef 172.16.1.7 detail
```

These commands are especially useful when troubleshooting outside source NAT and `add-route`.

### Debug NAT

In a controlled lab:

```text
R1# debug ip nat
```

Generate matching traffic and examine the translation behavior.

When finished:

```text
R1# undebug all
```

On platforms where forwarding is hardware-accelerated or handled by a separate forwarding plane, traditional NAT debugging might not display every packet.

Packet captures on both sides of the NAT router provide an additional verification method.

---

## Troubleshooting Scenarios

| Symptom | Possible cause |
|---|---|
| Second static NAT mapping is rejected | Overlapping mapping requires `extendable`, or mappings cannot be uniquely distinguished |
| Static PAT configuration contains an unexpected `extendable` keyword | IOS XE automatically appended it |
| NAT mapping exists, but outside packets never arrive | Missing ARP resolution or upstream routing |
| On-link translated address cannot be resolved | Missing NAT alias, `no-alias`, or another ARP-related problem |
| Traffic still works after configuring `no-alias` | Cached ARP entry, static ARP, more-specific route, or another ARP-response mechanism |
| Initial outside-initiated traffic fails with static network NAT | Corresponding on-link host alias has not been created |
| Outside source NAT fails despite a route to the outside global address | Missing or incorrect route to the outside local address |
| Network mapping with `add-route` does not install an aggregate route | Individual outside-local host routes are created on demand |
| Application fails despite correct IP and port translation | Embedded addressing information or ALG-related problem |
| NAT still translates header ports when `no-payload` is configured | Expected: `no-payload` affects embedded payload information, not header translation |
| No extended flow entries appear for a working static mapping | Mapping may be configured as `stateless` |
| Static mapping cannot be removed | Associated active translations may still exist |

---

## Command Summary

| Task | Command |
|---|---|
| Configure extendable static NAT | `ip nat inside source static <local-ip> <global-ip> extendable` |
| Suppress NAT alias creation | `ip nat inside source static <local-ip> <global-ip> no-alias` |
| Disable embedded payload translation | `ip nat inside source static <local-ip> <global-ip> no-payload` |
| Automatically install an outside-local route | `ip nat outside source static <global-ip> <local-ip> add-route` |
| Configure stateless static NAT | `ip nat inside source static <local-ip> <global-ip> stateless` |
| Configure stateless static PAT | `ip nat inside source static tcp <local-ip> <local-port> <global-ip> <global-port> extendable stateless` |
| Associate a route map with static NAT | `ip nat inside source static <local-ip> <global-ip> route-map <name>` |
| Display IP aliases | `show ip aliases` |
| Display NAT mappings | `show ip nat translations` |
| Display translation flags and state | `show ip nat translations verbose` |
| Display NAT configuration | `show running-config \| include ip nat` |
| Display ARP entries | `show ip arp` |
| Display routing information | `show ip route` |
| Clear dynamic translations | `clear ip nat translation *` |

---

## Key Takeaways

- `extendable` allows supported overlapping static NAT mappings to coexist.
- Static PAT mappings can share local and/or global IP addresses because their protocols and ports can distinguish them.
- IOS XE automatically appends `extendable` to static PAT mappings on the tested implementation.
- `extendable` does not itself determine which of multiple matching NAT rules should be used.
- NAT-generated IP aliases enable ARP resolution for on-link translated addresses.
- Inside source NAT aliases are associated with the **inside global** address.
- Outside source NAT aliases are associated with the **outside local** address.
- Off-link translated addresses do not normally require NAT aliases.
- `no-alias` suppresses automatic NAT alias creation without disabling the NAT mapping.
- NAT-generated aliases and ordinary proxy ARP are separate mechanisms.
- Static network NAT can create individual host translations and IP aliases on demand.
- Cached ARP entries from previous tests can conceal missing NAT alias behavior.
- `no-payload` prevents translation of embedded addressing information in application payloads, not packet-header translation.
- `add-route` is used with outside source NAT to install routes associated with outside local addresses.
- Ordinary single-host static outside source NAT with `add-route` installs a host route when configured.
- Static outside source network NAT with `add-route` can install individual host routes on demand rather than installing an aggregate route.
- `stateless` prevents ordinary per-flow NAT session creation for traffic matching only stateless mappings.
- Static NAT and stateless NAT are different concepts.
- Routing, ARP, NAT translation, and firewall policy must be considered separately when troubleshooting connectivity.

---

## References

- Cisco, [IP Addressing Configuration Guide, Cisco IOS XE 17.x: Configuring NAT for IP Address Conservation](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-addressing/b-ip-addressing/m_iadnat-addr-consv-xe.html)
- Cisco, [IP Addressing Configuration Guide, Cisco IOS XE 17.x: Information About Stateless Static NAT](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-addressing/b-ip-addressing/m_stateless_static_nat.html)
- Cisco, [IP Addressing Services Command Reference: ip nat inside source and ip nat outside source](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/ipaddr/command/ipaddr-cr-book/ipaddr-i3.html)
- Cisco, [Configure IP NAT Outside Source Static Command](https://www.cisco.com/c/en/us/support/docs/ip/ip-addressing-services/218383-use-ip-nat-command-for-configuration.html)
- Cisco, [Configure IP NAT Outside Source List Command](https://www.cisco.com/c/en/us/support/docs/ip/network-address-translation-nat/13770-1.html)
- Cisco, [Understand the NAT Order of Operation](https://www.cisco.com/c/en/us/support/docs/ip/network-address-translation-nat/6209-5.html)
- Cisco, [Network Address Translation (NAT) Frequently Asked Questions](https://www.cisco.com/c/en/us/support/docs/ip/network-address-translation-nat/26704-nat-faq-00.html)
- Jedadiah Casey, [Understanding and Configuring Network Address Translation (NAT) for Cisco CCNA, CCNP and CCIE](https://wax-trax.medium.com/understanding-and-configuring-network-address-translation-nat-for-cisco-ccna-ccnp-and-ccie-7fef92fa654c)
