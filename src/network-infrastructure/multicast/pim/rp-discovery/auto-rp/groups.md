# Auto-RP Groups and Auto-RP Listener

Auto-RP uses two well-known IPv4 multicast groups to exchange Rendezvous Point (RP) information:

| Multicast Group | Name | Purpose |
|---|---|---|
| **`224.0.1.39`** | CISCO-RP-ANNOUNCE | Candidate RPs advertise themselves to Mapping Agents |
| **`224.0.1.40`** | CISCO-RP-DISCOVERY | Mapping Agents distribute selected Group-to-RP mappings |

Both groups use **UDP destination port 496**.

Unlike PIM control messages sent to `224.0.0.13`, these Auto-RP messages can be forwarded across multiple routed hops.

However, Auto-RP introduces a problem in PIM Sparse Mode networks: **the messages used to discover the RP must be forwarded before an RP is known**.

IOS XE solves this using the `ip pim autorp listener` feature.

## Auto-RP Multicast Groups

### `224.0.1.39`: RP-Announce

Candidate RPs periodically send **RP-Announce messages** to `224.0.1.39`.

Mapping Agents join this multicast group to receive the announcements.

For example:

```text
R3 (Candidate RP)
         |
         | RP-Announce
         | 224.0.1.39
         v
    Multicast Network
         |
         v
R2 (Mapping Agent)
```

Each announcement identifies:

- The Candidate RP's IPv4 address.
- The multicast group ranges it supports.
- The advertisement Holdtime.
- Whether the advertised mapping uses PIM-SM or Bidirectional PIM.

The Mapping Agent maintains information about available Candidate RPs and selects the RP for each multicast group range.

See [Candidate RP Announcements](rp-announcements.md) for detailed configuration and message behavior.

### `224.0.1.40`: RP-Discovery

Mapping Agents periodically send **RP-Discovery messages** to `224.0.1.40`.

**All PIM-enabled routers automatically join `224.0.1.40`** to receive Group-to-RP mappings.

This happens when the first PIM-enabled interface becomes operational. No manual IGMP membership configuration is required.

For example:

```text
R2 (Mapping Agent)
         |
         | RP-Discovery
         | 224.0.1.40
         v
    Multicast Network
      /    |     \
     R1    R3     R4
```

The Mapping Agent sends its selected Group-to-RP mappings to this multicast group.

Every receiving PIM router can then learn the RP addresses required for PIM Sparse Mode operation.

The important distinction is:

| Group | Who Listens? |
|---|---|
| `224.0.1.39` | Auto-RP Mapping Agents |
| `224.0.1.40` | All PIM-enabled routers |

**The Mapping Agent listens to announcements, while all PIM routers listen to discovery messages.**

See [Mapping Agent and Discovery Messages](mapping-agent.md) for detailed RP selection and discovery behavior.

### Internetwork Control Multicast Addresses

Both Auto-RP groups belong to the `224.0.1.0/24` internetwork control block.

They are not part of the `224.0.0.0/24` local network control block.

Compare:

| Address | Purpose | Forwarded Between Routers? |
|---|---|---|
| `224.0.0.13` | PIMv2 control messages | No, ordinary packets remain link-local |
| `224.0.1.39` | Auto-RP RP-Announce | Yes |
| `224.0.1.40` | Auto-RP RP-Discovery | Yes |

Auto-RP uses the `scope` keyword to configure the **IPv4 TTL** of its packets.

For example:

```text
R3(config)# ip pim send-rp-announce Loopback0 scope 16
R2(config)# ip pim send-rp-discovery Loopback0 scope 16
```

The TTL determines how far the corresponding messages can travel across routed hops.

> **Important:** `224.0.1.39` and `224.0.1.40` are the **destination addresses of Auto-RP control messages**. They are not the multicast group ranges for which the Candidate RPs advertise themselves.

For example, a Candidate RP can send an announcement **to `224.0.1.39`** that advertises its RP address for **`239.1.0.0/16`**.

## The Auto-RP Bootstrap Problem

PIM Sparse Mode requires an RP mapping to build a shared tree for an ASM multicast group.

However, Auto-RP distributes RP information using multicast packets.

This creates a circular dependency:

```text
      Need RP information
              |
              v
    Need RP-Discovery messages
              |
              v
    Need multicast forwarding
              |
              v
      Need RP information
```

Consider the following network:

```text
  R3 (Candidate RP)
         |
         R1
         |
         R2 (Mapping Agent)
         |
         R4
         |
         R5
```

All interfaces are configured for PIM Sparse Mode.

R3 sends RP-Announce messages to `224.0.1.39`.

But R1 does not yet have an RP mapping for `224.0.1.39`, so ordinary PIM-SM forwarding cannot establish a shared tree for the group.

Consequently, the announcement might never reach R2.

The same problem applies to RP-Discovery messages destined for `224.0.1.40`.

R2 might successfully generate the discovery messages, but intermediate routers cannot forward them using ordinary Sparse Mode operation without an RP mapping.

**The routers need RP information to forward the messages that provide RP information.**

There are two solutions:

1. **Auto-RP Listener:** Enable Dense Mode forwarding specifically for the Auto-RP control groups.
2. **PIM Sparse-Dense Mode:** Use Dense Mode for multicast groups without an RP mapping.

## Auto-RP Listener

The preferred solution is the global configuration command:

```text
R1(config)# ip pim autorp listener
```

This command causes multicast traffic for the following groups to use **PIM Dense Mode forwarding**, even on interfaces configured for PIM Sparse Mode:

- `224.0.1.39`
- `224.0.1.40`

The feature is **disabled by default**.

### How Auto-RP Listener Works

When Auto-RP Listener is enabled, IOS XE treats the two Auto-RP control groups as Dense Mode groups.

Their traffic can therefore be flooded through the PIM network without requiring an RP mapping.

For example:

```text
                    R3
                Candidate RP
                     |
                     | RP-Announce
                     | 224.0.1.39
                     v
                    R1
                     |
                     | Dense Mode
                     v
                    R2
                Mapping Agent
                     |
                     | RP-Discovery
                     | 224.0.1.40
                     v
                    R4
                     |
                     | Dense Mode
                     v
                    R5
```

All interfaces remain configured for PIM Sparse Mode.

However, the Auto-RP groups are forwarded using PIM Dense Mode mechanisms, including flooding and pruning.

The process is:

1. R3 sends RP-Announce messages to `224.0.1.39`.
2. Intermediate routers forward them using Dense Mode.
3. R2 receives the advertisements and selects the RP mappings.
4. R2 sends RP-Discovery messages to `224.0.1.40`.
5. Intermediate routers forward those messages using Dense Mode.
6. PIM routers receive the mappings and can begin ordinary PIM-SM operation.

The Auto-RP control groups do not need an RP to be forwarded.

### Auto-RP Listener Does Not Change Other Groups

An important characteristic is that **Auto-RP Listener affects only the two Auto-RP control groups**.

For example:

| Multicast Group | Forwarding Mode |
|---|---|
| `224.0.1.39` | Dense Mode |
| `224.0.1.40` | Dense Mode |
| `239.1.1.1` | Sparse Mode |
| `239.2.2.2` | Sparse Mode |
| `239.3.3.3` | Sparse Mode |

Assuming the interfaces are configured for PIM Sparse Mode, ordinary ASM groups continue using shared trees toward their configured RPs.

**Auto-RP Listener does not globally enable PIM Dense Mode for all multicast traffic.**

It enables Dense Mode forwarding only where needed to bootstrap Auto-RP.

### Listening vs. Forwarding

The name `ip pim autorp listener` can be misleading.

PIM routers already **listen to `224.0.1.40` automatically** to receive RP-Discovery messages.

The purpose of `ip pim autorp listener` is to enable **Dense Mode forwarding of the Auto-RP groups across Sparse Mode interfaces**.

These are separate functions:

| Function | Requires `ip pim autorp listener`? |
|---|---|
| PIM router joins `224.0.1.40` to receive RP-Discovery | No |
| Mapping Agent joins `224.0.1.39` to receive RP-Announce | No |
| Dense Mode forwarding of Auto-RP groups across Sparse Mode interfaces | **Yes** |
| Normal PIM Sparse Mode forwarding for other groups | No |

Therefore, seeing that a router has joined `224.0.1.40` does not by itself confirm that Auto-RP Listener is enabled.

## IOS XE Configuration

Auto-RP Listener uses a global command with no arguments:

```text
ip pim autorp listener
```

### Configure Auto-RP Listener

On each router:

```text
R1(config)# ip multicast-routing
R1(config)# ip pim autorp listener

R1(config)# interface Ethernet0/0
R1(config-if)# ip pim sparse-mode

R1(config)# interface Ethernet0/1
R1(config-if)# ip pim sparse-mode
```

The `ip pim autorp listener` command is configured globally, not under individual interfaces.

Once enabled, it applies to Auto-RP control traffic traversing the router's PIM Sparse Mode interfaces.

> For simplicity, **enable `ip pim autorp listener` on all PIM routers in the Auto-RP domain**, including Candidate RPs and Mapping Agents.

This ensures that the Auto-RP control groups can be forwarded throughout the domain without requiring a separate RP mapping for those groups.

### Complete Auto-RP Example

Consider:

```text
R3 (Candidate RP)
 3.3.3.3
     |
     R1
     |
R2 (Mapping Agent)
 2.2.2.2
     |
     R4
```

Assume unicast routing and PIM connectivity are already operational.

**On all routers:**

```text
ip multicast-routing
ip pim autorp listener
```

Enable PIM Sparse Mode on the relevant routed interfaces:

```text
R1(config)# interface Ethernet0/0
R1(config-if)# ip pim sparse-mode

R1(config)# interface Ethernet0/1
R1(config-if)# ip pim sparse-mode
```

**On R3, the Candidate RP:**

```text
R3(config)# interface Loopback0
R3(config-if)# ip address 3.3.3.3 255.255.255.255
R3(config-if)# ip pim sparse-mode

R3(config)# ip access-list standard RP-GROUPS
R3(config-std-nacl)# permit 239.0.0.0 0.255.255.255

R3(config)# ip pim send-rp-announce Loopback0 scope 16 group-list RP-GROUPS
```

**On R2, the Mapping Agent:**

```text
R2(config)# interface Loopback0
R2(config-if)# ip address 2.2.2.2 255.255.255.255
R2(config-if)# ip pim sparse-mode

R2(config)# ip pim send-rp-discovery Loopback0 scope 16
```

The resulting control-plane operation is:

| Step | Source | Destination | Message |
|---|---|---|---|
| 1 | R3 | `224.0.1.39` | RP-Announce |
| 2 | R2 | `224.0.1.40` | RP-Discovery |
| 3 | R1, R4, etc. | Local RP mapping database | Install learned mappings |

Auto-RP Listener allows both multicast control groups to cross the Sparse Mode network.

After R4 receives the discovery message, it learns that `3.3.3.3` is the RP for `239.0.0.0/8`.

## Alternative: PIM Sparse-Dense Mode

The second solution is **PIM Sparse-Dense Mode**, configured per interface:

```text
R1(config)# interface Ethernet0/0
R1(config-if)# ip pim sparse-dense-mode
```

Sparse-Dense Mode determines how multicast traffic is forwarded based on the availability of an RP mapping.

| RP Mapping | Forwarding Mode |
|---|---|
| RP known | Sparse Mode |
| RP unknown | Dense Mode |

Because no RP mapping is initially available for `224.0.1.39` or `224.0.1.40`, these groups are forwarded using Dense Mode.

Therefore, Sparse-Dense Mode allows Auto-RP to operate without `ip pim autorp listener`.

### Sparse-Dense Mode Example

```text
R1(config)# ip multicast-routing

R1(config)# interface Ethernet0/0
R1(config-if)# ip pim sparse-dense-mode

R1(config)# interface Ethernet0/1
R1(config-if)# ip pim sparse-dense-mode
```

With this configuration, Auto-RP packets are flooded before RP mappings have been learned.

However, the behavior also applies to **other multicast groups without RP mappings**.

For example:

| Multicast Group | RP Known? | Forwarding Mode |
|---|---|---|
| `224.0.1.39` | No | Dense |
| `224.0.1.40` | No | Dense |
| `239.1.1.1` | Yes | Sparse |
| `239.2.2.2` | No | Dense |

This can cause unwanted multicast flooding.

### Auto-RP Listener vs. Sparse-Dense Mode

| Feature | Auto-RP Listener with Sparse Mode | Sparse-Dense Mode |
|---|---|---|
| Global command | `ip pim autorp listener` | Not required |
| Interface command | `ip pim sparse-mode` | `ip pim sparse-dense-mode` |
| Auto-RP control groups | Dense Mode | Dense Mode when no RP mapping exists |
| Other groups without an RP | Remain Sparse Mode | Dense Mode |
| Risk of unrelated Dense Mode flooding | No | Yes |
| Preferred configuration | **Yes** | No |

**PIM Sparse Mode with Auto-RP Listener is preferable** because it avoids unwanted Dense Mode flooding for unrelated multicast groups.

### Preventing Dense Mode Fallback

IOS XE also supports the global command:

```text
R1(config)# no ip pim dm-fallback
```

This prevents multicast groups in PIM Sparse-Dense Mode from falling back to Dense Mode when RP information is lost.

It is relevant when interfaces use `ip pim sparse-dense-mode`.

For example, if an existing Sparse Mode group loses its RP mapping, Dense Mode fallback could otherwise cause unwanted multicast flooding.

The command is not needed when all interfaces are configured for PIM Sparse Mode, because those interfaces do not fall back to Dense Mode.

**`no ip pim dm-fallback` is not a replacement for `ip pim autorp listener`.**

## Auto-RP Groups and Multicast Boundaries

The propagation of Auto-RP messages can also be controlled using multicast boundaries.

For example:

```text
R1(config)# access-list 10 deny host 224.0.1.39
R1(config)# access-list 10 deny host 224.0.1.40
R1(config)# access-list 10 permit any

R1(config)# interface Ethernet0/0
R1(config-if)# ip multicast boundary 10
```

This prevents Auto-RP control packets from crossing the interface in either direction.

However, this is different from the `filter-autorp` option:

```text
R1(config-if)# ip multicast boundary 10 filter-autorp
```

With `filter-autorp`, IOS XE also examines the **multicast group ranges advertised inside RP-Announce and RP-Discovery messages**, filtering group ranges denied by the boundary ACL.

The distinction is:

- **Filtering `224.0.1.39` and `224.0.1.40`:** Controls propagation of the Auto-RP control packets themselves.
- **Filtering advertised group ranges:** Controls which Group-to-RP mapping information is allowed to cross the boundary.

See [Multicast Boundaries and Administrative Scope](../../filtering/multicast-boundary.md) for detailed boundary configuration and behavior.

## IOS XE Verification

### Verify Auto-RP Listener Configuration

```text
R1# show running-config | include ip pim autorp listener
```

Expected configuration:

```text
ip pim autorp listener
```

Verify that the interfaces use Sparse Mode:

```text
R1# show ip pim interface
```

### Verify Auto-RP Status

Use:

```text
R1# show ip pim autorp
```

Example output:

```text
AutoRP Information:
  AutoRP is enabled.
  RP Discovery packet MTU is 0.
  224.0.1.40 is joined on Ethernet0/0.
  AutoRP groups over sparse mode interface is enabled

PIM AutoRP Statistics: Sent/Received
  RP Announce: 0/0, RP Discovery: 0/15
```

Important fields:

| Field | Meaning |
|---|---|
| `AutoRP is enabled` | Auto-RP functionality is enabled |
| `224.0.1.40 is joined on Ethernet0/0` | The router is listening for RP-Discovery messages |
| `AutoRP groups over sparse mode interface is enabled` | Auto-RP groups can be forwarded across Sparse Mode interfaces |
| `RP Announce: Sent/Received` | RP-Announce message counters |
| `RP Discovery: Sent/Received` | RP-Discovery message counters |

In this example, the router has received 15 RP-Discovery messages.

### Verify Multicast Routing State

Use:

```text
R1# show ip mroute 224.0.1.39
R1# show ip mroute 224.0.1.40
```

These commands display multicast routing state for the two Auto-RP control groups.

Check:

- **Incoming interface:** The RPF interface toward the sender.
- **Outgoing Interface List (OIL):** Interfaces to which the multicast packets are forwarded.
- **Flags:** Indicate forwarding state, including Dense Mode (`D`) where applicable.
- **Timers:** Indicate the age and expiration of multicast routing entries.

The control groups can appear in the multicast routing table even though no RP has been configured for them.

For example, an `(S,224.0.1.39)` entry represents multicast traffic from an Auto-RP Candidate RP to the announcement group.

An `(S,224.0.1.40)` entry represents RP-Discovery traffic from a Mapping Agent.

The source address identifies the router sending the corresponding Auto-RP packets.

### Verify Learned RP Mappings

```text
R1# show ip pim rp mapping
```

This displays the Group-to-RP mappings learned through Auto-RP.

For a particular multicast group:

```text
R1# show ip pim rp 239.1.1.1
```

A router can successfully receive RP-Discovery messages only if they reach it through the multicast network.

If the RP mapping is missing, verify both the Auto-RP control-group forwarding and the Candidate RP/Mapping Agent configuration.

### Debug Auto-RP

Use:

```text
R1# debug ip pim auto-rp
```

This displays Auto-RP activity, including announcement and discovery messages.

To stop debugging:

```text
R1# undebug all
```

### Troubleshooting Checklist

| Problem | Possible Cause |
|---|---|
| Candidate RP announcements do not reach the Mapping Agent | `224.0.1.39` not forwarded |
| Routers do not receive RP-Discovery messages | `224.0.1.40` not forwarded |
| Auto-RP fails across several routed hops | Missing Auto-RP Listener or insufficient TTL |
| Auto-RP works on directly connected routers but not distant routers | Sparse Mode forwarding problem or TTL limit |
| Unexpected multicast flooding | `ip pim sparse-dense-mode` allowing Dense Mode for groups without RPs |
| RP information differs between network regions | Auto-RP boundary, filtering, or discovery reachability |
| Auto-RP groups appear in the mroute table but RP mappings are absent | RP announcements or discovery messages not received or accepted |

## Key Points

- **`224.0.1.39`** is used for RP-Announce messages from Candidate RPs to Mapping Agents.
- **`224.0.1.40`** is used for RP-Discovery messages from Mapping Agents to PIM routers.
- Both groups use **UDP destination port 496**.
- Both addresses belong to the `224.0.1.0/24` internetwork control block and can be forwarded across routed hops.
- **Mapping Agents join `224.0.1.39`**, while **all PIM routers automatically join `224.0.1.40`**.
- Auto-RP has a bootstrap problem because multicast forwarding is needed to distribute the RP information required by Sparse Mode.
- **`ip pim autorp listener` enables Dense Mode forwarding for the two Auto-RP groups across PIM Sparse Mode interfaces.**
- Auto-RP Listener is **disabled by default** and is configured globally.
- Auto-RP Listener does **not** enable Dense Mode for ordinary multicast groups.
- For simplicity, Auto-RP Listener can be enabled on **all PIM routers in the Auto-RP domain**.
- `ip pim sparse-dense-mode` is an alternative, but it can flood other multicast groups without RP mappings.
- `no ip pim dm-fallback` prevents Dense Mode fallback when RP mappings are lost.
- Use `show ip pim autorp` to verify Auto-RP status and message counters.
- Use `show ip mroute 224.0.1.39` and `show ip mroute 224.0.1.40` to verify control-group forwarding.

## References

- [Cisco IOS XE: Configuring Basic IP Multicast](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-multicast/b-ip-multicast/m_imc_basic_cfg-0.html)
- [Cisco: PIM Auto-RP Behavior with Other RP Distribution Techniques](https://www.cisco.com/c/en/us/support/docs/ip/multicast/118405-config-rp-00.html)
- [Cisco: How to Implement a Filtering Policy for Rendezvous Points](https://www.cisco.com/c/en/us/support/docs/ip/ip-multicast/41680-rp-announce.pdf)
- [Cisco IOS XE: Multicast Configuration Guide, PIM](https://www.cisco.com/c/en/us/td/docs/switches/lan/c9000/multicast/multicast-configuration-guide/pim.html)