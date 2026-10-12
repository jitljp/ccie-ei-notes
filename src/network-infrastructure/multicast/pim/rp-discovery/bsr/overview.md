# BSR

The **Bootstrap Router (BSR)** mechanism allows PIM Sparse Mode (PIM-SM) routers to dynamically learn Rendezvous Point (RP) information.

Rather than manually configuring an RP address on every router, BSR distributes information about available RPs throughout the PIM domain.

BSR is a standards-based mechanism defined in **RFC 5059** and supports both IPv4 and IPv6.

The mechanism involves three key components:

| Component | Role |
|---|---|
| **Candidate BSR (C-BSR)** | Participates in the election to select the active BSR |
| **Candidate RP (C-RP)** | Advertises its willingness to serve as an RP for specified multicast groups |
| **Elected BSR** | Collects C-RP information and distributes it throughout the PIM domain |

**The BSR does not have to be an RP.** The two functions can reside on separate routers or on the same router.

## BSR Operation

Consider the following network:

```text
                    R1
               Candidate BSR
                       |
                       R2
                      /  \
                     /    \
                    R3    R4
                Candidate Candidate
                    RP       RP
                     \      /
                      \    /
                        R5
```

Assume:

- R1 is configured as a Candidate BSR.
- R3 and R4 are configured as Candidate RPs.
- All routers have PIM enabled on the relevant interfaces.
- R3 and R4 advertise themselves as RPs for `239.0.0.0/8`.

The basic BSR process consists of four stages.

### Step 1: BSR Election

Routers configured as Candidate BSRs participate in an election to determine which router will distribute RP information.

The election uses:

1. **Highest BSR priority**.
2. If priorities are equal, **highest BSR IP address**.

For example:

| Candidate BSR | Priority | Address |
|---|---|---|
| R1 | 10 | 1.1.1.1 |
| R2 | 20 | 2.2.2.2 |
| R3 | 20 | 3.3.3.3 |

R3 wins because it has the highest priority and, among the candidates tied at priority 20, the highest IP address.

The elected BSR is advertised throughout the PIM domain using **Bootstrap Messages (BSMs)**.

### Step 2: Candidate RPs Advertise to the BSR

After learning the elected BSR's address, each Candidate RP sends a **Candidate-RP-Advertisement (C-RP-Adv)** message to the BSR.

These messages are **unicast** to the BSR.

For example:

```text
R3 (C-RP) --------\
                   \
                    > R1 (BSR)
                   /
R4 (C-RP) --------/
```

Each advertisement identifies information such as:

- Candidate RP address.
- Multicast group ranges the RP supports.
- RP priority.
- RP advertisement Holdtime.

The elected BSR collects these advertisements and maintains a list of available RPs.

### Step 3: BSR Distributes the RP-Set

The BSR maintains an **RP-Set**, containing information about the candidate RPs and their associated multicast group ranges.

For example:

| Group Range | Candidate RP | RP Priority |
|---|---|---|
| `239.0.0.0/8` | `3.3.3.3` | 10 |
| `239.0.0.0/8` | `4.4.4.4` | 20 |

The elected BSR distributes this information through Bootstrap Messages.

```text
               R1 (BSR)
                  |
                  | Bootstrap Messages
                  | RP-Set
                  v
                  R2
                 /  \
                /    \
               R3    R4
                 \  /
                  R5
```

Each PIM router receives the RP-Set and stores the information needed to determine the RP for multicast groups.

**Bootstrap Messages are forwarded hop by hop**, so they do not require an existing multicast distribution tree or a previously known RP.

This avoids the bootstrap problem associated with Auto-RP.

### Step 4: Each Router Selects an RP

An important characteristic of BSR is that **the BSR does not centrally select the RP for every multicast group**.

Instead, each PIM router independently applies the RP selection algorithm to its learned RP-Set.

For PIM-SM, the selection process considers:

1. The most specific matching group range.
2. The **lowest RP priority**.
3. The RP hash calculation when multiple eligible RPs have equal priority.
4. The highest RP address if hash values are equal.

For example, if two RPs advertise the same group range:

| Candidate RP | RP Priority |
|---|---|
| `3.3.3.3` | 10 |
| `4.4.4.4` | 20 |

All routers select `3.3.3.3`, because **lower RP priority values are preferred**.

If the priorities are equal, the hash calculation can distribute different multicast groups among the available RPs.

This behavior is examined in [BSR Hashing](hashing.md).

> **Important:** BSR and RP priorities work in opposite directions:
>
> - **BSR election:** Highest priority wins.
> - **RP selection:** Lowest priority wins.

## BSR Message Types

BSR uses two PIM message types.

| Message | PIM Type | Destination | Purpose |
|---|---|---|---|
| **Bootstrap Message (BSM)** | 4 | All PIM Routers | BSR election and RP-Set distribution |
| **Candidate-RP-Advertisement** | 8 | Elected BSR | Candidate RP advertisement |

Both use IP protocol 103.

Bootstrap Messages normally use these multicast destinations:

| Address Family | Destination |
|---|---|
| IPv4 | `224.0.0.13` |
| IPv6 | `FF02::D` |

BSMs normally have a TTL or Hop Limit of 1 and are regenerated by each forwarding PIM router on its downstream interfaces.

Under certain circumstances, such as informing a newly discovered neighbor, a BSM may also be sent as unicast.

Candidate-RP-Advertisements are unicast directly to the elected BSR.

## BSR Timers

RFC 5059 defines the following default BSR timers:

| Timer | Default | Purpose |
|---|---|---|
| **Bootstrap Period** | 60 seconds | Interval between periodic Bootstrap Messages |
| **Bootstrap Timeout** | 130 seconds | Time before a previously known BSR is considered expired without receiving another valid BSM |
| **Minimum Bootstrap Interval** | 10 seconds | Minimum interval between originated BSMs |

Candidate RPs also periodically refresh their advertisements.

The default C-RP advertisement interval is **60 seconds**, with a typical Holdtime of **150 seconds**.

If a candidate RP stops advertising, its information eventually expires from the BSR's candidate RP database.

## BSR Failover

BSR supports redundancy through multiple Candidate BSRs.

For example:

```text
             R1 (C-BSR)
             Priority 20
                  |
                  R3
                  |
             R2 (C-BSR)
             Priority 10
```

R1 initially wins the election.

If R1 fails and its Bootstrap Messages are no longer received, other routers eventually expire the BSR information.

R2 can then become the elected BSR and begin distributing RP information.

Similarly, if an RP becomes unavailable and its advertisement expires, routers can select another candidate RP from the updated RP-Set.

However, **BSR redundancy and RP redundancy are separate mechanisms**.

Electing a new BSR does not necessarily replace an unavailable RP immediately, and changing the selected RP may require multicast distribution trees to be rebuilt.

## IPv4 IOS XE Configuration

### Configure a Candidate BSR

For example, configure R1 as a Candidate BSR:

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
| `Loopback0` | Interface whose IPv4 address is used as the BSR address |
| `30` | Hash mask length used for RP selection |
| `10` | BSR election priority |

The hash mask length controls which multicast group address bits are included in the RP hash calculation.

A hash mask length of 30 means the first 30 bits of the multicast group address are used as the hash input.

The BSR priority ranges from 0 to 255, with higher values preferred. The Cisco IOS XE default is 0.

### Configure a Candidate RP

On R3:

```text
R3(config)# interface Loopback0
R3(config-if)# ip address 3.3.3.3 255.255.255.255
R3(config-if)# ip pim sparse-mode

R3(config)# ip pim rp-candidate Loopback0
```

This advertises R3 as a candidate RP for all applicable groups.

To limit the advertisement to particular multicast groups:

```text
R3(config)# ip access-list standard RP-GROUPS
R3(config-std-nacl)# permit 239.0.0.0 0.255.255.255

R3(config)# ip pim rp-candidate Loopback0 group-list RP-GROUPS priority 10
```

The `group-list` determines which multicast group ranges are advertised.

The `priority` parameter determines RP preference during selection, with **lower values preferred**.

The candidate RP and BSR functions can be configured on the same router if desired.

## IPv6 IOS XE Configuration

IPv6 also supports dynamic RP discovery using BSR.

However, the IOS XE command syntax differs from IPv4.

### Configure an IPv6 Candidate BSR

For example, configure R1 as a Candidate BSR:

```text
R1(config)# ipv6 unicast-routing
R1(config)# ipv6 multicast-routing

R1(config)# interface Loopback0
R1(config-if)# ipv6 address 2001:DB8:1::1/128

R1(config)# ipv6 pim bsr candidate bsr 2001:DB8:1::1 124 priority 10
```

Here:

- `2001:DB8:1::1` is the BSR address.
- `124` is the IPv6 hash mask length.
- `10` is the BSR election priority.

### Configure an IPv6 Candidate RP

On R3:

```text
R3(config)# interface Loopback0
R3(config-if)# ipv6 address 2001:DB8:3::3/128

R3(config)# ipv6 pim bsr candidate rp 2001:DB8:3::3 priority 10
```

To restrict the RP advertisement to specific multicast groups:

```text
R3(config)# ipv6 access-list RP-GROUPS-V6
R3(config-ipv6-acl)# permit ipv6 any FF15::/16

R3(config)# ipv6 pim bsr candidate rp 2001:DB8:3::3 group-list RP-GROUPS-V6 priority 10
```

As with IPv4, the Candidate RP advertises itself to the elected BSR, and each router independently determines the RP for its multicast groups.

## BSR Borders

IOS XE supports configuring **BSR border interfaces** to prevent Bootstrap Messages (BSMs) from crossing between PIM domains.

For IPv4:

```text
R1(config)# interface Ethernet0/0
R1(config-if)# ip pim bsr-border
```

For IPv6:

```text
R1(config)# interface Ethernet0/0
R1(config-if)# ipv6 pim bsr border
```

These commands affect BSMs in **both directions**:

- **Incoming:** BSMs received on the interface are ignored and are not used to update the router's BSR election or RP-Set information.
- **Outgoing:** The router does not originate or forward BSMs out of the interface.

This applies to BSMs used for both **BSR election** and **RP-Set distribution**.

### Example

Consider two PIM domains connected through R2:

```text
      PIM Domain A                 PIM Domain B

      R1 (BSR-A)                    R3 (BSR-B)
          |                             |
          R2 -------------------------- R4
             E0/0
           BSR Border
```

R2 has `ip pim bsr-border` configured on Ethernet0/0, facing Domain B.

- BSMs from BSR-A are not forwarded out of Ethernet0/0 into Domain B.
- BSMs received from Domain B on Ethernet0/0 are ignored by R2.
- R2 can still receive and process BSMs on its other PIM interfaces.

Therefore, each domain can elect its **own BSR** and maintain a separate **RP-Set**.

### What Is Not Blocked?

A BSR border specifically controls Bootstrap Messages.

| Traffic | Blocked by BSR Border? |
|---|---|
| PIM Bootstrap Messages (Type 4) | **Yes, incoming and outgoing** |
| PIM Hello messages | No |
| PIM Join/Prune messages | No |
| PIM Register/Register-Stop | No |
| Candidate-RP-Advertisements (Type 8) | Not directly |
| Multicast data traffic | No |

**Candidate-RP-Advertisements are unicast**, so they are not directly filtered by the BSR border command. However, the boundary prevents routers from learning BSRs across it through BSMs, indirectly affecting where Candidate RPs send their advertisements.

> **Important:** A BSR border is not a general multicast boundary. It separates BSR control-plane domains, but does not prevent multicast traffic or other PIM signaling from crossing the interface. Use multicast boundary features when data-plane multicast isolation is also required.

For IPv6, `ipv6 pim bsr border` blocks BSMs for **all multicast scopes**, rather than a specific administrative scope.

## IOS XE Verification

### IPv4

| Command | Purpose |
|---|---|
| `show ip pim bsr-router` | Display elected BSR information |
| `show ip pim bsr` | Display BSR and candidate RP information, where supported |
| `show ip pim rp mapping` | Display learned and configured RP mappings |
| `show ip pim rp-hash 239.1.1.1` | Display the RP selected for a particular group |
| `show ip rpf <BSR-address>` | Verify the RPF path toward the elected BSR |

For example:

```text
R1# show ip pim bsr-router
R1# show ip pim rp mapping
R1# show ip pim rp-hash 239.1.1.1
```

### IPv6

| Command | Purpose |
|---|---|
| `show ipv6 pim bsr election` | Display elected BSR and election information |
| `show ipv6 pim bsr rp-cache` | Display candidate RP information received by the BSR |
| `show ipv6 pim bsr candidate-rp` | Display local Candidate RP state |
| `show ipv6 pim group-map` | Display group-to-RP mappings |
| `show ipv6 pim group-map FF15::101` | Verify the RP selected for a particular IPv6 group |

For example:

```text
R1# show ipv6 pim bsr election
R1# show ipv6 pim bsr rp-cache
R1# show ipv6 pim group-map
```

### Common Troubleshooting Problems

| Problem | Possible Cause |
|---|---|
| No BSR elected | No reachable Candidate BSR or BSM forwarding problem |
| BSR information differs between routers | BSR border, RPF failure, or BSM filtering |
| Candidate RP missing from RP-Set | No C-RP advertisements reaching the BSR |
| Group has no RP mapping | No C-RP advertised the appropriate group range |
| Different routers select different RPs | Inconsistent RP-Sets, priorities, hash mask lengths, or mapping precedence |
| RP mapping exists but multicast fails | RP unreachable or multicast forwarding/RPF problem |

## Key Points

- **BSR dynamically distributes RP information** throughout the PIM domain.
- A **Candidate BSR (C-BSR)** participates in the election for the active BSR.
- The **highest BSR priority wins**, with the highest IP address as the tie-breaker.
- **Candidate RPs (C-RPs)** advertise their RP addresses, supported group ranges, and priorities to the elected BSR.
- C-RP advertisements are **unicast** to the BSR using **PIM Type 8**.
- The BSR distributes the **RP-Set** using **Bootstrap Messages (PIM Type 4)**.
- BSMs are normally multicast to `224.0.0.13` (IPv4) or `FF02::D` (IPv6) and forwarded hop by hop.
- Each router independently selects its RP from the advertised RP-Set.
- **Lower RP priorities are preferred**, opposite to the BSR election rule.
- The default Bootstrap Period is **60 seconds**, and the Bootstrap Timeout is **130 seconds**.
- BSR supports IPv4 and IPv6.
- `ip pim bsr-border` and `ipv6 pim bsr border` control BSR propagation across interfaces.
- BSR eliminates the need to manually configure RP mappings on every router, but does not guarantee hitless RP failover.

## References

- [RFC 5059: Bootstrap Router (BSR) Mechanism for PIM](https://www.rfc-editor.org/rfc/rfc5059)
- [Cisco IOS XE: Configuring Basic IP Multicast](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/ip-multicast/ip-multicast/m_imc_basic_cfg-0.html)
- [Cisco IOS XE: IPv6 Multicast Bootstrap Router](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-multicast/b-ip-multicast/m_ip6-mcast-bsr.html)