# DSCP Marking and Remarking with MQC

**DSCP marking** is the process of setting the Differentiated Services Code Point (DSCP) value in an IPv4 or IPv6 packet to indicate its intended forwarding treatment.

**Remarking** changes an existing DSCP value to another value.

Cisco IOS XE uses the **Modular QoS Command-Line Interface (MQC)** to classify packets and apply marking or remarking actions.

Typical uses include:
- Marking voice traffic with DSCP EF (46).
- Marking business applications with DSCP AF31 (26).
- Remarking untrusted traffic to DSCP 0.
- Translating DSCP values between DiffServ domains.
- Remarking traffic that exceeds an agreed rate.
- Marking the outer IP header of a tunnel without changing the inner header.

Marking is commonly performed near the network edge, allowing downstream devices to classify traffic using the DSCP value rather than repeatedly examining packet addresses, ports, and application signatures.

---

## MQC Marking Fundamentals

MQC uses three primary configuration components.

| Component | Purpose |
|---|---|
| `class-map` | Identifies the traffic to classify |
| `policy-map` | Defines marking and other QoS actions |
| `service-policy` | Applies the policy to an interface |

For example:

```text
R1(config)# class-map match-all VOICE
R1(config-cmap)# match dscp cs0
R1(config-cmap)# exit
R1(config)# policy-map MARK-VOICE
R1(config-pmap)# class VOICE
R1(config-pmap-c)# set dscp ef
R1(config-pmap-c)# exit
R1(config)# interface GigabitEthernet1
R1(config-if)# service-policy input MARK-VOICE
```

This policy:
1. Classifies packets with DSCP CS0.
2. Changes their DSCP value to EF (46).
3. Applies the policy to packets entering GigabitEthernet1.

All other packets retain their original DSCP values.

The `match dscp cs0` command is simply an example of a classification criterion. In production, voice traffic should be identified using appropriate application, address, or protocol information rather than assuming all CS0 traffic is voice.

**The `set dscp` action changes the packet's DSCP field. It does not automatically enable priority queuing or bandwidth guarantees.**

---

## The `set dscp` Command

The primary MQC command for DSCP marking is:

```text
R1(config-pmap-c)# set dscp <value>
```

The value can be:
- A decimal number from **0 through 63**.
- A recognized DSCP name, such as `ef`, `af31`, or `cs3`.

For example:

```text
R1(config-pmap-c)# set dscp ef
```

Marks packets with DSCP 46.

The following command is equivalent:

```text
R1(config-pmap-c)# set dscp 46
```

Common marking commands include:

| Command | Resulting DSCP |
|---|---|
| `set dscp default` | CS0 (0) |
| `set dscp cs1` | CS1 (8) |
| `set dscp cs3` | CS3 (24) |
| `set dscp af21` | AF21 (18) |
| `set dscp af31` | AF31 (26) |
| `set dscp af41` | AF41 (34) |
| `set dscp ef` | EF (46) |
| `set dscp cs6` | CS6 (48) |

### `set dscp` vs. `set ip dscp`

Cisco IOS XE also supports:

```text
R1(config-pmap-c)# set ip dscp ef
```

**`set dscp` and `set ip dscp` perform the same function**, marking both IPv4 and IPv6 packets.

This differs from classification, where:

| Command | Behavior |
|---|---|
| `match dscp ef` | Matches IPv4 and IPv6 packets marked EF |
| `match ip dscp ef` | Matches only IPv4 packets marked EF |

For new configurations, `set dscp` is generally preferable because it is shorter and clearly applies to both protocols.

### Default Behavior

When no DSCP marking action is configured, packets normally **retain their existing DSCP values**.

For example:

```text
R1(config)# policy-map INGRESS
R1(config-pmap)# class VOICE
R1(config-pmap-c)# set dscp ef
```

| Traffic | Original DSCP | Result |
|---|---|---|
| Matches VOICE | CS0 | EF |
| Matches VOICE | AF31 | EF |
| Does not match VOICE | AF31 | AF31 |
| Does not match VOICE | CS6 | CS6 |

The original marking does not affect the value assigned by `set dscp`.

The `no set dscp` command removes a marking action from a policy class. It does not configure the class to remark packets to zero.

---

## Marking Traffic Based on ACLs

A common approach is to classify traffic using an access control list and then assign a DSCP value.

### Example: Marking Business Traffic AF31

Suppose R1 receives traffic from subnet `10.10.10.0/24` destined for an application server at `192.0.2.10` using TCP port 443.

The objective is to mark that traffic AF31 (26).

**Step 1: Create an ACL.**

```text
R1(config)# ip access-list extended BUSINESS-ACL
R1(config-ext-nacl)# permit tcp 10.10.10.0 0.0.0.255 host 192.0.2.10 eq 443
```

**Step 2: Create a class map.**

```text
R1(config)# class-map match-all BUSINESS
R1(config-cmap)# match access-group name BUSINESS-ACL
```

**Step 3: Create a policy map.**

```text
R1(config)# policy-map INGRESS-MARKING
R1(config-pmap)# class BUSINESS
R1(config-pmap-c)# set dscp af31
```

**Step 4: Apply the policy inbound.**

```text
R1(config)# interface GigabitEthernet1
R1(config-if)# service-policy input INGRESS-MARKING
```

Matching packets are marked AF31 as they enter R1.

Nonmatching packets retain their existing DSCP values.

Remember that, in an ACL used for QoS classification:
- `permit` means the packet matches the ACL.
- `deny` means the packet does not match the ACL for classification.

An ACL `deny` statement does not independently cause the QoS policy to drop the packet.

---

## Marking Multiple Traffic Classes

A single policy map can classify and mark several traffic classes.

For example, suppose an enterprise uses these markings:

| Traffic | DSCP |
|---|---|
| Voice | EF (46) |
| Business applications | AF31 (26) |
| Background traffic | CS1 (8) |
| All remaining traffic | CS0 (0) |

The traffic can be classified using separate ACLs and class maps.

Assume the ACLs `VOICE-ACL`, `BUSINESS-ACL`, and `BACKGROUND-ACL` have already been configured to identify the appropriate traffic.

```text
R1(config)# class-map match-all VOICE
R1(config-cmap)# match access-group name VOICE-ACL
R1(config-cmap)# exit
R1(config)# class-map match-all BUSINESS
R1(config-cmap)# match access-group name BUSINESS-ACL
R1(config-cmap)# exit
R1(config)# class-map match-all BACKGROUND
R1(config-cmap)# match access-group name BACKGROUND-ACL
R1(config-cmap)# exit
R1(config)# policy-map INGRESS-MARKING
R1(config-pmap)# class VOICE
R1(config-pmap-c)# set dscp ef
R1(config-pmap-c)# exit
R1(config-pmap)# class BUSINESS
R1(config-pmap-c)# set dscp af31
R1(config-pmap-c)# exit
R1(config-pmap)# class BACKGROUND
R1(config-pmap-c)# set dscp cs1
R1(config-pmap-c)# exit
R1(config-pmap)# class class-default
R1(config-pmap-c)# set dscp default
R1(config-pmap-c)# exit
R1(config)# interface GigabitEthernet1
R1(config-if)# service-policy input INGRESS-MARKING
```

### Class Evaluation Order

Named classes in a policy map are evaluated in order.

A packet is assigned to the **first matching class**.

Once a packet matches a class, the marking action associated with that class is applied. The packet is not subsequently evaluated against later named classes in the same policy.

For example, if a packet matches both `VOICE` and `BUSINESS`, and `VOICE` appears first, it is marked EF.

The predefined `class-default` handles traffic that does not match any earlier named class.

**`class-default` does not automatically reset DSCP to zero.** That behavior is explicitly configured with `set dscp default`.

---

## Remarking Untrusted Traffic

An important use of DSCP remarking is enforcing a **QoS trust boundary**.

Consider a PC that marks its own traffic EF (46), attempting to receive the same forwarding treatment as voice traffic.

If the network accepts that marking, the PC's traffic might consume resources reserved for legitimate voice traffic.

The network can instead identify authorized voice traffic and reset other traffic to Best Effort.

### Example: Trust Only Authorized Voice Traffic

Suppose the trusted voice subnet is `10.10.10.0/24`.

For this example, assume traffic sourced from this subnet is authorized to receive EF treatment.

```text
R1(config)# ip access-list standard AUTHORIZED-VOICE
R1(config-std-nacl)# permit 10.10.10.0 0.0.0.255
R1(config-std-nacl)# exit
R1(config)# class-map match-all VOICE
R1(config-cmap)# match access-group name AUTHORIZED-VOICE
R1(config-cmap)# exit
R1(config)# policy-map UNTRUSTED-INGRESS
R1(config-pmap)# class VOICE
R1(config-pmap-c)# set dscp ef
R1(config-pmap-c)# exit
R1(config-pmap)# class class-default
R1(config-pmap-c)# set dscp default
R1(config-pmap-c)# exit
R1(config)# interface GigabitEthernet1
R1(config-if)# service-policy input UNTRUSTED-INGRESS
```

The result is:

| Packet source | Original DSCP | Result |
|---|---|---|
| Authorized voice subnet | CS0 | EF |
| Authorized voice subnet | EF | EF |
| Other subnet | EF | CS0 |
| Other subnet | AF31 | CS0 |
| Other subnet | CS0 | CS0 |

This prevents untrusted markings from being carried into the trusted QoS domain.

In a real network, the classification should be specific enough to identify authorized voice traffic, not simply any packet claiming to originate from a particular subnet.

### DSCP Bleaching

**DSCP bleaching** means resetting the DSCP value of traffic to zero.

For example:

```text
R1(config)# policy-map RESET-DSCP
R1(config-pmap)# class class-default
R1(config-pmap-c)# set dscp default
```

This policy remarks all matching IP packets to DSCP CS0.

Bleaching commonly occurs at network boundaries where existing markings should not be trusted.

It does not mean dropping the packets.

---

## Remarking Based on Existing DSCP Values

MQC can classify packets by their existing DSCP value and replace that value with another.

This is useful when two DiffServ domains use different marking conventions.

### Example: Remark AF31 to AF21

Suppose an enterprise uses AF31 for business traffic, but its service provider expects AF21.

The enterprise router can remark packets before sending them to the provider.

```text
R1(config)# class-map match-all BUSINESS
R1(config-cmap)# match dscp af31
R1(config-cmap)# exit
R1(config)# policy-map PROVIDER-REMARK
R1(config-pmap)# class BUSINESS
R1(config-pmap-c)# set dscp af21
R1(config-pmap-c)# exit
R1(config)# interface GigabitEthernet2
R1(config-if)# service-policy output PROVIDER-REMARK
```

The result is:

| Original DSCP | Matches BUSINESS? | Outgoing DSCP |
|---|---|---|
| AF31 | Yes | AF21 |
| AF21 | No | AF21 |
| EF | No | EF |
| CS0 | No | CS0 |

This policy modifies only packets originally marked AF31.

Other packets retain their original DSCP values.

---

## Ingress vs. Egress Marking

DSCP marking can be performed in either direction.

| Direction | Common purpose |
|---|---|
| **Ingress** | Classify and mark packets as they enter the router |
| **Egress** | Remark packets before forwarding them toward the next network |

Ingress marking is commonly preferred when the objective is to establish a consistent QoS classification for subsequent forwarding decisions.

### Ingress Marking

```text
R1(config)# interface GigabitEthernet1
R1(config-if)# service-policy input INGRESS-MARKING
```

Packets are classified and marked as they enter the router.

The resulting DSCP values can then be used for subsequent QoS classification and forwarding decisions.

### Egress Marking

```text
R1(config)# interface GigabitEthernet2
R1(config-if)# service-policy output PROVIDER-REMARK
```

Packets are remarked as they leave the router.

An important distinction is that **egress remarking does not change the class or output queue already selected by the egress policy**.

For example:

```text
R1(config)# class-map match-all BUSINESS
R1(config-cmap)# match dscp af31
R1(config-cmap)# exit
R1(config)# policy-map WAN-QOS
R1(config-pmap)# class BUSINESS
R1(config-pmap-c)# bandwidth percent 30
R1(config-pmap-c)# set dscp af21
```

Packets originally marked AF31:
1. Match the `BUSINESS` class.
2. Receive the configured class-based queuing treatment.
3. Are remarked to AF21 for subsequent forwarding.

They do not move to another output queue because their DSCP value changed within the policy.

Egress remarking can, however, influence DSCP-based WRED drop treatment where supported and configured.

---

## IPv4 and IPv6 Marking

The `set dscp` command operates on both IPv4 and IPv6 packets.

For example:

```text
R1(config)# class-map match-all REALTIME
R1(config-cmap)# match dscp ef
R1(config-cmap)# exit
R1(config)# policy-map REMARK
R1(config-pmap)# class REALTIME
R1(config-pmap-c)# set dscp af41
```

This class matches both IPv4 and IPv6 packets marked EF.

The policy remarks matching packets to AF41.

### IPv6-Specific Marking

If a policy should match only IPv6 traffic, the class map can explicitly identify IPv6 packets.

For example:

```text
R1(config)# class-map match-all IPV6-TRAFFIC
R1(config-cmap)# match protocol ipv6
R1(config-cmap)# exit
R1(config)# policy-map MARK-IPV6
R1(config-pmap)# class IPV6-TRAFFIC
R1(config-pmap-c)# set dscp af21
```

This policy marks matching IPv6 traffic AF21 without selecting IPv4 traffic.

The `set dscp` action modifies the DSCP portion of the IPv6 Traffic Class field.

---

## Conditional Remarking with Policing

DSCP remarking can also be based on whether traffic conforms to a configured rate.

This is useful for **Assured Forwarding (AF)**, where packets within the same forwarding class can have different drop precedence levels.

For example:
- AF31: Low drop precedence.
- AF32: Medium drop precedence.
- AF33: High drop precedence.

Rather than dropping traffic immediately when it exceeds a configured rate, a policer can remark excess packets to a higher drop precedence.

### Example: Remark Excess AF31 Traffic to AF32

Suppose traffic marked AF31 should be allowed to transmit at a committed rate of 1 Mbps.

Packets exceeding this rate should be remarked AF32 rather than dropped.

```text
R1(config)# class-map match-all BUSINESS
R1(config-cmap)# match dscp af31
R1(config-cmap)# exit
R1(config)# policy-map POLICE-BUSINESS
R1(config-pmap)# class BUSINESS
R1(config-pmap-c)# police cir 1000000 conform-action transmit exceed-action set-dscp-transmit af32
```

The actions are:

| Traffic | Action | DSCP |
|---|---|---|
| Conforming | Transmit without remarking | AF31 |
| Exceeding | Remark and transmit | AF32 |

The `set-dscp-transmit` action performs two operations:
1. Changes the packet's DSCP value.
2. Transmits the packet rather than dropping it.

This is called **policer-based remarking** or **markdown**.

The higher drop precedence allows downstream devices to treat excess traffic as more likely to be dropped during congestion.

For example, a WRED policy can use different drop thresholds for AF31 and AF32 packets.

On platforms that use table maps for policer markdown, the equivalent operation uses a DSCP mapping table instead of a direct DSCP value.

Policing and rate-dependent actions are covered in detail in [4.4.e Policing and Shaping](../policing-shaping/overview.md).

---

## QoS Table Maps

Cisco IOS XE supports **table maps**, which translate existing QoS values into new values using a predefined mapping.

Rather than creating separate class maps and marking actions for every possible value, a table map allows multiple translations to be defined and applied through a single MQC action.

Table maps can be used to:
- **Remark DSCP values**, such as AF31 to AF21.
- **Translate between QoS fields**, such as CoS to DSCP.
- **Assign internal QoS values**, such as mapping DSCP to a QoS group or traffic class.
- **Perform policer-based markdown**, assigning different DSCP values to traffic that exceeds a configured rate.

A table map defines numerical mappings independently of any particular QoS field. The MQC `set` command determines which field supplies the input value and which field receives the result.

---

### Creating a Table Map

Suppose a service provider expects different DSCP markings from those used by an enterprise network:
- EF should remain EF.
- AF31 should be changed to AF21.
- AF41 should be changed to AF31.
- All other DSCP values should remain unchanged.

Configure:

```text
R1(config)# table-map DSCP-TRANSLATION
R1(config-tablemap)# map from 26 to 18
R1(config-tablemap)# map from 34 to 26
R1(config-tablemap)# default copy
```

The `map from <value> to <value>` command defines an individual mapping.

The table produces:

| Original DSCP | New DSCP |
|---|---|
| AF31 (26) | AF21 (18) |
| AF41 (34) | AF31 (26) |
| EF (46) | EF (46) |
| Other values | Unchanged |

The `default copy` command preserves values that do not have an explicit mapping.

**A table map does not modify traffic until it is referenced by a QoS policy.**

### Default Actions

A table map supports three default actions that determine how values without an explicit `map from` entry are handled.

| Command | Behavior |
|---|---|
| `default copy` | Copies the original input value to the output field |
| `default ignore` | Does not modify the output field for an unmapped input |
| `default <value>` | Assigns the specified value to all unmapped inputs |

For example:

```text
R1(config-tablemap)# default 0
```

When used for DSCP-to-DSCP translation, this sets all unmapped DSCP values to 0.

With:

```text
R1(config-tablemap)# default ignore
```

The table map performs no marking action for unmapped values. **It does not drop packets.**

For DSCP-to-DSCP translation, `default copy` and `default ignore` generally produce the same packet markings. The distinction becomes more important when translating between different QoS fields.

When neither a particular mapping nor an explicit default action is configured, Cisco IOS XE generally uses `default copy`.

---

### Applying a Table Map

After creating the table map, reference it in an MQC policy.

```text
R1(config)# policy-map TRANSLATE-DSCP
R1(config-pmap)# class class-default
R1(config-pmap-c)# set dscp dscp table DSCP-TRANSLATION
R1(config-pmap-c)# exit
R1(config)# interface GigabitEthernet2
R1(config-if)# service-policy output TRANSLATE-DSCP
```

The command:

`set dscp dscp table DSCP-TRANSLATION`

uses:
- **First `dscp`:** The destination field to modify.
- **Second `dscp`:** The source field whose existing value is examined.
- **`table`:** Specifies that a table map determines the new value.

The general syntax is:

`set <destination-field> <source-field> table <name>`

For example, `set dscp qos-group table MAP` uses the existing QoS group to determine the new DSCP value.

This distinction is important when translating between different QoS fields.

---

### Mapping Between Different QoS Fields

Table maps are not limited to DSCP-to-DSCP translation.

They can translate between Layer 2 markings, Layer 3 markings, and internal QoS identifiers.

Common examples include:

| Mapping | MQC command |
|---|---|
| DSCP → DSCP | `set dscp dscp table MAP` |
| CoS → DSCP | `set dscp cos table MAP` |
| DSCP → CoS | `set cos dscp table MAP` |
| IP Precedence → DSCP | `set dscp precedence table MAP` |
| DSCP → IP Precedence | `set precedence dscp table MAP` |
| DSCP → QoS group | `set qos-group dscp table MAP` |
| QoS group → DSCP | `set dscp qos-group table MAP` |
| DSCP → Traffic class | `set traffic-class dscp table MAP` |

The supported mappings and attachment directions depend on the platform.

### Example: CoS to DSCP

Suppose a switch receives Ethernet frames with the following CoS values:
- CoS 5: Voice traffic.
- CoS 3: Signaling traffic.
- Other values: Best Effort.

The objective is to assign corresponding DSCP markings.

```text
SW1(config)# table-map COS-TO-DSCP
SW1(config-tablemap)# map from 5 to 46
SW1(config-tablemap)# map from 3 to 24
SW1(config-tablemap)# default 0
```

This produces:

| Incoming CoS | Resulting DSCP |
|---|---|
| 5 | EF (46) |
| 3 | CS3 (24) |
| Other values | CS0 (0) |

Apply the table map:

```text
SW1(config)# policy-map MARK-DSCP
SW1(config-pmap)# class class-default
SW1(config-pmap-c)# set dscp cos table COS-TO-DSCP
SW1(config-pmap-c)# exit
SW1(config)# interface GigabitEthernet1/0/1
SW1(config-if)# service-policy input MARK-DSCP
```

The switch reads the existing **Layer 2 CoS value** and uses the table to assign the corresponding **Layer 3 DSCP value**.

### Example: DSCP to QoS Group

Table maps can also assign internal QoS identifiers without modifying packet headers.

Suppose a router should assign:
- QoS group 5 to EF traffic.
- QoS group 3 to AF31 traffic.
- QoS group 0 to all other traffic.

Configure:

```text
R1(config)# table-map DSCP-TO-QOS
R1(config-tablemap)# map from 46 to 5
R1(config-tablemap)# map from 26 to 3
R1(config-tablemap)# default 0
!
R1(config)# policy-map INTERNAL-QOS
R1(config-pmap)# class class-default
R1(config-pmap-c)# set qos-group dscp table DSCP-TO-QOS
```

This produces:

| Incoming DSCP | Assigned QoS group |
|---|---|
| EF (46) | 5 |
| AF31 (26) | 3 |
| Other values | 0 |

**The DSCP field is not changed.** The router assigns an internal QoS group that can be used for subsequent classification and forwarding decisions.

The QoS group is local to the device and is not carried in the IP header.

---

### Table Maps for Policer-Based Markdown

Table maps can also be used with **traffic policing** to remark packets according to whether they conform to a configured rate.

For example, an enterprise might use:
- AF31 for conforming traffic.
- AF32 for traffic exceeding its committed rate.
- AF33 for traffic exceeding a higher rate.

A table map can define the markdown applied to excess traffic.

```text
R1(config)# table-map EXCEED-MARKDOWN
R1(config-tablemap)# map from 26 to 28
R1(config-tablemap)# map from 28 to 30
R1(config-tablemap)# default copy
```

The table defines:

| Original DSCP | New DSCP |
|---|---|
| AF31 (26) | AF32 (28) |
| AF32 (28) | AF33 (30) |
| Other values | Unchanged |

A policer can reference this table:

```text
R1(config)# policy-map POLICE-TRAFFIC
R1(config-pmap)# class class-default
R1(config-pmap-c)# police cir 1000000
R1(config-pmap-c-police)# conform-action transmit
R1(config-pmap-c-police)# exceed-action set-dscp-transmit dscp table EXCEED-MARKDOWN
```

Packets conforming to the configured rate retain their existing markings.

Packets exceeding the rate are remarked according to the table and transmitted rather than immediately dropped.

The receiving network can then apply a higher drop probability to AF32 or AF33 traffic during congestion.

See [Traffic Policing](../policing-shaping/policing.md) for more information about policing actions.

---

### Verification

The primary command for displaying configured table maps is:

```text
R1# show table-map
```

This displays the configured mappings and default actions.

To verify a particular map:

```text
R1# show table-map DSCP-TRANSLATION
```

To verify that a policy references the table map:

```text
R1# show policy-map TRANSLATE-DSCP
```

To inspect an applied policy and its matching statistics:

```text
R1# show policy-map interface GigabitEthernet2
```

Use a packet capture to confirm that packets leave the device with the expected DSCP values.

For example, traffic originally marked AF31 should leave with AF21 when the `DSCP-TRANSLATION` table is applied.

**On Catalyst 9000 platforms, table-map marking is generally configured under `class-default`, rather than arbitrary named policy classes.** The exact restrictions depend on the platform and QoS implementation.

---

### Key Points

- Table maps translate one QoS value into another using predefined numerical mappings.
- They are not limited to DSCP remarking.
- `map from <value> to <value>` defines an explicit mapping.
- `default copy` copies unmapped values.
- `default ignore` leaves the destination field unchanged for unmapped inputs.
- `default <value>` assigns a specified value to unmapped inputs.
- In the MQC syntax, the **destination field comes first**, followed by the source field.
- Table maps can translate DSCP, CoS, IP Precedence, and internal QoS values.
- Policer-based markdown can use table maps to assign higher drop precedence to excess traffic.
- `show table-map` displays configured mappings.

---

## DSCP Marking with GRE Tunnels

With GRE tunneling, there are two IP headers:
- **Inner IP header:** Belongs to the original packet.
- **Outer IP header:** Added for GRE encapsulation.

Each header has its own DSCP field.

For example:

```text
             GRE-Encapsulated Packet

+----------------------------------------+
| Outer IPv4 Header                      |
| DSCP: EF (46)                          |
+----------------------------------------+
| GRE Header                             |
+----------------------------------------+
| Inner IPv4 Header                      |
| DSCP: AF41 (34)                        |
+----------------------------------------+
| Original Payload                       |
+----------------------------------------+
```

The inner and outer markings do not have to be identical.

### Marking the Inner Header

Consider the following policy:

```text
R1(config)# class-map match-all VIDEO
R1(config-cmap)# match dscp af41
R1(config-cmap)# exit
R1(config)# policy-map GRE-MARKING
R1(config-pmap)# class VIDEO
R1(config-pmap-c)# set dscp ef
R1(config-pmap-c)# exit
R1(config)# interface Tunnel100
R1(config-if)# service-policy output GRE-MARKING
```

The `set dscp ef` action changes the DSCP value in the original packet's IP header before GRE encapsulation.

With the normal GRE behavior of copying the inner DSCP into the outer header, both headers receive EF.

This may be undesirable if the objective is to preserve AF41 in the original packet while using EF across the tunnel.

### Marking Only the Outer Header

Cisco IOS XE provides the `set dscp tunnel` command for marking a tunnel header that has not yet been added.

For example:

```text
R1(config)# policy-map GRE-MARKING
R1(config-pmap)# class VIDEO
R1(config-pmap-c)# set dscp tunnel ef
```

When this policy is applied outbound to the tunnel interface:
- The original inner IP header retains AF41.
- The new outer IP header is marked EF.

After decapsulation, the original packet still has DSCP AF41.

| Command | Header modified |
|---|---|
| `set dscp ef` | Original IP header |
| `set dscp tunnel ef` | Newly imposed outer IP header |

The `set dscp tunnel` command is useful when a transport network requires different markings from the original network.

Support and exact encapsulation behavior depend on the tunnel and forwarding implementation.

---

## Internal QoS Marking with `qos-group`

Not all marking actions modify packet headers.

Cisco IOS XE can also assign an internal **QoS group** value to a packet.

For example:

```text
R1(config)# policy-map INGRESS
R1(config-pmap)# class BUSINESS
R1(config-pmap-c)# set qos-group 3
```

The `set qos-group` command assigns an internal classification identifier while the packet is processed by the router.

Unlike `set dscp`, **`set qos-group` does not modify the IP header**.

The internal value can then be used for classification in another policy:

```text
R1(config)# class-map match-all BUSINESS-GROUP
R1(config-cmap)# match qos-group 3
```

This is useful when traffic must be classified before an operation that makes the original classification fields unavailable, such as encryption.

| Marking | Written into IP header? | Carried to next router? |
|---|---|---|
| DSCP | Yes | Yes |
| QoS group | No | No |

QoS groups are local forwarding metadata, not end-to-end markings.

---

## Verification

The primary command for verifying an applied marking policy is:

```text
R1# show policy-map interface GigabitEthernet1
```

The output identifies:
- The applied service policy and direction.
- The configured traffic classes.
- Classification criteria.
- Packet and byte counters.
- Marking actions.
- Marking statistics where supported and enabled.

Other useful commands include:

| Command | Purpose |
|---|---|
| `show policy-map` | Display configured policy maps |
| `show policy-map INGRESS-MARKING` | Display a specific policy |
| `show policy-map interface GigabitEthernet1` | Verify policy attachment, matches, and actions |
| `show class-map` | Display configured class maps |
| `show running-config interface GigabitEthernet1` | Verify interface service-policy attachment |
| `show running-config | section policy-map` | Review policy-map configuration |

### Interpreting Policy Statistics

For example, if the `BUSINESS` class shows increasing packet counters, matching traffic is being classified into that class.

If the policy contains:

```text
R1(config-pmap-c)# set dscp af31
```

the matching traffic is configured to be marked AF31.

However, **matching counters alone do not prove that packets are leaving the device with the expected DSCP value**.

For end-to-end verification, use a packet capture at the appropriate location.

### Verifying with Wireshark

Wireshark can display the DSCP value in IPv4 and IPv6 headers.

Useful display filters include:

```text
ip.dsfield.dscp == 46
```

Matches IPv4 packets marked EF.

```text
ipv6.tclass.dscp == 26
```

Matches IPv6 packets marked AF31.

To verify remarking, capture packets before and after the marking device and compare their DSCP values.

For example:

```text
Before R1:  DSCP AF31 (26)
After R1:   DSCP AF21 (18)
```

This confirms that remarking occurred.

---

## Key Points

- `set dscp <value>` marks packets with the specified DSCP value.
- `set dscp` and `set ip dscp` both support IPv4 and IPv6.
- A packet retains its existing DSCP value unless a marking action changes it.
- `class-default` does not automatically remark traffic to CS0.
- `set dscp default` explicitly sets DSCP to 0.
- Named classes in an MQC policy are evaluated in order, and the first matching class is selected.
- Ingress marking establishes DSCP values that can be used for subsequent QoS processing.
- Egress remarking does not change the class or queue already selected by the egress policy.
- `set-dscp-transmit` can remark and forward traffic based on a policing result.
- Table maps can translate DSCP values using predefined mappings.
- `set dscp tunnel` can mark a newly imposed tunnel header without changing the original IP header.
- `set qos-group` assigns an internal classification value without changing the packet's IP header.
- `show policy-map interface` is the primary IOS XE command for verifying MQC marking policies.

---

## References

- [Cisco IOS XE 17.x: QoS Packet Marking](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/qos/b-quality-of-service/m_qos-mrkg.html)
- [Cisco IOS XE 17.x: Applying QoS Features Using the MQC](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/qos/b-quality-of-service/m_qos-apply.html)
- [Cisco IOS XE 17.x: QoS Packet Policing](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/qos/b-quality-of-service/m_qos-policing.html)
- [Cisco IOS XE 17: Quality of Service Configuration Guide](https://www.cisco.com/c/en/us/td/docs/switches/lan/c9000/qos/quality-of-service-configuration-guide/m-quality-of-service.html)
- [RFC 2474: Definition of the Differentiated Services Field (DS Field) in the IPv4 and IPv6 Headers](https://www.rfc-editor.org/rfc/rfc2474)
- [RFC 2597: Assured Forwarding PHB Group](https://www.rfc-editor.org/rfc/rfc2597)