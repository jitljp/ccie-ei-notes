# MLD Snooping

**MLD snooping** is the IPv6 equivalent of [IGMP Snooping](../snooping/igmp-snooping.md).

It allows a Layer 2 switch to examine **Multicast Listener Discovery (MLD)** messages and learn which ports lead toward IPv6 multicast listeners. The switch can then forward multicast data only where it is needed, rather than flooding it throughout the VLAN.

The main difference is the control protocol:

- **IPv4:** IGMP snooping examines IGMP messages (IP protocol 2).
- **IPv6:** MLD snooping examines MLD messages (ICMPv6, Next Header 58).

MLD snooping is a **Layer 2 feature**. The switch does not need to perform IPv6 multicast routing or run PIM to snoop MLD messages.

## Basic Operation

Consider a switch with two receiver-facing ports and an uplink to an IPv6 multicast router:

```text
                   R1
                    |
                  E0/0
                    |
                   SW1
                 /     \
              E0/1     E0/2
                |        |
               PC1      PC2
```

PC1 wants to receive traffic for `ff15::1234`; PC2 does not.

1. **MLD Query:** R1 sends an MLD General Query. SW1 forwards it through the VLAN and learns that Ethernet0/0 leads toward a multicast router.
2. **MLD Report:** PC1 sends a Report for `ff15::1234`. SW1 learns that Ethernet0/1 is a **listener port** for that group.
3. **Multicast forwarding:** When data for `ff15::1234` arrives, SW1 forwards it to Ethernet0/1, but not Ethernet0/2.

The learned state is conceptually:

```text
VLAN 10
Group ff15::1234
  Listener port: Ethernet0/1
  Mrouter port:  Ethernet0/0
```

This is the same fundamental operation as IGMP snooping, except that the group addresses and control messages are IPv6. Known multicast data is forwarded toward interested listener ports and appropriate mrouter ports.

IPv6 multicast packets use Ethernet destination addresses beginning with `33:33`, followed by the low-order 32 bits of the IPv6 multicast address. For example:

```text
IPv6 group: ff15::1234
Ethernet MAC: 33:33:00:00:12:34
```

This differs from the IPv4 multicast MAC prefix `01:00:5e`, but the purpose of snooping is the same.

> Normal Ethernet source-MAC learning cannot identify multicast listeners. MLD snooping learns their locations from MLD Reports and the ports on which those Reports arrive.

## MLD Versions and Layer 2 Forwarding

Catalyst IOS XE supports snooping of both MLD versions:

| MLD | IGMP equivalent | Relevant control messages |
|---|---|---|
| **MLDv1** | IGMPv2 | Queries, Reports, Done messages |
| **MLDv2** | IGMPv3 | Queries, Reports with INCLUDE/EXCLUDE source lists and state changes |

For example:

- An **MLDv1 Report** indicates interest in a multicast group `(G)`.
- An **MLDv2 Report** can express interest in specific `(S,G)` channels.

However, Catalyst 9000 uses **MLDv2 Basic Snooping Support (MBSS)**. Although the switch can interpret MLDv2 control messages, its Layer 2 forwarding decisions are based on the **destination IPv6 multicast group**, not both the source and group.

```text
Catalyst MLDv2 snooping forwarding:

(VLAN, G) -> listener ports

Not enhanced source-specific forwarding:

(VLAN, S, G) -> listener ports
```

This is directly comparable to **Basic IGMPv3 Snooping Support (BISS)** on Catalyst 9000 switches.

> Do not confuse MLDv2 source-filter information with Layer 2 source-specific forwarding. An IPv6 multicast router may maintain `(S,G)` routing state, but that does not mean the Layer 2 snooping table filters data using `S`.

## Listener Ports and Multicast Router Ports

MLD snooping maintains two important kinds of port information:

| Port type | Learned from | Purpose |
|---|---|---|
| **Listener port** | MLD Reports | Leads toward receivers interested in a group |
| **Multicast router (mrouter) port** | MLD Queries or IPv6 PIM messages | Leads toward an IPv6 multicast router |

As with IGMP snooping, a listener port may lead to **multiple receivers** behind another switch. A Report received on a port does not imply there is exactly one receiver there.

### Mrouter Port Discovery

IOS XE can dynamically discover IPv6 mrouter ports from MLD Queries and IPv6 PIM control traffic.

Dynamically learned mrouter ports age out if relevant control traffic stops. Cisco documents a default aging time of **5 minutes**. Statically configured mrouter ports do not age out.

Configure a static mrouter port for VLAN 10:

```text
SW1(config)# ipv6 mld snooping vlan 10 mrouter interface Ethernet0/0
```

Verify:

```text
SW1# show ipv6 mld snooping mrouter vlan 10
```

> MLD snooping learns IPv6 mrouter ports independently of IGMP snooping's IPv4 mrouter ports. Discovering an IPv4 multicast router does not automatically identify an IPv6 multicast router.

## MLD Queries and the Snooping Querier

Like IGMP snooping, MLD snooping relies on **periodic Queries** to refresh dynamic listener state:

```text
MLD General Query
       |
       v
Receivers send Reports
       |
       v
Snooping state is refreshed
```

Without periodic Queries, dynamic membership state can eventually expire, unless another mechanism maintains it.

An IPv6 multicast router normally provides the Queries, but on supported Catalyst IOS XE releases the switch can also act as an **MLD snooping querier** without enabling IPv6 multicast routing.

Enable the snooping querier globally:

```text
SW1(config)# ipv6 mld snooping querier
```

You can then disable it on specific VLANs if necessary:

```
SW1(config)# no ipv6 mld snooping vlan <vlan-id> querier
```

Verify the querier information:

```text
SW1# show ipv6 mld snooping querier        
Vlan      IP Address               MLD Version  Port              
-------------------------------------------------------------
10        FE80::10                 v2            Switch                   

SW1#show ipv6 mld snooping querier vlan 10
IP address               : FE80::10
MLD  version             : v2
Port                     : Switch
Max response time        : 10s
Query interval           : 125s
Robustness variable      : 2
```

The snooping querier is analogous to the [IGMP Snooping Querier](../igmp-querier/snooping-querier.md). It helps maintain listener information on a Layer 2 network that has no multicast router generating Queries.

**Operational note:** Cisco documents that some Catalyst snooping implementations do not process MLDv1 Reports until an IPv6 multicast router is detected in the VLAN. If Reports are visible in a capture but no listener entries appear, verify mrouter discovery and querier operation before troubleshooting the receiver.

## Listener Message Suppression

**MLD listener message suppression** is the IPv6 equivalent of IGMP snooping Report Suppression.

When multiple hosts report interest in the same group, a snooping switch can learn all the listener ports but avoid forwarding every duplicate Report upstream.

For MLDv1, Catalyst IOS XE forwards the first relevant Report toward multicast routers and suppresses subsequent duplicate Reports for the group. This is comparable to IGMPv2 snooping Report Suppression.

MLDv2 is different: its Reports contain source-filter information and are not treated as interchangeable group-only Reports.

The feature is shown as **Listener message suppression: Enabled** in the operational output on Catalyst IOS XE.

```
SW1#show ipv6 mld snooping vlan 10
Global MLD Snooping configuration:
-------------------------------------------
MLD snooping Oper State      : Enabled
MLDv2 snooping               : Enabled
Listener message suppression : Enabled
TCN solicit query            : Disabled
Robustness variable          : 2
Last listener query count    : 2
Last listener query interval : 1000
Check Hop-count=1            : No

Vlan 10:
--------
MLD snooping Admin State            : Enabled
MLD snooping Oper State             : Enabled
MLD immediate leave                 : Disabled
Explicit host tracking              : Enabled
Listener message suppression        : Enabled
Robustness variable                 : 2
Last listener query count           : 2
Last listener query interval        : 1000
Check Hop-count=1                   : Yes
Query Interval                      : 125
Max Response Time                   : 10000
```

Disable suppression globally:

```text
SW1(config)# no ipv6 mld snooping listener-message-suppression
```

Verify:

```
SW1#show ipv6 mld snooping vlan 10
Global MLD Snooping configuration:
-------------------------------------------
MLD snooping Oper State      : Enabled
MLDv2 snooping               : Enabled
Listener message suppression : Disabled
TCN solicit query            : Disabled
Robustness variable          : 2
Last listener query count    : 2
Last listener query interval : 1000
Check Hop-count=1            : No

Vlan 10:
--------
MLD snooping Admin State            : Enabled
MLD snooping Oper State             : Enabled
MLD immediate leave                 : Disabled
Explicit host tracking              : Enabled
Listener message suppression        : Enabled
Robustness variable                 : 2
Last listener query count           : 2
Last listener query interval        : 1000
Check Hop-count=1                   : Yes
Query Interval                      : 125
Max Response Time                   : 10000
```

> This is **switch-based** suppression. It is distinct from the host-based Report Suppression mechanism associated with MLDv1 and IGMPv2.

## Leave Processing and Immediate Leave

When an MLDv1 receiver leaves a group, it sends an **MLD Done** message, equivalent to an IGMPv2 Leave.

Without Immediate Leave, the snooping switch can send **Multicast-Address-Specific Queries (MASQs)** to determine whether another listener still exists behind the port.

```text
MLDv1 Done received
        |
        v
Send address-specific Queries
        |
        v
Other listeners respond?
   |                 |
  Yes                No
   |                 |
Keep port        Remove port
```

MLDv2 uses source-filter state-change Reports rather than a separate Done message. The same principle applies: a port should not be removed while another listener still needs the traffic.

### Immediate Leave

Enable MLDv1 Immediate Leave for VLAN 10:

```text
SW1(config)# ipv6 mld snooping vlan 10 immediate-leave
```

This removes a port's membership immediately when an MLDv1 Done message is received, without waiting for further Queries.

**Use Immediate Leave only when there is one receiver per switch port.** If multiple receivers are reachable behind the port, removing it immediately could interrupt another receiver.

Immediate Leave is **disabled by default**, as with IGMP snooping.

> This only applies to MLDv1, as MLDv2 doesn't use `Done` messages.

### Last-Listener Query Timers

IOS XE provides controls comparable to IGMP snooping's last-member timers:

```text
SW1(config)# ipv6 mld snooping last-listener-query-count 3
SW1(config)# ipv6 mld snooping last-listener-query-interval 500
```

Override them for a VLAN:

```text
SW1(config)# ipv6 mld snooping vlan 10 last-listener-query-count 3
SW1(config)# ipv6 mld snooping vlan 10 last-listener-query-interval 500
```

Defaults:

| Parameter | Default |
|---|---|
| Last-listener query count | 2 |
| Last-listener query interval | 1000 ms (1 second) |

The query count determines how many address-specific Queries are used in the leave-verification process; the interval controls the response timing.

## Explicit Host Tracking

Like IGMP snooping, MLD snooping can maintain **Explicit Host Tracking** information about individual listeners, rather than only a group-to-port forwarding entry.

For example:

```text
Group ff15::1234
  PC1 -> Ethernet0/1
  PC2 -> Ethernet0/1
```

Tracking individual listeners can help with leave processing and troubleshooting. MLDv2 adds per-host source-filter information, but **this does not change MBSS into source-specific Layer 2 forwarding**.

Catalyst IOS XE's `show ipv6 mld snooping` output includes an `Explicit host tracking` field. This is separate from the routed-interface command `ipv6 mld explicit-tracking`.

> MLD Snooping Explicit Tracking is **always enabled**, but you can effectively disable it with `ipv6 mld snooping explicit-tracking limit 0`

Verify:

```
SW1# show ipv6 mld snooping membership  <-- NOTHING IN EHT DATABASE

SW1# show ipv6 mld snooping address 
Flags: M -- MLD snooping, S -- Static, E -- EVPN sync

Vlan      Group/source             Type        Version     Port List
-----------------------------------------------------------------------
10        FF15::101                M           v2          Et0/1   
10        FF15::102                M           v2          Et0/1 
```

Although the feature still appears as `enabled`:

```
SW1# show ipv6 mld snooping vlan 10
Global MLD Snooping configuration:
-------------------------------------------
MLD snooping Oper State      : Enabled
MLDv2 snooping               : Enabled
Listener message suppression : Enabled
TCN solicit query            : Disabled
Robustness variable          : 2
Last listener query count    : 2
Last listener query interval : 1000
Check Hop-count=1            : No

Vlan 10:
--------
MLD snooping Admin State            : Enabled
MLD snooping Oper State             : Enabled
MLD immediate leave                 : Enabled
Explicit host tracking              : Enabled
Listener message suppression        : Enabled
Robustness variable                 : 2
Last listener query count           : 2
Last listener query interval        : 1000
Check Hop-count=1                   : Yes
Query Interval                      : 125
Max Response Time                   : 2500
```

## Enabling MLD Snooping

A significant default difference from IGMP snooping on Catalyst IOS XE is:

| Feature | Global default | Per-VLAN default |
|---|---|---|
| IGMP snooping | Enabled | Enabled |
| **MLD snooping** | **Disabled** | Enabled |

Therefore, even when the VLAN administrative state is enabled, its **operational state remains disabled** until MLD snooping is enabled globally.

Enable it:

```text
SW1(config)# ipv6 mld snooping
```

Explicitly enable it on a VLAN, if it has been disabled:

```text
SW1(config)# ipv6 mld snooping vlan 10
```

Disable snooping on only on a specific VLAN 10:

```text
SW1(config)# no ipv6 mld snooping vlan 10
```

Or disable it globally:

```text
SW1(config)# no ipv6 mld snooping
```

MLD snooping and IGMP snooping are independent and can run concurrently on the same VLAN.

## Static Listener Ports

A port can be configured as a static member of an IPv6 multicast group, without waiting for an MLD Report:

```text
SW1(config)# ipv6 mld snooping vlan 10 static ff15::1234 interface Ethernet0/1
```

This is the IPv6 counterpart of:

```text
SW1(config)# ip igmp snooping vlan 10 static 239.1.1.1 interface Ethernet0/1
```

A **static listener port** identifies where multicast data should be forwarded. A **static mrouter port** identifies where a multicast router is reachable. They are not interchangeable.

## Other Snooping Controls

### Robustness Variable

The MLD snooping robustness variable influences how many unanswered general Queries are tolerated before listener state is removed.

The global default is **2**. A VLAN-level value of `0` means that the global value is inherited.

```text
SW1(config)# ipv6 mld snooping robustness-variable 3
SW1(config)# ipv6 mld snooping vlan 10 robustness-variable 3
```

### Spanning-Tree Topology Changes

A spanning-tree topology change can invalidate learned multicast forwarding paths. As with IGMP snooping, temporary multicast flooding and query solicitation can help rebuild listener state.

Configure TCN-related behavior:

```text
SW1(config)# ipv6 mld snooping tcn query solicit
SW1(config)# ipv6 mld snooping tcn flood query count 3
```

The default TCN flood query count is **2**, and TCN query solicitation is **disabled by default**.

### Unknown Multicast

Unknown IPv6 multicast handling requires attention. Cisco documents that its MLD snooping implementation initially floods unknown IPv6 multicast data, but after discovering an IPv6 mrouter port, unknown multicast data is directed toward discovered mrouter ports.

> This differs from the typical IGMP snooping behavior of flooding unknown IPv4 multicast traffic throughout the VLAN, although forwarding behavior can be platform-dependent.

## Verification

| Command | Purpose |
|---|---|
| `show ipv6 mld snooping` | Global and VLAN administrative/operational state, defaults, and timers |
| `show ipv6 mld snooping vlan 10` | VLAN 10 snooping configuration |
| `show ipv6 mld snooping address vlan 10` | Learned and static IPv6 multicast groups and listener ports |
| `show ipv6 mld snooping address vlan 10 dynamic` | Dynamically learned groups |
| `show ipv6 mld snooping mrouter vlan 10` | Dynamic and static IPv6 multicast-router ports |
| `show ipv6 mld snooping querier vlan 10` | MLD querier information |

When troubleshooting, check **both the global and per-VLAN operational state**, then verify that the switch sees a querier/mrouter and has learned the expected listener ports.

## IGMP Snooping vs. MLD Snooping

| Feature | IGMP Snooping | MLD Snooping |
|---|---|---|
| IP version | IPv4 | IPv6 |
| Control protocol | IGMP (IP protocol 2) | MLD (ICMPv6) |
| Older version | IGMPv2 | MLDv1 |
| Source-aware version | IGMPv3 | MLDv2 |
| Catalyst 9000 Layer 2 forwarding | Group-based (BISS) | Group-based (MBSS) |
| Global default | Enabled | **Disabled** |
| Per-VLAN default | Enabled | Enabled |
| Listener and mrouter ports | Yes | Yes |
| Report suppression | IGMPv1/v2-style Reports | MLDv1 Reports |
| Immediate Leave | IGMPv2 Leaves | MLDv1 Done messages |
| Snooping querier | Supported | Supported |
| Static listener and mrouter ports | Supported | Supported |
| Layer 2 profile filtering / `max-groups` | Supported on applicable Catalyst platforms | No equivalent documented Catalyst 9000 IOS XE commands |

## Key Points

- **MLD snooping** is the IPv6 equivalent of IGMP snooping, learning multicast listener ports from MLD Reports.
- It operates at **Layer 2** and does not require IPv6 multicast routing or PIM on the switch.
- **MLDv1 corresponds to IGMPv2**; **MLDv2 corresponds to IGMPv3**.
- IPv6 multicast maps to Ethernet addresses beginning with **`33:33`**, rather than IPv4 multicast's `01:00:5e`.
- Catalyst 9000 **MBSS** forwards based on the destination multicast **group**, not `(S,G)`.
- MLD snooping is **globally disabled by default**, but enabled per VLAN by default.
- MLD Queries refresh listener state; mrouter ports are learned dynamically or configured statically.
- **Listener message suppression** reduces redundant MLDv1 Reports sent toward multicast routers.
- **Immediate Leave** removes an MLDv1 port without verification and is safe only with a single receiver per port.
- IOS XE supports static listener ports, a snooping querier, last-listener timers, robustness configuration, and TCN handling.

