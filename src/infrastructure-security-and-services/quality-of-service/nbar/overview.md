# 4.4.c Network Based Application Recognition (NBAR)

**Network Based Application Recognition (NBAR)** is a Cisco traffic classification engine that identifies network applications and protocols so that they can be monitored or assigned appropriate Quality of Service (QoS) treatment.

Ordinary traffic classification often relies on IP addresses, TCP/UDP port numbers, or existing DSCP markings. NBAR can examine additional information to recognize applications that cannot be reliably identified using those fields alone.

For example, a traditional ACL can match TCP port 443, but that only identifies traffic using the port. It does not necessarily identify which application is generating the traffic. NBAR can use application signatures and other available information to make a more specific classification.

**NBAR performs classification.** It does not inherently prioritize, mark, police, or shape traffic. Those actions can be applied to the recognized traffic using the **Modular QoS CLI (MQC)**.

---

## Why NBAR Is Needed

Applications do not always use a single, predictable TCP or UDP port.

Common classification challenges include:
- **Dynamic ports:** Some applications negotiate additional port numbers during a session.
- **Shared ports:** Many unrelated applications use the same port, particularly TCP 80 and TCP 443.
- **Application tunneling:** An application may operate inside another protocol.
- **Changing application behavior:** Applications may change their communication methods or signatures over time.

Consider two connections using TCP port 443:

| Connection | Port-based classification | Application-based classification |
|---|---|---|
| Web browsing | TCP 443 | Web browsing application, if identifiable |
| Video conferencing | TCP 443 | Video conferencing application, if identifiable |

An ACL that matches TCP port 443 cannot distinguish the two connections based on that port alone. NBAR may be able to distinguish them using signatures or other available application information.

Application recognition is especially useful at the **network edge**, where traffic can be classified and marked before entering the rest of a DiffServ domain.

---

## How NBAR Identifies Applications

Depending on the protocol, NBAR can use several classification methods:

| Method | Description |
|---|---|
| **Static port recognition** | Identifies applications using known TCP or UDP port numbers. |
| **Stateful inspection** | Tracks information from multiple packets in a flow, including protocols that negotiate dynamic ports. |
| **Deep Packet Inspection (DPI)** | Examines recognizable protocol fields or application signatures beyond basic Layer 3 and Layer 4 headers. |
| **Application metadata** | Uses information such as visible server names and DNS activity to help identify applications. |

NBAR supports recognition of a variety of IP protocols, including some protocols that do not use TCP or UDP.

### Stateful Application Recognition

Some applications cannot be identified accurately from a single packet.

For example, an application might negotiate a data connection using a control connection. NBAR can inspect the control exchange and associate subsequent traffic with the appropriate protocol.

NBAR may therefore initially report a flow using a general protocol classification and refine that classification after examining additional packets. Not every application can be recognized from its first packet.

### Encrypted Applications

NBAR does not need to decrypt all application traffic to perform useful classification. It can sometimes identify encrypted applications using information that remains visible, such as selected handshake fields, DNS information, or other recognizable characteristics.

However, **NBAR cannot inspect application content hidden by encryption**. Classification accuracy depends on which information is available and which signatures the platform supports.

For example, recognizing traffic as HTTPS is not the same as identifying the specific application carried over HTTPS.

---

## Two Main Uses of NBAR

NBAR is commonly used for **application visibility** and **application-based QoS classification**.

| Feature | Purpose | Primary command |
|---|---|---|
| **Protocol Discovery** | Identifies protocols on an interface and collects traffic statistics. | `ip nbar protocol-discovery` |
| **MQC classification** | Matches NBAR-recognized applications in a class map so that a policy can take action. | `match protocol` |

These functions can be used independently. **Enabling Protocol Discovery is not required for an MQC class map to match NBAR-recognized traffic.**

### Protocol Discovery

**NBAR Protocol Discovery** provides visibility into the applications traversing an interface without requiring a QoS policy.

For example:

```text
R1(config)# interface GigabitEthernet0/0/0
R1(config-if)# ip nbar protocol-discovery
```

Display the collected statistics:

```text
R1# show ip nbar protocol-discovery interface GigabitEthernet0/0/0
```

Protocol Discovery can report:
- Recognized protocols and applications.
- Input and output packet counts.
- Input and output byte counts.
- Traffic rates for each protocol.

It can help determine which applications consume bandwidth before designing a QoS policy.

**Protocol Discovery does not, by itself, change the forwarding treatment of packets.**

See [NBAR Operation and Protocol Discovery](protocol-discovery.md).

### NBAR Classification with MQC

NBAR can also be used inside an MQC **class map** with the `match protocol` command.

For example, the following policy identifies FTP traffic and marks it with DSCP CS1:

```text
R1(config)# class-map match-any BULK-FTP
R1(config-cmap)# match protocol ftp
R1(config-cmap)# exit
R1(config)# policy-map MARK-APPLICATIONS
R1(config-pmap)# class BULK-FTP
R1(config-pmap-c)# set dscp cs1
R1(config-pmap-c)# exit
R1(config-pmap)# exit
R1(config)# interface GigabitEthernet0/0/0
R1(config-if)# service-policy input MARK-APPLICATIONS
```

The process is:

1. NBAR identifies packets belonging to FTP traffic.
2. The MQC class map assigns matching packets to the `BULK-FTP` class.
3. The policy map marks matching packets with DSCP CS1.
4. Downstream devices can use the DSCP marking to apply the intended forwarding treatment.

CS1 is used here as an example marking for lower-priority bulk traffic. The appropriate marking for a real application depends on the network's QoS design.

The **class map identifies traffic**, while the **policy map defines the action**. NBAR itself does not set the DSCP value.

See [NBAR Classification with MQC](mqc-classification.md).

---

## NBAR and NBAR2

**NBAR2** is the enhanced application-recognition engine used by modern Cisco IOS XE platforms. It extends NBAR's classification capabilities through a larger application library, additional signatures, and application attributes.

A key NBAR2 feature is the **protocol pack**.

### Protocol Packs

A **protocol pack** contains the protocol definitions and signatures used by NBAR2 to recognize applications.

Cisco can provide updated protocol packs so that supported devices can recognize additional or changed applications **without upgrading the entire IOS XE image**.

A device includes a built-in protocol pack. On supported platforms, a compatible updated pack can be installed.

For example, display the active protocol pack:

```text
R1# show ip nbar protocol-pack active
```

The available applications and signatures depend on the active protocol pack, software release, platform, and licensing.

### Application Attributes

NBAR2 can also associate **attributes** with recognized applications, including categories, subcategories, and application groups.

These attributes allow an MQC class map to match groups of applications rather than listing each application individually.

For example, an enterprise might define a QoS class around a particular application category or traffic type. The classification can then follow the application attributes supplied by the active protocol pack.

### Custom Applications

When a required application is not recognized by the built-in signatures, **custom NBAR protocols** can be defined using criteria such as ports, addresses, or supported application-specific fields.

The `ip nbar custom` command provides this functionality on supported platforms.

See [NBAR2 Protocol Packs and Custom Applications](nbar2.md).

---

## NBAR in an End-to-End QoS Design

NBAR is typically most valuable where application recognition is needed, rather than on every device along the traffic path.

A typical design follows this process:

1. **Recognize the application** using NBAR at the network edge.
2. **Classify and mark** traffic using an MQC policy.
3. **Preserve the DSCP marking** through the network where appropriate.
4. **Apply queuing, policing, or other QoS treatment** based on the packet's class or DSCP marking.

This avoids requiring every router to repeat detailed application recognition. Core devices can generally use simpler DSCP-based **Behavior Aggregate (BA) classification**.

---

## Limitations and Design Considerations

NBAR is more flexible than basic port-based classification, but it is not a universal application detector.

- **IP traffic:** NBAR classifies IP traffic, not arbitrary non-IP traffic. MPLS-labeled packets must generally be classified before encapsulation or at a supported point where the IP packet is accessible.
- **Encrypted traffic:** Encryption can hide the information needed to identify a specific application.
- **Stateful inspection:** Recognition of some protocols requires observing multiple packets in a flow. Asymmetric paths can prevent the necessary information from being available to the same device.
- **Tunnels:** Classification depends on whether NBAR can inspect the original IP traffic at the configured attachment point. Classification before encapsulation or after decapsulation may be necessary.
- **Platform resources:** Application recognition uses processing and memory resources, and supported protocols and attachment points depend on the device.

The practical approach is to **classify and mark traffic where the application information is visible**, then use those markings for downstream QoS treatment.

---

## Verification

The following commands are useful when working with NBAR:

| Command | Purpose |
|---|---|
| `show ip nbar protocol-discovery` | Displays Protocol Discovery statistics. |
| `show ip nbar protocol-discovery interface <interface>` | Displays application statistics for a particular interface. |
| `show ip nbar protocol-pack active` | Displays the currently active NBAR2 protocol pack. |
| `show ip nbar version` | Displays NBAR software and protocol-pack version information. |
| `show ip nbar port-map` | Displays NBAR protocol-to-port mappings. |
| `show class-map` | Verifies MQC class-map match criteria. |
| `show policy-map interface` | Displays applied MQC policies and class statistics. |

Use **Protocol Discovery** to identify what traffic the device recognizes, and **`show policy-map interface`** to confirm that the configured QoS policy is matching traffic and applying the intended actions.

---

## Key Points

- **NBAR** identifies applications and protocols beyond ordinary address-based and port-based classification.
- It can use **static ports, stateful inspection, DPI, and available application metadata**.
- **Protocol Discovery** provides application visibility and traffic statistics without applying QoS actions.
- **`match protocol`** uses NBAR application recognition as an MQC classification criterion.
- **NBAR classifies traffic; MQC policies apply actions** such as DSCP marking, policing, and queuing.
- **NBAR2 protocol packs** allow supported devices to update application signatures separately from the IOS XE image.
- NBAR2 supports **application attributes** and **custom application definitions**.
- Application recognition may be limited by **encryption, asymmetric flows, encapsulation, and platform capabilities**.
- In a DiffServ design, NBAR is useful for **classification and marking at the edge**, while the core can use DSCP-based classification.

---

## Further Reading

- [NBAR Operation and Protocol Discovery](protocol-discovery.md)
- [NBAR Classification with MQC](mqc-classification.md)
- [NBAR2 Protocol Packs and Custom Applications](nbar2.md)
- [4.4.b Classification and Trust Boundary](../classification/overview.md)
- [4.4.d Marking DSCP Values in IPv4 and IPv6 Headers](../marking/overview.md)
- [4.4.h End-to-End Layer 3 QoS Using MQC](../mqc/overview.md)

### Cisco Documentation

- [Cisco: Classifying Network Traffic Using NBAR, IOS XE 17.x](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/qos/b-quality-of-service/m_clsfy-traffic-nbar-0.html)
- [Cisco: Enabling Protocol Discovery, IOS XE 17.x](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/qos/b-quality-of-service/m_nbar-protocl-discvry-0.html)
- [Cisco: Configuring NBAR Using the MQC](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/quality-of-service/quality-of-service/m_nbar-mqc.html)
- [Cisco: NBAR2 Protocol Pack](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/quality-of-service/quality-of-service/m_nbar-protocl-pack-1.html)
- [Cisco: NBAR2 Custom Protocol, IOS XE 17.x](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/qos/b-quality-of-service/m_nbar2-custm-protocl.html)
