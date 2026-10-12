# Mapping Agent and Discovery Messages

In Auto-RP, the **Mapping Agent (MA)** collects Candidate RP announcements, selects the RP for each multicast group range, and distributes the resulting **Group-to-RP mappings** throughout the network.

Candidate RPs send **RP-Announce** messages to `224.0.1.39`, which the Mapping Agent receives and processes.

The Mapping Agent then sends **RP-Discovery** messages to `224.0.1.40`, allowing other PIM routers to learn which RP to use for each multicast group.

Unlike BSR, **the Mapping Agent performs the RP selection itself**. Receiving routers use the mappings advertised by the Mapping Agent rather than independently selecting from a list of Candidate RPs.

## Mapping Agent Operation

Consider the following topology:

```text
                    R1
                     |
                 R2 (MA)
                  2.2.2.2
                  /    \
                 /      \
           R3 (C-RP)  R4 (C-RP)
            3.3.3.3    4.4.4.4
```

Assume:

- R2 is configured as an Auto-RP Mapping Agent.
- R3 and R4 are Candidate RPs for `239.0.0.0/8`.
- Auto-RP control messages can be forwarded throughout the network.

### Step 1: Receive Candidate RP Announcements

R3 and R4 periodically send RP-Announce messages to `224.0.1.39`.

```text
R3 (C-RP) --------\
                   \
                    > 224.0.1.39
                   /       |
R4 (C-RP) --------/        v
                        R2 (MA)
```

The Mapping Agent receives the announcements and maintains a database containing the Candidate RP addresses and their advertised group ranges.

For example:

| Group Range | Candidate RP | Holdtime |
|---|---|---|
| `239.0.0.0/8` | `3.3.3.3` | 180 seconds |
| `239.0.0.0/8` | `4.4.4.4` | 180 seconds |

Each announcement refreshes the corresponding Candidate RP information.

If an announcement is not refreshed before its Holdtime expires, the Mapping Agent removes the expired information.

See [Candidate RP Announcements](rp-announcements.md) for the advertisement format, timers, and Candidate RP configuration.

### Step 2: Select the RP

When multiple Candidate RPs advertise the same multicast group range, the Mapping Agent selects the **Candidate RP with the highest IPv4 address**.

In this example:

```text
Group Range: 239.0.0.0/8

Candidate RPs:
  3.3.3.3
  4.4.4.4

Selected RP: 4.4.4.4
```

Therefore, R4 is selected for `239.0.0.0/8`.

Auto-RP does **not** use Candidate RP priorities or the BSR hashing algorithm.

The Mapping Agent selects one RP for each advertised group range.

#### Overlapping Group Ranges

Candidate RPs can advertise overlapping group ranges.

For example:

| Candidate RP | Advertised Group Range |
|---|---|
| `3.3.3.3` | `239.0.0.0/8` |
| `4.4.4.4` | `239.1.0.0/16` |

Both advertisements are valid, so the Mapping Agent can distribute both mappings.

When a router looks up the RP for `239.1.1.1`, the more specific mapping takes precedence:

```text
239.1.1.1 matches:

  239.0.0.0/8   -> RP 3.3.3.3
  239.1.0.0/16  -> RP 4.4.4.4

Selected RP: 4.4.4.4
```

For `239.2.2.2`, only the `/8` mapping matches, so `3.3.3.3` is used.

This is distinct from choosing between multiple Candidate RPs advertising the **same group range**, where the highest RP address wins.

See [RP Mapping Selection and Precedence](../../group-to-rp/selection-precedence.md) for detailed group-to-RP selection rules.

### Step 3: Send RP-Discovery Messages

After selecting the RPs, R2 sends **RP-Discovery messages** to the well-known multicast address `224.0.1.40`.

```text
                  R2 (MA)
                     |
                     | RP-Discovery
                     | 224.0.1.40
                     v
               Multicast Network
                /     |     \
               R1     R3     R4
```

The messages contain the selected Group-to-RP mappings.

For example:

```text
RP-Discovery

Mapping Agent: 2.2.2.2

Group Range:   239.0.0.0/8
RP Address:    4.4.4.4
Holdtime:      180 seconds
```

The Mapping Agent can include multiple Group-to-RP mappings in a single RP-Discovery message.

### Step 4: Routers Install the Mappings

PIM routers listen for RP-Discovery messages on `224.0.1.40`.

After receiving the message, R1 learns:

```text
Group Range: 239.0.0.0/8
RP Address:  4.4.4.4
```

R1 can then use `4.4.4.4` as the RP for matching multicast groups.

For example, if R1 receives an IGMP membership report for `239.1.1.1`, it can send a PIM `(*,G)` Join toward the RP.

Similarly, a source-facing Designated Router (DR) can send PIM Register messages to the learned RP.

**The routers do not receive the complete Candidate RP database from the Mapping Agent.** They receive its selected Group-to-RP mappings.

## RP-Discovery Message Characteristics

Auto-RP uses multicast UDP messages rather than PIM control messages.

| Field | Value |
|---|---|
| Destination IPv4 address | **`224.0.1.40`** |
| UDP destination port | **496** |
| IP protocol | **17 (UDP)** |
| Default advertisement interval | **60 seconds** |
| Default Holdtime | **180 seconds** |
| TTL | Configured using `scope` |
| Address family | IPv4 only |

An RP-Discovery message identifies:

- **Mapping Agent:** The router distributing the RP information, identified by the source IPv4 address.
- **Selected RP addresses:** The RPs chosen by the Mapping Agent.
- **Multicast group ranges:** The groups associated with each selected RP.
- **Holdtime:** How long receiving routers retain the advertised information without a refresh.
- **PIM mode:** Whether the advertised mappings are for PIM Sparse Mode or Bidirectional PIM.

### RP-Announce vs. RP-Discovery

| Feature | RP-Announce | RP-Discovery |
|---|---|---|
| Sender | Candidate RP | Mapping Agent |
| Destination | `224.0.1.39` | `224.0.1.40` |
| Purpose | Advertise Candidate RP information | Distribute selected Group-to-RP mappings |
| Default interval | 60 seconds | 60 seconds |
| Default Holdtime | 180 seconds | 180 seconds |
| TTL configuration | `send-rp-announce ... scope` | `send-rp-discovery ... scope` |
| Recipient | Mapping Agents | PIM routers |

Both messages use UDP destination port 496.

## IOS XE Configuration

Configure a Mapping Agent using the following global configuration command:

```text
ip pim send-rp-discovery [<interface>] scope <ttl> [interval <seconds>]
```

The command arguments are:

| Parameter | Purpose |
|---|---|
| `<interface>` | Optional interface whose IPv4 address identifies the Mapping Agent as the RP-Discovery source |
| `scope <ttl>` | Sets the IPv4 TTL of RP-Discovery messages |
| `interval <seconds>` | Configures the RP-Discovery advertisement interval |

### Basic Mapping Agent Configuration

Configure R2 as a Mapping Agent using Loopback0 (`2.2.2.2`):

```text
R2(config)# ip multicast-routing
R2(config)# ip pim autorp listener

R2(config)# interface Loopback0
R2(config-if)# ip address 2.2.2.2 255.255.255.255
R2(config-if)# ip pim sparse-mode

R2(config)# interface Ethernet0/0
R2(config-if)# ip pim sparse-mode

R2(config)# ip pim send-rp-discovery Loopback0 scope 16
```

This configures R2 to:

1. Receive Candidate RP announcements sent to `224.0.1.39`.
2. Maintain a database of available Candidate RPs and their advertised group ranges.
3. Select an RP for each group range.
4. Send RP-Discovery messages to `224.0.1.40` with a TTL of 16.

The command can also be configured without specifying an interface:

```text
R2(config)# ip pim send-rp-discovery scope 16
```

In this case, IOS XE automatically selects the source address for the discovery messages.

Specifying a Loopback interface provides a stable, explicitly configured Mapping Agent address.

### RP-Discovery Scope

The `scope` keyword specifies the **IPv4 TTL** of RP-Discovery messages.

For example:

```text
R2(config)# ip pim send-rp-discovery Loopback0 scope 16
```

This sets the outgoing TTL to 16.

The TTL decreases as the messages traverse routed hops, limiting how far the RP information can propagate.

| Parameter | Value |
|---|---|
| Keyword | `scope` |
| Range | **1 to 255** |
| Default | No default; required |
| Purpose | Limits RP-Discovery message propagation |

The TTL must be sufficient to reach all routers that need the Group-to-RP mappings.

The Candidate RP announcement TTL and Mapping Agent discovery TTL are configured independently.

For example:

```text
R3(config)# ip pim send-rp-announce Loopback0 scope 8
R2(config)# ip pim send-rp-discovery Loopback0 scope 32
```

In this example:

- RP-Announce messages from R3 can travel up to their configured TTL limit of 8.
- RP-Discovery messages from R2 can travel up to their configured TTL limit of 32.

This allows Candidate RP advertisements to be restricted to a smaller area while distributing selected RP mappings throughout a larger PIM domain.

## RP-Discovery Interval and Holdtime

Mapping Agents periodically refresh the Group-to-RP mappings they distribute.

The default RP-Discovery interval is **60 seconds**.

| Timer | Default | Purpose |
|---|---|---|
| RP-Discovery Interval | **60 seconds** | Period between periodic discovery messages |
| RP-Discovery Holdtime | **180 seconds** | Time receiving routers retain mappings without a refresh |

### Configure the Discovery Interval

Use the `interval` keyword:

```text
R2(config)# ip pim send-rp-discovery Loopback0 scope 16 interval 30
```

This causes R2 to send periodic RP-Discovery messages every 30 seconds.

The configurable interval ranges from **1 to 16383 seconds**.

Reducing the interval increases the frequency of Group-to-RP updates but also increases control-plane traffic.

### Periodic and Triggered RP-Discovery Messages

RP-Discovery messages are sent:

1. **Periodically**, according to the configured discovery interval.
2. **When the Mapping Agent detects changes** to the Group-to-RP mapping information.

For example, suppose R4 is the selected RP for `239.0.0.0/8`:

```text
239.0.0.0/8 -> 4.4.4.4
```

If R4's announcement expires, the Mapping Agent can select R3 instead:

```text
239.0.0.0/8 -> 3.3.3.3
```

The Mapping Agent then distributes the updated mapping.

### RP-Discovery Mapping Expiration

Receiving routers maintain the learned Group-to-RP mappings as **soft state**.

Each RP-Discovery message refreshes the corresponding mapping's expiration timer.

With the default Holdtime of 180 seconds:

```text
RP-Discovery received
        |
        | 60 seconds
        v
RP-Discovery received
        |
        | 60 seconds
        v
RP-Discovery received
```

If RP-Discovery messages stop arriving, the learned mapping eventually expires.

A router that loses its Auto-RP mapping can no longer use that mapping to initiate new RP-based PIM-SM shared-tree operations.

> **Important:** Two separate expiration processes exist:
>
> - **Candidate RP expiration:** The Mapping Agent removes Candidate RP information if RP-Announce messages stop arriving.
> - **Group-to-RP mapping expiration:** Other routers remove learned mappings if RP-Discovery messages stop arriving.
>
> The two timers maintain different information and operate at different stages of Auto-RP.

## Multiple Mapping Agents

Auto-RP supports multiple Mapping Agents for redundancy.

For example:

```text
                 Multicast Network
                   /          \
                  /            \
             R1 (MA)         R2 (MA)
              1.1.1.1         2.2.2.2
                  \            /
                   \          /
                    R3 (C-RP)
                     3.3.3.3
```

Configure both Mapping Agents:

```text
R1(config)# ip pim send-rp-discovery Loopback0 scope 16

R2(config)# ip pim send-rp-discovery Loopback0 scope 16
```

Each Mapping Agent:

1. Receives RP-Announce messages.
2. Maintains its own Candidate RP information.
3. Independently selects RPs.
4. Sends RP-Discovery messages to `224.0.1.40`.

**Auto-RP does not use a BSR-style election to determine which Mapping Agent performs RP selection.**

Multiple Mapping Agents can distribute RP-Discovery information throughout the domain.

### Consistent Mapping Agent Information

All Mapping Agents should receive the same Candidate RP advertisements and use consistent filtering policies.

For example, if both Mapping Agents receive the following advertisements:

| Candidate RP | Group Range |
|---|---|
| `3.3.3.3` | `239.0.0.0/8` |
| `4.4.4.4` | `239.0.0.0/8` |

They both select:

```text
239.0.0.0/8 -> 4.4.4.4
```

However, if one Mapping Agent cannot receive R4's RP-Announce messages, it might select R3 instead.

The Mapping Agents could then distribute conflicting Group-to-RP information.

This is why consistent announcement reachability, filtering, and TTL configuration are important in deployments with multiple Mapping Agents.

### Mapping Agent vs. RP Redundancy

Mapping Agent redundancy and RP redundancy are separate mechanisms.

- **Multiple Mapping Agents** provide redundancy for the distribution of RP mappings.
- **Multiple Candidate RPs** provide alternative RPs for multicast groups.

If a Mapping Agent fails, another Mapping Agent can continue distributing RP mappings.

If the selected RP fails, the remaining Candidate RPs can be considered after the failed RP's advertisement expires.

RP failover can require rebuilding multicast distribution trees, so Auto-RP does not provide uninterrupted multicast forwarding by itself.

## Mapping Agent Announcement Filtering

By default, a Mapping Agent accepts RP-Announce messages from all Candidate RPs.

The `ip pim rp-announce-filter` command controls which announcements are accepted.

For example:

```text
R2(config)# ip pim rp-announce-filter rp-list 1 group-list 2
```

Both arguments reference standard IPv4 ACLs:

| Parameter | Purpose |
|---|---|
| `rp-list` | Selects the Candidate RPs subject to the filtering rule |
| `group-list` | Controls the advertised multicast groups accepted from those RPs |

**Candidate RPs not permitted by `rp-list` are not automatically rejected.** They are excluded from that filtering rule.

See [RP Announcement Filtering](../../filtering/rp-announcement-filter.md) for the complete ACL matching behavior and configuration examples.

## Auto-RP Discovery Forwarding

RP-Discovery messages must reach PIM routers before those routers can use the learned Group-to-RP mappings.

In a PIM Sparse Mode network, `ip pim autorp listener` enables Dense Mode forwarding for the Auto-RP control groups.

```text
R1(config)# ip pim autorp listener
```

The two Auto-RP control groups are:

| Group | Purpose |
|---|---|
| `224.0.1.39` | RP-Announce |
| `224.0.1.40` | RP-Discovery |

Other multicast groups continue using their configured PIM forwarding modes.

For simplicity, **enable `ip pim autorp listener` on all PIM routers in the Auto-RP domain**, including Candidate RPs and Mapping Agents.

See [Auto-RP Groups and Auto-RP Listener](groups.md) for detailed forwarding behavior.

Multicast boundaries can also restrict the propagation of RP-Discovery information.

See [Multicast Boundaries and Administrative Scope](../../filtering/multicast-boundary.md).

## IOS XE Verification

### Verify Mapping Agent Configuration

On the Mapping Agent:

```text
R2# show running-config | include ip pim send-rp-discovery
```

Verify the configured source interface, TTL, and discovery interval.

### Verify Auto-RP Statistics

```text
R2# show ip pim autorp
```

Example output:

```text
R2# show ip pim autorp

AutoRP Information:
  AutoRP is enabled.
  RP Discovery packet MTU is 0.
  224.0.1.40 is joined on Ethernet0/0.

PIM AutoRP Statistics: Sent/Received
  RP Announce: 0/15, RP Discovery: 8/0
```

The statistics use the format **Sent/Received**.

In this example:

- R2 has received 15 RP-Announce messages.
- R2 has sent 8 RP-Discovery messages.

On a router that only receives RP-Discovery messages, the counters might instead show:

```text
PIM AutoRP Statistics: Sent/Received
  RP Announce: 0/0, RP Discovery: 0/8
```

This confirms that the router is receiving discovery information.

### Verify Group-to-RP Mappings

```text
R2# show ip pim rp mapping
```

Example output:

```text
R2# show ip pim rp mapping

PIM Group-to-RP Mappings
This system is an RP-mapping agent (Loopback0)

Group(s) 239.0.0.0/8
  RP 4.4.4.4 (?), v2v1
    Info source: 4.4.4.4 (?), elected via Auto-RP
         Uptime: 00:05:12, expires: 00:02:45
  RP 3.3.3.3 (?), v2v1
    Info source: 3.3.3.3 (?), via Auto-RP
         Uptime: 00:04:38, expires: 00:02:51
```

This output indicates:

- R2 is configured as a Mapping Agent.
- Both R3 and R4 have advertised themselves for `239.0.0.0/8`.
- R4 is the **elected RP** because it has the higher IPv4 address.
- R3 remains in the Mapping Agent's Candidate RP information.
- The `expires` values indicate when the information will expire without a refresh.

On a router receiving RP-Discovery messages:

```text
R1# show ip pim rp mapping

PIM Group-to-RP Mappings

Group(s) 239.0.0.0/8
  RP 4.4.4.4 (?), v2v1
    Info source: 2.2.2.2 (?), elected via Auto-RP
         Uptime: 00:05:10, expires: 00:02:47
```

The receiving router learns the selected RP from R2, the Mapping Agent.

### Verify the RP Used for a Group

```text
R1# show ip pim rp 239.1.1.1
```

This checks the RP associated with a particular multicast group.

You can also verify that the RP address is reachable:

```text
R1# show ip rpf 4.4.4.4
```

### Debug Auto-RP Discovery

Use:

```text
R2# debug ip pim auto-rp
```

Example debug messages when the Mapping Agent builds a discovery packet:

```text
Auto-RP: Build RP-Discovery packet
Auto-RP:  Build mapping (239.0.0.0/8, RP:4.4.4.4)
Auto-RP: Send RP-discovery packet (1 RP entries)
```

On a receiving router:

```text
Auto-RP: Received RP-discovery, from 2.2.2.2, RP_cnt 1, holdtime 180 secs
Auto-RP:  update (239.0.0.0/8, RP:4.4.4.4)
```

These messages confirm:

- The Mapping Agent is sending RP-Discovery messages.
- The destination routers are receiving them.
- The expected Group-to-RP mappings are being advertised and installed.
- The advertised Holdtime is correct.

Disable debugging when finished:

```text
R2# undebug all
```

### Verify RP-Discovery Forwarding

On intermediate routers:

```text
R1# show ip mroute 224.0.1.40
R1# show ip pim interface
R1# show ip pim neighbor
```

Verify that `224.0.1.40` is being forwarded through the expected interfaces.

The RPF path toward the Mapping Agent's source address must also be valid for multicast forwarding.

## Common Troubleshooting Problems

| Problem | Possible Cause |
|---|---|
| Mapping Agent receives no RP-Announce messages | `224.0.1.39` not forwarded, insufficient TTL, or Candidate RP misconfiguration |
| Mapping Agent selects the wrong RP | Higher-address Candidate RP advertising the same group range, or filtering differences |
| Mapping Agent sends no RP-Discovery messages | Missing `ip pim send-rp-discovery` configuration |
| Routers receive no RP-Discovery messages | `224.0.1.40` not forwarded, insufficient TTL, or multicast boundary |
| Learned RP mappings expire | RP-Discovery messages are not reaching receivers consistently |
| Different Mapping Agents advertise different RPs | Inconsistent Candidate RP information or filtering policies |
| RP mapping is correct but multicast fails | RP unreachable, RPF failure, or PIM forwarding problem |

A useful troubleshooting sequence is:

1. Verify the Mapping Agent configuration.
2. Confirm that RP-Announce messages reach the Mapping Agent.
3. Verify the Candidate RP database and elected RP using `show ip pim rp mapping`.
4. Confirm that RP-Discovery messages are being sent to `224.0.1.40`.
5. Verify that intermediate routers forward the discovery messages.
6. Check that the receiving routers learn the expected Group-to-RP mappings.
7. Verify the unicast route and RPF path toward the selected RP.

## Key Points

- **Mapping Agents receive RP-Announce messages on `224.0.1.39`** and maintain information about available Candidate RPs.
- When multiple Candidate RPs advertise the same group range, the Mapping Agent selects the **highest RP IPv4 address**.
- Auto-RP does not use BSR RP priority or hashing.
- **RP-Discovery messages are sent to `224.0.1.40`**, using UDP destination port 496.
- Receiving routers learn the selected Group-to-RP mappings rather than the complete Candidate RP database.
- `ip pim send-rp-discovery` configures a Mapping Agent.
- The optional interface specifies the Mapping Agent source address.
- `scope` sets the IPv4 TTL, ranging from **1 to 255**.
- The default RP-Discovery interval is **60 seconds**, configurable with `interval`.
- RP-Discovery messages are sent periodically and when Group-to-RP mappings change.
- Learned Group-to-RP mappings expire if RP-Discovery messages are not refreshed.
- **Multiple Mapping Agents can operate in the same Auto-RP domain** for redundancy.
- `ip pim rp-announce-filter` controls which Candidate RP announcements the Mapping Agent accepts.
- `ip pim autorp listener` enables forwarding of the Auto-RP control groups on PIM Sparse Mode interfaces.
- Use `show ip pim autorp`, `show ip pim rp mapping`, and `debug ip pim auto-rp` to verify operation.

## References

- [Cisco IOS XE: Configuring Basic IP Multicast](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-multicast/b-ip-multicast/m_imc_basic_cfg-0.html)
- [Cisco: Review the IP Multicast Quick-Start Configuration Guide](https://www.cisco.com/c/en/us/support/docs/ip/ip-multicast/9356-48.html)
- [Cisco: How to Implement a Filtering Policy for Rendezvous Points](https://www.cisco.com/c/en/us/support/docs/ip/ip-multicast/41680-rp-announce.pdf)
- [Cisco IOS XE: Verifying IPv4 Multicast Forwarding Using the MFIB](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-multicast/b-ip-multicast/m_imc_verify_mfib-0.pdf)
- [Cisco: Debug IP PIM Auto-RP](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/debug/command/i1/db-i1-cr-book/db-i2.html)