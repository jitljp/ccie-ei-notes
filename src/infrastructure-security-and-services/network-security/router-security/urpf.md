# 4.2.b (iii) Unicast Reverse Path Forwarding

**Unicast Reverse Path Forwarding (uRPF)** is an IP source-validation feature that helps prevent packets with **spoofed source IP addresses** from entering a network. It checks the source IP address of a packet received on an interface against the router's **Forwarding Information Base (FIB)**.

uRPF has two primary modes:

- **Strict mode:** The interface on which the packet arrived must be a valid reverse-path interface for the source IP address.
- **Loose mode:** The source IP address must be reachable through a valid route, regardless of which interface received the packet.

uRPF is configured on an **ingress interface**, applies independently of ordinary interface ACLs, and is **disabled by default**.

---

## Why uRPF Is Needed

IPv4 does not inherently authenticate the source IP address in a packet. A host can create packets with a forged source address to:

- Conceal the actual source of unwanted traffic.
- Impersonate a host in another network.
- Generate denial-of-service traffic with misleading source addresses.
- Send traffic with source addresses that should never appear on a particular link.

For example:

```text
Host A                              R1
192.0.2.10 ---- Gi1 (LAN) ----- [Router] ---- Gi2 (WAN)

Legitimate packet received on Gi1:
  Source = 192.0.2.10

Spoofed packet received on Gi1:
  Source = 198.51.100.10
```

If R1's best route back to `198.51.100.10` points out Gi2 rather than Gi1, **strict uRPF on Gi1** rejects the spoofed packet. However, loose uRPF can accept the same packet if R1 has a valid nondefault route to `198.51.100.10` through Gi2.

**uRPF validates the source address, not the destination address.** The router performs an ordinary destination lookup separately when forwarding an accepted packet.

---

## How uRPF Works

When an IPv4 packet arrives on a uRPF-enabled interface, the router:

1. Reads the packet's **source IPv4 address**.
2. Performs a **reverse lookup** of that address in the FIB.
3. Checks the result against the configured uRPF mode.
4. Drops the packet if the check fails, unless a configured uRPF ACL permits an exception.
5. Continues normal processing and forwarding if the packet is accepted.

The router uses the **source address as the lookup key**, even though normal IP forwarding uses the destination address.

```text
                     Incoming IPv4 packet
                              |
                              v
                     Read SOURCE address
                              |
                              v
                       FIB lookup
                              |
                 +------------+------------+
                 |                         |
             Strict (rx)                Loose (any)
                 |                         |
       Reverse-path interface     Valid route exists to
       matches ingress?          source (any interface)?
                 |                         |
          Pass or fail              Pass or fail
```

The FIB is built by **Cisco Express Forwarding (CEF)** from the installed routing and forwarding information. A route appearing somewhere in a routing protocol database is not enough: the path must be usable for the source lookup in the relevant FIB.

**uRPF requires CEF** to function. It does not examine application sessions, TCP state, or whether a packet's claimed source belongs to the person who sent it.

### Ingress-Only Operation

uRPF checks packets as they **enter** an interface. It is not configured with an `in` or `out` keyword:

```text
R1(config)# interface GigabitEthernet1
R1(config-if)# ip verify unicast source reachable-via rx
```

This enables strict uRPF for packets **received on Gi1**. It does not enable an outbound uRPF check on packets transmitted out Gi1.

---

## Strict uRPF (`rx`)

In **strict mode**, a packet passes if the FIB identifies the **receiving interface as a valid interface for reaching the source address**.

```text
R1(config-if)# ip verify unicast source reachable-via rx
```

Consider this routing information on R1:

| Source prefix being checked | Reverse-path outgoing interface |
|---|---|
| `192.0.2.0/24` | Gi1 |
| `198.51.100.0/24` | Gi2 |
| `203.0.113.0/24` | Gi3 |

Assume a packet arrives on Gi1:

| Packet source | Reverse-path interface | Strict uRPF result |
|---|---|---|
| `192.0.2.10` | Gi1 | **Pass** |
| `198.51.100.10` | Gi2 | **Fail** |
| `203.0.113.10` | Gi3 | **Fail** |
| `10.10.10.10`, with no matching usable route | None | **Fail** |

Strict uRPF requires the return path to be **consistent with the ingress interface**. It does not require the packet to be a reply to an earlier request.

### Asymmetric Routing

Strict uRPF can reject legitimate packets when forward and return paths are different.

```text
                  Source network
                   203.0.113.0/24
                    /         \
                 R2             R3
                  \             /
                 Gi1           Gi2
                    \         /
                         R1
```

Suppose R1 receives a packet from `203.0.113.10` through **Gi1**, but R1's best route back to `203.0.113.0/24` points through **Gi2**.

- The packet may have arrived through a perfectly valid asymmetric path.
- **Strict uRPF on Gi1 fails**, because Gi1 is not the FIB's reverse-path interface.
- **Loose uRPF on Gi1 passes**, provided the source has a valid route.

A successful destination route to the packet's *destination* does not change this strict uRPF result.

### Equal-Cost Multipath (ECMP)

Strict uRPF supports **equal-cost reverse paths**. The receiving interface need only be **one of** the valid FIB return-path interfaces.

For example, if R1 has the following installed equal-cost paths:

```text
203.0.113.0/24 -> Gi1
203.0.113.0/24 -> Gi2
```

A packet sourced from `203.0.113.10` may pass strict uRPF when received on **either Gi1 or Gi2**.

**An alternate path that is not installed as an eligible FIB return path does not automatically satisfy strict uRPF.** For example, a less-preferred backup route in the routing protocol database does not count merely because it exists.

---

## Loose uRPF (`any`)

In **loose mode**, the router requires a valid route to the source address but does **not** require the reverse-path interface to match the packet's ingress interface.

```text
R1(config-if)# ip verify unicast source reachable-via any
```

Using the same R1 FIB:

| Packet source received on Gi1 | Route to source | Loose uRPF result |
|---|---|---|
| `192.0.2.10` | Gi1 | **Pass** |
| `198.51.100.10` | Gi2 | **Pass** |
| `203.0.113.10` | Gi3 | **Pass** |
| `10.10.10.10` | No usable nondefault route | **Fail** |

Loose uRPF is useful on interfaces where **asymmetric routing** is expected, such as some multihomed or provider-facing connections.

Its protection is weaker than strict uRPF: a spoofed source can pass if the router already has a valid route to that source prefix, even if the packet arrived from the wrong direction.

### Strict Versus Loose Comparison

| Property | Strict (`rx`) | Loose (`any`) |
|---|---|---|
| Checks source in FIB | Yes | Yes |
| Receiving interface must match reverse path | **Yes** | **No** |
| Supports asymmetric routing | Only when ingress is an eligible return path | **Yes** |
| Accepts ECMP return paths | **Yes**, for eligible interfaces | Yes |
| Rejects sources without a usable route | Yes | Yes |
| Main use case | Access/customer edges with predictable return paths | Networks with asymmetric return paths |

---

## Default Route Handling

By default, uRPF does **not** treat a match through only the default route (`0.0.0.0/0`) as sufficient for source verification.

To allow the default route to satisfy the source check, add **`allow-default`**:

```text
R1(config-if)# ip verify unicast source reachable-via any allow-default
```

The keyword is also available with strict mode:

```text
R1(config-if)# ip verify unicast source reachable-via rx allow-default
```

Consider this FIB:

```text
192.0.2.0/24      -> Gi1
198.51.100.0/24   -> Gi2
0.0.0.0/0         -> Gi2
```

A packet enters Gi1 with source `8.8.8.8`, which matches **only the default route**.

| Mode on Gi1 | Result | Reason |
|---|---|---|
| `rx` | **Fail** | The default route is ignored by uRPF. |
| `any` | **Fail** | The default route is ignored by uRPF. |
| `rx allow-default` | **Fail** | Default route points out Gi2, not Gi1. |
| `any allow-default` | **Pass** | Default route is accepted as a valid path to the source. |

If this same packet entered Gi2, `rx allow-default` could pass because the default route's outgoing interface matches the ingress interface.

**Important:** Configuring `any allow-default` on a router with an installed default route greatly weakens loose uRPF. Many otherwise unknown source addresses can pass using that default route.

### Longest-Prefix Match Still Matters

The source lookup uses the **longest matching FIB prefix**. An installed more-specific route takes precedence over the default route.

For example:

```text
0.0.0.0/0          -> Gi1
203.0.113.0/24     -> Gi2
```

A packet with source `203.0.113.10` arriving on Gi1 **fails strict uRPF**, even with `allow-default`. The `/24` route points out Gi2, so the router does not substitute the default route to make the check pass.

---

## Routes Pointing to Null0

A route that resolves to **Null0** represents a discard path, not a usable reverse path.

For example:

```text
R1(config)# ip route 203.0.113.0 255.255.255.0 Null0
```

With this route installed, a packet sourced from `203.0.113.10` will fail ordinary loose uRPF, even though the prefix appears in the FIB.

This is useful for **source-based blackholing**: install a discard route for a hostile source prefix so that uRPF-enabled interfaces reject packets claiming addresses in that prefix.

A route's presence alone is therefore not always sufficient. **Loose uRPF requires a usable forwarding path**, not merely any FIB entry.

---

## uRPF ACL Exceptions

A **uRPF ACL** can permit specific packets that would otherwise fail the reverse-path check. This is useful when an otherwise valid source arrives through a known asymmetric path.

The ACL is attached **as an argument to the uRPF interface command**, not with `ip access-group`.

### Example: Permit One Asymmetric Source

Suppose R1 normally reaches `203.0.113.0/24` through Gi2, but `203.0.113.10` legitimately sends some traffic through Gi1. Keep strict uRPF on Gi1 while allowing an exception for that host:

The uRPF command references a **numbered ACL**, such as ACL 101:

```text
R1(config)# ip access-list extended 101
R1(config-ext-nacl)# permit ip host 203.0.113.10 any
R1(config-ext-nacl)# deny ip any any
!
R1(config)# interface GigabitEthernet1
R1(config-if)# ip verify unicast source reachable-via rx 101
```

For a packet that fails strict uRPF:

- If ACL 101 **permits** it, uRPF allows the exception.
- If ACL 101 **denies** it (explicitly or implicitly), the packet is dropped.

For a packet that **passes** the FIB check, the uRPF ACL is not used as a general-purpose deny filter.

| FIB reverse-path result | uRPF ACL result | Final uRPF decision |
|---|---|---|
| Pass | Not consulted | **Pass** |
| Fail | Permit | **Pass by exception** |
| Fail | Deny | **Drop** |
| Fail | No match (implicit deny) | **Drop** |

**This is not how a regular interface ACL works.** A uRPF ACL is an *exception policy for failed source checks*, not an ACL that independently filters every arriving packet.

### Regular Interface ACL Versus uRPF ACL

```text
R1(config)# interface GigabitEthernet1
R1(config-if)# ip access-group INBOUND_FILTER in
R1(config-if)# ip verify unicast source reachable-via rx 101
```

- `INBOUND_FILTER` is an **ordinary inbound ACL**, which can permit or deny traffic according to its ACEs.
- `101` is the **uRPF exception ACL**, consulted when the uRPF reverse-path check fails.
- A packet must still satisfy all other applicable forwarding and security checks, including the ordinary inbound ACL.

The ordinary packet-processing sequence is **input ACL, uRPF check, destination forwarding lookup, output ACL**, followed by forwarding if the checks pass.

### Logging uRPF Failures

The uRPF ACL can include `log` or `log-input` to help identify rejected sources:

```text
R1(config)# ip access-list extended 101
R1(config-ext-nacl)# 10 permit ip host 203.0.113.10 any log-input
R1(config-ext-nacl)# 20 deny ip any any log-input
```

Logging is useful during troubleshooting, but high-rate logging can consume router CPU resources.

### uRPF Drop Counters

Cisco distinguishes two events:

- **uRPF drops:** Packets failing the reverse-path check and actually discarded.
- **uRPF suppressed drops:** Packets failing the reverse-path check but **forwarded because the uRPF ACL permitted them**.

The term *suppressed drop* means **the drop was suppressed**. It does not mean the packet was discarded.

---

## Configuring uRPF

### Example 1: Strict uRPF on a LAN Interface

R1 connects to a single local subnet on Gi1. It has a connected route to `192.0.2.0/24` through Gi1.

```text
R1(config)# ip cef
!
R1(config)# interface GigabitEthernet1
R1(config-if)# ip address 192.0.2.1 255.255.255.0
R1(config-if)# ip verify unicast source reachable-via rx
```

A packet claiming a source in the directly connected subnet can pass if the reverse path resolves to Gi1. A packet claiming a source whose reverse path points elsewhere fails strict uRPF.

### Example 2: Loose uRPF on an Asymmetric Link

R1 receives legitimate traffic through Gi2 even though return routes to some sources are through Gi3.

```text
R1(config)# interface GigabitEthernet2
R1(config-if)# ip verify unicast source reachable-via any
```

R1 accepts packet sources with valid routes through either Gi2, Gi3, or another eligible interface. It drops packet sources that have no usable route, subject to any explicit uRPF exception policy.

### Example 3: Strict uRPF with a Targeted Exception

```text
R1(config)# ip access-list extended 101
R1(config-ext-nacl)# permit ip host 203.0.113.10 any
R1(config-ext-nacl)# deny ip any any
!
R1(config)# interface GigabitEthernet1
R1(config-if)# ip verify unicast source reachable-via rx 101
```

This preserves strict checking for ordinary traffic while permitting the specified host to use an otherwise unexpected ingress path.

### Example 4: Allow the Default Route

```text
R1(config)# interface GigabitEthernet2
R1(config-if)# ip verify unicast source reachable-via any allow-default
```

This accepts sources whose only matching route is the router's default route, assuming the default route supplies an eligible forwarding path. Consider carefully whether the resulting protection is sufficient for the interface.

### Disable uRPF

```text
R1(config)# interface GigabitEthernet1
R1(config-if)# no ip verify unicast source reachable-via
```

---

## Additional Options

### `allow-self-ping`

The **`allow-self-ping`** option permits the router to ping its own interface addresses in situations where the uRPF check would otherwise reject the returning packet.

```text
R1(config-if)# ip verify unicast source reachable-via rx allow-self-ping
```

It is not required for normal transit traffic. Cisco cautions that enabling it can weaken protection against certain spoofing attacks, so use it only where needed.

### Source IPv4 and MAC Binding (`l2-src`)

On supported platforms, strict uRPF also provides an optional `l2-src` keyword for **source IPv4-to-MAC binding validation**:

```text
R1(config-if)# ip verify unicast source reachable-via rx l2-src
```

This is distinct from ordinary strict uRPF, which checks the **IP reverse path** but does **not** inherently compare the packet's source IP and Ethernet source MAC. The `l2-src` option is not available with `any` mode.

### IPv6 uRPF

The same basic strict and loose logic can apply to IPv6:

```text
R1(config)# interface GigabitEthernet1
R1(config-if)# ipv6 verify unicast source reachable-via rx
```

```text
R1(config-if)# ipv6 verify unicast source reachable-via any
```

Use `allow-default` where supported if the IPv6 default route should count during source verification. IPv4 and IPv6 uRPF are separately configured.

---

## uRPF with VRFs

If the ingress interface belongs to a **VRF**, source validation must use the route information appropriate to that forwarding context, not assume that the global routing table contains the same prefixes.

For example:

```text
R1(config)# vrf definition CUSTOMER
R1(config-vrf)# address-family ipv4
R1(config-vrf-af)# exit-address-family
!
R1(config)# interface GigabitEthernet1
R1(config-if)# vrf forwarding CUSTOMER
R1(config-if)# ip address 192.0.2.1 255.255.255.0
R1(config-if)# ip verify unicast source reachable-via rx
```

To investigate a failed source check on this interface, examine the **CUSTOMER VRF's** routing and CEF entries:

```text
R1# show ip route vrf CUSTOMER 192.0.2.10
R1# show ip cef vrf CUSTOMER 192.0.2.10 detail
```

A matching route existing only in the global table does not necessarily make the source reachable in the ingress VRF.

---

## Verification

### Confirm the Interface Configuration

```text
R1# show running-config interface GigabitEthernet1
```

Look for the uRPF configuration:

```text
ip verify unicast source reachable-via rx
```

### Display uRPF State and Counters

```text
R1# show ip interface GigabitEthernet1
```

The interface output can report the uRPF verification mode, its associated ACL, and counters such as **unicast RPF drops** and **unicast RPF suppressed drops**.

### Check the Source Route

```text
R1# show ip route 203.0.113.10
R1# show ip cef 203.0.113.10 detail
```

Determine:

1. Which **longest matching prefix** covers the source address?
2. Does it resolve to a **usable forwarding interface**?
3. In strict mode, does that interface match the **packet's ingress interface**?
4. Is the route a **default route**, and if so, is `allow-default` configured?
5. Is it a **Null0/discard** route?
6. For an ECMP destination, is the ingress interface among the installed equal-cost paths?

### View Router-Wide Drop Statistics

```text
R1# show ip traffic
```

Look for the **unicast RPF** entry in the IP drop statistics.

### Check uRPF ACL Matches

```text
R1# show ip access-lists 101
```

When a uRPF ACL is configured, match counters can help identify packets that failed the source check and were either permitted by an exception or denied.

---

## Troubleshooting Scenarios

### Scenario 1: Legitimate Traffic Fails Strict uRPF

**Symptom:** Packets from a legitimate source disappear at the ingress interface and the uRPF drop counter increases.

**Likely cause:** The FIB's best return path to that source uses **another interface**.

**Check:**

```text
R1# show ip cef 203.0.113.10 detail
R1# show ip interface GigabitEthernet1
```

**Resolution:** Fix an incorrect route, use loose mode if asymmetric paths are expected, or use a narrowly scoped uRPF ACL exception.

### Scenario 2: Loose uRPF Drops Traffic Despite a Default Route

**Symptom:** A packet's source address is reachable via the default route, but loose uRPF rejects it.

**Cause:** The default route is **not considered** unless `allow-default` is configured.

**Resolution:** Add `allow-default` if that behavior is intended, or provide an appropriate more-specific route.

### Scenario 3: Strict uRPF Fails Despite an Alternate Route

**Symptom:** A routing protocol knows an alternate path through the ingress interface, but strict uRPF still drops packets.

**Cause:** The alternate path is **not an installed eligible FIB reverse path**. Merely having a backup route in the routing protocol database is insufficient.

**Resolution:** Check which paths actually entered the FIB. Equal-cost installed paths can satisfy strict uRPF.

### Scenario 4: uRPF ACL Deny Does Not Block a Source

**Symptom:** An ACL referenced by `ip verify unicast source reachable-via rx 101` contains a `deny` for a source, but that source's packets still pass.

**Cause:** The source **passes** the normal reverse-path check, so the **uRPF exception ACL is not evaluated**.

**Resolution:** Use an ordinary inbound `ip access-group` ACL to deny the source unconditionally.

### Scenario 5: Packet Matches a Null0 Route

**Symptom:** Loose uRPF drops packets even though `show ip route` contains a route to their source prefix.

**Cause:** The route points to **Null0**, so the source has no usable reverse path.

**Resolution:** Determine whether the Null0 route was intentional source blackholing or an incorrect routing entry.

### Scenario 6: DHCP or Other Initial Packets Are Dropped

Packets using a temporary or unspecified source address such as `0.0.0.0` may fail a uRPF source check before normal address assignment. Account for the permitted bootstrapping traffic when placing source checks on interfaces that receive DHCP clients.

---

## uRPF Versus ACLs and IP Source Guard

| Feature | Main validation | Typical placement |
|---|---|---|
| **uRPF** | Source IP address against FIB reachability; optionally ingress-interface match | Routed ingress interface |
| **IPv4 interface ACL** | Explicit conditions such as source, destination, protocol, and ports | Routed ingress or egress interface |
| **IP Source Guard** | Host source address against authorized bindings, often learned from DHCP Snooping | Layer 2 access port |

An IPv4 ACL can enforce a defined list of authorized source prefixes on an access-facing router interface. uRPF instead relies on **current forwarding information** and can automatically adapt as routes change.

Neither strict nor loose uRPF independently proves that the sender owns the claimed source address. For example, strict mode may still permit a spoofed source within a prefix whose reverse path legitimately uses the ingress interface. Use ACLs and other source-validation features where finer-grained restrictions are required.

---

## Key Points

- **uRPF checks the packet's source IP address against the FIB** on the ingress interface.
- **Strict mode (`rx`)** requires the receiving interface to be an eligible reverse-path interface.
- **Loose mode (`any`)** requires a usable route to the source, but the outgoing interface can differ from the receiving interface.
- **Installed ECMP reverse paths** can all be valid in strict mode; noninstalled backup routes do not automatically count.
- **Default routes are ignored by default** for uRPF checks. Use `allow-default` to allow them.
- **Null0 routes do not satisfy ordinary source reachability checks**.
- A **uRPF ACL is consulted only after a failed reverse-path check**. Its `permit` entries provide exceptions; its `deny` entries do not override a successful FIB check.
- **uRPF suppressed drops** are packets that failed verification but were forwarded by an ACL exception.
- **CEF is required**, and the routing/forwarding context matters when VRFs are involved.
- uRPF complements, rather than replaces, ordinary interface ACLs.

---

## References

- [Cisco: Unicast Reverse Path Forwarding Strict Mode](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/ip-routing/b-ip-routing/m_cfg-unicast-rpf-0.html)
- [Cisco: Unicast Reverse Path Forwarding ACL Support](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/ip-routing/b-ip-routing/m_urpf-acl-sup.html)
- [Cisco: `ip verify unicast source reachable-via` Command Reference](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/interface/command/ir-cr-book/ir-i1.html)
- [Cisco: Unicast Reverse Path Forwarding for IPv6](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-addressing/b-ip-addressing/m_ip6-urpf-xe.html)
