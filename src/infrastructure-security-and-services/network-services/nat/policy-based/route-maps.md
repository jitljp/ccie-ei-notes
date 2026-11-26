# Route-Map-Based NAT and Conditional Translation

A **route map** allows a NAT router to select a translation according to conditions other than just the inside local IP address. For example, the same inside host can use different inside global addresses depending on the destination or the outgoing interface.

Route maps are particularly important for NAT because they affect not only **how traffic is classified**, but also **the type of translation entries created**.

Unlike ACL-based dynamic NAT, route-map-based dynamic NAT does not create a general-purpose **simple translation entry** unless the `reversible` keyword is configured. This allows different flows from the same inside local address to use different inside global addresses, depending on the configured policy.

This page focuses on route map evaluation, translation lookup, NAT exemption, conditional static mappings, and the `reversible` keyword. The following pages apply these concepts to more specialized designs.

---

## ACL-Based vs. Route-Map-Based NAT

Consider the following two alternative configurations:

```text
Local_Router(config)# access-list 10 permit 10.1.1.0 0.0.0.255
Local_Router(config)# ip nat pool PUBLIC 7.7.7.50 7.7.7.60 prefix-length 24
Local_Router(config)# ip nat inside source list 10 pool PUBLIC
```

Instead of referencing ACL 10 directly, reference it in a route map:

```text
Local_Router(config)# route-map NAT-POLICY permit 10
Local_Router(config-route-map)# match ip address 10
Local_Router(config-route-map)# exit
Local_Router(config)# no ip nat inside source list 10 pool PUBLIC
Local_Router(config)# ip nat inside source route-map NAT-POLICY pool PUBLIC
```

Clear active dynamic translations before replacing an in-use rule.

The two rules classify the same source network and use the same NAT pool. However, their translation behavior differs.

| NAT rule | Classification | Translation type |
|---|---|---|
| `list ... pool ...` | ACL | Simple address binding (with associated extended flow entries) |
| `list ... pool ... overload` | ACL | Extended flow translation |
| `route-map ... pool ...` | Route map | Extended flow translation |
| `route-map ... pool ... overload` | Route map | Extended flow translation with overload |

**Extended translation entries are not exclusive to PAT.** Route-map-based address-only dynamic NAT also uses extended entries. The absence of `overload` still matters: the rule does not enable ordinary many-to-one PAT or port overloading merely because a translation entry contains port information.

See [NAT Translation Tables and Verification](../verification.md) for the general distinction between simple and extended translations.

---

## Route Map Match Conditions

The principal match commands used for NAT on IOS XE routers are:

| Command | Information matched |
|---|---|
| `match ip address <acl>` | Packet fields permitted by the specified ACL |
| `match interface <interface>` | Selected outgoing interface |
| `match ip next-hop <acl>` | Selected forwarding next-hop IP address |

These are NAT-specific interpretations of route map match commands. For example, `match ip address` in a BGP redistribution route map matches route information, whereas in a NAT route map it classifies packets.

Not every route map match or set command is supported by every feature. Do not assume that a command available under route-map configuration mode is meaningful to NAT. These examples target routers; some Catalyst switching platforms support only address-based NAT route map matching.

### Matching Packet Addresses

A standard ACL matches the source IP address of inside-to-outside traffic:

```text
Local_Router(config)# ip access-list standard NAT-SOURCES
Local_Router(config-std-nacl)# permit 10.1.1.0 0.0.0.255
Local_Router(config-std-nacl)# exit
Local_Router(config)# route-map NAT-POLICY permit 10
Local_Router(config-route-map)# match ip address NAT-SOURCES
```

An extended ACL can distinguish traffic by destination, protocol, and applicable transport-layer ports:

```text
Local_Router(config)# ip access-list extended NAT-HTTPS
Local_Router(config-ext-nacl)# permit tcp 10.1.1.0 0.0.0.255 host 3.3.3.7 eq 443
Local_Router(config-ext-nacl)# exit
Local_Router(config)# route-map NAT-HTTPS permit 10
Local_Router(config-route-map)# match ip address NAT-HTTPS
```

The second route map matches HTTPS traffic from the inside network to RH7. It does not match ICMP Echo Requests or SSH traffic to RH7. The ACL describes the packet before inside source translation.

### Matching the Outgoing Interface

```text
Local_Router(config)# route-map NAT-ISP1 permit 10
Local_Router(config-route-map)# match ip address NAT-SOURCES
Local_Router(config-route-map)# match interface Ethernet0/1
```

The packet must have a matching source address **and** be forwarded through `Ethernet0/1`.

`match interface` matches the **outgoing interface**, not the incoming interface.

### Matching the Next-Hop Address

```text
Local_Router(config)# ip access-list standard ISP1-NEXT-HOP
Local_Router(config-std-nacl)# permit host 7.7.7.1
Local_Router(config-std-nacl)# exit
Local_Router(config)# route-map NAT-VIA-ISP1 permit 10
Local_Router(config-route-map)# match ip address NAT-SOURCES
Local_Router(config-route-map)# match ip next-hop ISP1-NEXT-HOP
```

This route map matches inside traffic whose selected forwarding next hop is `7.7.7.1`. The ACL `ISP1-NEXT-HOP` matches the next hop, not the packet's source or destination IP address.

### Combining Match Conditions

Different match types in the same sequence use **AND** logic. Multiple alternatives in a single supported match command use **OR** logic. For example:

```text
route-map NAT-POLICY permit 10
 match ip address NAT-SOURCES
 match interface Ethernet0/1 Ethernet0/2
```

The packet must match `NAT-SOURCES` and be forwarded through either `Ethernet0/1` or `Ethernet0/2`.

Separate permit sequences can also express alternatives. A sequence with no match conditions matches all remaining traffic.

---

## Route Map Evaluation: Permit and Deny

Route map sequences are evaluated in ascending sequence-number order until a matching sequence is found.

There are two distinct permit/deny decisions:

1. The ACL determines whether a particular `match ip address` condition is satisfied.
2. The route map determines whether a matching packet is eligible for the associated NAT rule.

| ACL result | Route map sequence | Result |
|---|---|---|
| Permit | Permit | Select the rule if all other conditions in the sequence match |
| Permit | Deny | Reject this NAT rule if all other conditions in the sequence match |
| Deny (explicit or implicit) | Permit | Sequence does not match; evaluate subsequent sequences |
| Deny (explicit or implicit) | Deny | Sequence does not match; evaluate subsequent sequences |

If no sequence matches, the route map rejects the packet for this NAT rule.

**An ACL deny does not mean that a route-map deny sequence matches.** A deny sequence needs its match conditions to succeed, which requires an ACL permit for `match ip address`.

### Example: Incorrect Exemption

```text
ip access-list extended EXEMPT
 deny ip 10.1.1.0 0.0.0.255 host 3.3.3.8
!
ip access-list standard NAT-SOURCES
 permit 10.1.1.0 0.0.0.255
!
route-map NAT-POLICY deny 10
 match ip address EXEMPT
!
route-map NAT-POLICY permit 20
 match ip address NAT-SOURCES
```

A packet from LH5 to RH8 is denied by ACL `EXEMPT`, so sequence 10 **does not match**. Sequence 20 then matches, selecting the associated NAT rule.

To make sequence 10 exclude RH8 traffic, use an ACL **permit** statement that identifies that traffic:

```text
ip access-list extended EXEMPT
 permit ip 10.1.1.0 0.0.0.255 host 3.3.3.8
!
route-map NAT-POLICY deny 10
 match ip address EXEMPT
!
route-map NAT-POLICY permit 20
 match ip address NAT-SOURCES
```

The ACL is a classifier in this context, not an interface packet filter.

A route map that denies a packet only excludes that packet from the associated NAT rule. Another configured NAT rule might still translate it. If no translation applies, ordinary forwarding can continue without translation, subject to routing and security policy.

---

## NAT Route Maps Do Not Need a Set Command

In PBR, a route map usually contains a `set` command that specifies the desired forwarding behavior. A NAT route map does not need such a command, because the associated NAT statement defines the action.

For example:

```text
route-map NAT-POLICY permit 10
 match ip address NAT-SOURCES
!
ip nat inside source route-map NAT-POLICY pool PUBLIC overload
```

The route map answers **which traffic?** The NAT statement answers **which translation?**

A NAT route map is referenced by `ip nat inside source route-map`, not applied to an interface with `ip policy route-map`.

Also, the sequence number belongs to the route map itself. It is not a global priority among different NAT statements. A sequence numbered 10 in one route map does not necessarily take precedence over a sequence numbered 20 in another route map.

---

## Translation Lookup vs. Policy Evaluation

For dynamic NAT, the router first checks for an applicable existing translation. If one exists, it can be used without reselecting a rule using its ACL or route map.

The following is a logical model for inside-to-outside traffic:

```text
                Packet arrives on NAT inside interface
                                  |
                                  v
                     Determine forwarding path
                                  |
                                  v
                 Matching NAT translation exists?
                         /                  \
                       Yes                   No
                        |                     |
                        v                     v
                 Reuse translation     Evaluate NAT rules
                        |                     |
                        |            Eligible rule selected?
                        |                 /         \
                        |               Yes          No
                        |                |            |
                        |                v            v
                        |         Create translation  No NAT
                        |                |            |
                        +----------------+------------+
                                         |
                                         v
                              Continue forwarding
```

This diagram assumes that any selected rule can successfully allocate its translation. Pool exhaustion or other allocation failures can prevent forwarding; do not treat allocation failure as an intentional NAT exemption.

### Why Simple Entries Can Cause Unexpected Results

Suppose an ACL-based dynamic NAT rule creates this simple binding:

```text
Pro  Inside global    Inside local    Outside local    Outside global
---  7.7.7.50         10.1.1.5        ---              ---
```

The entry contains no outside endpoint information. If LH5 sends traffic to a different outside host, that existing binding can still match the traffic.

A second ACL that selects a different pool for the new destination might never be consulted. Even an ACL that excludes the new destination from NAT does not necessarily prevent an existing simple binding from being reused.

**An extended ACL does not automatically cause an extended NAT translation.** When used directly with ordinary dynamic NAT (without overload), it can still produce a simple binding.

With a route-map-based rule, an extended entry can distinguish traffic to different outside endpoints. For example, using illustrative TCP port numbers:

```text
Pro  Inside global     Inside local      Outside local    Outside global
tcp  7.7.7.50:51000    10.1.1.5:51000    3.3.3.7:23       3.3.3.7:23
tcp  7.7.7.60:51001    10.1.1.5:51001    3.3.3.8:23       3.3.3.8:23
```

When the second flow does not match an existing extended translation, NAT evaluates its policies and can select a different pool. Neither translation requires `overload`, and the source ports in this example remain unchanged.

The same principle explains why route-map-based NAT is useful for conditional translation and traffic engineering.

---

## Lab Topology

We will continue using the topology from the static and dynamic NAT sections.

```text
       INSIDE NETWORK                         OUTSIDE NETWORK

  LH5: 10.1.1.5/24
          |
          |                   Local_Router                    ISP1
          +---- LAN ---------- E0/0    E0/1 ---------------- E0/1
          |                10.1.1.1   7.7.7.2              7.7.7.1
          |                NAT inside NAT outside              |
  LH6: 10.1.1.6/24                                  E0/0: 3.3.3.1/24
                                                               |
                                                          Outside LAN
                                                               |
                                                        +------+------+
                                                        |             |
                                                  RH7 3.3.3.7   RH8 3.3.3.8
```

LH5 and LH6 use `10.1.1.1` as their default gateway. RH7 and RH8 use `3.3.3.1`.

On Local_Router, configure the NAT interface roles and routing if not already configured:

```text
Local_Router(config)# interface Ethernet0/0
Local_Router(config-if)# ip nat inside
Local_Router(config-if)# exit
Local_Router(config)# interface Ethernet0/1
Local_Router(config-if)# ip nat outside
Local_Router(config-if)# exit
Local_Router(config)# ip route 0.0.0.0 0.0.0.0 7.7.7.1
```

To eliminate dependence on ARP resolution for the translated addresses, configure explicit host routes on ISP1. Also configure a route to the inside subnet so that untranslated traffic can succeed during the NAT exemption exercise.

```text
ISP1(config)# ip route 7.7.7.50 255.255.255.255 7.7.7.2
ISP1(config)# ip route 7.7.7.60 255.255.255.255 7.7.7.2
ISP1(config)# ip route 7.7.7.70 255.255.255.255 7.7.7.2
ISP1(config)# ip route 10.1.1.0 255.255.255.0 7.7.7.2
```

The last route is for lab observation only. In an actual Internet access design, untranslated private addresses generally lack return reachability.

Remove any previous NAT rules that would interfere with the exercises. Treat each exercise as an alternative configuration, not as an additional set of rules to leave enabled simultaneously.

---

## Lab 1: NAT Exemption with a Route Map

### Requirements

- Translate traffic from LH5 and LH6 to RH7 using interface-based PAT.
- Exempt traffic from LH5 and LH6 to RH8 from NAT.
- Do not use packet filtering to implement the exemption.

### Configuration

```text
Local_Router(config)# ip access-list extended EXEMPT-RH8
Local_Router(config-ext-nacl)# permit ip 10.1.1.0 0.0.0.255 host 3.3.3.8
Local_Router(config-ext-nacl)# exit
!
Local_Router(config)# ip access-list extended TRANSLATE-RH7
Local_Router(config-ext-nacl)# permit ip 10.1.1.0 0.0.0.255 host 3.3.3.7
Local_Router(config-ext-nacl)# exit
!
Local_Router(config)# route-map CONDITIONAL-NAT deny 10
Local_Router(config-route-map)# match ip address EXEMPT-RH8
Local_Router(config-route-map)# exit
!
Local_Router(config)# route-map CONDITIONAL-NAT permit 20
Local_Router(config-route-map)# match ip address TRANSLATE-RH7
Local_Router(config-route-map)# exit
!
Local_Router(config)# ip nat inside source route-map CONDITIONAL-NAT interface Ethernet0/1 overload
```

In this particular example, the explicit deny sequence is redundant because the permit sequence matches only RH7. It is included to demonstrate the deny logic.

For a general Internet PAT rule with RH8 as an exception, replace `TRANSLATE-RH7` with a standard ACL permitting `10.1.1.0/24`. Sequence 10 is then necessary to exclude RH8.

### Verification

```text
LH5# ping 3.3.3.7
LH5# ping 3.3.3.8
```

Both pings should succeed because ISP1 has a route to the inside network. However, their source addresses as seen on the outside differ:

| Destination | NAT rule result | Source on the outside |
|---|---|---|
| RH7 (`3.3.3.7`) | Permit sequence 20 | `7.7.7.2` |
| RH8 (`3.3.3.8`) | Deny sequence 10 | `10.1.1.5` |

Immediately check:

```text
Local_Router# show ip nat translations
Local_Router# show route-map CONDITIONAL-NAT
Local_Router# show access-lists
```

An ICMP translation for the ping to RH7 should be present if it has not aged out. No translation should have been created for the ping to RH8. A packet capture on the outside link provides direct confirmation of the source addresses.

If the ping to RH8 fails after removing the route to `10.1.1.0/24` from ISP1, that does not mean the route map dropped the packet. It means the untranslated traffic lacks a return path.

### Alternative: ACL-Based PAT

For this simple exemption, a route map is not strictly necessary. An extended ACL referenced directly by a PAT rule can accomplish the same classification:

```text
ip access-list extended NAT-ACL
 deny ip 10.1.1.0 0.0.0.255 host 3.3.3.8
 permit ip 10.1.1.0 0.0.0.255 host 3.3.3.7
!
ip nat inside source list NAT-ACL interface Ethernet0/1 overload
```

Use this instead of, not in addition to, the route-map-based rule. The route map becomes more valuable for additional match criteria and non-overloaded destination-dependent translations.

---

## Lab 2: Compare ACL-Based and Route-Map-Based Dynamic NAT

The objective is to demonstrate the consequences of translation lookup when using two pools without `overload`.

### Part A: ACL-Based Dynamic NAT

Remove the NAT rule from Lab 1 after clearing dynamic translations. Configure:

```text
Local_Router# clear ip nat translation *
Local_Router# configure terminal
Local_Router(config)# no ip nat inside source route-map CONDITIONAL-NAT interface Ethernet0/1 overload
!
Local_Router(config)# ip access-list extended TO-RH7
Local_Router(config-ext-nacl)# permit ip host 10.1.1.5 host 3.3.3.7
Local_Router(config-ext-nacl)# exit
Local_Router(config)# ip access-list extended TO-RH8
Local_Router(config-ext-nacl)# permit ip host 10.1.1.5 host 3.3.3.8
Local_Router(config-ext-nacl)# exit
!
Local_Router(config)# ip nat pool POOL-A 7.7.7.50 7.7.7.50 prefix-length 24
Local_Router(config)# ip nat pool POOL-B 7.7.7.60 7.7.7.60 prefix-length 24
Local_Router(config)# ip nat inside source list TO-RH7 pool POOL-A
Local_Router(config)# ip nat inside source list TO-RH8 pool POOL-B
```

Neither rule uses `overload`.

On LH5, initiate a session to RH7 first, then initiate a session to RH8 while the first binding remains active. Pings can be used, but keeping TCP sessions open makes it easier to inspect entries before they age out. If using Telnet, enable a reachable TCP listener on both outside hosts.

```text
LH5# ping 3.3.3.7
LH5# ping 3.3.3.8
```

Inspect the translations immediately:

```text
Local_Router# show ip nat translations
Local_Router# show ip nat statistics
```

The simple binding created by the first flow is expected to be reused for traffic to RH8. Thus, both destinations can see LH5 as `7.7.7.50`, despite the second ACL selecting POOL-B for new mappings to RH8.

Repeat the exercise after clearing translations, but contact RH8 first. Observe whether the binding to `7.7.7.60` is then reused for RH7.

### Part B: Route-Map-Based Dynamic NAT

Stop active traffic, clear translations, and replace the ACL-based rules with route-map-based rules:

```text
Local_Router# clear ip nat translation *
Local_Router# configure terminal
Local_Router(config)# no ip nat inside source list TO-RH7 pool POOL-A
Local_Router(config)# no ip nat inside source list TO-RH8 pool POOL-B
!
Local_Router(config)# route-map NAT-RH7 permit 10
Local_Router(config-route-map)# match ip address TO-RH7
Local_Router(config-route-map)# exit
Local_Router(config)# route-map NAT-RH8 permit 10
Local_Router(config-route-map)# match ip address TO-RH8
Local_Router(config-route-map)# exit
!
Local_Router(config)# ip nat inside source route-map NAT-RH7 pool POOL-A
Local_Router(config)# ip nat inside source route-map NAT-RH8 pool POOL-B
```

Repeat the pings in both orders. The expected translated addresses are now independent of which destination was contacted first:

| Traffic | Expected inside global address |
|---|---|
| LH5 to RH7 | `7.7.7.50` |
| LH5 to RH8 | `7.7.7.60` |

The extended translation entries distinguish the two destinations. Use `show ip nat translations verbose` and an outside packet capture to confirm the result. Output formatting and additional internal entries can differ by release.

Further destination-dependent designs are covered in [Destination-Based NAT and Traffic Engineering](traffic-engineering.md).

---

## Conditional Static NAT

A route map can also be associated with a static inside source mapping. Unlike dynamic NAT, the inside local and inside global addresses are explicitly specified.

```text
ip nat inside source static <inside-local> <inside-global> route-map <name>
```

### Example: Multiple Mappings for One Inside Host

Using the ACLs and route maps from Lab 2, remove the dynamic NAT rules and configure these static mappings instead:

```text
Local_Router# clear ip nat translation *
Local_Router# configure terminal
Local_Router(config)# no ip nat inside source route-map NAT-RH7 pool POOL-A
Local_Router(config)# no ip nat inside source route-map NAT-RH8 pool POOL-B
!
Local_Router(config)# ip nat inside source static 10.1.1.5 7.7.7.50 route-map NAT-RH7
Local_Router(config)# ip nat inside source static 10.1.1.5 7.7.7.60 route-map NAT-RH8
```

When configuring multiple static inside source NAT mappings for the same inside local address, the `route-map` keyword allows conditional selection between the mappings without requiring the `extendable` keyword.

| Inside local | Destination | Selected inside global |
|---|---|---|
| `10.1.1.5` | `3.3.3.7` | `7.7.7.50` |
| `10.1.1.5` | `3.3.3.8` | `7.7.7.60` |

The translated address is predetermined in each statement rather than allocated from a pool. Verify actual outgoing packets, not just the configured static entries.

On platforms that support route maps on static TCP/UDP mappings, the same principle can be applied to a specific transport-layer endpoint. Check the available options with `?` after the local and global address/port arguments. Do not assume all static PAT or network NAT variants support the same route-map options.

### Interaction with Dynamic NAT and PAT

An unconditional static mapping normally takes precedence over ordinary dynamic inside source NAT for its associated address. With conditional static NAT, the route map must also select the intended traffic for the mapping.

Do not rely on the order of statements in the running configuration to resolve competing route-map NAT rules. Cisco's classic IOS documentation describes cases where the relative names of static and dynamic NAT route maps influence precedence. Platform implementations differ, so avoid overlapping match criteria unless intentionally testing precedence.

For predictable behavior, explicitly exclude traffic intended for a conditional static rule from a competing dynamic PAT policy. Use distinct global addresses for static address mappings and dynamic address pools unless the design specifically supports sharing those resources.

---

## The reversible Keyword

For an ordinary route-map-based dynamic NAT rule, the initial traffic creates a flow-specific extended entry. A reply matching that entry is translated in the reverse direction without any special keyword.

However, an outside host initiating a **new** connection to the inside global address is a different case. A new inbound flow does not necessarily match the existing extended translation.

The `reversible` keyword enables new outside-initiated sessions associated with an existing route-map-based dynamic address allocation, provided the reversed packet matches the policy.

```text
ip nat inside source route-map <name> pool <name> reversible
```

### Important Restrictions

For the dynamic NAT outside-to-inside feature:

- An initial inside-to-outside session must first establish the dynamic address allocation.
- The ACL is written in the **inside-to-outside** direction, even when enabling new outside-initiated sessions.
- A new outside-to-inside flow must satisfy that policy in reverse.
- `match interface` and `match ip next-hop` are not supported for reversible route maps.
- This dynamic outside-initiation feature is for address NAT, not overloaded PAT.

**`reversible` is not required for ordinary reply traffic.** It is used to enable additional outside-initiated flows, not to make NAT bidirectional for an existing flow.

### Lab 3: Dynamic NAT with reversible

Remove the conditional static mappings from the previous exercise (stop active traffic and clear flow entries first). For simplicity, reuse only the policy matching LH5 to RH7:

```text
ip access-list extended REVERSIBLE-ACL
 permit ip host 10.1.1.5 host 3.3.3.7
!
route-map REVERSIBLE-NAT permit 10
 match ip address REVERSIBLE-ACL
!
ip nat pool REVERSIBLE-POOL 7.7.7.70 7.7.7.70 prefix-length 24
ip nat inside source route-map REVERSIBLE-NAT pool REVERSIBLE-POOL
```

First, test **without** `reversible`:

1. Initiate traffic from LH5 to RH7 to create the mapping. Keep the traffic active or test before the translation expires.
2. Confirm that normal replies to the original flow are translated correctly.
3. From RH7, initiate a separate TCP connection to `7.7.7.70` on a port that is listening on LH5. This is a new outside-initiated flow, not a reply to the original flow.

For the baseline rule, the new inbound flow should not be translated just because an extended entry already exists for the original flow.

Stop traffic, clear dynamic translations, and modify the rule:

```text
Local_Router# clear ip nat translation *
Local_Router# configure terminal
Local_Router(config)# no ip nat inside source route-map REVERSIBLE-NAT pool REVERSIBLE-POOL
Local_Router(config)# ip nat inside source route-map REVERSIBLE-NAT pool REVERSIBLE-POOL reversible
```

Repeat the tests, including initiating traffic from LH5 first. The new connection from RH7 can now be translated to LH5. Try the same new connection from RH8: it does not satisfy the reversed route map condition and should not create a translation under this rule.

| Traffic | Without `reversible` | With `reversible` |
|---|---|---|
| Initial LH5-to-RH7 flow | Translated | Translated |
| Reply to existing flow | Translated | Translated |
| New RH7-to-LH5 flow via the allocated global address | Not enabled by existing flow alone | Eligible for translation |
| New RH8-to-LH5 flow via the allocated global address | Not selected | Not selected |

Verify with `show ip nat translations` and packet captures. An existing NAT rule, host firewall, missing TCP listener, or incorrect return path can otherwise obscure the result.

### Static NAT and reversible

Cisco also documents static address NAT syntax using a route map with `reversible`:

```text
ip nat inside source static 10.1.1.5 7.7.7.70 route-map REVERSIBLE-NAT reversible
```

The purpose is to permit outside-initiated translation according to the reverse address policy rather than treating the mapping as an unrestricted inbound static address mapping.

**Check support on the target IOS XE image.** Cisco's NAT FAQ and command reference document this static form, but older documentation for the dynamic outside-to-inside feature explicitly excludes static NAT. Do not extrapolate the dynamic allocation prerequisite to a configured static binding, and do not assume identical behavior across releases. Verify both permitted and non-permitted outside sources before relying on the feature.

Conditional NAT is not a substitute for an interface ACL or firewall to protect a published service.

---

## Changing a Route Map with Active Translations

A route map change does not necessarily change existing translations immediately.

For example, suppose a flow from LH5 to RH7 already uses `7.7.7.50`. You modify the policy to select `7.7.7.60` for that traffic.

Existing translation state can continue to match packets from the flow, so the old inside global address may still be used. New flows that do not match existing entries are subject to the new policy.

For controlled lab comparisons:

1. Stop the traffic so translations are not immediately recreated.
2. Clear dynamic NAT translations.
3. Modify the rules or route maps.
4. Generate new traffic and inspect the resulting translations.

```text
Local_Router# clear ip nat translation *
```

This clears dynamic translation state and can disrupt active connections. Configured static mappings are not removed by clearing dynamic translations.

For targeted clearing and aging commands, see [NAT Translation Tables and Verification](../verification.md).

---

## Verification and Troubleshooting

```text
Local_Router# show running-config | include ip nat
Local_Router# show route-map
Local_Router# show access-lists
Local_Router# show ip nat translations
Local_Router# show ip nat translations verbose
Local_Router# show ip nat statistics
Local_Router# show ip route 3.3.3.7
Local_Router# show ip cef 3.3.3.7
```

| Symptom | What to check |
|---|---|
| Wrong inside global address | Existing translation, selected pool, overlapping NAT policies |
| Exempted traffic is translated | ACL permit/deny semantics, later permit sequences, another matching NAT rule, existing binding |
| A route map matches unexpected traffic | Empty match sequence, broad ACL, OR/AND relationships |
| Interface match does not work | Actual forwarding interface, PBR, platform support |
| Next-hop match does not work | Actual selected next hop, ACL contents, platform support |
| New connections fail while older ones work | Pool availability, translation limits, rule changes |
| Outbound translation works but no replies arrive | Route to inside global address, ARP, upstream filtering |
| New inbound sessions fail | Existing dynamic allocation, reversible support, reverse policy, PAT restrictions |

`show route-map` confirms the configured policy, but its counters (including counters labeled policy routing matches) are not reliable proof of NAT classification. NAT may reuse existing state without evaluating the route map. Check the NAT table and actual packets to verify the translation.

In an isolated lab, `debug ip nat` can provide additional translation information. Disable debugging with `undebug all` after the test.

---

## Key Points

- A NAT route map classifies traffic; the associated NAT statement specifies the translation.
- `match ip address` classifies packets using an ACL, while `match interface` and `match ip next-hop` match forwarding information on supporting routers.
- A matching ACL permit satisfies the route map's ACL condition. An ACL deny makes that condition fail.
- A matching route-map deny excludes traffic from that NAT rule; it does not drop the packet or prohibit every other NAT rule from applying.
- A NAT route map does not require a `set` command and is not applied with `ip policy route-map`.
- Route-map-based dynamic NAT creates extended translations even without `overload`.
- An extended ACL used directly by a non-overloaded dynamic NAT rule can create a simple binding that is reused for traffic to different destinations.
- An applicable existing translation can be used without consulting the NAT policy again.
- Route maps can be used to configure conditional static address mappings as well as dynamic NAT and PAT.
- Dynamic NAT's `reversible` keyword permits eligible new outside-initiated sessions after an inside-initiated allocation. It is not needed for normal replies.
- Reversible dynamic NAT does not support overload, interface matching, or next-hop matching.
- NAT classification and address allocation do not replace routing or security policy.

---

## References

- [Cisco: NAT Support for Multiple Pools Using Route Maps](https://www.cisco.com/c/en/us/support/docs/ip/network-address-translation-nat/13739-nat-routemap.html)
- [Cisco IOS XE 17.x: Configuring NAT for IP Address Conservation](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-addressing/b-ip-addressing/m_iadnat-addr-consv-xe.html)
- [Cisco IOS XE: Performance Routing with NAT](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-routing/b-ip-routing/m_pfr-nat-0.html)
- [Cisco: NAT Route Maps Outside-to-Inside Support](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/ipaddr_nat/configuration/15-mt/nat-15-mt-book/iadnat-rmap-outin.html)
- [Cisco: Network Address Translation (NAT) Frequently Asked Questions](https://www.cisco.com/c/en/us/support/docs/ip/network-address-translation-nat/26704-nat-faq-00.html)
- [Cisco IOS IP Addressing Services Command Reference: NAT Commands](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/ipaddr/command/ipaddr-cr-book/ipaddr-i3.html)
- [Cisco IOS XE Catalyst 9600: Configuring Network Address Translation](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9600/software/release/17-17/configuration_guide/ip/b_1717_ip_9600_cg/configuring_network_address_translation.html)
