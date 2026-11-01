# 4.2.c (i) RA Guard

**IPv6 Router Advertisement Guard (RA Guard)** is an IPv6 First Hop Security (FHS) feature that prevents unauthorized devices from sending **Router Advertisement (RA)** messages into a switched network. It inspects IPv6 Neighbor Discovery traffic on **ingress Layer 2 ports**, allowing legitimate routers to advertise themselves while blocking RAs from ordinary hosts.

Without RA Guard, any device on an IPv6 LAN could attempt to advertise itself as a default router, announce unauthorized IPv6 prefixes, or change host configuration parameters. RA Guard controls **which ports can originate router advertisements**, and can optionally validate the contents of advertisements received from authorized router-facing ports.

RA Guard is part of the IPv6 FHS infrastructure used alongside **SISF (Switch Integrated Security Features)** device tracking, DHCPv6 Guard, and IPv6 Source Guard. However, **RA Guard uses its own policy commands**. It does not require a binding table to decide whether a host-facing port is allowed to send RAs.

---

## Router Advertisements and Rogue Router Attacks

IPv6 hosts discover routers using **Neighbor Discovery (ND)**, not DHCPv6. Routers periodically transmit RAs and can also respond to **Router Solicitation (RS)** messages sent by hosts.

| Message | ICMPv6 type | Typical sender | Purpose |
| --- | ---: | --- | --- |
| **Router Solicitation (RS)** | 133 | Host | Ask routers to advertise themselves |
| **Router Advertisement (RA)** | 134 | Router | Advertise router availability and IPv6 configuration information |
| **Neighbor Solicitation (NS)** | 135 | Host or router | Address resolution, DAD, and neighbor reachability |
| **Neighbor Advertisement (NA)** | 136 | Host or router | Respond to NS and advertise neighbor information |
| **Redirect** | 137 | Router | Inform a host of a better first hop |

An RA can contain, among other information:

- **Router Lifetime:** Determines how long the sender can be used as a default router. A value of **0** means the sender is not advertising itself as a default router.
- **Router Preference:** Indicates *high*, *medium*, or *low* preference when selecting a default router.
- **Prefix Information Options (PIOs):** Advertise IPv6 prefixes and associated on-link/autonomous-address-configuration flags and lifetimes.
- **M flag (Managed Address Configuration):** Indicates that stateful DHCPv6 address configuration is available.
- **O flag (Other Configuration):** Indicates that other information, such as DNS configuration, is available via DHCPv6.
- **Current Hop Limit, Reachable Time, and Retrans Timer:** Supply parameters hosts may use for IPv6 operation.
- **Additional options:** May include MTU and Recursive DNS Server (RDNSS) information.

**DHCPv6 does not supply the IPv6 default gateway.** Hosts learn default routers through Router Advertisements.

### Rogue RA Example

```text
                      R1 (legitimate router)
                      FE80::1 / 2001:DB8:10::/64
                               |
                            Gi1/0/24
                          +----------+
                          |   SW1    |
                          +----------+
                          /          \
                     Gi1/0/1       Gi1/0/2
                        |              |
                       PC1        Rogue host
                                   FE80::BAD
```

The legitimate router advertises itself as a default gateway for VLAN 10. A rogue host on `Gi1/0/2` sends an RA with its own link-local address as the router address, potentially with a **high router preference**, a different prefix, or malicious configuration parameters.

If PC1 accepts the rogue advertisement, traffic may be redirected through the rogue device (**man-in-the-middle**), blackholed, or affected by incorrect addressing information. A rogue RA with **Router Lifetime 0** can also influence prefix or other configuration without making the sender a default router.

RA Guard stops the unwanted advertisement **at the switch port where it enters SW1**, before the switch forwards it to other hosts.

---

## RA Guard Operation

An RA Guard policy identifies the type of device connected to a switch port. The two principal roles are:

- **`device-role host`:** The port faces an ordinary host. Ingress RAs and ICMPv6 Redirect messages are blocked.
- **`device-role router`:** The port faces a legitimate IPv6 router. Ingress RAs are permitted, subject to any configured policy checks.

The default role within an RA Guard policy is **host**. A policy must be **attached** to a port or other supported target before it takes effect.

Two additional roles are **`switch`** and **`monitor`**. They are useful for identifying switch-to-switch links and monitoring devices, but their RA and RS handling differs from the `router` role. They are covered in [Switch and Monitor Roles](#switch-and-monitor-roles).

### Ingress Filtering

RA Guard filters traffic based on the port **receiving** the packet. It does not block a legitimate RA merely because that RA is being **sent out toward** a host-facing port.

```text
             Legitimate RA                         Rogue RA
               from R1                             from PC2
                  |                                   |
                  v                                   v
             Gi1/0/24                            Gi1/0/2
          device-role router                  device-role host
                  |                                   |
              PERMITTED                              DROPPED
                  |                                   X
                  v
                SW1
                  |
                  v
             Gi1/0/1
                PC1
```

| Packet received by SW1 | Ingress port role | Result |
| --- | --- | --- |
| RA from the legitimate router | `router` | Allowed if it passes any additional policy checks |
| RA from a host | `host` | **Dropped** |
| ICMPv6 Redirect from a host | `host` | **Dropped** |
| RS from a host | `host` | Allowed on ingress; must still reach a router-facing port |
| NS or NA from a host | `host` | Not blocked merely because RA Guard is enabled |
| Ordinary IPv6 data packet | Either | Not filtered by RA Guard itself |

RA Guard is therefore **not a general-purpose IPv6 packet filter**, nor does it replace ND validation or IPv6 Source Guard.

### Router Solicitations and Router-Facing Ports

A particularly important forwarding rule is that **router-facing ports should explicitly have an RA Guard policy with `device-role router`**.

In Cisco's SISF-based forwarding behavior, RS messages are replicated to interfaces identified as router-facing. If RA Guard protects host-facing ports but no router-facing interface is properly identified, the RS might not reach the router. Hosts may then have to wait for the next unsolicited RA instead of immediately receiving one.

**Protect both sides of the topology:** `host` on host access ports and `router` on the uplink to the legitimate router.

---

## Switch and Monitor Roles

In addition to `host` and `router`, RA Guard supports the **`switch`** and **`monitor`** device roles.

### Switch Role

`device-role switch` is functionally identical to `device-role host`. Both block incoming Router Advertisements (RAs) and Redirects.

The `switch` role is primarily an **administrative label** for ports connected to other switches.

```text
SW1(config)# ipv6 nd raguard policy SWITCHES
SW1(config-nd-raguard)# device-role switch
```

Use this role on switch-facing ports that should not receive RAs from downstream devices.

If an inter-switch link must carry legitimate RAs from routers connected to other switches, use `device-role router` instead.

### Monitor Role

`device-role monitor` also blocks incoming RAs and Redirects, but unlike `host` and `switch`, it **receives Router Solicitations (RSs) forwarded from other ports**.

```text
SW1(config)# ipv6 nd raguard policy MONITORS
SW1(config-nd-raguard)# device-role monitor
```

This allows a monitoring device to observe hosts attempting to discover IPv6 routers.

### Role Comparison

| Role | Incoming RAs/Redirects | Receives forwarded RSs |
| --- | --- | --- |
| **`host`** | Blocked | No |
| **`switch`** | Blocked | No |
| **`router`** | Permitted (subject to policy) | Yes |
| **`monitor`** | Blocked | Yes |

### Choosing the Correct Role

- Use **`host`** for ordinary client-facing ports.
- Use **`router`** wherever legitimate RAs must be accepted from the connected network, including inter-switch links behind which legitimate routers reside.
- Use **`switch`** for switch-facing ports that should not admit RAs from the connected network.
- Use **`monitor`** for ports that should receive forwarded RS messages for observation but must not originate RAs.

---

## RA Guard Configuration

RA Guard is configured by **creating a policy** and **attaching it to the relevant Layer 2 interfaces**.

### Step 1: Create a Host Policy

```text
SW1(config)# ipv6 nd raguard policy HOSTS
SW1(config-nd-raguard)# device-role host
SW1(config-nd-raguard)# exit
```

`device-role host` is the default for an RA Guard policy, but configuring it explicitly makes the purpose clear.

### Step 2: Create a Router Policy

```text
SW1(config)# ipv6 nd raguard policy ROUTERS
SW1(config-nd-raguard)# device-role router
SW1(config-nd-raguard)# exit
```

This role allows legitimate router advertisements to enter the switch. It is **not identical to `trusted-port`**, which bypasses further RA Guard checks; `device-role router` can still be combined with validation options.

### Step 3: Attach the Policies

For the topology above:

```text
SW1(config)# interface range GigabitEthernet1/0/1-2
SW1(config-if-range)# switchport mode access
SW1(config-if-range)# switchport access vlan 10
SW1(config-if-range)# ipv6 nd raguard attach-policy HOSTS
!
SW1(config)# interface GigabitEthernet1/0/24
SW1(config-if)# switchport mode access
SW1(config-if)# switchport access vlan 10
SW1(config-if)# ipv6 nd raguard attach-policy ROUTERS
```

Now:

- PC1 can receive RAs from R1 and use R1 as its IPv6 default router.
- The rogue host on `Gi1/0/2` cannot advertise itself as a router to the rest of the VLAN.
- RS messages from PC1 can reach the legitimate router-facing port.

The basic configuration does **not** require an IPv6 ACL, an IPv6 prefix list, an IPv6 binding table, or a separate global RA Guard enable command.

### Attaching the Default Host Policy

A policy name is optional:

```text
SW1(config)# interface GigabitEthernet1/0/1
SW1(config-if)# ipv6 nd raguard attach-policy
```

This applies the **default RA Guard policy**, with the **host role**. Use a named policy when you want explicit roles or additional checks.

---

## RA Guard Policy Options

A `device-role router` policy can apply extra validation to Router Advertisements instead of trusting every RA received on the router-facing port.

| Command | What it validates |
| --- | --- |
| `device-role host` | Blocks ingress RAs and Redirects from the port |
| `device-role router` | Allows a router to originate RAs, subject to configured checks |
| `match ipv6 access-list NAME` | Checks the **source IPv6 address** of the RA sender against an authorized IPv6 ACL |
| `match ra prefix-list NAME` | Checks advertised prefixes against an authorized IPv6 prefix list |
| `hop-limit minimum N` | Rejects RAs advertising a **Current Hop Limit** below N |
| `hop-limit maximum N` | Rejects RAs advertising a **Current Hop Limit** above N |
| `managed-config-flag on\|off` | Requires the RA's **M flag** to be 1 or 0 |
| `other-config-flag on\|off` | Requires the RA's **O flag** to be 1 or 0 |
| `router-preference maximum high\|medium\|low` | Rejects RAs with a router preference exceeding the configured maximum |
| `trusted-port` | Trusts the port and **disables further RA Guard validation** |

Checks that are not configured are not enforced. Multiple configured checks must be satisfied for an RA to pass.

### Source IPv6 Address Filtering

Use `match ipv6 access-list` to restrict which IPv6 source addresses may send RAs:

```text
SW1(config)# ipv6 access-list ALLOWED_RA_SENDERS
SW1(config-ipv6-acl)# permit ipv6 host FE80::1 any
SW1(config-ipv6-acl)# exit
!
SW1(config)# ipv6 nd raguard policy ROUTERS
SW1(config-nd-raguard)# device-role router
SW1(config-nd-raguard)# match ipv6 access-list ALLOWED_RA_SENDERS
```

Only an RA from a sender whose source address is authorized by the ACL passes this check. **The sender of an IPv6 RA normally uses its link-local address**, not its advertised global unicast prefix.

The ACL is referenced **inside the RA Guard policy**. It is not being installed as a general-purpose interface ACL with `ipv6 traffic-filter`.

### Advertised Prefix Filtering

Use `match ra prefix-list` to restrict the prefixes advertised in RA **Prefix Information Options**:

```text
SW1(config)# ipv6 prefix-list ALLOWED_RA_PREFIXES seq 5 permit 2001:DB8:10::/64
!
SW1(config)# ipv6 nd raguard policy ROUTERS
SW1(config-nd-raguard)# device-role router
SW1(config-nd-raguard)# match ra prefix-list ALLOWED_RA_PREFIXES
```

This validates the **advertised prefix**, not the IPv6 source address of the router. The two checks solve different problems and can be combined.

A prefix list is useful when a router-facing port should advertise only a particular subnet. For example, an RA that advertises `2001:DB8:99::/64` would fail the prefix-list check above.

### Current Hop Limit Validation

```text
SW1(config)# ipv6 nd raguard policy ROUTERS
SW1(config-nd-raguard)# hop-limit minimum 32
SW1(config-nd-raguard)# hop-limit maximum 128
```

These commands validate the **Current Hop Limit field inside the ICMPv6 RA**. This is the value the router advertises for hosts to use as their default IPv6 Hop Limit when sending packets.

**Do not confuse these two fields:**

- **IPv6 header Hop Limit:** An RA must be sent with an IPv6 header Hop Limit of **255**, so it cannot legitimately be forwarded from another link.
- **RA Current Hop Limit:** An advertised host parameter carried *inside* the RA, commonly a value such as 64. This is the field checked by the RA Guard `hop-limit` policy commands.

A restrictive RA Guard hop-limit policy can reject valid RAs if it does not account for the legitimate router's advertised value. An RA that specifies **Current Hop Limit 0** (unspecified) is also rejected when this validation is enabled.

### M and O Flag Validation

Use these commands to require particular DHCPv6-related flags in received RAs:

```text
SW1(config)# ipv6 nd raguard policy ROUTERS
SW1(config-nd-raguard)# managed-config-flag off
SW1(config-nd-raguard)# other-config-flag on
```

- `managed-config-flag off` accepts RAs with **M = 0**, and rejects RAs with M = 1.
- `other-config-flag on` accepts RAs with **O = 1**, and rejects RAs with O = 0.

These checks control **which flag values the RA is permitted to advertise**. They do not configure DHCPv6 on clients and do not themselves stop rogue DHCPv6 servers. That requires **DHCPv6 Guard**.

### Router Preference Validation

The `router-preference maximum` option establishes an **upper limit**, not an exact match:

```text
SW1(config)# ipv6 nd raguard policy ROUTERS
SW1(config-nd-raguard)# router-preference maximum medium
```

| Maximum configured | Low preference RA | Medium preference RA | High preference RA |
| --- | --- | --- | --- |
| `low` | Permit | Drop | Drop |
| `medium` | Permit | Permit | Drop |
| `high` | Permit | Permit | Permit |

Thus, `router-preference maximum high` effectively allows every valid advertised preference. The check does **not** establish a minimum router preference.

### Trusted Port Versus Router Role

A router-facing RA Guard policy can also contain:

```text
SW1(config)# ipv6 nd raguard policy TRUSTED_ROUTER
SW1(config-nd-raguard)# device-role router
SW1(config-nd-raguard)# trusted-port
```

`trusted-port` **disables RA Guard policing** on the interface where the policy is applied. Use it when the connected device is fully trusted and no additional RA validation is needed.

If you want to validate **source address, advertised prefix, or RA flags**, use `device-role router` **without** `trusted-port`.

---

## Complete Configuration with RA Validation

Consider a VLAN with:

- **VLAN:** 10
- **Legitimate router:** R1, connected to `Gi1/0/24`
- **R1 link-local address:** `FE80::1`
- **Authorized advertised prefix:** `2001:DB8:10::/64`
- **Host ports:** `Gi1/0/1-2`

First, give the router a stable link-local address for the source-address filter:

```text
R1(config)# ipv6 unicast-routing
R1(config)# interface GigabitEthernet1
R1(config-if)# ipv6 address FE80::1 link-local
R1(config-if)# ipv6 address 2001:DB8:10::1/64
R1(config-if)# no shutdown
```

Then configure SW1:

```text
! Authorize the legitimate RA sender and the advertised subnet.
ipv6 access-list ALLOWED_RA_SENDERS
 permit ipv6 host FE80::1 any
!
ipv6 prefix-list ALLOWED_RA_PREFIXES seq 5 permit 2001:DB8:10::/64
!
! Reject RAs entering from ordinary host ports.
ipv6 nd raguard policy HOSTS
 device-role host
!
! Permit RAs only from the authorized router and prefix.
ipv6 nd raguard policy ROUTERS
 device-role router
 match ipv6 access-list ALLOWED_RA_SENDERS
 match ra prefix-list ALLOWED_RA_PREFIXES
!
interface range GigabitEthernet1/0/1-2
 switchport mode access
 switchport access vlan 10
 ipv6 nd raguard attach-policy HOSTS
!
interface GigabitEthernet1/0/24
 switchport mode access
 switchport access vlan 10
 ipv6 nd raguard attach-policy ROUTERS
```

With this configuration, an RA must **arrive on the router-facing port**, have the **authorized source IPv6 address**, and have any advertised prefixes accepted by the **authorized prefix list** to pass the configured checks.

This example intentionally leaves `trusted-port` out of the router policy so the source and prefix checks are enforced.

---

## RA Guard on Trunk Ports and VLANs

RA Guard can be attached to a Layer 2 trunk, and supported IOS XE switches can scope a policy to selected VLANs on that interface:

```text
SW1(config)# interface GigabitEthernet1/0/24
SW1(config-if)# switchport mode trunk
SW1(config-if)# switchport trunk allowed vlan 10,20
SW1(config-if)# ipv6 nd raguard attach-policy ROUTERS vlan 10,20
```

This applies the `ROUTERS` policy to **VLANs 10 and 20 on that port**.

Some platforms also support VLAN-wide policy attachment using `vlan configuration`:

```text
SW1(config)# vlan configuration 10
SW1(config-vlan-config)# ipv6 nd raguard attach-policy HOSTS
```

Be careful with VLAN-wide host policies: the actual router-facing path still needs a **router-role exception or appropriately applied router policy**, or legitimate RAs and RS forwarding can be affected.

For lab and troubleshooting clarity, **interface-specific policies on host-facing and router-facing ports** are a good starting point.

---

## Multiple Routers Across Two Switches

RA Guard must also be designed for networks where **multiple routers share the same Layer 2 VLAN**, even if the routers connect to different switches.

Consider this topology:

```text
      R1                                   R2
  FE80::1                               FE80::2
2001:DB8:10::1                       2001:DB8:10::2
       | Gi1                              | Gi1
       |                                  |
   Gi1/0/24                           Gi1/0/24
  +---------+       VLAN 10           +---------+
  |   SW1   |=========================|   SW2   |
  +---------+ Gi1/0/48       Gi1/0/48 +---------+
       |                                  |
   Gi1/0/1                           Gi1/0/1
       |                                  |
      PC1                                PC2

           One Layer 2 LAN: VLAN 10
           Prefix: 2001:DB8:10::/64
```

- R1 and R2 are both legitimate IPv6 routers for **VLAN 10**.
- Both routers advertise the **same on-link prefix**, `2001:DB8:10::/64`.
- R1 advertises a **high** default-router preference; R2 advertises a **low** preference.
- The link between SW1 and SW2 extends VLAN 10 between the switches.

### Which Ports Need Which Roles?

| Switch | Interface | Connected device | RA Guard role |
| --- | --- | --- | --- |
| SW1 | `Gi1/0/1` | PC1 | `host` |
| SW1 | `Gi1/0/24` | R1 | `router` |
| SW1 | `Gi1/0/48` | SW2 | **`router`** |
| SW2 | `Gi1/0/1` | PC2 | `host` |
| SW2 | `Gi1/0/24` | R2 | `router` |
| SW2 | `Gi1/0/48` | SW1 | **`router`** |

**The inter-switch link uses `device-role router` at both ends**, because RAs from a legitimate router on one switch must be allowed to enter the other switch. The role describes the **type of IPv6 control traffic the port may receive**, not necessarily the type of physical device directly connected to it.

The `device-role switch` option is not appropriate for this transit link: it behaves like the host role for RA filtering and can block the RAs that must cross the link.

### How Router Advertisements Cross the Link

Suppose **R1 sends an RA** to the all-nodes multicast address `FF02::1`:

1. SW1 receives the RA on `Gi1/0/24` (**router role**), so it is permitted.
2. SW1 forwards the RA within VLAN 10, including toward PC1 and across `Gi1/0/48` to SW2.
3. SW2 receives the RA on its `Gi1/0/48` (**router role**), so it is permitted.
4. SW2 forwards the RA to PC2.

The same process works in the opposite direction when **R2 sends an RA**. As a result, **PC1 and PC2 can learn both R1 and R2 as default routers**.

RA Guard checks the RA at **ingress**. The `host` role on PC1's and PC2's access ports blocks RAs *originating from those hosts*, but does **not** prevent legitimate RAs from being forwarded *out* those ports toward the hosts.

### Router Solicitations Across the Link

Suppose PC1 sends an **RS** to `FF02::2` (all routers). It enters SW1 on the host-facing port. SW1 forwards the solicitation toward its router-facing ports, including the port to R1 and the transit link to SW2. SW2 can then forward the solicitation toward R2.

This is another reason the inter-switch ports need the **router role on both switches**. Without appropriate router-role ports, SISF-based RS forwarding can fail, delaying router discovery until unsolicited RAs arrive.

### Configuration

Configure both routers as members of the same IPv6 LAN:

**R1:**

```text
ipv6 unicast-routing
!
interface GigabitEthernet1
 ipv6 address FE80::1 link-local
 ipv6 address 2001:DB8:10::1/64
 ipv6 nd router-preference high
 no shutdown
```

**R2:**

```text
ipv6 unicast-routing
!
interface GigabitEthernet1
 ipv6 address FE80::2 link-local
 ipv6 address 2001:DB8:10::2/64
 ipv6 nd router-preference low
 no shutdown
```

Configure an RA Guard policy on **each switch**. The following example uses the basic router role without additional RA content checks.

**SW1:**

```text
vlan 10
!
ipv6 nd raguard policy HOSTS
 device-role host
!
ipv6 nd raguard policy ROUTERS
 device-role router
!
interface GigabitEthernet1/0/1
 switchport mode access
 switchport access vlan 10
 ipv6 nd raguard attach-policy HOSTS
!
interface GigabitEthernet1/0/24
 switchport mode access
 switchport access vlan 10
 ipv6 nd raguard attach-policy ROUTERS
!
interface GigabitEthernet1/0/48
 switchport mode trunk
 switchport trunk allowed vlan 10
 ipv6 nd raguard attach-policy ROUTERS vlan 10
```

**SW2:**

```text
vlan 10
!
ipv6 nd raguard policy HOSTS
 device-role host
!
ipv6 nd raguard policy ROUTERS
 device-role router
!
interface GigabitEthernet1/0/1
 switchport mode access
 switchport access vlan 10
 ipv6 nd raguard attach-policy HOSTS
!
interface GigabitEthernet1/0/24
 switchport mode access
 switchport access vlan 10
 ipv6 nd raguard attach-policy ROUTERS
!
interface GigabitEthernet1/0/48
 switchport mode trunk
 switchport trunk allowed vlan 10
 ipv6 nd raguard attach-policy ROUTERS vlan 10
```

On both switches, additional client-facing interfaces in VLAN 10 should also receive the `HOSTS` policy. Applying the policy only to PC1 and PC2's ports would leave other unprotected host ports capable of injecting rogue RAs.

### What If You Restrict RA Sender Addresses?

If you use `match ipv6 access-list` in router policies, account for **which router's RA will arrive on each interface**:

- **SW1's R1-facing port:** Permit `FE80::1`.
- **SW1's inter-switch port:** Permit `FE80::2` (R2's RAs enter SW1 here).
- **SW2's R2-facing port:** Permit `FE80::2`.
- **SW2's inter-switch port:** Permit `FE80::1` (R1's RAs enter SW2 here).

A policy that permits only R1's source address on **all** SW1 router-role ports would inadvertently **drop R2's legitimate advertisements** on the inter-switch link. You can use separate policies for the local-router port and the inter-switch port when more restrictive validation is needed.

### Which Router Do the Hosts Use?

RA Guard **does not elect a default router**. With both RAs delivered successfully, hosts can maintain both routers in their Default Router List. Selection is influenced by **Router Preference**, Neighbor Unreachability Detection, and host behavior.

In this example, a host that supports RFC 4191 generally prefers reachable **R1 (high)** over **R2 (low)**. If R1 becomes unreachable, the host can select R2. The transition depends on IPv6 neighbor reachability and router lifetime processing; **RA Guard does not perform the failover**.

### Troubleshooting This Topology

| Symptom | Likely RA Guard issue |
| --- | --- |
| PC1 discovers R1 but not R2 | SW1's inter-switch ingress port uses `host` or `switch`, or rejects R2's source/prefix |
| PC2 discovers R2 but not R1 | SW2's inter-switch ingress port uses `host` or `switch`, or rejects R1's source/prefix |
| PC1 cannot promptly solicit R2 | Router-role configuration missing on the inter-switch path or on R2's switch-facing port |
| PCs receive RAs from a rogue host | The rogue host's actual ingress access port is missing the `HOSTS` policy |

Useful checks on **both** switches:

```text
SW1# show ipv6 nd raguard policy
SW1# show running-config interface GigabitEthernet1/0/24
SW1# show running-config interface GigabitEthernet1/0/48
```

A packet capture on the SW1-SW2 link should show RAs from **both `FE80::1` and `FE80::2`** in VLAN 10. On an IPv6 host that supports the command, `show ipv6 routers` can display learned default routers and their advertised preferences.

**Key distinction:** Inter-switch links do not automatically need the router role. They need it **when legitimate RAs must enter through them**, as in this topology. A trunk carrying only hosts and no legitimate RA source behind it should not be given a permissive router-role policy unnecessarily.

---
## Verification and Troubleshooting

### Verify Policy Contents and Attachments

```text
SW1# show ipv6 nd raguard policy
SW1# show ipv6 nd raguard policy HOSTS
SW1# show ipv6 nd raguard policy ROUTERS
SW1# show running-config interface GigabitEthernet1/0/1
SW1# show running-config interface GigabitEthernet1/0/24
```

The policy output identifies configured options and the interfaces to which a policy has been applied. For example:

```text
Policy HOSTS configuration:
  device-role host
Policy applied on the following interfaces:
  Gi1/0/1     vlan all
  Gi1/0/2     vlan all
```

The output above is illustrative; the precise formatting varies across IOS XE releases.

Some platforms also provide RA Guard drop counters or the following targeted debugging command:

```text
SW1# debug ipv6 snooping raguard
```

Use debugging selectively on busy switches and disable it after troubleshooting with `undebug all`.

### Check IPv6 Router Discovery on the Host

A host should:

1. Send a **Router Solicitation** when it needs router information.
2. Receive a valid **Router Advertisement** from R1.
3. Learn R1's **link-local address** as its default-router next hop.
4. Learn the advertised prefix and other permitted configuration information.

The default route will typically use a **link-local next hop** such as `FE80::1`, even if the router advertises a global prefix like `2001:DB8:10::/64`.

### Common Problems

| Symptom | Likely cause | What to check |
| --- | --- | --- |
| Hosts receive RAs from a rogue PC | RA Guard missing from the rogue host's ingress port | Confirm `HOSTS` policy is attached to that port/VLAN |
| Hosts do not receive RAs from the legitimate router | Router uplink has `device-role host`, or a router policy check rejects RAs | Verify role, source ACL, advertised prefix, and flag checks |
| Hosts wait for periodic RAs after sending RS | Router-facing interface is not identified with `device-role router` | Confirm `ROUTERS` policy is attached to the correct uplink |
| RAs are accepted even when source/prefix validation was expected | Policy contains `trusted-port`, or checks were not configured | Remove unintended trust and verify the attached policy |
| RAs from a valid router are rejected | Source link-local address changed or authorized prefix list is too restrictive | Compare actual RA contents with the configured ACL/prefix list |
| RAs are blocked unexpectedly after changing DHCPv6 settings | M or O flag check no longer matches | Compare the flags advertised by the router with policy settings |
| High-preference rogue RAs pass a router port | `router-preference maximum high` allows all preference levels | Use `medium` or `low` if required, and restrict authorized senders |

For packet-level verification, capture ICMPv6 **type 133 (RS)** and **type 134 (RA)** on both sides of the switch. A packet that enters a host-facing port as an RA should **not** appear on other ports when the host policy is functioning correctly.

---

## RA Guard and Other IPv6 Security Features

RA Guard addresses a specific class of attacks: **unauthorized router advertisements and router redirects**. The rest of the IPv6 FHS feature set handles other threats.

| Feature | Primary function |
| --- | --- |
| **RA Guard** | Prevent hosts or unauthorized devices from sending RAs |
| **DHCPv6 Guard** | Block unauthorized DHCPv6 server and relay traffic |
| **SISF Device Tracking** | Discover and track IPv6 neighbor addresses and their attachment points |
| **IPv6 Binding Table** | Record observed or authorized neighbor address-to-MAC/port relationships |
| **ND Inspection / SISF validation** | Validate Neighbor Discovery information and help prevent address spoofing |
| **IPv6 Source Guard** | Filter IPv6 data traffic with unauthorized source addresses |

**RA Guard does not replace DHCPv6 Guard.** A device can send malicious DHCPv6 server messages without sending a Router Advertisement. Likewise, blocking rogue RAs does not by itself prevent a host from spoofing an IPv6 source address in ordinary traffic.

### Security Scope

RA Guard protects the **local Layer 2 path** where it is applied. It is not a defense against all attacks involving IPv6 tunnels, and older implementations have been vulnerable to evasion using IPv6 Extension Headers and fragmentation. Current implementations should follow the packet-parsing protections described in RFC 7113.

For CCIE troubleshooting, the most important practical distinction is **where the rogue RA enters**, **which device role is configured**, and **whether the router-facing port and optional RA validation checks are correct**.

---

## Key Points

- **RA Guard** blocks unauthorized ICMPv6 **Router Advertisements (Type 134)** received on protected Layer 2 ports.
- **`device-role host`** blocks ingress RAs and router redirects; **`device-role router`** identifies ports allowed to receive legitimate router control messages.
- RA Guard is **ingress filtering**. A valid RA received on a router-facing port can still be forwarded out host-facing ports.
- Host-facing ports and router-facing ports should be configured with the appropriate RA Guard policies so **Router Solicitations reach the router**.
- Define policies with **`ipv6 nd raguard policy`** and apply them with **`ipv6 nd raguard attach-policy`**.
- The RA Guard policy's default device role is **host**, but the feature does not enforce a policy until it is attached.
- `match ipv6 access-list` validates the **RA sender's IPv6 address**; `match ra prefix-list` validates **prefixes advertised in the RA**.
- `hop-limit` validates the **Current Hop Limit field carried in the RA**, not the mandatory IPv6 header Hop Limit of 255.
- `managed-config-flag` and `other-config-flag` check **M and O values**; `router-preference maximum` defines an **upper bound**.
- **`trusted-port` bypasses RA Guard checks**; do not configure it when source, prefix, or flag validation is required.
- **`device-role switch`** is host-like for ingress RA filtering; **`device-role monitor`** also blocks ingress RAs but receives forwarded RS messages for monitoring.
- On a **multi-router VLAN spanning switches**, the inter-switch link needs a **router-role policy at both ends** if it carries legitimate RAs in both directions. Host-facing access ports still use the host role.
- RA Guard and SISF device tracking are related FHS components, but **RA Guard does not depend on a host binding table to block rogue RAs**.

---

## References

- [Cisco: Configuring IPv6 First Hop Security (Catalyst 9600, IOS XE 26.x)](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9600/software/release/26-x/configuration_guide/sec/b_26x_sec_9600_cg/configuring_ipv6_first_hop_security.html)
- [Cisco: IPv6 RA Guard Configuration](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/security-vpn/security-vpn/m_ip6-ra-guard.html)
- [Cisco: IPv6 RA Guard Command Reference](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/ipv6/command/ipv6-cr-book/ipv6-i5.html)
- [Cisco: Device Role Command Reference](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/ipv6/command/ipv6-cr-book/ipv6-d2.html)
- [Cisco: RA Guard Device Roles and Configuration](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst3850/software/release/37e/consolidated_guide/b_37e_consolidated_3850_cg/b_37e_consolidated_3850_cg_chapter_01000111.html)
- [Cisco: IPv6 Default Router Preference](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-addressing/b-ip-addressing/m_ip6-def-router-pref.html)
- [RFC 4861: Neighbor Discovery for IP version 6](https://www.rfc-editor.org/rfc/rfc4861.html)
- [RFC 4191: Default Router Preferences and More-Specific Routes](https://www.rfc-editor.org/rfc/rfc4191.html)
- [RFC 7113: Implementation Advice for IPv6 Router Advertisement Guard](https://www.rfc-editor.org/rfc/rfc7113.html)
