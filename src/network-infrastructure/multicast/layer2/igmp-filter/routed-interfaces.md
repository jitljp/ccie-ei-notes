# IGMP Filtering on Routed Interfaces

**IGMP filtering on routed interfaces** allows a multicast router to control which groups and source/group channels directly connected receivers can join, and to limit the multicast routing state resulting from those joins.

Unlike the Layer 2 `ip igmp filter` and `ip igmp max-groups` features, these controls operate on the router rather than the switch's IGMP snooping table. `ip igmp access-group` filters IGMP membership requests; `ip igmp limit` restricts **IGMP-driven multicast routing state**.

IOS XE provides two primary commands:

| Command | Function | Configuration Mode |
|---|---|---|
| `ip igmp access-group` | Permits or denies IGMP group or channel memberships using an ACL | Interface |
| `ip igmp limit` | Limits **mroute states resulting from IGMP joins** | Global or interface |

These features are useful for restricting multicast subscriptions and limiting the amount of receiver-driven multicast state a router maintains.

## Layer 2 Filtering vs. Layer 3 Filtering

Consider the following topology:

```text
             Multicast Network
                    |
                  E0/0
                    |
                   R1
              10.3.4.1
                  E0/1
                    |
                   SW1
               VLAN 10
              /       \
           E0/1       E0/2
             |           |
            PC1         PC2
        10.3.4.10  10.3.4.11
```

R1 is a multicast router and IGMP querier for the receiver LAN.

SW1 can use **IGMP snooping** to learn which Layer 2 ports lead toward interested receivers:

```text
SW1 IGMP snooping state:

VLAN 10, 239.1.1.1
  -> Ethernet0/1
  -> Ethernet0/2
```

R1 instead maintains **Layer 3 IGMP state** for the directly connected network:

```text
R1 IGMP membership state:

Interface Ethernet0/1
  (*,239.1.1.1)
```

The router can enforce an admission policy **before accepting new IGMP membership state**:

```text
PC1 / PC2
    |
    | IGMP Membership Report
    v
   SW1
    |
    v
R1 Ethernet0/1
    |
    +-- ip igmp access-group  -> Is this membership allowed?
    |
    +-- ip igmp limit         -> Is there room for the IGMP-driven mroute state?
    |
    v
R1 IGMP membership database
    |
    v
Multicast routing / outgoing-interface state
```

**IGMP snooping is not required** for R1's Layer 3 filtering or admission controls. It is a separate Layer 2 forwarding optimization on SW1.

The Layer 3 commands can be applied to routed Ethernet interfaces or routed VLAN interfaces (SVIs).

## Filtering Group Membership with `ip igmp access-group`

The `ip igmp access-group` interface command associates an IP access control list (ACL) with incoming IGMP membership processing.

Syntax:

```text
R1(config-if)# ip igmp access-group <acl-number-or-name>
```

The ACL identifies which multicast group memberships should be accepted or denied. **It is not an interface packet-filtering ACL** such as `ip access-group ... in`.

The ACL is evaluated against multicast membership information carried in IGMP Reports, rather than simply using the ordinary IP source and destination addresses of an IGMP packet.

### Standard ACLs: Filter by Group Address

A **standard ACL** filters membership based on the **multicast group address (G)**.

For example, permit only groups within `239.1.1.0/24`:

```text
R1(config)# ip access-list standard ALLOWED-GROUPS
R1(config-std-nacl)# permit 239.1.1.0 0.0.0.255
R1(config-std-nacl)# exit
R1(config)# interface Ethernet0/1
R1(config-if)# ip igmp access-group ALLOWED-GROUPS
```

The standard ACL has an implicit `deny any`, so groups outside the permitted range are not admitted by this filter.

```text
IGMP Report for 239.1.1.10
    -> matches permit
    -> membership allowed

IGMP Report for 239.2.2.10
    -> matches implicit deny
    -> membership denied
```

> The address in the standard ACL is the **multicast group**, not the IP address of the host sending the IGMP Report.
> This is easy to confuse, as the group is specified in the **source** of the `access-list` command.

For example:

```text
Host sending Report: 10.3.4.10
Requested group:     239.1.1.10

ACL checks:          239.1.1.10
NOT:                 10.3.4.10
```

### Blacklisting a Group with a Standard ACL

To prohibit one group but allow other groups:

```text
R1(config)# ip access-list standard IGMP-BLOCKED-GROUPS
R1(config-std-nacl)# deny host 239.1.1.100
R1(config-std-nacl)# permit any
R1(config-std-nacl)# exit
R1(config)# interface Ethernet0/1
R1(config-if)# ip igmp access-group IGMP-BLOCKED-GROUPS
```

The resulting group policy is:

```text
239.1.1.100 -> Denied
239.1.1.101 -> Permitted
239.2.2.2   -> Permitted
```

If `permit any` were omitted, the implicit deny would also reject every group not explicitly permitted.
This is typical ACL logic.

## Extended ACLs: Filter IGMPv3 (S,G) Channels

IGMPv3 receivers can request multicast traffic from **specific sources**.

For example:

```text
Group:  232.1.1.1
Source: 10.1.1.10

Requested channel:
(10.1.1.10,232.1.1.1)
```

An **extended ACL** applied with `ip igmp access-group` can match the multicast **source S**, the multicast **group G**, or both.

For IGMPv3 channel filtering, the extended ACL's address fields are interpreted as:

```text
permit/deny igmp <multicast-source-S> <multicast-group-G>
```

**The source address in this ACL means the source of the multicast stream, not the receiver's IP address.**

### Example: Deny One Source for One Group

Suppose PC1 wants both of these SSM channels:

```text
S1 = 10.1.1.10
S2 = 10.1.1.20
G  = 232.1.1.1

(S1,G) = (10.1.1.10,232.1.1.1)
(S2,G) = (10.1.1.20,232.1.1.1)
```

Only S1 should be permitted:

```text
             Multicast Network
                    |
                  E0/0
                    |
                   R1
              10.3.4.1
                  E0/1
                    |
                   SW1
               VLAN 10
              /       \
           E0/1       E0/2
             |           |
            PC1         PC2
        10.3.4.10  10.3.4.11
```

S1 (`10.1.1.10`) and S2 (`10.1.1.20`) are multicast **sources reachable through the Multicast Network**, not the receiver addresses of PC1 or PC2.

Configure an extended ACL denying S2 for group G:

```text
R1(config)# ip access-list extended SSM-FILTER
R1(config-ext-nacl)# deny igmp host 10.1.1.20 host 232.1.1.1
R1(config-ext-nacl)# permit igmp any any
R1(config-ext-nacl)# exit
R1(config)# interface Ethernet0/1
R1(config-if)# ip igmp version 3
R1(config-if)# ip igmp access-group SSM-FILTER
```

> Instead of `deny igmp ...`, you can also use `deny ip ...` in the extended ACL. Both affect IGMP packets.

The expected admission behavior is:

| Receiver Request | Result |
|---|---|
| `(10.1.1.10,232.1.1.1)` | Permitted |
| `(10.1.1.20,232.1.1.1)` | Denied |
| `(10.1.1.20,232.1.1.2)` | Permitted by this ACL |

The final `permit igmp any any` is essential in this **blacklist** example to avoid unintentionally denying all other channels.

You can confirm that R1 denied the (10.1.1.20,232.1.1.1) join with a debug:

```
R1# debug ip igmp 
IGMP debugging is on
R1#
*Oct 10 02:11:39.367: IGMP(0)[default]: Received v3 Report for 1 group on Ethernet0/1 from 0.0.0.0
*Oct 10 02:11:39.367: IGMP(*): Source: 10.1.1.20, Group 232.1.1.1 access denied on Ethernet0/1
```

And use a `show` command to verify that `(10.1.1.10,232.1.1.1)` and `(10.1.1.20,232.1.1.2)` were accepted:

```
R1#show ip igmp membership tracked 
Flags: A  - aggregate, T - tracked
       L  - Local, S - static, V - virtual, R - Reported through v3 
       I - v3lite, U - Urd, M - SSM (S,G) channel 
       1,2,3 - The version of IGMP, the group is in
Channel/Group-Flags: 
       / - Filtering entry (Exclude mode (S,G), Include mode (G))
Reporter:
       <mac-or-ip-address> - last reporter if group is not explicitly tracked
       <n>/<m>      - <n> reporter in include mode, <m> reporter in exclude

 Channel/Group                  Reporter        Uptime   Exp.  Flags  Interface 
 *,232.1.1.1                    1/0             00:01:48 stop  3AT    Et0/1
 10.1.1.10,232.1.1.1            aabb.cc00.4e00  00:01:48 02:13 T      Et0/1
 *,232.1.1.2                    1/0             00:01:19 stop  3AT    Et0/1
 10.1.1.20,232.1.1.2            aabb.cc00.4e00  00:01:19 02:13 T      Et0/1
 *,224.0.1.40                   0/1             04:33:59 02:07 3LAT   Et0/1
 *,224.0.1.40                   10.3.4.1        04:33:59 02:07 3LT    Et0/1
```

### Filtering Sources Within One IGMPv3 Report

An IGMPv3 Membership Report can contain a source list for a group:

```text
MODE_IS_INCLUDE
Group: 232.1.1.1
Sources:
  10.1.1.10
  10.1.1.20
```

When the extended ACL denies only `10.1.1.20` for this group, the router can filter that channel while accepting the permitted one:

```text
Received IGMPv3 membership information:

(10.1.1.10,232.1.1.1) -> Permit
(10.1.1.20,232.1.1.1) -> Deny

Accepted source-specific membership:

(10.1.1.10,232.1.1.1)
```

Unlike IGMPv2 filtering, source-specific IGMPv3 filtering does not necessarily have to accept or reject *all* source records for the group together.

## The Special `(0,G)` Check in Extended IGMP ACLs

Extended ACLs referenced by `ip igmp access-group` have an important behavior that differs from how an ACL is normally used to filter IP packets: **IOS XE performs a preliminary group-level check before evaluating individual IGMPv3 sources**.

Cisco describes this process using a special tuple:

```text
(0.0.0.0,G)
```

Here, `G` is the multicast group address. `0.0.0.0` is a **synthetic wildcard-source value** used in the group-level ACL lookup; it is not the address of the host sending the IGMP Report and is not an actual multicast source.

Cisco also refers to this representation as `(*,G)`. **In this ACL context, that notation does not imply the group is using ASM or that PIM has created a shared tree.** The check can occur when processing an IGMPv3 SSM channel request.

### Two Stages of Evaluation

Suppose PC1 (`10.3.4.10`) sends an IGMPv3 INCLUDE-mode Report to R1 requesting these SSM channels:

```text
Group: 232.1.1.1
Source list: {10.1.1.10, 10.1.1.20}
Requested channels:
  (10.1.1.10,232.1.1.1)
  (10.1.1.20,232.1.1.1)
```

R1 receives the Report on Ethernet0/1 and evaluates the extended ACL as follows:

```text
        IGMPv3 Report received by R1
                     |
                     v
        Check (0.0.0.0,232.1.1.1)
                     |
             Preliminary check
                     |
           +---------+---------+
           |                   |
          DENY                 PERMIT
           |                   |
           v                   v
     Reject group        Evaluate each source
       request                  |
                    +-----------+-----------+
                    |                       |
          (10.1.1.10,G)             (10.1.1.20,G)
                    |                       |
              Permit/Deny              Permit/Deny
```

1. **Preliminary group-level check:** IOS XE evaluates `(0.0.0.0,G)` against the extended ACL, starting at the first entry.
 - If an explicit deny matches, or no entry matches and the lookup reaches the implicit deny, **the group request is rejected** before source-specific checks. 

2. **Individual source checks:** If the group passes the preliminary check, each `(S,G)` channel is evaluated separately. A permitted group-level check **does not automatically permit every source**. Denied sources are excluded from the admitted receiver state.

The **same ACL is evaluated again from the beginning for each lookup**. The first matching ACL entry controls each lookup; the router does not simply continue from the ACE that permitted `(0,G)`.

### What the ACL Address Fields Mean

For this feature, an extended ACL entry such as:

```text
deny igmp host 10.1.1.20 host 232.1.1.1
```

matches the **multicast source** `10.1.1.20` and **multicast group** `232.1.1.1`, not the ordinary IP source and destination addresses in the IGMP packet header.

| ACL source specification | What it matches in the IGMP admission checks |
|---|---|
| `host 0.0.0.0` | The synthetic `(0,G)` lookup specifically |
| `host 10.1.1.20` | The `(10.1.1.20,G)` channel lookup |
| `any` | Any source address, **including `0.0.0.0`** in the preliminary lookup and real multicast source addresses in subsequent lookups |

`any` has **no special IGMP meaning**. It is the normal ACL wildcard match (`0.0.0.0 255.255.255.255`). It matches the preliminary lookup simply because that lookup's source address happens to be `0.0.0.0`.

### Example 1: Two Ways to Deny an Entire Group

To block all memberships for SSM group `232.1.1.100`, use:

```text
R1(config)# ip access-list extended DENY-GROUP
R1(config-ext-nacl)# deny igmp any host 232.1.1.100
R1(config-ext-nacl)# permit igmp any any
R1(config-ext-nacl)# exit
R1(config)# interface Ethernet0/1
R1(config-if)# ip igmp access-group DENY-GROUP
```

The preliminary `(0.0.0.0,232.1.1.100)` lookup matches the first ACE, because `any` includes `0.0.0.0`:

```text
(0.0.0.0,232.1.1.100)
          |
          v
 deny igmp any host 232.1.1.100
          |
          v
         DENY
```

You could instead write:

```text
deny igmp host 0.0.0.0 host 232.1.1.100
permit igmp any any
```

This also **explicitly denies the preliminary group-level lookup**, so the entire group's membership request is rejected.

The two deny statements do **not** have identical matching scopes:

- `deny igmp host 0.0.0.0 host G` matches only the synthetic group-level lookup.

- `deny igmp any host G` also matches real `(S,G)` lookups, if such lookups are performed.

In this group-blocking example, **the outcome is the same** because both match and deny the initial `(0,G)` lookup. This applies to a group operating in either ASM or SSM; the `any` keyword does not mean "ASM."

### Example 2: Allow a Group but Block One SSM Source

Suppose the objective is to allow all sources for `232.1.1.1` **except** `10.1.1.20`.

```text
R1(config)# ip access-list extended SSM-SOURCE-FILTER
R1(config-ext-nacl)# permit igmp host 0.0.0.0 host 232.1.1.1
R1(config-ext-nacl)# deny igmp host 10.1.1.20 host 232.1.1.1
R1(config-ext-nacl)# permit igmp any host 232.1.1.1
R1(config-ext-nacl)# exit
R1(config)# interface Ethernet0/1
R1(config-if)# ip igmp access-group SSM-SOURCE-FILTER
```

The first ACE permits the **group-level** lookup, but cannot match ordinary source-specific lookups such as `(10.1.1.20,G)`.

The second ACE blocks that particular source. The third permits other sources in this group:

| Lookup | First matching entry | Outcome |
|---|---|---|
| `(0.0.0.0,232.1.1.1)` | `permit igmp host 0.0.0.0 host G` | Preliminary check passes |
| `(10.1.1.10,232.1.1.1)` | `permit igmp any host G` | Channel permitted |
| `(10.1.1.20,232.1.1.1)` | `deny igmp host 10.1.1.20 host G` | Channel denied |
| `(10.1.1.30,232.1.1.1)` | `permit igmp any host G` | Channel permitted |

The resulting source-specific membership can include `10.1.1.10` and `10.1.1.30`, but not `10.1.1.20`.

This ACL permits **only group `232.1.1.1`**. Other groups do not have a permit entry. If other groups should also be admitted, add appropriate additional permits (for example, a final `permit igmp any any`), taking care not to shadow your source-specific deny.

### Example 3: Why `permit igmp any host G` Is Not Interchangeable

Change only the first ACE in Example 2:

```text
ip access-list extended SSM-SOURCE-FILTER
 permit igmp any host 232.1.1.1
 deny igmp host 10.1.1.20 host 232.1.1.1
 permit igmp any host 232.1.1.1
```

The preliminary check still passes, but the **first ACE also matches every individual `(S,G)` lookup** for `232.1.1.1`:

```text
(0.0.0.0,232.1.1.1)   -> First ACE -> Permit
(10.1.1.10,232.1.1.1) -> First ACE -> Permit
(10.1.1.20,232.1.1.1) -> First ACE -> Permit
```

The second ACE is never reached. Source `10.1.1.20` is therefore **permitted**, contrary to the intended policy.

The distinction is:

```text
permit igmp host 0.0.0.0 host G
  -> Permits only the preliminary group-level lookup
permit igmp any host G
  -> Matches the preliminary lookup AND all source lookups
```

**ACL entry order still matters**.

The ACL evaluation restarts at the top for both the preliminary **(0,G)** check and the actual **(S,G)** check.

### Example 4: A Source Permit Followed by an Explicit `deny any any`

Consider:

```text
ip access-list extended SSM-WHITELIST
 permit igmp host 10.1.1.10 host 232.1.1.1
 deny igmp any any
```

A common mistake is to expect `10.1.1.10` to be allowed because of the first ACE.

However, the preliminary lookup is:

```text
(0.0.0.0,232.1.1.1)
```

That lookup does not match the source-specific permit; it **does** match the explicit `deny igmp any any`:

```text
(0,G)
  -> source-only permit: no match
  -> explicit deny any any: match
  -> group rejected before (S,G) evaluation
```

To express a strict source whitelist while explicitly permitting the group-level lookup, use:

```text
ip access-list extended SSM-WHITELIST
 permit igmp host 0.0.0.0 host 232.1.1.1
 permit igmp host 10.1.1.10 host 232.1.1.1
 deny igmp any any
```

Now the expected lookup results are:

| Lookup | Result |
|---|---|
| `(0.0.0.0,232.1.1.1)` | Permit |
| `(10.1.1.10,232.1.1.1)` | Permit |
| `(10.1.1.20,232.1.1.1)` | Deny |
| `(0.0.0.0,232.1.1.2)` | Deny (different group) |

### Example 5: Permitting `(0,G)` Does Not Permit All Sources

This example makes the stages particularly clear:

```text
ip access-list extended NO-SOURCES
 permit igmp host 0.0.0.0 host 232.1.1.1
 deny igmp any host 232.1.1.1
```

The preliminary `(0,G)` lookup matches the first permit. Each subsequent real `(S,G)` lookup fails the first ACE and matches the second deny:

```text
(0.0.0.0,232.1.1.1)   -> Permit (preliminary check)
(10.1.1.10,232.1.1.1) -> Deny
(10.1.1.20,232.1.1.1) -> Deny
```

So passing the preliminary check is not sufficient to admit a source-specific channel.

### Lab Verification: Implicit Deny of the Preliminary `(0,G)` Check

The **implicit deny** at the end of an extended ACL also applies to the preliminary `(0,G)` lookup in my lab testing.

An explicitly permitted `(S,G)` channel is not sufficient if the preliminary group-level lookup has no matching permit.

The following ACL was configured on R1 and applied to the receiver-facing `Ethernet0/1` interface:

```text
R1# show ip access-lists
Extended IP access list SSM-FILTER
    10 permit igmp host 10.1.1.20 host 232.1.1.1
```

During the test, R1 received an IGMPv3 Report for group `232.1.1.1` (with `10.1.1.20` the intended permitted source):

```text
R1# debug ip igmp
IGMP debugging is on
R1#
*Oct 10 03:19:58.057: IGMP(0)[default]: Received v3 Report for 1 group on Ethernet0/1 from 0.0.0.0
*Oct 10 03:19:58.057: IGMP(*): Group 232.1.1.1 access denied on Ethernet0/1
```

The ACL evaluation explains the result:

```text
Preliminary lookup: (0.0.0.0,232.1.1.1)

10 permit igmp host 10.1.1.20 host 232.1.1.1
   -> No match (0.0.0.0 is not 10.1.1.20)

Implicit deny
   -> Group rejected

Requested (10.1.1.20,232.1.1.1) channel
   -> Not evaluated after the group-level rejection
```

The debug message identifies a group-level access denial, consistent with the rejection occurring during the preliminary check. 

The `from 0.0.0.0` field, however, is the Report's **sender** field and must not be confused with the synthetic `(0,G)` ACL lookup.

To explicitly permit the preliminary check and whitelist this one SSM channel, use:

```text
R1(config)# ip access-list extended SSM-FILTER
R1(config-ext-nacl)# permit igmp host 0.0.0.0 host 232.1.1.1
R1(config-ext-nacl)# permit igmp host 10.1.1.20 host 232.1.1.1
```

The expected result is:

| Lookup | ACL result |
|---|---|
| `(0.0.0.0,232.1.1.1)` | Permit (first ACE) |
| `(10.1.1.20,232.1.1.1)` | Permit (second ACE) |
| `(10.1.1.10,232.1.1.1)` | Deny (implicit deny) |
| `(0.0.0.0,232.1.1.2)` | Deny (implicit deny) |

This corrected configuration follows the documented two-stage evaluation and avoids reliance on a source-specific permit to pass the preliminary check. 

#### Cisco Documentation Caveat

Cisco's *Customizing IGMP* documentation describes the `(0,G)` check and reminds readers that extended ACLs have an implicit deny. However, its example of permitting all groups from a single SSM source uses only:

```text
ip access-list extended SOURCE-ONLY
 permit igmp host 10.6.23.32 any
!
interface Ethernet0/1
 ip igmp access-group SOURCE-ONLY
```

There is **no explicit `(0,G)` permit**. If the preliminary lookup reaches the implicit deny, the example cannot admit the source-specific membership as described. 

> As always, make sure to verify the behavior of your configuring filters, instead of assuming the documentation is correct.
> The behavior I observed could be platform-specific, or it could be universal.

### Key Points

- `(0.0.0.0,G)` is a preliminary **group-level** ACL lookup, performed before the individual `(S,G)` lookups Cisco describes.

- When the preliminary `(0,G)` lookup is explicitly permitted, evaluation proceeds to the individual `(S,G)` checks; that permit does not itself admit all sources.

- `host 0.0.0.0` matches the synthetic group-level value; `any` also matches it using ordinary ACL wildcard matching.

- `deny igmp host 0.0.0.0 host G` and `deny igmp any host G` can both explicitly reject the entire group's membership request.

- `permit igmp any host G` can shadow a later source-specific deny because it matches both group- and source-level checks.

- The implicit ACL deny rejected an unmatched preliminary `(0,G)` lookup in my lab. A source-specific permit alone did not admit the channel.

- Cisco publishes a source-only permit example without an explicit `(0,G)` permit. That example **does not match the behavior I observed**, so do not assume it works universally.

- For source whitelists, explicitly permit `(0,G)`, then permit the intended `(S,G)` channels and rely on implicit deny for all others; verify the result on your IOS XE image.

## Limiting IGMP-Driven Mroute State with `ip igmp limit`

The `ip igmp limit` command limits the number of **multicast routing states (mroute states)** resulting from IGMP joins.

It is important to distinguish three things:

| Term | Meaning |
|---|---|
| **IGMP membership state** | Receiver-interest information maintained in the IGMP cache, including groups and IGMPv3 source lists |
| **Mroute state** | Multicast routing state, typically represented as `(*,G)` or `(S,G)`, used to determine multicast forwarding |
| **IGMP-driven mroute state** | Mroute state attributable to accepted IGMP joins. This is what `ip igmp limit` limits |

The limit is **enforced during IGMP join processing**, not only after multicast traffic begins flowing. When a new IGMP group or channel request would exceed the applicable limit, the request is rejected and is **not added to the IGMP cache**.

Existing accepted state is not replaced to make room for it.

Thus, `ip igmp limit 2` does **not** mean "accept two IGMP Report packets," "accept two receivers," or "allow only two entries in `show ip igmp membership tracked`." It is an **admission-control quota for IGMP-driven mroute state**.

It also is not a cap on all PIM or multicast routing state in the router; that is the broader purpose of `ip multicast route-limit`.

Two scopes are supported:

```text
Global limit
  -> IGMP-driven mroute states admitted across the router

Per-interface limit
  -> IGMP-driven mroute states resulting from joins on one Layer 3 interface
```

The numeric range is **1–64000**. By default, no IGMP state limit is configured.

### What Counts Toward the Limit?

The limiter evaluates **new group or channel state arising from IGMP joins**:

| Event | Effect on the IGMP-driven mroute quota |
|---|---|
| A receiver joins a new ASM group `(*,G)` | Can consume a new **group** state |
| A receiver joins a new SSM channel `(S,G)` | Can consume a new **channel** state |
| A receiver joins a second source for the same SSM group | A distinct `(S,G)` channel can consume another state |
| A second receiver requests a group/channel already admitted on that interface | Does **not** automatically create another distinct mroute state |
| A receiver refreshes an existing membership | Does **not** consume another state merely because another Report arrived |
| Transit PIM state created independently of an IGMP join | Not the resource targeted by `ip igmp limit` |

The number is **not simply a count of IGMP cache rows**.

In particular, `show ip igmp membership tracked` can display a group-level `(*,G)` record along with IGMPv3 `(S,G)` records; do not treat every displayed row as a separate consumed mroute slot without verifying platform accounting.

Likewise, the total number of entries shown by `show ip mroute count` can include mroutes that did **not** originate from IGMP joins.

### Per-Interface IGMP Limit

Configure a receiver-interface limit with:

```text
R1(config)# interface Ethernet0/1
R1(config-if)# ip igmp limit 2
```

Suppose two different ASM joins have already resulted in the following **counted IGMP-driven group states**:

```text
Ethernet0/1

(*,239.1.1.1) -> Admitted IGMP-driven mroute state (1/2)
(*,239.1.1.2) -> Admitted IGMP-driven mroute state (2/2)
```

A receiver then requests:

```text
(*,239.1.1.3)
```

The interface limit has been reached:

```text
Report for 239.1.1.3
        |
        v
R1 Ethernet0/1
        |
        v
Counted IGMP-driven mroute states = 2
Configured per-interface limit = 2
        |
        v
New state denied
```

This real `debug ip igmp` output illustrates R1 rejecting the third group at the configured limit:

```
*Oct 10 03:48:06.841: IGMP(0)[default]: Received v3 Report for 1 group on Ethernet0/1 from 0.0.0.0
*Oct 10 03:48:06.841: IGMP(0)[default]: Received Group record for group 239.1.1.3, mode 4 from 0.0.0.0 for 0 sources
*Oct 10 03:48:06.841: IGMP_ACL(0)[default]: Group 239.1.1.3 access denied on Ethernet0/1
*Oct 10 03:48:06.841: %IGMP-6-IGMP_GROUP_LIMIT: IGMP limit exceeded for group (*, 239.1.1.3) on Ethernet0/1 by host 0.0.0.0
```

The new join is **not admitted to the IGMP cache** because it would exceed the IGMP-driven mroute quota.

The `%IGMP-6-IGMP_GROUP_LIMIT` message identifies a rejected `(*,G)` group request; the `IGMP_ACL` debug line alone should not be mistaken for proof that an `ip igmp access-group` ACL caused the rejection.

Existing admitted states are not replaced simply to make room for the new one.

> There is no `replace` action equivalent to Layer 2 `ip igmp max-groups action replace`.

### IGMP Reports, Receiver Counts, and SSM Channels

The limit does **not** count IGMP packets, membership refreshes, or individual host subscriptions. For example, these Reports do not create three new IGMP-driven mroute states:

```text
Report: 239.1.1.1  -> New group join, if not already admitted
Report: 239.1.1.1  -> Refreshes the existing group
Report: 239.1.1.1  -> Refreshes the existing group
```

Likewise, two receivers on the **same interface** can share interest in an already-admitted ASM group:

```text
PC1 -> 239.1.1.1
PC2 -> 239.1.1.1

R1 Ethernet0/1:
  (*,239.1.1.1) -> One IGMP-driven group state
```

**SSM is different:** Separate source/group channels can require **separate IGMP-driven `(S,G)` mroute states**, even though their group address is identical.

For example, with `ip igmp limit 2` on Ethernet0/1 and no pre-existing counted states, a receiver sends one IGMPv3 INCLUDE-mode Report requesting:

```text
Group: 232.1.1.1
Sources: 10.1.1.10, 10.1.1.20, 10.1.1.30

Requested channels:
  (10.1.1.10,232.1.1.1)
  (10.1.1.20,232.1.1.1)
  (10.1.1.30,232.1.1.1)
```

These are **three distinct `(S,G)` channels**, not one merely because they arrived in one Report or share one group. If the first two consume the two available IGMP-driven mroute slots, the third is rejected:

```text
(10.1.1.10,232.1.1.1) -> Admitted (1/2)
(10.1.1.20,232.1.1.1) -> Admitted (2/2)
(10.1.1.30,232.1.1.1) -> Rejected (limit reached)
```

Cisco documents separate warning messages for denied `(*,G)` groups and `(S,G)` channels, respectively:

```text
%IGMP-6-IGMP_GROUP_LIMIT
%IGMP-6-IGMP_CHANNEL_LIMIT
```

### Global IGMP Limit

Configure a global limit in global configuration mode:

```text
R1(config)# ip igmp limit 3
```

The global limit caps **mroute state attributable to IGMP joins across the router**, rather than counting all multicast routes or limiting only one receiver-facing interface.

Using the same topology, PC1 and PC2 send their Membership Reports through SW1 to R1's receiver-facing **Ethernet0/1** interface:

```text
             Multicast Network
                    |
                  E0/0
                    |
                   R1
              10.3.4.1
                  E0/1
                    |
                   SW1
               VLAN 10
              /       \
           E0/1       E0/2
             |           |
            PC1         PC2
        10.3.4.10  10.3.4.11
```

Suppose three distinct ASM joins on Ethernet0/1 have each resulted in one counted IGMP-driven `(*,G)` state:

```text
R1, Ethernet0/1:

(*,239.1.1.1) -> Accepted (global 1/3)
(*,239.1.1.2) -> Accepted (global 2/3)
(*,239.1.1.3) -> Accepted (global 3/3)
```

A Report requesting the *new* group `239.1.1.4` is rejected because it would require an additional **IGMP-driven mroute state** beyond the global limit—even if Ethernet0/1 has no configured per-interface limit.

In this topology, Ethernet0/1 is the only receiver-facing interface. If R1 had multiple receiver-facing Layer 3 interfaces, the global limiter would apply to their combined IGMP-driven mroute states, while a per-interface limiter would apply to joins on only one interface.

### Combining Global and Interface Limits

Global and per-interface limits can be used together:

```text
R1(config)# ip igmp limit 5
R1(config)# interface Ethernet0/1
R1(config-if)# ip igmp limit 2
```

This configuration enforces two constraints:

```text
Ethernet0/1 -> At most 2 counted IGMP-driven mroute states
Entire R1   -> At most 5 counted IGMP-driven mroute states
```

Suppose PC1 joins `239.1.1.1` and PC2 joins `239.1.1.2`. Both are admitted, so Ethernet0/1 reaches its limit of two. A request for `239.1.1.3` on Ethernet0/1 is denied by the **interface** limit, even though the global count is only two out of five.

Conversely, with the following configuration:

```text
R1(config)# ip igmp limit 3
R1(config)# interface Ethernet0/1
R1(config-if)# ip igmp limit 5
```

a fourth distinct group on Ethernet0/1 is denied by the **global** limit, even though the interface is below its own limit of five.

**Both limits must be satisfied.** The first applicable limit to be reached prevents admission of additional counted state.

An interface `except` policy, covered next, excludes selected IGMP-driven group/channel states from the **interface** quota while still counting them against the **global** quota.

## The `except` Keyword: Exempt Memberships from an Interface Limit

On a per-interface `ip igmp limit`, the optional `except` keyword specifies an ACL identifying IGMP-driven group or channel states that **do not count toward that interface's mroute quota**.

Syntax:

```text
ip igmp limit <number> [except <acl-number-or-name>]
```

This is different from `ip igmp access-group`:

```text
ip igmp access-group
  -> Membership permitted or denied?

ip igmp limit ... except
  -> Should the resulting IGMP-driven mroute state count against this interface's limit?
```

A permitted ACL match for an `except` policy identifies **exempt IGMP-driven group/channel state**. The ACL does not, by itself, grant admission to otherwise prohibited memberships.

### Example: Exempt a Mandatory Announcement Group

Suppose a receiver should be able to join **two ordinary ASM groups** in addition to a mandatory announcement group, `239.255.255.254`.

Without an exception, all three groups would count against a limit of two.

Configure an exception ACL:

```text
R1(config)# ip access-list standard IGMP-EXEMPT
R1(config-std-nacl)# permit host 239.255.255.254
R1(config-std-nacl)# exit
R1(config)# interface Ethernet0/1
R1(config-if)# ip igmp limit 2 except IGMP-EXEMPT
```

Conceptually:

```text
Ethernet0/1

239.255.255.254 -> Exempt (does not count)
239.1.1.1       -> Counted (1/2)
239.1.1.2       -> Counted (2/2)
239.1.1.3       -> Denied: counted limit reached
```

The exception allows the special group to be admitted without consuming one of the interface's two counted IGMP-driven mroute slots.

### Exempting Source-Specific Channels

A standard ACL identifies groups `(*,G)`; an extended ACL can identify individual `(S,G)` channels.

For example, exempt only the channel `(10.1.1.10,232.1.1.1)`:

```text
R1(config)# ip access-list extended EXEMPT-CHANNEL
R1(config-ext-nacl)# permit ip host 10.1.1.10 host 232.1.1.1
R1(config-ext-nacl)# exit
R1(config)# interface Ethernet0/1
R1(config-if)# ip igmp limit 2 except EXEMPT-CHANNEL
```

In this context, the ACL's first address represents the multicast source and the second represents the multicast group. The exempt channel does not consume one of the two counted IGMP-driven mroute slots for that interface.

### `except` Does Not Exempt the Global Limit

This is a particularly important caveat.

IGMP-driven group/channel state matched by an interface `except` ACL is not charged to that **interface** mroute quota, but is still counted by the global IGMP state limiter when a **global limit** is configured.

For example:

```text
Global limit: 3
Ethernet0/1 limit: 2
Ethernet0/1 except: 239.255.255.254
```

Suppose the following states are admitted:

```text
239.255.255.254 -> Interface-exempt; global 1/3
239.1.1.1       -> Interface 1/2; global 2/3
239.1.1.2       -> Interface 2/2; global 3/3
```

An additional group is blocked by the global limit, even if it is listed in the interface's `except` ACL.

## Key Points

- `ip igmp access-group` filters multicast memberships on a routed interface using a standard or extended ACL.
- A **standard ACL** matches multicast **group addresses (G)**, not receiver host IP addresses.
- An **extended ACL** can match multicast **source/group (S,G)** channels, particularly useful for IGMPv3 and SSM.
- Cisco's extended IGMP ACL evaluation includes a preliminary `(0.0.0.0,G)` group-level check, followed by separate `(S,G)` checks when the group passes.
- `host 0.0.0.0` matches the group-level lookup; `any` also matches it and matches real source-specific lookups.
- The **implicit deny** rejects an unmatched preliminary `(0,G)` lookup in my tests. Cisco's source-only permit example is inconsistent with that observation; verify behavior on the target platform.
- For ASM, filtering IGMPv3 source records does not necessarily yield source-specific PIM forwarding behavior.
- `ip igmp limit` restricts **mroute states resulting from IGMP joins**, not the number of received Reports, receiver hosts, IGMP cache rows, or the bandwidth of streams.
- No IGMP state limiter is configured by default.
- Global and interface limits may be configured simultaneously; **both apply**.
- The per-interface `except` ACL exempts matched IGMP-driven group/channel states from the interface quota, **not from the global quota**.
- The Layer 3 limit does not have the Layer 2 throttling feature's `replace` action.
- IGMP router filtering and limits operate independently of Layer 2 IGMP snooping.
- These are membership-admission controls, not direct multicast data-plane ACLs.
