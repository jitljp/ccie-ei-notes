# Static RP

In PIM Sparse Mode (PIM-SM), a **Static RP** is a Rendezvous Point (RP) whose address is manually configured on the multicast routers.

Unlike BSR and Auto-RP, Static RP does not require an RP discovery protocol.

Instead, each router is explicitly configured with the **Group-to-RP mapping** it should use.

Static RP is supported for both IPv4 and IPv6 multicast.

## Static RP Operation

Consider the following network:

```text
              R3 (RP)
              3.3.3.3
                 |
                 R2
                /  \
               /    \
              R1    R4
             (FHR)  (LHR)
               |      |
             Source Receiver
                    239.1.1.1
```

Assume every router is configured with:

```text
Group: 239.1.1.1
RP:    3.3.3.3
```

When the receiver joins `239.1.1.1`:

1. R4 determines that the RP is `3.3.3.3`.
2. R4 sends a `(*,G)` Join toward the RP, following its RPF path.
3. R2 forwards the Join toward R3, establishing the shared tree.

When the source begins transmitting:

1. R1 determines that the RP is `3.3.3.3`.
2. R1 encapsulates the initial multicast packets in PIM Register messages.
3. R1 unicasts the Registers to R3.

**All routers must agree on the RP for a given multicast group**, regardless of whether the mapping is configured statically or learned dynamically.

## IPv4 Static RP Configuration

The basic command is:

```text
R1(config)# ip pim rp-address 3.3.3.3
```

This configures `3.3.3.3` as the RP for all applicable IPv4 ASM groups unless another mapping takes precedence.

For a simple static RP deployment, configure the same command on **every PIM router, including the RP itself**.

### Basic Example

On R3, configure the RP address on a loopback interface:

```text
R3(config)# ip multicast-routing

R3(config)# interface Loopback0
R3(config-if)# ip address 3.3.3.3 255.255.255.255
R3(config-if)# ip pim sparse-mode

R3(config)# interface Ethernet0/0
R3(config-if)# ip pim sparse-mode

R3(config)# ip pim rp-address 3.3.3.3
```

On the other PIM routers:

```text
R1(config)# ip multicast-routing

R1(config)# interface Ethernet0/0
R1(config-if)# ip pim sparse-mode

R1(config)# ip pim rp-address 3.3.3.3
```

Enable PIM on all other relevant multicast interfaces.

Using a loopback address for the RP provides a stable address independent of a particular physical link.

The RP address must be reachable through the network's routing information.

## IPv4 Group-Specific RP Mappings

The optional ACL argument allows the RP to be assigned only to particular multicast groups.

For example, configure R3 as the RP for:

```text
239.1.1.0/24
```

First, create a standard IPv4 ACL:

```text
R1(config)# ip access-list standard RP-GROUPS
R1(config-std-nacl)# permit 239.1.1.0 0.0.0.255
```

Then apply it:

```text
R1(config)# ip pim rp-address 3.3.3.3 RP-GROUPS
```

This maps groups from `239.1.1.0` through `239.1.1.255` to RP `3.3.3.3`.

The ACL matches **multicast group addresses (G)**, not source addresses (S).

Only groups permitted by the ACL are included in this mapping. Other groups must obtain an RP mapping through another configuration or discovery mechanism if they require an RP.

### Multiple RPs for Different Groups

Different RPs can serve different multicast group ranges.

For example:

| Group Range | RP |
|---|---|
| `239.1.1.0/24` | `1.1.1.1` |
| `239.2.2.0/24` | `2.2.2.2` |

Configure:

```text
R1(config)# ip access-list standard RP1-GROUPS
R1(config-std-nacl)# permit 239.1.1.0 0.0.0.255

R1(config)# ip access-list standard RP2-GROUPS
R1(config-std-nacl)# permit 239.2.2.0 0.0.0.255

R1(config)# ip pim rp-address 1.1.1.1 RP1-GROUPS
R1(config)# ip pim rp-address 2.2.2.2 RP2-GROUPS
```

Apply consistent mappings throughout the PIM domain.

A single RP can also serve multiple group ranges by including multiple `permit` statements in one ACL.

## IPv4 Static RP Override

When static RP configuration is combined with dynamic RP discovery, Cisco IOS XE supports the **`override`** keyword.

For example:

```text
R1(config)# ip pim rp-address 3.3.3.3 RP-GROUPS override
```

The behavior is:

| Configuration | RP Selection |
|---|---|
| Static RP without `override` | A conflicting dynamic RP mapping takes precedence |
| Static RP with `override` | The static RP mapping takes precedence over the conflicting dynamic mapping |

Dynamic mappings may come from BSR or Auto-RP.

This is useful when most group mappings are learned dynamically but selected groups must use a particular statically configured RP.

### Overlapping Static RP Mappings

A less obvious IOS XE behavior concerns multiple static RP configurations that match the same group.

**If multiple static RP mappings match the same group, IOS XE selects the RP with the highest IP address, regardless of reachability.**

For example:

```text
R1(config)# ip pim rp-address 1.1.1.1
R1(config)# ip pim rp-address 2.2.2.2
```

Both mappings apply to group `239.1.1.1`.

The selected RP is:

```text
2.2.2.2
```

This does not provide automatic RP failover. If `2.2.2.2` becomes unreachable, the router does not simply select `1.1.1.1` based on reachability.

> **Note:** IOS XE also restricts each static IPv4 RP address to one `ip pim rp-address` configuration entry. To assign multiple group ranges to the same RP, combine them in a single ACL.

## IPv6 Static RP Configuration

IPv6 PIM-SM uses the equivalent global command:

```text
R1(config)# ipv6 pim rp-address 2001:DB8:3::3
```

This configures `2001:DB8:3::3` as the RP for applicable routable IPv6 ASM multicast groups.

### Basic Example

On the RP:

```text
R3(config)# ipv6 unicast-routing
R3(config)# ipv6 multicast-routing

R3(config)# interface Loopback0
R3(config-if)# ipv6 address 2001:DB8:3::3/128

R3(config)# ipv6 pim rp-address 2001:DB8:3::3
```

On the other routers:

```text
R1(config)# ipv6 unicast-routing
R1(config)# ipv6 multicast-routing
R1(config)# ipv6 pim rp-address 2001:DB8:3::3
```

IPv6 multicast routing and PIM must be operational on the relevant interfaces. On IOS XE platforms where `ipv6 multicast-routing` enables IPv6 PIM automatically on IPv6-enabled interfaces, no separate per-interface PIM command is required.

Verify with:

```text
R1# show ipv6 pim interface
```

The RP's IPv6 address must be reachable by the other routers.

## IPv6 Group-Specific RP Mappings

As with IPv4, IPv6 supports restricting static RP mappings to particular multicast groups.

However, IPv6 uses an **IPv6 ACL** instead of an IPv4 standard ACL.

For example, assign `2001:DB8:3::3` as the RP for:

```text
FF15::/16
```

Configure:

```text
R1(config)# ipv6 access-list RP-GROUPS-V6
R1(config-ipv6-acl)# permit ipv6 any FF15::/16

R1(config)# ipv6 pim rp-address 2001:DB8:3::3 RP-GROUPS-V6
```

The IPv6 ACL syntax is important:

```text
permit ipv6 any FF15::/16
            |       |
            |       +-- Multicast group range
            +---------- Any source
```

The **destination field** matches the multicast group address.

The source field is not used to select a particular multicast source for the RP mapping.

Unlike the IPv4 standard ACL, an IPv6 ACL uses an extended-style source/destination format.

### Multiple IPv6 RPs

For example:

| Group Range | RP |
|---|---|
| `FF15::/16` | `2001:DB8:1::1` |
| `FF1E::/16` | `2001:DB8:2::2` |

Configure:

```text
R1(config)# ipv6 access-list RP1-GROUPS-V6
R1(config-ipv6-acl)# permit ipv6 any FF15::/16

R1(config)# ipv6 access-list RP2-GROUPS-V6
R1(config-ipv6-acl)# permit ipv6 any FF1E::/16

R1(config)# ipv6 pim rp-address 2001:DB8:1::1 RP1-GROUPS-V6
R1(config)# ipv6 pim rp-address 2001:DB8:2::2 RP2-GROUPS-V6
```

Each range is mapped to a different RP.

> **Note:** IPv6 PIM also supports Embedded RP, which can derive the RP address from the multicast group address itself. This is separate from manually configured Static RP mappings.

## IOS XE Verification

### IPv4 RP Mapping

```text
R1# show ip pim rp mapping
```

This displays the configured and dynamically learned RP mappings.

For a particular multicast group, also use:

```text
R1# show ip pim rp 239.1.1.1
```

The RP mapping output is the primary way to verify the selected RP and how it was learned.

### IPv6 RP Mapping

For IPv6, use:

```text
R1# show ipv6 pim group-map
```

To display mappings learned through static configuration:

```text
R1# show ipv6 pim group-map info-source static
```

For a particular multicast group:

```text
R1# show ipv6 pim group-map FF15::101
```

Verify that the selected RP matches the expected static mapping.

### Verify RP Reachability

A correct RP mapping does not guarantee that the RP is reachable.

Check the multicast RPF path toward the RP.

IPv4:

```text
R1# show ip rpf 3.3.3.3
```

IPv6:

```text
R1# show ipv6 rpf 2001:DB8:3::3
```

These commands identify the selected RPF interface and upstream neighbor toward the RP.

### Verify Multicast Forwarding

IPv4:

```text
R1# show ip mroute 239.1.1.1
```

IPv6:

```text
R1# show ipv6 mroute FF15::101
```

On non-RP routers, the `(*,G)` incoming interface should normally follow the RPF path toward the RP.

On the RP itself, the root of the shared tree has no upstream PIM neighbor toward another RP.

## Common Static RP Problems

| Problem | Possible Cause |
|---|---|
| No RP mapping for a group | Group not permitted by the ACL, or mapping missing |
| Different routers select different RPs | Inconsistent static mappings or conflicting dynamic mappings |
| Static RP ignored | BSR or Auto-RP mapping takes precedence without `override` |
| RP mapping exists but no shared tree forms | RP unreachable or no valid RPF path |
| PIM Register messages fail | Incorrect RP mapping, unicast connectivity, or filtering |
| IPv6 mapping not applied | Incorrect IPv6 ACL, especially destination multicast range |
| RP fails and multicast stops working | Static RP does not automatically select another RP |

## Key Points

- **Static RP** manually defines the Group-to-RP mapping without BSR or Auto-RP.
- IPv4 uses `ip pim rp-address`.
- IPv6 uses `ipv6 pim rp-address`.
- Configure consistent RP mappings on every PIM router, including the RP itself.
- IPv4 uses a **standard ACL** to restrict RP mappings to specific multicast groups.
- IPv6 uses an **IPv6 ACL**, matching the multicast group in the destination field.
- The IPv4 `override` keyword makes a static mapping take precedence over conflicting dynamic mappings.
- When multiple IPv4 static mappings overlap, the **highest RP address** is selected regardless of reachability.
- Static RP does not automatically provide RP redundancy or failover.
- A valid RP mapping must be accompanied by a valid **RPF path toward the RP**.
- Verify IPv4 mappings with `show ip pim rp mapping` and IPv6 mappings with `show ipv6 pim group-map`.