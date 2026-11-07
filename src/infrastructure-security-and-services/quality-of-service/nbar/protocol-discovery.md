# NBAR Operation and Protocol Discovery

**Network-Based Application Recognition (NBAR)** is a Cisco traffic-classification engine that identifies protocols and applications passing through a network device. It can recognize traffic using more than just IP addresses and TCP/UDP port numbers.

**NBAR Protocol Discovery** uses this classification engine to **identify and measure application traffic on an interface**. It helps an administrator determine which applications are using the network before designing QoS policies.

NBAR has two related uses:
- **Protocol Discovery:** Identifies applications and collects traffic statistics for monitoring.
- **MQC classification:** Identifies applications so a policy map can mark, police, or otherwise treat the matching traffic.

**Enabling Protocol Discovery does not, by itself, mark, prioritize, police, shape, or drop packets.**

---

## How NBAR Recognizes Applications

Ordinary port-based classification examines fields such as the IP protocol number and TCP/UDP source or destination ports. For example, a classifier might identify TCP destination port 80 as HTTP and TCP destination port 443 as HTTPS.

However, these fields do not always reveal the actual application:
- Multiple applications can use the same port, particularly TCP/UDP 443.
- An application can use a nonstandard or dynamically negotiated port.
- An application can tunnel its traffic inside another protocol.
- An application may not be identifiable until several packets have been exchanged.

NBAR can apply several complementary recognition methods:

| Method | How it works | Example |
|---|---|---|
| **IP protocol identification** | Identifies traffic from the IP protocol field | ICMP or GRE |
| **Port-based recognition** | Uses known TCP/UDP ports associated with a protocol | DNS on UDP port 53 |
| **Stateful inspection** | Tracks information across packets belonging to a flow | Recognizing a dynamically negotiated FTP data connection |
| **Deep Packet Inspection (DPI)** | Examines application-layer information and protocol signatures | Distinguishing applications carried over HTTP |
| **Metadata-based recognition** | Uses information such as DNS transactions or other visible connection metadata | Associating a server IP with an application learned from DNS |

Not every application uses every method. The available signatures and recognition methods are determined by the NBAR engine and the active protocol pack.

### Stateful Inspection

Some protocols establish additional connections using dynamically selected ports.

For example, with **FTP in active mode**:
1. A client establishes an FTP control connection to TCP port 21 on the server.
2. The client tells the server which port to use for the data connection.
3. The server establishes a separate data connection to that port.

A simple ACL that matches only TCP port 21 identifies the control connection but cannot reliably identify the dynamically selected data port as FTP.

NBAR can inspect the control exchange and associate the related data connection with the FTP application.

This requires the device to see enough of the relevant conversation to build and use the required flow state.

### Deep Packet Inspection

NBAR can inspect supported fields inside a packet's payload rather than relying only on Layer 3 and Layer 4 headers.

For example, with **unencrypted HTTP**, application-layer information such as the `Host` field, URL, or content type can help identify the application.

```text
                  Received IP Packet
                         |
                         v
               Examine IP protocol
                         |
                         v
                Examine TCP/UDP
                ports and flow state
                         |
                         v
              Inspect available
             application information
                         |
                         v
                Application result
                         |
             +-----------+-----------+
             |                       |
      Protocol Discovery        MQC class map
      traffic statistics        match protocol
```

**NBAR does not automatically decrypt encrypted application payloads.** When traffic is encrypted, recognition depends on the information still visible and the supported classification techniques.

---

## Flow-Based and Multipacket Classification

A **flow** is a set of related packets, typically identified by fields such as source and destination IP addresses, IP protocol, and TCP/UDP ports.

NBAR can maintain flow state to associate successive packets with the same conversation.

Some applications can be identified from the first packet. Others require **multipacket classification**, meaning NBAR examines additional packets before reaching a more specific result.

For example:
1. The first packets in a TCP connection may reveal only that the traffic uses TCP port 443.
2. Subsequent packets may contain recognizable application metadata.
3. NBAR can then classify the flow as a more specific application, if supported.

### Temporary and Final Classification

When NBAR cannot immediately identify an application, the initial result may be **unknown** or a less-specific underlying protocol.

With **multistage classification**, NBAR can report the underlying protocol while waiting for enough information to identify a more specific application.

For example, traffic may initially be recognized as HTTP and later recognized as a specific application carried over HTTP.

This means **classification can change during the lifetime of a flow**. In particular, the first packets of a flow may not receive the same application classification as later packets.

For QoS, that matters if different application classifications would receive different markings or be placed into different output queues.

---

## NBAR Protocol Discovery

**Protocol Discovery** identifies NBAR-supported traffic passing through an interface and maintains statistics for each recognized protocol or application.

It is useful for answering questions such as:
- Which applications are using a WAN interface?
- Which applications are generating the most traffic?
- How much traffic is entering versus leaving the interface?
- Is an application present before configuring a QoS policy?
- Is application traffic being recognized as expected?

Protocol Discovery gathers statistics independently of an MQC service policy. You do **not** need to configure a class map, policy map, or `service-policy` to use it.

### Statistics Collected

For each recognized protocol, Protocol Discovery can report:

| Statistic | Meaning |
|---|---|
| **Input packet count** | Number of matching packets received on the monitored interface |
| **Input byte count** | Number of matching bytes received on the interface |
| **Output packet count** | Number of matching packets switched toward the monitored output interface |
| **Output byte count** | Number of matching bytes switched toward the output interface |
| **Input bit rate** | Measured incoming traffic rate for the protocol, in bps |
| **Output bit rate** | Measured outgoing traffic rate for the protocol, in bps |
| **Maximum bit rate** | Highest measured bit rate over the reporting intervals |

The display commonly includes **30-second bit-rate** and **30-second maximum-bit-rate** fields.

Packet and byte counts are cumulative counters. Bit rates describe traffic activity over a measurement interval, not the total amount of traffic since monitoring began.

### Input vs. Output Statistics

Protocol Discovery can collect statistics in **both directions** on an enabled interface.

```text
                  R1

  LAN -------- Gi1/0 -------- Gi2/0 -------- WAN
                 |              |
              Input to       Output from
                Gi1/0          Gi2/0
```

For traffic forwarded from the LAN to the WAN, the packets can contribute to:
- **Input statistics** on `Gi1/0`.
- **Output statistics** on `Gi2/0`, if Protocol Discovery is enabled there as well.

An important detail: **Output statistics reflect packets switched toward an output interface, not necessarily packets successfully transmitted on the wire.** Packets counted by Protocol Discovery can still be dropped by an egress policer, ACL, or output queue.

Therefore, Protocol Discovery output counters are **not a substitute for interface transmission or QoS drop counters**.

---

## Enabling Protocol Discovery

Protocol Discovery is **disabled by default** and must be enabled on the interface to be monitored.

```text
R1(config)# interface GigabitEthernet1
R1(config-if)# ip nbar protocol-discovery
```

This enables application recognition and Protocol Discovery statistics collection for traffic traversing `GigabitEthernet1`.

By default, the command covers **both IPv4 and IPv6 traffic**.

### IPv4 and IPv6

Older releases also supported the optional keywords `ipv4` and `ipv6` to limit discovery to one IP version:

```text
R1(config-if)# ip nbar protocol-discovery ipv4
R1(config-if)# ip nbar protocol-discovery ipv6
```

**These keywords are deprecated starting with Cisco IOS XE 17.10.1.** Use the command without an IP-version keyword for current IOS XE configurations.

---

## Viewing Protocol Discovery Statistics

The primary verification command is:

```text
R1# show ip nbar protocol-discovery
```

Without additional arguments, this displays Protocol Discovery statistics for **all interfaces on which it is enabled**.

To display statistics for one interface:

```text
R1# show ip nbar protocol-discovery interface GigabitEthernet1
```

### Limiting the Output

The command supports several useful filters:

| Command | Purpose |
|---|---|
| `show ip nbar protocol-discovery` | Show Protocol Discovery statistics on enabled interfaces |
| `show ip nbar protocol-discovery interface GigabitEthernet1` | Show statistics for one interface |
| `show ip nbar protocol-discovery top-n 5` | Show the five most active recognized protocols |
| `show ip nbar protocol-discovery stats packet-count` | Show packet counts |
| `show ip nbar protocol-discovery stats byte-count` | Show byte counts |
| `show ip nbar protocol-discovery stats bit-rate` | Show measured bit rates |
| `show ip nbar protocol-discovery stats max-bit-rate` | Show maximum measured bit rates |
| `show ip nbar protocol-discovery protocol http` | Show statistics for the HTTP protocol |

You can combine an interface selection with supported statistics or protocol filters. For example:

```text
R1# show ip nbar protocol-discovery interface GigabitEthernet1 stats bit-rate
R1# show ip nbar protocol-discovery interface GigabitEthernet1 protocol http
R1# show ip nbar protocol-discovery interface GigabitEthernet1 top-n 5
```

The `top-n` option is particularly useful when many applications are recognized, because it restricts the output to the most active ones.

### Understanding the Output

A Protocol Discovery report organizes the statistics by **interface**, **application/protocol**, and **direction**. A simplified interpretation might be:

| Protocol | Input packets | Input bytes | Output packets | Output bytes |
|---|---:|---:|---:|---:|
| `http` | 12,000 | 15,000,000 | 800 | 100,000 |
| `dns` | 2,000 | 240,000 | 2,100 | 380,000 |
| `rtp` | 35,000 | 7,000,000 | 34,000 | 6,800,000 |

*These numbers are illustrative, not actual device output.*

From this report, you can determine the volume of each recognized protocol and compare the directions of traffic. A separate rate display is more useful for identifying **current bandwidth consumption**.

Do not assume the packet count with the highest value also has the highest byte count or bit rate. Protocols with small packets can produce many packets while consuming relatively little bandwidth.

---

## Clearing Protocol Discovery Counters

To reset the statistics for a particular interface:

```text
R1# clear ip nbar protocol-discovery interface GigabitEthernet1
```

To clear Protocol Discovery statistics for all enabled interfaces:

```text
R1# clear ip nbar protocol-discovery
```

Clearing counters is useful when testing a specific traffic flow:
1. Enable Protocol Discovery on the relevant interface.
2. Clear its statistics.
3. Generate the desired application traffic.
4. Display the counters again.
5. Verify that the expected application's counters increase.

Clearing the counters does **not** disable Protocol Discovery.

---

## Protocol Discovery vs. MQC Classification

Protocol Discovery and MQC can both use NBAR, but they serve different purposes.

| Feature | Protocol Discovery | NBAR with MQC |
|---|---|---|
| **Primary purpose** | Observe and measure application traffic | Apply QoS policy to selected applications |
| **Configuration** | `ip nbar protocol-discovery` | `class-map`, `match protocol`, `policy-map`, `service-policy` |
| **Application recognition** | Yes | Yes |
| **Per-application traffic statistics** | Yes | Policy class counters |
| **Packet marking, policing, or queuing** | No | Yes, through policy-map actions |
| **Requires an active service policy** | No | Yes, to enforce the QoS policy |

For example, a router can use Protocol Discovery to determine that FTP is consuming substantial WAN bandwidth. You can then configure a class map that matches FTP and use that class in a QoS policy.

```text
R1(config)# class-map match-all FTP
R1(config-cmap)# match protocol ftp
```

**This class map alone does not change how FTP traffic is forwarded.** It must be referenced by a policy map that is applied to traffic.

See [NBAR Classification with MQC](mqc-classification.md) for the full configuration process.

---

## Recognized, Unknown, and Misidentified Traffic

Protocol Discovery reports applications according to the classification results supplied by NBAR.

Traffic may appear as **unknown** when NBAR cannot yet identify it or does not have a suitable recognition signature.

Common reasons include:
- The application is not supported by the active protocol pack.
- Required payload information is encrypted or otherwise unavailable.
- The device has not seen enough packets to complete classification.
- The device does not see the necessary packets in both directions of a stateful exchange.
- The application uses an unexpected port or traffic pattern.

**Unknown does not mean that the packet was dropped or that the application is malicious.** It means NBAR did not provide a more specific classification for that traffic.

Likewise, a packet classified as `http`, `ssl`, or another general protocol is not necessarily identified as a specific application running over that protocol.

### Troubleshooting Missing Application Statistics

If an expected application does not appear, check the following:

1. **Interface:** Is Protocol Discovery enabled on the interface where that traffic actually passes?
2. **Traffic:** Is the application actively generating traffic through the device?
3. **Protocol support:** Does the active NBAR protocol pack recognize the application?
4. **Visibility:** Can NBAR see the packets or metadata needed to identify the application?
5. **Routing:** Does asymmetric routing prevent the device from observing the full conversation?
6. **Counters:** Are you looking at the correct interface, direction, and measurement interval?

Useful additional commands include:

```text
R1# show running-config interface GigabitEthernet1
R1# show ip nbar protocol-pack active
R1# show ip nbar protocol-attribute ftp
R1# show ip nbar resources flow
R1# show ip nbar statistics
R1# show interfaces GigabitEthernet1
```

These help verify the interface configuration, active protocol pack, application metadata, available flow resources, NBAR engine statistics, and interface traffic.

### Operational Limits

NBAR classification relies on visibility into the traffic and the capabilities of its recognition engine.

- **Asymmetric flows:** Stateful recognition can fail if the relevant packets traverse different routers.
- **Encryption:** Encrypted payloads cannot be inspected directly without decryption; NBAR may rely on other visible information.
- **Tunnels:** NBAR may identify the tunnel rather than the encapsulated application unless supported tunnel classification is enabled.
- **Protocol support:** An application must be supported by the recognition engine, protocol pack, or an appropriate custom definition to receive a specific classification.
- **Resource usage:** Stateful classification and DPI consume device processing and flow-table resources.

For supported protocols, signatures, protocol packs, and custom application definitions, see [NBAR2 Protocol Packs and Custom Applications](nbar2.md).

---

## Using Protocol Discovery to Design QoS

A practical workflow is:

1. Enable Protocol Discovery on the relevant ingress or WAN interface.
2. Observe application traffic under representative network conditions.
3. Use `top-n`, byte counts, and bit-rate statistics to identify bandwidth-intensive applications.
4. Determine which applications are delay-sensitive, business-critical, or suitable for best-effort treatment.
5. Configure MQC classes and policies based on those requirements.
6. Verify that the configured policies match the expected traffic using `show policy-map interface`.

Protocol Discovery helps determine **what traffic exists and how much is being carried**. It does not determine which applications deserve preferential treatment. That decision belongs to the network's QoS design.

---

## Key Points

- **NBAR** recognizes protocols and applications using port information, flow state, protocol signatures, and other available metadata.
- **Protocol Discovery** identifies and measures NBAR-supported traffic on an interface.
- `ip nbar protocol-discovery` enables monitoring; it does **not** apply QoS actions.
- Protocol Discovery is **disabled by default** and does **not** require an MQC service policy.
- The command enables statistics collection for **both IPv4 and IPv6**; the optional IP-version keywords are deprecated in current IOS XE releases.
- `show ip nbar protocol-discovery` displays application statistics, including packet counts, byte counts, and bit rates.
- Protocol Discovery maintains **input and output statistics**; output counts do not necessarily prove successful transmission.
- `top-n` displays the most active protocols, and `clear ip nbar protocol-discovery` resets statistics.
- **Stateful and multipacket classification** may require more than one packet before an application is fully identified.
- NBAR recognition can be limited by encryption, asymmetric routing, unsupported applications, tunnels, and available device resources.
- Use **MQC `match protocol`** when you want to apply a QoS policy to an NBAR-recognized application.

---

## Further Reading

- [4.4.c Network Based Application Recognition (NBAR)](overview.md)
- [NBAR Classification with MQC](mqc-classification.md)
- [NBAR2 Protocol Packs and Custom Applications](nbar2.md)
- [MQC Class Maps and Match Criteria](../classification/class-maps.md)
- [4.4.d Marking DSCP Values in IPv4 and IPv6 Headers](../marking/overview.md)

### Cisco Documentation

- [Cisco IOS XE 17.x: Enabling Protocol Discovery](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/qos/b-quality-of-service/m_nbar-protocl-discvry-0.html)
- [Cisco IOS XE 17.x: Classifying Network Traffic Using NBAR](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/qos/b-quality-of-service/m_clsfy-traffic-nbar-0.html)
- [Cisco IOS QoS Command Reference: show ip nbar protocol-discovery](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/qos/command/qos-cr-book/qos-s2.html)
- [Cisco IOS QoS Command Reference: clear ip nbar protocol-discovery](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/qos/command/qos-cr-book/qos_a1.html)
