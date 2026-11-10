
# Explicit Congestion Notification (ECN)

**Explicit Congestion Notification (ECN)** is a congestion avoidance mechanism that allows network devices to signal congestion by **marking packets instead of dropping them**.

ECN is defined in RFC 3168 and is commonly implemented alongside **Weighted Random Early Detection (WRED)**.

Without ECN, WRED signals congestion by randomly dropping packets before a queue becomes full.

With ECN, a router can mark eligible packets with the **Congestion Experienced (CE)** codepoint and forward them instead.

The receiving endpoint informs the sender of the congestion, allowing the sender to reduce its transmission rate without requiring packet loss as the initial congestion signal.

ECN therefore requires cooperation between:
- The sender, which indicates that packets are ECN-capable.
- The network device, which marks packets when congestion occurs.
- The receiver, which reports congestion to the sender.
- The sender's congestion-control algorithm, which reacts to the feedback.

---

## ECN Fields in IPv4 and IPv6

ECN uses two bits in the IP header.

For IPv4, these bits are the two least significant bits of the **Differentiated Services (DS) field**.

For IPv6, they are the two least significant bits of the **Traffic Class field**.

```text
       IPv4 DS Field / IPv6 Traffic Class

       +-----------------------+--------+
       |         DSCP          |  ECN   |
       +-----------------------+--------+
                6 bits           2 bits
```

The six DSCP bits determine the packet's differentiated services marking.

The two ECN bits indicate whether the packet supports ECN and whether congestion has been experienced.

### ECN Codepoints

The two ECN bits provide four possible values.

| Binary | Decimal | Name | Meaning |
|---|---:|---|---|
| `00` | 0 | Not-ECT | Not ECN-Capable Transport |
| `01` | 1 | ECT(1) | ECN-Capable Transport |
| `10` | 2 | ECT(0) | ECN-Capable Transport |
| `11` | 3 | CE | Congestion Experienced |

**Not-ECT (`00`)**
- Indicates that the packet is not using ECN.
- A router cannot use ECN marking to signal congestion for this packet.
- If WRED selects the packet for congestion signaling, it is dropped.

**ECT(0) (`10`) and ECT(1) (`01`)**
- Indicate that the packet belongs to an ECN-capable transport.
- Allow a network device to signal congestion by marking the packet CE.
- Traditional ECN treats both codepoints equivalently.

**CE (`11`)**
- Indicates that a network device has detected congestion.
- Is produced by changing an ECN-capable packet's ECN field to `11`.
- Must not be cleared by intermediate routers.

### ECN Is Separate from DSCP

ECN and DSCP use different bits within the same header field.

For example, consider a packet with:

```text
DSCP = AF31 (26)
ECN  = ECT(0) (2)
```

The combined eight-bit field is:

```text
DSCP       ECN
011010     10

Full field = 01101010
           = 0x6A
```

If a router marks the packet CE:

```text
DSCP       ECN
011010     11

Full field = 01101011
           = 0x6B
```

The packet retains DSCP AF31.

Only the ECN bits change from `10` to `11`.

**ECN marking does not change the packet's DSCP or its QoS scheduling priority.**

---

## ECN Operation

ECN allows the network to notify endpoints about congestion without necessarily discarding packets.

Consider a TCP connection between PC1 and Server1.

```text
      PC1                    R1                 Server1
   TCP Sender          Congested Router       TCP Receiver
       |                     |                     |
       |---- ECT(0) -------->|                     |
       |                     |                     |
       |                 Congestion                |
       |                  Detected                 |
       |                     |                     |
       |                     |------ CE ---------->|
       |                     |                     |
       |<--------------------|------ TCP ECE ------|
       |                     |                     |
   Reduce TCP                |                     |
   Sending Rate              |                     |
       |                     |                     |
       |---- TCP CWR ------->|-------------------->|
```

The process is:

1. PC1 sends a TCP data packet with the ECN field set to ECT(0) or ECT(1).
2. R1 experiences congestion and selects the packet for early congestion signaling.
3. Instead of dropping the packet, R1 changes its ECN field to CE (`11`).
4. R1 forwards the marked packet to Server1.
5. Server1 receives the packet and reports the congestion to PC1 using TCP's ECE flag.
6. PC1 reduces its congestion window according to its congestion-control algorithm.
7. PC1 acknowledges its congestion response using TCP's CWR flag.

The original data packet is successfully delivered.

ECN therefore avoids the retransmission that would have been necessary if the packet had been dropped.

However, the sender must still reduce its sending rate.

**ECN changes how congestion is signaled, not the requirement for senders to respond to congestion.**

---

## ECN and TCP

TCP supports ECN through two flags in the TCP header:

| TCP flag | Full name | Purpose |
|---|---|---|
| **ECE** | ECN-Echo | Reports congestion to the sender |
| **CWR** | Congestion Window Reduced | Indicates that the sender has responded to congestion |

These are separate from the ECN bits in the IP header.

The distinction is important:

- **IP ECN bits:** Communicate congestion capability and congestion experienced within the network.
- **TCP ECE/CWR flags:** Communicate ECN capability during connection establishment and congestion feedback between TCP endpoints.

### TCP ECN Negotiation

In classic TCP ECN, support is negotiated during the TCP three-way handshake.

The initiating host indicates ECN support using both ECE and CWR in the initial SYN.

An ECN-capable responding host acknowledges support using ECE in the SYN-ACK.

```text
          PC1                           Server1
           |                               |
           |------ SYN ------------------->|
           |       ECE = 1                 |
           |       CWR = 1                 |
           |                               |
           |<----- SYN-ACK ----------------|
           |       ECE = 1                 |
           |       CWR = 0                 |
           |                               |
           |------ ACK ------------------->|
           |                               |
           |     ECN-capable connection    |
```

The handshake establishes whether classic TCP ECN can be used.

The meanings of ECE and CWR during connection setup differ from their meanings during normal data transfer.

During the initial SYN, setting both flags advertises ECN capability. It does not indicate that congestion has already occurred.

If the endpoints do not successfully negotiate ECN, the connection normally uses traditional loss-based congestion signaling.

### TCP Congestion Feedback

After ECN negotiation, the sender can mark its data packets ECT(0) or ECT(1).

When a receiver obtains a packet marked CE, it sets the TCP **ECE flag** in its acknowledgment.

In classic TCP ECN, the receiver continues setting ECE on subsequent ACKs until it receives a CWR indication from the sender.

When the sender receives the ECE indication, it reduces its congestion window.

For classic ECN congestion control, this response is comparable to the reduction performed after congestion-related packet loss.

The sender then sets the **CWR flag** to acknowledge the congestion response.

The receiver can stop echoing that congestion episode after receiving CWR.

**CE is an IP-header codepoint. ECE and CWR are TCP-header flags.**

---

## ECN with WRED

Cisco IOS XE implements ECN as an extension of WRED on supported platforms.

Ordinary WRED uses:
- Average queue depth.
- Minimum threshold.
- Maximum threshold.
- Mark probability denominator.

ECN uses the same WRED congestion detection mechanism.

The difference is the action taken when WRED selects an eligible packet for early dropping.

| Mechanism | Congestion signal |
|---|---|
| WRED without ECN | Drop the selected packet |
| WRED with ECN, Not-ECT packet | Drop the selected packet |
| WRED with ECN, ECT(0)/ECT(1) packet | Mark CE and forward the packet |

ECN does not independently calculate congestion thresholds.

It relies on the configured WRED drop profiles.

See [Weighted Random Early Detection (WRED)](wred.md).

### ECN Packet Handling by WRED Threshold

On Cisco's documented classic IOS XE WRED/ECN implementation, packet handling depends on the average queue depth.

| Average queue depth | Behavior |
|---|---|
| Below minimum threshold | Forward without new ECN marking |
| Between minimum and maximum thresholds | Probabilistically drop Not-ECT packets or mark selected ECN-capable packets CE |
| At or above maximum threshold | WRED drops packets rather than relying on early ECN marking |

The exact boundary behavior and hardware implementation can vary by platform.

When the queue reaches its instantaneous queue limit, packets can also be dropped regardless of their ECN markings.

### Example: Partial Drop Region

Suppose WRED is configured with:

```text
Minimum threshold = 50 packets
Maximum threshold = 100 packets
MPD               = 10
```

The average queue depth is currently 75 packets.

WRED is in its probabilistic early-drop region.

The nominal random-drop probability is:

```text
P = (75 - 50) / (100 - 50) × (1 / 10)
  = 5%
```

When the algorithm selects a packet for congestion signaling:

| Incoming ECN field | Action |
|---|---|
| Not-ECT (`00`) | Drop |
| ECT(0) (`10`) | Mark CE and forward |
| ECT(1) (`01`) | Mark CE and forward |
| Already CE (`11`) | Already indicates congestion; no additional marking is needed |

On the documented IOS XE implementation, packets already marked CE are forwarded while in this early congestion region, subject to available queue capacity.

The router does not mark every ECN-capable packet CE merely because the average depth is between the thresholds.

**ECN marking occurs when WRED selects the packet for congestion signaling.**

### Queue Limit vs. WRED Maximum Threshold

The WRED maximum threshold and the queue limit are separate concepts.

Consider:

```text
Queue limit            = 200 packets
WRED minimum threshold = 50 packets
WRED maximum threshold = 100 packets
```

The WRED thresholds are evaluated against the **average queue depth**.

The queue limit is evaluated against the **instantaneous queue depth**.

Therefore, the following conditions are possible:

- The average depth is 75 packets, so WRED may randomly drop or mark packets.
- The instantaneous queue depth reaches 200 packets, causing queue-limit drops even if the average depth remains below 100.
- Sustained congestion raises the average depth beyond 100 packets, causing the documented WRED maximum-threshold dropping behavior.

**ECN does not guarantee that ECN-capable packets will never be dropped.**

---

## ECN Configuration with MQC

On supported Cisco IOS XE routers, ECN is configured within an MQC policy-map class.

WRED must first be enabled.

The relevant command is:

```text
R1(config-pmap-c)# random-detect ecn
```

This enables ECN support for the class's WRED congestion avoidance mechanism.

### Basic ECN Configuration

Suppose R1 has a WAN interface carrying best-effort TCP traffic.

Configure:
- Minimum bandwidth allocation of 30%.
- DSCP-based WRED.
- ECN marking for ECN-capable packets.

```text
R1(config)# policy-map WAN-QOS
R1(config-pmap)# class class-default
R1(config-pmap-c)# bandwidth percent 30
R1(config-pmap-c)# random-detect dscp-based
R1(config-pmap-c)# random-detect ecn
```

Apply the policy outbound:

```text
R1(config)# interface GigabitEthernet2
R1(config-if)# service-policy output WAN-QOS
```

The policy enables WRED and ECN for the `class-default` queue.

During WRED's early congestion region:
- Not-ECT packets can be dropped.
- ECT(0) and ECT(1) packets can be marked CE instead of being dropped.

The default WRED thresholds depend on the platform and queue configuration.

### ECN with Explicit WRED Thresholds

Suppose R1 should apply ECN to AF3 traffic.

Requirements:
- Match AF31, AF32, and AF33.
- Allocate a minimum of 30% bandwidth.
- Configure a queue limit of 200 packets.
- Use different WRED drop profiles for the three AF drop precedences.
- Enable ECN.

First, configure the class map:

```text
R1(config)# class-map match-any AF3-DATA
R1(config-cmap)# match dscp af31 af32 af33
```

Configure the policy:

```text
R1(config)# policy-map WAN-QOS
R1(config-pmap)# class AF3-DATA
R1(config-pmap-c)# bandwidth percent 30
R1(config-pmap-c)# queue-limit 200 packets
R1(config-pmap-c)# random-detect dscp-based
R1(config-pmap-c)# random-detect dscp af31 100 160 20
R1(config-pmap-c)# random-detect dscp af32 80 160 10
R1(config-pmap-c)# random-detect dscp af33 60 160 5
R1(config-pmap-c)# random-detect ecn
```

Apply the policy:

```text
R1(config)# interface GigabitEthernet2
R1(config-if)# service-policy output WAN-QOS
```

The configured WRED thresholds are:

| DSCP | Minimum | Maximum | MPD |
|---|---:|---:|---:|
| AF31 | 100 | 160 | 20 |
| AF32 | 80 | 160 | 10 |
| AF33 | 60 | 160 | 5 |

All three markings share one queue.

AF33 is eligible for congestion signaling earlier than AF32 or AF31.

For an ECN-capable packet, that signaling can be performed through CE marking rather than packet dropping.

This allows **differentiated congestion avoidance without necessarily introducing packet loss**.

### Disabling ECN

To disable ECN while retaining WRED:

```text
R1(config-pmap-c)# no random-detect ecn
```

WRED continues operating, but selected packets are dropped rather than ECN-marked.

To disable WRED entirely, remove the applicable `random-detect` configuration.

---

## ECN Verification

The primary commands for verifying ECN are:

```text
R1# show policy-map
R1# show policy-map interface GigabitEthernet2
```

### Verifying ECN Configuration

The `show policy-map` output should indicate that ECN is enabled.

For example, Cisco IOS XE can display:

```text
Policy Map WAN-QOS
  Class class-default
    Weighted Fair Queueing
      explicit congestion notification
```

Look for the indication:

```text
explicit congestion notification
```

This confirms that ECN has been enabled for the policy class.

### Verifying ECN Marking

The `show policy-map interface` command can display ECN marking statistics.

Depending on the IOS XE platform and software version, the output may include:

```text
ECN Mark
pkts/bytes
```

These counters report packets marked by the ECN mechanism.

Important statistics include:

| Statistic | Meaning |
|---|---|
| `Mean queue depth` | Average queue depth used by WRED |
| `Random drop` | Packets randomly discarded by WRED |
| `Tail drop` | Packets dropped because of WRED threshold or queue-limit behavior, depending on the reported counter |
| `ECN Mark` | Packets marked for congestion notification |
| `Minimum threshold` | Average depth at which early congestion signaling begins |
| `Maximum threshold` | Upper WRED threshold |
| `Mark probability` | Configured WRED maximum random-drop probability |

ECN marking counters and drop counters represent different outcomes.

For example, a class may report ECN marks without corresponding random drops if the selected packets were ECN-capable and could be marked instead.

However, this does not prove that all traffic is ECN-capable or that packet loss cannot occur.

### Packet Capture Verification

ECN can also be verified using a packet capture.

For example, Wireshark can display the ECN field within the IPv4 or IPv6 header.

For an ECN-capable packet before congestion marking:

```text
Differentiated Services Field: 0x02
    DSCP: CS0
    ECN: ECT(0)
```

After congestion marking:

```text
Differentiated Services Field: 0x03
    DSCP: CS0
    ECN: CE
```

For classic TCP ECN, subsequent packets can also be inspected for:
- ECE in the receiver's acknowledgments.
- CWR in packets from the sender.

This provides evidence of both network congestion marking and endpoint feedback.

---

## ECN Limitations and Design Considerations

### ECN Requires Endpoint Support

The router can mark packets CE only when they indicate ECN capability.

Enabling ECN on a router does not automatically enable ECN on end hosts.

For classic TCP ECN, both endpoints must successfully negotiate ECN support.

Packets marked Not-ECT are still subject to ordinary WRED dropping.

### ECN Is Not Limited to TCP

ECN is defined at the IP layer.

It can therefore be used by transport protocols other than TCP, provided they implement appropriate congestion feedback and response mechanisms.

For example, ECN can also be supported by UDP-based transport protocols with suitable congestion control, such as QUIC.

However, ordinary UDP traffic does not automatically support ECN.

### ECN Does Not Eliminate Congestion

ECN allows early congestion signaling without necessarily losing packets.

It does not increase link capacity.

If traffic sources ignore congestion signals or continue transmitting above the available service rate, queues can still overflow and packets can still be dropped.

### ECN Does Not Guarantee Zero Packet Loss

Even an ECN-capable packet can be dropped due to:
- Queue-limit exhaustion.
- Severe congestion beyond the applicable WRED marking region.
- Insufficient packet-buffer resources.
- Policing or other packet-dropping mechanisms.
- Network devices that do not support ECN marking.

ECN is an additional congestion signaling mechanism, not a guarantee of reliable delivery.

### ECN Should Be Applied to Appropriate Traffic

ECN is most useful for traffic with congestion-control mechanisms that respond to congestion feedback.

It is commonly appropriate for elastic data traffic.

Applying ECN to traffic that does not respond to congestion signals provides little congestion avoidance benefit.

ECN also does not replace scheduling mechanisms such as CBWFQ or LLQ.

### Platform Support

The ECN configuration described here applies to supported IOS XE MQC implementations.

On QFP-based routers, WRED and ECN are associated with actual output queues.

A parent policy class containing a child queuing policy does not itself provide a packet queue on the documented ASR 1000 scheduler.

Therefore, ECN should be configured within an appropriate queueing class, typically in the child policy when hierarchical queuing is used.

The precise command support, ECN marking behavior, and available counters can vary across platforms.

---

## Key Points

- **ECN** allows routers to signal congestion through packet marking rather than packet dropping.
- ECN uses the two least significant bits of the IPv4 DS field or IPv6 Traffic Class field.
- `00` = Not-ECT.
- `01` = ECT(1).
- `10` = ECT(0).
- `11` = CE.
- ECT(0) and ECT(1) indicate that a packet is ECN-capable.
- CE indicates that congestion has been experienced.
- ECN marking changes the ECN bits without changing the DSCP.
- Cisco IOS XE supports ECN as an extension of WRED on applicable platforms.
- WRED determines when congestion signaling is necessary.
- ECN determines whether a selected eligible packet can be marked instead of dropped.
- Not-ECT packets remain subject to ordinary WRED dropping.
- ECN-capable packets can still be dropped when queues become full or congestion is severe.
- In classic TCP ECN, ECE reports congestion and CWR acknowledges the sender's congestion response.
- `random-detect ecn` enables ECN for an MQC WRED class.
- `show policy-map interface` is the primary command for verifying ECN marking statistics.
- ECN does not replace congestion management, traffic shaping, or endpoint congestion control.

---

## References

- [Cisco IOS XE: WRED Explicit Congestion Notification](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/quality-of-service/quality-of-service/m_qos-conavd-wred-ecn-0.html)
- [Cisco IOS XE: Queue Limits and WRED](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/quality-of-service/quality-of-service/m_qos-queue_management_and_congestion_avoidance.html)
- [RFC 3168: The Addition of Explicit Congestion Notification (ECN) to IP](https://www.rfc-editor.org/rfc/rfc3168)
- [RFC 9768: More Accurate Explicit Congestion Notification (AccECN) Feedback in TCP](https://www.rfc-editor.org/rfc/rfc9768)
