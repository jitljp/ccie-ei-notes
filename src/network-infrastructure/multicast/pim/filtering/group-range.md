# Multicast Group-Range Filtering

**Multicast group-range filtering** restricts which multicast groups or channels a router is allowed to process and forward.

Unlike an interface-based multicast boundary, group-range filtering is configured in **global configuration mode** and applies throughout the relevant multicast routing context.

Both IPv4 and IPv6 support this feature.

For groups or channels denied by the policy, the router can prevent:

- IGMP or MLD membership state creation.
- PIM multicast routing state creation.
- PIM Join processing and propagation.
- Multicast Routing Information Base (MRIB) and Multicast Forwarding Information Base (MFIB) state creation.
- Multicast data forwarding.

**Group-range filtering affects both the control plane and data plane.** It is not simply an ACL that drops multicast packets after forwarding state has been created.

Cisco IOS XE uses the following commands:

| Address Family | Command |
|---|---|
| IPv4 | `ip multicast group-range` |
| IPv6 | `ipv6 multicast group-range` |

Both support global and VRF-specific configuration.

## Group-Range Filtering Operation

Consider the following network:

```text
          Multicast Sources
          S1          S2
           \          /
            \        /
                 R1
                  |
                 R2
               /    \
              /      \
            PC1      PC2
```

Suppose R2 is configured to allow one multicast group but deny another.

When R2 receives a membership report for an allowed group, it can create multicast listener state and initiate the appropriate PIM signaling.

For a denied group:

```text
IGMP/MLD Report
       |
       v
Group-Range ACL
       |
      DENY
       |
       v
No listener state
No multicast routing state
No PIM Join propagation
No multicast data forwarding
```

The important distinction is that group-range filtering **prevents the corresponding multicast state from being established**, rather than merely filtering data packets after the forwarding tree has been built.

Cisco documents this behavior for both IPv4 and IPv6.

## IPv4 Multicast Group-Range Filtering

The IPv4 command is:

```text
ip multicast [vrf <vrf-name>] group-range <acl>
```

It is entered in global configuration mode.

| Parameter | Description |
|---|---|
| `vrf <vrf-name>` | Optional VRF whose multicast traffic is restricted |
| `group-range` | Configures multicast group-range filtering |
| `<acl>` | Numbered or named IPv4 ACL identifying permitted and denied multicast groups or `(S,G)` channels |

Unlike `ip multicast boundary`, the command is not applied to an individual interface.

### ACL Matching

IOS XE supports two useful ACL matching modes for IPv4 group-range filtering:

| ACL type | Matches | Typical use |
|---|---|---|
| **Standard** | Multicast group address `(G)` | Permit or deny entire groups or group ranges |
| **Extended** | Multicast source `(S)` and destination group `(G)` | Permit or deny individual source-group channels |

For standard ACLs, the address field represents the **multicast group**, not the packet's unicast source.

For extended ACLs, the **source address field is `(S)`** and the **destination address field is `(G)`**. The source field does not identify the receiver or the host sending an IGMP Report.

For either ACL type:

- **Permit:** The matching group or channel is allowed by the group-range policy.
- **Deny:** The matching group or channel is blocked by the group-range policy.
- **Implicit deny:** Traffic not matching a permit entry is denied by the policy.

For example, a standard ACL can deny the entire `239.1.0.0/16` range:

```text
access-list 10 deny 239.1.0.0 0.0.255.255
access-list 10 permit any
```

An extended ACL can deny just one channel while allowing other sources for the same group:

```text
ip access-list extended MCAST-CHANNELS
 deny ip host 10.1.1.10 host 232.1.1.1
 permit ip any any
```

### Permit Only Specific IPv4 Multicast Groups

Suppose R1 should process only multicast groups within:

```text
239.1.0.0/16
```

Configure:

```text
R1(config)# ip access-list standard MCAST-GROUPS
R1(config-std-nacl)# permit 239.1.0.0 0.0.255.255
R1(config)# ip multicast group-range MCAST-GROUPS
```

The result is:

| Group | Result |
|---|---|
| `239.1.1.1` | Permitted |
| `239.1.2.2` | Permitted |
| `239.2.1.1` | Denied |
| `232.1.1.1` | Denied |

Because the ACL contains only one permit statement, all other groups are denied by the implicit deny.

This is a **whitelist-style** policy.

### Deny Specific IPv4 Multicast Groups

Suppose R1 should block `239.1.0.0/16` but allow other multicast groups.

Configure:

```text
R1(config)# ip access-list standard MCAST-GROUPS
R1(config-std-nacl)# deny 239.1.0.0 0.0.255.255
R1(config-std-nacl)# permit any
R1(config)# ip multicast group-range MCAST-GROUPS
```

The result is:

| Group | Result |
|---|---|
| `239.1.1.1` | Denied |
| `239.1.2.2` | Denied |
| `239.2.1.1` | Permitted |
| `232.1.1.1` | Permitted |

For the denied groups:

1. IGMP processing does not establish accepted listener state.
2. PIM does not establish the corresponding multicast forwarding state.
3. The router does not forward multicast data for the denied groups.

**The final `permit any` is essential** when the intention is to deny only a selected multicast group range.

### Reserved Multicast Control Groups

A restrictive `ip multicast group-range` ACL can affect **multicast control traffic**, but it does not block every multicast-based routing protocol in the same way.

The important distinction is between locally processed link-local routing protocol packets and multicast control groups used by protocols such as Auto-RP.

#### Link-Local Routing Protocol Traffic

The IPv4 `224.0.0.0/24` range is reserved for local-network control traffic. These packets are not forwarded by normal multicast routing.

In an IOS XE lab, the following extended ACL was applied globally:

```text
R3(config)# ip access-list extended ABC
R3(config-ext-nacl)# permit ip host 10.1.1.11 host 232.1.1.1
R3(config)# ip multicast group-range ABC
```

All other groups and channels were implicitly denied. Nevertheless, **PIM and OSPF adjacencies remained operational**.

| Multicast Address | Purpose | Observed Behavior |
|---|---|---|
| `224.0.0.5` | OSPF AllSPFRouters | OSPF adjacency remained FULL |
| `224.0.0.6` | OSPF AllDRouters | OSPF adjacency remained FULL |
| `224.0.0.13` | All PIM Routers | PIM neighbor adjacencies remained established |

This demonstrates that denying groups in the global ACL does **not necessarily stop locally processed link-local routing protocol packets**.

However, this does not establish an exemption for multicast control groups outside `224.0.0.0/24`.

#### Auto-RP Control Groups

Auto-RP uses two groups outside the link-local multicast range:

| Group | Purpose | Observed Behavior |
|---|---|---|
| `224.0.1.39` | RP-Announce | **Filtered by group-range ACL** |
| `224.0.1.40` | RP-Discovery | **Filtered by group-range ACL** |

**Both Auto-RP groups were verified to be subject to `ip multicast group-range` filtering in an IOS XE lab.**

When the corresponding ACL permit was removed, Auto-RP reception stopped and the previously learned RP mapping eventually expired.

The tests are described below.

### Auto-RP Discovery: Lab Verification

The test used this topology:

```text
Src --- R1 --- RP --- R3 --- R4 --- Rec1
              |      |
          C-RP + MA   Group-range ACL
         99.99.99.99
```

The router named `RP` acted as both the **Auto-RP Candidate RP** and **Mapping Agent**, using `99.99.99.99` as its RP address.

On RP:

```text
RP(config)# access-list 1 permit 239.0.0.0 0.255.255.255
RP(config)# ip pim send-rp-announce Loopback0 scope 16 group-list 1 interval 20
RP(config)# ip pim send-rp-discovery Loopback0 scope 16 interval 20
```

The relevant routers used PIM Sparse Mode with `ip pim autorp listener`.

#### Test 1: Permit RP-Discovery

On R3, the ACL included:

```text
R3(config)# ip access-list extended ABC
R3(config-ext-nacl)# 10 permit ip host 10.1.1.11 host 232.1.1.1
R3(config-ext-nacl)# 20 permit ip any host 239.1.1.1
R3(config-ext-nacl)# 30 permit ip any 224.0.1.0 0.0.0.255
R3(config-ext-nacl)# 40 deny ip any any
R3(config)# ip multicast group-range ABC
```

Entry 30 explicitly permits **both Auto-RP control groups**.

R3 successfully learned the RP mapping:

```text
R3# show ip pim rp mapping
PIM Group-to-RP Mappings
Group(s) 239.0.0.0/8
  RP 99.99.99.99 (?), v2v1
    Info source: 99.99.99.99 (?), elected via Auto-RP
```

**Note:** The ACL did not permit every group within the advertised `239.0.0.0/8` RP range, but R3 successfully learned the entire mapping.

Learning an RP mapping and authorizing multicast forwarding for every group covered by that mapping are separate operations.

#### Test 2: Deny RP-Discovery

Remove the permit for `224.0.1.0/24`, leaving the explicit deny to cover `224.0.1.40`:

```text
R3(config)# ip access-list extended ABC
R3(config-ext-nacl)# no 30
```

R3 immediately logged:

```text
Auto-RP(0): Closed socket for 224.0.1.40
```

The RP-Discovery received counter stopped increasing:

```text
R3# show ip pim autorp
PIM AutoRP Statistics: Sent/Received
  RP Announce: 0/0, RP Discovery: 0/22
```

The previously learned RP mapping did not disappear immediately. Its holdtime continued counting down until expiration:

```text
Auto-RP(0): Mapping (239.0.0.0/8, RP:99.99.99.99) expired
Auto-RP(0): Mapping for (239.0.0.0/8) deleted
```

Afterward, `show ip pim rp mapping` displayed an empty mapping table.

**Conclusion:** Denying `224.0.1.40` prevented R3 from continuing to receive Auto-RP Discovery messages. The existing mapping remained only until its holdtime expired.

### Auto-RP Announce: Lab Verification

To test `224.0.1.39` separately, the Auto-RP roles were changed:

```text
Src --- R1 --- RP --- R3 --- R4 --- Rec1
              |      |
             C-RP    MA
                      |
               Group-range ACL
```

- **RP** remained the Candidate RP, advertising `239.0.0.0/8`.
- **R3** became the Mapping Agent, responsible for receiving RP-Announce messages and distributing RP-Discovery messages.

The test explicitly permitted RP-Discovery (`224.0.1.40`) while independently permitting or denying RP-Announce (`224.0.1.39`).

#### Test 1: Permit RP-Announce

On R3, the relevant ACL entries were:

```text
R3(config)# ip access-list extended ABC
R3(config-ext-nacl)# 30 permit ip any host 224.0.1.39
R3(config-ext-nacl)# 35 permit ip any host 224.0.1.40
```

With the group-range ACL applied, R3 successfully received RP-Announce messages from `99.99.99.99`:

```text
Auto-RP(0): Received RP-announce packet of length 48,
            from 99.99.99.99, RP_cnt 1, ht 61
Auto-RP(0): Added with
            (239.0.0.0/8, RP:99.99.99.99)
```

R3 then generated and transmitted RP-Discovery messages:

```text
Auto-RP(0): Build RP-Discovery packet
Auto-RP(0): Build mapping
            (239.0.0.0/8, RP:99.99.99.99)
Auto-RP(0): Send RP-discovery packet ...
```

The statistics confirmed successful RP-Announce reception:

```text
R3# show ip pim autorp
PIM AutoRP Statistics: Sent/Received
  RP Announce: 0/1, RP Discovery: 2/22
```

R3's RP mapping table also confirmed that it was operating as the Mapping Agent:

```text
R3# show ip pim rp mapping
PIM Group-to-RP Mappings
This system is an RP-mapping agent
Group(s) 239.0.0.0/8
  RP 99.99.99.99 (?), v2v1
    Info source: 99.99.99.99 (?), elected via Auto-RP
```

#### Test 2: Deny RP-Announce

Next, remove only the permit for `224.0.1.39`:

```text
R3(config)# ip access-list extended ABC
R3(config-ext-nacl)# no 30
```

The permit for `224.0.1.40` remained configured.

After the change, the RP-Announce received counter stopped increasing:

```text
R3# show ip pim autorp
PIM AutoRP Statistics: Sent/Received
  RP Announce: 0/2, RP Discovery: 8/22
```

The existing RP mapping remained temporarily, but its expiration timer continued decreasing.

**Importantly, R3 continued generating RP-Discovery messages containing the cached mapping, even though it was no longer receiving new RP-Announce messages.**

For example:

```text
Auto-RP(0): Build RP-Discovery packet
Auto-RP(0): Build mapping
            (239.0.0.0/8, RP:99.99.99.99)
Auto-RP(0): Send RP-discovery packet ...
```

Eventually, the learned Candidate RP mapping expired:

```text
Auto-RP(0): Mapping (239.0.0.0/8, RP:99.99.99.99) expired
Auto-RP(0): Mapping for (239.0.0.0/8) deleted
```

The Mapping Agent then generated an RP-Discovery message containing **zero RP entries**:

```text
Auto-RP(0): Trigger to send empty discovery block
Auto-RP(0): Send RP-discovery packet of length 36
            ... (0 RP entries)
```

The RP mapping table was now empty.

This demonstrates that **the Mapping Agent can continue advertising previously learned RP mappings until they expire**, even after new RP-Announce reception has stopped.

#### Test 3: Restore RP-Announce

Finally, restore the permit for `224.0.1.39`:

```text
R3(config)# ip access-list extended ABC
R3(config-ext-nacl)# 30 permit ip any host 224.0.1.39
```

R3 again received an RP-Announce message:

```text
Auto-RP(0): Received RP-announce packet of length 48,
            from 99.99.99.99, RP_cnt 1, ht 61
Auto-RP(0): Added with
            (239.0.0.0/8, RP:99.99.99.99)
```

The RP mapping immediately reappeared:

```text
R3# show ip pim rp mapping
PIM Group-to-RP Mappings
This system is an RP-mapping agent
Group(s) 239.0.0.0/8
  RP 99.99.99.99 (?), v2v1
    Info source: 99.99.99.99 (?), elected via Auto-RP
```

**Conclusion:** Denying `224.0.1.39` prevented the Mapping Agent from continuing to receive RP-Announce messages. The mapping eventually expired, and restoring the permit allowed R3 to learn it again.

### Practical ACL Guidance

**Neither Auto-RP control group is exempt from `ip multicast group-range` filtering on the tested IOS XE image.**

For a restrictive ACL, explicitly permit the necessary Auto-RP groups:

```text
permit ip any host 224.0.1.39
permit ip any host 224.0.1.40
```

These entries match the **multicast destination addresses** used by Auto-RP messages.

This is distinct from `ip multicast boundary <acl> filter-autorp`, which additionally examines group-to-RP mapping information **inside** Auto-RP messages.

## IPv4 Source-Group Filtering

An **extended ACL** lets `ip multicast group-range` distinguish between two sources sending to the same multicast group.

Suppose R3 should accept traffic from `10.1.1.11` to `232.1.1.1`, but reject traffic from `10.1.1.10` to that same group:

```text
(10.1.1.10, 232.1.1.1)  Deny
(10.1.1.11, 232.1.1.1)  Permit
```

Configure:

```text
R3(config)# ip access-list extended ABC
R3(config-ext-nacl)# permit ip host 10.1.1.11 host 232.1.1.1
R3(config-ext-nacl)# deny ip host 10.1.1.10 host 232.1.1.1
R3(config-ext-nacl)# permit ip any any
R3(config-ext-nacl)# exit
R3(config)# ip multicast group-range ABC
```

| Channel | ACL result | Forwarding result |
|---|---|---|
| `(10.1.1.10, 232.1.1.1)` | Deny | Blocked |
| `(10.1.1.11, 232.1.1.1)` | Permit | Allowed |
| Other channels | Permit | Not blocked by this ACL |

The explicit permit for `10.1.1.11` is not strictly necessary because the final `permit ip any any` also permits that channel. It is included to make the intended source-specific policy clear.

### Lab Verification

This behavior was verified using **two source addresses on the same physical interface**:

- `10.1.1.10`: Primary IPv4 address of the source interface.
- `10.1.1.11`: Secondary IPv4 address of that interface.
- `232.1.1.1`: SSM group requested by a receiver interested in **both** channels.

With the group-range ACL applied:

```text
Src# ping 232.1.1.1 repeat 10 source 10.1.1.11
```

The permitted source received multicast ping replies, whereas traffic sourced from `10.1.1.10` received no replies.

To isolate the filter as the cause, remove the group-range policy:

```text
R3(config)# no ip multicast group-range ABC
```

The previously blocked source (`10.1.1.10`) now receives replies. Reapply the policy:

```text
R3(config)# ip multicast group-range ABC
```

Traffic from `10.1.1.10` is blocked again, while `10.1.1.11` remains permitted.

**The remove/reapply test confirms that IOS XE evaluated the source address in the extended ACL, rather than merely filtering the multicast group as a whole.** Since both source addresses use the same interface and the receiver requested both channels, the results also rule out different source-facing interfaces or missing receiver interest as the explanation.

> **CCIE note:** The extended ACL's `ip` protocol selector is the appropriate general choice when filtering a complete `(S,G)` channel. A protocol-specific ACL entry should not automatically be assumed to block all multicast protocol operations associated with that channel.

## IPv4 VRF-Specific Group-Range Filtering

IPv4 group-range filtering also supports VRF-specific configuration, including policies referencing extended ACLs when supported by the platform.

The syntax is:

```text
ip multicast vrf <vrf-name> group-range <acl>
```

For example, suppose R1 contains two VRFs:

```text
BLUE
RED
```

Each VRF should permit a different multicast group range.

Configure the ACLs:

```text
R1(config)# ip access-list standard BLUE-GROUPS
R1(config-std-nacl)# permit 239.1.0.0 0.0.255.255
R1(config)# ip access-list standard RED-GROUPS
R1(config-std-nacl)# permit 239.2.0.0 0.0.255.255
```

Apply the policies:

```text
R1(config)# ip multicast vrf BLUE group-range BLUE-GROUPS
R1(config)# ip multicast vrf RED group-range RED-GROUPS
```

The result is:

| VRF | Permitted Group Range |
|---|---|
| BLUE | `239.1.0.0/16` |
| RED | `239.2.0.0/16` |

Each policy controls multicast processing in its corresponding VRF.

The ACLs themselves remain ordinary named IPv4 ACLs; the VRF association is made by the `ip multicast vrf ... group-range` command.

> **Important:** A group-range policy does not enable multicast routing. The relevant multicast routing and PIM configurations must still be present.

## IPv6 Multicast Group-Range Filtering

IPv6 supports the same overall concept, but uses IPv6 ACLs.

The command is:

```text
ipv6 multicast [vrf <vrf-name>] group-range <acl-name>
```

It is configured in global configuration mode.

| Parameter | Description |
|---|---|
| `vrf <vrf-name>` | Optional VRF whose multicast processing is restricted |
| `group-range` | Enables multicast group/channel filtering |
| `<acl-name>` | Optional IPv6 ACL identifying permitted and denied groups/channels |

### IPv6 ACL Matching

IPv6 ACLs use source and destination IPv6 address fields.

For multicast group-range filtering:

| ACL Field | Multicast Meaning |
|---|---|
| Source IPv6 address | Multicast source `(S)` |
| Destination IPv6 address | Multicast group `(G)` |

This allows IPv6 group-range filtering to match either entire multicast groups or individual `(S,G)` channels.

For example:

```text
permit ipv6 any host FF15::101
```

Permits the multicast group `FF15::101` regardless of source.

By contrast:

```text
permit ipv6 host 2001:DB8:1::10 host FF3E::101
```

Permits the specified `(S,G)` channel.

> **Important:** The ACL source field represents the multicast source, not the IPv6 address of the receiver sending an MLD Report.

## IPv6 Group Filtering Configuration

### Permit Only Specific Groups

Suppose R1 should permit only multicast groups matching:

```text
FF15::/32
```

Configure:

```text
R1(config)# ipv6 access-list MCAST-GROUPS
R1(config-ipv6-acl)# permit ipv6 any FF15::/32
R1(config)# ipv6 multicast group-range MCAST-GROUPS
```

The result is:

| Group | Result |
|---|---|
| `FF15::101` | Permitted |
| `FF15::102` | Permitted |
| `FF15:1::101` | Denied |
| `FF18::101` | Denied |

The ACL's implicit deny rejects groups outside the permitted prefix.

**`FF15::/32` does not represent the entire IPv6 Site-Local multicast scope.**

The `/32` prefix matches only addresses whose first 32 bits are `FF15:0000`.

The IPv6 multicast scope is identified by the 4-bit scope field, independently of an ordinary IPv6 prefix length.

### Deny Specific Groups

Suppose R1 should deny `FF15::101` but permit other groups.

Configure:

```text
R1(config)# ipv6 access-list MCAST-GROUPS
R1(config-ipv6-acl)# deny ipv6 any host FF15::101
R1(config-ipv6-acl)# permit ipv6 any any
R1(config)# ipv6 multicast group-range MCAST-GROUPS
```

The result is:

| Group | Result |
|---|---|
| `FF15::101` | Denied |
| `FF15::102` | Permitted |
| `FF18::101` | Permitted |

For denied groups, IOS XE prevents corresponding MLD and multicast routing state creation and blocks multicast data forwarding.

## IPv6 Source-Group Filtering

IPv6 group-range filtering can also identify individual **`(S,G)` channels**.

Suppose two sources transmit to the same SSM group:

```text
S1 = 2001:DB8:1::10
S2 = 2001:DB8:1::20
G  = FF3E::101
```

The channels are:

```text
(2001:DB8:1::10, FF3E::101)
(2001:DB8:1::20, FF3E::101)
```

The goal is to deny the first channel while permitting the second.

Configure:

```text
R1(config)# ipv6 access-list MCAST-CHANNELS
R1(config-ipv6-acl)# deny ipv6 host 2001:DB8:1::10 host FF3E::101
R1(config-ipv6-acl)# permit ipv6 any any
R1(config)# ipv6 multicast group-range MCAST-CHANNELS
```

The intended authorization policy is:

| Channel | Result |
|---|---|
| `(2001:DB8:1::10, FF3E::101)` | Denied |
| `(2001:DB8:1::20, FF3E::101)` | Permitted |
| `(2001:DB8:1::10, FF3E::102)` | Permitted |

This allows selective channel authorization without denying the entire multicast group.

### Group-Level and Source-Level Checks

Multicast protocols may perform both group-level and source-specific authorization checks.

For example, MLDv2 can perform a preliminary group-level check before evaluating individual sources.

If the group-level check is denied, the router may reject the membership before the individual `(S,G)` checks are reached.

Therefore, when building a restrictive whitelist for specific channels, explicitly account for group-level authorization where required.

For example:

```text
R1(config)# ipv6 access-list MCAST-CHANNELS
R1(config-ipv6-acl)# permit ipv6 host :: host FF3E::101
R1(config-ipv6-acl)# permit ipv6 host 2001:DB8:1::10 host FF3E::101
```

The first entry permits an MLD-style preliminary `(0,G)` check.

The second permits the specific `(S,G)` channel.

However, this example should not be treated as a universal guarantee that every PIM or multicast forwarding authorization check will use exactly the same lookup sequence.

**The ACL must be evaluated according to the protocol operation being authorized.**

See [MLD Filtering and Limits](../../layer2/mld/filtering.md) for the detailed MLDv2 ACL evaluation behavior.

## IPv6 VRF-Specific Filtering

IPv6 also supports VRF-specific group-range policies.

The syntax is:

```text
ipv6 multicast vrf <vrf-name> group-range <acl-name>
```

For example:

```text
R1(config)# ipv6 access-list BLUE-GROUPS
R1(config-ipv6-acl)# permit ipv6 any FF15::/32
R1(config)# ipv6 access-list RED-GROUPS
R1(config-ipv6-acl)# permit ipv6 any FF18::/32
```

Apply the policies:

```text
R1(config)# ipv6 multicast vrf BLUE group-range BLUE-GROUPS
R1(config)# ipv6 multicast vrf RED group-range RED-GROUPS
```

The result is:

| VRF | Permitted Group Range |
|---|---|
| BLUE | `FF15::/32` |
| RED | `FF18::/32` |

Each VRF has its own multicast authorization policy.

Multicast routing must also be enabled within the relevant VRF.

## Group-Range Filtering vs. Multicast Boundaries

Both features can restrict multicast processing, but their configuration scopes differ.

| Feature | Group-Range Filtering | Multicast Boundary |
|---|---|---|
| Configuration mode | Global | Interface |
| Applicable interfaces | Router-wide within the selected routing context | Configured interface |
| Filters multicast data | Yes | Yes |
| Affects multicast state creation | Yes | Yes |
| IPv4 ACL filtering | Standard `(G)` and extended `(S,G)` ACLs (lab-verified) | Standard and extended ACL capabilities |
| IPv6 ACL filtering | IPv6 ACL | Scope-based interface boundary |
| VRF-specific configuration | Yes | Through the interface's VRF |
| Primary use | Router-wide multicast authorization | Control multicast forwarding across a boundary |

For example, if a router should never process multicast group `239.1.1.1` in a particular routing context, a group-range policy is appropriate.

If the group should be permitted within the network but prevented from crossing a particular interface, an interface multicast boundary is more appropriate.

### Group-Range Filtering vs. IGMP/MLD Filtering

IGMP and MLD access-group commands primarily control which listener memberships are accepted.

Group-range filtering is broader.

| Function | IGMP/MLD Access-Group | Multicast Group-Range |
|---|---|---|
| Controls receiver membership admission | Yes | Yes |
| Controls multicast routing state | Indirectly through accepted memberships | Directly |
| Controls PIM state creation | Not directly | Yes |
| Blocks multicast data forwarding | Not directly | Yes |
| Scope | Interface | Global or VRF |

**Rejecting a membership report is not necessarily equivalent to prohibiting the group throughout the router.**

Other receivers, static forwarding state, or other multicast operations may still create forwarding interest for a group rejected by a particular IGMP/MLD access-group.

Group-range filtering provides a broader multicast authorization mechanism.

## Key Points

- **IPv4 and IPv6 both support multicast group-range filtering.**
- `ip multicast group-range` provides IPv4 multicast **group and source-group channel** filtering.
- `ipv6 multicast group-range` provides IPv6 multicast group/channel filtering.
- Both commands are configured in **global configuration mode**, not on individual interfaces.
- Both support VRF-specific configuration.
- Group-range filtering affects **both the control plane and data plane**.
- For denied groups, the router prevents associated IGMP/MLD state, PIM state, MRIB/MFIB state, and multicast data forwarding.
- IPv4 group-range filtering supports **standard `(G)` ACLs** and, as verified in an IOS XE lab, **extended `(S,G)` ACLs**.
- IPv4 extended ACLs use the source field for `(S)` and the destination field for `(G)`; this was confirmed by blocking only one of two sources for `232.1.1.1`.
- Removing and reapplying the IPv4 group-range policy restored and then blocked the denied source, confirming the filter's effect.
- IPv6 group-range filtering supports IPv6 ACL matching of groups and `(S,G)` channels.
- Link-local OSPF (`224.0.0.5/.6`) and PIM Hello (`224.0.0.13`) traffic continued operating despite the restrictive group-range ACL in an IOS XE lab.
- Both Auto-RP RP-Announce (`224.0.1.39`) and RP-Discovery (`224.0.1.40`) were filtered in IOS XE testing. Explicitly permit these groups when Auto-RP is required.
- Group-range filtering provides broader protection than interface-level IGMP/MLD membership filtering.
