# Label Filtering

This page covers **LDP label filtering**.

If a router learns prefixes in the routing table, by default it can allocate local labels for them and advertise those label bindings to LDP peers.

Example:

```text
P1 learns these prefixes in the IGP:

1.1.1.1/32
2.2.2.2/32
3.3.3.3/32
4.4.4.4/32
5.5.5.5/32
10.1.2.0/30
10.1.3.0/30
10.2.3.0/30
```

In most MPLS L3VPN designs, the most important LDP labels are the labels used to reach PE loopbacks, because PE loopbacks are usually the BGP next hops.

```text
BGP route points to remote PE loopback.
IGP provides reachability to that loopback.
LDP provides a transport label for that loopback.
```

If labels are created for every infrastructure prefix, the LIB and label advertisements can become larger than necessary.

Label filtering allows you to control which label bindings are allocated, advertised, or accepted.

---

## What Label Filtering Does

Label filtering controls LDP label bindings.

If you filter an LDP label for a prefix, the IP route can still exist in the routing table.

Example:

```text
P1 still has an OSPF route to 5.5.5.5/32.
But P1 may not have an LDP label for 5.5.5.5/32.
```

That means normal IP reachability can still work, but MPLS forwarding toward that FEC may not work.
That is not necessarily an issue, as long as you allow labels for the required FECs.

---

## Basic Topology

Use this topology:

```text
            P3
           /  \
          /    \
 PE1 --- P1 --- P2 --- PE2
```

Loopbacks:

```text
PE1 Loopback0: 1.1.1.1/32
P1  Loopback0: 2.2.2.2/32
P2  Loopback0: 3.3.3.3/32
P3  Loopback0: 4.4.4.4/32
PE2 Loopback0: 5.5.5.5/32
```

LDP IDs:

```text
PE1: 1.1.1.1:0
P1:  2.2.2.2:0
P2:  3.3.3.3:0
P3:  4.4.4.4:0
PE2: 5.5.5.5:0
```

Assume the IGP and LDP are already working.

---

## Why Filter Labels?

Common reasons:

```text
Reduce LIB size.
Reduce label advertisement messages.
Reduce unnecessary label bindings for link prefixes.
Force labels to exist only for loopbacks or PE loopbacks.
```

A common design idea is:

```text
Only create labels for loopbacks.
Do not create labels for transit link subnets.
```

Useful labels:

```text
1.1.1.1/32 (PE1 loopback)
5.5.5.5/32 (PE2 loopback)
```

These are the PE loopbacks that are used as BGP next hops, so they are the only truly necessary labels
to allow labeled traffic to be carried through the MPLS core.

---

## Types of LDP Label Filtering

There are three label filtering concepts to understand:

| Filtering Type | Direction | What It Controls |
| :--- | :--- | :--- |
| 1. Local label allocation filtering | Local | Which prefixes the local router allocates labels for |
| 2. Outbound label advertisement filtering | Outbound | Which local label bindings the router advertises to LDP peers |
| 3. Inbound label binding filtering | Inbound | Which remote label bindings the router accepts from a peer |

These are related, but they are not the same.

```text
Local allocation filtering:
Do I create a local label for this prefix?

Outbound advertisement filtering:
Do I advertise my local label for this prefix to this peer?

Inbound label binding filtering:
Do I accept the label binding this peer advertised to me?
```

---

## Default LDP Label Behavior

By default, LDP allocates local labels for non-BGP prefixes in the global routing table.

In an MPLS L3VPN core, this usually means LDP assigns labels for provider infrastructure routes, such as PE and P loopbacks.

LDP does not normally allocate labels for BGP-learned customer or Internet routes.
The PE only needs an LDP transport label for the BGP next hop, not for every BGP prefix.

---

## Local Label Allocation Filtering

Local label allocation filtering controls whether the local router allocates a local label for a prefix.

This is the most direct way to reduce local labels.

```text
No local label allocated
  ↓
No local binding for that FEC
  ↓
Nothing to advertise for that FEC
```

---

### Local Allocation: Host Routes

The simplest option is to allocate local labels only for host routes.

Example on P1:

```text
P1(config)# mpls ldp label
P1(config-ldp-lbl)# allocate global host-routes
```

This tells P1:

```text
Allocate local labels only for /32 host routes in the global routing table.
```

This is useful when all important infrastructure loopbacks are /32s.

Before:

```text
P1# show mpls ldp bindings | i lib|local
  lib entry: 1.1.1.1/32, rev 2
        local binding:  label: 22
  lib entry: 2.2.2.2/32, rev 4
        local binding:  label: imp-null
  lib entry: 3.3.3.3/32, rev 6
        local binding:  label: 23
  lib entry: 4.4.4.4/32, rev 8
        local binding:  label: 24
  lib entry: 5.5.5.5/32, rev 10
        local binding:  label: 25
  lib entry: 10.1.1.0/24, rev 30
        local binding:  label: imp-null
  lib entry: 10.1.2.0/24, rev 31
        local binding:  label: imp-null
  lib entry: 10.1.3.0/24, rev 32
        local binding:  label: imp-null
  lib entry: 10.2.2.0/24, rev 33
        local binding:  label: 20
  lib entry: 10.2.3.0/24, rev 34
        local binding:  label: 16
```

P1 configuration:

```
P1(config)# mpls ldp label
P1(config-ldp-lbl)# allocate global host-routes 
```

After:

```
P1# show mpls ldp bindings | i lib|local
  lib entry: 1.1.1.1/32, rev 2
        local binding:  label: 22
  lib entry: 2.2.2.2/32, rev 4
        local binding:  label: imp-null
  lib entry: 3.3.3.3/32, rev 6
        local binding:  label: 23
  lib entry: 4.4.4.4/32, rev 8
        local binding:  label: 24
  lib entry: 5.5.5.5/32, rev 10
        local binding:  label: 25
  lib entry: 10.1.1.0/24, rev 35
        no local binding
  lib entry: 10.1.2.0/24, rev 36
        no local binding
  lib entry: 10.1.3.0/24, rev 37
        no local binding
  lib entry: 10.2.2.0/24, rev 38
        no local binding
  lib entry: 10.2.3.0/24, rev 39
        no local binding
```

As the above output shows, P1 only assigns local bindings for the /32 FECs.

---

### Local Allocation: Prefix List

You can also use a prefix list to control which prefixes receive local labels.

Command:

```text
mpls ldp label
 allocate global prefix-list <prefix-list-name>
```

Example requirement:

```text
P1 should allocate local labels only for PE loopbacks.
```

PE loopbacks:

```text
PE1: 1.1.1.1/32
PE2: 5.5.5.5/32
```

Before:

```text
P1# show mpls ldp bindings | i lib|local
  lib entry: 1.1.1.1/32, rev 2
        local binding:  label: 22
  lib entry: 2.2.2.2/32, rev 4
        local binding:  label: imp-null
  lib entry: 3.3.3.3/32, rev 6
        local binding:  label: 23
  lib entry: 4.4.4.4/32, rev 8
        local binding:  label: 24
  lib entry: 5.5.5.5/32, rev 10
        local binding:  label: 25
  lib entry: 10.1.1.0/24, rev 30
        local binding:  label: imp-null
  lib entry: 10.1.2.0/24, rev 31
        local binding:  label: imp-null
  lib entry: 10.1.3.0/24, rev 32
        local binding:  label: imp-null
  lib entry: 10.2.2.0/24, rev 33
        local binding:  label: 20
  lib entry: 10.2.3.0/24, rev 34
        local binding:  label: 16
```

P1 configuration:

```text
ip prefix-list LDP-LOCAL-LABELS permit 1.1.1.1/32
ip prefix-list LDP-LOCAL-LABELS permit 5.5.5.5/32
!
mpls ldp label
 allocate global prefix-list LDP-LOCAL-LABELS
```

After:

```
P1# show mpls ldp bindings | i lib|local
  lib entry: 1.1.1.1/32, rev 2
        local binding:  label: 22
  lib entry: 2.2.2.2/32, rev 45
        no local binding
  lib entry: 3.3.3.3/32, rev 46
        no local binding
  lib entry: 4.4.4.4/32, rev 47
        no local binding
  lib entry: 5.5.5.5/32, rev 10
        local binding:  label: 25
  lib entry: 10.1.1.0/24, rev 48
        no local binding
  lib entry: 10.1.2.0/24, rev 49
        no local binding
  lib entry: 10.1.3.0/24, rev 50
        no local binding
  lib entry: 10.2.2.0/24, rev 51
        no local binding
  lib entry: 10.2.3.0/24, rev 52
        no local binding
```

P1 now only assigns local bindings for the FECs matched by the prefix list: 1.1.1.1/32 and 5.5.5.5/32

---

## Outbound Label Filtering

Outbound label advertisement filtering controls which local label bindings a router advertises to LDP peers.

This is different from local label allocation filtering.

```text
Local label allocation filtering:
Controls whether a local label exists.

Outbound label advertisement filtering:
Controls whether an existing local label is advertised.
```

A router can allocate a local label for a prefix but choose not to advertise that label to some or all LDP peers.

Commands:

```text
no mpls ldp advertise-labels
mpls ldp advertise-labels for <prefix-acl>
mpls ldp advertise-labels for <prefix-acl> to <peer-acl>
```

The ACLs are standard IP ACLs.

```text
prefix-acl = prefixes/FECs whose labels should be advertised
peer-acl   = LDP peers that should receive those label advertisements
```

- The `prefix-acl` matches the prefix's network address.
- The `peer-acl` matches the peer LDP router ID.

---

### Outbound Filtering Logic

By default, LDP advertises all local labels.

If you configure only this:

```text
mpls ldp advertise-labels for ADV_LABEL
```

but do not disable default advertisement, unmatched labels will still be advertised.

For a strict whitelist, use this pattern:

```text
no mpls ldp advertise-labels
mpls ldp advertise-labels for ADV_LABEL
```

---

### Outbound Filtering: Advertise Only Loopback Labels

Example requirement:

```text
P1 should only advertise the labels for loopback prefixes.
```

Loopbacks:

```text
1.1.1.1/32
2.2.2.2/32
3.3.3.3/32
4.4.4.4/32
5.5.5.5/32
```

ACL:

```text
P1(config)# ip access-list standard ADV_LABEL
P1(config-std-nacl)# permit 1.1.1.1
P1(config-std-nacl)# permit 2.2.2.2
P1(config-std-nacl)# permit 3.3.3.3
P1(config-std-nacl)# permit 4.4.4.4
P1(config-std-nacl)# permit 5.5.5.5
```

Advertise labels for prefixes matched by the ACL:

```
P1(config)# mpls ldp advertise-labels for ADV_LABEL
```

However, P1 still advertises all of its local bindings, not just the /32 loopbacks:

```
P1# show mpls ldp bindings detail | ex remote
Advertisement spec:
        Prefix acl = ADV_LABEL

  lib entry: 1.1.1.1/32, rev 104, chkpt: none
        local binding:  label: 22 (owner LDP)
          Advertised to:
          3.3.3.3:0              4.4.4.4:0              1.1.1.1:0                         
        Advert acl(s): Prefix acl ADV_LABEL
  lib entry: 2.2.2.2/32, rev 105, chkpt: none
        local binding:  label: imp-null (owner LDP)
          Advertised to:
          3.3.3.3:0              4.4.4.4:0              1.1.1.1:0                          
        Advert acl(s): Prefix acl ADV_LABEL
  lib entry: 3.3.3.3/32, rev 106, chkpt: none
        local binding:  label: 21 (owner LDP)
          Advertised to:
          3.3.3.3:0              4.4.4.4:0              1.1.1.1:0                         
        Advert acl(s): Prefix acl ADV_LABEL
  lib entry: 4.4.4.4/32, rev 107, chkpt: none
        local binding:  label: 18 (owner LDP)
          Advertised to:
          3.3.3.3:0              4.4.4.4:0              1.1.1.1:0                          
        Advert acl(s): Prefix acl ADV_LABEL
  lib entry: 5.5.5.5/32, rev 108, chkpt: none
        local binding:  label: 25 (owner LDP)
          Advertised to:
          3.3.3.3:0              4.4.4.4:0              1.1.1.1:0                       
        Advert acl(s): Prefix acl ADV_LABEL
  lib entry: 10.1.1.0/24, rev 109, chkpt: none
        local binding:  label: imp-null (owner LDP)
          Advertised to:
          3.3.3.3:0              4.4.4.4:0              1.1.1.1:0                       
  lib entry: 10.1.2.0/24, rev 110, chkpt: none
        local binding:  label: imp-null (owner LDP)
          Advertised to:
          3.3.3.3:0              4.4.4.4:0              1.1.1.1:0                       
  lib entry: 10.1.3.0/24, rev 111, chkpt: none
        local binding:  label: imp-null (owner LDP)
          Advertised to:
          3.3.3.3:0              4.4.4.4:0              1.1.1.1:0                      
  lib entry: 10.2.2.0/24, rev 112, chkpt: none
        local binding:  label: 26 (owner LDP)
          Advertised to:
          3.3.3.3:0              4.4.4.4:0              1.1.1.1:0                      
  lib entry: 10.2.3.0/24, rev 113, chkpt: none
        local binding:  label: 27 (owner LDP)
          Advertised to:
          3.3.3.3:0              4.4.4.4:0              1.1.1.1:0              
```

As noted above, you must disable the default advertisement behavior, which advertises all local bindings:

```
P1(config)# no mpls ldp advertise-labels
```

Now P1 only advertises labels for the FECs matched by the ACL:

```
P1# show mpls ldp bindings detail | ex remote
Advertisement spec:
        Prefix acl = ADV_LABEL

  lib entry: 1.1.1.1/32, rev 114, chkpt: none
        local binding:  label: 22 (owner LDP)
          Advertised to:
          3.3.3.3:0              4.4.4.4:0              1.1.1.1:0                        
        Advert acl(s): Prefix acl ADV_LABEL
  lib entry: 2.2.2.2/32, rev 115, chkpt: none
        local binding:  label: imp-null (owner LDP)
          Advertised to:
          3.3.3.3:0              4.4.4.4:0              1.1.1.1:0                        
        Advert acl(s): Prefix acl ADV_LABEL
  lib entry: 3.3.3.3/32, rev 116, chkpt: none
        local binding:  label: 21 (owner LDP)
          Advertised to:
          3.3.3.3:0              4.4.4.4:0              1.1.1.1:0                     
        Advert acl(s): Prefix acl ADV_LABEL
  lib entry: 4.4.4.4/32, rev 117, chkpt: none
        local binding:  label: 18 (owner LDP)
          Advertised to:
          3.3.3.3:0              4.4.4.4:0              1.1.1.1:0                        
        Advert acl(s): Prefix acl ADV_LABEL
  lib entry: 5.5.5.5/32, rev 118, chkpt: none
        local binding:  label: 25 (owner LDP)
          Advertised to:
          3.3.3.3:0              4.4.4.4:0              1.1.1.1:0              
        Advert acl(s): Prefix acl ADV_LABEL
  lib entry: 10.1.1.0/24, rev 119, chkpt: none
        local binding:  label: imp-null (owner LDP)
  lib entry: 10.1.2.0/24, rev 120, chkpt: none
        local binding:  label: imp-null (owner LDP)
  lib entry: 10.1.3.0/24, rev 121, chkpt: none
        local binding:  label: imp-null (owner LDP)
  lib entry: 10.2.2.0/24, rev 122, chkpt: none
        local binding:  label: 26 (owner LDP)
  lib entry: 10.2.3.0/24, rev 123, chkpt: none
        local binding:  label: 27 (owner LDP)
```

---

### Outbound Filtering: Advertise a Label Only to One Peer

You can also advertise labels only to selected LDP peers.

Requirements:

```
P1 should advertise only PE1's loopback to P2 and P3.
P1 should advertise only PE2's loopback to PE1.
```

> I removed the previous section's filter from P1's configuration.

Review of the configured IPs/RIDs:

```
PE1 loopback: 1.1.1.1/32
PE2 loopback: 5.5.5.5/32
PE1 RID: 1.1.1.1
P2 RID: 3.3.3.3
P3 RID: 4.4.4.4
```

P1 configuration:

```
ip access-list standard PE1
 permit 1.1.1.1
!
ip access-list standard PE2
 permit 5.5.5.5
!
ip access-list standard P2-P3
 permit 3.3.3.3
 permit 4.4.4.4
!
no mpls ldp advertise-labels
mpls ldp advertise-labels for PE1 to P2-P3
mpls ldp advertise-labels for PE2 to PE1
```

Result:

```
P1# show mpls ldp bindings detail | ex remote
Advertisement spec:
        Prefix acl = PE1; Peer acl = P2-P3
        Prefix acl = PE2; Peer acl = PE1

  lib entry: 1.1.1.1/32, rev 29, chkpt: none
        local binding:  label: 20 (owner LDP)
          Advertised to:
          4.4.4.4:0              3.3.3.3:0   → ADVERTISED ONLY TO P2/P3
        Advert acl(s): Prefix acl PE1; Peer acl P2-P3
  lib entry: 2.2.2.2/32, rev 24, chkpt: none
        local binding:  label: imp-null (owner LDP)
  lib entry: 3.3.3.3/32, rev 25, chkpt: none
        local binding:  label: 16 (owner LDP)
  lib entry: 4.4.4.4/32, rev 26, chkpt: none
        local binding:  label: 23 (owner LDP)
  lib entry: 5.5.5.5/32, rev 30, chkpt: none
        local binding:  label: 24 (owner LDP)
          Advertised to:
          1.1.1.1:0                           → ADVERTISED ONLY TO PE1
        Advert acl(s): Prefix acl PE2; Peer acl PE1
  lib entry: 10.1.1.0/24, rev 12, chkpt: none
        local binding:  label: imp-null (owner LDP)
  lib entry: 10.1.2.0/24, rev 14, chkpt: none
        local binding:  label: imp-null (owner LDP)
  lib entry: 10.1.3.0/24, rev 16, chkpt: none
        local binding:  label: imp-null (owner LDP)
  lib entry: 10.2.2.0/24, rev 18, chkpt: none
        local binding:  label: 17 (owner LDP)
  lib entry: 10.2.3.0/24, rev 20, chkpt: none
        local binding:  label: 19 (owner LDP)
  lib entry: 10.10.10.0/24, rev 22, chkpt: none
        no local binding
  lib entry: 192.168.1.0/24, rev 21, chkpt: none
        no local binding
```

This is very granular. It's easy to break MPLS forwarding if you filter labels that downstream routers need,
so be careful.

---

## Inbound Label Filtering

Inbound label binding filtering controls which remote label bindings a router accepts from a specific LDP peer.

Command:

```text
mpls ldp neighbor <peer-rid> labels accept <prefix-acl>
```

Like in outbound filtering, the `prefix-acl` matches the prefix's network address.

Let's look at an example.

---

### Inbound Filtering Example

Example requirement:

```text
P1 should only accept the label for PE1's loopback from PE1.
```

Here's the before. P1 accepts bindings for all FECs from PE1:

```
P1#show mpls ldp bindings neighbor 1.1.1.1
  lib entry: 1.1.1.1/32, rev 2
        remote binding: lsr: 1.1.1.1:0, label: imp-null
  lib entry: 2.2.2.2/32, rev 4
        remote binding: lsr: 1.1.1.1:0, label: 17
  lib entry: 3.3.3.3/32, rev 6
        remote binding: lsr: 1.1.1.1:0, label: 19
  lib entry: 4.4.4.4/32, rev 8
        remote binding: lsr: 1.1.1.1:0, label: 21
  lib entry: 5.5.5.5/32, rev 10
        remote binding: lsr: 1.1.1.1:0, label: 24
  lib entry: 10.1.1.0/24, rev 12
        remote binding: lsr: 1.1.1.1:0, label: imp-null
  lib entry: 10.1.2.0/24, rev 14
        remote binding: lsr: 1.1.1.1:0, label: 18
  lib entry: 10.1.3.0/24, rev 16
        remote binding: lsr: 1.1.1.1:0, label: 22
  lib entry: 10.2.2.0/24, rev 18
        remote binding: lsr: 1.1.1.1:0, label: 20
  lib entry: 10.2.3.0/24, rev 20
        remote binding: lsr: 1.1.1.1:0, label: 23
  lib entry: 10.10.10.0/24, rev 22
        remote binding: lsr: 1.1.1.1:0, label: 16
  lib entry: 192.168.1.0/24, rev 21
        remote binding: lsr: 1.1.1.1:0, label: imp-null
```

P1 configuration:

```text
ip access-list standard PE1
 permit 1.1.1.1
!
mpls ldp neighbor 1.1.1.1 labels accept PE1
```

After:

```
P1# show mpls ldp bindings neighbor 1.1.1.1
  lib entry: 1.1.1.1/32, rev 2
        remote binding: lsr: 1.1.1.1:0, label: imp-null
```

P1 now only accepts the binding for 1.1.1.1/32 from PE1.

---

## Local Allocation vs Advertisement vs Inbound Filtering

Quick review of the three filtering concepts covered above:

| Question | Feature |
| :--- | :--- |
| Should I create a local label for this prefix? | Local label allocation filtering |
| Should I advertise my local label to this peer? | Outbound label advertisement filtering |
| Should I accept this remote label from this peer? | Inbound label binding filtering |

```text
Allocate = create local label
Advertise = send local label to peer
Accept = receive remote label from peer
```

---

## L3VPN Considerations

In MPLS L3VPN, the packet usually has two labels:

```text
Outer label = transport label
Inner label = VPN label
```

LDP provides the outer transport label.

MP-BGP provides the inner VPN label.

Label filtering as covered on this page affects the LDP transport labels. 
It does not filter VPN labels advertised by MP-BGP.

---

## Example Lab Tasks

### Task: Allocate Labels Only for Loopbacks

Use local label allocation filtering.

```text
mpls ldp label
 allocate global host-routes
```

This is the cleanest option if loopbacks are /32s.

---

### Task: Allocate Labels Only for PE Loopbacks

Use a prefix list.

```text
ip prefix-list LDP-PE-LOOPBACKS permit 1.1.1.1/32
ip prefix-list LDP-PE-LOOPBACKS permit 5.5.5.5/32
!
mpls ldp label
 allocate global prefix-list LDP-PE-LOOPBACKS
```

---

### Task: Advertise Labels Only for Loopbacks

Use outbound advertisement filtering.

```text
ip access-list standard LDP-LOOPBACKS
 permit 1.1.1.1
 permit 2.2.2.2
 permit 3.3.3.3
 permit 4.4.4.4
 permit 5.5.5.5
!
no mpls ldp advertise-labels
mpls ldp advertise-labels for LDP-LOOPBACKS
```

---

### Task: Accept Labels Only for Specific FECs from a Peer

Use inbound label binding filtering.

```text
ip access-list standard LDP-ACCEPT-FECS
 permit 5.5.5.5
!
mpls ldp neighbor 3.3.3.3 labels accept LDP-ACCEPT-FECS
```

This accepts only labels for `5.5.5.5/32` from peer `3.3.3.3`.

---

## Key Points

* LDP label filtering controls which bindings are allocated, advertised, or accepted.
* Local allocation filtering determines the prefixes/FECs for which the router assigns labels.
  * By default, it creates label bindings for all non-BGP prefixes.
  * Enter `mpls ldp label` config mode to modify.
  * `allocate global host-routes` allocates labels only for /32 host routes.
  * `allocate global prefix-list <prefix-list>` alloates labels only for permitted prefixes.
* Outbound label filtering controls which local bindings the router advertises to peers.
  * Use `no mpls ldp advertise labels` to disable the default behavior of advertising all local labels.
  * Use `mpls ldp advertise-labels for <prefix-acl>` to advertise labels for prefixes whose network address is permitted.
  * Use `mpls ldp advertise-labels for <prefix-acl> to <peer-acl>` to advertise specific labels only to specific peers.
* Inbound label filtering controls which remote label bindings a router accepts from a specific peer.
  * Use `mpls ldp neighbor <peer-rid> labels accept <prefix-acl>` to configure inbound filtering.