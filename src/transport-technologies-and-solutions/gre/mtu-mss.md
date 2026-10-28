# GRE MTU, Fragmentation, and MSS

GRE adds headers to the original packet. If the resulting packet is larger than the **underlay path MTU**, it may be fragmented or dropped.

Three mechanisms are important when troubleshooting packet-size problems over GRE:

- **MTU** limits the size of an IP packet that can be forwarded without fragmentation.
- **Fragmentation** splits an IPv4 packet into smaller fragments when necessary and permitted.
- **TCP MSS** limits the amount of TCP application data in an individual TCP segment.

**Path MTU Discovery (PMTUD)** allows endpoints to learn a smaller usable packet size instead of relying on fragmentation.

This page focuses on point-to-point GRE with an **IPv4 underlay**. The main examples use IPv4 passenger traffic; IPv6-over-IPv4 GRE is covered near the end.

---

## Basic Topology

Use the same R1, ISP, and R2 topology as the previous GRE pages:

```text
                        IPv4 underlay

R1 E0/0 ------------------- ISP ------------------- E0/0 R2
192.0.2.1                                       198.51.100.2
    |                                                 |
    |                 GRE overlay                     |
    +=================================================+
    |                                                 |
Tunnel0                                            Tunnel0
10.0.12.1/30                                       10.0.12.2/30
    |                                                 |
Loopback1                                           Loopback1
10.1.1.1/24                                         10.2.2.2/24
```

The tunnel endpoints are `192.0.2.1` and `198.51.100.2`. Their tunnel interface addresses are `10.0.12.1/30` and `10.0.12.2/30`.

Assume the following is already configured on both routers, including the underlay static routes from the earlier lab.

### R1

```cisco
interface Tunnel0
 ip address 10.0.12.1 255.255.255.252
 tunnel source Ethernet0/0
 tunnel destination 198.51.100.2
!
ip route 198.51.100.2 255.255.255.255 192.0.2.2
ip route 10.2.2.0 255.255.255.0 10.0.12.2
```

### R2

```cisco
interface Tunnel0
 ip address 10.0.12.2 255.255.255.252
 tunnel source Ethernet0/0
 tunnel destination 192.0.2.1
!
ip route 192.0.2.1 255.255.255.255 198.51.100.1
ip route 10.1.1.0 255.255.255.0 10.0.12.1
```

The default tunnel mode is `gre ip`, so there is no need to configure it explicitly.

---

## MTU and GRE Encapsulation Overhead

**MTU** (Maximum Transmission Unit) is the maximum Layer 3 packet size that an interface or path can carry in one piece.

For basic GRE over IPv4, each encapsulated packet has these headers:

```text
+-------------------------+
| Outer IPv4 header       | 20 bytes
+-------------------------+
| GRE header              |  4 bytes
+-------------------------+
| Original IP packet      | Variable
+-------------------------+
```

The basic GRE-over-IPv4 overhead is **24 bytes**:

```text
Outer IPv4 header: 20 bytes
GRE header:         4 bytes
---------------------------
Total overhead:    24 bytes
```

If the underlay supports a 1500-byte IP packet, the largest passenger IP packet that fits without fragmentation is:

```text
Underlay MTU - GRE overhead = Maximum passenger IP packet

1500 - 24 = 1476 bytes
```

A 1476-byte original IP packet becomes a 1500-byte GRE packet.

A 1500-byte original IP packet becomes a **1524-byte GRE packet**, which cannot cross a 1500-byte underlay as a single IPv4 datagram.

> These calculations refer to Layer 3 IP packet sizes. Ethernet headers and FCS are not included in the 1500-byte IP MTU.

### Optional GRE Fields Increase Overhead

The basic GRE header is 4 bytes, but optional fields add overhead:

| GRE configuration | GRE header | Outer IPv4 + GRE overhead | Maximum passenger packet (1500-byte underlay) |
| --- | ---: | ---: | ---: |
| Basic GRE | 4 bytes | **24 bytes** | **1476 bytes** |
| `tunnel key` | 8 bytes | 28 bytes | 1472 bytes |
| `tunnel sequence-datagrams` | 8 bytes | 28 bytes | 1472 bytes |
| `tunnel checksum` | 8 bytes | 28 bytes | 1472 bytes |
| Key + sequence | 12 bytes | 32 bytes | 1468 bytes |
| Key + sequence + checksum | 16 bytes | 36 bytes | 1464 bytes |

Each optional GRE field shown above adds **4 bytes**. These figures assume an outer IPv4 header without options and no additional encapsulation, such as IPsec.

For comparison, GRE using an **IPv6 outer header** has a basic overhead of **44 bytes** (40-byte IPv6 header + 4-byte GRE header).

---

## Physical MTU vs. Tunnel IP MTU vs. Path MTU

These values are related but are not the same.

| MTU | Meaning |
| --- | --- |
| **Physical interface MTU** | Maximum packet size supported on a local physical interface. |
| **Tunnel IP MTU** | Maximum original IPv4 packet size the router attempts to send through the tunnel before considering inner fragmentation or an ICMP error. |
| **Underlay path MTU** | Smallest MTU on the complete path between the GRE tunnel source and destination. |

For a basic GRE-over-IPv4 tunnel with a 1500-byte physical MTU, Cisco IOS calculates a default **tunnel IPv4 MTU of 1476 bytes**.

```text
Physical IP MTU: 1500
GRE overhead:      24
Tunnel IPv4 MTU: 1476
```

The locally calculated tunnel MTU does **not** necessarily account for a smaller link somewhere in the ISP network.

Example:

```text
R1 -------- ISP1 -------- ISP2 -------- R2
   1500          1400           1500

Underlay path MTU = 1400
```

Even if R1's local outbound interface has an MTU of 1500, the largest GRE passenger packet that fits this particular path without fragmentation is:

```text
1400 - 24 = 1376 bytes
```

A 1476-byte passenger packet creates a 1500-byte outer GRE packet. It cannot cross the 1400-byte link without **outer IPv4 fragmentation**, unless the packet size is reduced first.

### View the Tunnel IP MTU

```cisco
R1# show ip interface Tunnel0
```

Look for the IPv4 MTU reported for the tunnel interface.

Also inspect the physical and tunnel interfaces:

```cisco
R1# show interfaces Ethernet0/0
R1# show interfaces Tunnel0
```

`show interfaces Tunnel0` and `show ip interface Tunnel0` may display different MTU values because one reports an interface-level MTU and the other reports the **IPv4 protocol MTU**. For GRE packet-size calculations, check the IP MTU.

---

## Configure the Tunnel IP MTU

Use `ip mtu` under the tunnel interface to set its IPv4 MTU explicitly.

For basic GRE over a 1500-byte underlay:

```cisco
R1(config)# interface Tunnel0
R1(config-if)# ip mtu 1476
```

On R2:

```cisco
R2(config)# interface Tunnel0
R2(config-if)# ip mtu 1476
```

Because 1476 is commonly the automatically calculated value in this scenario, the explicit command is usually unnecessary.

For an underlay path with a **1400-byte MTU**, the corresponding maximum passenger IP packet is 1376 bytes:

```cisco
R1(config-if)# ip mtu 1376
```

`ip mtu` changes the tunnel's IPv4 packet-size limit. It does **not** change the underlay's MTU, and it does **not** prevent the outer GRE packet from encountering an even smaller MTU farther along the path.

---

## IPv4 Fragmentation Before GRE Encapsulation

Consider a **1500-byte original IPv4 packet** entering R1's tunnel, with an IPv4 MTU of 1476 bytes.

If the original packet's **DF (Don't Fragment) bit is 0**, R1 can fragment the original IPv4 packet **before** adding GRE encapsulation.

### Example: Inner IPv4 Fragmentation

```text
Original IPv4 packet: 1500 bytes
Tunnel IPv4 MTU:      1476 bytes
GRE overhead:           24 bytes
Physical MTU:         1500 bytes
DF bit:                   0
```

R1 creates two inner IPv4 fragments:

| Inner fragment | Inner IP header | Inner IP payload | Inner fragment size |
| --- | ---: | ---: | ---: |
| First | 20 bytes | 1456 bytes | **1476 bytes** |
| Second | 20 bytes | 24 bytes | **44 bytes** |

R1 then adds a **separate outer IPv4 header and GRE header to each inner fragment**:

```text
First GRE packet:

20-byte outer IPv4 header
 4-byte GRE header
1476-byte inner IPv4 fragment
-----------------------------
1500 bytes

Second GRE packet:

20-byte outer IPv4 header
 4-byte GRE header
  44-byte inner IPv4 fragment
-----------------------------
  68 bytes
```

Both GRE packets fit the 1500-byte underlay.

```text
Original IPv4 packet: 1500 bytes
              |
              | Fragment before GRE encapsulation
              v
     +-----------------+----------------+
     | 1476-byte frag  | 44-byte frag   |
     +-----------------+----------------+
              |                 |
              | Add 24 bytes    | Add 24 bytes
              v                 v
     1500-byte GRE packet   68-byte GRE packet
              |                 |
              +--------+--------+
                       |
                       v
                  IPv4 underlay
```

R2 decapsulates the packets and forwards the **inner IPv4 fragments** toward the destination. The **final IPv4 destination** reassembles them into the original packet.

This is **fragmentation before GRE encapsulation**.

### What If the Original IPv4 DF Bit Is Set?

If the original packet has **DF = 1** and exceeds the tunnel IPv4 MTU, R1 cannot fragment it.

Instead, R1:

1. Drops the original IPv4 packet.
2. Sends an ICMP **Destination Unreachable, Fragmentation Needed** message to its original IPv4 source.
3. Reports an MTU of 1476 bytes in this example.

The ICMP message is **Type 3, Code 4**.

```text
1500-byte inner IPv4 packet (DF = 1)
                    |
                    v
               R1 Tunnel0
               IP MTU 1476
                    |
                    v
                  DROP
                    |
                    v
    ICMP Type 3, Code 4 to original source
                  MTU 1476
```

A source using PMTUD can reduce its packet size and retransmit.

---

## IPv4 Fragmentation After GRE Encapsulation

**Outer IPv4 fragmentation** is different. It happens to the already-encapsulated GRE packet as it crosses the IPv4 underlay.

Assume R1 sends a GRE packet of **1500 bytes**, but an intermediate ISP interface has an MTU of **1400 bytes**.

If the **outer IPv4 DF bit is 0**, an underlay router can fragment the outer IPv4 packet:

```text
R1                      ISP                      R2
 |                       |                        |
 | 1500-byte GRE packet  |                        |
 +---------------------->|                        |
 |                       | 1400-byte path MTU     |
 |                       |                        |
 |                       | Fragment OUTER IPv4    |
 |                       | packet                 |
 |                       +----------------------->|
 |                       |  Fragment 1            |
 |                       +----------------------->|
 |                       |  Fragment 2            |
 |                       |                        |
 |                       |               Reassemble outer
 |                       |               IPv4 packet, then
 |                       |               remove GRE header
```

In this case:

- The **underlay router** fragments the outer IPv4 packet.
- Only the first outer fragment contains the GRE header.
- The **GRE tunnel destination** must reassemble the complete outer IPv4 datagram before decapsulating it.
- The inner IP packet has **not necessarily** been fragmented.

This distinction is especially important for troubleshooting. Outer fragmentation introduces reassembly work on the GRE endpoint, while inner fragmentation may require reassembly by the final passenger destination.

If the **outer DF bit is 1**, an underlay router cannot fragment the GRE packet. It drops the packet and sends ICMP Type 3, Code 4 toward the **GRE source**, rather than directly to the original host.

### Comparison

| Behavior | Inner IPv4 fragmentation | Outer IPv4 fragmentation |
| --- | --- | --- |
| Occurs | **Before** GRE encapsulation | **After** GRE encapsulation |
| Trigger | Passenger packet exceeds tunnel IP MTU | Encapsulated packet exceeds underlay link MTU |
| Fragmented header | Original IPv4 header | Outer IPv4 header |
| Reassembly | Final destination of the original IPv4 packet | GRE tunnel endpoint |
| Controlled by | Inner IPv4 DF bit | Outer IPv4 DF bit |

---

## Tunnel Path MTU Discovery

By default, GRE over IPv4 does **not** perform tunnel PMTUD. The **outer IPv4 DF bit is normally cleared**, even when the original IPv4 packet has DF set.

This permits an intermediate underlay router to fragment the outer GRE packet if necessary.

Enable GRE tunnel PMTUD with:

```cisco
R1(config)# interface Tunnel0
R1(config-if)# tunnel path-mtu-discovery
```

The same command can be configured on R2 to enable discovery in the reverse direction:

```cisco
R2(config)# interface Tunnel0
R2(config-if)# tunnel path-mtu-discovery
```

### How GRE Tunnel PMTUD Works

With `tunnel path-mtu-discovery` enabled, the router **copies the inner IPv4 DF bit into the outer IPv4 header**.

For a packet with DF set:

1. R1 encapsulates the original packet in GRE and IPv4, with outer DF set.
2. If an intermediate underlay link cannot carry the packet, the intermediate router drops it.
3. The intermediate router sends ICMP Type 3, Code 4 to R1, reporting the smaller underlay MTU.
4. R1 uses the reported path MTU to derive a smaller effective tunnel MTU, accounting for GRE overhead.
5. If a later original packet exceeds that effective MTU and has DF set, R1 drops it and sends an appropriate ICMP error to the original source.
6. The original source can reduce its packet size to match the smaller usable path MTU.

For example:

```text
Actual underlay path MTU: 1400 bytes
GRE overhead:              24 bytes
Effective passenger MTU: 1376 bytes
```

The tunnel ingress effectively participates in **two levels of PMTUD**:

```text
Original host <---- PMTUD ----> R1 tunnel ingress
                                       |
                                       |
                                       v
                        GRE tunnel source <---- PMTUD ----> underlay
```

An effective path MTU learned dynamically may be maintained internally and might not appear as a changed configured MTU in `show ip interface Tunnel0`.

### DF Bit Summary

| Setting | Outer IPv4 DF behavior |
| --- | --- |
| Default GRE (no tunnel PMTUD) | Normally cleared, allowing outer IPv4 fragmentation |
| `tunnel path-mtu-discovery` | Copies the inner IPv4 DF bit into the outer IPv4 header |

Tunnel PMTUD depends on ICMP **Fragmentation Needed** messages being able to reach the tunnel source. Filtering those ICMP messages can cause a **PMTUD black hole**, where small packets work but larger packets fail.

The optional `age-timer` and `min-mtu` arguments can influence tunnel PMTUD behavior:

```cisco
R1(config-if)# tunnel path-mtu-discovery age-timer 10
R1(config-if)# tunnel path-mtu-discovery age-timer infinite
R1(config-if)# tunnel path-mtu-discovery min-mtu 1200
```

The age timer controls when a learned MTU is reset so that a larger path MTU can be discovered later. Support and accepted ranges for the optional arguments depend on the platform.

> `tunnel path-mtu-discovery` is **not** the same command as `ip tcp path-mtu-discovery`, which controls PMTUD for IPv4 TCP connections originated by the router itself.

---

## TCP Maximum Segment Size (MSS)

**TCP MSS** is the maximum number of **TCP application-data bytes** that can be carried in a TCP segment.

It does **not** include the IP header or TCP header.

For IPv4 and TCP without options:

```text
IPv4 packet size = IPv4 header + TCP header + TCP data

                 = 20 bytes + 20 bytes + MSS
```

For a GRE tunnel with a **1476-byte tunnel IPv4 MTU**:

```text
Tunnel IPv4 MTU: 1476 bytes
IPv4 header:       20 bytes
TCP header:        20 bytes
--------------------------
Maximum TCP MSS: 1436 bytes
```

A TCP segment with 1436 bytes of data occupies 1476 bytes as an IPv4 packet and 1500 bytes after basic GRE-over-IPv4 encapsulation.

```text
20-byte outer IPv4 header
 4-byte GRE header
20-byte inner IPv4 header
20-byte TCP header
1436 bytes TCP data
--------------------------
1500-byte GRE packet
```

### MTU and MSS Reference Table

Assuming standard 20-byte IPv4 and TCP headers and a 1500-byte underlay:

| GRE overhead | Maximum passenger IPv4 MTU | Maximum IPv4 TCP MSS |
| ---: | ---: | ---: |
| 24 bytes (basic GRE) | 1476 | **1436** |
| 28 bytes (one optional field) | 1472 | **1432** |
| 32 bytes (two optional fields) | 1468 | **1428** |
| 36 bytes (three optional fields) | 1464 | **1424** |

These are maximums calculated from the indicated packet sizes, not mandatory configuration values. A smaller underlay MTU or additional encapsulation requires smaller values.

---

## Configure TCP MSS Adjustment

The `ip tcp adjust-mss` interface command limits the MSS value advertised in **TCP SYN packets**, including SYN-ACK packets, that traverse the interface.

For basic GRE over a 1500-byte path, configure a maximum MSS of **1436 bytes**:

### R1

```cisco
interface Tunnel0
 ip mtu 1476
 ip tcp adjust-mss 1436
```

### R2

```cisco
interface Tunnel0
 ip mtu 1476
 ip tcp adjust-mss 1436
```

The MSS adjustment is applied to the **TCP MSS option** during connection setup. It lowers an advertised MSS that exceeds the configured limit, rather than forcing all hosts to advertise exactly that value.

It does **not**:

- Change the IP MTU.
- Change existing TCP sessions that have already completed their handshake.
- Change the size of UDP or ICMP packets.
- Guarantee that an unexpectedly smaller underlay path MTU cannot still cause fragmentation or loss.

For an underlay path MTU of 1400 bytes:

```text
1400 - 24 (GRE) = 1376-byte passenger IPv4 MTU
1376 - 40 (IPv4 + TCP) = 1336-byte TCP MSS
```

Configure:

```cisco
interface Tunnel0
 ip mtu 1376
 ip tcp adjust-mss 1336
```

`ip tcp adjust-mss` applies to **transit TCP connections**. For TCP connections originated or terminated on the router itself, IOS also has separate TCP settings, such as `ip tcp mss`.

### Why Adjust MSS If We Already Set the MTU?

Setting the tunnel IP MTU to 1476 does **not** automatically ensure that hosts behind R1 and R2 will send packets of 1476 bytes or less.

For example, a host on a normal Ethernet LAN may advertise an MSS of 1460, corresponding to a 1500-byte IPv4 packet. Without MSS adjustment or successful PMTUD, those packets may require fragmentation or may be dropped at the GRE ingress.

Lowering the advertised MSS during the TCP handshake encourages both hosts to send smaller segments **before** packet-size problems occur.

---

## Comparing `ip mtu`, `ip tcp adjust-mss`, and Tunnel PMTUD

These commands solve related but different problems.

| Command | Main purpose | Affects |
| --- | --- | --- |
| `ip mtu 1476` | Limits the size of the **inner IPv4 packet** sent through the tunnel | IPv4 passenger traffic |
| `ip tcp adjust-mss 1436` | Limits the MSS advertised by transit TCP connections | TCP traffic only |
| `tunnel path-mtu-discovery` | Discovers a smaller usable IPv4 transport path MTU and enables corresponding DF handling | GRE outer IPv4 forwarding and PMTUD |

A common configuration for a basic GRE-over-IPv4 tunnel is:

```cisco
interface Tunnel0
 ip mtu 1476
 ip tcp adjust-mss 1436
 tunnel path-mtu-discovery
```

The first two values assume a 1500-byte underlay and no additional GRE fields. Tunnel PMTUD can account for lower MTUs further along the underlay when ICMP feedback is available.

A more conservative approach, particularly when there is **additional IPsec overhead**, is to reduce both values:

```cisco
interface Tunnel0
 ip mtu 1400
 ip tcp adjust-mss 1360
```

**1400/1360 are not universal GRE defaults.** They are deliberately smaller values that leave more room for encapsulation overhead. The correct values depend on the actual transport path and all encapsulation layers.

---

## IPv6 over IPv4 GRE

GRE can carry an **inner IPv6 packet** over an IPv4 underlay. The basic GRE-over-IPv4 overhead is still **24 bytes**.

```text
Outer IPv4 header: 20 bytes
GRE header:         4 bytes
Inner IPv6 header: 40 bytes
TCP header:        20 bytes
TCP payload:       Variable
```

With a 1500-byte underlay:

```text
Maximum inner IPv6 packet = 1500 - 24 = 1476 bytes

Maximum IPv6 TCP MSS = 1476 - 40 (IPv6) - 20 (TCP)
                     = 1416 bytes
```

The IPv6 protocol MTU can be configured separately from the IPv4 protocol MTU:

```cisco
interface Tunnel0
 ipv6 mtu 1476
 ipv6 tcp adjust-mss 1416
```

The `ipv6 tcp adjust-mss` command is the IPv6 equivalent of `ip tcp adjust-mss`.

IPv6 handling is different from IPv4 in one important respect: **intermediate IPv6 routers do not fragment transit IPv6 packets**. If an IPv6 packet cannot be forwarded because it is too large, the router sends an ICMPv6 **Packet Too Big** message (Type 2) toward the IPv6 source.

The IPv6 minimum link MTU is **1280 bytes**. With an IPv4 underlay, fragmentation of the **outer IPv4 packet** may still be possible, but this does not mean an IPv6 transit router can fragment the inner IPv6 packet.

For a smaller effective IPv6 MTU, the IPv6 source must adapt its packet size or perform IPv6 source fragmentation when appropriate.

See [GRE Tunnels for IPv6 over IPv4](ipv6-over-ipv4.md) for the full IPv6 tunnel configuration.

---

## GRE with IPsec: Additional Overhead

When GRE is protected by IPsec, the packet requires space for both **GRE encapsulation** and **IPsec headers, trailers, and possible padding**.

Consequently, the maximum usable passenger MTU can be smaller than 1476 bytes even when the physical link MTU is 1500.

For many common GRE-over-IPsec deployments, Cisco's GRE fragmentation guidance uses a **1400-byte tunnel IP MTU** and corresponding **1360-byte IPv4 TCP MSS** as practical values.

```cisco
interface Tunnel0
 ip mtu 1400
 ip tcp adjust-mss 1360
```

These values are an example, not a requirement. The actual overhead depends on IPsec mode, encryption and integrity algorithms, padding, outer IP headers, and other encapsulation.

If GRE is fragmented **before** IPsec protection and the resulting protected packets are also fragmented, the network can experience **double fragmentation**, which adds overhead and complicates reassembly.

---

## Verify MTU and Test Packet Sizes

### Inspect Tunnel Settings

```cisco
R1# show ip interface Tunnel0
R1# show interfaces Tunnel0
R1# show running-config interface Tunnel0
```

Useful information includes the IPv4 MTU, configured MSS adjustment, tunnel mode, source, and destination.

### Test the GRE Tunnel with DF Set

On Cisco IOS, use extended ping to test different **IP datagram sizes** while setting the DF bit.

For example, to test the basic 1476-byte tunnel MTU:

```text
R1# ping
Protocol [ip]:
Target IP address: 10.0.12.2
Repeat count [5]:
Datagram size [100]: 1476
Timeout in seconds [2]:
Extended commands [n]: y
Source address or interface: 10.0.12.1
Type of service [0]:
Set DF bit in IP header? [no]: yes
```

A **1476-byte IPv4 datagram** should fit a basic 1500-byte GRE underlay when there is no additional overhead.

Repeat the test with a **1477-byte datagram**. With a 1476-byte tunnel IP MTU and DF set, the packet is too large and should fail rather than be fragmented before encapsulation.

A small ping succeeding does not prove that larger TCP or UDP packets will work.

### Test the Underlay Separately

To test the path between the **outer tunnel endpoints**, ping the remote tunnel destination rather than the remote tunnel IP address:

```cisco
R1# ping 198.51.100.2 source 192.0.2.1
```

Use an extended ping with DF and increasing packet sizes to investigate the maximum unfragmented **underlay** packet size.

A tunnel can pass small pings while large packets are being dropped because the underlay cannot carry the encapsulated packet.

### Check IPv6 MTU

```cisco
R1# show ipv6 interface Tunnel0
```

If carrying IPv6, test IPv6 end-to-end connectivity and verify ICMPv6 Packet Too Big messages are not being blocked.

### Useful Diagnostic Commands

```cisco
R1# show ip interface Tunnel0
R1# show interfaces Tunnel0
R1# show running-config interface Tunnel0
R1# show ip route 198.51.100.2
R1# show ip cef 198.51.100.2
R1# debug ip icmp
```

`debug ip icmp` can help confirm IPv4 **Fragmentation Needed** messages. Disable debugging afterward:

```cisco
R1# undebug all
```

---

## Common Problems

| Symptom | Possible cause | What to check |
| --- | --- | --- |
| Small pings succeed, large pings fail | GRE overhead exceeds effective MTU | Tunnel MTU, underlay path MTU, DF bit |
| TCP handshake works, data transfer stalls | PMTUD black hole or oversized TCP segments | ICMP filtering, MSS adjustment |
| Tunnel pings work but application traffic fails | Larger packets, different protocols, or TCP MSS | Test with larger DF pings and TCP flows |
| High CPU or poor throughput | Excessive outer fragmentation and reassembly | Packet captures, MTU, IPsec overhead |
| Tunnel MTU looks correct but large packets still fail | Intermediate underlay link has lower MTU | Underlay PMTUD and ICMP Type 3, Code 4 |
| GRE Key or sequencing is enabled | Additional GRE header overhead | Recalculate passenger MTU and MSS |
| IPv6 traffic fails at larger packet sizes | IPv6 PMTUD or inner IPv6 MTU problem | `ipv6 mtu`, ICMPv6 Type 2 |

---

## Key Points

- Basic GRE over IPv4 adds **24 bytes**: 20 bytes for the outer IPv4 header and 4 bytes for GRE.
- With a **1500-byte underlay MTU**, the maximum unfragmented inner IP packet is **1476 bytes**.
- Optional GRE Checksum, Key, and Sequence Number fields each add **4 bytes**.
- A tunnel's locally calculated IP MTU does not necessarily reflect a **smaller MTU elsewhere in the underlay**.
- `ip mtu` sets the tunnel's **inner IPv4 packet MTU**; it does not change the physical underlay MTU.
- With inner IPv4 **DF = 0**, an oversized IPv4 packet can be fragmented **before** GRE encapsulation.
- With inner IPv4 **DF = 1**, an oversized inner packet is dropped and ICMP **Type 3, Code 4** is sent to the original source.
- **Outer IPv4 fragmentation** occurs after GRE encapsulation; the tunnel endpoint must reassemble outer fragments before decapsulation.
- By default, GRE's **outer IPv4 DF bit is cleared**, permitting outer fragmentation.
- `tunnel path-mtu-discovery` enables tunnel PMTUD and copies the **inner IPv4 DF bit** to the **outer IPv4 header**.
- GRE tunnel PMTUD depends on ICMP fragmentation-needed messages reaching the **GRE source**.
- TCP MSS excludes the IP and TCP headers.
- For basic GRE over a 1500-byte IPv4 underlay, the calculated **IPv4 TCP MSS is 1436 bytes**.
- `ip tcp adjust-mss` changes the MSS option in transit TCP SYN packets; it does **not** change the IP MTU or UDP packet sizes.
- For basic IPv6-over-IPv4 GRE, a 1476-byte IPv6 MTU gives a calculated **IPv6 TCP MSS of 1416 bytes**.
- **IPv6 transit routers do not fragment IPv6 packets**; IPv6 PMTUD uses ICMPv6 Packet Too Big.
- GRE over IPsec requires additional MTU allowance beyond basic GRE overhead.

---

## References

- [Cisco: Resolve IPv4 Fragmentation, MTU, MSS, and PMTUD Issues with GRE and IPsec](https://www.cisco.com/c/en/us/support/docs/ip/generic-routing-encapsulation-gre/25885-pmtud-ipfrag.html)
- [Cisco: IOS Interface and Hardware Component Command Reference, tunnel path-mtu-discovery](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/interface/command/ir-cr-book/ir-t2.html)
- [Cisco: IOS IP Application Services Command Reference, ip tcp adjust-mss and ipv6 tcp adjust-mss](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/ipapp/command/iap-cr-book/iap-i2.html)
- [RFC 2784: Generic Routing Encapsulation (GRE)](https://www.rfc-editor.org/rfc/rfc2784)
- [RFC 2890: Key and Sequence Number Extensions to GRE](https://www.rfc-editor.org/rfc/rfc2890)
- [RFC 8200: Internet Protocol, Version 6 (IPv6) Specification](https://www.rfc-editor.org/rfc/rfc8200)
- [RFC 8201: Path MTU Discovery for IPv6](https://www.rfc-editor.org/rfc/rfc8201)
