# Auto-RP

**Auto-RP** is a Cisco-developed mechanism for dynamically distributing Rendezvous Point (RP) information throughout an IPv4 PIM multicast network.

Like the Bootstrap Router (BSR) mechanism, Auto-RP eliminates the need to manually configure group-to-RP mappings on every router.

However, Auto-RP differs from BSR in two important ways:

- **Auto-RP uses multicast UDP messages** to advertise and distribute RP information, rather than PIM Bootstrap Messages.
- **An Auto-RP Mapping Agent selects the RP** for each multicast group range. With BSR, each PIM router independently selects its RP from an advertised RP-Set.

Auto-RP supports **IPv4 only**. IPv6 PIM uses mechanisms such as BSR or static RP mappings instead.

## Auto-RP Components

Auto-RP involves two main roles:

| Component | Function |
|---|---|
| **Candidate RP (C-RP)** | Advertises its willingness to act as the RP for specified multicast groups |
| **Mapping Agent (MA)** | Collects C-RP announcements, selects the RPs, and distributes the resulting group-to-RP mappings |

The Mapping Agent effectively replaces the BSR's RP information distribution function, but it also performs the RP selection itself.

A router can act as both a Candidate RP and Mapping Agent.

## Auto-RP Message Types

Auto-RP uses two well-known IPv4 multicast groups:

| Message | Destination | Purpose |
|---|---|---|
| **RP-Announce** | `224.0.1.39` | Candidate RPs advertise themselves to Mapping Agents |
| **RP-Discovery** | `224.0.1.40` | Mapping Agents distribute selected group-to-RP mappings |

Both message types use **UDP destination port 496**.

This is an important difference from BSR:

| Auto-RP | BSR |
|---|---|
| Uses UDP, port 496 | Uses PIM, IP protocol 103 |
| RP-Announce: `224.0.1.39` | Candidate-RP-Advertisement: unicast to BSR |
| RP-Discovery: `224.0.1.40` | Bootstrap Message: `224.0.0.13` |
| Multicast messages forwarded using multicast routing | BSMs forwarded hop by hop as PIM control messages |

Although Auto-RP distributes information used by PIM, **Auto-RP messages themselves are not PIM messages**.

Also, `224.0.1.39` and `224.0.1.40` belong to the `224.0.1.0/24` internetwork control block, not the link-local `224.0.0.0/24` block.

Therefore, unlike PIM Hellos and ordinary IPv4 BSMs, Auto-RP messages can be forwarded across multiple routed hops using their configured TTL.

## Auto-RP Operation

Consider the following topology:

```text
                  R1
                   |
               R2 (MA)
                /    \
               /      \
         R3 (C-RP)  R4 (C-RP)
          3.3.3.3    4.4.4.4
                        |
                        R5
```

Assume:

- R2 is configured as an Auto-RP Mapping Agent.
- R3 and R4 are Candidate RPs for `239.0.0.0/8`.
- PIM is enabled on the relevant routed interfaces.
- Auto-RP control messages can be forwarded throughout the multicast domain.

### Step 1: Candidate RPs Send RP-Announce Messages

R3 and R4 periodically send **RP-Announce messages** to `224.0.1.39`.

```text
R3 (C-RP) ------\
                 \
                  > 224.0.1.39
                 /       |
R4 (C-RP) ------/        v
                       R2 (MA)
```

For example, R3 advertises:

```text
RP-Announce

RP Address: 3.3.3.3
Group Range: 239.0.0.0/8
```

R4 advertises:

```text
RP-Announce

RP Address: 4.4.4.4
Group Range: 239.0.0.0/8
```

Unlike BSR C-RP advertisements, **these messages are multicast rather than unicast to a known controller address**.

Candidate RPs do not need to discover the Mapping Agent's address before sending their announcements.

This differs from BSR, where C-RPs first learn the elected BSR through Bootstrap Messages.

### Step 2: Mapping Agent Selects the RP

R2 receives the announcements and builds a database of Candidate RPs.

For example:

| Group Range | Candidate RP |
|---|---|
| `239.0.0.0/8` | `3.3.3.3` |
| `239.0.0.0/8` | `4.4.4.4` |

The Mapping Agent resolves conflicting RP announcements and selects an RP for each group range.

When multiple Candidate RPs advertise the same group range, the Mapping Agent selects the **highest RP IP address**.

Therefore:

```text
Group Range: 239.0.0.0/8

Candidate RPs:
  3.3.3.3
  4.4.4.4

Selected RP: 4.4.4.4
```

Overlapping group ranges are also considered, with more specific group ranges taking precedence where applicable.

This differs significantly from BSR, where routers consider Candidate RP priorities and may use RP hashing.

**Auto-RP does not use the BSR RP hashing algorithm.**

### Step 3: Mapping Agent Sends RP-Discovery Messages

After selecting the RP mappings, R2 periodically sends **RP-Discovery messages** to `224.0.1.40`.

```text
                 R2 (MA)
                    |
                    | RP-Discovery
                    | 224.0.1.40
                    v
              Multicast Network
               /     |      \
              R1     R3      R4
                              |
                              R5
```

The discovery message contains the Mapping Agent's selected group-to-RP mappings.

For example:

```text
RP-Discovery

Group Range: 239.0.0.0/8
RP Address:  4.4.4.4
```

All routers receiving the discovery message learn the same mapping.

### Step 4: Routers Use the Learned RP Mapping

R5 receives the RP-Discovery message and installs:

```text
Group: 239.1.1.1
RP:    4.4.4.4
```

The router can then use the RP address for normal PIM-SM operations, including:

- Sending `(*,G)` Joins toward the RP.
- Determining the destination of PIM Register messages when acting as a source-facing DR.

**The receiving routers do not repeat the Mapping Agent's RP selection process.**

They use the mappings advertised by the Mapping Agent.

## Auto-RP vs. BSR

The following table summarizes the most important differences.

| Feature | Auto-RP | BSR |
|---|---|---|
| Origin | Cisco-developed mechanism | Standards-based, RFC 5059 |
| Address families | **IPv4 only** | IPv4 and IPv6 |
| Main roles | Candidate RP, Mapping Agent | Candidate RP, Candidate BSR |
| RP announcement destination | `224.0.1.39` | Unicast to elected BSR |
| RP information distribution | `224.0.1.40` | PIM Bootstrap Messages |
| Message transport | UDP port 496 | PIM, IP protocol 103 |
| RP selection performed by | **Mapping Agent** | **Each receiving router** |
| RP selection among equal group ranges | Highest RP IP | Lowest RP priority, then highest hash |
| Hash mask | Not used | Advertised by BSR |
| RP information distributed | Selected group-to-RP mappings | RP-Set containing eligible Candidate RPs |
| RP discovery before multicast forwarding | Requires Auto-RP control-group forwarding | BSMs use special hop-by-hop forwarding |
| Primary IOS XE bootstrap solution | `ip pim autorp listener` | No equivalent required |

The biggest conceptual distinction is:

**Auto-RP distributes the Mapping Agent's selection, while BSR distributes the candidates and lets each router select.**

## The Auto-RP Bootstrap Problem

An important limitation of Auto-RP is that it uses **multicast traffic to distribute the RP information needed for multicast forwarding**.

Consider a network using only PIM Sparse Mode.

In ordinary PIM-SM ASM operation, routers need a group-to-RP mapping before they can build the shared tree for a multicast group.

However, Auto-RP uses these multicast groups to distribute RP information:

```text
224.0.1.39 - RP-Announce
224.0.1.40 - RP-Discovery
```

This creates a circular dependency:

```text
Need RP mapping
      |
      v
Need Auto-RP messages
      |
      v
Need multicast forwarding
      |
      v
Need RP mapping
```

If routers cannot forward the Auto-RP groups, the Mapping Agent might never receive the RP announcements, and other routers might never receive RP-Discovery messages.

**BSR avoids this bootstrap problem because Bootstrap Messages are PIM control messages forwarded hop by hop, without requiring an existing multicast distribution tree.**

### Solution 1: Auto-RP Listener

The preferred IOS XE solution is:

```text
R1(config)# ip pim autorp listener
```

This global command enables **Dense Mode flooding specifically for the two Auto-RP multicast groups**, even when the interfaces operate in PIM Sparse Mode.

Configure PIM Sparse Mode normally:

```text
R1(config)# ip pim autorp listener

R1(config)# interface Ethernet0/0
R1(config-if)# ip pim sparse-mode

R1(config)# interface Ethernet0/1
R1(config-if)# ip pim sparse-mode
```

The forwarding behavior is:

| Multicast Group | Forwarding Behavior |
|---|---|
| `224.0.1.39` | Dense Mode flooding |
| `224.0.1.40` | Dense Mode flooding |
| Other ASM groups | Normal PIM Sparse Mode |

The Auto-RP control groups can therefore reach the Mapping Agents and PIM routers before the RP mappings are learned.

**Auto-RP Listener does not turn every multicast group into a Dense Mode group.**

### Solution 2: PIM Sparse-Dense Mode

Alternatively, interfaces can use:

```text
R1(config)# interface Ethernet0/0
R1(config-if)# ip pim sparse-dense-mode
```

Sparse-dense mode determines forwarding behavior based on whether an RP mapping is known:

- **RP known:** Operates in Sparse Mode.
- **No RP known:** Can operate in Dense Mode.

This allows Auto-RP groups to be flooded before the routers learn RP mappings.

However, it can also cause other groups without RP mappings to be flooded in Dense Mode.

For this reason, **PIM Sparse Mode with `ip pim autorp listener` is generally preferable**.

## IOS XE Configuration

The following example uses:

| Router | Role | Address |
|---|---|---|
| R2 | Mapping Agent | `2.2.2.2` |
| R3 | Candidate RP | `3.3.3.3` |

Assume unicast routing and PIM neighbor connectivity are already operational.

### Enable Multicast Routing and Auto-RP Listener

On the routers participating in Auto-RP:

```text
R1(config)# ip multicast-routing
R1(config)# ip pim autorp listener
```

Configure PIM Sparse Mode on the relevant routed interfaces:

```text
R1(config)# interface Ethernet0/0
R1(config-if)# ip pim sparse-mode
```

Repeat the appropriate configuration on the other routers and interfaces in the Auto-RP domain.

### Configure a Candidate RP

On R3:

```text
R3(config)# interface Loopback0
R3(config-if)# ip address 3.3.3.3 255.255.255.255
R3(config-if)# ip pim sparse-mode

R3(config)# ip pim send-rp-announce Loopback0 scope 16
```

This configures R3 to send RP-Announce messages to `224.0.1.39`.

The command arguments are:

| Argument | Purpose |
|---|---|
| `Loopback0` | Supplies the advertised Candidate RP address |
| `scope 16` | Sets the IP TTL of RP-Announce messages to 16 |

Without a `group-list`, the router advertises itself for all applicable multicast groups.

To restrict the advertisement to `239.0.0.0/8`:

```text
R3(config)# ip access-list standard RP-GROUPS
R3(config-std-nacl)# permit 239.0.0.0 0.255.255.255

R3(config)# ip pim send-rp-announce Loopback0 scope 16 group-list RP-GROUPS
```

The standard ACL matches **multicast group addresses**, not multicast source addresses.

### Configure a Mapping Agent

On R2:

```text
R2(config)# interface Loopback0
R2(config-if)# ip address 2.2.2.2 255.255.255.255
R2(config-if)# ip pim sparse-mode

R2(config)# ip pim send-rp-discovery Loopback0 scope 16
```

This configures R2 to receive Candidate RP announcements and distribute selected RP mappings using `224.0.1.40`.

The `scope 16` keyword specifies the TTL of its RP-Discovery messages.

The Candidate RP and Mapping Agent functions may also be configured on the same router.

### The Meaning of `scope`

In Auto-RP commands, **`scope` specifies an IPv4 TTL**, not an IPv6 multicast scope or a group address range.

For example:

```text
R3(config)# ip pim send-rp-announce Loopback0 scope 16
```

This limits the distance that RP-Announce messages can travel through the routed network.

The two Auto-RP roles have independent TTL settings:

```text
ip pim send-rp-announce Loopback0 scope 16
ip pim send-rp-discovery Loopback0 scope 16
```

The announcement TTL must be sufficient to reach the Mapping Agents.

The discovery TTL must be sufficient to reach the routers that need the resulting RP mappings.

## Auto-RP Timers

IOS XE periodically sends Auto-RP announcements and discovery messages.

| Timer | Default | Purpose |
|---|---|---|
| **RP-Announce Interval** | 60 seconds | Candidate RP sends advertisements |
| **RP-Discovery Interval** | 60 seconds | Mapping Agent distributes selected mappings |

Both intervals can be configured with the optional `interval` keyword.

For example:

```text
R3(config)# ip pim send-rp-announce Loopback0 scope 16 interval 30
```

```text
R2(config)# ip pim send-rp-discovery Loopback0 scope 16 interval 30
```

More frequent updates increase control-plane traffic but can reduce the delay in propagating changes.

Auto-RP mappings are maintained as soft state and expire if they are not refreshed.

The detailed advertisement format and expiration behavior are covered in the following pages.

## Auto-RP Filtering and Boundaries

Auto-RP supports several mechanisms for controlling which RP information is advertised and where it can propagate.

### Candidate RP Group Filtering

On a Candidate RP, use a standard ACL with `group-list` to restrict advertised multicast group ranges:

```text
R3(config)# ip access-list standard RP-GROUPS
R3(config-std-nacl)# permit 239.1.0.0 0.0.255.255

R3(config)# ip pim send-rp-announce Loopback0 scope 16 group-list RP-GROUPS
```

Only the permitted group ranges are announced.

### Mapping Agent Announcement Filtering

By default, an Auto-RP Mapping Agent accepts announcements from all Candidate RPs.

The `ip pim rp-announce-filter` command allows the Mapping Agent to filter incoming RP-Announce messages based on **Candidate RP addresses** and **multicast group ranges**.

```text
R2(config)# ip pim rp-announce-filter rp-list 1 group-list 2
```

Both parameters reference **standard IPv4 ACLs**:

- **`rp-list`:** Identifies the Candidate RPs subject to the filtering rule.
- **`group-list`:** Determines which multicast groups those RPs are permitted to advertise.

Unlike a conventional whitelist, Candidate RPs not matched by `rp-list` are not automatically rejected.

See [RP Announcement Filtering](../../filtering/rp-announcement-filter.md) for detailed configuration and filtering behavior.

### Auto-RP Boundaries

IOS XE also supports multicast boundaries to limit the propagation of Auto-RP information.

For example, the `filter-autorp` keyword can be used with `ip multicast boundary` to filter group ranges contained inside Auto-RP announcements and discovery messages.

This is different from BSR's `ip pim bsr-border`, which blocks Bootstrap Messages on an interface.

For more details, see:

- [Multicast Boundaries and Administrative Scope](../../filtering/multicast-boundary.md) for multicast boundaries and the `filter-autorp` keyword.
- [RP Announcement Filtering](../../filtering/rp-announcement-filter.md) for filtering Candidate RP announcements on Auto-RP Mapping Agents.
- [Candidate BSR and BSR Election](../bsr/candidate-bsr.md#bsr-borders) for the BSR border feature.

## Mapping Agent Redundancy

Multiple Mapping Agents can be configured for redundancy.

Each Mapping Agent can receive RP-Announce messages and independently determine which RP mappings to advertise.

Unlike BSR, **Auto-RP does not use a PIM Bootstrap Message election between Candidate BSRs**.

When multiple Mapping Agents are present, IOS XE uses Mapping Agent address preference and received RP-Discovery information to resolve competing advertisements.

It is important that Mapping Agents:

- Receive the same Candidate RP announcements.
- Use consistent filtering policies.
- Have sufficient scope to distribute RP-Discovery messages throughout the intended domain.

Otherwise, different routers might learn inconsistent RP mappings.

As with BSR, Auto-RP redundancy does not guarantee uninterrupted multicast forwarding following an RP failure.

## Common Troubleshooting Problems

| Problem | Possible Cause |
|---|---|
| Mapping Agent receives no RP announcements | `224.0.1.39` not forwarded, insufficient announcement TTL, or Candidate RP misconfiguration |
| Routers do not learn RP mappings | `224.0.1.40` not forwarded, insufficient discovery TTL, or missing Mapping Agent |
| Auto-RP fails in an all-sparse-mode network | `ip pim autorp listener` not enabled |
| Unexpected RP selected | Conflicting Candidate RP announcements or overlapping group ranges |
| Candidate RP not accepted | Mapping Agent `rp-announce-filter` ACL |
| RP mappings differ across routers | Multiple Mapping Agents, filtering, incomplete discovery propagation, or expired mappings |
| Unwanted dense-mode multicast flooding | Use of `ip pim sparse-dense-mode` instead of sparse mode with Auto-RP Listener |

A useful troubleshooting sequence is:

1. Verify Candidate RP announcements.
2. Verify that `224.0.1.39` can reach the Mapping Agent.
3. Verify that the Mapping Agent selects the expected RP.
4. Verify that `224.0.1.40` can reach all required routers.
5. Verify the resulting group-to-RP mappings.
6. Verify normal PIM-SM forwarding toward the selected RP.

## Key Points

- **Auto-RP is a Cisco-developed IPv4 RP discovery mechanism.**
- It uses two roles: **Candidate RP** and **Mapping Agent**.
- Candidate RPs multicast **RP-Announce messages to `224.0.1.39`**.
- Mapping Agents multicast **RP-Discovery messages to `224.0.1.40`**.
- Auto-RP uses **UDP port 496**, not PIM IP protocol 103.
- **The Mapping Agent selects the RP and distributes the selected mappings.**
- With BSR, **each router independently selects an RP from the advertised RP-Set**.
- Auto-RP does **not** use the BSR RP hashing algorithm.
- In traditional IOS XE Auto-RP, the **highest RP address is preferred** when Candidate RPs advertise the same group range.
- Auto-RP requires its two control multicast groups to be forwarded before RP mappings are available.
- **`ip pim autorp listener`** enables Dense Mode forwarding for the Auto-RP groups while other traffic remains in Sparse Mode.
- `ip pim sparse-dense-mode` is another solution, but can cause unwanted Dense Mode flooding.
- **`scope` specifies the TTL** of Auto-RP control messages.
- Use `ip pim send-rp-announce` to configure a Candidate RP.
- Use `ip pim send-rp-discovery` to configure a Mapping Agent.
- The default RP-Announce and RP-Discovery intervals are **60 seconds**.
- Use `ip pim rp-announce-filter` on the Mapping Agent to restrict accepted Candidate RP announcements.
- **Auto-RP does not support IPv6.**

## References

- [Cisco IOS XE: Configuring Basic IP Multicast](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-multicast/b-ip-multicast/m_imc_basic_cfg-0.html)
- [Cisco Catalyst 9000 IOS XE: Multicast Configuration Guide, PIM](https://www.cisco.com/c/en/us/td/docs/switches/lan/c9000/multicast/multicast-configuration-guide/pim.html)
- [Cisco: Secure IP Multicast Deployments](https://www.cisco.com/c/en/us/support/docs/ip/ip-multicast/218004-secure-ip-multicast-deployments.html)
- [Cisco: Review the IP Multicast Quick-Start Configuration Guide](https://www.cisco.com/c/en/us/support/docs/ip/ip-multicast/9356-48.html)