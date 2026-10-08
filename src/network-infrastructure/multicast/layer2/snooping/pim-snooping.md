
# PIM Snooping

**PIM Snooping** is a Layer 2 feature that allows a switch to inspect **Protocol Independent Multicast (PIM)** messages exchanged between multicast routers and selectively forward multicast traffic to router-facing ports.

It complements IGMP Snooping:

- **IGMP Snooping:** Controls multicast forwarding toward receiver (host) ports.
- **PIM Snooping:** Controls multicast forwarding toward multicast router ports.

PIM Snooping is **disabled by default** on Cisco IOS XE switches that support it.

## PIM Snooping Operation

By default, a switch running IGMP Snooping forwards multicast traffic to all multicast router ports, even if some routers do not need the traffic.

**PIM Snooping prevents this unnecessary forwarding** by examining PIM control messages exchanged between routers.

The switch examines:

- **PIM Hello messages:** Discover PIM routers connected to the VLAN.
- **PIM Join messages:** Identify routers requesting multicast traffic.
- **PIM Prune messages:** Identify routers that no longer require certain multicast traffic.

Based on these messages, the switch maintains multicast forwarding state for each VLAN and forwards traffic only to the necessary router ports (subject to exceptions such as Designated Router flooding).

PIM Snooping also optimizes PIM control traffic by forwarding Join/Prune messages only toward the appropriate upstream router instead of flooding them to all router ports.

PIM neighbor and multicast forwarding state age out according to the holdtimes advertised in PIM messages.

**Note:** The switch does not participate in PIM routing. It simply examines PIM messages passing through it and does not require an IP address or PIM configuration on an SVI.

## Configuration

On supported IOS XE switches, enable PIM Snooping globally and for the required VLAN:

```text
SW1(config)# ip pim snooping
SW1(config)# ip pim snooping vlan 10
```

To disable PIM Snooping:

```text
SW1(config)# no ip pim snooping vlan 10
SW1(config)# no ip pim snooping
```

### Designated Router Flooding

By default, PIM Snooping forwards multicast traffic to the **PIM Designated Router (DR)** even if it has not explicitly requested that traffic.

This behavior can be disabled:

```text
SW1(config)# no ip pim snooping dr-flood
```

With DR flooding disabled, the DR receives multicast traffic only for groups it has explicitly joined.

**Important:** Do not disable DR flooding on VLANs containing directly connected multicast sources, because doing so can prevent necessary traffic from reaching the DR.

## Verification

```text
SW1# show ip pim snooping
SW1# show ip pim snooping vlan 10
SW1# show ip pim snooping vlan 10 neighbor
SW1# show ip pim snooping vlan 10 mroute
SW1# show ip pim snooping vlan 10 statistics
```

These commands display the operational status, discovered PIM neighbors, multicast forwarding entries, and statistics.

## Important Considerations

- PIM Snooping is supported for IPv4 multicast forwarding; it does not replace MLD Snooping for IPv6 receivers.
- **IGMP Snooping should also be enabled** to ensure multicast traffic reaches local receivers correctly.
- Multicast routers must support PIMv2.
- Auto-RP traffic (224.0.1.39 and 224.0.1.40) is flooded to all PIM router ports.
- Availability and specific behavior depend on the switch platform and IOS XE release.
