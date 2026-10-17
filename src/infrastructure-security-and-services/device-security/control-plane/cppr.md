# Control Plane Protection and Subinterfaces

**Control Plane Protection (CPPr)** extends Control Plane Policing (CoPP) by dividing control-plane traffic into separate categories and allowing independent policies to be applied to each category.

Traditional CoPP applies an **aggregate policy** to traffic entering the control plane.

CPPr adds three logical **control-plane subinterfaces**:

- **Host:** IP traffic destined for the router itself.
- **Transit:** IP traffic traversing the router that requires software forwarding.
- **CEF-Exception:** Packets requiring exceptional processing and certain non-IP control traffic.

CPPr also introduces **port filtering** and **queue thresholding**, which provide additional protection for traffic entering through the host subinterface.

The primary benefit is **more granular control over CPU-bound traffic**, allowing different protection policies for different packet-processing paths.

---

## Aggregate CoPP vs. CPPr

Traditional CoPP uses the aggregate control-plane interface:

```text
R1(config)# control-plane
R1(config-cp)# service-policy input COPP-IN
```

This applies the policy to applicable incoming control-plane traffic without separating it into host, transit, and CEF-exception categories.

CPPr allows additional policies to be attached to individual logical subinterfaces:

```text
R1(config)# control-plane host
R1(config-cp-host)# service-policy input CPPR-HOST

R1(config)# control-plane transit
R1(config-cp-transit)# service-policy input CPPR-TRANSIT

R1(config)# control-plane cef-exception
R1(config-cp-cef-exception)# service-policy input CPPR-CEF-EXCEPTION
```

The subinterfaces are **logical control-plane processing paths**, not ordinary physical or VLAN subinterfaces. They do not require interface numbers or IP addresses.

### Processing Order

On platforms supporting aggregate CoPP and CPPr together, the aggregate policy is applied first.

Packets permitted by the aggregate policy can then be processed by the corresponding CPPr subinterface policy.

```text
               CPU-Bound Traffic
                       |
                       v
                Aggregate CoPP
                       |
                 Permit / Drop
                       |
                 Permitted Traffic
                       |
                       v
                 CPPr Classifier
                       |
            +----------+----------+
            |          |          |
            v          v          v
           Host      Transit   CEF-Exception
            |          |          |
            v          v          v
        Host Policy   Transit    Exception
                      Policy      Policy
            |          |          |
            +----------+----------+
                       |
                       v
                 CPU Processing
```

A packet is classified into **one** CPPr category, not all three.

If the aggregate CoPP policy drops a packet, it does not proceed to the CPPr subinterface policy.

If both policies permit the packet, it can proceed toward CPU processing.

The aggregate policy and subinterface policy therefore provide separate layers of policing.

---

## Control-Plane Host Subinterface

The **host subinterface** handles IP traffic directly destined for the router.

Examples include:

- SSH and Telnet management connections.
- SNMP requests.
- BGP packets.
- OSPF and EIGRP routing protocol packets.
- Traffic terminating on the router, such as supported tunnel termination traffic.

```text
PC1
 |
 | SSH to 10.0.12.1
 |
 v
R1 Ethernet0/0
 |
 v
CPPr Host
 |
 v
SSH Process
```

The important characteristic is that the **router is the destination of the IP packet**.

For example, a packet destined for R1's Ethernet0/0 address is host traffic, assuming it is not redirected into an exception-processing path.

### Host Classification Exceptions

Not every packet intended for the router is necessarily processed through the host subinterface.

In Cisco's CPPr architecture, certain packets are classified as CEF-exception traffic instead.

Examples include:

- IP packets with IP options.
- IP packets with TTL values of 0 or 1.
- ARP packets.
- Certain non-IP Layer 2 control packets.

Therefore, **the host subinterface does not necessarily see all traffic related to the router itself**.

For example, ARP is classified through the CEF-exception path rather than the host path in the documented CPPr implementation.

### Configuring Host Policing

Suppose R1 should police incoming SSH traffic at 256 kbps.

First, create the ACL:

```text
R1(config)# ip access-list extended CPPR-SSH-ACL
R1(config-ext-nacl)# permit tcp any any eq 22
```

Next, create a class map:

```text
R1(config)# class-map match-all CPPR-SSH
R1(config-cmap)# match access-group name CPPR-SSH-ACL
```

Create a policy map:

```text
R1(config)# policy-map CPPR-HOST
R1(config-pmap)# class CPPR-SSH
R1(config-pmap-c)# police cir 256000 bc 8000 conform-action transmit exceed-action drop
```

Finally, apply the policy to the host subinterface:

```text
R1(config)# control-plane host
R1(config-cp-host)# service-policy input CPPR-HOST
```

The policy limits matching SSH traffic entering the host path.

The `class-default` class has no configured policing action, so other host traffic is not restricted by this policy.

**The main difference from traditional CoPP is the attachment point:** `control-plane host` instead of `control-plane`.

---

## Control-Plane Transit Subinterface

The **transit subinterface** receives IP traffic that is **not destined for the router itself** but requires software forwarding by the route processor.

For example, certain tunnel-processing or software-switched transit packets may use this path.

```text
PC1                          PC2
 |                            ^
 | Transit Packet             |
 v                            |
R1 Ethernet0/0                |
 |                            |
 v                            |
CPPr Transit                  |
 |                            |
 v                            |
Software Forwarding ----------+
```

The router processes the packet in software and then forwards it toward the destination.

### Transit vs. Normal Forwarding

The distinction is important:

| Packet | CPPr Transit? |
|---|---|
| Packet forwarded normally by CEF or hardware | No |
| Packet destined for an SSH service on the router | No, normally Host |
| Transit packet requiring software forwarding | Yes, when classified on the Transit path |
| Packet redirected because of a CEF exception | No, CEF-Exception |

**Normal transit traffic does not automatically enter the CPPr transit subinterface.**

Most normally forwarded packets remain in the data plane and never reach the CPU.

### Configuring Transit Policing

On platforms supporting the transit subinterface, a separate policy can be applied to this traffic category.

For example, create a policy that polices all traffic entering the transit path at 8 Mbps:

```text
R1(config)# policy-map CPPR-TRANSIT
R1(config-pmap)# class class-default
R1(config-pmap-c)# police cir 8000000 bc 250000 conform-action transmit exceed-action drop
```

Apply it:

```text
R1(config)# control-plane transit
R1(config-cp-transit)# service-policy input CPPR-TRANSIT
```

This limits traffic classified into the transit processing path.

It **does not limit ordinary hardware-forwarded transit traffic**.

> This rate is only an example. Policing all software-switched transit traffic can disrupt legitimate forwarding if the rate is too low. Verify the traffic and platform behavior before enforcing drops.

---

## Control-Plane CEF-Exception Subinterface

The **CEF-exception subinterface** handles packets that require exceptional processing or are directly enqueued for control-plane processing outside the normal host path.

Examples include:

- ARP packets.
- Certain Layer 2 control and keepalive packets.
- IP packets with IP options.
- IP packets with TTL values of 0 or 1.
- Packets redirected from CEF because they require special processing.

```text
Incoming Packet
       |
       v
 Forwarding Path
       |
       | Exception or direct punt
       v
CPPr CEF-Exception
       |
       v
 Exception Processing
```

The CEF-exception category is different from both host and transit traffic.

**It describes an exceptional processing path rather than simply whether the packet's destination is local or remote.**

### Example: Expired TTL

Suppose R1 receives an IPv4 packet with a TTL of 1.

The packet cannot be forwarded normally because its TTL would expire.

Depending on the platform and processing path, it may be classified as CEF-exception traffic.

```text
PC1 ---- TTL=1 ----> R1
                       |
                       v
                 CEF-Exception
                       |
                       v
                CPU Processing
```

This allows exceptional traffic to be policed separately from ordinary management or routing protocol traffic.

### Configuring CEF-Exception Policing

On platforms supporting the CEF-exception subinterface, a separate policy can be applied.

For example, initially configure a monitoring-only policer for this path:

```text
R1(config)# policy-map CPPR-CEF-EXCEPTION
R1(config-pmap)# class class-default
R1(config-pmap-c)# police cir 4000000 bc 125000 conform-action transmit exceed-action transmit
```

Apply it:

```text
R1(config)# control-plane cef-exception
R1(config-cp-cef-exception)# service-policy input CPPR-CEF-EXCEPTION
```

Both conforming and exceeding traffic is transmitted, allowing policing statistics to be observed before enabling drops.

> **Be particularly careful when policing CEF-exception traffic.** This path can include essential ARP and other control packets. Dropping excessive traffic indiscriminately can disrupt connectivity or control-plane operation.

---

## Aggregate CoPP and CPPr Policies Together

An aggregate CoPP policy can coexist with separate CPPr subinterface policies on supported platforms.

For example:

```text
R1(config)# control-plane
R1(config-cp)# service-policy input COPP-AGGREGATE

R1(config)# control-plane host
R1(config-cp-host)# service-policy input CPPR-HOST

R1(config)# control-plane transit
R1(config-cp-transit)# service-policy input CPPR-TRANSIT

R1(config)# control-plane cef-exception
R1(config-cp-cef-exception)# service-policy input CPPR-CEF-EXCEPTION
```

Each policy operates on its respective control-plane processing path.

| Policy | Applies To |
|---|---|
| `COPP-AGGREGATE` | All applicable incoming control-plane traffic |
| `CPPR-HOST` | Host traffic permitted through the aggregate policy |
| `CPPR-TRANSIT` | Transit traffic permitted through the aggregate policy |
| `CPPR-CEF-EXCEPTION` | CEF-exception traffic permitted through the aggregate policy |

### Example: Two Layers of Policing

Suppose a packet is classified as SSH host traffic.

The processing sequence is:

1. The packet encounters the aggregate CoPP policy.
2. If the aggregate policy drops it, processing stops.
3. If the aggregate policy permits it, the packet enters the host path.
4. The host policy applies its own policing rules.
5. If the host policy permits it, the packet proceeds toward the SSH process.

The packet must pass both applicable policies.

**Tokens from an aggregate CoPP policer and a CPPr subinterface policer are maintained independently.** Passing one policer does not guarantee that the packet passes the other.

---

## CPPr-Specific Protection Features

In addition to ordinary MQC policing, CPPr introduces two specialized mechanisms.

| Feature | Purpose | Supported Subinterface |
|---|---|---|
| **Port filtering** | Drops traffic directed to selected or closed TCP/UDP ports | Host only |
| **Queue thresholding** | Limits the number of queued packets associated with particular protocols | Host only |

These mechanisms use special MQC policy types:

```text
class-map type port-filter
policy-map type port-filter
```

```text
class-map type queue-threshold
policy-map type queue-threshold
```

They are applied under the host subinterface using the corresponding service-policy type.

For example, the attachment syntax is:

```text
R1(config)# control-plane host
R1(config-cp)# service-policy type port-filter input <policy-name>
```

```text
R1(config)# control-plane host
R1(config-cp)# service-policy type queue-threshold input <policy-name>
```

These are distinct from ordinary policing service policies.

Configuration details are covered in [CPPr Port Filtering and Queue Thresholding](port-filtering.md).

---

## CPPr Restrictions and Platform Support

CPPr is not universally supported across all IOS XE routers and switches.

### IPv4 Input Path

Cisco documents traditional CPPr as supporting the **IPv4 input path only**.

This is an important distinction from aggregate CoPP, which supports IPv6 traffic classification on certain IOS XE platforms.

**Do not assume that an IPv6 packet is protected by a CPPr host, transit, or CEF-exception policy.**

IPv6 control-plane protection must be verified for the specific platform and feature implementation.

### CEF Dependency

CPPr subinterface processing depends on **Cisco Express Forwarding (CEF)** to redirect IP packets into the appropriate processing paths.

Cisco documents that disabling CEF globally removes active protection and policing policies on the CPPr subinterfaces.

Aggregate CoPP policies can continue functioning independently.

Therefore:

- CPPr subinterface policing depends on CEF.
- Aggregate CoPP does not share this specific dependency.

### Input Policing Only

The documented CPPr host, transit, and CEF-exception subinterfaces support **input policing**.

```text
R1(config)# control-plane host
R1(config-cp)# service-policy input CPPR-HOST
```

Output CoPP, where supported, is associated with the aggregate control-plane interface rather than these CPPr subinterfaces.

### Platform-Specific Commands

Not every IOS XE platform exposes all three CPPr subinterfaces.

For example, Cisco's command reference documents a restricted `control-plane [host]` syntax for ASR 1000 routers.

Other hardware-forwarding platforms provide different punt policing architectures.

In particular, Catalyst 9000 switches use **System CoPP**, rather than the traditional host/transit/CEF-exception configuration model.

Always verify available commands on the target platform.

---

## IOS XE Verification

The main verification command is:

```text
R1# show policy-map control-plane
```

On platforms supporting CPPr subinterfaces, additional options can display each policy independently.

### Display All Control-Plane Policies

```text
R1# show policy-map control-plane all
```

This can display aggregate and subinterface policy information.

### Host Policy

```text
R1# show policy-map control-plane host
```

Displays the policy attached to the host subinterface and its statistics.

### Transit Policy

```text
R1# show policy-map control-plane transit
```

Displays the policy attached to the transit subinterface.

### CEF-Exception Policy

```text
R1# show policy-map control-plane cef-exception
```

Displays the policy attached to the CEF-exception subinterface.

### Display Control-Plane Features

On supported platforms:

```text
R1# show control-plane features
```

Displays enabled control-plane protection features.

Additional commands include:

```text
R1# show control-plane aggregate
R1# show control-plane host
R1# show control-plane transit
R1# show control-plane cef-exception
R1# show control-plane counters
```

These commands and their output vary by IOS XE platform and release.

### Troubleshooting Missing Counters

If a CPPr class does not show expected packet matches, consider the following:

1. **Wrong subinterface:** The packet might be classified as CEF-exception rather than host traffic.
2. **Normal forwarding:** The packet might be forwarded entirely by the data plane and never enter the transit subinterface.
3. **Aggregate policy drops:** An aggregate policy might discard the packet before subinterface processing.
4. **ACL mismatch:** The class map might not match the intended traffic.
5. **Platform limitations:** The platform might not support the expected CPPr processing path.

For example, ARP traffic would not be expected to increment a host-only policer merely because the ARP request concerns the router's IP address.

---

## Key Points

- **CPPr extends CoPP** by providing separate policing for three logical control-plane subinterfaces.
- **Host:** IP traffic directly destined for the router, such as SSH, SNMP, BGP, OSPF, and EIGRP.
- **Transit:** IP traffic traversing the router that requires software forwarding.
- **CEF-Exception:** Traffic requiring exceptional processing, including certain packets with IP options, low TTL values, ARP, and non-IP control traffic.
- Normal hardware-forwarded transit traffic does **not** pass through the CPPr transit subinterface.
- The three CPPr traffic categories are mutually exclusive.
- Configure the subinterfaces with `control-plane host`, `control-plane transit`, and `control-plane cef-exception`.
- Apply a subinterface policing policy with **`service-policy input <policy-map>`**.
- Aggregate CoPP and CPPr subinterface policies can coexist on supported platforms.
- **Aggregate CoPP is applied before the CPPr subinterface policy.**
- Packets dropped by aggregate CoPP do not reach the subinterface policer.
- CPPr policers maintain independent token buckets from aggregate CoPP policers.
- **Port filtering** and **queue thresholding** are supported only on the host subinterface in the documented CPPr architecture.
- Traditional CPPr is documented as **IPv4 input-path only**.
- CPPr subinterface protection depends on **CEF**.
- CPPr subinterfaces support **input policing**, not output policing, in the documented implementation.
- Platform support varies, and some IOS XE devices do not expose all three CPPr subinterfaces.
- Use `show policy-map control-plane all` and `show policy-map control-plane <subinterface>` to verify policies and counters.

---

## References

- [Cisco - Control Plane Protection](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/qos_plcshp/configuration/15-mt/qos-plcshp-15-mt-book/qos-plcshp-cpp.html)
- [Cisco IOS XE 17.x - Control Plane Policing](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/qos/b-quality-of-service/m_qos-plcshp-ctrl-pln-plc-0.html)
- [Cisco - Control Plane Command Reference](https://www.cisco.com/c/en/us/td/docs/ios/qos/command/reference/qos_book/qos_a1.html)
- [Cisco IOS XE Software Hardening Guide](https://sec.cloudapps.cisco.com/security/center/resources/IOS_XE_hardening)
- [Cisco - Harden IOS Devices](https://www.cisco.com/c/en/us/support/docs/ip/access-lists/13608-21.html)