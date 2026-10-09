
# IGMP Snooping Querier

An **IGMP snooping querier** allows a Layer 2 switch to generate IGMP Queries **without performing Layer 3 multicast routing**.

This is useful when IGMP snooping is enabled on a VLAN that has no multicast router.

Normally, a multicast router sends periodic IGMP General Queries, prompting receivers to refresh their memberships. Without a querier, dynamically learned IGMP snooping entries can eventually expire.

The IGMP snooping querier solves this problem by allowing the switch itself to generate Queries.

## Why an IGMP Snooping Querier Is Needed

Consider a VLAN containing a multicast source and receiver:

```text
                 SW1
                VLAN 10
                /    \
               /      \
           Source    Receiver
```

The receiver joins multicast group `239.1.1.1` by sending an IGMP Membership Report.

SW1 learns the receiver's port:

```text
VLAN 10
239.1.1.1
→ Ethernet0/2
```

However, there is no multicast router to send periodic IGMP General Queries.

The initial Membership Report allows SW1 to learn the group, but without periodic Queries, the membership entry may eventually expire.

With an IGMP snooping querier enabled:

```text
                 SW1
           Snooping Querier
                VLAN 10
                /    \
               /      \
           Source    Receiver
```

SW1 periodically generates General Queries.

The receiver responds with Membership Reports, allowing SW1 to refresh its snooping forwarding state.

## Basic Operation

The IGMP snooping querier performs three main functions:

1. **Generates periodic IGMP General Queries** to discover or refresh receiver memberships.
2. **Observes Membership Reports** to maintain Layer 2 multicast forwarding state.
3. **Detects an existing multicast querier** and stops originating periodic Queries when appropriate.

For example:

```text
SW1 sends General Query
          |
          v
Receiver sends Membership Report
          |
          v
SW1 refreshes listener port
          |
          v
Multicast traffic forwarded
only toward interested receivers
```

The switch performs these functions without building a multicast routing table or participating in PIM.

> The IGMP snooping querier does not forward multicast traffic between VLANs. It only provides the IGMP Queries needed to maintain membership information within a VLAN.

## IGMP Snooping Querier vs Multicast Router

Although an IGMP snooping querier and a multicast router can both send IGMP Queries, their responsibilities differ.

| Feature | Multicast Router | IGMP Snooping Querier |
|---|---|---|
| Sends IGMP Queries | Yes | Yes |
| Processes Membership Reports | Yes | Yes, for Layer 2 snooping |
| Maintains Layer 3 IGMP membership state | Yes | No |
| Routes multicast between subnets | With multicast routing enabled | No |
| Requires PIM | Commonly used with PIM | No |

For example:

```text
Multicast Router
→ sends IGMP Queries
→ learns receiver memberships
→ participates in multicast routing

IGMP Snooping Querier
→ sends IGMP Queries
→ maintains Layer 2 snooping memberships
→ does not route multicast
```

The snooping querier is therefore **not a replacement for multicast routing**.

It is intended primarily for VLANs where multicast communication occurs locally.

## Configuration

On Catalyst 9000 IOS XE, IGMP snooping is enabled by default, but the **IGMP snooping querier is disabled by default**.

Enable it globally:

```text
SW1(config)# ip igmp snooping querier 
Switch Querier function cannot be operationally enabled on some Vlans because the required conditions have not been met.
```

Notice the error message.

To send queries, **the switch needs an IP address** to use as the source of the Query packets.

As the following output shows, the IGMP Snooping Querier function isn't active on any VLANs, despite being configured:

```
SW1# show ip igmp snooping querier 
Vlan      IP Address               IGMP Version  Port              
-------------------------------------------------------------
```

Let's configure the VLAN 10 SVI:

```
SW1(config)# interface vlan 10
SW1(config-if)# ip address 10.10.10.1 255.255.255.0
```

Now it's enabled on the VLAN:

```
SW1# show ip igmp snooping querier 
Vlan      IP Address               IGMP Version  Port              
-------------------------------------------------------------
10        10.3.4.1                 v2            Switch                   
```

Once enabled globally, you can then disable it on specific VLANs with:
```
no ip igmp snooping vlan <vlan-id> querier
```

## IGMP Snooping Querier Source Address

The snooping querier needs a **source IPv4 address** for its IGMP Queries.

This is important because other devices use the Query's source address to identify the querier.

The source address can be explicitly configured:

```text
SW1(config)# ip igmp snooping querier address 10.10.10.1
```

Or configured for a particular VLAN:

```text
SW1(config)# ip igmp snooping vlan 10 querier address 10.10.10.1
```

Use a unique address appropriate for the VLAN, rather than an address already assigned to another device.

> The VLAN-specific command takes precedence over the global command.

### Automatic Source Address Selection

If an explicit source address is not configured, Catalyst IOS XE attempts to select an address automatically.

Depending on the configuration, it can use:

- An IP address configured for the VLAN's SVI
- A globally configured IGMP snooping querier address
- Another available IP address on the switch

For example, an SVI can supply the source address:

```text
SW1(config)# interface Vlan10
SW1(config-if)# ip address 10.3.4.1 255.255.255.0
SW1(config-if)# no shutdown
```

> **Important:** If the switch cannot find a usable source IPv4 address, it does not generate IGMP General Queries, 
> even if the snooping querier feature is enabled, as shown earlier.

Explicitly configuring the querier source address avoids ambiguity about which address IOS XE selects.

## Interaction with an Existing Multicast Router

A snooping querier is primarily intended for VLANs without a multicast router.

Suppose SW1 has its snooping querier enabled:

```text
                   SW1
              Snooping Querier
                 /      \
               PC1      PC2
```

SW1 periodically sends General Queries.

Now a multicast router is connected:

```text
                   R1
                    |
                   SW1
                 /     \
               PC1     PC2
```

According to some Cisco documentation, the snooping querier detects the presence of a multicast router and transitions to the **Non-Querier** state.

```text
Multicast router detected
          |
          v
SW1 stops generating periodic IGMP General Queries
          |
          v
R1 provides IGMP Queries
          |
          v
SW1 snoops Queries and Reports
```

However, **I could not replicate this behavior in IOS XE** (IOL and CAT9000v in CML).

The switch remained the Querier due to its lower IP address, even after connecting a router.

```
SW1# show ip igmp snooping querier vlan 10 detail
IP address               : 10.3.4.1
IGMP version             : v2
Port                     : Switch
Max response time        : 16s


Global IGMP switch querier status
--------------------------------------------------------
admin state                    : Enabled
admin version                  : 2
source IP address              : 0.0.0.0
query-interval (sec)           : 60
max-response-time (sec)        : 10
querier-timeout (sec)          : 120
tcn query count                : 2
tcn query interval (sec)       : 10

Vlan 10:   IGMP switch querier status
--------------------------------------------------------
elected querier is 10.3.4.5 (this switch querier)
--------------------------------------------------------
admin state                    : Enabled (state inherited)
admin version                  : 2
source IP address              : 10.3.4.1
query-interval (sec)           : 60
max-response-time (sec)        : 10
querier-timeout (sec)          : 120
tcn query count                : 2
tcn query interval (sec)       : 10
operational state              : Querier  <-- THIS SWITCH IS STILL THE QUERIER
operational version            : 2
tcn query pending count        : 0
```

Only after modifying the switch's IP address to `10.3.4.5` did it give up its role:

```
SW1# show ip igmp snooping querier vlan 10 detail
IP address               : 10.3.4.3  <-- MULTICAST ROUTER'S IP
IGMP version             : v2
Port                     : Et0/0
Max response time        : 2s


Global IGMP switch querier status
--------------------------------------------------------
admin state                    : Enabled
admin version                  : 2
source IP address              : 0.0.0.0
query-interval (sec)           : 60
max-response-time (sec)        : 10
querier-timeout (sec)          : 120
tcn query count                : 2
tcn query interval (sec)       : 10

Vlan 10:   IGMP switch querier status
--------------------------------------------------------
elected querier is 10.3.4.3 on port Et0/0
--------------------------------------------------------
admin state                    : Enabled (state inherited)
admin version                  : 2
source IP address              : 10.3.4.5  <-- THIS SWITCH'S IP
query-interval (sec)           : 60
max-response-time (sec)        : 10
querier-timeout (sec)          : 120
tcn query count                : 2
tcn query interval (sec)       : 10
operational state              : Non-Querier  <-- NO LONGER THE QUERIER
operational version            : 2
tcn query pending count        : 0
```

> Always verify the operational querier state with `show ip igmp snooping querier detail`
> rather than assuming the switch will relinquish the role when a multicast router is detected.

But really, you generally won't enable the IGMP Snooping Querier option on a LAN with a multicast router.

## IGMP Snooping Querier States

The snooping querier's administrative and operational states are different.

| State | Meaning |
|---|---|
| Administratively disabled | Feature is not enabled |
| Querier | Switch is actively generating Queries |
| Non-Querier | Another querier is active |
| Operationally disabled | Feature is configured but cannot operate |

On Catalyst IOS XE, the snooping querier becomes operationally disabled when:

- IGMP snooping is disabled in the VLAN.
- PIM is enabled on the corresponding VLAN SVI.

For example:

```text
SW1(config)# interface Vlan10
SW1(config-if)# ip pim sparse-mode
```

enables PIM on the SVI, allowing the switch to participate in Layer 3 multicast routing.

The separate Layer 2 snooping querier feature is then disabled for that VLAN.


## IGMP Version Support

The IOS XE IGMP snooping querier supports **IGMPv1, IGMPv2, and IGMPv3**, although the supported versions and functionality may vary by platform.

The default querier version is IGMPv2.

Configure the version globally:

```text
SW1(config)# ip igmp snooping querier version <1-3>
```

Or configure it per VLAN:

```text
SW1(config)# ip igmp snooping vlan 10 querier version <1-3>
```

### IGMPv3 Limitations

On IOS XE, the snooping querier can generate **IGMPv3 General Queries** to refresh multicast receiver membership information.

However, this does not necessarily provide the **full functionality of a Layer 3 IGMPv3 querier**.

For example, in my IOL lab testing:

- The snooping querier successfully generated IGMPv3 General Queries.
- Receivers responded with IGMPv3 Membership Reports containing source-specific information.
- The snooping querier did **not** generate Group-and-Source-Specific Queries in response to `BLOCK_OLD_SOURCES` Reports.

This is consistent with Catalyst 9000's **Basic IGMPv3 Snooping Support (BISS)**, which maintains Layer 2 forwarding state based on multicast groups `(G)` rather than individual source-group combinations `(S,G)`.

Therefore:

```text
IGMPv3 Snooping Querier
→ Can generate IGMPv3 General Queries
→ Refreshes Layer 2 multicast membership state
→ May lack source-specific Query functionality

Layer 3 IGMPv3 Querier
→ Generates General Queries
→ Generates Group-Specific Queries
→ Generates Group-and-Source-Specific Queries
→ Maintains source-specific receiver membership state
```

> **Important:** The absence of Group-and-Source-Specific Queries was observed in IOL lab testing, not established as a universal IOS XE limitation. For full IGMPv3 source-specific membership management, use a Layer 3 IGMPv3 querier.

Do not confuse `ip igmp snooping querier version 3` with `ip igmp version 3`. The former configures the Layer 2 snooping querier; the latter configures IGMP on a Layer 3 interface.

## IGMP Snooping Querier Timers

Catalyst IOS XE provides several configurable snooping querier parameters.

| Parameter | Default | Purpose |
|---|---:|---|
| Query Interval | 60 seconds | Frequency of General Queries |
| Max Response Time | 10 seconds | Maximum receiver response delay |
| Querier Expiry | 120 seconds | Time before the detected querier is considered expired |
| TCN Query Count | 2 | Number of additional Queries following a topology change |
| TCN Query Interval | 10 seconds | Interval between those Queries |

### Query Interval

Configure the interval between General Queries:

```text
SW1(config)# ip igmp snooping querier query-interval 30
```

Or per VLAN:

```text
SW1(config)# ip igmp snooping vlan 10 querier query-interval 30
```

The querier now sends periodic General Queries every 30 seconds while it is active.

This is a different command from the Layer 3 router configuration:

```text
R1(config-if)# ip igmp query-interval 30
```

### Maximum Response Time

Configure the maximum time hosts may wait before responding to General Queries:

```text
SW1(config)# ip igmp snooping querier max-response-time 5
```

Or per VLAN:

```text
SW1(config)# ip igmp snooping vlan 10 querier max-response-time 5
```

This setting applies to IGMPv2/v3 Queries. IGMPv1 Queries do not use a configurable Max Response Time.

### Querier Expiry Timer

Configure how long the switch waits before considering a detected querier expired:

```text
SW1(config)# ip igmp snooping querier timer expiry 180
```

Or per VLAN:

```text
SW1(config)# ip igmp snooping vlan 10 querier timer expiry 180
```

The supported range is 60–300 seconds.

Shorter values allow quicker takeover if the existing querier fails, but an excessively short timeout can cause unnecessary transitions.

Longer values reduce the chance of false failure detection but delay takeover.

## Topology Change Notification Queries

Spanning Tree topology changes can invalidate Layer 2 multicast forwarding information.

For example:

```text
STP topology change
        |
        v
Receiver may now be reachable through a different switch port
        |
        v
IGMP snooping state may need to be refreshed
```

Catalyst IOS XE supports additional IGMP Queries following **Topology Change Notifications (TCNs)**.

These Queries can accelerate the relearning of receiver memberships after the Layer 2 topology changes.

Two parameters control this behavior.

### TCN Query Count

Configures how many additional Queries are sent:

```text
SW1(config)# ip igmp snooping querier tcn query count 3
```

### TCN Query Interval

Configures the interval between those Queries:

```text
SW1(config)# ip igmp snooping querier tcn query interval 5
```

For example:

```text
TCN occurs
    |
    v
General Query #1
    |
    | 5 seconds
    v
General Query #2
    |
    | 5 seconds
    v
General Query #3
```

These settings help the switch recover its multicast membership information more quickly after topology changes.

Do not confuse them with the separate IGMP snooping TCN query-solicitation and multicast-flooding features.

## IGMP Snooping Querier vs Snooping Leave Processing

The snooping querier generates periodic General Queries to maintain membership information.

However, the switch's **IGMP snooping leave-processing mechanisms** are separate features.

For example:

```text
ip igmp snooping querier query-interval 30
```

controls the snooping querier's periodic General Queries.

Whereas:

```text
ip igmp snooping vlan 10 last-member-query-interval 500
```

controls last-member verification during IGMP snooping leave processing.

These are not interchangeable.

The snooping querier does not need to be active for all of the switch's local IGMP snooping leave-processing mechanisms to function.

## Verification

The primary verification command is:

```text
SW1# show ip igmp snooping querier
```

This displays the detected querier's IP address, IGMP version, and associated port for each VLAN.

For detailed information:

```text
SW1# show ip igmp snooping querier detail
```

Or for a particular VLAN:

```text
SW1# show ip igmp snooping querier vlan 10 detail
```

Example output (abridged):

```text
Vlan 10: IGMP querier status
-----------------------------------------
elected querier is 10.10.10.1

admin state          : Enabled
admin version        : 2
source IP address    : 10.10.10.1
query-interval (sec) : 60
max-response-time    : 10
querier-timeout (sec): 120
operational state    : Querier
operational version  : 2
```

The most important distinction is:

```text
admin state
→ whether the feature is configured

operational state
→ whether the feature is actually active
```

For example:

```text
admin state       : Enabled
operational state : Non-Querier
```

means that the feature is enabled but another device is currently providing Queries.

## Key Points

- The **IGMP snooping querier** generates IGMP Queries without requiring multicast routing.
- It is primarily useful in Layer 2 VLANs without a multicast router.
- On Catalyst 9000 IOS XE, the feature is **disabled by default**.
- It requires a usable source IPv4 address.
- Although Cisco documentation states that the snooping querier becomes a non-querier when it detects a multicast router, lab testing on IOL and CAT9000v showed that the switch can remain the querier if it has the lowest IP address.
- The snooping querier supports **IGMPv1, IGMPv2, and IGMPv3**.
- IGMPv3 snooping queriers can generate **General Queries** to maintain membership state, but may not support full IGMPv3 functionality, such as **Group-and-Source-Specific Queries** in response to membership changes.
- Global and per-VLAN configuration are supported.
- Use `show ip igmp snooping querier detail` to verify its operational state and elected querier.

