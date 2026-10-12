# Candidate BSR and BSR Election

In PIM Sparse Mode (PIM-SM), a **Candidate Bootstrap Router (C-BSR)** is a router configured to participate in the election of the **Bootstrap Router (BSR)**.

The elected BSR is responsible for collecting Candidate RP information and distributing the resulting **RP-Set** throughout the PIM domain.

Multiple routers can be configured as Candidate BSRs, but normally **only one BSR is elected for a given PIM domain**.

The BSR election is performed using **PIM Bootstrap Messages (BSMs)**, rather than a separate election message type.

## BSR Election Rules

Candidate BSRs are compared using two values, in this order:

1. **Highest BSR priority wins.**
2. If priorities are equal, **highest BSR IP address wins**.

The BSR priority ranges from **0 to 255**, with 255 being the highest.

For example:

| Candidate BSR | BSR Address | Priority |
|---|---|---|
| R1 | `1.1.1.1` | 10 |
| R2 | `2.2.2.2` | 20 |
| R3 | `3.3.3.3` | 20 |

R3 wins because R2 and R3 have the highest priority, and R3 has the higher IP address.

The IP address is only considered when priorities are equal.

> **Important:** BSR election priority and RP selection priority work in opposite directions.
>
> - **BSR election:** Highest priority wins.
> - **RP selection:** Lowest priority wins.

## BSR Election Process

Consider the following topology:

```text
               R1 (C-BSR)
                 1.1.1.1
               Priority 10
                    |
                    R3
                    |
               R2 (C-BSR)
                 2.2.2.2
               Priority 20
```

Both R1 and R2 are configured as Candidate BSRs.

### Step 1: Candidates Advertise Their BSR Information

BSR election information is carried in **Bootstrap Messages (PIM Type 4)**.

Each BSM includes:

- BSR address.
- BSR priority.
- Hash mask length.
- RP-Set information, if available.

For IPv4, Bootstrap Messages normally use:

```text
Destination IP: 224.0.0.13
IP Protocol:    103 (PIM)
PIM Type:       4 (Bootstrap)
TTL:            1
```

The elected BSR's address is carried inside the BSM.

**The source IP address of the BSM is the address of the router transmitting that particular hop**, which may be different from the elected BSR's address.

### Step 2: BSMs Propagate Through the PIM Domain

Bootstrap Messages are propagated hop by hop between PIM neighbors.

For example:

```text
R1 (Candidate BSR)
      |
      | BSM
      v
      R3
      |
      | BSM
      v
      R2 (Candidate BSR)
```

Each forwarding router regenerates the BSM with a TTL of 1, allowing it to reach other parts of the PIM domain.

Unlike Auto-RP, **BSR does not require an existing multicast distribution tree** to distribute its messages.

However, BSM forwarding is subject to PIM neighbor checks, RPF checks toward the BSR, and configured BSR boundaries.

### Step 3: Candidates Compare BSR Information

R1 and R2 compare the BSR information they learn.

In this example:

```text
R1: Priority 10, Address 1.1.1.1
R2: Priority 20, Address 2.2.2.2
```

R2 is preferred because its priority is higher.

When R1 learns about the preferred BSR, it stops attempting to become the elected BSR and accepts R2's BSMs.

R2 becomes the elected BSR after the election process completes.

```text
R1: Candidate BSR (not elected)
R2: Elected BSR
```

**The non-elected Candidate BSR remains available to take over if the elected BSR fails.**

## Candidate BSR States

RFC 5059 defines three Candidate BSR states.

| State | Description |
|---|---|
| **Candidate-BSR (C-BSR)** | Another, more preferred BSR is currently known |
| **Pending-BSR (P-BSR)** | No more preferred BSR is known, but the router has not yet completed the election |
| **Elected-BSR (E-BSR)** | The router is the elected BSR and originates periodic BSMs |

A newly configured Candidate BSR initially enters the **Pending-BSR** state.

### Pending-BSR

The Pending-BSR state introduces a delay before a Candidate BSR begins acting as the elected BSR.

This prevents multiple candidates from immediately declaring themselves elected and causing unnecessary changes.

The delay is determined by a **randomized override timer**.

RFC 5059 defines a randomized interval influenced by the candidate's priority and address.

The resulting interval is approximately **5 to 23 seconds**:

- Higher-priority candidates generally use shorter delays.
- Lower-priority candidates generally use longer delays.

This gives more preferred candidates an opportunity to become elected first.

If a Pending-BSR receives a BSM from a more preferred BSR, it transitions to Candidate-BSR instead.

If the timer expires without learning about a more preferred BSR, it becomes the Elected-BSR.

### Simplified State Transitions

```text
                 Candidate BSR
                       |
                       v
                  Pending-BSR
                    /     \
                   /       \
        Preferred BSM      Timer expires
               |                |
               v                v
          Candidate-BSR     Elected-BSR
               |                |
               |                |
         Current BSR       Preferred BSM
           expires           received
               |                |
               v                v
          Pending-BSR      Candidate-BSR
```

This is a simplified representation of the RFC state machine. The complete behavior also accounts for changes to BSR priority and reception of non-preferred BSMs.

## Bootstrap Timer and BSR Failure

Once elected, the BSR periodically originates Bootstrap Messages.

RFC 5059 defines these default timers:

| Timer | Default | Purpose |
|---|---|---|
| **Bootstrap Period** | 60 seconds | Interval between periodic BSMs |
| **Bootstrap Timeout** | 130 seconds | Time after which a learned BSR expires without receiving valid BSMs |
| **Minimum Bootstrap Interval** | 10 seconds | Minimum interval between originated BSMs |
| **Randomized Override Interval** | Approximately 5 to 23 seconds | Election delay used by Candidate BSRs |

### Example: BSR Failure

Consider:

```text
               R1 (Elected BSR)
                 Priority 20
                      |
                      R3
                      |
               R2 (Candidate BSR)
                 Priority 10
```

Initially, R1 is elected.

R2 receives R1's BSMs and maintains its Candidate-BSR state.

If R1 fails:

1. R2 stops receiving valid BSMs from R1.
2. The **Bootstrap Timeout** expires, normally after 130 seconds.
3. R2 enters the Pending-BSR state.
4. After its randomized election delay, R2 becomes the Elected-BSR.
5. R2 begins originating BSMs advertising itself as the BSR.

Other routers learn the new BSR through BSM propagation.

> **Note:** BSR failover is not instantaneous. The Bootstrap Timeout and election delay contribute to convergence time. Group-to-RP mappings have separate expiration mechanisms and are not necessarily discarded immediately when the current BSR expires.

## BSR Address and Hash Mask Length

A Candidate BSR advertises a **BSR address** and a **hash mask length**.

These serve different purposes.

| Parameter | Purpose |
|---|---|
| **BSR Address** | Identifies the BSR and serves as the election tie-breaker |
| **BSR Priority** | Primary election criterion |
| **Hash Mask Length** | Controls the group-address bits used in RP hashing |

The hash mask length **does not influence the BSR election**.

It is advertised by the elected BSR and used by PIM routers when selecting an RP from multiple eligible Candidate RPs with equal priority.

For example, with IPv4:

```text
Hash Mask Length: 30
```

The first 30 bits of the multicast group address are used as input to the RP hash calculation.

Groups sharing those 30 bits use the same masked group value when calculating the hash.

RFC 5059 recommends:

- **IPv4:** 30 bits.
- **IPv6:** 126 bits.

The hash calculation itself is covered in [BSR Hashing](hashing.md).

## IPv4 IOS XE Configuration

Configure R1 as a Candidate BSR using its Loopback0 address.

```text
R1(config)# ip multicast-routing

R1(config)# interface Loopback0
R1(config-if)# ip address 1.1.1.1 255.255.255.255
R1(config-if)# ip pim sparse-mode

R1(config)# ip pim bsr-candidate Loopback0 30 10
```

The command arguments are:

| Argument | Meaning |
|---|---|
| `Loopback0` | Interface supplying the BSR address |
| `30` | Hash mask length |
| `10` | BSR priority |

The configured interface must be PIM-enabled, and its address must be reachable throughout the PIM domain.

### Configure a Second Candidate BSR

On R2:

```text
R2(config)# interface Loopback0
R2(config-if)# ip address 2.2.2.2 255.255.255.255
R2(config-if)# ip pim sparse-mode

R2(config)# ip pim bsr-candidate Loopback0 30 20
```

R2 has the higher priority and therefore becomes the preferred BSR.

### Default Values

On Cisco IOS XE:

| Parameter | Default |
|---|---|
| Candidate BSR | Disabled |
| BSR Priority | **0** |
| Hash Mask Length | **0** |

RFC 5059 specifies a recommended default BSR priority of **64**, whereas Cisco IOS XE uses **0** as its implementation default.

Therefore, it is generally preferable to configure the intended BSR priority and hash mask length explicitly.

> **Note:** BSR priority and hash mask length are independent. The hash mask length can affect RP distribution across multicast groups, but it does not determine which Candidate BSR wins the election.

## IPv6 IOS XE Configuration

IPv6 supports the same election mechanism, but uses different command syntax.

For example, configure R1 as a Candidate BSR:

```text
R1(config)# ipv6 unicast-routing
R1(config)# ipv6 multicast-routing

R1(config)# interface Loopback0
R1(config-if)# ipv6 address 2001:DB8:1::1/128

R1(config)# ipv6 pim bsr candidate bsr 2001:DB8:1::1 126 priority 10
```

The arguments are:

| Argument | Meaning |
|---|---|
| `2001:DB8:1::1` | Candidate BSR IPv6 address |
| `126` | IPv6 hash mask length |
| `priority 10` | BSR election priority |

Configure another Candidate BSR:

```text
R2(config)# ipv6 pim bsr candidate bsr 2001:DB8:2::2 126 priority 20
```

R2 is preferred because it has the higher BSR priority.

As with IPv4, the BSR address must be reachable throughout the relevant PIM domain.

### IPv4 vs. IPv6 Syntax

| Operation | IPv4 | IPv6 |
|---|---|---|
| Candidate BSR | `ip pim bsr-candidate Loopback0 30 10` | `ipv6 pim bsr candidate bsr 2001:DB8:1::1 126 priority 10` |
| BSR election priority | Highest wins | Highest wins |
| Address tie-breaker | Highest IPv4 address | Highest IPv6 address |
| Recommended hash mask | 30 bits | 126 bits |
| Default BSR priority | 0 | 0 |
| Default hash mask | 0 bits | 126 bits |
| BSM destination | `224.0.0.13` | `FF02::D` |

## Filtering Candidate RPs with `accept-rp-candidate`

By default, the elected BSR accepts valid **Candidate-RP-Advertisement (C-RP-Adv) messages** from any Candidate RP.

IOS XE supports the optional **`accept-rp-candidate`** keyword to restrict which Candidate RPs are accepted by the BSR.

This is useful for preventing unauthorized or incorrectly configured routers from advertising themselves as RPs.

The filter is configured as part of the Candidate BSR command. It is not a separate global command.

### IPv4 Standard ACL

For IPv4, the simplest approach is to use a **standard IPv4 ACL** identifying the permitted Candidate RP addresses.

For example, allow only `3.3.3.3` and `4.4.4.4` to become Candidate RPs:

```text
R1(config)# ip access-list standard ALLOWED-C-RPS
R1(config-std-nacl)# permit host 3.3.3.3
R1(config-std-nacl)# permit host 4.4.4.4

R1(config)# ip pim bsr-candidate Loopback0 30 10 accept-rp-candidate ALLOWED-C-RPS
```

The behavior is:

| Candidate RP | Accepted? |
|---|---|
| `3.3.3.3` | Yes |
| `4.4.4.4` | Yes |
| `5.5.5.5` | No |

Candidate RP advertisements that do not match a `permit` statement are rejected by the BSR.

The rejected Candidate RPs are not included in the RP-Set distributed by that BSR.

**The ACL identifies allowed Candidate RP addresses, not multicast group addresses.**

### IPv4 Extended ACL

IOS XE also supports **extended IPv4 ACLs** for Candidate RP filtering.

Extended ACLs allow matching PIM protocol traffic and IP source/destination fields.

For example:

```text
R1(config)# ip access-list extended ALLOWED-C-RPS
R1(config-ext-nacl)# permit pim host 3.3.3.3 any
R1(config-ext-nacl)# permit pim host 4.4.4.4 any

R1(config)# ip pim bsr-candidate Loopback0 30 10 accept-rp-candidate ALLOWED-C-RPS
```

Here, `pim` matches IP protocol 103. `ip` also works.

The source field identifies the permitted Candidate RP advertisement senders, while `any` allows any destination address.

> **Note:** When using an extended ACL, consider that the outer source address of a unicast C-RP advertisement and its advertised RP address are distinct fields. The precise matching behavior should be verified on the target IOS XE platform. A standard ACL matching the intended Candidate RP addresses is generally the simpler configuration.

### IPv6 ACL

For IPv6, the filter is configured using an **IPv6 ACL**.

For example, permit only Candidate RP `2001:DB8:3::3`:

```text
R1(config)# ipv6 access-list ALLOWED-C-RPS-V6
R1(config-ipv6-acl)# permit ipv6 host 2001:DB8:3::3 any

R1(config)# ipv6 pim bsr candidate bsr 2001:DB8:1::1 126 priority 10 accept-rp-candidate ALLOWED-C-RPS-V6
```

The IPv6 ACL uses an extended-style source/destination format.

Cisco documents matching permitted Candidate RP addresses in the **source field** and using `any` for the destination field.

Other Candidate RPs are denied by the implicit deny at the end of the ACL.

### Candidate RP Filtering vs. Group Filtering

It is important to distinguish **filtering which routers can become RPs** from **filtering which multicast groups they advertise**.

| Configuration | Applied On | Purpose |
|---|---|---|
| `ip pim bsr-candidate ... accept-rp-candidate` | Candidate BSR | Restricts accepted IPv4 Candidate RPs |
| `ipv6 pim bsr candidate bsr ... accept-rp-candidate` | Candidate BSR | Restricts accepted IPv6 Candidate RPs |
| `ip pim rp-candidate ... group-list` | Candidate RP | Restricts advertised IPv4 multicast group ranges |
| `ipv6 pim bsr candidate rp ... group-list` | Candidate RP | Restricts advertised IPv6 multicast group ranges |

For example, R3 can advertise itself only for `239.1.0.0/16`:

```text
R3(config)# ip access-list standard RP-GROUPS
R3(config-std-nacl)# permit 239.1.0.0 0.0.255.255

R3(config)# ip pim rp-candidate Loopback0 group-list RP-GROUPS
```

Meanwhile, R1 can restrict which Candidate RPs it accepts:

```text
R1(config)# ip access-list standard ALLOWED-C-RPS
R1(config-std-nacl)# permit host 3.3.3.3

R1(config)# ip pim bsr-candidate Loopback0 30 10 accept-rp-candidate ALLOWED-C-RPS
```

The combined result is:

- R3 advertises itself as a Candidate RP for `239.1.0.0/16`.
- R1 accepts R3's Candidate RP advertisements.
- Other Candidate RPs are rejected by R1's filter.

**`accept-rp-candidate` filters RP candidacy at the BSR; `group-list` controls which group prefixes a Candidate RP advertises.**

### Filtering and BSR Redundancy

The filtering configuration matters on the **elected BSR**.

If multiple Candidate BSRs are configured for redundancy, they should use consistent Candidate RP acceptance policies.

Otherwise, a BSR failover could change which Candidate RPs are accepted and advertised throughout the domain.

> **Important:** `accept-rp-candidate` does not block PIM Bootstrap Messages, multicast data packets, or source registration. It controls which Candidate RP information the elected BSR accepts. Command availability and ACL handling can vary by IOS XE platform.

## BSR Election and RPF

Bootstrap Messages use a special hop-by-hop forwarding process.

When a router receives a BSM, it verifies that the message arrived from an acceptable upstream PIM neighbor toward the BSR identified inside the message.

In particular, RFC 5059 requires ordinary multicast BSMs to pass an **RPF neighbor check toward the BSR address**.

For example:

```text
             R1 (BSR)
                 |
                 R2
                /  \
               /    \
              R3    R4
```

R3 learns that R1 is the elected BSR.

Suppose its RPF path toward R1 points through R2.

A BSM received from R2 is valid from an RPF perspective.

However, if R3 receives an otherwise ordinary BSM identifying R1 as the BSR from an unexpected PIM neighbor, it fails the BSM RPF check and is discarded.

This helps prevent loops and the acceptance of incorrectly propagated BSMs.

A valid BSM must also originate from a directly connected PIM neighbor with active Hello state.

> **Important:** The BSM RPF lookup is performed toward the **BSR address**, not toward a multicast source, receiver, or Candidate RP.

Special startup mechanisms, such as No-Forward BSMs and certain unicast BSMs, have additional processing rules.

## BSR Borders

BSR messages can be prevented from crossing an interface using the **BSR border** feature.

IPv4:

```text
R1(config)# interface Ethernet0/0
R1(config-if)# ip pim bsr-border
```

IPv6:

```text
R1(config)# interface Ethernet0/0
R1(config-if)# ipv6 pim bsr border
```

These commands prevent Bootstrap Messages from being sent or accepted through the interface.

This allows separate PIM domains to maintain independent BSR elections.

The feature does not directly block other PIM message types or multicast data packets.

## IOS XE Verification

### IPv4 BSR Information

```text
R1# show ip pim bsr-router
```

This displays information about the elected BSR, including its address, priority, and hash mask length.

On the elected BSR, verify that the router identifies itself as the BSR.

On other routers, verify that the same BSR is learned.

Additional commands:

```text
R1# show ip pim rp mapping
R1# show ip pim rp-hash 239.1.1.1
```

These verify that BSR information has resulted in usable group-to-RP mappings.

### Verify Candidate RP Filtering

On the elected BSR, verify that the filter is configured:

```text
R1# show running-config | include ip pim bsr-candidate
R1# show access-lists ALLOWED-C-RPS
```

Verify learned RP mappings:

```text
R1# show ip pim rp mapping
```

Only permitted Candidate RPs should appear among the BSR-learned mappings.

For IPv6:

```text
R1# show running-config | include ipv6 pim bsr candidate bsr
R1# show ipv6 access-list ALLOWED-C-RPS-V6
R1# show ipv6 pim bsr rp-cache
```

The RP cache is particularly useful because it shows Candidate RP information received by the elected BSR.

Compare the cache with the permitted addresses in the ACL.

### IPv6 BSR Information

```text
R1# show ipv6 pim bsr election
```

Example output:

```text
R1# show ipv6 pim bsr election
PIMv2 BSR information

BSR Election Information
  Scope Range List: ff00::/8
  This system is the Bootstrap Router (BSR)
  BSR Address: 2001:DB8:1::1
  Uptime: 00:11:55, BSR Priority: 10, Hash mask length: 126
  RPF: FE80::2,Ethernet0/0
  BS Timer: 00:00:07
  This system is candidate BSR
  Candidate BSR address: 2001:DB8:1::1, priority: 10, hash mask length: 126
```

Important fields include:

| Field | Meaning |
|---|---|
| **BSR Address** | Currently elected BSR |
| **BSR Priority** | Election priority |
| **Hash mask length** | Mask used in RP hashing |
| **RPF** | RPF neighbor and interface associated with the BSR |
| **BS Timer** | Time until the next BSM is originated on the elected BSR, or until BSR information expires on another router |
| **Candidate BSR address** | Local candidate's configured address |

Additional IPv6 verification:

```text
R1# show ipv6 pim bsr rp-cache
R1# show ipv6 pim group-map
```

The first displays learned Candidate RP information on the elected BSR.

The second displays group-to-RP mappings.

## Troubleshooting BSR Election

| Problem | Possible Cause |
|---|---|
| Unexpected BSR elected | Incorrect priorities or higher BSR address in a priority tie |
| No BSR learned | Missing PIM neighbors, invalid RPF path, BSM filtering, or BSR border |
| Two different BSRs appear elected | Partitioned BSM propagation, missing connectivity, or inconsistent BSR boundaries |
| BSR election succeeds but no RP mappings exist | No Candidate RP advertisements received |
| Candidate RP missing from the BSR RP-Set | `accept-rp-candidate` ACL denies it, or its advertisements are not reaching the BSR |
| RP mappings change after BSR failover | Candidate BSRs use inconsistent acceptance policies |
| BSR election continually changes | Unstable connectivity, intermittent BSM loss, or changing Candidate BSR state |
| IPv6 BSR not learned | Incorrect BSR address, lack of reachability, or IPv6 PIM/BSM forwarding problem |

A useful troubleshooting sequence is:

1. Verify the Candidate BSR configuration.
2. Verify that the BSR addresses are reachable.
3. Verify PIM neighbors along the path.
4. Verify that Bootstrap Messages can cross the required interfaces.
5. Verify the elected BSR on multiple routers.
6. Verify that Candidate RPs are advertising and their mappings are distributed.
7. If a Candidate RP is missing, check any `accept-rp-candidate` ACL on the elected BSR.

## Key Points

- **Candidate BSRs participate in an election to determine which router distributes RP information.**
- The **highest BSR priority wins**.
- If priorities are equal, the **highest BSR IP address wins**.
- The BSR election uses **PIM Bootstrap Messages (Type 4)**.
- A Candidate BSR can be in **Candidate-BSR, Pending-BSR, or Elected-BSR** state.
- The Pending-BSR state introduces a **randomized election delay**, approximately 5 to 23 seconds.
- The elected BSR normally originates BSMs every **60 seconds**.
- Other routers expire learned BSR information after **130 seconds** without valid BSMs.
- The BSR address is used for election tie-breaking and BSM RPF processing.
- The hash mask length influences RP hashing, **not BSR election**.
- IOS XE defaults to a BSR priority of **0**, whereas RFC 5059 recommends 64.
- IPv4 uses `ip pim bsr-candidate`.
- IPv6 uses `ipv6 pim bsr candidate bsr`.
- **`accept-rp-candidate` restricts which Candidate RPs the elected BSR accepts.**
- IPv4 supports **standard and extended ACLs** for Candidate RP filtering; IPv6 uses an **IPv6 ACL**.
- Candidate RP filtering is distinct from a Candidate RP's `group-list`, which controls advertised multicast group ranges.
- BSR border interfaces prevent BSMs from crossing between multicast domains.

## References

- [RFC 5059, Section 3.1: Bootstrap Router Election](https://www.rfc-editor.org/rfc/rfc5059.html#section-3.1)
- [RFC 5059, Section 5: Timers and Timer Values](https://www.rfc-editor.org/rfc/rfc5059.html#section-5)
- [Cisco IOS XE: Configuring Basic IP Multicast](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/ip-multicast/ip-multicast/m_imc_basic_cfg-0.html)
- [Cisco IOS XE: IPv6 Multicast Bootstrap Router](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/ip-multicast/ip-multicast/m_ip6-mcast-bsr.html)
- [Cisco IOS XE: `ip pim bsr-candidate` Command Reference](https://www.cisco.com/c/en/us/td/docs/routers/sdwan/command/iosxe/qualified-cli-command-reference-guide/m-ip-commands.html)
- [Cisco IPv6 Command Reference: `ipv6 pim bsr candidate bsr`](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/ipv6/command/ipv6-cr-book/ipv6-i4.html)