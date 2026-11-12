# 1.5.a (i) Peer Groups and Templates

**BGP peer groups** and **peer templates** allow multiple BGP neighbors to share common configuration settings.

Without these features, settings such as `update-source`, `timers`, `route-map`, and `send-community` must be configured separately for each neighbor.

Both methods simplify BGP configuration, reduce duplication, and make large BGP deployments easier to manage.

Cisco IOS XE supports two primary approaches:

1. **Peer groups:** Multiple neighbors share configuration defined under a named peer group.
2. **Peer templates:** Neighbors inherit reusable session and routing policy configurations from templates.

Peer templates provide greater flexibility through **configuration inheritance**.

---

## BGP Peer Groups

A **BGP peer group** is a named group of neighbors that share common BGP configuration.

Instead of configuring identical settings on every neighbor, the settings are configured once on the peer group.

For example:

```text
R1(config)# router bgp 65001
R1(config-router)# neighbor IBGP-PEERS peer-group
R1(config-router)# neighbor IBGP-PEERS remote-as 65001
R1(config-router)# neighbor IBGP-PEERS update-source Loopback0
R1(config-router)# neighbor 2.2.2.2 peer-group IBGP-PEERS
R1(config-router)# neighbor 3.3.3.3 peer-group IBGP-PEERS
```

Both neighbors inherit:

- `remote-as 65001`
- `update-source Loopback0`

Additional settings can be configured on the peer group, including address-family-specific routing policies.

Peer groups simplify configuration, particularly when many neighbors require identical settings.

A peer group can also contain EBGP neighbors from different ASNs if the `remote-as` is configured individually on each neighbor rather than on the group.

See [BGP Peer Groups](peer-groups.md) for detailed configuration and verification.

---

## BGP Peer Templates

**BGP peer templates** provide a more flexible way to share configuration between neighbors.

Unlike peer groups, peer templates separate BGP configuration into two categories.

| Template type | Purpose | Example settings |
|---|---|---|
| **Peer session template** | Common BGP session parameters | `remote-as`, `update-source`, `timers`, `password`, `ebgp-multihop` |
| **Peer policy template** | Address-family-specific routing policies | `route-map`, `prefix-list`, `send-community`, `next-hop-self`, `route-reflector-client` |

### Peer Session Templates

A **peer session template** defines common BGP session parameters.

For example:

```text
R1(config)# router bgp 65001
R1(config-router)# template peer-session IBGP-SESSION
R1(config-router-stmp)# update-source Loopback0
R1(config-router-stmp)# timers 10 30
R1(config-router-stmp)# exit-peer-session
```

Apply the template to neighbors:

```text
R1(config-router)# neighbor 2.2.2.2 remote-as 65001
R1(config-router)# neighbor 2.2.2.2 inherit peer-session IBGP-SESSION
R1(config-router)# neighbor 3.3.3.3 remote-as 65001
R1(config-router)# neighbor 3.3.3.3 inherit peer-session IBGP-SESSION
```

Both neighbors inherit the configured `update-source` and BGP timers.

### Peer Policy Templates

A **peer policy template** defines common BGP routing policies for an address family.

For example:

```text
R1(config)# router bgp 65001
R1(config-router)# template peer-policy IBGP-POLICY
R1(config-router-ptmp)# next-hop-self
R1(config-router-ptmp)# send-community both
R1(config-router-ptmp)# exit-peer-policy
```

Apply the template under the IPv4 unicast address family:

```text
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 2.2.2.2 inherit peer-policy IBGP-POLICY
R1(config-router-af)# neighbor 3.3.3.3 inherit peer-policy IBGP-POLICY
```

Both neighbors inherit the configured routing policy.

This example assumes the neighbors have already been defined and activated for IPv4 unicast.

**A neighbor can inherit both a peer session template and a peer policy template.** However, it cannot simultaneously use peer groups and peer templates.

See [BGP Peer Templates](peer-templates.md) for detailed configuration and inheritance behavior.

---

## Peer Template Inheritance

One of the main advantages of peer templates is **inheritance**.

A template can inherit settings from another template, allowing common configurations to be reused.

For example:

```text
              COMMON-SESSION
               timers 10 30
                     |
                     | inherit
                     v
               IBGP-SESSION
          update-source Loopback0
                     |
              +------+------+
              |             |
              v             v
           Peer R2        Peer R3
```

Both R2 and R3 inherit settings from `IBGP-SESSION`, which itself inherits the timers from `COMMON-SESSION`.

### Inheritance Rules

| Feature | Peer session templates | Peer policy templates |
|---|---|---|
| Directly inherited templates per neighbor | 1 | 1 |
| Indirectly inherited templates | Up to 7 | Up to 7 |
| Maximum total inherited templates | 8 | 8 |
| Can inherit multiple templates directly within one template | No | Yes |
| Inheritance sequencing | Chain-based | Sequence-number-based |
| Duplicate settings | More directly applied settings take precedence | Later-applied settings take precedence |

For peer policy templates, inherited templates are evaluated in ascending sequence-number order. Unlike route-map sequences, **all inherited policy template sequences are evaluated**.

A setting configured directly in a template takes precedence over the same setting inherited from another template.

---

## BGP Dynamic Update Groups

Historically, BGP peer groups were important for improving the efficiency of BGP UPDATE message generation.

Instead of generating identical UPDATE messages separately for every neighbor, BGP could reuse UPDATE processing for neighbors with common outbound policies.

Modern Cisco IOS XE uses **BGP dynamic update groups**.

The BGP process automatically groups neighbors that have compatible outbound advertisement characteristics, allowing them to share UPDATE generation and replication.

**Dynamic update groups are independent of manually configured BGP peer groups.**

For example:

- Two neighbors with identical outbound policies may belong to the same dynamic update group even if they use different peer templates.
- Neighbors configured with different outbound policies may belong to different dynamic update groups.
- Dynamic update groups are calculated automatically and do not require manual configuration.

This separation allows peer templates to provide configuration flexibility without sacrificing UPDATE generation efficiency.

Verify dynamic update groups with:

```text
R1# show ip bgp update-group
R1# show ip bgp replication
```

---

## Peer Groups vs. Peer Templates

| Feature | Peer Groups | Peer Templates |
|---|---|---|
| Share common configuration | Yes | Yes |
| Support session settings | Yes | Yes |
| Support routing policy settings | Yes | Yes |
| Separate session and policy configuration | No | Yes |
| Support template inheritance | No | Yes |
| Support multilevel inheritance | No | Yes |
| Compatible with dynamic update groups | Yes | Yes |
| Configuration flexibility | Lower | Higher |

**Peer templates are generally preferred for complex BGP deployments** because they support modular configuration and inheritance.

Peer groups remain useful when several neighbors share straightforward, common settings.

---

## Verification Commands

| Command | Purpose |
|---|---|
| `show ip bgp summary` | Display BGP neighbors and session states |
| `show ip bgp neighbors` | Display detailed neighbor configuration and status |
| `show ip bgp neighbors <IP> policy` | Display policies configured directly or inherited by a neighbor |
| `show ip bgp template peer-session` | Display configured peer session templates |
| `show ip bgp template peer-policy` | Display configured peer policy templates |
| `show ip bgp template peer-policy <NAME> detail` | Display detailed policy template inheritance |
| `show ip bgp update-group` | Display dynamically calculated update groups |
| `show running-config \| section router bgp` | Examine configured peer groups, templates, and neighbors |

---

## Key Points

- **Peer groups** share common BGP configuration among multiple neighbors.
- **Peer session templates** define session-level settings.
- **Peer policy templates** define address-family-specific routing policies.
- **Peer templates support inheritance**, allowing modular and reusable configuration.
- A neighbor can use both session and policy templates, but **cannot combine peer groups with peer templates**.
- **Dynamic update groups** automatically optimize UPDATE message generation independently of peer group or template configuration.

---

## Cisco Documentation

- [Configuring a Basic BGP Network, Cisco IOS XE 17.x](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-routing/b-ip-routing/m_irg-basic-net-0.html)
- [Cisco IOS BGP Command Reference: Peer Groups and Templates](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_bgp/command/irg-cr-book/bgp-m1.html)
- [Cisco IOS BGP Command Reference: Template Inheritance](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_bgp/command/irg-cr-book/bgp-c1.html)
