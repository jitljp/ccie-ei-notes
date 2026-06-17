# MPLS Ping and Traceroute

MPLS ping and traceroute are tools used to verify that a Label Switched Path is working.

Normal IP ping and traceroute are still useful, but they do not always prove that MPLS forwarding is working.

```text
Normal ping:
Tests IP reachability.

MPLS LSP ping:
Tests label-switched path reachability.

Normal traceroute:
Shows the IP path.

MPLS LSP traceroute:
Shows the MPLS LSP path for a specific FEC.
```

For CCIE EI, this is useful before moving into L3VPNs because L3VPN traffic depends on a working transport LSP to the remote PE loopback.

---

## Basic Topology

Use the same topology as the previous MPLS pages:

```text
            P3
           /  \
          /    \
 PE1 --- P1 --- P2 --- PE2
```

Loopbacks:

```text
PE1 Loopback0: 1.1.1.1/32
P1  Loopback0: 2.2.2.2/32
P2  Loopback0: 3.3.3.3/32
P3  Loopback0: 4.4.4.4/32
PE2 Loopback0: 5.5.5.5/32
```

LDP IDs:

```text
PE1: 1.1.1.1:0
P1:  2.2.2.2:0
P2:  3.3.3.3:0
P3:  4.4.4.4:0
PE2: 5.5.5.5:0
```

Assume the provider IGP and LDP are already working.

The main transport LSP we will test is:

```text
PE1 → PE2

FEC: 5.5.5.5/32
```

This is important because PE2's loopback will later be used as the BGP next hop for VPNv4 routes.

---

## The Problem

In an MPLS core, a router can have normal IP reachability but still have a broken MPLS data plane.

Example:

```text
PE1 wants to reach PE2's loopback.

Destination: 5.5.5.5/32
```

The control plane may look correct:

```text
IGP route exists to 5.5.5.5/32.
LDP neighbor relationships are up.
LDP bindings exist for 5.5.5.5/32.
```

But MPLS forwarding might still fail, and a normal IP ping may not detect the issues.

Example:

```text
PE1# ping 5.5.5.5 source l0
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 5.5.5.5, timeout is 2 seconds:
Packet sent with a source address of 1.1.1.1 
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/1/2 ms
```

This proves IP reachability to PE2's loopback.

It does not necessarily prove that labeled traffic follows a valid LSP to PE2.

---

## MPLS ping

**MPLS ping** verifies that an LSP works for a specific FEC.

In an LDP-based MPLS core, the FEC is usually an IGP prefix.

Example:

```text
FEC: 5.5.5.5/32
```

This is PE2's loopback.

The command is:

```text
ping mpls ipv4 <prefix>/<mask-length> [source <source-ip>]
```

Example from PE1:

```text
PE1# ping mpls ipv4 5.5.5.5/32 source 1.1.1.1
Sending 5, 72-byte MPLS Echos to 5.5.5.5/32, 
     timeout is 2 seconds, send interval is 0 msec:

Codes: '!' - success, 'Q' - request not sent, '.' - timeout,
  'L' - labeled output interface, 'B' - unlabeled output interface, 
  'D' - DS Map mismatch, 'F' - no FEC mapping, 'f' - FEC mismatch,
  'M' - malformed request, 'm' - unsupported tlvs, 'N' - no label entry, 
  'P' - no rx intf label prot, 'p' - premature termination of LSP, 
  'R' - transit router, 'I' - unknown upstream index,
  'l' - Label switched with FEC change, 'd' - see DDMAP for return code,
  'X' - unknown return code, 'x' - return code 0

Type escape sequence to abort.
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/2/3 ms
 Total Time Elapsed 11 ms
```

This tests whether PE1 can send an MPLS echo request over the LSP for `5.5.5.5/32`.

Notice the various codes that indicate potential issues in the LSP.

---

### MPLS Echo Request and Reply

MPLS LSP ping uses MPLS echo request and echo reply packets.

The echo request is sent through the MPLS LSP.

```text
PE1 creates MPLS echo request
  ↓
PE1 pushes the label for 5.5.5.5/32
  ↓
Packet follows the LSP through the MPLS core
  ↓
PE2 processes the echo request
  ↓
PE2 sends an echo reply back to PE1
```

---

### MPLS Echo Request Destination

The MPLS echo request is forwarded using the label stack associated with the FEC.

The IP destination inside the echo request is not the same as the target FEC.

It uses a special `127.x.x.x` destination.

This prevents the packet from being normally IP forwarded if the LSP is broken.

In simple terms:

```text
The label stack should carry the packet.
The inner IP destination should not route the packet normally.
```

That is why MPLS LSP ping is better than normal ping for testing the MPLS data plane.

---

### Basic MPLS LSP Ping Example

Let's examine a Request ⇄ Reply exchange between PE1 and PE2.

Request as sent from PE1 → P1:

```
Frame 3: 114 bytes on wire (912 bits), 114 bytes captured (912 bits)
Ethernet II, Src: aa:bb:cc:00:13:00 (aa:bb:cc:00:13:00), Dst: aa:bb:cc:00:0f:00 (aa:bb:cc:00:0f:00)
MultiProtocol Label Switching Header, Label: 17, Exp: 0, S: 1, TTL: 255
    0000 0000 0000 0001 0001 .... .... .... = MPLS Label: 17
    .... .... .... .... .... 000. .... .... = MPLS Experimental Bits: 0
    .... .... .... .... .... ...1 .... .... = MPLS Bottom Of Label Stack: 1
    .... .... .... .... .... .... 1111 1111 = MPLS TTL: 255
Internet Protocol Version 4, Src: 1.1.1.1, Dst: 127.0.0.1
    0100 .... = Version: 4
    .... 0110 = Header Length: 24 bytes (6)
    Differentiated Services Field: 0x00 (DSCP: CS0, ECN: Not-ECT)
    Total Length: 96
    Identification: 0x11a4 (4516)
    Flags: 0x4000, Don't fragment
    ...0 0000 0000 0000 = Fragment offset: 0
    Time to live: 1
    Protocol: UDP (17)
    Header checksum: 0x51e2 [validation disabled]
    [Header checksum status: Unverified]
    Source: 1.1.1.1
    Destination: 127.0.0.1
    Options: (4 bytes), Router Alert
User Datagram Protocol, Src Port: 3503, Dst Port: 3503
    Source Port: 3503
    Destination Port: 3503
    Length: 72
    Checksum: 0x390d [unverified]
    [Checksum Status: Unverified]
    [Stream index: 0]
    [Timestamps]
Multiprotocol Label Switching Echo
    Version: 1
    Global Flags: 0x0000
    Message Type: MPLS Echo Request (1)
    Reply Mode: Reply via an IPv4/IPv6 UDP packet (2)
    Return Code: No return code (0)
    Return Subcode: 0
    Sender's Handle: 0xca4edb32
    Sequence Number: 1
    Timestamp Sent: Jun 16, 2026 02:05:49.614999999 UTC
    Timestamp Received: Feb  7, 2036 06:28:16.000000000 UTC
    Vendor Private
    Target FEC Stack
        Type: Target FEC Stack (1)
        Length: 12
        FEC Element 1: Generic IPv4 prefix
```

A few notable things:

* The packet is **MPLS-labeled** when PE1 sends it to P1.
  * The top label is `17`.
  * The `S` bit is `1`, so this is the bottom of the label stack.
  * This means the packet has a **single MPLS label**, which makes sense because this is testing the transport LSP to PE2, not an L3VPN packet with an inner VPN label.
* The MPLS TTL is `255`, but the inner IP TTL is `1`.
  * The MPLS TTL controls how far the labeled packet can travel through the MPLS core.
  * The inner IP TTL is not meant to route this packet normally.
  * This is an MPLS echo packet, not a normal customer/data packet.
* The inner IP destination is `127.0.0.1`.
  * MPLS echo requests use a special loopback destination so the packet is not accidentally forwarded as a normal IP packet if the LSP is broken.
  * The packet should be carried by the MPLS label stack, not by normal IP forwarding to the inner destination.
* The IP header includes the **Router Alert** IP option.
  * This tells routers that the packet may need special processing.
  * MPLS echo packets use this because intermediate/egress routers may need to inspect or process the packet rather than treating it like ordinary transit traffic.
* The UDP source and destination ports are both `3503`.
  * UDP port `3503` is used for MPLS echo packets.
* The MPLS Echo message type is **MPLS Echo Request**.
  * This is the request sent by PE1 to test the LSP toward PE2.
  * The corresponding reply should be an MPLS Echo Reply.
* The reply mode is Reply via an IPv4/IPv6 UDP packet.
  * This means PE1 is asking PE2 to send the reply back using a normal IP/UDP packet.
  * The request tests the MPLS LSP.
  * The reply does not necessarily return over the same MPLS LSP.
* The packet includes a **Target FEC Stack**.
  * This tells the receiver which FEC is being tested.
  * In this case, it identifies PE2's loopback FEC `5.5.5.5/32`.

And here is the Reply as sent from PE2 → P2:

```
Frame 7: 94 bytes on wire (752 bits), 94 bytes captured (752 bits)
Ethernet II, Src: aa:bb:cc:00:15:00 (aa:bb:cc:00:15:00), Dst: aa:bb:cc:00:2a:00 (aa:bb:cc:00:2a:00)
MultiProtocol Label Switching Header, Label: 16, Exp: 6, S: 1, TTL: 255
    0000 0000 0000 0001 0000 .... .... .... = MPLS Label: 16
    .... .... .... .... .... 110. .... .... = MPLS Experimental Bits: 6
    .... .... .... .... .... ...1 .... .... = MPLS Bottom Of Label Stack: 1
    .... .... .... .... .... .... 1111 1111 = MPLS TTL: 255
Internet Protocol Version 4, Src: 10.2.2.2, Dst: 1.1.1.1
    0100 .... = Version: 4
    .... 0101 = Header Length: 20 bytes (5)
    Differentiated Services Field: 0xc0 (DSCP: CS6, ECN: Not-ECT)
    Total Length: 76
    Identification: 0x105f (4191)
    Flags: 0x0000
    ...0 0000 0000 0000 = Fragment offset: 0
    Time to live: 255
    Protocol: UDP (17)
    Header checksum: 0x9c7c [validation disabled]
    [Header checksum status: Unverified]
    Source: 10.2.2.2
    Destination: 1.1.1.1
User Datagram Protocol, Src Port: 3503, Dst Port: 3503
    Source Port: 3503
    Destination Port: 3503
    Length: 56
    Checksum: 0xebb9 [unverified]
    [Checksum Status: Unverified]
    [Stream index: 3]
    [Timestamps]
Multiprotocol Label Switching Echo
    Version: 1
    Global Flags: 0x0000
    Message Type: MPLS Echo Reply (2)
    Reply Mode: Reply via an IPv4/IPv6 UDP packet (2)
    Return Code: Replying router is an egress for the FEC at stack depth RSC (3)
    Return Subcode: 1
    Sender's Handle: 0xca4edb32
    Sequence Number: 1
    Timestamp Sent: Jun 16, 2026 02:05:49.614999999 UTC
    Timestamp Received: Jun 16, 2026 02:05:49.615999999 UTC
    Vendor Private
```

Some notable things:
* The reply is also **MPLS-labeled** when PE2 sends it to P2.
  * The top label is `16`.
  * The `S` bit is `1`, so this is a single-label packet.
  * This does not mean the reply is testing the same LSP in reverse.
  * It means PE2 is using MPLS forwarding to reach the reply destination, `1.1.1.1`.
* The IP destination is PE1's loopback, `1.1.1.1`.
  * PE2 sends the echo reply back toward the sender as an IP/UDP packet.
  * Since the path to `1.1.1.1` goes through the MPLS core, PE2 pushes an MPLS transport label.
* The label `16` is the return-path transport label toward PE1.
  * The original echo request tested the LSP from PE1 to PE2's FEC.
  * This reply is being forwarded toward PE1.
  * Therefore, the label in the reply is for the reverse direction, toward `1.1.1.1`.
* The return code indicates success.
  * `Return Code: Replying router is an egress for the FEC at stack depth RSC (3)`
  * This means the replying router, PE2, believes it is the correct egress for the FEC being tested.
  * `Return Subcode: 1` indicates the relevant label stack depth.
* The reply uses DSCP `CS6`, and the MPLS EXP value is also `6`.
  * IP DSCP: `0xc0`, which is `CS6`.
  * MPLS EXP: `6`.
  * The MPLS EXP value was derived from the IP precedence bits.
  * CS6 is used for control-plane or network-control traffic.
* The source IP is `10.2.2.2`, not PE2's loopback.
  * This is PE2's outgoing interface address toward P2.
  * The destination is PE1's loopback, `1.1.1.1`.
  * So PE2 is sending an IP/UDP MPLS Echo Reply from its outgoing interface toward the original sender.

---

### MTU Testing with MPLS LSP Ping

MPLS adds one or more labels to a packet.

Each MPLS label stack entry is 4 bytes.

If a packet is already near the interface MTU, adding labels can cause MTU problems.

MPLS LSP ping can test different packet sizes with the `sweep` option.

The basic syntax is:

```text
ping mpls ipv4 <fec> sweep <min-size> <max-size> <increment>
```

The `increment` value is the number of bytes the size increases for each subsequent probe.

Example:

```text
PE1# ping mpls ipv4 5.5.5.5/32 source 1.1.1.1 sweep 1400 1500 1 repeat 1
Sending 1, [1400..1500]-byte MPLS Echos to 5.5.5.5/32, 
     timeout is 2 seconds, send interval is 0 msec:

Codes: '!' - success, 'Q' - request not sent, '.' - timeout,
  'L' - labeled output interface, 'B' - unlabeled output interface, 
  'D' - DS Map mismatch, 'F' - no FEC mapping, 'f' - FEC mismatch,
  'M' - malformed request, 'm' - unsupported tlvs, 'N' - no label entry, 
  'P' - no rx intf label prot, 'p' - premature termination of LSP, 
  'R' - transit router, 'I' - unknown upstream index,
  'l' - Label switched with FEC change, 'd' - see DDMAP for return code,
  'X' - unknown return code, 'x' - return code 0

Type escape sequence to abort.
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
!!!!!!!!!!!!!!!!!!!!!!!!!!!QQQQ
Success rate is 96 percent (97/101), round-trip min/avg/max = 1/1/2 ms
 Total Time Elapsed 110 ms
```

In this example, the sweep tests sizes from 1400 to 1500 bytes.

```text
1400, 1401, 1402 ... 1499, 1500
```

With the default MTU of 1500, the last 4 probes are too large after the MPLS label is added.

```text
1497 + 4 = 1501
1498 + 4 = 1502
1499 + 4 = 1503
1500 + 4 = 1504
```

This is important for MPLS L3VPNs, where packets usually carry both a transport label and a VPN label.

---

## MPLS traceroute

**MPLS traceroute** traces the LSP for a specific FEC.

The command is:

```text
traceroute mpls ipv4 <prefix>/<mask-length> [source <source-ip>]
```

Example from PE1:

```text
PE1# traceroute mpls ipv4 5.5.5.5/32 source 1.1.1.1
Tracing MPLS Label Switched Path to 5.5.5.5/32, timeout is 2 seconds

Codes: '!' - success, 'Q' - request not sent, '.' - timeout,
  'L' - labeled output interface, 'B' - unlabeled output interface, 
  'D' - DS Map mismatch, 'F' - no FEC mapping, 'f' - FEC mismatch,
  'M' - malformed request, 'm' - unsupported tlvs, 'N' - no label entry, 
  'P' - no rx intf label prot, 'p' - premature termination of LSP, 
  'R' - transit router, 'I' - unknown upstream index,
  'l' - Label switched with FEC change, 'd' - see DDMAP for return code,
  'X' - unknown return code, 'x' - return code 0

Type escape sequence to abort.
  0 10.1.1.1 MRU 1500 [Labels: 17 Exp: 0]
L 1 10.1.1.2 MRU 1500 [Labels: 20 Exp: 0] 1 ms
L 2 10.1.2.2 MRU 1500 [Labels: implicit-null Exp: 0] 1 ms
! 3 10.2.2.2 2 ms
```

---

### How MPLS LSP Traceroute Works

MPLS LSP traceroute uses MPLS echo requests with increasing TTL values.

```text
First probe:  MPLS TTL = 1
Second probe: MPLS TTL = 2
Third probe:  MPLS TTL = 3
```

Each hop where the TTL expires processes the MPLS echo request and replies with information about the downstream mapping.

In simple form:

```text
TTL 1 expires at first MPLS hop.
TTL 2 expires at second MPLS hop.
TTL 3 expires at third MPLS hop.
```

This allows the router to discover the MPLS LSP hop by hop.

---

### Tracing the Alternate Path Through P3

In this topology, P3 provides an alternate path if the P1-P2 link goes down:

```text
            P3
           /  \
          /    \
 PE1 --- P1 XX P2 --- PE2
```

Then MPLS traceroute should show an extra MPLS hop:

```
PE1# traceroute mpls ipv4 5.5.5.5/32 source 1.1.1.1
Tracing MPLS Label Switched Path to 5.5.5.5/32, timeout is 2 seconds

Codes: '!' - success, 'Q' - request not sent, '.' - timeout,
  'L' - labeled output interface, 'B' - unlabeled output interface, 
  'D' - DS Map mismatch, 'F' - no FEC mapping, 'f' - FEC mismatch,
  'M' - malformed request, 'm' - unsupported tlvs, 'N' - no label entry, 
  'P' - no rx intf label prot, 'p' - premature termination of LSP, 
  'R' - transit router, 'I' - unknown upstream index,
  'l' - Label switched with FEC change, 'd' - see DDMAP for return code,
  'X' - unknown return code, 'x' - return code 0

Type escape sequence to abort.
  0 10.1.1.1 MRU 1500 [Labels: 17 Exp: 0]
L 1 10.1.1.2 MRU 1500 [Labels: 22 Exp: 0] 1 ms
L 2 10.1.3.2 MRU 1500 [Labels: 20 Exp: 0] 5 ms
L 3 10.2.3.1 MRU 1500 [Labels: implicit-null Exp: 0] 3 ms
! 4 10.2.2.2 2 ms
```

---

### Normal Traceroute vs MPLS Traceroute

Normal IP traceroute with MPLS traceroute are similar tools, but different.

Normal traceroute:

```text
PE1# traceroute 5.5.5.5 source 1.1.1.1 numeric 
Type escape sequence to abort.
Tracing the route to 5.5.5.5
VRF info: (vrf in name/id, vrf out name/id)
  1 10.1.1.2 [MPLS: Label 17 Exp 0] 4 msec 3 msec 1 msec
  2 10.1.2.2 [MPLS: Label 20 Exp 0] 0 msec 0 msec 1 msec
  3 10.2.2.2 1 msec *  5 msec
```

MPLS traceroute:

```text
PE1# traceroute mpls ipv4 5.5.5.5/32 source 1.1.1.1
Tracing MPLS Label Switched Path to 5.5.5.5/32, timeout is 2 seconds

Codes: '!' - success, 'Q' - request not sent, '.' - timeout,
  'L' - labeled output interface, 'B' - unlabeled output interface, 
  'D' - DS Map mismatch, 'F' - no FEC mapping, 'f' - FEC mismatch,
  'M' - malformed request, 'm' - unsupported tlvs, 'N' - no label entry, 
  'P' - no rx intf label prot, 'p' - premature termination of LSP, 
  'R' - transit router, 'I' - unknown upstream index,
  'l' - Label switched with FEC change, 'd' - see DDMAP for return code,
  'X' - unknown return code, 'x' - return code 0

Type escape sequence to abort.
  0 10.1.1.1 MRU 1500 [Labels: 17 Exp: 0]
L 1 10.1.1.2 MRU 1500 [Labels: 20 Exp: 0] 2 ms
L 2 10.1.2.2 MRU 1500 [Labels: implicit-null Exp: 0] 2 ms
! 3 10.2.2.2 3 ms
```

Normal traceroute tests the IP path.

MPLS LSP traceroute tests the label-switched path for a FEC.

The normal IP traceroute does give label information, but the MPLS traceroute gives more LSP-specific information,
which is especially useful if there are problems in the LSP.

---

## TTL Propagation and Normal Traceroute

MPLS has its own TTL field.

By default, IOS XE copies the IP TTL into the MPLS TTL when the label is pushed.

```text
IP TTL
  ↓
MPLS TTL
```

This is called **TTL propagation**.

With TTL propagation enabled, a normal IP traceroute can show MPLS core routers.

Example from a CE1 to CE2:

```text
CE1# traceroute 11.11.11.1 source l0 numeric
Type escape sequence to abort.
Tracing the route to 11.11.11.1
VRF info: (vrf in name/id, vrf out name/id)
  1 192.168.1.1 1 msec 1 msec 0 msec
  2 10.1.1.2 [MPLS: Label 17 Exp 0] 2 msec 1 msec 1 msec
  3 10.1.2.2 [MPLS: Label 20 Exp 0] 2 msec 1 msec 2 msec
  4 10.2.2.2 1 msec 2 msec 1 msec
  5 192.168.2.2 2 msec *  6 msec
```

This shows the path from CE1 → PE1 → P1 → P2 → PE2 → CE2.

If TTL propagation is disabled, the MPLS core can be hidden from normal traceroute.

Command:

```text
no mpls ip propagate-ttl
```

This hides the MPLS core from forwarded customer traffic but still allows locally generated provider traffic to reveal the MPLS core.

Here's that same traceroute from CE1 to CE2 with TTL propagation disabled on PE1:

```text
CE1#traceroute 11.11.11.1 source l0 numeric
Type escape sequence to abort.
Tracing the route to 11.11.11.1
VRF info: (vrf in name/id, vrf out name/id)
  1 192.168.1.1 1 msec 1 msec 0 msec
  2 10.2.2.2 3 msec 1 msec 2 msec
  3 192.168.2.2 2 msec *  9 msec
```

This shows the path from CE1 → PE1 → PE2 → CE2.

With TTL propagation disabled the MPLS header uses TTL 255 by default, so the TTL doesn't expire as the messages
traverse the MPLS core.

The P routers are hidden from the customer traceroute.

---

### TTL propagation configuration options

The command shown previously disables TTL propagation entirely:

```text
PE1(config)# no mpls ip propagate-ttl
```

With this configured, the IP TTL is not copied into the MPLS TTL when the label is pushed.

Instead, the MPLS TTL starts at 255.

```text
IP packet enters MPLS core
  ↓
IP TTL is not copied
  ↓
MPLS TTL starts at 255
```

This hides the MPLS core from normal traceroute.

There are also two optional keywords:

```text
PE1(config)# no mpls ip propagate-ttl ?
  forwarded  Propagate IP TTL for forwarded traffic
  local      Propagate IP TTL for locally originated traffic
  <cr>       <cr>
```

The `forwarded` keyword affects traffic forwarded through the router.

```text
PE1(config)# no mpls ip propagate-ttl forwarded
```

This is commonly used by service providers.

It hides the MPLS core from customer traceroutes, but still allows traceroutes sourced from the PE router itself to show the MPLS core.

```text
Customer traceroute:
CE1 → PE1 → PE2 → CE2

Provider traceroute from PE1:
PE1 → P1 → P2 → PE2
```

The `local` keyword affects traffic originated by the router itself.

```text
PE1(config)# no mpls ip propagate-ttl local
```

This disables TTL propagation for locally generated traffic, such as traceroutes sourced from the PE router.

In summary:

| Command                              | Effect                                                                    |
| :----------------------------------- | :------------------------------------------------------------------------ |
| `no mpls ip propagate-ttl`           | Disable TTL propagation for both forwarded and locally originated traffic |
| `no mpls ip propagate-ttl forwarded` | Disable TTL propagation only for forwarded traffic                        |
| `no mpls ip propagate-ttl local`     | Disable TTL propagation only for locally originated traffic               |

For most MPLS service provider examples, the most common option is:

```text
no mpls ip propagate-ttl forwarded
```

This hides the provider core from customer traceroutes while still allowing the provider to troubleshoot the MPLS core from PE routers.

---

## Load Balancing and Destination Addresses

MPLS LSP ping and traceroute can use different destination addresses in the MPLS echo request.

This can be useful when ECMP exists.

Different destination values may exercise different load-sharing paths.

Here's the basic syntax:

```
traceroute mpls ipv4 <fec> destination <start-address> <end-address> <increment>
```

This can help reveal multiple possible LSP paths.

I modified the OSPF cost so P1 performs load balancing over two paths:

```
P1 → P2 → PE2
P1 → P3 → P2 → PE2
```

Here is the output:

```
PE1# mpls ipv4 5.5.5.5/32 source 1.1.1.1 destination 127.0.0.1 127.0.0.6 5
Tracing MPLS Label Switched Path to 5.5.5.5/32, timeout is 2 seconds

Codes: '!' - success, 'Q' - request not sent, '.' - timeout,
  'L' - labeled output interface, 'B' - unlabeled output interface, 
  'D' - DS Map mismatch, 'F' - no FEC mapping, 'f' - FEC mismatch,
  'M' - malformed request, 'm' - unsupported tlvs, 'N' - no label entry, 
  'P' - no rx intf label prot, 'p' - premature termination of LSP, 
  'R' - transit router, 'I' - unknown upstream index,
  'l' - Label switched with FEC change, 'd' - see DDMAP for return code,
  'X' - unknown return code, 'x' - return code 0

Type escape sequence to abort.
Destination address 127.0.0.1
  0 10.1.1.1 MRU 1500 [Labels: 17 Exp: 0]
L 1 10.1.1.2 MRU 1500 [Labels: 22 Exp: 0] 1 ms
L 2 10.1.3.2 MRU 1500 [Labels: 20 Exp: 0] 1 ms
L 3 10.2.3.1 MRU 1500 [Labels: implicit-null Exp: 0] 3 ms
! 4 10.2.2.2 2 ms
Destination address 127.0.0.6
  0 10.1.1.1 MRU 1500 [Labels: 17 Exp: 0]
L 1 10.1.1.2 MRU 1500 [Labels: 20 Exp: 0] 1 ms
L 2 10.1.2.2 MRU 1500 [Labels: implicit-null Exp: 0] 1 ms
! 3 10.2.2.2 1 ms
```

As you can see, the two traceroutes used different LSPs to reach the FEC.

---

## Key Points

* MPLS ping verifies LSP connectivity for a specific FEC.
  * Use `ping mpls ipv4 <prefix>/<mask-length>`.
* MPLS echo requests are sent through the LSP being tested.
  * They use UDP port `3503`.
  * The inner IP destination is a special `127.x.x.x` address.
  * The Target FEC Stack identifies the FEC being tested.
* MPLS echo replies are MPLS Echo Reply messages carried in IP/UDP.
  * The request tests the LSP to the target FEC.
  * The reply does not necessarily return over the same LSP.
  * The reply may still be MPLS-labeled if the return path uses MPLS.
* MPLS ping can test MTU with the `sweep` option.
  * Each MPLS label adds 4 bytes.
* MPLS traceroute traces the LSP for a specific FEC.
  * Use `traceroute mpls ipv4 <prefix>/<mask-length>`.
  * It uses MPLS echo requests with increasing MPLS TTL values.
  * It gives more LSP-specific information than normal IP traceroute.
* Normal traceroute can show MPLS core hops if TTL propagation is enabled.
  * By default, the IP TTL is copied into the MPLS TTL.
  * `no mpls ip propagate-ttl` hides the MPLS core from normal traceroute.
  * `no mpls ip propagate-ttl forwarded` hides the core from customer traceroutes only.
  * `no mpls ip propagate-ttl local` affects locally generated traffic.
* MPLS traceroute can test ECMP paths by varying the destination address.
  * Use `destination <start-address> <end-address> <increment>`.
  * Different destination values may hash to different equal-cost LSPs.