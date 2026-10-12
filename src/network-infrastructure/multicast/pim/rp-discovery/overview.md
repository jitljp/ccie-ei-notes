# 1.6.c (ii) Static RP, BSR, Auto-RP

In traditional PIM Sparse Mode (PIM-SM), the **Rendezvous Point (RP)** acts as a meeting point between multicast sources and receivers.

When a receiver joins a multicast group, its Last-Hop Router (LHR) initially builds a shared tree toward the RP.

Meanwhile, the source-facing Designated Router (DR) registers active multicast sources with the RP.

For this process to work, PIM routers must know **which RP is responsible for each multicast group**.

This is called **Group-to-RP Mapping**.

There are three primary methods of distributing RP information:

- **Static RP:** Manually configure the RP address on each router.
- **Bootstrap Router (BSR):** Dynamically distribute RP information using PIM messages.
- **Auto-RP:** Dynamically distribute RP information using Cisco's Auto-RP mechanism.

## Role of the RP

Consider the following network:

```text
             Source
           10.1.1.10
               |
            R1 (FHR)
               |
               R2
              /  \
             /    \
       R3 (RP)     R4
       3.3.3.3      |
           \        |
            \ ----R5 (LHR)
                    |
                Receiver
                 239.1.1.1
```

Assume the RP mapping is:

```text
Group: 239.1.1.1
RP:    3.3.3.3
```

The RP serves two important functions:

| Function | Operation |
|---|---|
| Receiver discovery | LHRs send `(*,G)` Joins toward the RP, establishing the shared tree |
| Source discovery | Source-facing DRs unicast PIM Register messages to the RP |

When an interested receiver and an active source exist, the RP can forward multicast traffic along the shared tree.

The RP also establishes an `(S,G)` path toward the source to receive multicast packets natively.

Once the LHR switches to a Shortest-Path Tree (SPT), the multicast traffic may bypass the RP entirely.

The RP is essential for initial source and receiver discovery in PIM-SM ASM, but does not necessarily remain in the multicast forwarding path,
especially after the SPT switchover.

## Group-to-RP Mapping

A multicast network can have multiple RPs, each responsible for different multicast groups.

For example:

| Multicast Group Range | RP |
|---|---|
| `239.1.1.0/24` | `1.1.1.1` |
| `239.2.2.0/24` | `2.2.2.2` |
| `239.3.3.0/24` | `3.3.3.3` |

When a router needs to build a shared tree for `239.2.2.10`, it determines that the RP is `2.2.2.2` and performs an RPF lookup toward that address.

Similarly, when a source-facing DR receives traffic for `239.2.2.10`, it uses the mapping to identify the destination of its PIM Register messages.

**Routers must agree on the effective RP for a given multicast group.**

Inconsistent RP mappings can result in failed shared-tree Joins, unsuccessful source registration, or multicast traffic loss.

> **Note:** Multiple RPs can exist for different groups, but this is different from Anycast RP, where multiple routers share the same RP address. Anycast RP is covered separately.

## Static RP

With **Static RP**, the RP address is manually configured on the PIM routers.

For example:

```text
R1(config)# ip pim rp-address 3.3.3.3
```

This maps the default applicable IPv4 ASM group range to RP `3.3.3.3`.

The command can also reference an ACL to restrict the mapping to particular multicast groups.

### Characteristics

- Simple to configure and troubleshoot.
- No RP discovery messages are required.
- Each router must be configured with the appropriate RP mappings.
- Changes to RP assignments must be distributed manually.
- Does not provide automatic RP discovery or replacement if an RP fails.

Static RP is particularly useful in small networks or when precise control over RP selection is required.

See [Static RP](static-rp.md).

## Bootstrap Router (BSR)

**BSR** is a standards-based RP discovery mechanism defined for PIMv2.

Instead of manually configuring the RP address on every router, BSR dynamically distributes information about available RPs throughout the PIM domain.

BSR uses two distinct roles:

| Role | Function |
|---|---|
| **Candidate BSR (C-BSR)** | Participates in an election to become the Bootstrap Router |
| **Candidate RP (C-RP)** | Advertises its willingness to serve as an RP for specified multicast groups |

### Basic BSR Operation

```text
Candidate BSRs
      |
      v
BSR Election
      |
      v
Elected BSR
      ^
      |
Candidate RP Advertisements
      |
      v
BSR distributes RP-Set
      |
      v
All PIM Routers
      |
      v
Each router selects RP for group
```

The process is:

1. Candidate BSRs participate in an election to select the active BSR.
2. Candidate RPs unicast their RP advertisements to the elected BSR.
3. The BSR collects these advertisements into an **RP-Set**.
4. The BSR distributes the RP-Set throughout the PIM domain using **Bootstrap Messages (BSMs)**.
5. Each PIM router independently selects the RP for each multicast group.

Bootstrap Messages are forwarded hop by hop between PIM routers, using PIM control-plane signaling.

For IPv4, BSMs use the All PIM Routers address:

```text
224.0.0.13
```

An important distinction is that **the BSR does not centrally select the RP for every group**.

Instead, it distributes the candidate information. Each router applies the RP selection algorithm, including priority and hashing where applicable.

### Basic IOS XE Commands

Configure a candidate BSR:

```text
R2(config)# ip pim bsr-candidate Loopback0 30 10
```

- **`Loopback0`:** Interface whose IPv4 address is used as the candidate BSR address.
- **`30`:** Hash mask length. Specifies the number of most significant multicast group address bits used in the RP selection hash calculation.
- **`10`:** BSR priority. The candidate with the **highest priority** wins the BSR election. If priorities are equal, the highest BSR IP address wins.

Configure a candidate RP:

```text
R3(config)# ip pim rp-candidate Loopback0
```

The candidate BSR and candidate RP functions can reside on the same router or on different routers.

BSR supports both IPv4 and IPv6 PIM-SM.

See [BSR](bsr/overview.md).

## Auto-RP

**Auto-RP** is Cisco's mechanism for dynamically distributing RP information in IPv4 PIM networks.

Unlike BSR, Auto-RP uses special multicast groups to exchange RP information.

Auto-RP has two important roles:

| Role | Function |
|---|---|
| **Candidate RP** | Announces its willingness to serve as an RP |
| **Mapping Agent** | Receives RP announcements, selects RP mappings, and distributes them to routers |

### Basic Auto-RP Operation

```text
Candidate RPs
      |
      | RP-Announce
      | 224.0.1.39
      v
  Mapping Agent
      |
      | RP-Discovery
      | 224.0.1.40
      v
  All PIM Routers
```

Auto-RP uses two well-known IPv4 multicast groups:

| Group | Message | Purpose |
|---|---|---|
| `224.0.1.39` | RP-Announce | Candidate RPs announce themselves to mapping agents |
| `224.0.1.40` | RP-Discovery | Mapping agents distribute group-to-RP mappings |

The mapping agent selects the RP mapping for each group and advertises it to other routers.

This differs from BSR, where the BSR distributes an RP-Set and each receiving router performs its own RP selection.

### Basic IOS XE Commands

Configure a candidate RP:

```text
R3(config)# ip pim send-rp-announce Loopback0 scope 16
```

Configure a mapping agent:

```text
R2(config)# ip pim send-rp-discovery Loopback0 scope 16
```

The `scope` parameter controls the TTL used for Auto-RP announcements and discovery messages, limiting how far they can travel through the network.

### The Auto-RP Bootstrap Problem

Auto-RP uses multicast traffic to distribute the very RP mappings that PIM-SM needs to forward multicast traffic.

This creates a bootstrap problem in networks using PIM Sparse Mode exclusively.

Before routers learn their RP mappings, how can they forward Auto-RP traffic?

One solution is:

```text
R1(config)# ip pim autorp listener
```

This allows the two Auto-RP control groups, `224.0.1.39` and `224.0.1.40`, to be flooded using Dense Mode behavior across PIM Sparse Mode interfaces.

The rest of the multicast traffic continues to use normal Sparse Mode forwarding.

Auto-RP is an IPv4 mechanism; it is not used for IPv6 RP discovery.

See [Auto-RP](auto-rp/overview.md).

## Static RP vs. BSR vs. Auto-RP

| Feature | Static RP | BSR | Auto-RP |
|---|---|---|---|
| Type | Manual | Dynamic | Dynamic |
| Standards-based | Yes | Yes | Cisco mechanism |
| IPv4 support | Yes | Yes | Yes |
| IPv6 support | Yes | Yes | No |
| RP information source | Local configuration | RP-Set from elected BSR | RP-Discovery from mapping agent |
| Central RP selection | Manually specified | No, each router selects | Mapping agent selects |
| Automatic distribution | No | Yes | Yes |
| RP candidate advertisements | Not required | Unicast to BSR | Multicast to `224.0.1.39` |
| Distribution mechanism | None | PIM Bootstrap Messages | Multicast to `224.0.1.40` |

## RP Redundancy

Dynamic RP discovery can help routers learn alternative RP candidates if the currently selected RP becomes unavailable.

However, **RP discovery and RP redundancy are not the same thing**.

BSR and Auto-RP distribute mappings and support changes to those mappings. They do not, by themselves, guarantee uninterrupted multicast traffic when an active RP fails.

RP redundancy can also be implemented using mechanisms such as:

- IPv4 Anycast RP with MSDP.
- IPv6 PIM Anycast RP.

These mechanisms are covered in separate sections.

## Key Points

- Traditional PIM-SM ASM requires **Group-to-RP mappings** so routers know where to send shared-tree Joins and PIM Registers.
- **Static RP** manually configures RP mappings on routers.
- **BSR** dynamically distributes candidate RP information using PIM Bootstrap Messages.
- With BSR, **each router independently selects its RP** based on the advertised information.
- **Auto-RP** uses a mapping agent to select and distribute RP mappings.
- Auto-RP uses `224.0.1.39` for RP announcements and `224.0.1.40` for RP discovery.
- `ip pim autorp listener` allows Auto-RP control traffic to be flooded across Sparse Mode interfaces.
- Static RP and BSR support both IPv4 and IPv6; Auto-RP is IPv4-only.
- Different multicast group ranges can use different RPs.
- RP discovery determines the appropriate RP, but does not guarantee that the RP is reachable or that multicast forwarding will succeed.
- **SSM does not require an RP**, because receivers join source-specific trees directly.