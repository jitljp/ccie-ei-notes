# QoS Trust Boundaries

A **Quality of Service (QoS) trust boundary** defines where a network begins accepting QoS markings assigned by another device.

QoS markings, such as **DSCP** and **CoS**, indicate the forwarding treatment a packet should receive. However, the network must determine whether those markings are trustworthy.

For example, a PC could mark all its traffic as DSCP EF (46), attempting to receive the same low-latency forwarding treatment intended for voice traffic.

If the network accepts those markings without validation, the PC's traffic could consume resources reserved for voice and other important applications.

A QoS trust boundary prevents this by controlling which devices are allowed to assign authoritative QoS markings.

The basic principle is:

- **Trusted traffic:** Accept existing QoS markings, subject to the network's policies.
- **Untrusted traffic:** Classify and mark traffic according to the network's QoS policies.

Trust boundaries are typically established at the **network edge**, where traffic enters the network from endpoint devices or external networks.

---

## Trust Boundaries and Classification

Classification and trust are closely related, but they are different concepts.

| Concept | Purpose |
|---|---|
| **Classification** | Identifies traffic and assigns packets to QoS classes |
| **Marking** | Sets or changes QoS values such as DSCP and CoS |
| **Trust boundary** | Determines whether existing QoS markings should be accepted |

Classification occurs throughout a QoS-enabled network.

However, detailed classification and initial marking are typically performed near the network edge.

Once packets have been classified and marked by a trusted device, downstream devices can use their existing DSCP markings to select the appropriate forwarding treatment.

```text
       Untrusted Network 
       End Devices
            |
            v -- Trust Boundary
    +------------------+
    | Classification   |
    | Marking          |
    | Policing         |
    +------------------+
            |
            |
            v
    +------------------+
    | Distribution     |
    | Match DSCP       |
    +------------------+
            |
            v
    +------------------+
    | Core             |
    | Match DSCP       |
    +------------------+
            |
            v
           WAN
```

The network edge establishes the markings, and the rest of the network can use them without repeating detailed application classification.

---

## Trusted and Untrusted Interfaces

Network interfaces can be configured to accept existing QoS markings or apply policies that replace them.

### Trusted Interfaces

A **trusted interface** accepts the existing QoS markings of incoming traffic.

For example, an access switch may trust packets received from another switch that already classified and marked the traffic.

```text
      Access SW              Distribution SW
          |                        |
          +------------------------+
               Trusted Link
```

If the access switch marks voice packets EF and business traffic AF31, the distribution switch can use those markings for classification.

Trusted interfaces are commonly used for:

- Links between managed switches.
- Links between trusted routers.
- Connections to authorized QoS-marking devices.
- Links between devices within the same DiffServ domain.

### Untrusted Interfaces

An **untrusted interface** does not accept the incoming device's QoS markings as authoritative.

Instead, the receiving network device applies its own classification and marking policy.

For example:

```text
       Untrusted PC
            |
            v
       Access Switch
            |
       Classify Traffic
            |
       Set DSCP Values
            |
            v
       Trusted Network
```

The switch may:

1. Ignore the incoming DSCP or CoS value.

2. Classify traffic based on addresses, ports, or applications.

3. Assign the appropriate DSCP value.

4. Police traffic to prevent excessive use of a particular class.

For example, ordinary PC traffic might be remarked to DSCP 0, while a recognized business application is marked AF31.

**Untrusted does not necessarily mean that every packet must be marked DSCP 0.** The network can assign appropriate markings after independently classifying the traffic.

---

## Static Trust vs. Conditional Trust

Two important approaches to establishing a QoS trust boundary are **static trust** and **conditional trust**.

| Trust type | Behavior |
|---|---|
| **Static trust** | Accepts incoming QoS markings without requiring detection of a particular device |
| **Conditional trust** | Accepts incoming QoS markings only when a specified trusted device is detected |

### Static Trust

With static trust, an interface accepts the relevant QoS markings from incoming traffic.

For example, a switch uplink can be configured to trust incoming DSCP values.

This is appropriate when the connected device is another managed network device that already enforces the organization's QoS policy.

However, statically trusting an interface connected to an ordinary PC would allow the PC to select its own QoS markings.

### Conditional Trust

**Conditional trust** allows a switch to accept QoS markings only when a particular type of device is detected.

A common example is a switch interface connected to a Cisco IP phone.

The switch can use **Cisco Discovery Protocol (CDP)** to detect the phone and enable the configured trust behavior.

If the phone is not detected, the interface does not activate the conditional trust state.

This prevents an ordinary PC from receiving the same trust treatment simply by being connected directly to a switchport intended for an IP phone.

Conditional trust is particularly useful when users frequently move endpoints between switchports.

---

## The IP Phone and PC Trust Boundary

A common enterprise topology connects a PC through the integrated Ethernet switch in a Cisco IP phone.

```text
                  Cisco IP Phone
    PC            +-----------------+
     |            |                 |
     +------------| PC Port         |
                  |                 |
                  | Network Port    |
                  +--------+--------+
                           |
                           |
                      Access Switch
                           |
                      Distribution
```

The phone has two main traffic sources:

- **Phone-generated traffic:** Voice bearer and call-signaling traffic.
- **PC-generated traffic:** Traffic forwarded through the phone's integrated switch.

A typical design trusts the phone's QoS markings but does not trust arbitrary markings assigned by the PC.

### Location of the Trust Boundary

In this design, the QoS trust boundary is logically extended **to the IP phone**.

```text
   Untrusted                Trusted
      PC                    IP Phone
       |                       |
       |                       |
       +-----> PC Port         |
                 |             |
                 v             |
           Trust Boundary      |
                 |             |
                 +-------------+
                       |
                       v
                  Access Switch
                       |
                       v
                  Network Core
```

The phone can control the Layer 2 priority of traffic arriving through its PC port.

The access switch can then accept markings from the trusted phone while applying the appropriate policies to other traffic.

There are two separate trust decisions:

1. **PC to phone:** Whether the phone accepts or overwrites CoS markings from the attached PC.

2. **Phone to switch:** Whether the access switch accepts QoS markings received from the phone.

These decisions are controlled by different commands.

### Typical Voice Markings

Cisco IP phones commonly use the following markings:

| Traffic | CoS | DSCP |
|---|---:|---|
| Voice bearer (RTP) | 5 | EF (46) |
| Call signaling | 3 | CS3 (24) |
| Ordinary PC data | 0 | Usually CS0 (0) after appropriate edge marking |

The phone marks its own voice traffic.

It can also overwrite the CoS value of frames received from its PC port to prevent the PC from claiming high Layer 2 priority.

However, **overwriting CoS does not necessarily overwrite the IP packet's DSCP field**.

This distinction is important when designing a trust boundary that protects both Layer 2 and Layer 3 QoS.

---

## Configuring the IP Phone's PC-Port Trust Boundary

The `switchport priority extend` command controls how a Cisco IP phone handles CoS markings received from its PC port.

The syntax is:

```text
SW1(config-if)# switchport priority extend {cos <0-7> | trust}
```

### Overwrite PC CoS Markings

To configure the phone to overwrite PC traffic's CoS markings with 0:

```text
SW1(config)# interface GigabitEthernet1/0/1
SW1(config-if)# switchport priority extend cos 0
```

The switch communicates this setting to the phone using CDP.

The phone then overwrites the Layer 2 CoS priority of frames received from its PC port with CoS 0.

For example:

| Incoming PC CoS | CoS after phone |
|---|---|
| 0 | 0 |
| 3 | 0 |
| 5 | 0 |
| 7 | 0 |

The phone's own voice markings are unaffected.

**CoS 0 is the default PC-port trust behavior in the Cisco voice VLAN configuration.**

### Trust PC CoS Markings

Alternatively:

```text
SW1(config-if)# switchport priority extend trust
```

This instructs the phone to preserve CoS markings received from the connected PC.

For example:

| Incoming PC CoS | CoS after phone |
|---|---|
| 0 | 0 |
| 3 | 3 |
| 5 | 5 |
| 7 | 7 |

This extends Layer 2 CoS trust to the PC.

It is appropriate only when the PC is authorized to assign its own QoS markings.

### CoS Trust vs. DSCP Trust

The `switchport priority extend` command controls **Layer 2 CoS markings**, not general IP DSCP remarking.

For example, suppose a PC sends an IP packet with:

- CoS 5.
- DSCP EF (46).

If the phone is configured with:

```text
switchport priority extend cos 0
```

The phone can change the CoS value to 0 while the IP packet's DSCP remains EF.

Therefore, additional QoS policies may be needed on the switch to ensure that PC-generated DSCP markings are not trusted.

The final DSCP treatment depends on the switch's classification, trust, and marking policies.

---

## QoS Trust on Catalyst 9000 Switches

Cisco Catalyst 9000 switches use the MQC framework for QoS classification, marking, policing, and queuing.

A key characteristic is that **incoming DSCP markings are preserved by default**.

### Default Trust Behavior

On Catalyst 9000 switches, incoming packets retain their DSCP markings unless a QoS policy changes them.

For example:

```text
PC (DSCP EF)
      |
      v
 Catalyst 9300
      |
      v
Packet still marked EF
```

Without an ingress remarking policy or conditional trust restriction, the switch does not automatically change the PC's DSCP to 0.

**The absence of an explicit trust command does not mean the interface is untrusted on Catalyst 9000 switches.**

### Conditional Trust on Catalyst 9000

The `trust device` command enables conditional trust based on the connected device type.

For example:

```text
SW1(config)# interface GigabitEthernet1/0/1
SW1(config-if)# switchport mode access
SW1(config-if)# switchport access vlan 10
SW1(config-if)# switchport voice vlan 20
SW1(config-if)# trust device cisco-phone
```

This configures conditional trust for a Cisco IP phone.

The switch uses CDP to detect the phone.

| Condition | Trust behavior |
|---|---|
| Cisco IP phone detected | Phone trust condition is satisfied |
| Cisco IP phone not detected | Incoming markings are not trusted and are remarked to 0 |

The command allows the port to dynamically change its trust behavior depending on the connected device.

When the phone is detected, conditional trust applies to traffic arriving through that port, **including PC traffic forwarded through the phone**. The switch does not automatically distinguish the PC's IP packets from the phone's IP packets.

| Source of incoming traffic | DSCP behavior with `trust device cisco-phone` |
|---|---|
| Cisco phone | Preserved when the phone is detected |
| PC connected directly to switch | Remarked to 0 when no phone is detected |
| PC connected through Cisco phone | Preserved when the phone is detected, unless an ingress policy remarks it |

The phone's default PC-port setting overwrites PC **CoS** markings with 0, but does not change the PC's **DSCP**. An ingress QoS policy is needed if the switch must independently remark PC-generated DSCP values.

---

## AutoQoS

**AutoQoS** automatically generates QoS policies for common deployment scenarios, such as Cisco IP phones, trusted network links, and untrusted endpoints. Depending on the selected option, it can configure classification, marking, conditional trust, policing, and queuing.

For example, `auto qos voip cisco-phone` generates the relevant QoS configuration for a Cisco IP phone connection. The resulting policies can be inspected and adjusted as needed.

For CCIE EI, the priority is understanding and configuring these underlying mechanisms manually with MQC, rather than memorizing AutoQoS templates.

---

## CoS Trust vs. DSCP Trust

CoS and DSCP operate at different layers.

| Characteristic | CoS | DSCP |
|---|---|---|
| Layer | 2 | 3 |
| Field size | 3 bits | 6 bits |
| Possible values | 8 | 64 |
| Location | IEEE 802.1Q tag | IPv4/IPv6 header |
| Preserved across routed hops | No | Typically yes |
| Primary use | Ethernet QoS | IP QoS |

### Trusting CoS

When a switch trusts CoS, it classifies traffic based on the incoming Layer 2 priority.

The switch may then map CoS into an internal DSCP value.

This is useful on a phone-facing switchport where voice traffic is tagged with CoS 5.

### Trusting DSCP

When a switch trusts DSCP, it classifies traffic based on the IP packet's existing DSCP value.

This is useful on routed interfaces and links between managed network devices.

### Important Difference

Suppose a PC sends a packet with:

- CoS 0.
- DSCP EF (46).

If the receiving switch trusts CoS, the packet may be internally classified according to CoS 0 and mapped to the corresponding DSCP value.

If it trusts DSCP, it may accept EF directly.

Therefore, **trusting CoS and trusting DSCP can produce different forwarding treatments for the same packet**.

The exact result depends on the platform's mapping tables and configured policies.

---

## Enforcing a Trust Boundary with MQC

Trust boundaries can also be enforced using explicit MQC classification and marking policies.

This approach is useful when the switch or router needs to distinguish authorized applications rather than accepting all markings from a connected device.

Consider an interface receiving untrusted traffic.

The desired behavior is:

- Mark approved business traffic AF31.
- Remark all other traffic to DSCP 0.

### Step 1: Identify Authorized Traffic

```text
R1(config)# ip access-list extended BUSINESS-ACL
R1(config-ext-nacl)# permit tcp any host 192.0.2.10 eq 8443
R1(config-ext-nacl)# exit
R1(config)# class-map match-all BUSINESS
R1(config-cmap)# match access-group name BUSINESS-ACL
```

### Step 2: Configure the Marking Policy

```text
R1(config)# policy-map UNTRUSTED-INGRESS
R1(config-pmap)# class BUSINESS
R1(config-pmap-c)# set dscp af31
R1(config-pmap-c)# exit
R1(config-pmap)# class class-default
R1(config-pmap-c)# set dscp default
```

### Step 3: Apply the Policy

```text
R1(config)# interface GigabitEthernet1
R1(config-if)# service-policy input UNTRUSTED-INGRESS
```

This policy ignores the endpoint's chosen DSCP values for the purpose of marking.

Instead:

- Approved business traffic receives AF31.
- All other traffic receives DSCP 0.

For example:

| Original DSCP | Traffic | Resulting DSCP |
|---|---|---|
| EF | Approved business application | AF31 |
| CS0 | Approved business application | AF31 |
| EF | Other traffic | CS0 |
| AF41 | Other traffic | CS0 |

The network establishes its own QoS markings rather than accepting arbitrary values from the endpoint.

**This is classification and remarking, not packet filtering.** The policy does not block packets unless a dropping or policing action is separately configured.

---

## Trust Boundaries at WAN and Provider Connections

Trust boundaries are also important when connecting networks belonging to different organizations.

For example:

```text
     Enterprise                 Service Provider
+------------------+        +------------------+
| Customer Router  |--------| Provider Router  |
+------------------+        +------------------+
                       ^
                       |
                    Trust Boundary
```

The enterprise may mark traffic according to its own QoS policy.

However, the service provider does not necessarily accept those markings.

At the provider edge, the provider can:

- Accept trusted customer DSCP values.
- Map customer DSCP values to provider service classes.
- Remark unsupported or unauthorized DSCP values.
- Police traffic entering specific service classes.

For example, a customer may be permitted to send up to 5 Mbps of EF-marked traffic.

The provider could allow conforming EF traffic while remarking or dropping traffic that exceeds the agreed rate.

A **Traffic Conditioning Agreement (TCA)** can define the acceptable classifications, traffic profiles, and actions applied at the DiffServ domain boundary.

Therefore, **trusting a customer's DSCP markings does not imply that the provider must accept unlimited traffic in those classes**.

See [DiffServ Domains and Traffic Conditioning](../diffserv/domains.md).

---

## Trust Boundaries and Policing

Trust boundaries and policing are often used together.

Trust determines whether the network accepts the incoming traffic's classification.

Policing controls how much traffic can receive a particular forwarding treatment.

For example, a trusted IP phone normally generates a relatively predictable amount of voice traffic.

However, if the phone malfunctions or generates excessive high-priority traffic, trusting its markings alone does not prevent it from consuming too much priority-queue bandwidth.

An ingress policy can police the voice class to a configured rate.

Traffic exceeding the permitted rate can be:

- Dropped.
- Remarked to a lower-priority DSCP value.
- Forwarded with another configured treatment.

This protects the network from excessive traffic within trusted QoS classes.

See [Traffic Policing](../policing-shaping/policing.md).

---

## Trust Boundaries and Security

A QoS trust boundary is a **traffic-treatment control**, not a security authentication mechanism.

For example, Cisco's conditional phone trust commonly uses CDP to detect an expected Cisco IP phone.

This is useful for dynamically determining whether QoS markings should be accepted.

However, CDP detection should not be treated as cryptographic authentication of an endpoint.

A device that is detected as a phone is not automatically guaranteed to be authorized or uncompromised.

Other network security mechanisms, such as 802.1X and appropriate access controls, can be used separately to establish endpoint identity and restrict network access.

Similarly, trusting a device's QoS markings does not mean that all packets from that device must be given high priority.

Markings can still be validated, remarked, and policed according to network policy.

---

## Verification

On Catalyst 9000 switches, useful commands include:

```text
SW1# show running-config interface GigabitEthernet1/0/1
SW1# show cdp neighbors
SW1# show cdp neighbors detail
SW1# show policy-map interface GigabitEthernet1/0/1
SW1# show interfaces GigabitEthernet1/0/1 switchport
```

These commands can verify:

- Whether the expected phone is detected.
- Whether conditional trust is configured.
- Which ingress QoS policies are applied.
- Whether packets match the expected ingress classes.
- Whether the expected markings and policing actions are configured.
- The voice VLAN and phone-port priority settings.

### Troubleshooting Checklist

When QoS trust behavior is not working as expected, verify:

1. **Device detection:** Is the expected Cisco IP phone detected through CDP?
2. **Trust mode:** Is the interface configured to trust the intended marking?
3. **PC-port behavior:** Is the phone configured to trust or overwrite the PC's CoS markings?
4. **Ingress policy:** Is an MQC or AutoQoS policy changing the incoming DSCP value?
5. **CoS-to-DSCP mapping:** Is the expected mapping configured?
6. **Class matching:** Are packets being classified into the intended QoS classes?
7. **Egress policy:** Is the output interface providing the appropriate forwarding treatment?

Packet captures can also help verify whether DSCP values change as packets cross the trust boundary.

---

## Key Points

- A **QoS trust boundary** determines where existing QoS markings are accepted.
- Trust boundaries are typically established at the network edge.
- **Static trust** accepts incoming markings without checking for a particular connected device.
- **Conditional trust** accepts markings only when the expected trusted device is detected.
- In an IP phone/PC topology, the logical CoS trust boundary can be extended to the **IP phone**.
- The phone can overwrite PC CoS markings using `switchport priority extend cos 0`.
- `switchport priority extend trust` instructs the phone to trust PC CoS markings.
- **Overwriting CoS does not necessarily overwrite DSCP.**
- Catalyst 9000 switches preserve incoming DSCP markings by default.
- `trust device cisco-phone` configures conditional trust for Cisco IP phones on Catalyst 9000 switches.
- AutoQoS generates QoS configuration for common deployment scenarios; its underlying MQC policies are more important to understand.
- MQC can enforce trust boundaries by independently classifying and remarking untrusted traffic.
- **Trust and policing serve different purposes:** trust controls whether markings are accepted, while policing controls traffic rates.
- A QoS trust boundary does not authenticate a device or guarantee that its traffic is legitimate.

---

## Further Reading

- [Traffic Classification Methods](methods.md)
- [MQC Class Maps and Match Criteria](class-maps.md)
- [DiffServ Domains and Traffic Conditioning](../diffserv/domains.md)
- [CoS, IP Precedence, and DSCP Mapping](../marking/cos-dscp-mapping.md)
- [DSCP Marking and Remarking with MQC](../marking/mqc-marking.md)
- [Traffic Policing](../policing-shaping/policing.md)

### Cisco Documentation

- [Catalyst 9000 QoS and Queuing White Paper](https://www.cisco.com/c/en/us/products/collateral/switches/catalyst-9000/white-paper-c11-742388.html)
- [Configure AutoQoS on Catalyst 9000 Switches](https://www.cisco.com/c/en/us/support/docs/switches/catalyst-9200-series-switches/222225-configure-autoqos-on-catalyst-9000-switc.html)
- [Cisco IOS XE: Configure Voice VLAN](https://www.cisco.com/c/en/us/td/docs/switches/lan/c9000/lyr2-fwd/vlan/vlan-configuration-guide/configure-voice-vlan.html)
