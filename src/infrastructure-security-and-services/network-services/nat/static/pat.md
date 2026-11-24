# Static PAT and Port Forwarding

**Static Port Address Translation (PAT)** creates a fixed mapping between an inside host's IP address and TCP/UDP port and an inside global IP address and TCP/UDP port.

It is commonly used for **port forwarding**, allowing outside devices to access a particular service running on an inside host without requiring a dedicated outside IP address for that host.

Unlike ordinary static NAT, which maps entire IP addresses, static PAT identifies a specific **protocol, IP address, and port combination**.

For example:

```text
Inside local:    10.1.1.5:443
Inside global:   7.7.7.50:8443
Protocol:        TCP
```

An outside device connecting to `7.7.7.50` on TCP port `8443` is redirected to `10.1.1.5` on TCP port `443`.

Static PAT can:

- Preserve the original TCP/UDP port number.
- Translate one TCP/UDP port to a different port.
- Allow different inside servers to share a single inside global IP address by using different ports.
- Allow outside hosts to initiate connections without requiring an existing inside-initiated session.

**Static PAT does not require the `overload` keyword.** Static PAT creates a fixed endpoint mapping, whereas dynamic PAT (NAT overload) dynamically allocates translations for individual connections.

---

## Configuration Syntax

The basic command is:

```text
Router(config)# ip nat inside source static {tcp | udp} <inside-local-ip> <local-port> <inside-global-ip> <global-port>
```

| Argument | Description |
|---|---|
| `tcp \| udp` | Transport protocol to translate |
| `inside-local-ip` | Inside host's original IP address |
| `local-port` | TCP/UDP port used by the inside host |
| `inside-global-ip` | IP address representing the inside host to outside devices |
| `global-port` | TCP/UDP port used by outside devices |

The **inside local address and port come first**, followed by the **inside global address and port**.

For example:

```text
R1(config)# ip nat inside source static tcp 10.1.1.5 443 7.7.7.50 8443
```

This creates the following mapping:

```text
TCP 10.1.1.5:443 <----> 7.7.7.50:8443
```

The command creates a static translation that exists independently of any particular TCP connection.

It does not require:

- A NAT pool.
- An ACL selecting traffic for translation.
- The `overload` keyword.
- An existing connection initiated by the inside host.

However, the interfaces must have the appropriate NAT roles, and the routing and security configuration must permit the traffic.

### Static NAT vs. Static PAT

| Feature | Static NAT | Static PAT |
|---|---|---|
| Translation matches | IP address | Protocol, IP address, and port |
| TCP/UDP ports translated | No | Yes, if different ports are configured |
| Affects ICMP traffic | Yes | No, for TCP/UDP-specific mappings |
| Multiple inside servers sharing one global IP | Not with ordinary one-to-one mappings | Yes, using different ports |
| Inbound initiation | Supported | Supported for mapped ports |
| Command | `ip nat inside source static` | `ip nat inside source static tcp/udp` |

**Important:** Static PAT does not translate every packet from an inside host. The transport protocol and relevant port must match the configured static PAT entry.

---

## Lab Topology

We'll continue using the same topology from the previous static NAT lessons.

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
| Local_Host_5 | Inside web server | `10.1.1.5/24` |
| Local_Host_6 | Additional inside host | `10.1.1.6/24` |
| Local_Router | `E0/0`, NAT inside | `10.1.1.1/24` |
| Local_Router | `E0/1`, NAT outside | `7.7.7.2/24` |
| ISP1 | Toward Local_Router | `7.7.7.1/24` |
| ISP1 | Toward remote hosts | `3.3.3.1/24` |
| Remote_Host_7 | Outside client | `3.3.3.7/24` |
| Remote_Host_8 | Additional outside client | `3.3.3.8/24` |

Assume:

- LH5 runs an HTTPS server listening on TCP port `443`.
- LH5 and LH6 use `10.1.1.1` as their default gateway.
- RH7 and RH8 use `3.3.3.1` as their default gateway.
- Local_Router has a default route toward `7.7.7.1`.
- No filtering prevents the desired TCP connections.

We want RH7 to connect to the HTTPS server using:

```text
7.7.7.50:8443
```

Local_Router will translate this to:

```text
10.1.1.5:443
```

---

## Configure Static PAT

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

### Step 3: Configure Static PAT

```text
Local_Router(config)# ip nat inside source static tcp 10.1.1.5 443 7.7.7.50 8443
```

The mapping is:

| Field | Value |
|---|---|
| Protocol | TCP |
| Inside local IP | `10.1.1.5` |
| Inside local port | `443` |
| Inside global IP | `7.7.7.50` |
| Inside global port | `8443` |

**Note:** If you are reusing the previous static NAT lab, remove its address-only static NAT entry before configuring this mapping:

```text
Local_Router(config)# no ip nat inside source static 10.1.1.5 7.7.7.50
```

Also remove any previous network-wide mappings that overlap this exercise. Address-only static NAT and static PAT mappings involving the same addresses can create conflicts.

### Step 4: Configure the Default Route

```text
Local_Router(config)# ip route 0.0.0.0 0.0.0.0 7.7.7.1
```

Local_Router uses this route to reach the outside network.

No additional route to `10.1.1.5` is necessary because `10.1.1.0/24` is directly connected.

### Step 5: Verify the Static Translation

```text
Local_Router# show ip nat translations
Pro  Inside global         Inside local         Outside local    Outside global
tcp  7.7.7.50:8443        10.1.1.5:443        ---              ---
```

The output shows a static TCP translation.

Unlike a dynamic translation, this mapping is present before any traffic matches it.

The `---` values in the outside address columns indicate that the configured static mapping is not restricted to a particular outside host.

**Any outside client can potentially use the mapping**, provided the packets reach the NAT router, pass the relevant security policies, and the inside service responds.

The displayed output is representative. Some IOS XE platforms may also display additional flow-specific entries after connections are established.

---

## Outside-to-Inside Packet Flow

Suppose RH7 initiates a TCP connection to the web server.

RH7 chooses TCP source port `51514` and sends a SYN packet:

```text
Source:       3.3.3.7:51514
Destination:  7.7.7.50:8443
```

When the packet arrives at Local_Router's outside interface, the router looks for a matching static PAT entry.

The configured entry matches because:

- The protocol is TCP.
- The destination IP address is `7.7.7.50`.
- The destination TCP port is `8443`.

Local_Router translates the destination address and port:

```text
Destination IP:    7.7.7.50 -> 10.1.1.5
Destination port:  8443     -> 443
```

The packet forwarded toward LH5 is:

```text
Source:       3.3.3.7:51514
Destination:  10.1.1.5:443
```

### Translation Table

| Field | Before NAT | After NAT |
|---|---|---|
| Source IP | `3.3.3.7` | `3.3.3.7` |
| Source TCP port | `51514` | `51514` |
| Destination IP | `7.7.7.50` | `10.1.1.5` |
| Destination TCP port | `8443` | `443` |

The source IP address and source port remain unchanged.

### Packet Flow Diagram

```text
 RH7                        ISP1                 Local_Router                 LH5
3.3.3.7                                        Static PAT                10.1.1.5
   |                           |                     |                        |
   | TCP SYN                   |                     |                        |
   | Src 3.3.3.7:51514         |                     |                        |
   | Dst 7.7.7.50:8443         |                     |                        |
   +-------------------------->+-------------------->|                        |
                                                   NAT                        |
                                         7.7.7.50:8443                        |
                                               ->                             |
                                         10.1.1.5:443                         |
                                                     |                        |
                                                     | TCP SYN                |
                                                     | Src 3.3.3.7:51514      |
                                                     | Dst 10.1.1.5:443       |
                                                     +----------------------->|
```

For traditional outside-to-inside NAT, destination translation occurs before the routing lookup used to forward the packet toward the inside host.

After translation, Local_Router forwards the packet using its connected route to `10.1.1.0/24`.

---

## Inside-to-Outside Return Traffic

LH5 responds with a TCP SYN-ACK.

Before NAT:

```text
Source:       10.1.1.5:443
Destination:  3.3.3.7:51514
```

Local_Router uses the static PAT mapping in the reverse direction.

It changes the source address and port:

```text
Source IP:    10.1.1.5 -> 7.7.7.50
Source port:  443      -> 8443
```

The packet forwarded toward RH7 is:

```text
Source:       7.7.7.50:8443
Destination:  3.3.3.7:51514
```

### Translation Table

| Field | Before NAT | After NAT |
|---|---|---|
| Source IP | `10.1.1.5` | `7.7.7.50` |
| Source TCP port | `443` | `8443` |
| Destination IP | `3.3.3.7` | `3.3.3.7` |
| Destination TCP port | `51514` | `51514` |

The outside client sees the reply coming from the same endpoint it contacted:

```text
7.7.7.50:8443
```

It does not see the internal server address or port.

**Key point:** Although the command is called `ip nat inside source static`, it translates the **destination** on outside-to-inside packets and the **source** on inside-to-outside packets.

This is the same directional principle as ordinary static inside source NAT.

---

## Static PAT Is Not Restricted to Outside-initiated traffic

The mapping is bidirectional, but the configured protocol and port still matter.

For example, suppose LH5 itself initiates a TCP connection using source port `443`:

```text
Source:       10.1.1.5:443
Destination:  3.3.3.7:5000
```

Because the source matches the inside-local endpoint in the static PAT mapping, Local_Router can translate it to:

```text
Source:       7.7.7.50:8443
Destination:  3.3.3.7:5000
```

However, ordinary client connections typically use ephemeral source ports.

Suppose LH5 initiates a connection to an outside HTTPS server:

```text
Source:       10.1.1.5:51000
Destination:  3.3.3.7:443
```

The static PAT mapping does **not** match:

```text
TCP 10.1.1.5:443 <-> 7.7.7.50:8443
```

The packet's **source port is 51000**, not `443`.

It does not matter that the packet's destination port is `443`.

A separate NAT rule, such as dynamic PAT overload, can translate this connection.

**Static inside source PAT matches the inside host's local source port when traffic travels from inside to outside, not the destination port of an outside service.**

---

## Port Preservation vs. Port Translation

Static PAT does not require the local and global ports to be different.

### Port Preservation

Suppose the internal HTTPS server listens on TCP port `443`, and outside clients should also use TCP port `443`.

Configure:

```text
R1(config)# ip nat inside source static tcp 10.1.1.5 443 7.7.7.50 443
```

Mapping:

```text
Inside local:    10.1.1.5:443
Inside global:   7.7.7.50:443
```

The IP address changes, but the port remains `443`.

This is still **static PAT**, because the mapping explicitly identifies TCP port `443`.

### Port Translation

To allow outside clients to connect using TCP port `8443` while the internal server continues listening on `443`:

```text
R1(config)# ip nat inside source static tcp 10.1.1.5 443 7.7.7.50 8443
```

Mapping:

```text
Inside local:    10.1.1.5:443
Inside global:   7.7.7.50:8443
```

This changes both the IP address and port number.

### Examples

| Inside local | Inside global | Purpose |
|---|---|---|
| `10.1.1.5:80` | `7.7.7.50:80` | HTTP with port preservation |
| `10.1.1.5:443` | `7.7.7.50:8443` | HTTPS with port translation |
| `10.1.1.5:22` | `7.7.7.50:2222` | SSH with port translation |
| `10.1.1.6:3389` | `7.7.7.50:13389` | RDP with port translation |

The port number does not determine the application by itself. Static PAT translates the packet headers; it does not automatically convert one application protocol into another.

For example, forwarding TCP port `8443` to TCP port `443` does not itself provide HTTPS. The inside server must actually run an appropriate HTTPS service.

---

## TCP vs. UDP Static PAT

Static PAT must specify the transport protocol.

For example:

```text
R1(config)# ip nat inside source static tcp 10.1.1.5 53 7.7.7.50 53
```

This translates **TCP port 53**, not UDP port 53.

To translate UDP port 53:

```text
R1(config)# ip nat inside source static udp 10.1.1.5 53 7.7.7.50 53
```

For an inside DNS server supporting both protocols, configure both mappings:

```text
R1(config)# ip nat inside source static tcp 10.1.1.5 53 7.7.7.50 53
R1(config)# ip nat inside source static udp 10.1.1.5 53 7.7.7.50 53
```

The mappings do not conflict simply because the numeric port is identical.

TCP port `53` and UDP port `53` are separate transport-layer endpoints.

### Can Static PAT Translate TCP to UDP?

No.

The `tcp` and `udp` keywords select the protocol to match. Static PAT can change the IP address and port number, but it does not convert TCP traffic into UDP traffic or vice versa.

### Does a TCP/UDP Static PAT Rule Translate ICMP?

No.

For example:

```text
R1(config)# ip nat inside source static tcp 10.1.1.5 443 7.7.7.50 8443
```

An ICMP Echo Request sent to `7.7.7.50` does not match this TCP static PAT mapping.

Therefore, the following tests have different meanings:

```text
RH7# ping 7.7.7.50
```

This does not test whether the TCP port-forwarding rule works.

```text
RH7# telnet 7.7.7.50 8443
```

This can be used as a basic TCP connection test if RH7 is an IOS device.

The inside server must be listening on the translated TCP port for the connection to succeed. An HTTPS service might accept the TCP connection but require TLS before returning application data.

---

## Multiple Static PAT Mappings Using One Global IP Address

One advantage of static PAT is that **multiple inside hosts can share a single inside global IP address**, provided their mapped external endpoints are distinguishable.

Suppose we want to publish the following services:

| Inside server | Local service | Global endpoint |
|---|---|---|
| LH5 (`10.1.1.5`) | TCP 443 (HTTPS) | `7.7.7.50:8443` |
| LH6 (`10.1.1.6`) | TCP 22 (SSH) | `7.7.7.50:2222` |
| LH6 (`10.1.1.6`) | UDP 53 (DNS) | `7.7.7.50:53` |

Configure:

```text
Local_Router(config)# ip nat inside source static tcp 10.1.1.5 443 7.7.7.50 8443 extendable
Local_Router(config)# ip nat inside source static tcp 10.1.1.6 22 7.7.7.50 2222 extendable
Local_Router(config)# ip nat inside source static udp 10.1.1.6 53 7.7.7.50 53 extendable
```

**Note:** IOS XE automatically appends the `extendable` keyword to static PAT commands on supported releases, even if it is omitted when entering the command.

This keyword allows multiple static PAT mappings to use the same inside local or inside global IP address, provided the mappings can be distinguished by their protocol and port information.

Unlike ordinary address-only static NAT, you generally do not need to explicitly configure `extendable` when configuring multiple static PAT mappings.

### Translation Table

A representative translation table would contain:

```text
Pro  Inside global         Inside local         Outside local    Outside global
tcp  7.7.7.50:8443        10.1.1.5:443        ---              ---
tcp  7.7.7.50:2222        10.1.1.6:22         ---              ---
udp  7.7.7.50:53          10.1.1.6:53         ---              ---
```

When RH7 connects to:

- `7.7.7.50:8443/TCP`, traffic is forwarded to `10.1.1.5:443`.
- `7.7.7.50:2222/TCP`, traffic is forwarded to `10.1.1.6:22`.
- `7.7.7.50:53/UDP`, traffic is forwarded to `10.1.1.6:53`.

The outside IP address is identical for all three mappings.

### Can Two Mappings Use the Same Global Port?

Two ordinary static PAT rules cannot independently assign the same **protocol, global IP address, and global port** to different inside destinations.

For example:

```text
ip nat inside source static tcp 10.1.1.5 443 7.7.7.50 8443
ip nat inside source static tcp 10.1.1.6 443 7.7.7.50 8443
```

These rules attempt to assign the same global TCP endpoint to two inside hosts.

The router cannot uniquely determine which inside server should receive an incoming connection based on that endpoint alone.

To distinguish the services, use different global ports or different global IP addresses.

For example:

```text
ip nat inside source static tcp 10.1.1.5 443 7.7.7.50 8443
ip nat inside source static tcp 10.1.1.6 443 7.7.7.50 9443
```

Now each inside server has a unique outside-facing TCP endpoint.

---

## Static PAT Using an Interface Address

Instead of specifying a literal inside global IP address, static PAT can use an interface's IP address.

The syntax is:

```text
Router(config)# ip nat inside source static tcp <inside-local-ip> <local-port> interface <interface> <global-port>
```

For example:

```text
Local_Router(config)# ip nat inside source static tcp 10.1.1.5 443 interface Ethernet0/1 8443
```

Local_Router's `Ethernet0/1` address is `7.7.7.2`.

Therefore, the mapping is effectively:

```text
TCP 10.1.1.5:443 <----> 7.7.7.2:8443
```

RH7 connects to:

```text
7.7.7.2:8443
```

Local_Router forwards the connection to:

```text
10.1.1.5:443
```

This example is an alternative to the previous mapping using `7.7.7.50`. Remove the previous overlapping or unnecessary lab rules before testing this version.

### Advantages

Using the interface address:

- Avoids manually specifying a separate inside global IP address.
- Can be useful when the outside interface's address is assigned dynamically.
- Avoids requiring the upstream router to resolve a separate translated IP address in this topology.

The outside interface's address still must be reachable from the outside client.

### Static PAT vs. Interface Overload

Do not confuse these commands:

**Static PAT:**

```text
ip nat inside source static tcp 10.1.1.5 443 interface Ethernet0/1 8443
```

**Dynamic PAT overload:**

```text
ip nat inside source list 10 interface Ethernet0/1 overload
```

The first creates a fixed TCP port mapping.

The second uses the interface address to create dynamic translations for traffic selected by ACL `10`.

Also consider services running on the NAT router itself. A static PAT rule using the outside interface address and a TCP/UDP port required by the router's own control or management plane may cause a conflict.

---

## Static PAT and Dynamic PAT Together

Static PAT is commonly used to publish inside services, while dynamic PAT provides general outbound connectivity.

For example, Local_Router might use:

```text
Local_Router(config)# access-list 10 permit 10.1.1.0 0.0.0.255
Local_Router(config)# ip nat inside source list 10 interface Ethernet0/1 overload
```

This translates ordinary matching outbound connections using `7.7.7.2`, with dynamically assigned source ports as necessary.

A separate static PAT rule can publish the HTTPS server:

```text
Local_Router(config)# ip nat inside source static tcp 10.1.1.5 443 interface Ethernet0/1 8443
```

The intended behavior is:

| Traffic | NAT mechanism |
|---|---|
| Outside client to `7.7.7.2:8443/TCP` | Static PAT to `10.1.1.5:443` |
| LH5 initiating an ordinary HTTPS connection with an ephemeral source port | Dynamic PAT overload |
| LH6 initiating an ordinary outbound connection | Dynamic PAT overload |

On supported IOS XE implementations, a static PAT entry can reserve a specific global endpoint while dynamic PAT uses other available ports.

However, NAT rule and translation conflicts must still be considered, particularly when static mappings and dynamic pools or interface overload use overlapping addresses.

The exact supported combinations and precedence should be verified on the target platform and release.

**Static PAT does not replace dynamic PAT for general outbound Internet connectivity.**

---

## Reachability of the Inside Global Address

Static PAT does not automatically guarantee that an outside packet can reach the NAT router.

The inside global address still needs to be reachable through ordinary IP routing and Layer 2 forwarding.

### Example 1: Inside Global Address on the Outside Subnet

In the lab:

```text
Local_Router E0/1:  7.7.7.2/24
ISP1 E0/1:          7.7.7.1/24
Inside global:      7.7.7.50
```

ISP1 considers `7.7.7.50` directly connected because it belongs to `7.7.7.0/24`.

It therefore attempts to resolve the address using ARP.

```text
ISP1: Who has 7.7.7.50?
                    |
                    v
             Local_Router
```

In IOS XE, NAT creates an IP alias that allows Local_Router to answer ARP Requests for a translated address.

Verify the behavior using:

```text
Local_Router# show ip aliases
```

On ISP1:

```text
ISP1# show ip arp 7.7.7.50
```

If ISP1 cannot resolve `7.7.7.50`, it cannot deliver the packet to Local_Router.

This can prevent outside-initiated communication even if the static PAT mapping appears in the NAT translation table.

**ARP and NAT translation are separate issues.** A valid static PAT entry does not prove that the upstream router can deliver packets to the translated address.

ARP behavior can vary with the NAT implementation, the translated address, and options such as `no-alias`.

### Example 2: Inside Global Address on a Separate Routed Prefix

A cleaner design is to use a translated address outside the directly connected transit subnet.

For example:

```text
Local_Router E0/1:  7.7.7.2/24
ISP1 E0/1:          7.7.7.1/24
Inside global:      203.0.113.50
```

Configure static PAT:

```text
Local_Router(config)# ip nat inside source static tcp 10.1.1.5 443 203.0.113.50 8443
```

On ISP1, configure a route toward the translated address:

```text
ISP1(config)# ip route 203.0.113.50 255.255.255.255 7.7.7.2
```

ISP1 now forwards packets for `203.0.113.50` toward the next hop `7.7.7.2`.

It resolves **7.7.7.2**, not `203.0.113.50`, using ARP.

```text
RH7
 |
 | Destination: 203.0.113.50:8443
 v
ISP1
 |
 | Route: 203.0.113.50/32 via 7.7.7.2
 v
Local_Router
 |
 | Static PAT:
 | 203.0.113.50:8443 -> 10.1.1.5:443
 v
LH5
```

The static PAT mapping handles address and port translation. The ISP1 static route handles delivery to the NAT router.

---

## Static PAT and Security

Static PAT is not a firewall.

Creating this rule:

```text
ip nat inside source static tcp 10.1.1.5 443 7.7.7.50 8443
```

does not restrict access to particular outside clients.

Any client that can reach `7.7.7.50:8443`, and is not blocked by an ACL or firewall, can potentially attempt to connect to LH5.

A network administrator should separately control:

- Which outside source addresses are permitted.
- Which ports and protocols are reachable.
- Whether the inside application requires authentication.
- Whether management services are exposed unnecessarily.

### ACL and NAT Order of Operations

The placement of an ACL matters because the packet's addresses and ports change as it passes through the NAT router.

In traditional IOS outside-to-inside processing, the simplified order is:

```text
Packet arrives on outside interface
             |
             v
        Input ACL
             |
             v
    Outside-to-inside NAT
             |
             v
        Route lookup
             |
             v
      Forward inside
```

An **inbound ACL on the outside interface** therefore ordinarily evaluates the original destination address and port:

```text
Destination: 7.7.7.50:8443
```

It does not evaluate the translated destination:

```text
Destination: 10.1.1.5:443
```

An ACL on the inside-facing output path can encounter the already translated values.

These are the traditional IOS processing relationships. Advanced forwarding features can introduce additional considerations.

See [NAT Packet Processing and Order of Operations](../order-of-operations.md).

---

## Verification and Troubleshooting

### Verify the Configuration

```text
Local_Router# show running-config | include ip nat
```

For the basic static PAT exercise, the relevant configuration should include:

```text
ip nat inside source static tcp 10.1.1.5 443 7.7.7.50 8443
```

Verify the inside and outside interfaces:

```text
Local_Router# show ip interface Ethernet0/0
Local_Router# show ip interface Ethernet0/1
```

### Verify the Translation Table

```text
Local_Router# show ip nat translations
```

Expected static mapping:

```text
Pro  Inside global         Inside local         Outside local    Outside global
tcp  7.7.7.50:8443        10.1.1.5:443        ---              ---
```

For additional details:

```text
Local_Router# show ip nat translations verbose
```

The static mapping normally remains present while configured, regardless of whether a client is connected.

Depending on the platform and implementation, additional per-flow translation entries may appear while traffic is active.

### Verify NAT Statistics

```text
Local_Router# show ip nat statistics
```

Check the number of translations, configured inside/outside interfaces, and translation statistics.

Do not assume that the presence of a configured static mapping proves it is forwarding traffic.

### Verify Routing

```text
Local_Router# show ip route 10.1.1.5
Local_Router# show ip route 3.3.3.7
```

Local_Router needs routes to the inside server and outside client.

On ISP1, verify reachability to the inside global address:

```text
ISP1# show ip route 7.7.7.50
ISP1# show ip arp 7.7.7.50
```

For a separate routed inside global address, verify the route toward Local_Router instead.

### Test the Service

From RH7, test the published TCP endpoint.

If using an IOS router as RH7:

```text
RH7# telnet 7.7.7.50 8443
```

If using a Linux host with curl:

```bash
curl -k https://7.7.7.50:8443/
```

The HTTPS test requires LH5 to be running a working HTTPS server on TCP port `443`.

The `-k` option tells curl to skip certificate validation, which is sometimes useful in an isolated lab with a self-signed certificate.

Do not rely only on an ICMP ping to test TCP static PAT.

### Debug NAT

In a controlled lab:

```text
Local_Router# debug ip nat
```

Generate matching traffic and examine whether translation occurs.

Disable debugging when finished:

```text
Local_Router# undebug all
```

Debug support and message format vary across platforms and forwarding paths.

### Common Problems

| Symptom | Possible cause |
|---|---|
| Static PAT entry is missing | Incorrect or rejected configuration |
| Outside client cannot connect | Routing, ARP, ACL, or application problem |
| Ping fails, but TCP port forwarding works | Expected: TCP static PAT does not translate ICMP |
| TCP works, but UDP fails | Only TCP static PAT was configured |
| Inside server receives packets but doesn't reply | Server application, host firewall, or return routing problem |
| Replies leave with unexpected source address/port | NAT mismatch, competing translation, or asymmetric path |
| One service works but another does not | Missing rule, wrong local/global port, or protocol mismatch |
| Another static PAT rule is rejected | Overlapping or conflicting local/global endpoints |
| Outside connection reaches router instead of inside service | Interface-address port conflict or platform-specific NAT handling |

---

## Key Takeaways

- **Static PAT** maps a particular protocol, inside local IP address, and local port to an inside global IP address and global port.
- The syntax is `ip nat inside source static tcp/udp <local-ip> <local-port> <global-ip> <global-port>`.
- The local and global ports can be identical or different.
- Static PAT mappings are configured in advance and can support outside-initiated connections.
- Outside-to-inside packets have their **destination address and port** translated.
- Inside-to-outside packets matching the mapping have their **source address and port** translated.
- Static PAT only matches the specified protocol and port, not every packet sent by the inside host.
- TCP and UDP require separate mappings, even when using the same numeric port.
- Multiple inside services can share a global IP address using different external endpoints.
- The `extendable` keyword supports certain additional or overlapping static mappings but does not eliminate endpoint conflicts.
- Interface-based static PAT can use the NAT outside interface's IP address.
- Static PAT and dynamic PAT overload are different mechanisms and can serve complementary purposes.
- Routing, ARP, ACLs, and the inside application must all work for port forwarding to succeed.

---

## References

- [Cisco IOS XE 17.x: Configuring NAT for IP Address Conservation](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-addressing/b-ip-addressing/m_iadnat-addr-consv-xe.html)
- [Cisco IOS IP Addressing Services Command Reference: ip nat inside source](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/ipaddr/command/ipaddr-cr-book/ipaddr-i3.html)
- [Cisco: Configure Network Address Translation](https://www.cisco.com/c/en/us/support/docs/ip/network-address-translation-nat/13772-12.html)
- [Cisco: Understand the NAT Order of Operation](https://www.cisco.com/c/en/us/support/docs/ip/network-address-translation-nat/6209-5.html)
- [Cisco: Troubleshoot Network Address Translation with Frequently Asked Questions](https://www.cisco.com/c/en/us/support/docs/ip/network-address-translation-nat/218012-troubleshoot-network-address-translation.html)
