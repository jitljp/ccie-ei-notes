# Source-Active Messages and SA Cache

**Source-Active (SA) messages** are the primary mechanism used by Multicast Source Discovery Protocol (MSDP) to advertise active multicast sources between Rendezvous Points (RPs).

In an IPv4 Anycast RP deployment, SA messages allow one RP to learn about sources that have registered with another RP.

An SA message identifies:

- The **multicast source (S)**.
- The **multicast group (G)**.
- The **originating RP**, identified by its unique MSDP Originator ID.

The receiving RP stores this information in its **SA cache** and can use it to establish a PIM `(S,G)` source tree when receivers are interested in the advertised group.

**MSDP provides source discovery, while PIM builds the multicast forwarding tree.**

## SA Message Origination

An RP running MSDP normally originates SA messages for **locally registered multicast sources**.

A locally registered source is one that the RP has learned about through the PIM source registration process, rather than through an SA message received from another MSDP peer.

Consider the following topology:

```text
Src --- R1 --- RP1 --- RP2 --- R4 --- Rec
```

Assume the following addresses:

| Device | Address |
|---|---|
| Src | `192.0.2.10` |
| Multicast group | `239.1.1.1` |
| Anycast RP | `10.0.0.100` |
| RP1 MSDP address | `10.0.0.1` |
| RP2 MSDP address | `10.0.0.2` |

### Step 1: Source Registers with RP1

Src begins sending multicast traffic to `239.1.1.1`.

R1 receives the traffic and sends PIM Register messages to the Anycast RP address:

```text
10.0.0.100
```

Unicast routing directs these messages to RP1.

RP1 learns the active source:

```text
(192.0.2.10, 239.1.1.1)
```

### Step 2: RP1 Originates an SA Message

Because the source has registered locally, RP1 originates an MSDP SA message advertising the source.

For example:

```text
MSDP SA Message

Originating RP: 10.0.0.1
Source (S):     192.0.2.10
Group (G):      239.1.1.1
```

RP1 sends this advertisement to RP2 over their established MSDP TCP session.

```text
RP1 --------------------------> RP2
           MSDP SA
```

The SA message uses RP1's **unique Originator ID**, `10.0.0.1`, not the shared Anycast RP address, `10.0.0.100`.

### Step 3: RP2 Processes the SA Message

When RP2 receives the SA message, it:

1. Performs the MSDP Peer-RPF check.
2. Applies any configured incoming SA filters.
3. Accepts the advertised source information if the checks succeed.
4. Stores the information in its SA cache.
5. Determines whether it has downstream interest in the advertised multicast group.

If RP2 has interested receivers for `239.1.1.1`, it can initiate a PIM `(S,G)` Join toward `192.0.2.10`.

If there are no interested receivers, RP2 retains the source information in its SA cache but does not need to join the source tree.

> **Important:** Receiving an SA message does not automatically create multicast forwarding state or cause the receiving RP to send a PIM Join. A source tree is built when there is downstream interest in the group.

## SA Message Format

MSDP messages use a **Type-Length-Value (TLV)** format.

An IPv4 Source-Active message uses **MSDP TLV Type 1**.

The important fields are:

| Field | Description |
|---|---|
| Type | `1`, IPv4 Source-Active |
| Length | Total length of the SA TLV |
| Entry Count | Number of `(S,G)` entries advertised |
| RP Address | Unique address identifying the originating RP |
| Source Prefix Length | `32` for an IPv4 source |
| Group Address | Multicast group address, `G` |
| Source Address | Multicast source address, `S` |
| Encapsulated Data | Optional multicast data packet |

The format can be represented as:

```text
MSDP SA TLV
+----------------------------------+
| Type = 1       | Length          |
+----------------------------------+
| Entry Count                      |
+----------------------------------+
| RP Address                       |
+----------------------------------+
| Reserved       | Source Prefix   |
+----------------------------------+
| Group Address                    |
+----------------------------------+
| Source Address                   |
+----------------------------------+
| Additional (S,G) entries...      |
+----------------------------------+
| Optional Encapsulated Data       |
+----------------------------------+
```

An SA message can contain **multiple `(S,G)` entries** associated with the same originating RP.

For example:

```text
RP Address: 10.0.0.1
Entry Count: 3

(S,G):
  (192.0.2.10, 239.1.1.1)
  (192.0.2.10, 239.1.1.2)
  (192.0.2.20, 239.1.1.3)
```

This allows an RP to advertise multiple active sources efficiently.

### RP Address Field

In an Anycast RP deployment, the RP Address field must identify the physical RP that originated the SA.

For example:

| Originating RP | SA RP Address |
|---|---|
| RP1 | `10.0.0.1` |
| RP2 | `10.0.0.2` |

The shared Anycast RP address, `10.0.0.100`, must not be used.

This is configured using:

```text
RP1(config)# ip msdp originator-id Loopback1
RP2(config)# ip msdp originator-id Loopback1
```

See [MSDP Peering and Originator ID](peering.md).

## SA Advertisement Interval

An originating RP periodically refreshes its advertisements while the corresponding multicast sources remain active.

RFC 3618 defines an **SA Advertisement Period of 60 seconds**.

An RP advertises its active sources during each advertisement period.

For example:

```text
Time 0:
  Source becomes active.
  RP1 originates an SA.

Time 60:
  RP1 periodically advertises the source.

Time 120:
  RP1 periodically advertises the source.

Time 180:
  RP1 periodically advertises the source.
```

An RP should also originate an initial SA promptly when it first discovers a new active source, without necessarily waiting for the next periodic advertisement.

Periodic advertisements are required because the receiving RP maintains SA information as soft state.

If the source stops transmitting and its registration state expires, the originating RP stops advertising it.

The receiving RP eventually removes the corresponding SA cache entry if it no longer receives advertisements.

**The 60-second SA Advertisement Period is separate from the MSDP TCP Keepalive Interval**, even though both have a default value of 60 seconds.

## SA Messages with Encapsulated Multicast Data

An SA message can optionally contain an encapsulated multicast data packet.

This is primarily useful when a source first becomes active.

Consider a receiver that is already interested in `239.1.1.1`, but RP2 has not yet established an `(S,G)` source tree.

When RP1 sends its initial SA message, it can include a multicast packet from the source.

```text
RP1 -------------------------> RP2
       MSDP SA Message
       + Encapsulated Data
```

RP2 can decapsulate the packet and forward it along an existing shared tree, providing initial data delivery while the native source tree is being established.

However, this encapsulation is optional.

Once the native `(S,G)` tree is established, multicast traffic flows using normal PIM forwarding.

### Initial vs. Periodic SA Messages

| SA Message | Encapsulated Data |
|---|---|
| Initial SA for a newly active source | May contain a multicast packet |
| Periodic SA refresh | Normally contains only source information |

**MSDP does not continuously tunnel multicast data between RPs.**

Its primary function is source discovery, not multicast data transport.

## SA Message Propagation

When an MSDP speaker receives an SA message, it may forward that message to other MSDP peers.

However, forwarding is subject to **MSDP Peer-RPF** and filtering rules.

For example, consider three MSDP speakers:

```text
RP1 -------- RP2 -------- RP3
```

Suppose RP1 originates an SA message.

RP2 receives the advertisement and may forward it to RP3.

```text
RP1 ---- SA ----> RP2 ---- SA ----> RP3
```

The original RP Address field remains unchanged:

```text
Originating RP: 10.0.0.1
```

RP2 does not replace that field with its own address.

This distinction is important:

| Operation | Description |
|---|---|
| SA origination | An RP creates an SA message for a locally registered source |
| SA forwarding | An MSDP speaker propagates an SA message originated by another RP |

**An RP does not originate a new SA merely because it learned about a source through an MSDP peer.**

In larger topologies, Peer-RPF and mesh-group rules control SA propagation and prevent loops.

These mechanisms are covered in [MSDP Peer-RPF and Mesh Groups](peer-rpf.md).

## The SA Cache

The **SA cache** stores information about active multicast sources learned through MSDP.

Each cached entry associates a source and group with the originating RP.

For example, after RP2 receives an SA from RP1:

```text
Source:         192.0.2.10
Group:          239.1.1.1
Originating RP: 10.0.0.1
MSDP Peer:      10.0.0.1
```

The SA cache allows RP2 to remember this source information even when no receivers are currently interested in the group.

### SA Cache vs. Multicast Routing Table

The SA cache is different from the multicast routing table.

| SA Cache | Multicast Routing Table |
|---|---|
| Maintained by MSDP | Maintained by PIM/multicast routing |
| Records advertised active sources | Maintains multicast forwarding state |
| Contains source, group, and originating RP information | Contains RPF interface, RPF neighbor, outgoing interfaces, and other state |
| Does not require interested receivers | Forwarding state depends on multicast routing and receiver interest |
| Viewed using `show ip msdp sa-cache` | Viewed using `show ip mroute` |

**An SA cache entry does not necessarily mean an `(S,G)` forwarding entry exists.**

### Example: No Interested Receivers

Suppose RP2 receives an SA message for:

```text
(192.0.2.10, 239.1.1.1)
```

However, Rec has not joined the group.

RP2 stores the source information:

```text
SA Cache:
  (192.0.2.10, 239.1.1.1)
```

But RP2 does not send an `(S,G)` Join toward the source merely because the SA exists.

The source information remains available for future receiver joins.

### Example: Receiver Joins Later

Now suppose Rec joins `239.1.1.1`.

R4 sends a PIM `(*,G)` Join toward RP2.

RP2 checks its SA cache and finds:

```text
Source: 192.0.2.10
Group:  239.1.1.1
```

Because the source is already known, RP2 can immediately initiate an `(S,G)` Join toward `192.0.2.10`.

It does not need to wait for RP1's next periodic SA message.

**SA caching reduces the delay between a receiver joining a group and the RP discovering an already active source.**

## SA Cache Aging

SA cache entries are maintained as **soft state**.

Each entry has an aging timer that is refreshed when a valid SA advertisement for the source and group is received.

For example:

```text
RP1                          RP2
 |                             |
 |---------- SA -------------> |
 |                       Cache Entry
 |                             |
 |       60 seconds            |
 |                             |
 |---------- SA -------------> |
 |                       Refresh Timer
 |                             |
```

If RP1 stops advertising the source, RP2's cache entry eventually expires.

The SA cache timer is distinct from:

- The MSDP peer Hold Timer.
- The SA Advertisement Period.
- PIM multicast routing state timers.

An SA cache entry may expire even while the MSDP TCP session remains established.

Likewise, a cached source can remain known for some time after the MSDP peer or source becomes unavailable, until the relevant state expires or is cleared.

### SA Caching on IOS XE

On current supported IOS XE releases, **MSDP SA caching is enabled by default and is mandatory**.

There is normally no need to configure an additional command to enable SA caching.

Older Cisco software documented:

```text
ip msdp cache-sa-state
```

This command was used to enable SA caching on releases where it was not enabled automatically.

For current IOS XE operation, focus on verifying the cache rather than manually enabling it.

## IOS XE SA Origination Filtering

By default, an MSDP-enabled RP originates SA messages for all locally registered multicast sources.

The **`ip msdp redistribute`** command controls which multicast `(S,G)` entries are eligible for **local SA origination**.

Despite its name, this command does not perform ordinary unicast route redistribution.

The syntax is:

```text
ip msdp redistribute [list <acl>] [asn <as-path-acl>] [route-map <name>]
```

| Option | Purpose |
|---|---|
| No options | Suppresses all locally originated SA messages |
| `list <acl>` | Filters locally originated SA messages based on source and group addresses |
| `asn <as-path-acl>` | Filters SA origination using a BGP AS-path access list |
| `route-map <name>` | Filters SA origination using a route map |

Multiple filtering options can be combined. When multiple filters are specified, **all conditions must be satisfied** for the source to be advertised.

### Default Behavior vs. `ip msdp redistribute`

There is an important distinction between **not configuring the command** and **configuring it without options**.

**Default: Command Not Configured**

When `ip msdp redistribute` is not configured, the RP automatically originates SA messages for all locally registered multicast sources.

For example, if RP1 has the following locally registered sources:

```text
(192.0.2.10, 239.1.1.1)
(192.0.2.20, 239.1.1.2)
```

RP1 originates SA messages advertising both sources.

No `ip msdp redistribute` configuration is required.

**`ip msdp redistribute` Without Options**

Configure:

```text
RP1(config)# ip msdp redistribute
```

This **disables local SA origination entirely**.

RP1 no longer originates SA messages for its locally registered sources.

However, it can still:

- Receive SA messages from other MSDP peers.
- Process and cache received SA information.
- Forward received SA messages to other peers, subject to the normal forwarding rules and filters.

**The command suppresses local SA origination, not MSDP operation as a whole.**

To remove the restriction and restore the default behavior:

```text
RP1(config)# no ip msdp redistribute
```

The behavior is summarized below:

| Configuration | Local SA Origination |
|---|---|
| Command not configured | All locally registered sources advertised |
| `ip msdp redistribute` | No local sources advertised |
| `ip msdp redistribute list <acl>` | Only permitted `(S,G)` entries advertised |
| `no ip msdp redistribute` | Restores default behavior |

### Permit Specific Sources and Groups

The `list` option restricts SA origination using an IPv4 ACL.

An **extended ACL** can match both the multicast source and group, whereas a standard ACL can match only the source address.

Suppose RP1 should advertise only:

```text
Source: 192.0.2.10
Group:  239.1.1.1
```

Configure an extended ACL:

```text
RP1(config)# ip access-list extended MSDP-ORIGINATE
RP1(config-ext-nacl)# permit ip host 192.0.2.10 host 239.1.1.1
```

Apply it:

```text
RP1(config)# ip msdp redistribute list MSDP-ORIGINATE
```

The extended ACL uses:

- **Source field:** Multicast source address, `S`.
- **Destination field:** Multicast group address, `G`.

For example:

| Source | Group | SA Origination |
|---|---|---|
| `192.0.2.10` | `239.1.1.1` | Permitted |
| `192.0.2.10` | `239.1.1.2` | Denied |
| `192.0.2.20` | `239.1.1.1` | Denied |

The ACL's implicit deny prevents unmatched source/group combinations from being originated as SA messages by RP1.

**Unlike the command without options, `ip msdp redistribute list` allows selected sources to be advertised instead of suppressing all local advertisements.**

### Filter by Source Address Only

The `ip msdp redistribute list` command requires an **extended IPv4 ACL**, even when filtering only by multicast source address.

To permit all sources in `192.0.2.0/24`, regardless of the multicast group:

```text
RP1(config)# ip access-list extended MSDP-SOURCES
RP1(config-ext-nacl)# permit ip 192.0.2.0 0.0.0.255 any

RP1(config)# ip msdp redistribute list MSDP-SOURCES
```

The extended ACL matches:

- **Source field:** Multicast source address, `S`.
- **Destination field:** Multicast group address, `G`.

Using `any` in the destination field allows the permitted sources to be advertised for any multicast group.

| Source | Group | SA Origination |
|---|---|---|
| `192.0.2.10` | `239.1.1.1` | Permitted |
| `192.0.2.20` | `239.2.2.2` | Permitted |
| `192.0.3.10` | `239.1.1.1` | Denied |

**Only extended ACLs are supported**, but either the source or group can be matched independently by using `any` for the other field.

### AS-Path and Route-Map Filtering

The command also supports more advanced filtering using:

```text
ip msdp redistribute asn <as-path-acl>
ip msdp redistribute route-map <name>
```

- **`asn`:** Uses a BGP AS-path access list to determine which sources are eligible for SA origination.
- **`route-map`:** Uses route-map matching criteria to control which multicast source entries are advertised.

These options are primarily relevant to more complex MSDP deployments involving multiple routing domains.

For a typical CCIE Enterprise Infrastructure **Anycast RP deployment**, filtering by `(S,G)` using the `list` option is generally sufficient.

### SA Origination vs. SA Forwarding

`ip msdp redistribute` controls **locally originated** SA messages.

It does not prevent the router from forwarding SA messages received from other MSDP peers.

For example, consider:

```text
RP1 -------- RP2 -------- RP3
```

Suppose RP2 is configured with:

```text
RP2(config)# ip msdp redistribute
```

RP2 will:

- **Not originate** SA messages for its own locally registered sources.
- **Still forward** valid SA messages received from RP1 toward RP3, subject to Peer-RPF and filtering rules.

To filter SA messages sent to or received from specific peers, use:

```text
ip msdp sa-filter in <peer-address> list <acl>
ip msdp sa-filter out <peer-address> list <acl>
```

The distinction is:

| Command | Function |
|---|---|
| `ip msdp redistribute` | Controls local SA origination |
| `ip msdp sa-filter in` | Controls SA messages accepted from a peer |
| `ip msdp sa-filter out` | Controls SA messages sent to a peer |

> **Note:** `ip msdp redistribute` can also be used to advertise sources known through multicast routing state that have not registered with the local RP. However, Cisco strongly recommends against advertising sources that have not registered with the RP.

### Verification

Check the configured SA origination policy:

```text
RP1# show running-config | include ip msdp redistribute
```

On the receiving RP, verify the advertised source information:

```text
RP2# show ip msdp sa-cache
```

Examine MSDP peer statistics:

```text
RP1# show ip msdp peer 10.0.0.2
```

When testing changes, remember that previously advertised SA entries may remain in the receiving RP's cache until they expire.

## Key Points

- By default, an MSDP-enabled RP advertises all locally registered sources.
- **`ip msdp redistribute` without options suppresses all local SA origination.**
- **`no ip msdp redistribute` restores the default behavior**, allowing locally registered sources to be advertised.
- `list` filters SA origination using a standard or extended IPv4 ACL.
- Extended ACLs match the `(S,G)` combination; standard ACLs match the source address.
- `asn` and `route-map` provide additional SA origination filtering options.
- `ip msdp redistribute` does not prevent forwarding SA messages received from other MSDP peers.
- Use `ip msdp sa-filter in/out` to control SA information exchanged with individual peers.

## Incoming and Outgoing SA Filtering

IOS XE supports filtering Source-Active (SA) messages received from or sent to individual MSDP peers.

Two commands are available:

```text
ip msdp sa-filter in <peer-address> [list <acl>] [rp-list <acl>]
ip msdp sa-filter out <peer-address> [list <acl>] [rp-list <acl>]
```

- **`in`:** Filters SA information received from the specified MSDP peer.
- **`out`:** Filters SA information sent to the specified MSDP peer.

Two ACL options are supported:

| Option | ACL Type | Matches |
|---|---|---|
| `list` | Extended IPv4 ACL | Source and group addresses `(S,G)` advertised in the SA |
| `rp-list` | Standard IPv4 ACL | Originating RP address carried in the SA |

Both options are optional and can be used independently or together.

- **Neither option:** Blocks **all SA messages** in the specified direction for that peer.
- **`list` only:** Filters by `(S,G)`, regardless of the originating RP.
- **`rp-list` only:** Filters by originating RP, regardless of the advertised `(S,G)`.
- **Both configured:** The SA entry must be permitted by **both ACLs**.

### Filtering All SA Messages from a Peer

To block all incoming SA messages from RP1 (`10.0.0.1`) on RP2:

```text
RP2(config)# ip msdp sa-filter in 10.0.0.1
```

To prevent RP1 from sending any SA messages to RP2 (`10.0.0.2`):

```text
RP1(config)# ip msdp sa-filter out 10.0.0.2
```

These commands block SA information in the specified direction without requiring an ACL.

**The MSDP TCP session remains established**, but SA information is blocked by the configured filter.

To restore the default behavior, allowing SA messages without filtering:

```text
RP2(config)# no ip msdp sa-filter in 10.0.0.1
RP1(config)# no ip msdp sa-filter out 10.0.0.2
```

**Important:** Not configuring `ip msdp sa-filter` allows SA messages by default. Configuring it with only a peer address blocks all SA messages in the specified direction.

### Incoming SA Filter Using `list`

An incoming SA filter controls which SA information a router accepts from a specified MSDP peer.

Suppose RP2 should accept only:

```text
Source: 192.0.2.10
Group:  239.1.1.1
```

Configure an extended ACL:

```text
RP2(config)# ip access-list extended MSDP-ACCEPT
RP2(config-ext-nacl)# permit ip host 192.0.2.10 host 239.1.1.1
```

Apply it:

```text
RP2(config)# ip msdp sa-filter in 10.0.0.1 list MSDP-ACCEPT
```

The extended ACL matches:

- **Source field:** Multicast source address, `S`.
- **Destination field:** Multicast group address, `G`.

For example:

| Source | Group | Result |
|---|---|---|
| `192.0.2.10` | `239.1.1.1` | Accepted |
| `192.0.2.10` | `239.1.1.2` | Rejected |
| `192.0.2.20` | `239.1.1.1` | Rejected |

The ACL's implicit deny rejects unmatched `(S,G)` entries.

### Incoming SA Filter Using `rp-list`

The `rp-list` option filters SA messages based on the **originating RP address carried inside the SA message**.

This is not necessarily the same as the MSDP peer address.

For example, RP2 might receive an SA message from RP1 that originally came from another RP.

```text
MSDP Peer:      10.0.0.1
Originating RP: 10.0.0.3
Source:         192.0.2.10
Group:          239.1.1.1
```

The `rp-list` matches `10.0.0.3`, not `10.0.0.1`.

To accept only SA messages originated by RP1 (`10.0.0.1`), configure a **standard ACL**:

```text
RP2(config)# ip access-list standard MSDP-ORIGINATORS
RP2(config-std-nacl)# permit host 10.0.0.1
```

Apply it:

```text
RP2(config)# ip msdp sa-filter in 10.0.0.1 rp-list MSDP-ORIGINATORS
```

The behavior is:

| MSDP Peer | Originating RP | Result |
|---|---|---|
| `10.0.0.1` | `10.0.0.1` | Accepted |
| `10.0.0.1` | `10.0.0.2` | Rejected |
| `10.0.0.1` | `10.0.0.3` | Rejected |

The `rp-list` ACL does not examine the multicast source or group addresses.

**`rp-list` filters based on where the SA originated, not necessarily which peer forwarded it.**

### Combining `list` and `rp-list`

Both options can be configured together.

For example, suppose RP2 should accept SA information from RP1 only if:

- The originating RP is `10.0.0.1`.
- The source is `192.0.2.10`.
- The group is `239.1.1.1`.

Configure the extended ACL:

```text
RP2(config)# ip access-list extended MSDP-GROUPS
RP2(config-ext-nacl)# permit ip host 192.0.2.10 host 239.1.1.1
```

Configure the standard ACL:

```text
RP2(config)# ip access-list standard MSDP-RPS
RP2(config-std-nacl)# permit host 10.0.0.1
```

Apply both:

```text
RP2(config)# ip msdp sa-filter in 10.0.0.1 list MSDP-GROUPS rp-list MSDP-RPS
```

The resulting behavior is:

| Originating RP | Source | Group | Result |
|---|---|---|---|
| `10.0.0.1` | `192.0.2.10` | `239.1.1.1` | Accepted |
| `10.0.0.1` | `192.0.2.20` | `239.1.1.1` | Rejected |
| `10.0.0.1` | `192.0.2.10` | `239.1.1.2` | Rejected |
| `10.0.0.3` | `192.0.2.10` | `239.1.1.1` | Rejected |

**Both ACLs must permit their respective fields for the SA information to be accepted.**

A deny in either ACL, including an implicit deny, causes the matching SA information to be rejected.

### Outgoing SA Filtering

An outgoing SA filter controls which SA information is sent to a specified MSDP peer.

For example, RP1 should advertise only:

```text
Source: 192.0.2.10
Group:  239.1.1.1
```

Configure an extended ACL:

```text
RP1(config)# ip access-list extended MSDP-ADVERTISE
RP1(config-ext-nacl)# permit ip host 192.0.2.10 host 239.1.1.1
```

Apply it:

```text
RP1(config)# ip msdp sa-filter out 10.0.0.2 list MSDP-ADVERTISE
```

RP1 sends only the permitted `(S,G)` information to RP2.

The `rp-list` option is also supported for outgoing filters:

```text
RP1(config)# ip access-list standard MSDP-RPS
RP1(config-std-nacl)# permit host 10.0.0.1

RP1(config)# ip msdp sa-filter out 10.0.0.2 rp-list MSDP-RPS
```

This permits SA information originated by `10.0.0.1` to be sent to RP2, regardless of the multicast source and group.

The `list` and `rp-list` options can also be combined for outgoing filtering, using the same AND logic as incoming filters.

Unlike `ip msdp redistribute`, an outgoing SA filter applies to both:

- SA messages originated locally.
- SA messages received from other peers and forwarded toward the specified peer.

### Comparison of SA Filtering Commands

| Command | Function |
|---|---|
| `ip msdp redistribute` | Controls local SA origination |
| `ip msdp sa-filter in ... list` | Filters received SA information by `(S,G)` |
| `ip msdp sa-filter in ... rp-list` | Filters received SA information by originating RP |
| `ip msdp sa-filter out ... list` | Filters outgoing SA information by `(S,G)` |
| `ip msdp sa-filter out ... rp-list` | Filters outgoing SA information by originating RP |

**These filters control MSDP source advertisements. They are not multicast data-plane ACLs.**

However, rejecting an SA advertisement can indirectly prevent multicast connectivity if a remote RP needs that advertisement to discover the source.

## Limiting SA Cache Entries

MSDP can learn large numbers of active multicast sources.

IOS XE provides the following command to limit how much SA state is accepted from a particular peer:

```text
ip msdp sa-limit <peer-address> <limit>
```

For example, on RP2:

```text
RP2(config)# ip msdp sa-limit 10.0.0.1 100
```

This limits the amount of SA state RP2 accepts into its cache from RP1.

Once the configured limit is reached, additional SA information from the peer is ignored until cache capacity becomes available.

**`ip msdp sa-limit` limits cached SA state, not the number of SA messages received per second.**

This command can protect the router against excessive SA state caused by misconfiguration or malicious advertisements.

The limit should account for the number of active `(S,G)` sources expected from the peer.

## IOS XE Verification

### Verify MSDP Peering

Before troubleshooting SA advertisements, verify that the TCP session is established.

```text
RP1# show ip msdp summary
RP2# show ip msdp summary
```

Both RPs should report the peer relationship as **Up**.

An MSDP session can be established even when no active sources are being advertised.

### Display the SA Cache

On RP2:

```text
RP2# show ip msdp sa-cache
```

A learned entry for the example source should identify:

```text
(S,G):          (192.0.2.10, 239.1.1.1)
Originating RP: 10.0.0.1
Peer:           10.0.0.1
```

Actual IOS XE output also includes timers and other information, depending on the platform and release.

The important fields are:

| Field | Meaning |
|---|---|
| `(S,G)` | Advertised source and multicast group |
| `RP` | RP Address field from the SA message |
| `Peer` | MSDP peer from which the SA was learned |
| Timers | Age and remaining lifetime of cached state |

If the cache entry is being refreshed by periodic SA messages, its expiration timer should be refreshed.

### Display SA Counts

```text
RP2# show ip msdp count
```

This displays SA state counters, including the number of source and group entries learned from MSDP peers.

The command is useful for determining whether RP2 is accumulating the expected source information.

### Verify Detailed Peer Statistics

```text
RP2# show ip msdp peer 10.0.0.1
```

Examine:

- SA messages received.
- SA messages sent.
- SA information stored in the cache.
- Configured incoming and outgoing filters.
- Peer status and reset counters.

### Compare the SA Cache and Multicast Routing Table

On RP2:

```text
RP2# show ip msdp sa-cache
RP2# show ip mroute 239.1.1.1
```

Interpret the results carefully:

| SA Cache | Multicast Routing State | Possible Explanation |
|---|---|---|
| Entry present | No `(S,G)` forwarding entry | Source learned, but no interested receivers |
| Entry present | `(S,G)` present | Source learned and source-tree state established |
| Entry absent | No `(S,G)` | Source not learned through MSDP |
| Entry absent | `(S,G)` present | Source learned through another mechanism, or SA state expired |

**The SA cache and multicast routing table are separate databases. An entry in one does not guarantee a corresponding entry in the other.**

### Clear the SA Cache

Clear all cached SA information:

```text
RP2# clear ip msdp sa-cache
```

Clear SA cache entries for a specific group:

```text
RP2# clear ip msdp sa-cache 239.1.1.1
```

This can be useful for observing how new SA advertisements repopulate the cache.

Clearing the SA cache does not directly remove all PIM multicast forwarding state.

### Debug SA Processing

For detailed MSDP debugging:

```text
RP2# debug ip msdp
```

This can display received SA messages, Peer-RPF results, and SA forwarding activity.

On IOS XE platforms supporting the option, use:

```text
RP2# debug ip msdp routes
```

Disable debugging when finished:

```text
RP2# undebug all
```

## Common Troubleshooting Problems

| Problem | Possible Cause |
|---|---|
| RP1 does not originate SA messages | Source has not registered locally, MSDP not operational, or SA origination filter |
| RP2 receives no SA messages | MSDP peering down, outgoing SA filter, or no active registered sources |
| RP2 receives SAs but rejects them | Peer-RPF failure or incoming SA filter |
| SA message has the shared Anycast RP address | Missing or incorrect `ip msdp originator-id` |
| SA cache entry disappears | Periodic advertisements stopped or entry aged out |
| SA cache contains source but no `(S,G)` forwarding state | No receiver interest or a PIM/RPF problem |
| RP2 builds an `(S,G)` entry but traffic does not arrive | Multicast data-plane RPF problem, upstream forwarding failure, or filtering |
| Unexpected sources appear in SA cache | SA origination or filtering policy too permissive |
| Additional SA information is rejected | Configured `ip msdp sa-limit` reached |

When troubleshooting, distinguish three separate operations:

1. **MSDP peering:** Is the TCP session established?
2. **MSDP source discovery:** Are valid SA messages being received and cached?
3. **PIM forwarding:** Does the receiving RP establish the required multicast forwarding state?

A successful MSDP session does not guarantee successful source discovery, and successful source discovery does not guarantee multicast data delivery.

## Key Points

- **MSDP SA messages advertise active multicast sources** between RPs.
- An RP normally originates SA messages for **locally registered sources**.
- IPv4 Source-Active messages use **MSDP TLV Type 1**.
- SA messages contain the originating RP address and one or more `(S,G)` entries.
- In an Anycast RP deployment, the SA RP Address field must contain the unique Originator ID, **not the shared Anycast RP address**.
- The default **SA Advertisement Period is 60 seconds**.
- An initial SA message may contain an encapsulated multicast data packet.
- Periodic SA advertisements normally contain source information without multicast data.
- MSDP speakers may forward received SA messages, subject to Peer-RPF and filtering rules.
- **The SA cache stores active source information learned through MSDP**.
- SA cache entries are maintained as soft state and expire without refreshes.
- An SA cache entry does not necessarily create an `(S,G)` multicast forwarding entry.
- A receiving RP can use its SA cache to build source trees immediately when new receivers join.
- Current supported IOS XE releases enable SA caching by default.
- `ip msdp redistribute` controls locally originated SA information.
- `ip msdp sa-filter in/out` controls SA information received from or sent to individual peers.
- `ip msdp sa-limit` restricts SA state accepted into the cache from a peer.
- Verify SA information using `show ip msdp sa-cache`, `show ip msdp count`, and `show ip msdp peer`.

## References

- [Cisco IOS XE: Using MSDP to Interconnect Multiple PIM-SM Domains](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-multicast/b-ip-multicast/m_imc_msdp_im_pim_sim.html)
- [Cisco IOS XE: Multicast Configuration Guide, MSDP](https://www.cisco.com/c/en/us/td/docs/switches/lan/c9000/multicast/multicast-configuration-guide/msdp.html)
- [Cisco: Multicast Source Discovery Protocol SA Filter Recommendations](https://www.cisco.com/c/en/us/support/docs/ip/ip-multicast/13717-49.html)
- [RFC 3618: Multicast Source Discovery Protocol](https://www.rfc-editor.org/rfc/rfc3618)
- [RFC 3446: Anycast Rendezvous Point Mechanism Using PIM and MSDP](https://www.rfc-editor.org/rfc/rfc3446)