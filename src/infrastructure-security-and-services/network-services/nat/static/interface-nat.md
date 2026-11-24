# Interface NAT and Default/DMZ Host

Instead of explicitly specifying an **inside global IP address**, Cisco IOS XE allows certain NAT mappings to reference an interface. The IP address of that interface is used as the inside global address.

This is particularly useful when:

- The outside interface receives its IP address dynamically via DHCP.
- Only one public IPv4 address is available.
- Internal services must be published using the router's outside interface address.
- A particular inside host should receive otherwise unmatched inbound traffic.

Cisco IOS XE supports three relevant configuration mechanisms:

| Mechanism | Command |
|---|---|
| Interface-based static NAT | `ip nat inside source static <local-ip> interface <interface>` |
| Interface-based static PAT | `ip nat inside source static tcp/udp <local-ip> <local-port> interface <interface> <global-port>` |
| Interface-based dynamic PAT | `ip nat inside source list <acl> interface <interface> overload` |

Interface-based address-only static NAT is also used to implement the **NAT Default Inside Server** feature, sometimes referred to as a **DMZ host**.

> The term *interface NAT* in this page refers to NAT mappings that use an interface's IP address. It should not be confused with NAT Virtual Interface (NVI).

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
| Remote_Host_7 | Outside host | `3.3.3.7/24` |
| Remote_Host_8 | Outside host | `3.3.3.8/24` |

Unlike the preceding exercises, which used `7.7.7.50` as a separate inside global address, this page uses Local_Router's actual outside interface address: `7.7.7.2`.

### Base Configuration

On Local_Router:

```text
Local_Router(config)# interface Ethernet0/0
Local_Router(config-if)# ip address 10.1.1.1 255.255.255.0
Local_Router(config-if)# ip nat inside
Local_Router(config-if)# no shutdown
!
Local_Router(config)# interface Ethernet0/1
Local_Router(config-if)# ip address 7.7.7.2 255.255.255.0
Local_Router(config-if)# ip nat outside
Local_Router(config-if)# no shutdown
!
Local_Router(config)# ip route 0.0.0.0 0.0.0.0 7.7.7.1
```

Assume:

- LH5 and LH6 use `10.1.1.1` as their default gateway.
- RH7 and RH8 use `3.3.3.1` as their default gateway.
- ISP1 knows both outside networks through connected routes.
- ISP1 does not have a route to `10.1.1.0/24`.

Remove the NAT mappings from the previous exercises before beginning this lab.

---

## Interface-Based Static NAT

The basic syntax is:

```text
Router(config)# ip nat inside source static <inside-local-ip> interface <interface>
```

For example:

```text
Local_Router(config)# ip nat inside source static 10.1.1.5 interface Ethernet0/1
```

Instead of specifying an inside global IP address explicitly, the command references `Ethernet0/1`.

Because `Ethernet0/1` currently has the address `7.7.7.2`, the effective address mapping is:

```text
Inside local                         Inside global
  10.1.1.5       <--------------->      7.7.7.2
```

This is an **address-only static NAT mapping**. It is not restricted to a particular TCP or UDP port.

| Field | Value |
|---|---|
| Inside local | `10.1.1.5` |
| Inside global | `7.7.7.2` |
| Referenced interface | `Ethernet0/1` |
| Protocol | Not restricted to TCP or UDP |
| Port translation | No explicit port mapping |

### Verify the Mapping

```text
Local_Router# show ip nat translations
Pro  Inside global      Inside local       Outside local      Outside global
---  7.7.7.2            10.1.1.5           ---                ---
```

The output is representative of the configured address-only mapping.

Although the configuration references an interface, the translation table displays the actual inside global IP address.

### Inside-to-Outside Translation

Suppose LH5 sends a packet to RH7.

| Field | Before NAT | After NAT |
|---|---|---|
| Source IP | `10.1.1.5` | `7.7.7.2` |
| Destination IP | `3.3.3.7` | `3.3.3.7` |

LH5 is represented to the outside network using Local_Router's outside interface address.

### Outside-to-Inside Translation

For an eligible inbound packet addressed to `7.7.7.2`:

| Field | Before NAT | After NAT |
|---|---|---|
| Source IP | `3.3.3.7` | `3.3.3.7` |
| Destination IP | `7.7.7.2` | `10.1.1.5` |

The address-only mapping can provide a fallback destination for inbound traffic that does not match a more-specific NAT translation.

This is the basis of the NAT Default Inside Server feature.

---

## Interface Reference vs. Explicit IP Address

Consider the following two commands:

```text
R1(config)# ip nat inside source static 10.1.1.5 7.7.7.50
```

```text
R1(config)# ip nat inside source static 10.1.1.5 interface Ethernet0/1
```

The first command explicitly specifies an inside global address.

The second references an interface and uses its address as the inside global address.

| Feature | Explicit address | Interface reference |
|---|---|---|
| Global address source | Manually specified IP | Interface's IP address |
| Configuration follows interface address changes | No | Yes, on supported implementations |
| Separate translated IP required | Typically yes | No |
| Separate NAT alias potentially required | Yes, for on-link global addresses | No separate address to resolve |
| Suitable for DHCP-assigned WAN addresses | Less convenient | Yes |

When the intended inside global address is the router's own interface address, use the supported interface-based NAT syntax rather than assuming that a literal-address mapping is interchangeable with it.

Cisco documents restrictions concerning the use of a router's own IP addresses in ordinary NAT pools and static mappings.

### Dynamically Assigned Outside Address

For example, the outside interface could receive its address via DHCP:

```text
R1(config)# interface Ethernet0/1
R1(config-if)# ip address dhcp
R1(config-if)# ip nat outside
R1(config-if)# no shutdown
!
R1(config)# ip nat inside source static 10.1.1.5 interface Ethernet0/1
```

The NAT configuration does not need to contain the actual DHCP-assigned address.

If the interface's address changes, the interface-based mapping can follow the new address.

However:

- Existing sessions using the old inside global address can be interrupted.
- Remote clients must learn the new address.
- NAT does not automatically update external DNS records.

Interface-based NAT simplifies configuration when addressing is dynamic, but it does not guarantee uninterrupted connectivity after an address change.

---

## Interface-Based Static PAT

As covered in [Static PAT and Port Forwarding](pat.md), the inside global address of a static PAT mapping can also be specified using an interface.

The syntax is:

```text
Router(config)# ip nat inside source static {tcp | udp} <local-ip> <local-port> interface <interface> <global-port>
```

For example:

```text
Local_Router(config)# ip nat inside source static tcp 10.1.1.6 22 interface Ethernet0/1 2222
```

The effective mapping is:

```text
TCP 10.1.1.6:22 <-------> 7.7.7.2:2222
```

RH7 can connect to LH6's SSH server using:

```text
Remote_Host_7# ssh -l admin -p 2222 7.7.7.2
```

Assuming SSH is configured on LH6, the connection is forwarded to `10.1.1.6:22`.

### Packet Translation

| Direction | Before NAT | After NAT |
|---|---|---|
| Outside to inside | Destination `7.7.7.2:2222` | Destination `10.1.1.6:22` |
| Inside to outside | Source `10.1.1.6:22` | Source `7.7.7.2:2222` |

Only traffic matching the relevant TCP endpoint uses this mapping.

### Multiple Interface-Based Static PAT Mappings

Multiple services can share the interface address:

```text
Local_Router(config)# ip nat inside source static tcp 10.1.1.5 443 interface Ethernet0/1 8443
Local_Router(config)# ip nat inside source static tcp 10.1.1.6 22 interface Ethernet0/1 2222
Local_Router(config)# ip nat inside source static udp 10.1.1.6 53 interface Ethernet0/1 53
```

The resulting mappings are:

| Protocol | Inside global | Inside local |
|---|---|---|
| TCP | `7.7.7.2:8443` | `10.1.1.5:443` |
| TCP | `7.7.7.2:2222` | `10.1.1.6:22` |
| UDP | `7.7.7.2:53` | `10.1.1.6:53` |

IOS XE automatically appends the `extendable` keyword to static PAT mappings to enable this behavior.

**Interface-based static PAT does not require the `overload` keyword.** Each port mapping is explicitly configured.

---

## NAT Default Inside Server

The **NAT Default Inside Server** feature allows otherwise unmatched inbound traffic addressed to a NAT interface's IP address to be forwarded to a designated inside host.

Cisco documents this feature for situations in which inbound connections can use unpredictable ports, such as certain online gaming applications.

Rather than configuring an individual static PAT mapping for every required port, the administrator configures an address-only interface mapping:

```text
Local_Router(config)# ip nat inside source static 10.1.1.5 interface Ethernet0/1
```

LH5 becomes the **default inside server** associated with the outside interface address.

The feature provides fallback translation when inbound traffic does not match an existing fully extended translation or static port mapping.

There is no separate `default-server` or `dmz-host` keyword in this configuration.

### Default Inside Server Concept

```text
                    Outside network
                           |
                           |
                    Destination IP:
                       7.7.7.2
                           |
                           v
                   +--------------+
                   | Local_Router |
                   +--------------+
                           |
                 Check NAT translations
                           |
           +---------------+----------------+
           |                                |
  Matching specific                  No matching specific
    translation                          translation
           |                                |
           v                                v
   Translate using                  Translate using the
   matching entry                   default inside server
                                            |
                                            v
                                        10.1.1.5
```

The default inside server is a **fallback**, not an override of all other NAT translations.

### Why This Feature Is Useful

Consider a gaming device that needs to accept incoming UDP traffic on ports that are not known in advance.

Without a default inside server, the administrator might need to:

- Configure numerous static PAT mappings.
- Modify the configuration whenever the required ports change.
- Depend on dynamically created translations being available when inbound packets arrive.

A default inside server allows eligible traffic without a more-specific translation to be directed to the device.

However, this convenience comes at the cost of potentially exposing more services to the outside network.

---

## Default Inside Server with Static PAT

Suppose we want:

- LH5 to be the default inside server.
- LH6 to receive incoming SSH connections addressed to TCP port `2222` on Local_Router.

Configure:

```text
Local_Router(config)# ip nat inside source static 10.1.1.5 interface Ethernet0/1
Local_Router(config)# ip nat inside source static tcp 10.1.1.6 22 interface Ethernet0/1 2222
```

Both mappings use the same inside global IP address: `7.7.7.2`.

However, the static PAT entry is specific to TCP port `2222`, whereas the address-only mapping is not restricted to a particular port.

### Expected Translation Table

```text
Local_Router# show ip nat translations
Pro  Inside global       Inside local       Outside local    Outside global
---  7.7.7.2             10.1.1.5           ---              ---
tcp  7.7.7.2:2222        10.1.1.6:22        ---              ---
```

### Example 1: Incoming Connection to TCP Port 2222

RH7 sends:

```text
Source:       3.3.3.7:51000
Destination:  7.7.7.2:2222
Protocol:     TCP
```

The packet matches the static PAT entry:

```text
TCP 7.7.7.2:2222 <-> 10.1.1.6:22
```

Local_Router translates the destination and forwards the packet to LH6.

```text
Source:       3.3.3.7:51000
Destination:  10.1.1.6:22
Protocol:     TCP
```

The default inside server is not used.

### Example 2: Incoming Connection to TCP Port 443

RH7 sends:

```text
Source:       3.3.3.7:51001
Destination:  7.7.7.2:443
Protocol:     TCP
```

Assuming no more-specific translation or router-local handling takes precedence, the default inside server mapping is used.

After NAT:

```text
Source:       3.3.3.7:51001
Destination:  10.1.1.5:443
Protocol:     TCP
```

The address changes, but the destination port remains `443`.

### Example 3: Incoming Connection to UDP Port 5000

RH7 sends:

```text
Source:       3.3.3.7:51002
Destination:  7.7.7.2:5000
Protocol:     UDP
```

If no more-specific translation exists, the default inside server mapping can translate the packet to:

```text
Source:       3.3.3.7:51002
Destination:  10.1.1.5:5000
Protocol:     UDP
```

Unlike a TCP-specific or UDP-specific static PAT rule, the address-only fallback mapping is not restricted to a particular transport protocol.

### Summary

| Incoming destination | Selected mapping | Translated destination |
|---|---|---|
| `7.7.7.2:2222/TCP` | Static PAT | `10.1.1.6:22/TCP` |
| `7.7.7.2:443/TCP` | Default inside server | `10.1.1.5:443/TCP` |
| `7.7.7.2:80/TCP` | Default inside server | `10.1.1.5:80/TCP` |
| `7.7.7.2:5000/UDP` | Default inside server | `10.1.1.5:5000/UDP` |
| `7.7.7.2:2222/UDP` | Default inside server | `10.1.1.5:2222/UDP` |

The last example is important: the static PAT entry for TCP port `2222` does not also match UDP port `2222`.

These results assume that there is no matching existing dynamic translation, competing router-local service, or filtering policy.

---

## Default Inside Server with Dynamic PAT

A common design combines three mechanisms:

1. **Dynamic PAT** for ordinary outbound connections.
2. **Static PAT** for explicitly published inside services.
3. **Default inside server** for other eligible inbound traffic.

All three mechanisms can use the same outside interface address on supported IOS XE implementations.

Consider the following configuration:

```text
Local_Router(config)# ip nat inside source static 10.1.1.5 interface Ethernet0/1
Local_Router(config)# ip nat inside source static tcp 10.1.1.6 22 interface Ethernet0/1 2222
!
Local_Router(config)# ip access-list standard NAT-OVERLOAD
Local_Router(config-std-nacl)# permit 10.1.1.6
!
Local_Router(config)# ip nat inside source list NAT-OVERLOAD interface Ethernet0/1 overload
```

In this example:

- LH5 has an address-only static interface mapping.
- LH6 has a static PAT mapping for its SSH service.
- LH6 can also use dynamic PAT for other outbound connections.

The NAT selection ACL is deliberately limited to LH6 to keep the roles of the two inside hosts clear.

### Outbound Dynamic PAT Connection

Suppose LH6 initiates an HTTPS connection to RH7:

```text
Source:       10.1.1.6:51000
Destination:  3.3.3.7:443
```

Local_Router creates a dynamic PAT translation.

For example:

```text
Source:       7.7.7.2:40001
Destination:  3.3.3.7:443
```

The translated source port is illustrative. IOS XE may preserve the original source port if it is available.

### Return Traffic

RH7 responds:

```text
Source:       3.3.3.7:443
Destination:  7.7.7.2:40001
```

The packet matches the existing dynamic PAT translation.

Local_Router translates the destination to:

```text
Destination:  10.1.1.6:51000
```

Even though LH5 is the default inside server, the reply is delivered to LH6.

**The default inside server does not steal return traffic belonging to an existing matching dynamic PAT translation.**

### Representative Translation Table

```text
Local_Router# show ip nat translations
Pro  Inside global       Inside local       Outside local      Outside global
---  7.7.7.2             10.1.1.5           ---                ---
tcp  7.7.7.2:2222        10.1.1.6:22        ---                ---
tcp  7.7.7.2:40001       10.1.1.6:51000     3.3.3.7:443        3.3.3.7:443
```

The three entries serve different purposes:

| Entry | Purpose |
|---|---|
| Address-only static entry | Default inside server |
| Static TCP port entry | Publish LH6's SSH service |
| Fully extended dynamic entry | Track LH6's outbound HTTPS connection |

The precise output and associated flow entries can vary by platform and release.

### When the Dynamic Translation Expires

Dynamic PAT translations are subject to protocol-specific timeouts.

If the dynamic entry associated with `7.7.7.2:40001` expires, a subsequent inbound packet to that port can no longer use the expired mapping.

If the packet does not match another translation and is eligible for fallback NAT, it may be forwarded to LH5 using the default inside server mapping.

This illustrates an important security implication:

**Traffic that is no longer associated with an existing dynamic translation is not necessarily discarded when a default inside server is configured.**

Appropriate firewall or ACL policies are still necessary.

---

## Translation Selection and Precedence

For the purposes of understanding the NAT Default Inside Server feature, consider the following simplified inbound translation lookup:

```text
Incoming packet destined for the interface address
                         |
                         v
       Matching fully extended translation?
                  /              \
                Yes               No
                 |                 |
                 v                 v
          Use translation    Matching static
                              port translation?
                              /             \
                            Yes             No
                             |               |
                             v               v
                       Use static PAT   Use default
                                        inside server
                                        if configured
```

The default inside server is represented by an address-only static mapping.

It is less specific than mappings identifying the relevant transport protocol, ports, or outside endpoints.

### Example

Assume the translation table contains:

```text
Pro  Inside global       Inside local       Outside local      Outside global
---  7.7.7.2             10.1.1.5           ---                ---
tcp  7.7.7.2:2222        10.1.1.6:22        ---                ---
tcp  7.7.7.2:40001       10.1.1.6:51000     3.3.3.7:443        3.3.3.7:443
```

| Incoming packet | Translation used |
|---|---|
| TCP from `3.3.3.7:443` to `7.7.7.2:40001` | Existing dynamic PAT entry |
| TCP from `3.3.3.8:52000` to `7.7.7.2:2222` | Static PAT entry |
| TCP from `3.3.3.8:52000` to `7.7.7.2:8080` | Default inside server |
| UDP from `3.3.3.8:53000` to `7.7.7.2:2222` | Default inside server |

**Important:** This is a conceptual explanation of the documented default-server fallback behavior, not a universal NAT lookup algorithm for every combination of NAT features.

Policy-based NAT, VRF-aware NAT, NAT ALGs, existing flow state, and platform-specific behavior can introduce additional considerations.

Also, the existence of a dynamic PAT entry using a particular global port does not necessarily mean every packet addressed to that port matches the entry. The other identifying fields in the translation can matter.

---

## Interaction with the Router's Own Services

Using a separate inside global IP address and using the router's actual interface IP address introduce different considerations.

For example:

```text
ip nat inside source static 10.1.1.5 7.7.7.50
```

Here, the translated address is distinct from the router's outside interface address.

However:

```text
ip nat inside source static 10.1.1.5 interface Ethernet0/1
```

Here, the translated address is also the router's own interface address, `7.7.7.2`.

Outside packets addressed to `7.7.7.2` could potentially be intended for either:

- A service running on Local_Router itself.
- A service published through static PAT.
- An inside host associated with an existing dynamic PAT translation.
- The default inside server.

### Management Port Conflicts

Suppose Local_Router itself has an SSH server listening on TCP port `22`.

Also suppose the following NAT mapping is configured:

```text
Local_Router(config)# ip nat inside source static tcp 10.1.1.6 22 interface Ethernet0/1 22
```

The same outside-facing TCP endpoint is now potentially associated with both the router's SSH service and a static PAT mapping.

The NAT implementation and platform determine whether the configuration is accepted and how the conflict is handled.

Do not assume that the router's own services always take precedence over NAT, or that NAT always takes precedence over router-local services.

A better design is to use a different external port for the inside SSH server:

```text
Local_Router(config)# ip nat inside source static tcp 10.1.1.6 22 interface Ethernet0/1 2222
```

This avoids an explicit static PAT conflict on TCP port `22`.

However, when a default inside server is also configured, the handling of otherwise unmatched management traffic should still be tested.

### Management Recommendations

When configuring a default inside server:

- Prefer managing the NAT router through a dedicated management interface or network.
- Avoid relying exclusively on the outside interface for remote administration.
- Check for conflicts involving SSH, HTTPS, VPN services, and other router-local protocols.
- Verify router-local reachability separately from translated reachability.

Changing NAT configuration remotely can disrupt access to the router itself.

---

## Routing and ARP Considerations

One advantage of using the outside interface address is that a separate translated IP address does not need to be delivered to the NAT router.

Consider the previous static NAT exercise:

```text
Outside interface:   7.7.7.2
Inside global:       7.7.7.50
```

ISP1 needs to resolve `7.7.7.50` through ARP because it considers that address directly connected.

Depending on the NAT configuration, Local_Router may need to provide an ARP response through a NAT-generated alias.

With interface-based NAT:

```text
Outside interface:   7.7.7.2
Inside global:       7.7.7.2
```

ISP1 simply resolves Local_Router's actual interface address.

```text
ISP1                              Local_Router
  |                                     |
  | ARP Request: Who has 7.7.7.2?       |
  +------------------------------------>|
  |                                     |
  | ARP Reply: 7.7.7.2 is at my MAC     |
  |<------------------------------------+
```

No separate translated address needs to be resolved.

### Comparison

| Inside global address | Upstream delivery mechanism |
|---|---|
| Separate address on connected subnet | ARP resolution for the translated address, potentially using a NAT alias |
| Separate address on routed prefix | Routing toward the NAT router's next-hop address |
| Outside interface address | Normal routing and ARP toward the interface address |

This simplifies upstream reachability, but it does not eliminate the need for routing between the outside client, NAT router, and inside host.

The inside host must also send its replies through the appropriate NAT router.

---

## NAT Default Inside Server vs. DMZ Host

Consumer routers frequently provide a feature called **DMZ host** or **exposed host**.

Despite the name, this often refers to a NAT feature similar in purpose to the Cisco NAT Default Inside Server: unmatched eligible inbound traffic is directed to a specified internal device.

For example:

```text
DMZ host: 10.1.1.5
```

This is conceptually similar to:

```text
ip nat inside source static 10.1.1.5 interface Ethernet0/1
```

However, implementation details differ across vendors.

### A DMZ Host Is Not a True DMZ

A traditional **demilitarized zone (DMZ)** is a separate network or security zone used to isolate externally accessible services from trusted internal systems.

For example:

```text
                       Internet
                           |
                           |
                      +----------+
                      | Firewall |
                      +----------+
                       /        \
                      /          \
                     /            \
               DMZ network     Internal network
              172.16.10.0/24    10.1.1.0/24
                     |                |
                Public server      Trusted PCs
```

Firewall policies control traffic between the outside network, DMZ, and internal network.

A default inside server does not inherently create that separation.

```text
                       Internet
                           |
                       NAT router
                           |
                    10.1.1.0/24
                           |
                 +---------+---------+
                 |                   |
           Default server       Trusted PC
             10.1.1.5            10.1.1.6
```

Both hosts may remain on the same inside subnet.

If the default inside server is compromised, NAT alone does not prevent it from communicating with other devices on that subnet.

| Feature | NAT Default Inside Server / DMZ host | Traditional DMZ |
|---|---|---|
| Primary purpose | Redirect otherwise unmatched inbound traffic | Isolate externally accessible systems |
| Separate subnet required | No | Typically yes |
| Security zone created | No | Yes, in a zone-based design |
| Firewall policy inherently configured | No | Requires appropriate policy |
| Provides internal network isolation | No | With appropriate design and policy |

**A NAT Default Inside Server is a translation mechanism, not a security architecture.**

---

## Security Considerations

A default inside server can expose significantly more traffic to an inside host than a single static PAT mapping.

For example:

```text
ip nat inside source static tcp 10.1.1.5 443 interface Ethernet0/1 443
```

This exposes one specific TCP endpoint through NAT.

By contrast:

```text
ip nat inside source static 10.1.1.5 interface Ethernet0/1
```

This allows a much broader range of eligible unmatched inbound traffic to be translated toward LH5.

Without appropriate filtering, LH5 may receive traffic targeting services that were not intended to be publicly accessible.

### NAT Does Not Replace a Firewall

The default inside server configuration does not:

- Authenticate outside clients.
- Restrict access to trusted source addresses.
- Inspect application traffic for malicious content.
- Isolate the inside server from other inside hosts.
- Automatically implement a stateful security policy.

Use ACLs, Zone-Based Firewall (ZBF), or an external firewall as appropriate.

### ACL Placement

For traditional outside-to-inside NAT, an inbound interface ACL evaluates packets before destination translation.

Suppose we want to permit HTTPS traffic toward the default inside server.

The packet arrives with:

```text
Destination IP:   7.7.7.2
Destination port: TCP 443
```

Therefore, an outside interface ACL would match the inside global address:

```text
permit tcp any host 7.7.7.2 eq 443
```

It would not match `10.1.1.5` as the destination address at that point in the processing path.

For the static PAT example:

```text
ip nat inside source static tcp 10.1.1.6 22 interface Ethernet0/1 2222
```

The corresponding pre-NAT ACL entry would be:

```text
permit tcp any host 7.7.7.2 eq 2222
```

Notice that the ACL matches the **global port (2222)**, not the **local port (22)**.

These are individual ACL entries, not a complete outside interface security policy. A complete policy must also consider legitimate return traffic, routing protocols, management traffic, and other required services.

See [NAT Packet Processing and Order of Operations](../order-of-operations.md) for details.

---

## Verification and Troubleshooting

### Verify the NAT Configuration

```text
Local_Router# show running-config | include ip nat
```

For the combined example, the configuration should include the equivalent of:

```text
ip nat inside
ip nat outside
ip nat inside source list NAT-OVERLOAD interface Ethernet0/1 overload
ip nat inside source static 10.1.1.5 interface Ethernet0/1
ip nat inside source static tcp 10.1.1.6 22 interface Ethernet0/1 2222
```

IOS XE may include additional automatically generated keywords, such as `extendable`.

### Verify the Referenced Interface's Address

```text
Local_Router# show ip interface brief
```

Representative output:

```text
Interface     IP-Address      OK? Method Status  Protocol
Ethernet0/0   10.1.1.1        YES manual up      up
Ethernet0/1   7.7.7.2         YES manual up      up
```

Unlike a literal-address mapping, the effective global address of an interface-based mapping depends on the referenced interface.

If the interface has no usable IP address, the mapping cannot provide the intended translated endpoint.

### Verify the Translation Table

```text
Local_Router# show ip nat translations
Local_Router# show ip nat translations verbose
```

Check for:

- The address-only default inside server entry.
- The expected static PAT entries.
- Dynamic PAT translations created by outbound traffic.
- The correct inside global address.
- The correct protocol and port information.

### Verify NAT Statistics

```text
Local_Router# show ip nat statistics
```

Check:

- Configured inside and outside interfaces.
- Active static and dynamic translations.
- Translation hits and misses.
- Expired translations.

The exact statistics available depend on the platform.

### Test Static PAT

From RH7, initiate an SSH connection to:

```text
7.7.7.2:2222
```

Verify that the connection reaches LH6 rather than LH5.

If RH7 is an IOS router:

```text
Remote_Host_7# ssh -l admin -p 2222 7.7.7.2
```

The SSH username and server configuration must be valid on LH6.

### Test the Default Inside Server

Configure a test service on LH5 that does not have a corresponding static PAT mapping.

For example, use an HTTP server listening on TCP port `8080`.

From a Linux outside host:

```bash
curl http://7.7.7.2:8080/
```

The intended result is a connection to LH5's TCP port `8080`.

Using a distinguishable service on LH5 is preferable to relying solely on ping.

A successful ping to the NAT router's interface address does not necessarily prove that the default inside server received the packet, because the address also belongs to the router itself.

A packet capture on LH5's interface can confirm whether translation occurred.

### Test Dynamic PAT

Initiate an outbound connection from LH6 to RH7.

While the connection is active:

```text
Local_Router# show ip nat translations
```

Confirm that a dynamic PAT entry is created and that replies return to LH6 rather than being forwarded to LH5.

This demonstrates that an existing matching translation takes precedence over the fallback mapping.

### Debug NAT

In a controlled lab:

```text
Local_Router# debug ip nat
```

Generate traffic for each of the following:

- An explicitly mapped static PAT port.
- An unmatched port intended for the default inside server.
- An outbound connection using dynamic PAT.
- Return traffic belonging to that dynamic PAT connection.

Compare the resulting translations.

Disable debugging when finished:

```text
Local_Router# undebug all
```

Debug support and visibility vary across IOS XE forwarding implementations.

### Common Problems

| Symptom | Possible cause |
|---|---|
| Interface-based mapping has an unexpected global address | Referenced interface's IP address changed |
| Interface-based mapping does not function | Interface has no IP address or feature is unsupported |
| All intended inbound services fail | Routing, NAT interface roles, ACL, or firewall problem |
| Static PAT works but default inside server does not | Missing address-only mapping, competing translation, or platform limitation |
| Traffic reaches the wrong inside host | More-specific translation matched instead of the default mapping |
| TCP port is forwarded but equivalent UDP port is not | Protocol-specific static PAT mapping |
| Router management becomes inaccessible | NAT conflict involving the router's own interface address or service |
| Outbound connections from another inside host fail | Missing or incorrect dynamic PAT configuration |
| Outside client cannot connect despite successful NAT | Inside server is not listening or its host firewall blocks the connection |

---

## Removing the Default Inside Server

To remove the fallback mapping:

```text
Local_Router(config)# no ip nat inside source static 10.1.1.5 interface Ethernet0/1
```

The separately configured static PAT and dynamic PAT rules can remain in place.

For example:

```text
ip nat inside source static tcp 10.1.1.6 22 interface Ethernet0/1 2222
ip nat inside source list NAT-OVERLOAD interface Ethernet0/1 overload
```

The router can continue providing:

- Static port forwarding to LH6.
- Dynamic PAT for matching outbound connections.

However, the address-only fallback translation to LH5 is no longer configured.

Depending on active translations and the IOS XE implementation, existing flow entries may require clearing before the configuration can be removed or changed.

In an isolated lab, dynamic translations can be cleared with:

```text
Local_Router# clear ip nat translation *
```

This can interrupt active translated connections. Use caution on production devices.

The command is not a substitute for removing a configured static NAT rule.

---

## Command Summary

| Task | Command |
|---|---|
| Configure interface-based static NAT / default inside server | `ip nat inside source static <local-ip> interface <interface>` |
| Configure interface-based static TCP PAT | `ip nat inside source static tcp <local-ip> <local-port> interface <interface> <global-port>` |
| Configure interface-based static UDP PAT | `ip nat inside source static udp <local-ip> <local-port> interface <interface> <global-port>` |
| Configure interface-based dynamic PAT | `ip nat inside source list <acl> interface <interface> overload` |
| Remove default inside server | `no ip nat inside source static <local-ip> interface <interface>` |
| Display translations | `show ip nat translations` |
| Display detailed translations | `show ip nat translations verbose` |
| Display NAT statistics | `show ip nat statistics` |
| Verify interface addressing | `show ip interface brief` |
| Debug NAT | `debug ip nat` |

---

## Key Takeaways

- Interface-based NAT uses a referenced interface's IP address as the **inside global address**.
- Both static NAT and static PAT can reference an interface rather than a literal global IP address.
- Interface-based NAT is particularly useful when the outside address is dynamically assigned.
- An address-only static interface mapping implements the **NAT Default Inside Server** feature.
- The command is `ip nat inside source static <local-ip> interface <interface>`.
- The default inside server is a fallback for eligible inbound traffic without a matching fully extended translation or static port mapping.
- A matching static PAT entry takes precedence over the default inside server.
- A matching existing dynamic PAT entry also takes precedence over the default inside server.
- Static PAT, dynamic PAT, and a default inside server can coexist using the same interface address on supported implementations.
- Because the translated address is the interface's actual address, a separate translated-address ARP alias is unnecessary.
- Using the router's own interface address for NAT introduces potential conflicts with router-local services.
- A default inside server is conceptually similar to the DMZ host feature found on consumer routers.
- A DMZ host is not a true DMZ network or security zone.
- A default inside server does not provide firewall protection. Configure security policies separately.

---

## References

- Cisco, [IP Addressing Configuration Guide, Cisco IOS XE 17.x: Configuring NAT for IP Address Conservation](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-addressing/b-ip-addressing/m_iadnat-addr-consv-xe.html) (see *Configuring the NAT Default Inside Server Feature*)
- Cisco, [IP Addressing: NAT Configuration Guide, Cisco IOS XE Release 3S: Configuring NAT for IP Address Conservation](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/ipaddr_nat/configuration/xe-3s/nat-xe-3s-book/iadnat-addr-consv.html)
- Cisco, [Network Address Translation (NAT) Frequently Asked Questions](https://www.cisco.com/c/en/us/support/docs/ip/network-address-translation-nat/26704-nat-faq-00.html)
- Cisco, [Understand the NAT Order of Operation](https://www.cisco.com/c/en/us/support/docs/ip/network-address-translation-nat/6209-5.html)
- Cisco, [Troubleshoot Network Address Translation with Frequently Asked Questions](https://www.cisco.com/c/en/us/support/docs/ip/network-address-translation-nat/218012-troubleshoot-network-address-translation.html)
