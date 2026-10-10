# IGMP Maximum Groups and Throttling

**IGMP throttling** limits the number of IPv4 multicast groups that a Layer 2 interface can dynamically join through IGMP snooping.

IGMP snooping normally learns a port as a listener whenever it receives a valid IGMP Membership Report. Without a configured limit, the switch does not impose a per-port maximum through the IGMP throttling feature.

For example:

```text
                        R1
                   IGMP Querier
                         |
                       E0/0
                         |
                        SW1
                    VLAN 10
                  /          \
               E0/1          E0/2
                |              |
               PC1            PC2

            Limit: 2       No limit
```

Suppose PC1 requests three multicast channels:

```text
239.1.1.1 → Channel A
239.1.1.2 → Channel B
239.1.1.3 → Channel C
```

With a maximum of **two groups** configured on Ethernet0/1, SW1 can learn only two dynamic group memberships on that port at a time. What happens when PC1 requests a third group depends on the **throttling action**.

| Action | Behavior when the limit is reached |
|---|---|
| `deny` (default) | Drop the new IGMP join Report; do not learn the additional group on the port |
| `replace` | Replace a randomly selected existing group entry on the port with the newly requested group |

**IGMP throttling limits membership state, not the rate of IGMP packets.** It is not an IGMP control-plane policing or packets-per-second feature.

## Configuring the Maximum Number of Groups

Configure the limit under the Layer 2 interface:

```text
SW1(config)# interface Ethernet0/1
SW1(config-if)# switchport mode access
SW1(config-if)# switchport access vlan 10
SW1(config-if)# ip igmp max-groups 2
```

Syntax:

```text
ip igmp max-groups <number>
```

Cisco documents the numeric range as **0 to 4294967294** on Catalyst 9000 IOS XE. The default is **no maximum configured**—not a fixed number of groups.

With the example above:

```text
Ethernet0/1
  Maximum groups: 2
  Throttling action: deny (default)
```

You do **not** need to configure `action deny` explicitly to obtain the default behavior.

### What Exactly Is Counted?

The restriction concerns the number of **distinct dynamically learned multicast group memberships associated with the Layer 2 interface**, rather than the number of IGMP Reports or individual hosts.

For example:

```text
VLAN 10

239.1.1.1 → Ethernet0/1
239.1.1.2 → Ethernet0/1

Ethernet0/1 group count = 2
```

A third, different group would exceed a limit of two. Repeated Reports for a group already learned on the port do not represent a *third distinct group*.

Likewise, a single port may lead to multiple receivers:

```text
                        SW1
                         |
                       E0/1
                         |
                        SW2
                      /     \
                    PC1     PC2
```

If both PCs join `239.1.1.1`, SW1 still needs only one listener-port association for that group on Ethernet0/1:

```text
239.1.1.1 → Ethernet0/1

One distinct group on the port,
even if multiple receivers exist behind it
```

The limit is **per configured Layer 2 interface**. It is not a switch-wide maximum on the number of receivers or a global limit on all multicast groups in a VLAN.

## Default Throttling Action: Deny

When the interface has reached its configured maximum, `deny` rejects a new join Report that would require another group entry.

Consider Ethernet0/1 with a limit of two:

```text
SW1(config)# interface Ethernet0/1
SW1(config-if)# ip igmp max-groups 2
```

Initially, the port has no dynamic memberships:

```text
Ethernet0/1

Current groups: 0 / 2
```

### Step 1: PC1 Joins the First Group

PC1 sends an IGMP Membership Report for `239.1.1.1`:

```text
PC1
 |
 | IGMP Report: 239.1.1.1
 v
SW1 Ethernet0/1
 |
 v
Current count = 0
Maximum       = 2
 |
 v
Accept membership
```

The snooping state becomes:

```text
239.1.1.1 → Ethernet0/1

Current groups: 1 / 2
```

### Step 2: PC1 Joins the Second Group

PC1 sends a Report for `239.1.1.2`:

```text
Current count = 1
Maximum       = 2
        |
        v
Accept membership
```

SW1 now knows:

```text
239.1.1.1 → Ethernet0/1
239.1.1.2 → Ethernet0/1

Current groups: 2 / 2
```

The port has reached the limit, but **its existing groups are not removed merely because the limit has been reached**.

### Step 3: PC1 Attempts a Third Group

PC1 sends a Report for `239.1.1.3`:

```text
PC1
 |
 | IGMP Report: 239.1.1.3
 v
SW1 Ethernet0/1
 |
 v
Current count = 2
Maximum       = 2
 |
 v
Action = deny
 |
 v
Drop the new join Report
Do not add Ethernet0/1 to 239.1.1.3
```

The existing membership state remains:

```text
239.1.1.1 → Ethernet0/1
239.1.1.2 → Ethernet0/1

239.1.1.3 → Not learned on Ethernet0/1
```

You can confirm with `debug ip igmp max-groups`:

```
SW1#debug ip igmp max-groups 
event debugging is on
SW1#
*Oct 10 01:03:07.216: IGMPTHROTTLE: group 239.1.1.1 added to Et0/1: count(1) limit(2)
*Oct 10 01:03:07.717: IGMPTHROTTLE: Et0/1 already a member of group 239.1.1.1
*Oct 10 01:03:16.081: IGMPTHROTTLE: group 239.1.1.2 added to Et0/1: count(2) limit(2)
*Oct 10 01:03:16.517: IGMPTHROTTLE: Et0/1 already a member of group 239.1.1.2
*Oct 10 01:03:31.209: IGMPTHROTTLE: limit exceeded adding 239.1.1.3 to Et0/1: count(2) limit(2)
*Oct 10 01:03:31.517: IGMPTHROTTLE: limit exceeded adding 239.1.1.3 to Et0/1: count(2) limit(2)

SW1# show ip igmp snooping groups 
Flags: I -- IGMP snooping, S -- Static, P -- PIM snooping, A -- ASM mode
       E -- EVPN sync

Vlan      Group/source             Type        Version     Port List
-----------------------------------------------------------------------
10        224.0.1.40               I           v3          Et0/0 
10        239.1.1.1                I           v3          Et0/1
10        239.1.1.2                I           v3          Et0/1
```

SW1 learned `239.1.1.1` and `239.1.1.2` on `E0/1`, but denied `239.1.1.3`.

The `deny` action is **not a blanket denial of IGMP membership**. It comes into play when the configured limit is already occupied and a new group is requested.

Explicit configuration is possible, but normally unnecessary:

```text
SW1(config-if)# ip igmp max-groups action deny
```

## Alternative Throttling Action: Replace

Instead of denying an additional group, IOS XE can replace one of the existing dynamic group entries with the newly requested group.

Configure:

```text
SW1(config)# interface Ethernet0/1
SW1(config-if)# ip igmp max-groups 2
SW1(config-if)# ip igmp max-groups action replace
```

The key command is:

```text
ip igmp max-groups action replace
```

Suppose Ethernet0/1 has already learned:

```text
239.1.1.1 → Ethernet0/1
239.1.1.2 → Ethernet0/1

Current groups: 2 / 2
```

PC1 now requests `239.1.1.3`:

```text
PC1
 |
 | IGMP Report: 239.1.1.3
 v
SW1 Ethernet0/1
 |
 v
Maximum already reached
 |
 v
Action = replace
 |
 v
Remove one randomly selected group entry
 |
 v
Learn 239.1.1.3 on Ethernet0/1
```

One **possible** result is:

```text
Before:
239.1.1.1 → Ethernet0/1
239.1.1.2 → Ethernet0/1

New join:
239.1.1.3

After (example):
239.1.1.1 → Ethernet0/1
239.1.1.3 → Ethernet0/1

239.1.1.2 → Removed from Ethernet0/1
```

However, IOS XE could instead choose `239.1.1.1` for removal.

According to Cisco documentation, **the replacement is random.** It is not necessarily the oldest group, the first group learned, or the group that has been inactive longest.

> In my testing using IOL in CML, the switch always removes the **most recently added group**; it's not random.
> However, Cisco documentation states it is random, so always verify before using `action random`.

### Replacement Can Interrupt Existing Streams

The replaced group might still have active receivers.

For example:

```text
                  SW1
                   |
                 E0/1
                   |
                  SW2
                 /   \
               PC1   PC2

PC1 wants 239.1.1.1
PC2 wants 239.1.1.2
```

If a new membership request causes `239.1.1.1` to be replaced, traffic for that group may stop being forwarded through Ethernet0/1, even though PC1 still wants it.

Subsequent membership signaling can also cause membership state to change again. Do not treat `replace` as a guarantee of uninterrupted access to the most recently joined channels.

Also, `replace` can result in an infinite loop of groups being added and replaced, like in the following `debug` output:

```
*Oct 10 01:06:00.866: IGMPTHROTTLE: limit exceeded adding 239.1.1.3 to Et0/1: count(2) limit(2) a group will be deleted
*Oct 10 01:06:00.866: IGMPTHROTTLE: deleting 239.1.1.2 from Et0/1
*Oct 10 01:06:00.866: IGMPTHROTTLE: group 239.1.1.2 removed from Et0/1: count(1) limit(2)
*Oct 10 01:06:00.866: IGMPTHROTTLE: Et0/1 not a member of group 239.1.1.2
*Oct 10 01:06:01.617: IGMPTHROTTLE: limit exceeded adding 239.1.1.2 to Et0/1: count(2) limit(2) a group will be deleted
*Oct 10 01:06:01.617: IGMPTHROTTLE: deleting 239.1.1.3 from Et0/1
*Oct 10 01:06:01.617: IGMPTHROTTLE: group 239.1.1.3 removed from Et0/1: count(1) limit(2)
*Oct 10 01:06:01.618: IGMPTHROTTLE: Et0/1 not a member of group 239.1.1.3
*Oct 10 01:06:02.117: IGMPTHROTTLE: limit exceeded adding 239.1.1.3 to Et0/1: count(2) limit(2) a group will be deleted
*Oct 10 01:06:02.117: IGMPTHROTTLE: deleting 239.1.1.2 from Et0/1
*Oct 10 01:06:02.117: IGMPTHROTTLE: group 239.1.1.2 removed from Et0/1: count(1) limit(2)
*Oct 10 01:06:02.117: IGMPTHROTTLE: Et0/1 not a member of group 239.1.1.2
*Oct 10 01:06:02.418: IGMPTHROTTLE: limit exceeded adding 239.1.1.2 to Et0/1: count(2) limit(2) a group will be deleted
*Oct 10 01:06:02.418: IGMPTHROTTLE: deleting 239.1.1.3 from Et0/1
etc...
```

## Deny vs. Replace

With a limit of two and two memberships already learned:

| Event | `deny` | `replace` |
|---|---|---|
| A third group is requested | New join rejected | New join admitted by evicting an existing group |
| Existing group entries | Preserved | One may be removed |
| Risk to active receivers | New receiver cannot join its requested group | A previously received stream may be interrupted |
| Default action | Yes | No |

Both actions maintain the configured maximum during normal admission of new dynamic group memberships.

## A Port Limit Does Not Block the Group Everywhere

IGMP throttling is applied per Layer 2 interface.

Consider:

```text
                         R1
                          |
                         SW1
                        VLAN 10
                     /          \
                   E0/1        E0/2
                    |            |
                   PC1          PC2

                Limit 2       No limit
```

Suppose Ethernet0/1 already belongs to two groups and then PC1 tries to join `239.1.1.3`. The `deny` action prevents SW1 from adding **Ethernet0/1** to that group.

PC2 may still join the same group through Ethernet0/2:

```text
VLAN 10

239.1.1.1 → Ethernet0/1
239.1.1.2 → Ethernet0/1
239.1.1.3 → Ethernet0/2
```

Therefore, the group itself can still exist in the VLAN's IGMP snooping table.

The relevant question when checking throttling is:

> Is the restricted **port** present in the listener-port list for the group?

Do not conclude that throttling failed merely because the denied group appears elsewhere in the snooping table.

## Multiple Receivers Behind One Port

A Layer 2 switch port can represent multiple downstream receivers, especially when it connects to another switch.

```text
                         SW1
                          |
                        E0/1
                          |
                         SW2
                       /     \
                     PC1     PC2
```

Suppose Ethernet0/1 has a maximum of two groups:

```text
PC1 joins 239.1.1.1 → One group entry
PC2 joins 239.1.1.1 → Same group entry
PC2 joins 239.1.1.2 → Second group entry
PC1 joins 239.1.1.3 → Third group requested
```

With `deny`, the last request cannot create a third dynamic group association on Ethernet0/1.

This illustrates why **IGMP throttling is a per-port group limit, not a per-host subscription limit**.

It also explains why a restrictive maximum on an uplink serving many receivers can have a much greater impact than the same limit on an individual access port.

## Combining IGMP Profiles and Throttling

IGMP profiles and IGMP throttling control different aspects of multicast membership:

```text
IGMP profile
→ WHICH groups may be joined?

IGMP maximum groups
→ HOW MANY groups may be learned?
```

They can be configured on the same supported Layer 2 access port.

For example:

```text
SW1(config)# ip igmp profile 10
SW1(config-igmp-profile)# permit
SW1(config-igmp-profile)# range 239.1.1.1 239.1.1.100
SW1(config-igmp-profile)# exit
!
SW1(config)# interface Ethernet0/1
SW1(config-if)# switchport mode access
SW1(config-if)# switchport access vlan 10
SW1(config-if)# ip igmp filter 10
SW1(config-if)# ip igmp max-groups 2
```

The intended admission policy is:

```text
Group must be permitted by profile 10
                AND
The port must have room under its group limit
```

For example, assume Ethernet0/1 has already joined `239.1.1.1` and `239.1.1.2`:

| New request | Profile 10 | Group limit | Result |
|---|---|---|---|
| `239.1.1.3` | Permits group | Already at 2/2 | Denied by throttling |
| `239.2.2.2` | Denies group | Already at 2/2 | Denied by policy |

The example describes the combined policy outcome; it does not imply a fixed internal processing order between profile filtering and throttling.

> Catalyst 9000 IOS XE documentation states that IGMPv3 join/leave messages are not supported by the **IGMP profile filtering** feature. Do not assume that combining an IGMP profile and a group limit provides IGMPv3 `(S,G)` source-specific admission control.

## Existing Memberships, Aging, and Configuration Changes

An IGMP group limit does not permanently reserve slots for particular addresses.

Dynamic IGMP snooping membership can be removed through normal processes, including leaving a group and membership aging.

For example, with `deny` and a limit of two:

```text
Initially:
239.1.1.1 → Ethernet0/1
239.1.1.2 → Ethernet0/1

Count: 2/2
```

If membership in `239.1.1.1` ends and its snooping state is removed:

```text
239.1.1.2 → Ethernet0/1

Count: 1/2
```

A subsequent request for `239.1.1.3` can then be admitted:

```text
239.1.1.2 → Ethernet0/1
239.1.1.3 → Ethernet0/1

Count: 2/2
```

### Changing the Throttling Configuration After Groups Are Learned

Cisco specifically warns about configuring a limit or throttling action **after** an interface has already learned multicast entries.

- With `deny`, existing entries are not immediately removed merely because the action is selected; they can age out through normal snooping processes.
- With `replace`, previously learned forwarding-table entries can be removed when the throttling configuration is applied, in addition to random replacement behavior on subsequent over-limit joins.

The exact impact depends on the state and platform. To minimize disruption, configure the intended maximum and action **before receivers begin joining**, when possible.

## Interface Restrictions

On Catalyst 9000 IOS XE, the feature is for **Layer 2 interfaces**.

Cisco documents support for:

```text
Layer 2 physical switch ports
Logical Layer 2 EtherChannel interfaces (Port-channel)
```

It does **not** support setting these limits on:

```text
Routed physical interfaces
Switch Virtual Interfaces (SVIs)
Physical ports that are members of an EtherChannel
```

For example, on a supported Layer 2 port-channel, apply the policy to the **logical Port-channel**, not its individual Ethernet member interfaces:

```text
SW1(config)# interface Port-channel1
SW1(config-if)# ip igmp max-groups 10
```

This is different from `ip igmp limit`, which operates on the IGMP state of a **Layer 3 interface or router** and is covered in **IGMP Filtering on Routed Interfaces**.

## Verifying the Configuration

### Verify the Maximum and Action

The principal configuration verification command is:

```text
SW1# show running-config interface Ethernet0/1
```

For example:

```text
interface Ethernet0/1
 switchport mode access
 switchport access vlan 10
 ip igmp max-groups 2
 ip igmp max-groups action replace
```

This confirms both the configured limit and the non-default action.

If the interface has:

```text
ip igmp max-groups 2
```

but no explicit `ip igmp max-groups action` command, the default action is **`deny`**.

## IGMP Throttling vs. Other Multicast Limits

Do not confuse the following commands:

| Command | Scope | What it controls |
|---|---|---|
| `ip igmp filter <profile>` | Layer 2 interface | **Which** multicast groups may be dynamically learned |
| `ip igmp max-groups <number>` | Layer 2 interface | **How many** multicast groups may be dynamically learned |
| `ip igmp max-groups action {deny \| replace}` | Layer 2 interface | What happens when a **new group** would exceed the configured maximum |
| `ip igmp access-group <acl>` | Layer 3 interface | Which IGMP memberships a router accepts |
| `ip igmp limit <number>` | Layer 3 IGMP state | Limits IGMP membership state on a router (globally or per interface, as supported) |

The Layer 3 commands are covered in **IGMP Filtering on Routed Interfaces**.

## Key Points

- **IGMP throttling** restricts the number of dynamically learned multicast groups associated with a Layer 2 interface; it does not limit IGMP packets per second.
- Configure a limit with `ip igmp max-groups <number>`. The documented range is **0–4294967294**; the default is **no limit**.
- The throttling action applies when the configured maximum has been reached and an additional group is requested.
- **`deny` is the default action**: reject the new join without evicting the existing group entries.
- **`replace`** removes an existing group entry to admit the newly requested group; it can interrupt an active multicast stream.
- An action configured without a maximum has **no throttling effect**.
- The limit applies to a **Layer 2 interface**, not individually to every host behind that interface or globally to the entire VLAN.
- On supported platforms, apply the restriction to a logical Layer 2 Port-channel rather than its physical EtherChannel members; routed interfaces and SVIs do not support this command.
- Changing throttling settings after memberships have already been learned may affect existing entries, especially with `replace`.
- Use `show running-config interface` to verify the configured limit and action, and `show ip igmp snooping groups` to verify the resulting listener-port state.
- **`ip igmp max-groups` is not the same as `ip igmp limit`**; the latter is a Layer 3 IGMP state-limiting feature.
