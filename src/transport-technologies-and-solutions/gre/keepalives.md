# GRE Tunnel State and Keepalives

A **point-to-point GRE tunnel** is a logical interface, but it does not behave exactly like a physical point-to-point link.

By default, GRE is **stateless**. A router can bring its local GRE tunnel interface up without verifying that the remote tunnel endpoint exists or is able to receive packets.

**GRE keepalives** provide optional reachability detection. When configured, they allow a router to bring down the tunnel's line protocol if responses stop arriving.

This page covers how tunnel interfaces determine their state, how GRE keepalives work, how to configure them, and how tunnel state affects routing and failover.

---

## Basic Topology

Use the same topology from the earlier GRE pages:

```text
                    IPv4 underlay

R1 E0/0 --------------- ISP --------------- E0/0 R2
192.0.2.1                                  198.51.100.2
    |                                            |
    |                GRE tunnel                  |
    +============================================+
    |                                            |
Tunnel0                                       Tunnel0
10.0.12.1/30                                  10.0.12.2/30
    |                                            |
Loopback1                                      Loopback1
10.1.1.1/24                                    10.2.2.2/24
```

The tunnel endpoints are:

| Router | Tunnel source | Tunnel destination | Tunnel IP |
| --- | --- | --- | --- |
| R1 | `192.0.2.1` | `198.51.100.2` | `10.0.12.1/30` |
| R2 | `198.51.100.2` | `192.0.2.1` | `10.0.12.2/30` |

Assume the following base configuration is already present.

### R1

```cisco
interface Ethernet0/0
 ip address 192.0.2.1 255.255.255.252
 no shutdown
!
interface Tunnel0
 ip address 10.0.12.1 255.255.255.252
 tunnel source Ethernet0/0
 tunnel destination 198.51.100.2
!
ip route 198.51.100.2 255.255.255.255 192.0.2.2
```

### ISP

```cisco
interface Ethernet0/0
 ip address 192.0.2.2 255.255.255.252
 no shutdown
!
interface Ethernet0/1
 ip address 198.51.100.1 255.255.255.252
 no shutdown
```

### R2

```cisco
interface Ethernet0/0
 ip address 198.51.100.2 255.255.255.252
 no shutdown
!
interface Tunnel0
 ip address 10.0.12.2 255.255.255.252
 tunnel source Ethernet0/0
 tunnel destination 192.0.2.1
!
ip route 192.0.2.1 255.255.255.255 198.51.100.1
```

These are ordinary point-to-point GRE tunnels. `tunnel mode gre ip` is the default and does not need to be entered.

---

## GRE Tunnel Interface States

Use `show interfaces Tunnel0` to check the interface and line-protocol state.

| State | Meaning |
| --- | --- |
| `up/up` | The tunnel is administratively enabled and its local requirements are satisfied. With keepalives enabled, the keepalive check must also pass. |
| `up/down` | The interface is administratively enabled, but its line protocol is down. |
| `administratively down/down` | The tunnel interface has been shut down with `shutdown`. |

An `up/up` state **does not necessarily mean that traffic can reach the remote endpoint**.

This distinction is especially important when GRE keepalives are not configured.

### What Makes a GRE Tunnel Come Up?

Without keepalives, the normal requirements for a point-to-point GRE tunnel include:

1. The tunnel interface is administratively enabled.
2. A valid local tunnel source IP address is available, or the configured source interface is operational and has an appropriate IP address.
3. A tunnel destination is configured.
4. The routing table has a valid route to the tunnel destination that does **not** resolve through the tunnel itself.

The route to the tunnel destination can be a specific route or a default route.

```text
R1 tunnel destination: 198.51.100.2

R1 routing table:
198.51.100.2/32 via 192.0.2.2, Ethernet0/0

Result:
R1 can resolve the tunnel destination through the underlay.
```

The router checks that the destination is **routable**, not necessarily that it is **reachable**.

> A valid route to the remote endpoint is not proof that packets can actually traverse the underlay and return.

---

## The Problem with Stateless GRE

Suppose the link between the ISP and R2 fails, while R1's own Ethernet0/0 interface remains up.

```text
R1 ---------------- ISP ------X------ R2
 |                                      |
 +============= Tunnel0 ===============+
```

R1 still has this static route:

```cisco
ip route 198.51.100.2 255.255.255.255 192.0.2.2
```

From R1's perspective:

- Its tunnel source interface is up.
- Its routing table still contains a route toward the tunnel destination.
- No GRE keepalive mechanism is checking the remote endpoint.

Therefore, `Tunnel0` can remain `up/up` even though GRE packets cannot reach R2.

```text
R1# show interfaces Tunnel0
Tunnel0 is up, line protocol is up
```

A static route through the tunnel can also remain installed:

```cisco
ip route 10.2.2.0 255.255.255.0 Tunnel0
```

R1 can continue forwarding packets into `Tunnel0`, but the packets are lost in the underlay.

This is a **traffic black hole**.

---

## GRE Keepalives

GRE keepalives allow a tunnel endpoint to determine whether GRE traffic can travel to the remote endpoint and a response can return.

**Keepalives are disabled by default** on point-to-point GRE tunnels.

Enable them under the tunnel interface:

```cisco
R1(config)# interface Tunnel0
R1(config-if)# keepalive 10 3
```

Syntax:

```cisco
keepalive [seconds [retries]]
```

| Parameter | Purpose | Default when keepalives are enabled without arguments |
| --- | --- | --- |
| `seconds` | Interval between keepalive transmissions | 10 seconds |
| `retries` | Number of unanswered keepalives tolerated before the line protocol is brought down | 3 |

Examples:

```cisco
keepalive
```

Enables GRE keepalives with the default interval and retry count (10 seconds and 3 retries).

```cisco
keepalive 5 4
```

Sends keepalives every 5 seconds and brings down the line protocol after the configured number of unanswered keepalives.

```cisco
no keepalive
```

Disables tunnel keepalives.

Keepalive values do not have to match between tunnel endpoints.

---

## Configuring Keepalives on R1 and R2

To enable keepalives in both directions, configure them independently on each router.

### R1

```cisco
interface Tunnel0
 keepalive 5 3
```

### R2

```cisco
interface Tunnel0
 keepalive 5 3
```

Each router sends its own keepalive requests and monitors whether its requests result in responses.

```text
R1                                         R2
 |                                          |
 |----------- Keepalive request ----------->|
 |<---------- Keepalive response -----------|
 |                                          |
 |<---------- Keepalive request ------------|
 |----------- Keepalive response ---------->|
 |                                          |
```

These arrows illustrate the logical request/response behavior. The actual GRE keepalive packet structure is unusual and is explained below.

---

## How GRE Keepalive Packets Work

GRE does **not** use a negotiated keepalive session or a conventional request/response exchange in which the remote router must generate a new reply.

Instead, the sender **prebuilds the reply packet inside the keepalive request**.

The remote router only needs to perform ordinary GRE decapsulation and IP forwarding.

This is why a router can respond to GRE keepalives **without having keepalives enabled locally**.

### Step 1: R1 Builds the Response Packet

R1 first constructs a packet that is intended to travel **from R2 back to R1**:

```text
IPv4 header:
  Source:      198.51.100.2  (R2)
  Destination: 192.0.2.1     (R1)
  IP protocol: 47           (GRE)

GRE header:
  Protocol Type: 0x0000     (keepalive indicator)
```

There is no normal passenger IP packet after this inner GRE header.

Notice that the source and destination IPv4 addresses are **already reversed**. R1 creates the packet that it wants R2 to return.

### Step 2: R1 Encapsulates the Response in GRE

R1 sends the prebuilt response through `Tunnel0`, adding an outer GRE header and outer IPv4 header:

```text
+-----------------------------------------------+
| Outer IPv4 header                             |
| Source: 192.0.2.1                             |
| Destination: 198.51.100.2                     |
| Protocol: 47 (GRE)                            |
+-----------------------------------------------+
| Outer GRE header                              |
| Protocol Type: 0x0800 (IPv4)                  |
+-----------------------------------------------+
| Inner IPv4 header                             |
| Source: 198.51.100.2                          |
| Destination: 192.0.2.1                        |
| Protocol: 47 (GRE)                            |
+-----------------------------------------------+
| Inner GRE header                              |
| Protocol Type: 0x0000 (keepalive)             |
+-----------------------------------------------+
```

The keepalive request is therefore **a GRE packet carrying another IPv4/GRE packet**.

The outer packet goes from R1 to R2. The embedded packet is already addressed from R2 back to R1.

### Step 3: R2 Decapsulates and Forwards the Packet

When R2 receives the outer GRE packet, it removes the outer IPv4 and GRE headers.

The remaining packet is:

```text
IPv4 header:
  Source:      198.51.100.2
  Destination: 192.0.2.1
  Protocol:    47

GRE header:
  Protocol Type: 0x0000
```

R2 performs an IP forwarding lookup for `192.0.2.1` and forwards that packet toward R1 through the **underlay**.

R2 does not need to construct a separate keepalive response or have `keepalive` configured.

### Step 4: R1 Receives the Response

R1 receives the returned GRE packet, recognizes the GRE Protocol Type value of `0x0000`, and treats it as a keepalive response.

R1 resets its missed-keepalive counter and discards the keepalive packet.

The complete process is:

```text
R1                                     R2
 |                                      |
 |-- Outer GRE packet ----------------->|
 |   (includes prebuilt reply)          |
 |                                      |
 |                     Remove outer GRE |
 |                     and IPv4 headers |
 |                                      |
 |<-- Prebuilt IPv4/GRE reply ----------|
 |                                      |
 | Recognize GRE PT 0x0000              |
 | Reset keepalive counter              |
 |                                      |
```

**Important:** The returned keepalive packet is forwarded using the **underlay routing table**, not sent back through the GRE overlay as ordinary passenger traffic.

---

## Are Keepalives Required on Both Tunnel Endpoints?

**No.** Configuring GRE keepalives on only one endpoint is supported.

For example:

### R1

```cisco
interface Tunnel0
 keepalive 5 3
```

### R2

```cisco
interface Tunnel0
 no keepalive
```

R1 sends the keepalives. R2 decapsulates them and forwards the prebuilt responses without originating keepalives itself.

However, the **detection is one-sided**:

| Condition | R1 | R2 |
| --- | --- | --- |
| Normal connectivity | Keepalives succeed; tunnel stays up | Tunnel stays up based on local checks |
| GRE path fails, but local tunnel requirements remain valid | Missed keepalives can bring line protocol down | Tunnel may remain `up/up` |

If you want both routers to detect a failure using GRE keepalives, configure them on **both** ends.

### Do the Intervals Have to Match?

No. For example, R1 can use `keepalive 5 3` and R2 can use `keepalive 10 5`.

Each router independently determines whether it has received responses to its own keepalive requests.

---

## Keepalive Failure and Tunnel State

Assume R1 has:

```cisco
interface Tunnel0
 keepalive 5 3
```

If R1 stops receiving keepalive responses:

1. R1 continues transmitting keepalives at the configured interval.
2. The missed-keepalive count increases.
3. After the configured retry threshold is reached, R1 brings the tunnel's **line protocol down**.
4. The tunnel stops forwarding ordinary data traffic, although keepalive processing continues.
5. When a valid response arrives again, R1 can bring the line protocol back up.

Typical state before failure:

```text
R1# show interfaces Tunnel0
Tunnel0 is up, line protocol is up
```

Typical state after keepalive failure:

```text
R1# show interfaces Tunnel0
Tunnel0 is up, line protocol is down
```

The tunnel interface has **not** been administratively shut down. It is still available to send keepalives and detect recovery.

For a `keepalive 5 3` configuration, the detection time is roughly on the order of three 5-second intervals. Do not treat this as an exact 15-second guarantee because detection depends on when the failure occurs relative to transmissions and on platform processing.

---

## Effect on Static and Dynamic Routing

A change in tunnel line-protocol state can affect the routing table.

### Static Routes

Assume R1 has a route to R2's LAN:

```cisco
ip route 10.2.2.0 255.255.255.0 Tunnel0
```

While `Tunnel0` is up, the static route can be installed and used.

When GRE keepalive failure brings the tunnel line protocol down, the static route that depends on `Tunnel0` is no longer a valid forwarding path and is removed from the active routing table.

```text
Tunnel0 up/up
  -> Static route through Tunnel0 can be installed

Tunnel0 up/down
  -> Static route through Tunnel0 is no longer active
```

If a valid **floating static route** exists through another path, it can become active after the primary route disappears.

GRE keepalives therefore help prevent static routes from continuing to forward traffic into a failed tunnel.

### Dynamic Routing Protocols

OSPF and EIGRP can operate over the tunnel. When the tunnel's line protocol goes down, the corresponding routing-protocol adjacency also goes down and the affected routes can be withdrawn.

A GRE keepalive is separate from a routing protocol's own Hello/hold timer mechanism:

| Mechanism | What it checks |
| --- | --- |
| GRE keepalive | Whether a GRE keepalive request produces a returned GRE response |
| OSPF Hello/Dead timers | Whether the OSPF neighbor remains responsive |
| EIGRP Hello/Hold timers | Whether the EIGRP neighbor remains responsive |
| BGP keepalive/Hold timer | Whether the BGP session is alive |

It is possible for a routing-protocol neighbor to go down **before** GRE keepalives declare the tunnel down, or vice versa.

---

## Failure Scenarios

### 1. Local Tunnel Source Interface Goes Down

If R1 uses Ethernet0/0 as its tunnel source and Ethernet0/0 goes down, `Tunnel0` loses its valid source interface.

The line protocol can go down immediately due to the local dependency. Keepalive expiration is **not required**.

### 2. Route to the Remote Tunnel Destination Disappears

If R1 has no valid route to `198.51.100.2`, including no usable default route, the tunnel cannot resolve its destination.

The line protocol goes down because of the missing underlay route.

### 3. Tunnel Destination Resolves Through the Tunnel

If R1's route to `198.51.100.2` resolves through `Tunnel0`, GRE would have to use the tunnel to send its own outer packets.

This is **recursive routing** and can bring the tunnel down.

```text
To send GRE:
  Need a route to 198.51.100.2

Route to 198.51.100.2:
  Use Tunnel0

But Tunnel0 requires GRE delivery to 198.51.100.2
  -> Recursive dependency
```

Maintain an independent underlay route to each tunnel destination.

### 4. Remote Router or Intermediate Network Fails

R1's source interface and underlay route may remain valid even though packets are lost farther along the path.

- Without keepalives: `Tunnel0` can remain `up/up`.
- With keepalives: missed responses can cause `Tunnel0` to become `up/down`.

This is the principal reason for configuring GRE keepalives.

### 5. GRE Is Filtered

GRE over IPv4 uses **IP protocol 47**, not TCP or UDP port 47.

If an ACL or firewall blocks GRE packets or their return traffic, keepalives can fail even if ordinary IP reachability tests such as ping succeed.

An ICMP ping to the remote tunnel destination does **not** prove that GRE is permitted across the network.

---

## Keepalive Limitations and Considerations

### Point-to-Point GRE vs. Multipoint GRE

GRE tunnel keepalives are supported for **point-to-point GRE tunnels**.

For multipoint GRE (`tunnel mode gre multipoint`), configuring the ordinary `keepalive` command does not provide effective GRE keepalive monitoring.

### GRE over IPsec

Cisco documents that GRE keepalives are **not supported with `tunnel protection ipsec profile`**.

The prebuilt response is returned through the underlay rather than being generated as ordinary outbound tunnel traffic. Consequently, the response can bypass tunnel-interface IPsec protection and be dropped when encrypted packets are required.

Do not assume that adding `keepalive` to a GRE-over-IPsec tunnel will provide valid failure detection. IPsec SA state, routing-protocol detection, and other platform-supported mechanisms may be more appropriate.

GRE protected by a **crypto map on the physical interface** has different packet-processing behavior; Cisco documents cases where the returning packet is encrypted successfully. The placement of IPsec protection matters.

### GRE Tunnels with VRFs

When the tunnel's **transport VRF** (`tunnel vrf`) differs from its **inner/overlay VRF** (`vrf forwarding`), GRE keepalives can fail because the decapsulated, prebuilt reply is routed in a different VRF from the underlay.

This can produce an `up/down` tunnel even when ordinary traffic would otherwise work.

See [GRE Tunnels and VRFs](vrf.md) for the distinction between transport and overlay VRFs.

### Other Operational Considerations

- Keepalive success indicates that the keepalive round trip works. It does not guarantee that all passenger traffic or routing protocols are permitted.
- One-sided keepalives can produce different tunnel states on R1 and R2.
- Keepalive timers on the endpoints are independent and need not match.
- Very aggressive timers can produce false failures when the device or network is congested.
- GRE keepalive behavior should not be assumed to apply identically to every other tunnel mode.

---

## Verification and Troubleshooting

### Check Tunnel State

```cisco
R1# show interfaces Tunnel0
R1# show ip interface brief | include Tunnel0
```

Look for whether the tunnel is `up/up` or `up/down`, the configured tunnel source and destination, and whether keepalives are enabled.

### Check the Configuration

```cisco
R1# show running-config interface Tunnel0
```

Confirm the tunnel source, tunnel destination, and `keepalive` setting.

### Check Underlay Routing

```cisco
R1# show ip route 198.51.100.2
R1# show ip cef 198.51.100.2
R1# ping 198.51.100.2 source 192.0.2.1
```

Check that the route to `198.51.100.2` resolves toward the ISP, not through `Tunnel0`.

Keep in mind that successful ICMP testing does not prove that IP protocol 47 is permitted.

### Check Overlay Connectivity

```cisco
R1# ping 10.0.12.2 source 10.0.12.1
R1# ping 10.2.2.2 source 10.1.1.1
R1# show ip route 10.2.2.0
```

The first ping tests the tunnel interface addresses. The second tests traffic between the simulated LANs and requires the appropriate overlay routes on both routers.

### Debug GRE Keepalives

Cisco provides the following debugging command:

```cisco
R1# debug tunnel keepalive
```

To stop debugging:

```cisco
R1# undebug all
```

The debug can help confirm whether the router is transmitting keepalives and whether its keepalive counter continues to increase.

Use debug commands carefully on production routers.

---

## Key Points

- GRE is **stateless** by default. `up/up` does not prove the remote endpoint is reachable.
- A point-to-point GRE tunnel normally needs a valid tunnel source and a nonrecursive route to the tunnel destination to bring up its line protocol.
- The remote tunnel endpoint need not actually be reachable for the tunnel to show `up/up` when keepalives are disabled.
- GRE keepalives are **disabled by default**.
- Configure them with `keepalive [seconds [retries]]` under the tunnel interface.
- `keepalive` enables the default **10-second interval** and **3 retries**.
- GRE keepalives **do not have to be configured on both ends**.
- The sender embeds a **prebuilt return packet** inside the outgoing GRE keepalive request.
- The remote router decapsulates the request and forwards the prebuilt reply without needing its own keepalive configuration.
- The keepalive reply is a GRE packet with **Protocol Type `0x0000`**.
- Keepalive timers on opposite endpoints are **independent** and do not have to match.
- When keepalives fail, the configured router brings its tunnel line protocol down but continues sending keepalives to detect recovery.
- Interface-dependent static routes can be removed when the tunnel line protocol goes down, allowing alternative routes to be installed.
- GRE keepalives do not work as a failure-detection mechanism on multipoint GRE tunnels.
- GRE keepalives have important limitations with IPsec tunnel protection and mismatched transport/overlay VRFs.

---

## References

- [Cisco: Understand GRE Tunnel Keepalives](https://www.cisco.com/c/en/us/support/docs/ip/generic-routing-encapsulation-gre/118370-technote-gre-00.html)
- [Cisco: Determine What Impacts GRE Tunnel Interface States](https://www.cisco.com/c/en/us/support/docs/ip/generic-routing-encapsulation-gre/118361-technote-gre-00.html)
- [Cisco: How GRE Keepalives Work](https://www.cisco.com/c/en/us/support/docs/ip/generic-routing-encapsulation-gre/63760-gre-keepalives-63760.html)
- [Cisco: IP Routing Configuration Guide, Implementing Tunnels](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/ip-routing/b-ip-routing/m_ir-impl-tun-xe.html)
