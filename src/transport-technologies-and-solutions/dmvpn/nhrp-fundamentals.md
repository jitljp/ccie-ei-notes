# NHRP Fundamentals

**NHRP** (Next Hop Resolution Protocol) allows a DMVPN router to discover the **NBMA (underlay) address** associated with an **overlay next hop**. It supplies the address mappings that multipoint GRE (mGRE) needs to send packets to dynamically selected tunnel peers.

In a static point-to-point GRE tunnel, the remote endpoint is explicitly configured with `tunnel destination`. An mGRE tunnel has no single configured destination, so the router must determine the appropriate outer destination address for each remote peer. **NHRP provides that mapping.**

NHRP is defined in [RFC 2332](https://www.rfc-editor.org/rfc/rfc2332.html). Its use with mGRE is a central component of Cisco DMVPN.

---

## Overlay Addresses and NBMA Addresses

NHRP works with two different address spaces:

| Address | Purpose | Example |
| --- | --- | --- |
| **Overlay (protocol) address** | Identifies a router within the DMVPN tunnel network; may be used as a routing next hop. | `172.16.0.12` |
| **NBMA address** | Identifies the transport endpoint to which an encapsulated packet must be delivered across the underlay. | `198.51.100.13` |

**NBMA** stands for **Non-Broadcast Multiple Access**. In DMVPN, the NBMA address is normally the tunnel source's WAN/underlay address. Although the underlay itself may consist of ordinary Ethernet and IP links, **NBMA** is the terminology NHRP uses for the underlying transport network.

For example, Spoke2 might use:

```text
Spoke2 tunnel address: 172.16.0.12
Spoke2 NBMA address:   198.51.100.13
```

A router that needs to send GRE packets directly to Spoke2 must be able to resolve:

```text
172.16.0.12 (overlay next hop)
              |
              v
198.51.100.13 (outer GRE destination)
```

This resembles ARP conceptually, but **NHRP maps an overlay protocol address to an NBMA address**, rather than an IPv4 address to an Ethernet MAC address. The actual underlay still uses normal routing and, where applicable, ARP to deliver the encapsulated packets.

NHRP does **not** independently advertise the spoke's LAN prefixes or choose the best overlay route. A routing protocol or static route determines which overlay next hop should be used; NHRP supplies its NBMA mapping. Phase 3 additionally supports NHRP routes and next-hop overrides, covered later in the chapter.

---

## NHRP Roles: NHS and NHC

NHRP defines two important roles:

| Role | Full name | Function in DMVPN |
| --- | --- | --- |
| **NHS** | Next Hop Server | Maintains registrations and helps forward resolution requests toward the destination. Usually a hub. |
| **NHC** | Next Hop Client | Registers its own overlay-to-NBMA mapping and requests resolution of remote peers. Usually a spoke. |

The terms **hub** and **spoke** describe the DMVPN topology. **NHS** and **NHC** describe NHRP functions. In a conventional DMVPN deployment, hubs act as NHSs and spokes act as NHCs.

```text
                    Hub1
                     NHS
                      |
               DMVPN overlay
                /     |     \\
               /      |      \\
           Spoke1   Spoke2   Spoke3
            NHC      NHC      NHC
```

The NHS gives spokes a stable point through which they can register and initiate resolution. The spokes do not need to maintain permanent static mappings for every other spoke.

**An NHS does not need to forward every data packet.** In Phases 2 and 3, the hub assists with NHRP control-plane operations while spoke-to-spoke user traffic can take a direct GRE path.

---

## NHRP Registration

A spoke must register with its NHS so that the NHS knows how to reach it. Registration establishes an association between the spoke's **tunnel address** and its **NBMA address**.

Consider Hub1 and Spoke1:

```text
Hub1:
  Tunnel0:      172.16.0.1
  NBMA:         198.51.100.1
Spoke1:
  Tunnel0:      172.16.0.11
  NBMA:         198.51.100.9
Spoke1                         Hub1 (NHS)
   |                              |
   |--- NHRP Registration Req --->|
   |   172.16.0.11 ->             |
   |   198.51.100.9               |
   |                              | Stores mapping
   |<-- NHRP Registration Reply --|
   |                              |
```

### Registration Sequence

1. Spoke1 knows Hub1's tunnel address and NBMA address from its configuration (or another supported NHS discovery method).
2. Spoke1 sends an **NHRP Registration Request** to the NHS through the tunnel.
3. The request identifies Spoke1's overlay address and NBMA address, as well as information such as the advertised holding time.
4. Hub1 processes the request and creates or updates its NHRP registration entry.
5. Hub1 returns an **NHRP Registration Reply**, indicating whether registration was accepted.
6. Spoke1 periodically refreshes its registration to keep the entry from expiring.

At the hub, the resulting information is conceptually:

| Registered overlay address | NBMA address | Learned through |
| --- | --- | --- |
| `172.16.0.11` | `198.51.100.9` | Spoke1 registration |
| `172.16.0.12` | `198.51.100.13` | Spoke2 registration |
| `172.16.0.13` | `198.51.100.17` | Spoke3 registration |

These are **dynamic NHRP mappings**, not manually configured static mappings on the hub.

### Unique Registrations and Changing NBMA Addresses

An NHRP registration normally identifies a unique protocol address. In traditional configurations, the **unique** flag prevents a new registration with the same overlay address but a different NBMA address from silently replacing the existing entry.

This can be useful for detecting two routers accidentally configured with the same tunnel IP address.

However, it can cause difficulties when a spoke's WAN address legitimately changes. Suppose:

```text
Original registration:
  172.16.0.11 -> 198.51.100.9
Spoke's WAN address changes:
  172.16.0.11 -> new NBMA address
```

With a unique registration, the NHS may reject the replacement mapping while the previous registration remains valid.

The spoke can allow its registration to replace an existing mapping with a different NBMA address:

```cisco
interface Tunnel0
 ip nhrp registration no-unique
```

This is especially relevant to dynamic WAN addressing. Cisco's NHRP smart defaults also enable `registration no-unique` automatically in supported implementations.

**Do not confuse `registration no-unique` with allowing duplicate overlay addresses.** Overlay tunnel addresses should still be unique per DMVPN cloud; the command changes how the NHS treats a replacement NBMA mapping for the same overlay address.

### Registration and Tunnel State Are Different

A tunnel interface can appear `up/up` even when NHRP registration has failed. An mGRE interface does not prove that an NHS is reachable merely because its line protocol is up.

Similarly, successful NHRP registration demonstrates that the spoke can communicate with the NHS, but it does **not** prove that:

- Overlay routing neighbors have formed.
- Internal LAN prefixes are reachable.
- Direct spoke-to-spoke tunnels are working.
- Phase 3 shortcuts have been installed.

Use `show ip nhrp nhs detail` on the spoke and `show ip nhrp` on the hub to examine registration rather than relying on the tunnel interface state alone.

---

## NHRP Resolution

**Registration** tells the NHS where a spoke is. **Resolution** allows another router to discover how to reach that spoke.

In a Phase 2 DMVPN, a spoke's routing table may contain a route to a remote LAN with the destination spoke's **tunnel IP address** as the next hop.

For example, suppose Spoke1 wants to reach Spoke2's LAN:

```text
Spoke1 route:
  10.12.12.0/24 -> 172.16.0.12 (Spoke2 Tunnel0)
Spoke1 currently has no NHRP mapping for 172.16.0.12.
```

NHRP resolution proceeds as follows:

```text
Spoke1                   Hub1                      Spoke2
   |                      |                           |
   |-- Resolution Req --->|                           |
   |  Target: 172.16.0.12 |-- Resolution Req -------->|
   |                      |                           |
   |                      |           Creates mapping |
   |<---------------------------- Resolution Reply ---|
   |                      |   NBMA: 198.51.100.13    |
   |                      |                           |
   |<============ Direct GRE packets ===============>|
```

1. Spoke1's routing lookup identifies Spoke2's **overlay address** as the next hop.
2. Because Spoke1 has no usable NHRP mapping for that next hop, it sends an **NHRP Resolution Request** toward its NHS, Hub1.
3. Hub1 uses its registration information and overlay reachability to forward the request toward Spoke2.
4. Spoke2 receives the request and returns an **NHRP Resolution Reply** with its NBMA information, normally directly to Spoke1.
5. Spoke1 caches Spoke2's overlay-to-NBMA mapping.
6. Subsequent packets can be GRE-encapsulated directly toward Spoke2's NBMA address, provided the underlay and any required IPsec protection permit the path.

**The hub normally forwards the resolution request; the destination spoke sends the resolution reply.** The hub is not simply a centralized ARP server that always answers on behalf of the destination spoke.

NHRP may also learn useful information from NHRP control packets traversing a router, but such learned entries are not the same as authoritative registrations.

### What Happens While Resolution Is Pending?

The first packet does not automatically travel directly to the destination spoke. While resolution is pending, forwarding depends on the available route and implementation behavior; packets can be delayed or dropped, or they may follow an existing hub path. After resolution completes, the corresponding traffic can use the direct tunnel.

In **Phase 3**, a source spoke initially forwards toward the hub and is prompted to discover a better path by an **NHRP redirect**. The resolution process then supports a **shortcut** for traffic matching the resolved destination. That difference is examined in the Phase 3 pages.

---

## NHRP Message Types

The original NHRP specification defines these packet types:

| Type | Message | Purpose |
| ---: | --- | --- |
| **1** | Resolution Request | Requests NBMA information for a destination protocol address. |
| **2** | Resolution Reply | Returns the requested mapping or an unsuccessful resolution result. |
| **3** | Registration Request | Registers an NHC's protocol-to-NBMA mapping with an NHS. |
| **4** | Registration Reply | Acknowledges the result of a registration attempt. |
| **5** | Purge Request | Instructs a peer to invalidate previously learned mapping information. |
| **6** | Purge Reply | Acknowledges a purge request. |
| **7** | Error Indication | Reports an NHRP processing error. |

### Purge Messages

Mappings can become invalid before their holding times expire. **NHRP Purge Requests** provide a way to tell other routers that certain cached information must be removed; the recipient acknowledges the purge with a **Purge Reply**.

For example, a change to the prefix or next hop previously returned by NHRP may require the requesting router to discard an old shortcut mapping instead of continuing to use it.

### Negative Resolution and Errors

An NHS or destination might be unable to supply a valid mapping. NHRP can return an unsuccessful resolution result, and an NHRP cache may record negative or incomplete resolution state. Repeatedly attempting to forward via an unresolved peer will not create a working direct tunnel.

**NHRP Error Indications** report protocol-processing problems, such as an invalid or unrecognized mandatory extension or an authentication failure.

Cisco's Phase 3 **redirect/traffic-indication** mechanism extends the basic registration-and-resolution behavior. It is covered separately under [NHRP Redirects and Shortcuts](phase3/nhrp/redirect-shortcut.md).

---

## Static and Dynamic NHRP Mappings

An NHRP mapping can be **statically configured** or **dynamically learned**.

| Mapping type | How it is created | Typical use |
| --- | --- | --- |
| **Static** | `ip nhrp map` | A spoke's mapping of its known hub tunnel address to the hub NBMA address. |
| **Dynamic registration** | Received NHRP Registration Request | An NHS learns a spoke's tunnel-to-NBMA mapping. |
| **Dynamic resolution** | Received NHRP resolution information | A spoke learns another spoke's NBMA address for direct forwarding. |

### Static Hub Mapping on a Spoke

For the chapter's addressing plan, Spoke1 can use:

```cisco
interface Tunnel0
 ip nhrp map 172.16.0.1 198.51.100.1
 ip nhrp nhs 172.16.0.1
```

The two commands serve different purposes:

- `ip nhrp map 172.16.0.1 198.51.100.1` defines the **tunnel-IP-to-NBMA mapping** for Hub1.
- `ip nhrp nhs 172.16.0.1` tells Spoke1 to use **Hub1's tunnel IP address** as an NHS.

**The `ip nhrp nhs` address is the hub's tunnel address, not its WAN/NBMA address.** Confusing those addresses is a common reason for registration failures.

### Dynamic Mappings on the Hub

The hub does not need individual `ip nhrp map` commands for all its spokes. Instead, the spoke registrations populate the hub's NHRP cache automatically.

The hub can then forward unicast GRE traffic toward the registered spokes without a separate tunnel interface or a separately configured GRE destination for each spoke.

---

## NHRP Multicast Mappings

An mGRE tunnel can send unicast traffic to many remote endpoints, but multicast or broadcast traffic requires special handling because the underlay normally does not deliver overlay multicast to every DMVPN router automatically.

An important example is **routing-protocol multicast**, such as EIGRP hellos or OSPF hellos on supported network types.

### On a Spoke

A spoke commonly sends multicast traffic to its hub using:

```cisco
interface Tunnel0
 ip nhrp map multicast 198.51.100.1
```

This identifies Hub1's **NBMA address** as a destination for multicast packets transmitted through the tunnel.

The `multicast` mapping is **not** the same as the unicast `ip nhrp map` entry. Both may be needed:

```cisco
ip nhrp map 172.16.0.1 198.51.100.1
ip nhrp map multicast 198.51.100.1
```

The first maps Hub1's overlay address for unicast forwarding; the second permits replication of multicast/broadcast traffic toward Hub1.

### On a Hub

A hub can learn multicast replication destinations from spoke registrations:

```cisco
interface Tunnel0
 ip nhrp map multicast dynamic
```

With dynamic multicast mapping, the hub maintains a list of registered spokes' NBMA addresses and can **replicate** an overlay multicast packet as separate GRE packets sent to those endpoints.

```text
                  Hub1
            Multicast received
                   |
         +---------+---------+
         |         |         |
       GRE to    GRE to    GRE to
       Spoke1    Spoke2    Spoke3
```

The underlay does not need to run a multicast routing protocol to carry these replicated packets.

**Multicast replication destinations are not learned merely because a spoke has a unicast NHRP entry for another spoke.** In typical DMVPN, spokes direct multicast toward configured NHSs; the NHS learns registered spoke endpoints for multicast replication.

\> On some IOS XE releases, including releases using NHRP smart defaults, dynamic multicast mapping can be enabled automatically. The explicit command remains useful for understanding and documenting the intended behavior.

### Verifying Multicast Mappings

```cisco
show ip nhrp multicast
show ip nhrp
show dmvpn
```

If unicast tunnel pings work but an EIGRP or OSPF adjacency does not form, missing multicast replication is one of the first things to investigate.

---

## NHRP Network ID, GRE Tunnel Key, and Authentication

These three parameters have different purposes and should not be confused.

### NHRP Network ID

```cisco
interface Tunnel0
 ip nhrp network-id 100
```

The NHRP network ID identifies an NHRP network **locally on the router**. It does not have to be numerically identical on every router in a DMVPN cloud.

Using the same value throughout the cloud is still a sensible convention. It makes configurations easier to read, but matching network IDs do not by themselves prove that two routers belong to the same working DMVPN cloud.

Some newer IOS XE NHRP implementations can derive a default network ID from the GRE tunnel key or tunnel interface number when NHRP smart defaults are in effect.

### GRE Tunnel Key

```cisco
interface Tunnel0
 tunnel key 100
```

The **GRE tunnel key** is an optional field in the GRE header used to distinguish tunnel traffic. When configured, the key must match between communicating endpoints in that DMVPN cloud.

Unlike the locally significant NHRP network ID, the GRE key is associated with packets on the wire. A mismatch can prevent received GRE packets from matching the intended tunnel interface.

**The GRE tunnel key is not a cryptographic key.** It does not encrypt traffic or authenticate peers.

### NHRP Authentication

```cisco
interface Tunnel0
 ip nhrp authentication DMVPN
```

The NHRP authentication string is checked in NHRP control messages. The configured string must match between participating routers that exchange those NHRP messages.

This provides a basic consistency/authentication check, but **NHRP authentication is not a substitute for IKEv2/IPsec**. In the commonly used Cisco implementation, the NHRP authentication extension uses a cleartext string.

| Feature | What it identifies or checks | Must match across peers? |
| --- | --- | --- |
| `ip nhrp network-id` | Local NHRP interface/network context | **No**, locally significant |
| `tunnel key` | GRE tunnel identifier in the packet | **Yes**, when used on communicating peers |
| `ip nhrp authentication` | NHRP control-plane authentication string | **Yes**, when configured |

---

## NHRP Cache Entries and Flags

Use the following command to inspect mappings:

```cisco
show ip nhrp
```

The output associates an overlay IP address or prefix with a tunnel interface, an NBMA address, a mapping type, flags, and potentially a remaining expiration time.

### Static, Dynamic, and Incomplete State

- **Static:** Configured explicitly; a normal static mapping does not age out like a dynamically learned mapping.
- **Dynamic:** Learned through registration, resolution, or another NHRP learning mechanism; usually subject to aging.
- **Incomplete:** Resolution has been attempted but a usable mapping has not yet been obtained.

### Common Cache Flags

| Flag | Meaning |
| --- | --- |
| `authoritative` | The mapping originates from the router responsible for the information or an authoritative NHRP source. |
| `unique` | The registered protocol-address mapping is intended to be unique; a conflicting NBMA mapping cannot simply overwrite it. |
| `registered` | The entry was created from an NHRP registration request. |
| `used` | The mapping has recently been used for packet forwarding. |
| `router` | The mapping refers to a router that provides access to another network or host. |
| `implicit` | Mapping information was learned incidentally while processing NHRP control messages. |
| `negative` | A requested mapping could not be resolved. |
| `local` | A locally maintained entry for a prefix for which the router has responded to an NHRP request. |

**`registered` and `used` mean different things.** A hub can maintain a registration for a spoke even if that mapping has not recently carried data traffic. The `used` flag relates to forwarding use and can affect refreshing behavior.

Also, the `authoritative` flag does not mean that the packet has been cryptographically authenticated by IPsec; it describes the authority of the NHRP mapping information.

---

## Holding Time and Registration Refresh

NHRP mappings are not necessarily permanent. The router that advertises mapping information includes a **holding time** indicating how long the recipient may treat that information as valid.

```cisco
interface Tunnel0
 ip nhrp holdtime 600
```

A holding time of **600 seconds** means that a receiving router can retain the advertised mapping for up to **10 minutes** before it must be refreshed or removed.

### Registration Refresh Interval

Spokes periodically refresh their NHS registrations. In the conventional NHRP behavior, registration requests are sent every **one-third of the configured NHRP holdtime** unless the interval is overridden.

For example:

| Configured holdtime | Normal registration refresh interval |
| ---: | ---: |
| 300 seconds | 100 seconds |
| 450 seconds | 150 seconds |
| 600 seconds | 200 seconds |
| 7200 seconds | 2400 seconds |

Cisco documentation has described **7200 seconds** as the traditional NHRP default holdtime; newer **NHRP smart defaults** use **600 seconds**, with registrations every 200 seconds. The effective default therefore depends on the NHRP implementation and feature behavior. An explicit holdtime avoids ambiguity.

The refresh interval can also be configured independently:

```cisco
interface Tunnel0
 ip nhrp holdtime 600
 ip nhrp registration timeout 100
```

In this example, the advertised holding time is **600 seconds**, while the registration interval is **100 seconds**. The `registration timeout` command does **not** change the lifetime advertised in the mapping.

### Registration Retransmissions and NHS Reachability

When a spoke does not receive a Registration Reply, NHRP retransmits registration requests. Cisco documents exponential retransmission intervals of **1, 2, 4, 8, 16, 32, and 64 seconds** for the classic NHRP registration process.

A spoke can mark an NHS unresponsive when registration replies are not received, while continuing registration probes so that the NHS can be used again after it responds.

The important distinction is:

- **Registration refresh** maintains an otherwise healthy mapping at the NHS.
- **Retransmission/retry behavior** handles a request for which no reply was received.
- **Holding time** controls the validity of advertised mapping information.

These mechanisms are separate from IGP hello/hold timers and IKEv2/IPsec security-association lifetimes.

### Resolution Cache Refresh

A dynamically resolved spoke-to-spoke mapping can also be refreshed while in use. If the mapping is no longer needed, it can be allowed to expire. Expiration of an NHRP mapping can remove the associated forwarding state and, where tunnel protection is used, may affect the corresponding IPsec peer session.

Do not assume that **IPsec SA lifetime = NHRP cache lifetime**. The protocols maintain distinct state and timers.

---

## NHRP with Multiple NHSs

A spoke may register with more than one NHS. In the chapter's dual-hub **single-cloud** design, the spoke has one mGRE interface and two NHSs:

```text
                      Hub1 (NHS)
                     /
Spoke1 Tunnel0 -----+
                     \\
                      Hub2 (NHS)
```

With the proposed addressing, Spoke1 can have:

```cisco
interface Tunnel0
 ip nhrp map 172.16.0.1 198.51.100.1
 ip nhrp map 172.16.0.2 198.51.100.5
 ip nhrp map multicast 198.51.100.1
 ip nhrp map multicast 198.51.100.5
 ip nhrp nhs 172.16.0.1
 ip nhrp nhs 172.16.0.2
```

Each NHS must be reachable through the underlay and must be able to receive the spoke's NHRP registration. **Registration with one hub does not automatically imply registration with the other hub.** Each NHS maintains the information it learns, unless additional synchronization mechanisms are explicitly provided.

In a **dual-cloud** design, each spoke has a separate mGRE tunnel interface and NHRP context for each cloud. The two clouds usually use different overlay subnets and may use different GRE tunnel keys.

Multiple NHSs can improve availability, but they do not by themselves determine which hub the routing protocol uses as the forwarding next hop. **Hub preference and failover are partly routing decisions, not simply NHRP decisions.**

The details of single-cloud and dual-cloud configuration appear later in the chapter.

---

## NHRP, Routing, and Packet Forwarding

It is useful to separate three operations:

1. **Overlay routing:** Chooses a route to the destination network and, when applicable, an overlay next hop.
2. **NHRP:** Resolves the selected overlay next hop to its remote NBMA address, or supplies Phase 3 shortcut forwarding information.
3. **Underlay routing:** Delivers the encapsulated GRE packet to that NBMA address.

For a Phase 2 example:

```text
Destination LAN:        10.12.12.0/24
Overlay next hop:       172.16.0.12
NHRP-mapped NBMA:       198.51.100.13
Underlay next hop:      Local ISP router
```

The data-plane sequence is:

```text
Original packet for 10.12.12.12
             |
             v
Overlay route selects 172.16.0.12
             |
             v
NHRP resolves 172.16.0.12 -> 198.51.100.13
             |
             v
mGRE encapsulates toward 198.51.100.13
             |
             v
Underlay forwards the outer packet
```

A valid NHRP mapping is **necessary but not sufficient** for forwarding. The router also needs an appropriate overlay route, a working underlay path to the NBMA address, and functional IPsec protection if it is configured.

### Phase 3 Exception: NHRP Shortcuts Can Affect Forwarding

The statement that "NHRP does not install routes" is **too broad** for modern Phase 3 DMVPN. Phase 3 shortcut switching can create **NHRP routes** or **next-hop overrides** in addition to regular tunnel-to-NBMA mappings.

Useful commands include:

```cisco
show ip nhrp shortcut
show ip route nhrp
show ip route next-hop-override
```

Therefore, distinguish **basic NHRP address resolution** from **Phase 3 NHRP shortcut forwarding**. The details belong in [NHRP Redirects and Shortcuts](phase3/nhrp/redirect-shortcut.md).

---

## NHRP and NAT

If a spoke is behind NAT, the address it puts into NHRP control information may differ from the outer source IP address the hub actually sees.

Cisco's DMVPN NAT support can record both the **claimed (pre-NAT) NBMA address** and the **observed (post-NAT) address**, and NHRP NAT extensions can assist endpoint discovery.

However, two spokes behind NAT are not guaranteed to establish a direct tunnel. NAT behavior, path reachability, and IPsec NAT traversal can restrict spoke-to-spoke forwarding or force traffic through a hub.

For the labs in this chapter, all five DMVPN routers have directly reachable underlay addresses, so NAT is not required to demonstrate NHRP fundamentals.

---

## NHRP Verification and Troubleshooting

### Important Show Commands

| Command | What to inspect |
| --- | --- |
| `show dmvpn` | DMVPN peers, tunnel endpoints, and session state. |
| `show dmvpn detail` | More detailed tunnel, NHRP, and related session information. |
| `show ip nhrp` | Static and dynamic overlay-to-NBMA mappings, flags, and expiration timers. |
| `show ip nhrp detail` | Additional information about NHRP cache entries. |
| `show ip nhrp nhs detail` | Configured NHSs, reply state, registration requests, failures, and retries. |
| `show ip nhrp multicast` | Multicast replication mappings and NBMA destinations. |
| `show ip nhrp traffic` | Counters for NHRP registration, resolution, purge, error, and other control messages. |
| `show ip route` | Overlay routes and their selected next hops. |
| `show ip cef` | Forwarding information used by the data plane. |

### NHRP Packet Counters

`show ip nhrp traffic` is particularly helpful for locating the point where a control-plane exchange stops.

For example:

- **Registration Requests sent, but no Registration Replies received:** Examine NHS reachability, the `ip nhrp nhs` address, mapping configuration, NHRP authentication, and any IPsec protection.
- **Hub receives Registration Requests but does not show a valid spoke mapping:** Examine registration errors, address conflicts, and holding-time behavior.
- **Resolution Requests increase, but Resolution Replies do not:** Determine whether requests are reaching the destination spoke and whether replies can return directly.
- **Dynamic mapping exists but data fails:** Examine overlay routes, actual NBMA reachability, IPsec SAs, and GRE encapsulation.
- **Unicast works but routing adjacencies fail:** Examine static and dynamic NHRP multicast mappings.

### Debugging

When deeper inspection is needed:

```cisco
debug nhrp
```

NHRP debugging can reveal registration and resolution exchanges, the source and destination protocol/NBMA addresses, request IDs, and error information. On a busy hub it can generate substantial output; use targeted debugging when possible.

Cisco also supports commands such as `debug nhrp condition` to constrain output to a peer or tunnel.

### Troubleshooting Order

A practical sequence is:

1. Confirm the source router has underlay reachability to its NHS NBMA address.
2. Confirm that the configured NHS **tunnel IP** is correct and has a usable NHRP mapping to the NHS **NBMA IP**.
3. Examine the spoke's NHS registration status.
4. Examine the hub's dynamically registered mapping for that spoke.
5. Verify NHRP multicast mappings if routing neighbors use multicast hellos.
6. Verify the overlay route and its next hop.
7. If a direct spoke-to-spoke path is expected, examine resolution requests and replies and the resulting NHRP cache entry.
8. If IPsec protection is configured, examine IKEv2/IPsec session status separately.

A functional NHRP registration confirms only one part of the DMVPN control plane. It should not be treated as proof that routing, shortcuts, or encrypted forwarding are working.

---

## Key Points

- **NHRP resolves an overlay next-hop address to a transport/NBMA address** for mGRE forwarding.
- **NHSs** maintain registrations and assist with resolution; **NHCs** register and request resolution.
- **Registration Requests/Replies** establish and refresh tunnel-to-NBMA mappings at the NHS.
- **Resolution Requests/Replies** allow a spoke to discover another spoke's NBMA address when direct forwarding is supported.
- For typical Phase 2 spoke-to-spoke resolution, the **hub forwards the request and the destination spoke sends the reply**.
- `ip nhrp map` configures a **static unicast overlay-to-NBMA mapping**; `ip nhrp nhs` identifies the NHS by its **tunnel IP address**.
- `ip nhrp map multicast` specifies multicast replication destinations; `ip nhrp map multicast dynamic` allows the hub to learn destinations from registrations.
- The **NHRP network ID is locally significant**. A configured **GRE tunnel key must match** between communicating peers.
- **NHRP authentication** is not a replacement for IKEv2/IPsec protection.
- **Holdtime** controls the lifetime of advertised mappings. Conventional registration refresh occurs every **one-third of the holdtime**, unless overridden.
- Traditional NHRP defaults and newer **smart defaults** can differ, particularly for holdtime and `registration no-unique`.
- **Unique registrations** can prevent a changed NBMA address from overwriting an existing registration; `ip nhrp registration no-unique` permits replacement.
- **Multiple NHSs** require independent successful registration; adding an NHS does not itself determine overlay route preference.
- Phase 3 can install **NHRP routes and next-hop overrides**, not just address mappings.
- Use `show ip nhrp`, `show ip nhrp nhs detail`, `show ip nhrp multicast`, and `show ip nhrp traffic` to isolate control-plane failures.

---

## References

- [Cisco: Dynamic Multipoint VPN](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/security-vpn/security-vpn/m_sec-conn-dmvpn-dmvpn-0.html)
- [Cisco: Configuring NHRP](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/ipaddr_nhrp/configuration/15-s/nhrp-15-s-book/config-nhrp.html)
- [Cisco: Troubleshoot DMVPN Issues](https://www.cisco.com/c/en/us/support/docs/security/dynamic-multipoint-vpn-dmvpn/111976-dmvpn-troubleshoot-00.html)
- [Cisco: Troubleshoot DMVPN Phase 2 Spoke-to-Spoke Tunnel](https://www.cisco.com/c/en/us/support/docs/security/dynamic-multipoint-vpn-dmvpn/221665-troubleshoot-dmvpn-phase-2-spoke-to-spok.html)
- [Cisco: Shortcut Switching Enhancements for NHRP in DMVPN Networks](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/ip-addressing/ip-addressing/m_nhrp-switch-enhancemts-dmvpn-0.html)
- [Cisco: DMVPN Dynamic Tunnels Between Spokes Behind a NAT Device](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/sec_conn_dmvpn/configuration/xe-3s/sec-conn-dmvpn-xe-3s-book/sec-conn-dmvpn-dt-spokes-b-nat.html)
- [RFC 2332: NBMA Next Hop Resolution Protocol (NHRP)](https://www.rfc-editor.org/rfc/rfc2332.html)
- [IANA: NHRP Packet Types](https://www.iana.org/assignments/nhrp-types)
