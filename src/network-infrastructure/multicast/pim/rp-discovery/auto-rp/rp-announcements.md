# Candidate RP Announcements

In Auto-RP, a **Candidate Rendezvous Point (C-RP)** is a router configured to advertise its willingness to serve as an RP for particular IPv4 multicast groups.

Candidate RPs periodically send **RP-Announce messages** to the well-known multicast address `224.0.1.39`, allowing Auto-RP Mapping Agents to discover them.

Unlike BSR, **Candidate RPs do not need to discover the Mapping Agent's address before sending advertisements**.

## Candidate RP Announcement Operation

Consider the following network:

```text
                 R1
                  |
              R2 (MA)
               /    \
              /      \
        R3 (C-RP)  R4 (C-RP)
         3.3.3.3    4.4.4.4
```

Assume:

- R2 is an Auto-RP Mapping Agent (MA).
- R3 and R4 are Candidate RPs for `239.0.0.0/8`.
- Auto-RP multicast control traffic can traverse the network.

### Step 1: Candidate RPs Send RP-Announce Messages

R3 and R4 send periodic multicast **RP-Announce messages** to `224.0.1.39`.

```text
R3 (C-RP) --------\
                   \
                    > 224.0.1.39
                   /       |
R4 (C-RP) --------/        v
                        R2 (MA)
```

Each announcement identifies the Candidate RP and the multicast group ranges for which it is willing to serve as RP.

For example:

```text
R3 RP-Announce:

RP Address:  3.3.3.3
Group Range: 239.0.0.0/8
Holdtime:    180 seconds
```

```text
R4 RP-Announce:

RP Address:  4.4.4.4
Group Range: 239.0.0.0/8
Holdtime:    180 seconds
```

The announcements are multicast across the PIM network, rather than unicast to a specific Mapping Agent.

### Step 2: Mapping Agent Processes the Announcements

R2 receives both RP-Announce messages and maintains information about the available Candidate RPs.

For example:

| Group Range | Candidate RP | Holdtime |
|---|---|---|
| `239.0.0.0/8` | `3.3.3.3` | 180 seconds |
| `239.0.0.0/8` | `4.4.4.4` | 180 seconds |

When multiple Candidate RPs advertise the same group range, the Mapping Agent resolves the conflict.

In Auto-RP, **the highest Candidate RP IP address is preferred**.

Therefore, R2 selects `4.4.4.4` for `239.0.0.0/8`.

This differs from BSR, where RP priority and hashing determine the selected RP on each receiving router.

### Step 3: Mapping Agent Distributes the Selected Mapping

R2 sends RP-Discovery messages to `224.0.1.40`.

For example:

```text
RP-Discovery:

Group Range: 239.0.0.0/8
RP Address:  4.4.4.4
```

Other PIM routers receive the discovery message and learn the group-to-RP mapping.

**Candidate RP announcements do not directly install RP mappings on every PIM router.** The Mapping Agent receives the announcements, selects the RP, and distributes the resulting mappings.

See [Mapping Agent and Discovery Messages](mapping-agent.md).

## RP-Announce Message Characteristics

Auto-RP uses multicast UDP packets rather than PIM control messages.

| Field | Value |
|---|---|
| Destination IPv4 address | **`224.0.1.39`** |
| UDP destination port | **496** |
| IP protocol | **17 (UDP)** |
| Default advertisement interval | **60 seconds** |
| Default Holdtime | **180 seconds** |
| TTL | Configured using `scope` |
| Address family | IPv4 only |

Each RP-Announce message conveys:

- **Candidate RP address:** The address advertised as the RP.
- **Multicast group ranges:** Groups for which the router is willing to serve as RP.
- **Holdtime:** How long the Mapping Agent considers the announcement valid.
- **PIM mode:** Whether the group mapping is for ordinary PIM-SM or Bidirectional PIM.

A single Candidate RP can advertise multiple multicast group ranges.

### Auto-RP vs. BSR Candidate RP Advertisements

| Feature | Auto-RP | BSR |
|---|---|---|
| Message | RP-Announce | Candidate-RP-Advertisement |
| Destination | `224.0.1.39` | Elected BSR address |
| Transport | Multicast UDP | Unicast PIM |
| Protocol number | 17 | 103 |
| Default advertisement interval | 60 seconds | 60 seconds |
| Default Holdtime | 180 seconds | 150 seconds |
| RP priority | Not used | Supported |
| RP selection | Mapping Agent | Each receiving router |
| IPv6 support | No | Yes |

## IPv4 IOS XE Configuration

Configure a Candidate RP using:

```text
ip pim send-rp-announce {<interface> | <ipv4-address>} scope <ttl> [group-list <acl>] [interval <seconds>] [bidir]
```

The command is entered in global configuration mode.

| Parameter | Description |
|---|---|
| `<interface>` | Interface whose IPv4 address is advertised as the RP |
| `<ipv4-address>` | Local IPv4 address to advertise as the RP |
| `scope <ttl>` | TTL of the RP-Announce messages |
| `group-list <acl>` | Standard IPv4 ACL identifying advertised multicast group ranges |
| `interval <seconds>` | Period between RP-Announce messages |
| `bidir` | Advertises the specified groups for Bidirectional PIM |

### Basic Candidate RP

On R3:

```text
R3(config)# ip multicast-routing

R3(config)# interface Loopback0
R3(config-if)# ip address 3.3.3.3 255.255.255.255
R3(config-if)# ip pim sparse-mode

R3(config)# interface Ethernet0/0
R3(config-if)# ip pim sparse-mode

R3(config)# ip pim send-rp-announce Loopback0 scope 16
```

This configures R3 to advertise itself as a Candidate RP using `3.3.3.3`.

The router sends RP-Announce messages to `224.0.1.39` through its PIM-enabled interfaces.

With no `group-list`, the router advertises itself for the entire IPv4 multicast range:

```text
224.0.0.0/4
```

This does not mean an RP is needed for every address in that range. For example, SSM groups do not use RPs.

### Candidate RP Address

The address advertised by Auto-RP is determined by the interface or IPv4 address specified in `ip pim send-rp-announce`.

For example:

```text
R3(config)# ip pim send-rp-announce Loopback0 scope 16
```

If Loopback0 has address `3.3.3.3`, R3 advertises:

```text
RP Address: 3.3.3.3
```

The command can also explicitly reference a local IPv4 address:

```text
R3(config)# ip pim send-rp-announce 3.3.3.3 scope 16
```

The specified IPv4 address must belong to the router.

**A Loopback interface is preferred** because the RP address remains available independently of individual physical interface failures.

The RP address must also be reachable by the PIM routers that learn the mapping.

> **Important:** The interface in `ip pim send-rp-announce` identifies the RP address. It does not restrict RP-Announce transmission to that interface.

## Restrict Advertised Multicast Groups

The `group-list` keyword controls which multicast groups the Candidate RP advertises itself for.

It uses a **standard IPv4 ACL**.

For example, configure R3 as a Candidate RP only for `239.1.0.0/16`:

```text
R3(config)# ip access-list standard RP-GROUPS
R3(config-std-nacl)# permit 239.1.0.0 0.0.255.255

R3(config)# ip pim send-rp-announce Loopback0 scope 16 group-list RP-GROUPS
```

The ACL matches **multicast group addresses**, not multicast source addresses or the Candidate RP's address.

The resulting advertisement identifies:

```text
RP Address:  3.3.3.3
Group Range: 239.1.0.0/16
```

### Multiple Group Ranges

A Candidate RP can advertise multiple group ranges by configuring multiple ACL entries.

For example:

```text
R3(config)# ip access-list standard RP-GROUPS
R3(config-std-nacl)# permit 239.1.0.0 0.0.255.255
R3(config-std-nacl)# permit 239.2.0.0 0.0.255.255
R3(config-std-nacl)# permit 239.3.3.0 0.0.0.255

R3(config)# ip pim send-rp-announce Loopback0 scope 16 group-list RP-GROUPS
```

R3 advertises itself for:

- `239.1.0.0/16`
- `239.2.0.0/16`
- `239.3.3.0/24`

This allows one RP to serve several noncontiguous multicast group ranges.

The Mapping Agent can then select R3 for these groups based on the advertisements it receives from all Candidate RPs.

### Candidate RP Group List vs. Mapping Agent Filtering

Two different filtering mechanisms are involved.

| Command | Configured On | Purpose |
|---|---|---|
| `ip pim send-rp-announce ... group-list` | Candidate RP | Controls which multicast group ranges the RP advertises |
| `ip pim rp-announce-filter` | Mapping Agent | Controls which incoming Candidate RP announcements are accepted |

For example, R3 might advertise itself for `239.1.0.0/16`, but the Mapping Agent can reject that advertisement using its own filtering policy.

See [RP Announcement Filtering](../../filtering/rp-announcement-filter.md) for the detailed ACL matching behavior.

## RP-Announce Scope

The `scope` keyword specifies the **IPv4 TTL** of RP-Announce messages.

For example:

```text
R3(config)# ip pim send-rp-announce Loopback0 scope 16
```

This sets the outgoing RP-Announce TTL to 16.

As the messages traverse routed hops, their TTL decreases.

The TTL must be sufficient for the advertisements to reach every intended Mapping Agent.

| Parameter | Value |
|---|---|
| Keyword | `scope` |
| Range | **1 to 255** |
| Default | No default; required |
| Function | Limits the distance RP-Announce messages can travel |

Unlike IPv6 PIM BSR's `scope` keyword, **Auto-RP `scope` is not a multicast address scope value**.

It limits packet propagation through the IPv4 network.

Also, the Candidate RP announcement TTL and Mapping Agent discovery TTL are configured independently:

```text
R3(config)# ip pim send-rp-announce Loopback0 scope 16
R2(config)# ip pim send-rp-discovery Loopback0 scope 32
```

In this example:

- RP announcements can reach Mapping Agents within the announcement TTL limit.
- RP-Discovery messages can reach routers within the discovery TTL limit.

The two TTL values do not have to be identical.

## RP-Announce Interval and Holdtime

Candidate RPs periodically refresh their advertisements.

IOS XE uses the following defaults:

| Parameter | Default |
|---|---|
| RP-Announce Interval | **60 seconds** |
| RP-Announce Holdtime | **180 seconds** |

The Holdtime is automatically calculated as:

```text
Holdtime = 3 × RP-Announce Interval
```

With the default interval:

```text
Holdtime = 3 × 60
         = 180 seconds
```

### Configure the Advertisement Interval

Use the `interval` keyword:

```text
R3(config)# ip pim send-rp-announce Loopback0 scope 16 interval 30
```

This changes the advertisement interval to 30 seconds.

The Holdtime automatically becomes:

```text
Holdtime = 3 × 30
         = 90 seconds
```

The `interval` value ranges from **1 to 16383 seconds**.

### Advertisement Expiration

The Mapping Agent maintains an expiry timer for each learned Candidate RP announcement.

Consider the default timers:

```text
RP-Announce sent
      |
      | 60 seconds
      v
RP-Announce sent
      |
      | 60 seconds
      v
RP-Announce sent
```

Each received announcement refreshes the Candidate RP information at the Mapping Agent.

If R3 stops advertising itself and no refresh is received, its information expires after the Holdtime.

With the default settings, this takes **180 seconds** from the last accepted announcement.

The Mapping Agent can then select another Candidate RP and distribute an updated RP-Discovery mapping.

**The RP-Announce Holdtime is distinct from the RP-Discovery Holdtime**, which controls how long other routers retain mappings received from the Mapping Agent.

## Candidate RP for Bidirectional PIM

Auto-RP can advertise Candidate RPs for Bidirectional PIM groups.

Use the `bidir` keyword:

```text
R3(config)# ip access-list standard BIDIR-GROUPS
R3(config-std-nacl)# permit 239.10.0.0 0.0.255.255

R3(config)# ip pim send-rp-announce Loopback0 scope 16 group-list BIDIR-GROUPS bidir
```

This advertises R3 as a Bidirectional PIM RP for `239.10.0.0/16`.

The RP-Announce message identifies the bidirectional mode so that the Mapping Agent can distribute the appropriate group-to-RP mapping.

Without `bidir`, the Candidate RP advertises ordinary PIM-SM RP mappings.

The routers within the multicast domain must use a consistent PIM mode for each group range.

See [Bidirectional PIM](../../bidirectional-pim.md) for the forwarding behavior.

## RP-Announce Forwarding

Auto-RP RP-Announce messages are multicast to `224.0.1.39`.

Because Auto-RP uses multicast traffic for RP discovery, the network must be able to forward these messages **before the relevant RP mappings have been learned**.

In a PIM Sparse Mode network, this is achieved using:

```text
R1(config)# ip pim autorp listener
```

This enables Dense Mode forwarding for the two Auto-RP control groups:

```text
224.0.1.39 - RP-Announce
224.0.1.40 - RP-Discovery
```

Ordinary PIM-SM ASM groups continue using Sparse Mode.

Without appropriate Auto-RP control-group forwarding, Candidate RPs can transmit their announcements while Mapping Agents on other routed networks fail to receive them.

**The Auto-RP Listener feature is configured on routers that must forward Auto-RP control traffic**, not exclusively on the Candidate RP or Mapping Agent.

> For simplicity, you can just enable `ip pim autorp listener` on all PIM routers in the Auto-RP domain.

See [Auto-RP Groups and Auto-RP Listener](groups.md) for the complete forwarding behavior.

## IOS XE Verification

### Verify Candidate RP Configuration

On the Candidate RP:

```text
R3# show running-config | include ip pim send-rp-announce
```

Verify the configured RP address, TTL, group list, and interval.

If a group list is configured:

```text
R3# show access-lists RP-GROUPS
```

Verify that the expected multicast group prefixes are permitted.

### Auto-RP Status and Statistics

```text
R3# show ip pim autorp
```

This displays Auto-RP status and statistics, including RP-Announce and RP-Discovery messages sent and received.

For example:

```text
AutoRP Information:
  AutoRP is enabled.

PIM AutoRP Statistics: Sent/Received
  RP Announce: 5/0, RP Discovery: 0/4
```

The counters help identify whether the Candidate RP is transmitting announcements and receiving discovery information.

### Verify Mapping Agent Reception

On the Mapping Agent:

```text
R2# show ip pim autorp
```

Check the RP-Announce received counter.

The Mapping Agent must receive the Candidate RP advertisements before it can build and distribute RP mappings.

Verify learned mappings:

```text
R2# show ip pim rp mapping
```

### Debug Auto-RP Announcements

Enable:

```text
R3# debug ip pim auto-rp
```

Example debug messages:

```text
Auto-RP: Build RP-Announce packet for 3.3.3.3
Auto-RP:  Build announce entry for (239.1.0.0/16)
Auto-RP: Send RP-Announce packet, IP source 3.3.3.3, ttl 16
```

These messages confirm:

- The Candidate RP address.
- The advertised multicast group range.
- The source address of the RP-Announce message.
- The outgoing TTL.

On the Mapping Agent, the same debug command helps identify received announcements and mapping database updates.

Use this command carefully, since debug output can be CPU-intensive.

### Verify Control-Group Forwarding

On intermediate routers:

```text
R1# show ip mroute 224.0.1.39
R1# show ip pim interface
R1# show ip pim neighbor
```

Verify that the Auto-RP announcement group is being forwarded through the expected multicast interfaces.

## Common Troubleshooting Problems

| Problem | Possible Cause |
|---|---|
| Candidate RP sends no announcements | Missing `ip pim send-rp-announce` configuration or multicast routing not operational |
| Mapping Agent receives no announcements | `224.0.1.39` not forwarded, TTL too low, or multicast boundary |
| Incorrect Candidate RP address | Wrong interface or IPv4 address specified |
| Expected group missing | Incorrect `group-list` ACL |
| Candidate RP is rejected | Mapping Agent `rp-announce-filter` policy |
| Candidate RP repeatedly expires | RP-Announce messages are not reaching the Mapping Agent consistently |
| Wrong RP selected | Another Candidate RP advertises the same group range with a higher IP address |
| Incorrect forwarding mode | `bidir` mismatch between RP advertisements and group configuration |

A useful troubleshooting sequence is:

1. Verify the Candidate RP configuration and group list.
2. Verify that RP-Announce messages are being sent to `224.0.1.39`.
3. Verify that the TTL is sufficient to reach the Mapping Agent.
4. Verify Auto-RP multicast forwarding along the path.
5. Verify the Mapping Agent receives and accepts the announcements.
6. Verify the selected group-to-RP mappings.

## Key Points

- **Candidate RPs advertise themselves using RP-Announce messages sent to `224.0.1.39`.**
- RP-Announce messages use **UDP destination port 496**.
- Candidate RPs do not need to discover the Mapping Agent before sending announcements.
- `ip pim send-rp-announce` configures an IPv4 Auto-RP Candidate RP.
- The configured interface or IPv4 address identifies the **advertised RP address**, not a single outgoing announcement interface.
- `scope` sets the IPv4 TTL, ranging from **1 to 255**.
- `group-list` uses a **standard IPv4 ACL** to restrict advertised multicast group ranges.
- Without `group-list`, the Candidate RP advertises itself for `224.0.0.0/4`.
- The default RP-Announce interval is **60 seconds**.
- The Holdtime is **three times the advertisement interval**, giving a default of **180 seconds**.
- The `bidir` keyword advertises the RP for Bidirectional PIM groups.
- **The Mapping Agent selects the RP**, preferring the highest RP address when multiple candidates advertise the same group range.
- Auto-RP RP-Announce messages require multicast forwarding through the network, unlike BSR's unicast Candidate-RP-Advertisements.
- `debug ip pim auto-rp` displays advertisement generation, reception, and mapping updates.
- Auto-RP supports **IPv4 only**.

## References

- [Cisco IOS XE: Configuring Basic IP Multicast](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/ip-multicast/ip-multicast/m_imc_basic_cfg-0.html)
- [Cisco IOS XE: IP Multicast Routing Command Reference](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9600/software/release/17-13/command_reference/b_1713_9600_cr/ip_multicast_routing_commands.html)
- [Cisco: Review the IP Multicast Quick-Start Configuration Guide](https://www.cisco.com/c/en/us/support/docs/ip/ip-multicast/9356-48.html)
- [Cisco: Debug IP PIM Auto-RP](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/debug/command/i1/db-i1-cr-book/db-i2.html)