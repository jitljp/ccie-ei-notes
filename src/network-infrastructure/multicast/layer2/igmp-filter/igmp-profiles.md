# IGMP Profiles and Filtering

**IGMP profiles** allow a Layer 2 switch to control which IPv4 multicast groups hosts connected to its ports can join.

Normally, **IGMP snooping** dynamically learns multicast group memberships from IGMP Membership Reports without restricting which groups hosts can join.

An IGMP profile introduces a **membership policy** that permits or denies particular multicast groups.

For example:

```text
                   R1
             IGMP Querier
                    |
                  E0/0
                    |
                   SW1
             /       |       \
          E0/1     E0/2     E0/3
           |         |         |
          PC1       PC2       PC3
```

Suppose the network provides three multicast streams:

```text
239.1.1.1 → Channel A
239.1.1.2 → Channel B
239.1.1.3 → Channel C
```

The administrator wants different membership policies for each port:

| Port | Policy |
|---|---|
| Ethernet0/1 | Permit only Channel A |
| Ethernet0/2 | Deny Channel C |
| Ethernet0/3 | No restrictions |

Without IGMP filtering, all three PCs could dynamically join any of the multicast groups.

With IGMP profiles, SW1 can apply different membership restrictions to each port.

## IGMP Profile Components

An IGMP profile consists of three main elements:

```text
IGMP Profile
    |
    +-- Profile number
    |
    +-- Action
    |     permit / deny
    |
    +-- Multicast group ranges
          single group
          or
          range of groups
```

For example:

```text
SW1(config)# ip igmp profile 10
SW1(config-igmp-profile)# permit
SW1(config-igmp-profile)# range 239.1.1.1
```

This creates profile 10, which permits membership in `239.1.1.1`.

The commands have the following roles:

| Command | Purpose |
|---|---|
| `ip igmp profile 10` | Creates IGMP profile 10 |
| `permit` | Permits membership in matching groups |
| `deny` | Denies membership in matching groups |
| `range` | Defines the multicast group addresses that the action applies to |

The action is configured **once for the entire profile**, not individually for each multicast group.

## Creating an IGMP Profile

IGMP profiles are created in global configuration mode:

```text
SW1(config)# ip igmp profile 10
SW1(config-igmp-profile)#
```

The command enters **IGMP profile configuration mode**.

The profile number can range from:

```text
1 to 4294967295
```

An IGMP profile does not take effect immediately after being created.

It must be associated with a Layer 2 switch port using:

```text
ip igmp filter <profile-number>
```

For example:

```text
SW1(config)# interface Ethernet0/1
SW1(config-if)# ip igmp filter 10
```

This applies profile 10 to Ethernet0/1.

The distinction is:

```text
ip igmp profile
→ Define a membership policy

ip igmp filter
→ Apply the policy to a port
```

## Permitting Multicast Groups

The `permit` action creates a **whitelist** of multicast groups.

For example:

```text
SW1(config)# ip igmp profile 10
SW1(config-igmp-profile)# permit
SW1(config-igmp-profile)# range 239.1.1.1
SW1(config-igmp-profile)# end
SW1# show ip igmp profile 10
IGMP Profile 10
    permit
    range 239.1.1.1 239.1.1.1
```

When this profile is applied to Ethernet0/1, the port can dynamically join `239.1.1.1`, but not other multicast groups.

Consider:

```text
                   R1
                    |
                   SW1
                    |
                  E0/1
                    |
                   PC1

             Profile 10
             permit
             239.1.1.1
```

### PC1 Joins a Permitted Group

PC1 sends an IGMPv2 Membership Report for:

```text
239.1.1.1
```

SW1 checks the IGMP profile applied to Ethernet0/1:

```text
IGMP Report received
        |
        v
Ingress: Ethernet0/1
        |
        v
IGMP Profile 10
        |
        v
Does 239.1.1.1 match the range?
        |
       YES
        |
        v
Action: permit
        |
        v
Process membership normally
        |
        v
Learn Ethernet0/1 as a listener
```

The resulting IGMP snooping state can contain:

```text
VLAN 10

239.1.1.1
→ Ethernet0/1
```

The IGMP Report is allowed to proceed through normal snooping and forwarding processing toward the multicast router.

You can view this with `debug ip igmp filter`:

```
*Oct 10 00:21:21.320: IGMPFILTER: igmp_filter_process_pkt(): checking group 239.1.1.1 from Et0/1: permit
```

### PC1 Joins a Nonpermitted Group

Now PC1 sends a Membership Report for:

```text
239.1.1.2
```

The group does not match the permitted range.

```text
IGMP Report received
        |
        v
Ingress: Ethernet0/1
        |
        v
IGMP Profile 10
        |
        v
Does 239.1.1.2 match the range?
        |
        NO
        |
        v
Membership denied
        |
        v
No dynamic listener entry created
```

The switch does not dynamically learn Ethernet0/1 as a listener for `239.1.1.2`.

Therefore:

```text
239.1.1.1 → Permitted
239.1.1.2 → Denied
239.1.1.3 → Denied
```

A `permit` profile permits **only the groups included in its configured ranges**.

You can view this with `debug ip igmp filter`:

```
*Oct 10 00:21:22.023: IGMPFILTER: igmp_filter_process_pkt(): checking group 239.1.1.2 from Et0/1: deny
```

## Configuring Multiple Multicast Groups

The `range` command can be entered multiple times within the same IGMP profile.

For example:

```text
SW1(config)# ip igmp profile 10
SW1(config-igmp-profile)# permit
SW1(config-igmp-profile)# range 239.1.1.1
SW1(config-igmp-profile)# range 239.1.1.3
SW1(config-igmp-profile)# range 239.2.2.2
```

Profile 10 now permits:

```text
239.1.1.1
239.1.1.3
239.2.2.2
```

Other multicast groups are not permitted by the profile.

Importantly, **the `permit` action applies to every configured range**.

There are no separate actions associated with the individual `range` commands.

## Configuring a Range of Multicast Groups

Instead of configuring individual group addresses, `range` can specify a continuous range.

Syntax:

```text
range <start-address> [end-address]
```

For example:

```text
SW1(config)# ip igmp profile 10
SW1(config-igmp-profile)# permit
SW1(config-igmp-profile)# range 239.1.1.1 239.1.1.100
```

This permits multicast addresses from:

```text
239.1.1.1
    through
239.1.1.100
```

Both endpoints are included.

Unlike an IP ACL, the `range` command does not use a subnet mask or wildcard mask.

The starting multicast address must be lower than or equal to the ending multicast address.

### Multiple Ranges

Multiple ranges can also be combined:

```text
SW1(config)# ip igmp profile 10
SW1(config-igmp-profile)# permit
SW1(config-igmp-profile)# range 239.1.1.1 239.1.1.100
SW1(config-igmp-profile)# range 239.2.2.1 239.2.2.100
SW1(config-igmp-profile)# range 239.3.3.3
```

The permitted groups are:

```text
239.1.1.1 - 239.1.1.100

239.2.2.1 - 239.2.2.100

239.3.3.3
```

A single profile therefore supports a collection of individual multicast groups and continuous ranges.

## Denying Multicast Groups

An IGMP profile can also use the `deny` action to create a **blacklist**.

For example:

```text
SW1(config)# ip igmp profile 20
SW1(config-igmp-profile)# deny
SW1(config-igmp-profile)# range 239.1.1.3
```

This denies membership in:

```text
239.1.1.3
```

but permits groups outside the denied range.

Consider the same topology:

```text
                   R1
                    |
                   SW1
                    |
                  E0/2
                    |
                   PC2

             Profile 20
              deny
             239.1.1.3
```

If PC2 sends a Report for `239.1.1.3`:

```text
IGMP Report
Group: 239.1.1.3
        |
        v
Ethernet0/2
        |
        v
IGMP Profile 20
        |
        v
Matches denied range
        |
        v
Report dropped
        |
        v
No dynamic listener entry created
```

However, a Report for `239.1.1.1` does not match the denied range:

```text
IGMP Report
Group: 239.1.1.1
        |
        v
IGMP Profile 20
        |
        v
Does not match denied range
        |
        v
Permitted
```

Therefore:

```text
239.1.1.1 → Permitted
239.1.1.2 → Permitted
239.1.1.3 → Denied
```

## Default IGMP Profile Action

The default action for an IGMP profile is **`deny`**.

For example:

```text
SW1(config)# ip igmp profile 20
SW1(config-igmp-profile)# range 239.1.1.3
```

is equivalent to:

```text
SW1(config)# ip igmp profile 20
SW1(config-igmp-profile)# deny
SW1(config-igmp-profile)# range 239.1.1.3
```

In both cases:

```text
239.1.1.3 → Denied

All other groups → Permitted
```

Be careful not to confuse the **default action within a profile** with the default behavior of the switch.

By default, IOS XE has:

```text
No IGMP profiles defined

No IGMP profiles applied to ports

All group memberships allowed
```

Therefore, the default `deny` action does **not** mean that all multicast group memberships are denied by default.

It means that **matching ranges in a profile use `deny` unless `permit` is configured**.

## Permit vs. Deny Profiles

The two types of profiles behave differently:

| Group Address | `permit` Profile | `deny` Profile |
|---|---|---|
| Matches a configured range | Permitted | Denied |
| Does not match a configured range | Denied | Permitted |

```text
PERMIT PROFILE

Configured ranges
      |
      +---- Matching groups: Permit
      |
      +---- Other groups: Deny
```

```text
DENY PROFILE

Configured ranges
      |
      +---- Matching groups: Deny
      |
      +---- Other groups: Permit
```

The difference is especially important when deciding whether to permit only explicitly authorized multicast groups or to block only a small set of groups.

## IGMP Profiles Are Not ACLs

Although IGMP profiles resemble ACLs, their configuration model is different.

An ACL can contain a mixture of permit and deny statements:

```text
permit ...
deny ...
permit ...
deny ...
```

An IGMP profile instead has **one action that applies to every configured range**.

For example:

```text
SW1(config)# ip igmp profile 10
SW1(config-igmp-profile)# permit
SW1(config-igmp-profile)# range 239.1.1.1
SW1(config-igmp-profile)# range 239.1.1.2
```

Both ranges are permitted.

Entering:

```text
SW1(config-igmp-profile)# deny
```

does not add a separate deny entry.

Instead, it changes the action for the **entire profile**:

```text
239.1.1.1 → Denied
239.1.1.2 → Denied

Other groups → Permitted
```

There is no ACL-style first-match evaluation, rule order, or per-range permit/deny action.

If different switch ports require different policies, create separate profiles and apply them to the appropriate interfaces.

## Complete Configuration Example

Consider the following topology:

```text
                         R1
                   IGMP Querier
                     10.10.10.1
                         |
                       E0/0
                         |
                        SW1
                      VLAN 10
                  /      |      \
                 /       |       \
              E0/1     E0/2     E0/3
               |         |         |
              PC1       PC2       PC3
         10.10.10.11  .12       .13
```

All three PCs are in VLAN 10.

The multicast groups are:

| Group | Description |
|---|---|
| `239.1.1.1` | Channel A |
| `239.1.1.2` | Channel B |
| `239.1.1.3` | Channel C |

The requirements are:

- **PC1:** Allow Channels A and B only.
- **PC2:** Deny Channel B, but allow other groups.
- **PC3:** No restrictions.

### Step 1: Create Profile 10

Profile 10 permits Channels A and B:

```text
SW1(config)# ip igmp profile 10
SW1(config-igmp-profile)# permit
SW1(config-igmp-profile)# range 239.1.1.1 239.1.1.2
SW1(config-igmp-profile)# exit
```

The resulting policy is:

```text
239.1.1.1 → Permit
239.1.1.2 → Permit
239.1.1.3 → Deny
```

### Step 2: Create Profile 20

Profile 20 denies Channel B:

```text
SW1(config)# ip igmp profile 20
SW1(config-igmp-profile)# deny
SW1(config-igmp-profile)# range 239.1.1.2
SW1(config-igmp-profile)# exit
```

The resulting policy is:

```text
239.1.1.1 → Permit
239.1.1.2 → Deny
239.1.1.3 → Permit
```

### Step 3: Apply the Profiles

Confirm the configured profiles:

```
SW1# show ip igmp profile 
IGMP Profile 10
    permit
    range 239.1.1.1 239.1.1.2
IGMP Profile 20
    range 239.1.1.2 239.1.1.2
```

> The `deny` action for Profile 20 does not appear because it is the default configuration.

Apply profile 10 to PC1's port:

```text
SW1(config)# interface Ethernet0/1
SW1(config-if)# switchport mode access
SW1(config-if)# switchport access vlan 10
SW1(config-if)# ip igmp filter 10
```

Apply profile 20 to PC2's port:

```text
SW1(config)# interface Ethernet0/2
SW1(config-if)# switchport mode access
SW1(config-if)# switchport access vlan 10
SW1(config-if)# ip igmp filter 20
```

PC3's port has no IGMP profile applied:

```text
SW1(config)# interface Ethernet0/3
SW1(config-if)# switchport mode access
SW1(config-if)# switchport access vlan 10
```

Assuming ordinary IGMPv2 membership processing and no additional restrictions, the resulting policies are:

| Host | 239.1.1.1 | 239.1.1.2 | 239.1.1.3 |
|---|---|---|---|
| PC1 | Permit | Permit | Deny |
| PC2 | Permit | Deny | Permit |
| PC3 | Permit | Permit | Permit |

### Step 4: Observe Membership Learning

Suppose the hosts send the following IGMPv2 Membership Reports:

```text
PC1 joins:
  239.1.1.1 → Accepted
  239.1.1.3 → Denied

PC2 joins:
  239.1.1.1 → Accepted
  239.1.1.2 → Denied

PC3 joins:
  239.1.1.2 → Accepted
  239.1.1.3 → Accepted
```

SW1 should learn the following listener-port relationships:

```text
VLAN 10

239.1.1.1
→ Ethernet0/1
→ Ethernet0/2

239.1.1.2
→ Ethernet0/3

239.1.1.3
→ Ethernet0/3
```

Notice that `239.1.1.2` is still present in the snooping table even though PC2 was denied membership.

This is because **IGMP filtering is enforced per port**.

A group denied on one port may still be permitted and learned on other ports in the same VLAN.

## Verifying IGMP Profiles

### Checking IGMP Filtering Status

Cisco IOS XE provides `show ip igmp filter` to display IGMP filtering status:

```text
SW1# show ip igmp filter
IGMP filter enabled
```

This sample output indicates that the filtering function is enabled. **It does not mean that a profile is applied to every port**, or identify which profile a particular port uses.

To verify the actual membership policy, inspect the profile configuration and the interface to which it is applied.

### Displaying IGMP Profiles

The primary profile verification command is:

```text
SW1# show ip igmp profile
```

This displays all configured IGMP profiles.

For example:

```text
SW1# show ip igmp profile 
IGMP Profile 10
    permit
    range 239.1.1.1 239.1.1.2
IGMP Profile 20
    range 239.1.1.2 239.1.1.2
```

Notice that profile 20 does not display the `deny` keyword.

### Verifying Interface Application

Use `show running-config interface` to verify which profile is applied:

```text
SW1# show running-config interface Ethernet0/1
```

Relevant configuration:

```text
interface Ethernet0/1
 switchport mode access
 switchport access vlan 10
 ip igmp filter 10
```

For Ethernet0/2:

```text
SW1# show running-config interface Ethernet0/2
```

Relevant configuration:

```text
interface Ethernet0/2
 switchport mode access
 switchport access vlan 10
 ip igmp filter 20
```

### Verifying IGMP Snooping State

Use:

```text
SW1# show ip igmp snooping groups vlan 10
```

After the membership Reports in the previous example, the expected listener state is:

```text
SW1# show ip igmp snooping groups vlan 10
Flags: I -- IGMP snooping, S -- Static, E -- EVPN sync

Vlan      Group                    Type        Version     Port List
-----------------------------------------------------------------------
10        239.1.1.1                I           v2          Et0/1, Et0/2
10        239.1.1.2                I           v2          Et0/3
10        239.1.1.3                I           v2          Et0/3
```

You can also view each Join attempt with `debug ip igmp filter`:

```
*Oct 10 00:41:31.020: IGMPFILTER: igmp_filter_process_pkt(): checking group 239.1.1.1 from Et0/1: permit
*Oct 10 00:41:31.632: IGMPFILTER: igmp_filter_process_pkt(): checking group 239.1.1.3 from Et0/1: deny
*Oct 10 00:41:35.331: IGMPFILTER: igmp_filter_process_pkt(): checking group 239.1.1.1 from Et0/2: permit
*Oct 10 00:41:36.042: IGMPFILTER: igmp_filter_process_pkt(): checking group 239.1.1.2 from Et0/2: deny
*Oct 10 00:41:39.164: IGMPFILTER: igmp_filter_process_pkt() checking group from Et0/3 : no profile attached
*Oct 10 00:41:39.765: IGMPFILTER: igmp_filter_process_pkt() checking group from Et0/3 : no profile attached
```

## Reusing IGMP Profiles

An IGMP profile can be applied to multiple Layer 2 switch ports.

For example:

```text
SW1(config)# interface range Ethernet0/1 - 2
SW1(config-if-range)# ip igmp filter 10
```

Both interfaces now use profile 10.

Conceptually:

```text
                    SW1
                 /       \
              E0/1       E0/2
               |           |
              PC1         PC2

                  |
              Profile 10
                  |
           Same membership
               policy
```

Each port still maintains its own dynamically learned multicast memberships.

The profile determines which groups can be learned on each port.

However, **only one IGMP profile can be applied to an individual port**.

A port cannot independently reference multiple profiles to combine their policies.

## Multiple Receivers Behind One Port

An IGMP profile applies to the switch port, not to an individual host IP address.

Consider:

```text
                   R1
                    |
                   SW1
                    |
                  E0/1
                    |
                   SW2
                 /     \
                /       \
              PC1       PC2
```

Suppose SW1 has profile 10 on Ethernet0/1:

```text
SW1(config)# interface Ethernet0/1
SW1(config-if)# ip igmp filter 10
```

The profile applies to IGMP membership Reports received through Ethernet0/1.

Therefore, it affects both PC1 and PC2 because their Reports arrive at SW1 through the same port.

Conceptually:

```text
PC1 Report ----\
                \
                 SW2 ---- E0/1 ---- SW1
                /
PC2 Report ----/
                          |
                      Profile 10
```

IGMP profile filtering does not identify separate hosts behind a shared port and assign different policies to them.

If PC1 and PC2 require different membership restrictions, filtering should be applied at access ports where their Reports can be distinguished.

## Modifying IGMP Profiles

An existing profile can be modified by entering its configuration mode.

For example:

```text
SW1(config)# ip igmp profile 10
```

### Adding a Range

To add another permitted group:

```text
SW1(config-igmp-profile)# range 239.3.3.3
```

The new range uses the profile's existing action.

### Removing a Range

Use the `no range` command.

For example:

```text
SW1(config-igmp-profile)# no range 239.3.3.3
```

For a range:

```text
SW1(config-igmp-profile)# no range 239.1.1.1 239.1.1.100
```

### Changing the Action

The profile action can be changed:

```text
SW1(config)# ip igmp profile 10
SW1(config-igmp-profile)# deny
```

This changes the action for all configured ranges.

For example, if the profile previously contained:

```text
permit
range 239.1.1.1 239.1.1.100
```

changing it to `deny` means those same groups are now denied.

The ranges themselves do not need to be reconfigured.

Because a profile can be reused across multiple ports, changing the profile can affect every port to which it is applied.

### Removing an IGMP Profile

To remove a profile from the configuration:

```text
SW1(config)# no ip igmp profile 10
```

To remove a profile association from a port without deleting the profile:

```text
SW1(config)# interface Ethernet0/1
SW1(config-if)# no ip igmp filter
```

Removing an interface's filtering configuration restores unrestricted dynamic IGMP membership learning on that port, assuming no other restriction applies.

When modifying live filtering policies, verify the resulting snooping membership state. Previously learned dynamic entries may need to age out or be refreshed before observed forwarding state fully reflects the new policy.

## IGMP Profile Limitations

On Catalyst 9000 IOS XE switches, several restrictions apply.

### Layer 2 Interfaces Only

IGMP profiles are supported on **Layer 2 physical switch ports**.

They are not supported on:

```text
Routed interfaces
Switch Virtual Interfaces (SVIs)
Physical EtherChannel member ports
```

For multicast membership filtering on routed interfaces, IOS XE provides:

```text
ip igmp access-group
```

This is a separate Layer 3 mechanism.

### Dynamic Membership Only

IGMP profiles control dynamically learned memberships.

For example:

```text
PC1
 |
 | IGMP Membership Report
 v
SW1
 |
 v
IGMP Profile
 |
 v
Dynamic membership permitted/denied
```

They do not override statically configured IGMP snooping memberships.

For example:

```text
SW1(config)# ip igmp snooping vlan 10 static 239.1.1.1 interface Ethernet0/1
```

This creates a static multicast listener-port association independently of dynamic IGMP Reports.

An IGMP profile denying `239.1.1.1` does not prevent this static entry from being configured.

### IGMPv3 Filtering Is Not Supported

Cisco documents that **IGMPv3 join and leave messages are not supported by the IGMP filtering feature** on Catalyst 9000 IOS XE.

This limitation does not mean Catalyst 9000 switches lack IGMPv3 snooping support.

The distinction is:

```text
IGMPv3 Snooping
→ Supported

IGMP Profile Filtering of
IGMPv3 membership changes
→ Not supported
```

If you configure the hosts to use **IGMPv3**, the filters will not block their Joins:

```
SW1# show ip igmp snooping groups vlan 10
Flags: I -- IGMP snooping, S -- Static, E -- EVPN sync

Vlan      Group                    Type        Version     Port List
-----------------------------------------------------------------------
10        239.1.1.1                I           v3       Et0/1, Et0/2
10        239.1.1.2                I           v3       Et0/2, Et0/3
10        239.1.1.3                I           v3       Et0/1, Et0/3
```

Additionally, IGMP profiles match **multicast group addresses**, not individual source addresses.

For example:

```text
Group: 239.1.1.1

Source 10.1.1.1
Source 10.2.2.2
```

An IGMP profile does not provide separate permit/deny decisions for:

```text
(10.1.1.1, 239.1.1.1)

(10.2.2.2, 239.1.1.1)
```

That is different from the source-filtering semantics of IGMPv3 and the Layer 3 filtering available with `ip igmp access-group`.

### Membership Filtering Is Not Data Filtering

An IGMP profile filters membership signaling.

It is **not a multicast data-plane ACL**.

For example:

```text
IGMP Membership Report
        |
        v
IGMP Profile
        |
        v
Membership denied
        |
        v
No dynamic listener state
```

This prevents normal dynamic membership learning on the port.

However, multicast data might still reach the port through mechanisms such as unknown multicast flooding, topology-change flooding, or static forwarding entries.

Cisco describes IGMP filtering as applying to **group-specific IGMP Queries and membership Reports**, including Join and Leave signaling. **General IGMP Queries are not filtered** by this feature.

The filtering policy is still a membership-control mechanism, not a multicast data-plane ACL.

## Key Points

- **IGMP profiles** define which multicast groups Layer 2 switch ports can dynamically join.
- Create a profile with `ip igmp profile <number>`.
- A profile has one action: `permit` or `deny`.
- The action applies to **all configured ranges**, not individual entries.
- The default profile action is `deny`, meaning matching groups are denied and nonmatching groups are permitted.
- A `permit` profile allows only groups matching its configured ranges.
- The `range` command accepts individual multicast groups or continuous inclusive ranges.
- Multiple ranges can be configured in one profile.
- Profiles are inactive until applied to an interface using `ip igmp filter`.
- A profile can be reused across multiple ports, but only one profile can be applied per port.
- IGMP profiles affect dynamically learned memberships, not static multicast forwarding entries.
- On Catalyst 9000 IOS XE, IGMP profile filtering does not support IGMPv3 membership changes.
- Use `show ip igmp filter` for filtering status, `show ip igmp profile` for profile configuration, and `show ip igmp snooping groups` to inspect dynamically learned memberships.

