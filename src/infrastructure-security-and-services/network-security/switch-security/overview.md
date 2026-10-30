# 4.2.a Switch Security Features

Cisco switches provide several **Layer 2 security features** that control which traffic can enter or cross a switched network. Some features filter packets using access lists, some limit excessive traffic, and others validate host addresses or restrict which devices can use a switch port.

This section covers the six switch security topics in the CCIE Enterprise Infrastructure blueprint: **VACLs and PACLs, Storm Control, DHCP Snooping and Option 82, IP Source Guard, Dynamic ARP Inspection, and Port Security**. It also covers **Private VLAN (PVLAN)**, which covers an additional method of isolating devices within a Layer 2 network.

The features solve different problems and can be used together. For example, **DHCP Snooping** can learn legitimate host address bindings, **IP Source Guard** can use those bindings to filter spoofed IP traffic, and **Dynamic ARP Inspection** can use them to validate ARP messages.

---

## Switch Security Features at a Glance

| Feature | Primary purpose | Main scope or basis |
| --- | --- | --- |
| **VLAN ACL (VACL)** | Filter traffic within a VLAN using a VLAN access map. | VLAN-wide policy |
| **Port ACL (PACL)** | Filter traffic arriving on a Layer 2 switch port. | Ingress switch port |
| **Storm Control** | Limit excessive broadcast, multicast, or unknown-unicast traffic. | Ingress traffic rate on a port |
| **DHCP Snooping** | Block unauthorized DHCP server messages and learn legitimate DHCP bindings. | VLAN enablement and trusted/untrusted ports |
| **DHCP Option 82** | Carry information identifying where a DHCP client is connected. | DHCP relay-agent information |
| **IP Source Guard (IPSG)** | Block IP packets whose source address does not match an authorized binding. | Host IP/MAC bindings on a Layer 2 port |
| **Dynamic ARP Inspection (DAI)** | Reject ARP messages with invalid IP-to-MAC bindings. | ARP validation on untrusted ports |
| **Port Security** | Restrict which and how many source MAC addresses can use a switch port. | Secure MAC addresses per port |
| **Private VLANs (PVLANs)** | Restrict Layer 2 communication among ports within a PVLAN domain. | Primary/secondary VLANs and PVLAN port roles |

**Filtering**, **validation**, **traffic suppression**, and **isolation** are not interchangeable. A feature that blocks forged ARP messages does not necessarily block a host from sending excessive broadcast traffic, for example.

---

## VACLs and PACLs

**Access control lists** are not limited to routed interfaces. Catalyst switches also support ACL-based filtering in the switching path.

- **VACL (VLAN ACL):** Uses a VLAN access map to control traffic associated with a VLAN. It is especially useful for filtering **bridged traffic between hosts in the same VLAN**, which an ordinary routed-interface ACL does not inspect merely because the hosts share an SVI.
- **PACL (Port ACL):** Applies an IP or MAC ACL to a **Layer 2 switch port**, filtering traffic entering that port.

A useful distinction is the point where the policy is applied:

```text
PACL:  Traffic enters a particular Layer 2 port -> Port ACL
VACL:  Traffic is processed within a selected VLAN -> VLAN access map
RACL:  Traffic enters or leaves a Layer 3 interface -> Routed ACL
```

VACLs are not configured with a simple inbound or outbound direction like routed ACLs. PACLs are applied on ingress. Where multiple ACL types apply, **processing order and precedence matter**, so filtering behavior must be verified rather than assumed.

See [VACL and PACL](vacl-pacl.md).

---

## Storm Control

**Storm Control** protects switch ports from excessive traffic that could consume bandwidth or overwhelm a Layer 2 network.

It can monitor and suppress selected traffic types:

- **Broadcast:** For example, excessive ARP broadcasts.
- **Multicast:** Excessive multicast traffic received on a port.
- **Unknown unicast:** Unicast frames flooded because the destination MAC address is not known in the forwarding table.

The switch compares measured traffic against a configured threshold, which can be expressed using different units depending on the platform and command options. When the threshold is exceeded, the switch can suppress traffic and may support additional actions such as shutting down the port.

Storm Control controls **traffic volume**. It does not determine whether a particular source MAC address, IP address, or ARP message is legitimate.

See [Storm Control](storm-control.md).

---

## DHCP Snooping and DHCP Option 82

**DHCP Snooping** helps prevent unauthorized DHCP servers from supplying incorrect addressing information to clients. It distinguishes between **trusted** and **untrusted** switch ports:

- **Untrusted ports:** Normally face DHCP clients. DHCP server messages received on these ports are rejected.
- **Trusted ports:** Normally lead toward an authorized DHCP server or relay agent and can receive server-side DHCP messages.

DHCP Snooping is enabled globally and for the relevant VLANs. The switch examines DHCP exchanges and can maintain a **DHCP Snooping binding database**, recording associations such as a client's **IP address, MAC address, VLAN, interface, and lease time**.

That database is important because other security features can use it to validate host traffic.

### DHCP Option 82

**DHCP Option 82**, the *Relay Agent Information Option*, carries information about where a DHCP client is connected. Common suboptions include:

- **Circuit ID:** Identifies a circuit, port, or other client attachment point.
- **Remote ID:** Identifies the access device or another administratively defined remote identifier.

A DHCP relay agent or a DHCP Snooping switch can insert Option 82 information, depending on the configuration. The DHCP server can use it when making address-assignment decisions.

**Option 82 is not itself a security binding database or a replacement for DHCP Snooping.** Trust settings and handling of packets that already contain Option 82 are important in designs with multiple switches.

See [DHCP Snooping and DHCP Option 82](dhcp-snooping/overview.md) and [DHCP Option 82](dhcp-snooping/option82.md).

---

## IP Source Guard

**IP Source Guard (IPSG)** limits the IP traffic a host can send through a Layer 2 port by checking its **source IP address**, and optionally its **source MAC address**, against authorized bindings.

These bindings commonly come from the **DHCP Snooping binding database**, but suitable **static IP source bindings** can be configured for devices that do not use DHCP.

For example, if a host is authorized to use `192.0.2.10` on a particular port, IPSG can prevent that port from accepting IP packets claiming an unauthorized source address.

IPSG validates **IP data traffic**. It is not the mechanism used to inspect ARP messages; that is the role of DAI.

See [IP Source Guard](ip-source-guard.md).

---

## Dynamic ARP Inspection

**Dynamic ARP Inspection (DAI)** protects against ARP spoofing and ARP poisoning by validating ARP messages received on **untrusted ports**.

A switch can compare information in ARP packets against the **DHCP Snooping binding database** to determine whether the claimed IP-to-MAC association is valid. **ARP ACLs** can also be used to authorize statically addressed devices, where appropriate.

DAI can additionally perform configurable checks on fields in ARP packets and can rate-limit ARP traffic.

A crucial distinction is that DAI applies to **ARP messages**, whereas IPSG applies to **IP traffic**. They can use the same source of binding information but protect against different types of spoofing.

See [Dynamic ARP Inspection](dynamic-arp-inspection.md).

---

## Port Security

**Port Security** controls the source MAC addresses permitted on a switch port. It can limit the number of secure addresses and specify which addresses are authorized.

Secure MAC addresses can be:

- **Static:** Manually configured.
- **Dynamic:** Learned automatically as secure addresses.
- **Sticky:** Learned automatically and added to the running configuration.

When a violation occurs, the configured mode determines how the switch reacts. The principal modes are **protect**, **restrict**, and **shutdown**. Port Security also supports MAC address aging and violation monitoring.

Port Security is **MAC-based port admission control**, not IP source validation or ARP inspection. A host using an allowed source MAC address could still attempt IP or ARP spoofing unless another feature blocks it.

See [Port Security](port-security.md).

---

## How DHCP Snooping, IPSG, and DAI Work Together

These three features are closely related and are best understood as a coordinated access-layer design.

Consider a host connected to an untrusted access port:

```text
                   Authorized DHCP Server
                             |
                         Trusted link
                             |
                           SW1
                     DHCP Snooping
                      Binding Table
                             |
                       Untrusted port
                             |
                        Host A
                   192.0.2.10 / MAC A
```

1. **DHCP Snooping** permits the legitimate DHCP exchange and learns that Host A's IP address and MAC address belong on the specified VLAN and port.
2. **IP Source Guard** uses the authorized binding to validate the source of IP traffic entering that port.
3. **Dynamic ARP Inspection** uses the binding to validate ARP messages entering untrusted ports.

| Host action | Relevant feature |
| --- | --- |
| A rogue DHCP server sends DHCP Offers on an untrusted port. | **DHCP Snooping** rejects the unauthorized server messages. |
| Host A sends an IP packet claiming an unauthorized source IP address. | **IP Source Guard** can drop the packet. |
| Host A sends an ARP reply claiming an unauthorized IP-to-MAC mapping. | **Dynamic ARP Inspection** can drop the ARP message. |
| Host A sends excessive broadcast traffic. | **Storm Control** can suppress traffic above its configured threshold. |
| A different source MAC address appears on a secured port. | **Port Security** can take the configured violation action. |

**DHCP Snooping does not automatically enable IPSG or DAI.** Each feature must be configured separately. Likewise, statically addressed devices require appropriate binding or ARP ACL arrangements where these checks are enforced.

The **trusted/untrusted settings for DHCP Snooping and DAI are separate**. Trust only the interfaces that should be exempt from the respective checks. Incorrect trust placement can bypass security checks or prevent legitimate hosts from communicating.

---

## Private VLANs

**Private VLANs (PVLANs)** offer a different kind of Layer 2 protection: **traffic isolation** between hosts that can share the same IP subnet and default gateway.

A PVLAN consists of a **primary VLAN** and one or more **secondary VLANs**:

- **Isolated VLAN:** Host ports cannot communicate directly with other PVLAN host ports.
- **Community VLAN:** Host ports can communicate with members of the same community and with appropriate promiscuous ports.
- **Promiscuous port:** Provides access to shared services, such as a default gateway, for associated secondary VLANs.

PVLANs are **not explicitly listed among the six 4.2.a switch-security blueprint items**. They remain in this mdBook as a useful supplementary switching topic. Unlike DHCP Snooping, DAI, or IPSG, PVLANs do not check whether a host's claimed IP or MAC address is legitimate; they restrict **which Layer 2 ports may communicate**.

See the [Private VLANs](private-vlans/overview.md) chapter for VLAN types, forwarding rules, configuration, trunks, and Layer 3 connectivity.

---

## Key Points

- **VACLs and PACLs** filter traffic using VLAN-wide and ingress-port policies, respectively.
- **Storm Control** suppresses excessive broadcast, multicast, or unknown-unicast traffic.
- **DHCP Snooping** blocks unauthorized DHCP server messages and builds a binding database.
- **DHCP Option 82** identifies client attachment information in DHCP messages.
- **IP Source Guard** validates source IP addresses, optionally together with MAC addresses, using authorized bindings.
- **Dynamic ARP Inspection** validates ARP messages, commonly using DHCP Snooping bindings or ARP ACLs.
- **Port Security** limits and controls source MAC addresses on a switch port.
- **DHCP Snooping, IPSG, and DAI** are complementary features, not one automatically enabled security mechanism.
- **PVLANs** provide Layer 2 host isolation and are included as supplementary material.

---

## References

- [Cisco: Catalyst 9300 IPv4 ACLs (Port ACLs and VLAN Maps)](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9300/software/release/17-17/configuration_guide/sec/b_1717_sec_9300_cg/configuring_ipv4_acls.html)
- [Cisco: Catalyst 9300 Configuring Port-Based Traffic Control (Storm Control)](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9300/software/release/17-2/configuration_guide/sec/b_172_sec_9300_cg/configuring_port_based_traffic_control.html)
- [Cisco: Catalyst 9300 Configuring DHCP (DHCP Snooping and Option 82)](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9300/software/release/17-13/configuration_guide/ip/b_1713_ip_9300_cg/configuring_dhcp.html)
- [Cisco: Troubleshoot Dynamic ARP Inspection and IP Source Guard](https://www.cisco.com/c/en/us/support/docs/switches/lan-switch-software/222274-troubleshoot-dynamic-arp-inspection-dai.html)
- [Cisco: Catalyst 9300 Configuring Dynamic ARP Inspection](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9300/software/release/17-18/configuration_guide/sec/b_1718_sec_9300_cg/configuring_dynamic_arp_inspection.html)
- [Cisco: Catalyst 9300 Port Security](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9300/software/release/17-17/configuration_guide/sec/b_1717_sec_9300_cg/port_security.html)
- [Cisco: Catalyst 9300 Configuring Private VLANs](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9300/software/release/17-14/configuration_guide/vlan/b_1714_vlan_9300_cg/configuring_private_vlans.html)
