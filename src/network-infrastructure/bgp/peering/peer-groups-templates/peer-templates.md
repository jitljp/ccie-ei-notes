# BGP Peer Templates

**BGP peer templates** allow multiple neighbors to inherit common BGP configuration settings.

Unlike [BGP Peer Groups](peer-groups.md), peer templates separate session configuration from address-family-specific routing policies and support **multilevel inheritance**.

There are two types:

| Template type | Purpose | Example commands |
|---|---|---|
| **Peer session template** | BGP session configuration | `remote-as`, `update-source`, `timers`, `ebgp-multihop` |
| **Peer policy template** | Address-family-specific configuration | `route-map`, `prefix-list`, `send-community`, `next-hop-self`, `route-reflector-client` |

A neighbor can inherit **one peer session template and one peer policy template per address family**, with additional templates inherited indirectly.

---

## Peer Session Templates

**Peer session templates** define parameters that control BGP neighbor sessions.

They are created under BGP router configuration mode:

```text
R1(config)# router bgp 65001
R1(config-router)# template peer-session IBGP-SESSION
R1(config-router-stmp)# update-source Loopback0
R1(config-router-stmp)# timers 10 30
R1(config-router-stmp)# exit-peer-session
```

The template does not establish any BGP sessions by itself.

### Applying a Peer Session Template

Consider R1 with two IBGP neighbors:

```text
                    AS 65001

           R2                    R3
       Lo0: 2.2.2.2          Lo0: 3.3.3.3
             \                  /
              \                /
               \              /
                     R1
                Lo0: 1.1.1.1
```

Assume all loopback addresses are reachable through an IGP.

Configure R1:

```text
R1(config)# router bgp 65001
R1(config-router)# no bgp default ipv4-unicast
R1(config-router)# template peer-session IBGP-SESSION
R1(config-router-stmp)# update-source Loopback0
R1(config-router-stmp)# timers 10 30
R1(config-router-stmp)# exit-peer-session
R1(config-router)# neighbor 2.2.2.2 remote-as 65001
R1(config-router)# neighbor 2.2.2.2 inherit peer-session IBGP-SESSION
R1(config-router)# neighbor 3.3.3.3 remote-as 65001
R1(config-router)# neighbor 3.3.3.3 inherit peer-session IBGP-SESSION
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 2.2.2.2 activate
R1(config-router-af)# neighbor 3.3.3.3 activate
```

Both neighbors inherit:

- `update-source Loopback0`
- `timers 10 30`

The `inherit peer-session` command is entered under BGP router configuration mode, not address-family configuration mode.

**Important:** Cisco's documented template configuration procedure requires an explicit `neighbor remote-as` statement to establish the neighbor definition, even when the session template contains a `remote-as` setting.

---

## Peer Session Template Inheritance

A peer session template can inherit settings from another session template.

For example, configure common timers in one template:

```text
R1(config)# router bgp 65001
R1(config-router)# template peer-session COMMON-SESSION
R1(config-router-stmp)# timers 10 30
R1(config-router-stmp)# exit-peer-session
```

Create another template that inherits these settings:

```text
R1(config-router)# template peer-session IBGP-SESSION
R1(config-router-stmp)# update-source Loopback0
R1(config-router-stmp)# inherit peer-session COMMON-SESSION
R1(config-router-stmp)# exit-peer-session
```

Apply `IBGP-SESSION` to a neighbor:

```text
R1(config-router)# neighbor 2.2.2.2 remote-as 65001
R1(config-router)# neighbor 2.2.2.2 inherit peer-session IBGP-SESSION
```

The inheritance structure is:

```text
              COMMON-SESSION
                timers 10 30
                     |
                     v
                IBGP-SESSION
          update-source Loopback0
                     |
                     v
                   R2
                 2.2.2.2
```

R2 inherits both settings:

| Setting | Source |
|---|---|
| `update-source Loopback0` | Directly inherited `IBGP-SESSION` |
| `timers 10 30` | Indirectly inherited `COMMON-SESSION` |

### Session Template Inheritance Limits

A neighbor can inherit:

- **One directly applied peer session template**.
- **Up to seven additional indirectly inherited session templates**.
- **Eight session templates in total**.

Each session template can inherit only **one other session template directly**.

Therefore, session template inheritance forms a chain rather than a branching structure.

```text
TEMPLATE-A
    |
    v
TEMPLATE-B
    |
    v
TEMPLATE-C
    |
    v
 Neighbor
```

A session template cannot contain two different `inherit peer-session` statements.

### Session Template Precedence

When inherited session templates configure the same parameter, the template closest to the neighbor takes precedence.

For example:

```text
R1(config)# router bgp 65001
R1(config-router)# template peer-session COMMON
R1(config-router-stmp)# timers 30 90
R1(config-router-stmp)# exit-peer-session
R1(config-router)# template peer-session IBGP
R1(config-router-stmp)# timers 10 30
R1(config-router-stmp)# inherit peer-session COMMON
R1(config-router-stmp)# exit-peer-session
```

A neighbor inheriting `IBGP` uses:

```text
timers 10 30
```

The directly inherited `IBGP` template overrides the indirectly inherited `COMMON` template.

**Inheritance is evaluated from the most distant ancestor toward the directly applied template.**

---

## Peer Policy Templates

**Peer policy templates** define address-family-specific BGP settings.

Unlike session templates, policy templates control route advertisement, filtering, attribute handling, and other address-family behaviors.

Configure a policy template under BGP router configuration mode:

```text
R1(config)# router bgp 65001
R1(config-router)# template peer-policy IBGP-POLICY
R1(config-router-ptmp)# next-hop-self
R1(config-router-ptmp)# send-community both
R1(config-router-ptmp)# exit-peer-policy
```

### Applying a Peer Policy Template

Peer policy templates are applied to neighbors under an address family.

```text
R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 2.2.2.2 inherit peer-policy IBGP-POLICY
R1(config-router-af)# neighbor 3.3.3.3 inherit peer-policy IBGP-POLICY
```

Both neighbors inherit:

- `next-hop-self`
- `send-community both`

The neighbors must already be defined and activated for the intended address family.

**Unlike session templates, peer policy templates are applied under address-family configuration mode.**

The same policy template can be applied to different supported address families, subject to the commands being valid for those address families.

---

## Peer Policy Template Inheritance

Peer policy templates can inherit configuration from other policy templates.

Unlike session templates, a policy template can directly inherit **multiple policy templates**.

Each inherited policy template is assigned a sequence number.

### Example

Create two templates:

```text
R1(config)# router bgp 65001
R1(config-router)# template peer-policy COMMON-POLICY
R1(config-router-ptmp)# send-community both
R1(config-router-ptmp)# exit-peer-policy
R1(config-router)# template peer-policy NEXT-HOP-POLICY
R1(config-router-ptmp)# next-hop-self
R1(config-router-ptmp)# exit-peer-policy
```

Create a third template that inherits both:

```text
R1(config-router)# template peer-policy IBGP-POLICY
R1(config-router-ptmp)# inherit peer-policy COMMON-POLICY 10
R1(config-router-ptmp)# inherit peer-policy NEXT-HOP-POLICY 20
R1(config-router-ptmp)# exit-peer-policy
```

Apply it to a neighbor:

```text
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 2.2.2.2 inherit peer-policy IBGP-POLICY
```

Inheritance structure:

```text
    COMMON-POLICY       NEXT-HOP-POLICY
    send-community       next-hop-self
         |                     |
         | Sequence 10         | Sequence 20
         |                     |
         +----------+----------+
                    |
                    v
               IBGP-POLICY
                    |
                    v
                Neighbor R2
```

R2 inherits both `send-community both` and `next-hop-self`.

---

## Peer Policy Inheritance Sequence Numbers

The `inherit peer-policy` command uses a **sequence number** to determine the order in which inherited templates are evaluated.

Syntax:

```text
R1(config-router-ptmp)# inherit peer-policy <NAME> <SEQUENCE>
```

The lowest sequence number is evaluated first.

**All inherited peer policy templates are evaluated, regardless of their sequence numbers.**

This is different from route-map processing, where a matching sequence normally terminates evaluation.

### Conflicting Settings

Consider:

```text
R1(config)# router bgp 65001
R1(config-router)# template peer-policy POLICY-A
R1(config-router-ptmp)# weight 100
R1(config-router-ptmp)# exit-peer-policy
R1(config-router)# template peer-policy POLICY-B
R1(config-router-ptmp)# weight 200
R1(config-router-ptmp)# exit-peer-policy
R1(config-router)# template peer-policy COMBINED
R1(config-router-ptmp)# inherit peer-policy POLICY-A 10
R1(config-router-ptmp)# inherit peer-policy POLICY-B 20
R1(config-router-ptmp)# exit-peer-policy
```

The evaluation order is:

| Sequence | Template | Setting |
|---|---|---|
| 10 | POLICY-A | `weight 100` |
| 20 | POLICY-B | `weight 200` |

**Final inherited weight: 200**

The later template overrides the earlier setting.

If the sequence numbers were reversed, the final inherited weight would be 100.

### Locally Configured Template Settings

Settings configured directly in the receiving template take precedence over inherited settings.

For example:

```text
R1(config-router)# template peer-policy COMBINED
R1(config-router-ptmp)# inherit peer-policy POLICY-A 10
R1(config-router-ptmp)# inherit peer-policy POLICY-B 20
R1(config-router-ptmp)# weight 300
R1(config-router-ptmp)# exit-peer-policy
```

**Final inherited weight: 300**

The directly configured `weight 300` overrides the values inherited from both templates.

A setting configured directly on the neighbor also takes precedence over an inherited setting of the same type.

For example:

```text
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 2.2.2.2 inherit peer-policy COMBINED
R1(config-router-af)# neighbor 2.2.2.2 weight 400
```

**Final weight for R2: 400**

The precedence for overlapping settings is:

1. Settings configured directly on the neighbor.
2. Settings configured directly in the applied peer policy template.
3. Inherited policy templates, with later-applied settings overriding earlier ones.

---

## Policy Template Inheritance Limits

A neighbor can use:

- One directly applied peer policy template per address family.
- Up to seven additional indirectly inherited policy templates.
- Eight policy templates in total per inheritance hierarchy.

Unlike session templates, policy templates support branching inheritance.

For example:

```text
          POLICY-A        POLICY-B
               \          /
                \        /
                 POLICY-C
                    |
                    v
                 Neighbor
```

The total number of templates in the inheritance hierarchy is limited to eight.

---

## Complete Configuration Example

This example combines peer session and peer policy templates.

### Topology

```text
                      AS 65001

             R2                    R3
         Lo0: 2.2.2.2          Lo0: 3.3.3.3
               \                  /
                \                /
                 \              /
                      R1
                 Lo0: 1.1.1.1
```

R1 establishes IBGP sessions with R2 and R3.

Requirements:

1. Use Loopback0 as the BGP update source.
2. Configure keepalive and hold timers of 10 and 30 seconds.
3. Advertise standard and extended BGP communities.
4. Configure `next-hop-self` for both neighbors.
5. Configure the neighbors as route reflector clients.

Assume the interfaces and IGP reachability are already configured.

### R1 Configuration

```text
R1(config)# router bgp 65001
R1(config-router)# no bgp default ipv4-unicast
!
! Session template
R1(config-router)# template peer-session IBGP-SESSION
R1(config-router-stmp)# update-source Loopback0
R1(config-router-stmp)# timers 10 30
R1(config-router-stmp)# exit-peer-session
!
! Common policy template
R1(config-router)# template peer-policy COMMON-POLICY
R1(config-router-ptmp)# send-community both
R1(config-router-ptmp)# next-hop-self
R1(config-router-ptmp)# exit-peer-policy
!
! Route reflector client policy
R1(config-router)# template peer-policy RR-CLIENT-POLICY
R1(config-router-ptmp)# inherit peer-policy COMMON-POLICY 10
R1(config-router-ptmp)# route-reflector-client
R1(config-router-ptmp)# exit-peer-policy
!
! Neighbors
R1(config-router)# neighbor 2.2.2.2 remote-as 65001
R1(config-router)# neighbor 2.2.2.2 inherit peer-session IBGP-SESSION
R1(config-router)# neighbor 3.3.3.3 remote-as 65001
R1(config-router)# neighbor 3.3.3.3 inherit peer-session IBGP-SESSION
!
! IPv4 unicast address family
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 2.2.2.2 activate
R1(config-router-af)# neighbor 3.3.3.3 activate
R1(config-router-af)# neighbor 2.2.2.2 inherit peer-policy RR-CLIENT-POLICY
R1(config-router-af)# neighbor 3.3.3.3 inherit peer-policy RR-CLIENT-POLICY
```

Both neighbors inherit:

| Setting | Template |
|---|---|
| `update-source Loopback0` | IBGP-SESSION |
| `timers 10 30` | IBGP-SESSION |
| `send-community both` | COMMON-POLICY (indirect) |
| `next-hop-self` | COMMON-POLICY (indirect) |
| `route-reflector-client` | RR-CLIENT-POLICY |

The remote routers must also configure compatible BGP sessions.

Note that `next-hop-self` does not necessarily rewrite the NEXT_HOP of routes reflected between IBGP clients. Its behavior still follows normal BGP next-hop handling rules.

---

## Peer Groups vs. Peer Templates

| Feature | Peer Groups | Peer Templates |
|---|---|---|
| Share session settings | Yes | Yes |
| Share routing policies | Yes | Yes |
| Separate session and policy settings | No | Yes |
| Multilevel inheritance | No | Yes |
| Session template inheritance | No | Chain-based |
| Policy template inheritance | No | Sequence-number-based |
| Multiple policy templates combined | No | Yes |
| Can use both on one neighbor | No | No |

Peer templates provide more flexibility than traditional peer groups, particularly when neighbors share some settings but require different combinations of other settings.

**A BGP neighbor cannot simultaneously belong to a peer group and inherit peer templates.**

---

## Verification

### Display Session Templates

```text
R1# show ip bgp template peer-session
```

Display a specific template:

```text
R1# show ip bgp template peer-session IBGP-SESSION
```

These commands display configured session templates and their inheritance information.

### Display Policy Templates

```text
R1# show ip bgp template peer-policy
```

Display a specific policy template:

```text
R1# show ip bgp template peer-policy RR-CLIENT-POLICY
```

Display detailed inherited policy information:

```text
R1# show ip bgp template peer-policy RR-CLIENT-POLICY detail
```

The `detail` keyword is particularly useful for identifying inherited settings and their source templates.

### Display Effective Neighbor Policies

```text
R1# show ip bgp neighbors 2.2.2.2 policy
```

For more detailed information:

```text
R1# show ip bgp neighbors 2.2.2.2 policy detail
```

These commands help identify the policies applied directly to a neighbor and those inherited through templates.

### Verify BGP Sessions

```text
R1# show bgp ipv4 unicast summary
R1# show bgp ipv4 unicast neighbors 2.2.2.2
```

Verify the resulting BGP session parameters, route reflector status, and routing policies.

### Examine the Configuration

```text
R1# show running-config | section router bgp
```

The running configuration shows template definitions and inheritance statements, but it does not necessarily expand every inherited setting under each individual neighbor.

Use the template and neighbor policy verification commands to inspect the effective configuration.

---

## Common Issues

| Problem | Possible cause |
|---|---|
| Neighbor does not inherit session settings | Missing or incorrect `inherit peer-session` |
| Neighbor definition is rejected | Missing explicit `remote-as` |
| Policy settings do not apply | Template not inherited under the correct address family |
| No IPv4 unicast routes exchanged | Neighbor not activated for IPv4 unicast |
| Unexpected inherited setting | Another template overrides it |
| Unexpected policy precedence | Incorrect policy inheritance sequence numbers |
| Cannot inherit multiple session templates | Only one direct session template inheritance is supported |
| Too many inherited templates | Eight-template inheritance limit exceeded |
| Peer template configuration rejected | Neighbor already belongs to a peer group |

---

## Command Summary

| Command | Purpose |
|---|---|
| `template peer-session NAME` | Create a peer session template |
| `template peer-policy NAME` | Create a peer policy template |
| `exit-peer-session` | Exit session-template configuration mode |
| `exit-peer-policy` | Exit policy-template configuration mode |
| `neighbor IP inherit peer-session NAME` | Apply a session template to a neighbor |
| `neighbor IP inherit peer-policy NAME` | Apply a policy template under an address family |
| `inherit peer-session NAME` | Inherit another session template |
| `inherit peer-policy NAME SEQ` | Inherit a policy template with an evaluation sequence |
| `show ip bgp template peer-session` | Display session templates |
| `show ip bgp template peer-policy` | Display policy templates |
| `show ip bgp template peer-policy NAME detail` | Display detailed policy inheritance |
| `show ip bgp neighbors IP policy detail` | Display effective neighbor policy information |

---

## Key Points

- Peer templates separate BGP **session parameters** from **address-family-specific policies**.
- A neighbor can directly inherit **one session template and one policy template per address family**.
- Session templates support chain-based inheritance with a maximum of eight templates.
- Policy templates support multiple inheritance using sequence numbers.
- **Lower policy sequence numbers are evaluated first.**
- **All policy inheritance sequences are evaluated**, unlike route-map processing.
- Later-applied policy settings override earlier inherited settings.
- Directly configured neighbor settings take precedence over corresponding inherited settings.
- Peer groups and peer templates cannot be combined on the same neighbor.

---

## Cisco Documentation

- [Configuring a Basic BGP Network, Cisco IOS XE 17.x](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-routing/b-ip-routing/m_irg-basic-net-0.html)
- [Cisco IOS XE BGP Configuration Guide](https://www.cisco.com/c/en/us/td/docs/switches/lan/c9000/lyr3-fwd/bgp/bgp-configuration-guide/routing-bgp.html)
- [Cisco IOS BGP Command Reference: Inherit Peer-Session and Peer-Policy](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_bgp/command/irg-cr-book/bgp-c1.html)
