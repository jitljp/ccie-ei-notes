# NBAR Classification with MQC

**Network Based Application Recognition (NBAR)** can identify IP traffic by its application or protocol, rather than relying only on IP addresses and TCP/UDP port numbers.

Cisco IOS XE integrates NBAR with the **Modular QoS Command-Line Interface (MQC)**. This allows applications recognized by NBAR to be placed into QoS classes and given different forwarding treatments.

For example, a router could:
- Identify Real-time Transport Protocol (RTP) traffic and mark it with DSCP AF41.
- Identify HTTP traffic and mark it with DSCP AF21.
- Identify FTP traffic and mark it with DSCP CS1.
- Use the resulting DSCP values to select output queues on a WAN interface.

NBAR **classifies** traffic. MQC determines what **action** is applied to that traffic.

---

## NBAR and MQC

An NBAR-based QoS policy uses the same three MQC components as other QoS policies:

| Component | Purpose |
|---|---|
| **Class map** | Selects an application or protocol using `match protocol` |
| **Policy map** | Assigns QoS actions, such as marking, policing, or queuing |
| **Service policy** | Applies the policy to an interface in the input or output direction |

The overall process is:

```text
               Incoming Packets
                      |
                      v
          +-----------------------+
          |  NBAR Classification  |
          |                       |
          | match protocol rtp    |
          | match protocol http   |
          | match protocol ftp    |
          +-----------------------+
                      |
                      v
          +-----------------------+
          |      MQC Policy       |
          |                       |
          | RTP  -> Set AF41      |
          | HTTP -> Set AF21      |
          | FTP  -> Set CS1       |
          +-----------------------+
                      |
                      v
               Forward Packets
```

A class map by itself does **not** mark, police, or prioritize traffic. A policy map must define the desired action, and the policy must be applied to an interface.

---

## Matching Applications with `match protocol`

The **`match protocol`** command identifies traffic recognized by NBAR.

The general syntax is:

```text
R1(config)# class-map [match-all | match-any] <class-name>
R1(config-cmap)# match protocol <protocol-name>
```

For example:

```text
R1(config)# class-map match-all RTP-MEDIA
R1(config-cmap)# match protocol rtp
```

This class identifies traffic that NBAR recognizes as **RTP**.

The match is based on NBAR's recognition of the protocol. It is not equivalent to matching a particular UDP port range with an ACL.

### Examples of Protocol Matching

| Command | Traffic identified |
|---|---|
| `match protocol http` | HTTP |
| `match protocol ftp` | FTP |
| `match protocol rtp` | RTP |
| `match protocol dns` | DNS |
| `match protocol ssh` | SSH |

The list of recognized applications depends on the NBAR engine and its installed protocol pack. Check the router's `match protocol ?` options for the available protocol names.

**Recognizing a protocol does not necessarily identify the specific business application.** For example, generic RTP matching can include both audio and video streams. Do not assume that all RTP traffic should automatically receive DSCP EF.

### NBAR vs. Port-Based Classification

Consider an application using a dynamic TCP port.

A traditional ACL might require a fixed port:

```text
R1(config)# ip access-list extended APP-ACL
R1(config-ext-nacl)# permit tcp any any eq 5000
```

An NBAR class can instead select a supported application based on its recognized protocol characteristics:

```text
R1(config)# class-map match-all APP-TRAFFIC
R1(config-cmap)# match protocol <supported-application>
```

NBAR can use port information, protocol signatures, and stateful inspection, depending on the application. This makes it useful for applications that use dynamic ports or share ports with unrelated traffic.

---

## `match-all` vs. `match-any`

Like other MQC classes, NBAR classes support **`match-all`** and **`match-any`**.

| Mode | Behavior |
|---|---|
| `match-all` | All match statements must match (AND) |
| `match-any` | At least one match statement must match (OR) |

If neither is specified, **`match-all` is the default**.

### Matching Multiple Applications

To place HTTP and FTP into the same class:

```text
R1(config)# class-map match-any DATA-APPS
R1(config-cmap)# match protocol http
R1(config-cmap)# match protocol ftp
```

This matches traffic recognized as **HTTP OR FTP**.

Using `match-all` here would require traffic to match both protocol criteria, which is not the intended classification.

### Combining an Application with an ACL

Suppose only RTP traffic originating from `10.10.10.0/24` should receive a particular marking.

First, define an ACL identifying the source subnet:

```text
R1(config)# ip access-list standard BRANCH-USERS
R1(config-std-nacl)# permit 10.10.10.0 0.0.0.255
```

Then combine the ACL and NBAR criteria:

```text
R1(config)# class-map match-all BRANCH-RTP
R1(config-cmap)# match access-group name BRANCH-USERS
R1(config-cmap)# match protocol rtp
```

Both conditions must be satisfied:
- The packet's source IP address must belong to `10.10.10.0/24`.
- NBAR must recognize the packet as RTP.

This allows a QoS policy to distinguish the same application coming from different users, subnets, or sites.

---

## Applying QoS Actions to NBAR Classes

After defining the class maps, create a **policy map** that specifies the treatment for each class.

### Example: Mark Applications at the Network Edge

Suppose R1 receives unmarked traffic from a LAN on `GigabitEthernet1` and forwards it toward a WAN.

The requirements are:

| Application | Marking | DSCP decimal value |
|---|---|---:|
| RTP media | AF41 | 34 |
| HTTP | AF21 | 18 |
| FTP | CS1 | 8 |

**Step 1: Create NBAR class maps.**

```text
R1(config)# class-map match-all RTP-MEDIA
R1(config-cmap)# match protocol rtp
R1(config-cmap)# exit
R1(config)# class-map match-all WEB
R1(config-cmap)# match protocol http
R1(config-cmap)# exit
R1(config)# class-map match-all FILE-TRANSFER
R1(config-cmap)# match protocol ftp
```

**Step 2: Configure DSCP marking actions.**

```text
R1(config)# policy-map NBAR-MARK
R1(config-pmap)# class RTP-MEDIA
R1(config-pmap-c)# set dscp af41
R1(config-pmap-c)# exit
R1(config-pmap)# class WEB
R1(config-pmap-c)# set dscp af21
R1(config-pmap-c)# exit
R1(config-pmap)# class FILE-TRANSFER
R1(config-pmap-c)# set dscp cs1
```

**Step 3: Attach the policy to the LAN-facing interface.**

```text
R1(config)# interface GigabitEthernet1
R1(config-if)# service-policy input NBAR-MARK
```

R1 classifies incoming packets using NBAR and marks matching packets with the configured DSCP values.

The policy does not change packets that fail to match the three named classes because no marking action has been configured for `class-default`.

### Class Evaluation Order

Within a policy map, classes are evaluated in their configured order. A packet is assigned to the **first class it matches**.

For example, if an application could match two classes, the earlier class determines its treatment. The router does not apply both classes' marking actions to the same packet.

Traffic that matches no named class belongs to **`class-default`**.

---

## Using DSCP for Downstream Queuing

An efficient DiffServ design performs application recognition at the **network edge**, then uses DSCP to classify traffic at congestion points.

For example:

```text
      LAN                         WAN
       |                           |
       v                           v
  GigabitEthernet1             GigabitEthernet2
       |                           |
       |             R1            |
       +-------------+-------------+
                     |
        NBAR identifies applications
                     |
        Incoming packets are marked
                     |
          WAN queues match DSCP
```

The WAN policy does not need to repeat application inspection.

**Step 1: Define DSCP-based classes.**

```text
R1(config)# class-map match-all AF41-MEDIA
R1(config-cmap)# match dscp af41
R1(config-cmap)# exit
R1(config)# class-map match-all AF21-WEB
R1(config-cmap)# match dscp af21
R1(config-cmap)# exit
R1(config)# class-map match-all CS1-BULK
R1(config-cmap)# match dscp cs1
```

**Step 2: Allocate bandwidth to those classes.**

```text
R1(config)# policy-map WAN-OUT
R1(config-pmap)# class AF41-MEDIA
R1(config-pmap-c)# bandwidth percent 30
R1(config-pmap-c)# exit
R1(config-pmap)# class AF21-WEB
R1(config-pmap-c)# bandwidth percent 20
R1(config-pmap-c)# exit
R1(config-pmap)# class CS1-BULK
R1(config-pmap-c)# bandwidth percent 5
```

**Step 3: Apply the policy in the WAN egress direction.**

```text
R1(config)# interface GigabitEthernet2
R1(config-if)# service-policy output WAN-OUT
```

The `bandwidth percent` commands allocate minimum bandwidth shares when the interface is congested. They do not limit each class to its configured percentage; classes can use additional bandwidth when it is available.

The complete design therefore uses:
- **Ingress NBAR classification** to identify applications and mark packets.
- **Egress DSCP classification** to assign packets to WAN queues.

This avoids requiring NBAR inspection at every router along the path.

See [DSCP Marking and Remarking with MQC](../marking/mqc-marking.md) and [4.4.h End-to-End Layer 3 QoS Using MQC](../mqc/overview.md).

---

## More Specific Application Matching

NBAR supports more specific matching for some protocols. A common example is **HTTP**.

### HTTP Host Matching

```text
R1(config)# class-map match-all INTRANET
R1(config-cmap)# match protocol http host intranet.example.com
```

This identifies unencrypted HTTP traffic using the specified HTTP Host field.

### HTTP URL Matching

```text
R1(config)# class-map match-all REPORTS
R1(config-cmap)# match protocol http url /reports/*
```

This identifies HTTP requests whose URL path matches the specified pattern.

### HTTP MIME-Type Matching

```text
R1(config)# class-map match-all PDF-DOWNLOADS
R1(config-cmap)# match protocol http mime application/pdf
```

This matches HTTP traffic associated with the specified MIME type.

These techniques depend on the relevant **HTTP information being visible**. For normal HTTPS traffic, the HTTP Host, URL path, and MIME headers are encrypted and cannot be read by ordinary NBAR HTTP matching.

Modern NBAR2 application identification can use other information, including supported signatures and metadata, but cannot generally examine encrypted application payloads without decryption.

See [NBAR2 Protocol Packs and Custom Applications](nbar2.md).

---

## Unknown Applications and `class-default`

NBAR cannot immediately identify every application. Some applications require **multipacket classification**, meaning NBAR must examine several packets in a flow before it can recognize the application. Other flows may never be identified because their applications are unsupported or the information needed for recognition is unavailable.

NBAR distinguishes between traffic that is **temporarily unidentified** and traffic that is **finally determined to be unknown**.

| Match criterion | What it matches |
|---|---|
| **`match protocol unknown`** | Any traffic not currently classified as a known application, including temporarily unidentified traffic **and** traffic finally determined to be unknown |
| **`match protocol unknown final`** | Only traffic that NBAR has finished attempting to classify and determined to be unknown |
| **MQC `class-default`** | Any traffic that does not match a preceding named class in the policy, whether NBAR recognizes its application or not |

### Temporarily Unknown vs. Finally Unknown

Consider two new flows:

1. **Recognizable HTTP flow:** Its initial packets may be classified as `unknown` while NBAR examines the connection. Once NBAR identifies HTTP, subsequent packets are classified as `http` instead.
2. **Unrecognizable application flow:** Its initial packets are also classified as `unknown`. If NBAR finishes its recognition process without identifying the application, subsequent traffic is classified as `unknown final`.

```text
New Flow
   |
   v
NBAR has not identified the application
   |
   +-- While still inspecting --> unknown (temporary)
   |
   +-- Classification completes
            |
            +-- Application recognized --> e.g., http
            |
            +-- Application not recognized --> unknown final
```

The `unknown` match includes **both temporary and final unknown traffic**. Adding `final` excludes the temporary unknown packets.

| Traffic at the time it is classified | `match protocol unknown` | `match protocol unknown final` |
|---|---|---|
| Packet from a flow still being examined | Match | No match |
| Packet after NBAR identifies the flow as HTTP | No match | No match |
| Packet after NBAR determines the application is unknown | Match | Match |

Packets already processed while a flow was temporarily unknown are **not retroactively reclassified** when NBAR later identifies the application.

### Configuring Unknown Traffic Matching

To match **all currently unidentified traffic**, including packets still undergoing classification:

```text
R1(config)# class-map match-all NBAR-UNKNOWN
R1(config-cmap)# match protocol unknown
```

To match **only traffic finally determined to be unknown**:

```text
R1(config)# class-map match-all NBAR-UNKNOWN-FINAL
R1(config-cmap)# match protocol unknown final
```

For example, an ingress policy could remark traffic with a final unknown classification:

```text
R1(config)# policy-map NBAR-MARKING
R1(config-pmap)# class NBAR-UNKNOWN-FINAL
R1(config-pmap-c)# set dscp cs1
R1(config)# interface GigabitEthernet1
R1(config-if)# service-policy input NBAR-MARKING
```

Using `unknown final` avoids remarking a flow **solely because NBAR has not finished examining its initial packets**. It does not mean the unidentified traffic is malicious.

### Difference from `class-default`

**`class-default` is an MQC fallback class, not an NBAR application classification.** For example, if a policy explicitly matches HTTP and `unknown final`, then another recognized application, such as DNS, can still enter `class-default` because it matched neither named class. Temporarily unknown packets can also enter `class-default` if no other class matches them.

The distinction is:
- **`unknown`:** NBAR has not identified the application at the time of classification.
- **`unknown final`:** NBAR has completed classification without identifying the application.
- **`class-default`:** No preceding MQC class matched the packet.

---

## Protocol Discovery vs. MQC Classification

**NBAR Protocol Discovery** and **NBAR-based MQC classification** both use application recognition, but they serve different purposes.

| Feature | Protocol Discovery | NBAR with MQC |
|---|---|---|
| Main purpose | Observe and report applications | Select applications for QoS treatment |
| Configuration | `ip nbar protocol-discovery` | `match protocol` in an MQC class map |
| Apply DSCP marking | No | Yes, with `set dscp` |
| Apply policing or queuing | No | Yes, through the policy map |
| Statistics | Per-application discovery statistics | Per-class QoS policy statistics |
| Verification | `show ip nbar protocol-discovery` | `show policy-map interface` |

**`ip nbar protocol-discovery` is not required for an NBAR-based MQC policy to operate.** It is useful for identifying the applications present in the network before creating a policy, and for obtaining application visibility while troubleshooting.

For more detail, see [NBAR Operation and Protocol Discovery](protocol-discovery.md).

---

## Verification

Use the following commands to verify NBAR-based MQC configuration and behavior:

| Command | Purpose |
|---|---|
| `show class-map` | Displays classes and their match statements |
| `show class-map <name>` | Displays a specific class map |
| `show policy-map` | Displays configured policy maps and QoS actions |
| `show policy-map interface <interface>` | Displays applied policies and matching statistics |
| `show ip nbar port-map` | Displays NBAR's protocol-to-port mappings |
| `show ip nbar protocol-discovery` | Displays NBAR Protocol Discovery statistics, if enabled |
| `show running-config interface <interface>` | Confirms the service policy attached to an interface |

### Verify the Class Maps

```text
R1# show class-map
```

Check that each class includes the intended `match protocol` statement.

### Verify the Service Policy

```text
R1# show policy-map interface GigabitEthernet1
```

For each class, check:
1. Whether the service policy is attached to the correct interface and direction.
2. Whether the class is matching packets.
3. Whether packet and byte counters are increasing when the relevant application generates traffic.
4. Whether the intended marking, policing, or queuing actions are shown.

The exact counters displayed depend on the platform and QoS implementation.

### Verify Protocol-to-Port Mappings

```text
R1# show ip nbar port-map
R1# show ip nbar port-map http
```

These commands show the protocol-to-port associations known to NBAR. They are helpful when diagnosing applications that use expected or nonstandard ports.

**A protocol-to-port mapping is not a guarantee that NBAR uses only the listed ports.** Some applications are identified using additional protocol characteristics or stateful inspection.

---

## Troubleshooting NBAR-Based Classification

If an NBAR class does not match the expected packets, investigate the following:

| Problem | What to check |
|---|---|
| The class counter remains at zero | Confirm that the policy is applied to the correct interface and direction |
| Application is not identified | Confirm the application is recognized by the installed NBAR engine/protocol pack |
| Traffic matches an unexpected class | Check class order and `match-all`/`match-any` logic |
| HTTPS is not matched by an HTTP-specific rule | HTTP headers are encrypted inside TLS |
| Only some packets match | Some applications require multiple packets before NBAR can identify the flow |
| Stateful classification fails | Confirm both directions of the flow follow a path supported by the NBAR classifier |
| Packets are tunneled or MPLS-labeled | Inspect and classify the original IP traffic at a supported point before encapsulation |
| Classification works, but QoS is ineffective | Verify the policy map's actions, DSCP markings, and downstream queuing configuration |

### Common Troubleshooting Mistakes

**1. Enabling Protocol Discovery without configuring a service policy.**

```text
R1(config-if)# ip nbar protocol-discovery
```

This enables application reporting, **not DSCP marking or queuing**.

**2. Creating a class map without applying a policy.**

```text
R1(config)# class-map match-all RTP-MEDIA
R1(config-cmap)# match protocol rtp
```

This defines a match criterion, but does not enforce any QoS treatment by itself.

**3. Confusing application identification with prioritization.**

A match statement such as `match protocol rtp` does not automatically place RTP packets into a priority queue. A policy map must explicitly specify the QoS action.

**4. Using HTTP header criteria to identify HTTPS traffic.**

For HTTPS, the HTTP host/path/MIME information is normally encrypted. Use an application signature supported by the installed NBAR engine or classify the traffic by other visible attributes.

---

## Key Points

- **NBAR** identifies supported applications and protocols; **MQC** applies QoS actions to the resulting traffic classes.
- **`match protocol <protocol>`** is the main MQC command used to classify traffic using NBAR.
- **`match-all`** requires every condition to match; **`match-any`** requires at least one condition to match.
- Class maps can combine NBAR protocol matching with other criteria, such as an ACL.
- **Class maps alone do not modify traffic.** A policy map and a service policy are required to apply QoS actions.
- Use **ingress NBAR classification and DSCP marking** near the network edge, then use **DSCP-based queuing** at downstream congestion points.
- **Protocol Discovery is optional for NBAR-based MQC classification** and does not itself enforce QoS.
- **`class-default` is not synonymous with NBAR unknown.** It includes all traffic that fails to match preceding named classes.
- Encrypted protocols, unsupported applications, tunneling, and asymmetric paths can affect application recognition.
- **`show policy-map interface`** is the primary command for verifying NBAR-based QoS class matches and actions.

---

## Further Reading

- [4.4.c Network Based Application Recognition (NBAR)](overview.md)
- [NBAR Operation and Protocol Discovery](protocol-discovery.md)
- [NBAR2 Protocol Packs and Custom Applications](nbar2.md)
- [MQC Class Maps and Match Criteria](../classification/class-maps.md)
- [DSCP Marking and Remarking with MQC](../marking/mqc-marking.md)
- [4.4.h End-to-End Layer 3 QoS Using MQC](../mqc/overview.md)

### Cisco Documentation

- [Cisco: Configuring NBAR Using the MQC](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/quality-of-service/quality-of-service/m_nbar-mqc.html)
- [Cisco: Classifying Network Traffic Using NBAR](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/quality-of-service/quality-of-service/m_clsfy-traffic-nbar-0.html)
- [Cisco: Applying QoS Features Using the MQC, IOS XE 17.x](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/qos/b-quality-of-service/m_qos-apply.html)
- [Cisco: Cisco IOS Quality of Service Solutions Command Reference (`match protocol`)](https://www.cisco.com/c/en/us/td/docs/ios/qos/command/reference/qos_book/qos_m1.html)
