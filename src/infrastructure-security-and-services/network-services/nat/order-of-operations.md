# NAT Packet Processing and Order of Operations

**Network Address Translation (NAT)** modifies IP addresses and, with **Port Address Translation (PAT)**, transport-layer port numbers as packets cross a NAT device.

On Cisco IOS XE, correctly predicting NAT behavior requires understanding both **which packet fields are translated** and **when translation occurs relative to routing and other packet-processing features**.

The fundamental rule for traditional inside/outside NAT is:

- **Inside to outside:** Routing is performed **before NAT**.
- **Outside to inside:** NAT is performed **before routing**.

This ordering has important consequences for static NAT, PAT, outside source NAT, policy-based NAT, ACLs, and overlapping networks.

> **Scope:** The examples below describe the *logical* order of operations for conventional IOS/IOS XE NAT with interfaces configured as `ip nat inside` and `ip nat outside`. Cisco's classic order-of-operations reference was developed for IOS, and the exact feature pipeline can vary by IOS XE release, platform, hardware forwarding implementation, and enabled services. Use the two routing/NAT rules as the primary model, rather than treating the full legacy feature list as a universal IOS XE packet-processing diagram.

For a review of **inside local**, **inside global**, **outside local**, and **outside global**, see [NAT Address Terminology and Translation Types](terminology.md).

---

## NAT Directions and Interface Roles

Traditional NAT identifies two sides of a translation boundary:

| Interface command | Meaning |
|---|---|
| `ip nat inside` | Identifies an interface on the inside side of the NAT boundary |
| `ip nat outside` | Identifies an interface on the outside side of the NAT boundary |

A typical topology is:

```text
        Inside Network                   Outside Network

      PC1                                   Web Server
   10.1.1.10                              203.0.113.10
       |                                       |
       |                                       |
       +----------- R1 ------------------------+
                 G1  |  G2
             10.1.1.1    192.0.2.2
           NAT inside    NAT outside
```

Packets traveling from PC1 toward the web server cross the NAT boundary **inside to outside**. Reply packets cross **outside to inside**.

**Important:** `inside source` and `outside source` describe *which host's address is being translated*, not necessarily the packet's direction or the header field modified for every packet.

For example, `ip nat outside source static` translates the **source** address of a packet entering from outside, but the **destination** address of a packet traveling from inside to outside.

---

## Inside-to-Outside Order of Operations

For a packet received on an **inside** interface and forwarded toward an **outside** interface, the simplified processing order is:

```text
        Packet enters NAT inside interface
                         |
                         v
           Input ACL and other checks
                         |
                         v
             Policy-based routing
               (if configured)
                         |
                         v
               Routing lookup
          (using pre-NAT addresses)
                         |
                         v
              NAT translation
                         |
                         v
             Output processing
            (including output ACL)
                         |
                         v
         Packet exits NAT outside interface
```

The critical point is that **R1 must be able to route the packet toward the outside before the NAT translation is performed**.

In an ordinary inside source NAT deployment, the router looks up the destination and determines the outgoing interface. NAT then translates the inside local source address to its inside global representation.

### Example: Dynamic PAT

Suppose PC1 opens an HTTPS connection to the web server:

```text
Source:      10.1.1.10:49152
Destination: 203.0.113.10:443
Protocol:    TCP
```

R1 is configured to overload its outside interface address:

```text
R1(config)# interface GigabitEthernet1
R1(config-if)# ip address 10.1.1.1 255.255.255.0
R1(config-if)# ip nat inside

R1(config)# interface GigabitEthernet2
R1(config-if)# ip address 192.0.2.2 255.255.255.252
R1(config-if)# ip nat outside

R1(config)# access-list 10 permit 10.1.1.0 0.0.0.255
R1(config)# ip nat inside source list 10 interface GigabitEthernet2 overload
R1(config)# ip route 0.0.0.0 0.0.0.0 192.0.2.1
```

The packet is processed as follows:

1. R1 receives the packet on `GigabitEthernet1`, its NAT inside interface.
2. Input features, including any input ACL, process the original packet.
3. R1 looks up `203.0.113.10` in the routing table. The default route points toward `192.0.2.1` through `GigabitEthernet2`.
4. NAT matches the source address `10.1.1.10` against ACL 10 and creates or reuses a PAT translation.
5. R1 changes the source address to `192.0.2.2`, and may change the source port to maintain a unique translation. Assume the allocated port is `40001`.
6. R1 forwards the translated packet toward the next hop.

| Field | Before NAT | After NAT |
|---|---|---|
| Source IP | `10.1.1.10` | `192.0.2.2` |
| Source TCP port | `49152` | `40001` |
| Destination IP | `203.0.113.10` | `203.0.113.10` |
| Destination TCP port | `443` | `443` |

```text
Inside                                     Outside

10.1.1.10:49152  ------------------------>  203.0.113.10:443
                         R1
                  Route first
                  Translate second
                         |
                         v
192.0.2.2:40001  ------------------------>  203.0.113.10:443
```

The outside server sees the **inside global** address and translated TCP port, not PC1's original private address and port.

> The PAT source port shown here is illustrative. IOS XE might preserve the original port if it is available or allocate a different port; do not assume a specific translated port number.


### Why Routing Comes First

For traditional **inside-to-outside NAT**, Cisco performs the routing lookup before translating the packet.

This allows the router to determine the **outgoing interface** before applying NAT.

The outgoing interface is important because:

- **NAT direction:** The router uses the inside/outside interface roles to determine the applicable NAT processing.
- **Policy-based NAT:** NAT route maps can match the outgoing interface using `match interface`, allowing different translations depending on the selected path.
- **Interface PAT:** When using `ip nat inside source ... interface ... overload`, the translated source address comes from the specified interface.

For example, consider a router with two ISP connections:

```text
                  R1
                 /  \
                /    \
             ISP1    ISP2
              G2      G3
```

Policy-based NAT can use different address pools depending on whether routing selects G2 or G3 as the outgoing interface.

```text
Packet arrives on inside interface
                 |
                 v
       Routing lookup
                 |
                 v
       Outgoing interface
           G2 or G3
                 |
                 v
       NAT policy selection
                 |
                 v
       Source NAT/PAT
                 |
                 v
       Forward packet
```

**Important:** Routing before NAT is not inherently necessary for every translation. With ordinary inside source NAT, the destination remains unchanged, so translating the source first would not eliminate the need for a valid route.

However, Cisco's traditional NAT processing performs routing first for inside-to-outside traffic. If the destination is unreachable, the packet cannot be forwarded successfully, regardless of whether a NAT mapping exists.

The opposite order applies to traditional **outside-to-inside** traffic. In that direction, NAT translates the destination address before routing so the router can select a route toward the inside-local destination.

---

## Outside-to-Inside Order of Operations

For a packet received on an **outside** interface and forwarded toward an **inside** interface, the simplified order is reversed:

```text
        Packet enters NAT outside interface
                         |
                         v
           Input ACL and other checks
                         |
                         v
              NAT translation
                         |
                         v
             Policy-based routing
               (if configured)
                         |
                         v
               Routing lookup
         (using post-NAT destination)
                         |
                         v
             Output processing
            (including output ACL)
                         |
                         v
         Packet exits NAT inside interface
```

The key point is that the **destination can be translated before the routing lookup**.

This lets the router receive a packet addressed to an inside global address, translate its destination to the inside local address, and then route it toward the actual inside host.

### Example: Return Traffic for PAT

Continuing the previous HTTPS example, the server replies to the translated address and port:

```text
Source:      203.0.113.10:443
Destination: 192.0.2.2:40001
Protocol:    TCP
```

R1 processes the reply as follows:

1. The packet enters `GigabitEthernet2`, the NAT outside interface.
2. An inbound ACL on the outside interface, if configured, evaluates the packet as it arrives, with destination `192.0.2.2:40001`.
3. NAT looks up the existing translation and changes the destination to `10.1.1.10:49152`.
4. R1 performs the routing lookup for **`10.1.1.10`**, rather than `192.0.2.2`.
5. The connected route for `10.1.1.0/24` points out `GigabitEthernet1`.
6. R1 forwards the packet to PC1.

| Field | Before NAT | After NAT |
|---|---|---|
| Source IP | `203.0.113.10` | `203.0.113.10` |
| Source TCP port | `443` | `443` |
| Destination IP | `192.0.2.2` | `10.1.1.10` |
| Destination TCP port | `40001` | `49152` |

```text
Outside                                     Inside

203.0.113.10:443  ----------------------->  192.0.2.2:40001
                         R1
                  Translate first
                  Route second
                         |
                         v
203.0.113.10:443  ----------------------->  10.1.1.10:49152
```

### Why the Post-NAT Destination Matters

With a static inside source mapping:

```text
R1(config)# ip nat inside source static 10.1.1.10 198.51.100.10
```

An external host can send a packet to `198.51.100.10`, provided the packet is actually delivered to R1 and other policies permit it.

When that packet enters R1's outside interface:

1. NAT translates destination `198.51.100.10` to `10.1.1.10`.
2. Routing is performed toward `10.1.1.10`.

The route to the **inside local** address determines where the translated packet is forwarded.

**Two distinct routing requirements must be satisfied:**

- The upstream network must deliver traffic for the **inside global** address to the NAT router. Depending on addressing and topology, this might involve routing or ARP behavior.
- The NAT router must be able to forward the translated packet toward the **inside local** address.

The existence of a static translation alone does not guarantee both forms of reachability.

---

## NAT Direction Is Not the Same as Source or Destination Translation

The NAT order-of-operations rule describes the **direction across the inside/outside boundary**. It does not mean that only source fields change from inside to outside or that only destination fields change from outside to inside.

Consider the two common static command families:

| Configuration | Inside to outside | Outside to inside |
|---|---|---|
| `ip nat inside source static` | Translate **source**, inside local to inside global | Translate **destination**, inside global to inside local |
| `ip nat outside source static` | Translate **destination**, outside local to outside global | Translate **source**, outside global to outside local |

These translations still operate within the same directional processing model.

### Outside Source NAT Example

An outside host uses its real address `203.0.113.10`, but inside devices must refer to it as `172.16.50.10`:

```text
R1(config)# ip nat outside source static 203.0.113.10 172.16.50.10
```

Here:

- **Outside global:** `203.0.113.10`
- **Outside local:** `172.16.50.10`

#### Outside to Inside

A packet arrives from the outside host:

```text
Before NAT: 203.0.113.10  --->  10.1.1.10
After NAT:  172.16.50.10  --->  10.1.1.10
```

NAT modifies the **source** address before R1 routes the packet toward the inside host.

#### Inside to Outside

The inside host responds to the outside-local address:

```text
Before NAT: 10.1.1.10  --->  172.16.50.10
After NAT:  10.1.1.10  --->  203.0.113.10
```

R1 must make a routing decision for **`172.16.50.10` before NAT changes the destination to `203.0.113.10`**.

This is why Cisco documents a routing requirement for **outside-local addresses** when using outside source NAT. A suitable route must resolve toward the correct outside forwarding path. Depending on the IOS XE feature and configuration, the `add-route` option can install the necessary route, or it can be configured manually.

**CCIE takeaway:** For `ip nat outside source`, an inside-to-outside packet may require a route to an **outside local** address even though its destination will be changed to the **outside global** address before transmission.

---

## Static NAT, Dynamic NAT, and Translation State

Routing and NAT are separate operations. Even when the packet is routable, translation depends on the availability of an appropriate NAT mapping.

### Static Mappings

A static NAT mapping is configured in advance:

```text
R1(config)# ip nat inside source static 10.1.1.10 198.51.100.10
```

It allows the router to recognize both translation directions, subject to routing, filtering, and other forwarding requirements.

### Dynamic NAT and PAT

A dynamic mapping is created when eligible traffic causes the router to allocate translation resources:

```text
R1(config)# access-list 10 permit 10.1.1.0 0.0.0.255
R1(config)# ip nat inside source list 10 interface GigabitEthernet2 overload
```

For a typical inside-initiated PAT session:

1. The inside packet is routed toward the outside.
2. NAT checks for a matching existing translation.
3. If necessary, NAT allocates a new translation and port mapping.
4. Return packets use the existing translation to reach the correct inside host.

An unsolicited outside packet that has **no matching translation** generally cannot use an ordinary inside-initiated dynamic PAT entry to reach an inside host. Static mappings, explicit port forwarding, and certain other NAT features provide different inbound behavior.

### Existing Translations vs. New Mappings

When troubleshooting, distinguish between:

- **Configuration:** Which NAT rules are present?
- **Selection:** Which rule should match a new flow?
- **State:** Does a translation already exist for the flow?
- **Routing:** Can the packet be forwarded before or after translation as required?

In particular, changing NAT configuration does not necessarily remove translations that were already allocated. Existing entries can continue to affect testing until they expire or are cleared.

---

## NAT, ACLs, and Policy-Based Routing

Understanding which version of a packet a feature processes helps explain many NAT troubleshooting problems.

Cisco's traditional processing model places input ACL processing before NAT in both directions. It places PBR before NAT on inside-to-outside traffic, and after NAT on outside-to-inside traffic. It also places output ACL processing after the relevant NAT translation.

### Addresses Seen by Interface ACLs

For the PAT example above:

| Packet direction | Input ACL sees | Output ACL sees |
|---|---|---|
| Inside to outside | Source `10.1.1.10` | Source `192.0.2.2` |
| Outside to inside | Destination `192.0.2.2` | Destination `10.1.1.10` |

This table assumes the conventional IOS processing order and the ordinary interface ACL attachment points described above. Verify interactions on the actual IOS XE platform when other features are involved.

An ACL used by a NAT rule has a **different purpose** from an interface ACL:

- An ACL referenced by `ip nat inside source list` identifies traffic eligible for translation. A `permit` means **eligible for NAT**, not necessarily permission to forward through an interface.
- An ACL attached with `ip access-group` filters traffic. A `deny` can **drop the packet**, rather than merely exclude it from NAT.

For example:

```text
R1(config)# access-list 10 permit 10.1.1.0 0.0.0.255
R1(config)# ip nat inside source list 10 interface GigabitEthernet2 overload
```

ACL 10 does **not** automatically allow those packets through an inbound or outbound interface ACL. It selects which sources qualify for this NAT rule.

### Policy-Based Routing

**Policy-Based Routing (PBR)** may override the normal destination-based routing decision before inside-to-outside NAT.

This interaction is particularly important with multiple WAN connections:

```text
                  ISP1
                 /
Inside --- R1 --+
                 \
                  ISP2
```

For example:

1. A packet enters R1 from the inside network.
2. PBR selects ISP2 as the forwarding path.
3. NAT applies an eligible translation, which may be selected using a NAT route map and the chosen egress interface.
4. The packet leaves through ISP2 with an appropriate translated source address.

If NAT selects ISP1's address while the packet is forwarded through ISP2, the ISP might reject the packet or the return path might not lead back through the correct NAT router and translation state.

**Remember:** A PBR route map and a NAT route map are distinct mechanisms, even though both use route-map syntax. A PBR route map changes forwarding decisions; a NAT route map selects translation behavior.

See [Route-Map-Based NAT and Conditional Translation](policy-based/route-maps.md) and [Multi-ISP NAT and Failover](policy-based/multi-egress.md).

---

## NAT and Routing with VRFs

With **VRF-aware NAT**, routing lookups must occur in the correct routing context, not necessarily the global routing table.

For example, an interface may belong to `VRF_A` while the WAN-facing interface belongs to the global routing table. NAT must be configured for the supported relationship between those routing contexts.

The same directional principle still helps explain packet processing, but a plain `show ip route` on the router might not display the route used for a packet in a VRF.

Useful commands include:

```text
R1# show ip route vrf VRF_A
R1# show ip route vrf VRF_A 203.0.113.10
R1# show ip cef vrf VRF_A 203.0.113.10 detail
R1# show ip nat translations
```

### VASI NAT

**VRF-Aware Software Infrastructure (VASI)** uses a virtual interface pair to forward packets between routing contexts within an IOS XE router.

When VASI and NAT are combined, the packet may undergo **multiple routing lookups**, one on each side of the VASI pair, before or after translation as determined by the interface roles and packet direction.

For VASI troubleshooting, explicitly trace:

1. The original ingress VRF.
2. The route toward the first VASI interface.
3. The partner VASI interface and its VRF.
4. Which interfaces are configured as NAT inside and outside.
5. The NAT translation and the subsequent forwarding path.
6. The reverse path in the appropriate routing contexts.

Do not assume that every VASI packet follows the single-lookup topology used in a basic NAT example.

Traditional NAT-on-a-stick designs can also require PBR or an additional logical forwarding path to force traffic across an inside/outside NAT boundary. Merely routing traffic back out the same interface does not automatically provide the intended translation. See [NAT-on-a-Stick with PBR](policy-based/nat-on-a-stick.md).

**IOS XE does not support NAT Virtual Interface (NVI)** (`ip nat enable`). VASI is a distinct feature, not an alternative spelling of NVI.

See [VRF-Aware NAT, PAT](vrf/overview.md), [VASI NAT](vasi/overview.md), and the existing [VASI routing page](../../../network-infrastructure/routing-concepts/route-leaking/vasi.md).

---

## NAT and IPsec

Cisco's classic order-of-operations reference places IPsec decryption and inbound processing before ordinary outside-to-inside NAT. On the outbound path, NAT processing precedes the classic crypto-map encryption decision.

This helps explain why NAT and IPsec configurations must be coordinated:

- A packet might need to match a crypto policy using a particular address representation.
- A NAT exemption or an explicitly designed translation policy may be required when traffic should enter an IPsec tunnel without unwanted address changes.
- NAT Traversal (NAT-T) addresses the transport of IPsec through a NAT device; it does not change the basic meaning of inside local and inside global addresses.

The precise pipeline depends on the VPN technology, attachment point, and IOS XE platform. Do not apply the legacy crypto-map ordering indiscriminately to every tunnel or SD-WAN forwarding path.

---

## NAT Troubleshooting Workflow

When a NAT configuration appears correct but traffic fails, use a consistent procedure.

### Step 1: Confirm the Interface Roles

```text
R1# show running-config interface GigabitEthernet1
R1# show running-config interface GigabitEthernet2
```

Verify `ip nat inside`, `ip nat outside`, the IP addresses, and any relevant VRF assignments.

### Step 2: Verify the Route at the Correct Processing Stage

For **inside-to-outside** traffic, check how the **pre-NAT destination** is routed:

```text
R1# show ip route 203.0.113.10
R1# show ip cef 203.0.113.10 detail
```

For **outside-to-inside** traffic, verify how the **post-NAT destination** is routed:

```text
R1# show ip route 10.1.1.10
R1# show ip cef 10.1.1.10 detail
```

For **outside source NAT**, specifically check the outside-local destination that will be examined *before* inside-to-outside translation:

```text
R1# show ip route 172.16.50.10
```

Also verify the upstream route or ARP resolution used to deliver packets to the inside-global address when appropriate.

### Step 3: Verify NAT Configuration and State

```text
R1# show running-config | include ^ip nat
R1# show ip nat translations
R1# show ip nat translations verbose
R1# show ip nat statistics
```

Examine the inside local/global and outside local/global columns, the transport ports, the translation counters, and the NAT pool information where relevant.

### Step 4: Check ACLs, PBR, and NAT Route Maps

```text
R1# show access-lists
R1# show route-map
R1# show running-config interface GigabitEthernet1
R1# show running-config interface GigabitEthernet2
```

Determine whether an interface ACL drops a packet, whether PBR chooses an unexpected exit, or whether a NAT rule does not match the desired flow.

### Step 5: Test One Flow in Both Directions

For example, generate a single TCP connection or ICMP exchange and record:

| Observation point | Questions |
|---|---|
| Inside ingress | What are the original source and destination? |
| Pre-NAT routing | Is the original destination reachable through the expected path? |
| NAT translation | Which address or port is being changed? |
| Outside egress | Does the packet have the intended translated address? |
| Outside ingress | Does the reply arrive at the NAT router? |
| Post-NAT routing | Is the translated destination reachable on the inside? |

On an appropriate lab device, `debug ip nat` can help confirm individual translations. Debugging commands can produce significant output and affect forwarding performance; use platform-appropriate packet tracing and debug procedures, especially on production IOS XE devices.

---

## Common NAT Order-of-Operations Mistakes

| Mistake | Correct explanation |
|---|---|
| Assuming NAT always precedes routing | Inside-to-outside traffic is routed before NAT in the traditional model |
| Assuming routing always precedes NAT | Outside-to-inside traffic is translated before routing in the traditional model |
| Checking the public destination on an outside-to-inside routing lookup | The router can route using the translated **inside local** destination |
| Checking the outside global destination for an inside-to-outside `outside source` packet | The router initially routes using the **outside local** destination |
| Assuming NAT automatically provides routing | NAT changes addresses; routes must still be present at the correct processing stages |
| Confusing a NAT ACL with an interface ACL | NAT ACLs select eligible traffic; interface ACLs permit or deny forwarding |
| Assuming dynamic PAT supports arbitrary inbound sessions | Typical dynamic PAT requires an existing translation for the returning traffic |
| Assuming PBR and NAT are unrelated | PBR can determine the path that influences NAT policy selection |
| Applying the classic single-routing-table example to VASI | VASI traffic can cross multiple routing contexts and require multiple lookups |

---

## Key Takeaways

1. **Inside to outside: route first, translate second.**
2. **Outside to inside: translate first, route second.**
3. **Translation type and packet direction are separate concepts.** An `outside source` NAT rule can modify a destination address on an inside-to-outside packet.
4. **The routing lookup must use the appropriate destination address at that stage.** This is especially important for outside source NAT.
5. **Input ACLs and output ACLs may see different address representations** because of NAT processing order.
6. **PBR, VRFs, and VASI complicate the forwarding path**, so verify routes, interfaces, and NAT state together.

---

## References

- [Cisco: Understand the NAT Order of Operation](https://www.cisco.com/c/en/us/support/docs/ip/network-address-translation-nat/6209-5.html)
- [Cisco: IP Addressing Configuration Guide, IOS XE 17.x, Configuring NAT for IP Address Conservation](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-addressing/b-ip-addressing/m_iadnat-addr-consv-xe.html)
- [Cisco: Configure IP NAT Outside Source Static Command](https://www.cisco.com/c/en/us/support/docs/ip/ip-addressing-services/218383-use-ip-nat-command-for-configuration.html)
- [Cisco: Configure VRF-Aware Software Infrastructure NAT on IOS XE](https://www.cisco.com/c/en/us/support/docs/ip/network-address-translation-nat/200255-Configure-VRF-Aware-Software-Infrastruct.html)
