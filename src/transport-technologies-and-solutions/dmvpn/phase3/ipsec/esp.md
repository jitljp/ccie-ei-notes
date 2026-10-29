# IPsec ESP and Tunnel Protection

**IPsec Encapsulating Security Payload (ESP)** protects the GRE packets carried across a DMVPN underlay. **IKEv2** authenticates the peers and negotiates the keys and security associations (SAs); **ESP** uses those negotiated SAs to protect actual traffic.

In DMVPN, an **IPsec transform set** defines the ESP algorithms and encapsulation mode, an **IPsec profile** collects the relevant security settings, and `tunnel protection ipsec profile` applies them to an mGRE tunnel interface. This allows a single mGRE interface to protect traffic to many dynamically discovered peers without configuring a separate static crypto map for every spoke.

This page covers ESP packet protection, transport and tunnel mode, IPsec transform sets and profiles, tunnel protection, and the DMVPN-specific implications of dynamic peers.

---

## GRE over IPsec in DMVPN

**GRE and ESP have different responsibilities:**

| Component | Responsibility |
| --- | --- |
| **GRE/mGRE** | Encapsulates overlay packets and permits multiple remote tunnel endpoints on a tunnel interface. |
| **NHRP** | Resolves an overlay tunnel/shortcut destination to the remote NBMA address. |
| **IKEv2** | Authenticates the remote peer and negotiates IKE and IPsec security associations. |
| **ESP** | Protects the GRE traffic using the negotiated IPsec SAs. |

The order of operations is important: **GRE encapsulation occurs before IPsec protection** on the sending router, and the receiving router removes the IPsec protection before decapsulating GRE.

```text
                Sending router                              Receiving router

    Original IP packet                                               Original IP packet
           |                                                                   ^
           v                                                                   |
       GRE encapsulation                                                GRE decapsulation
           |                                                                   ^
           v                                                                   |
       IPsec ESP protection                                           ESP verification /
           |                                                          decryption
           v                                                                   ^
      Underlay forwarding ------ ISP / underlay network -----------------------+
```

For example, Spoke1 can send traffic directly to Spoke2 after an NHRP shortcut resolves Spoke2's NBMA address. With tunnel protection enabled, the resulting GRE packets must be protected by an appropriate IPsec SA before crossing the underlay.

**IPsec does not establish the DMVPN overlay route or the NHRP shortcut.** It secures the traffic sent to the tunnel peer selected by the DMVPN forwarding process.

---

## ESP Security Services

ESP is **IP protocol 50** when transported directly over IP. It is different from **GRE (IP protocol 47)** and from IKEv2's UDP control traffic.

| Security service | How ESP provides it |
| --- | --- |
| **Confidentiality** | Encrypts protected data so an observer cannot read its contents. |
| **Integrity** | Detects unauthorized changes to the protected ESP packet. |
| **Data-origin authentication** | Verifies that the packet was produced by a peer possessing the SA's cryptographic key. |
| **Anti-replay protection** | Uses ESP sequence numbers and a receiving replay window to reject duplicates or sufficiently old packets. |

ESP can use separate encryption and integrity algorithms, or an **authenticated-encryption algorithm** such as AES-GCM that provides both confidentiality and integrity.

### ESP Header and Trailer

An ESP-protected packet has the following general structure:

```text
+------------------+------------------+-----------------------------+-----------------+
| Outer IP header  | ESP header       | Encrypted ESP payload       | Authentication  |
|                  | SPI + sequence # | and trailer                 | tag / ICV       |
+------------------+------------------+-----------------------------+-----------------+
                  <---- integrity/authentication coverage as applicable ---->
```

The exact layout depends on the algorithm and whether ESP uses transport or tunnel mode.

| Field | Role |
| --- | --- |
| **SPI (Security Parameter Index)** | 32-bit identifier used by the receiver to select the inbound ESP SA. |
| **Sequence number** | 32-bit value carried in the ESP header, used for anti-replay protection; Extended Sequence Numbers (ESN) can expand the logical counter to 64 bits. |
| **Encrypted payload** | Protected GRE data in transport mode, or an encapsulated IP packet in tunnel mode. |
| **ESP trailer** | Includes padding, pad length, and a Next Header field that identifies the protected payload type. |
| **Integrity tag / ICV** | Validates protected content; its precise placement and format depend on the negotiated algorithm. |

**The ESP SPI and sequence number are visible on the wire**, but they are integrity-protected when ESP integrity protection is enabled. The outer IP header remains available for routing; ESP does not encrypt that outer header.

### ESP SAs Are Unidirectional

An IKEv2 **Child SA** normally establishes a pair of unidirectional ESP SAs:

```text
                Spoke1                            Spoke2

     Outbound ESP SA (SPI A) --------------------> Inbound ESP SA (SPI A)
      Inbound ESP SA (SPI B) <-------------------- Outbound ESP SA (SPI B)
```

Each unidirectional ESP SA has its own **32-bit SPI**, keys, sequence number state, and negotiated protection parameters. The **inbound SPI on one router corresponds to the outbound SPI on the other router** for the same traffic direction.

Do not confuse the **two 64-bit SPIs identifying an IKEv2 SA** with the **32-bit SPI for each unidirectional ESP SA**.

### Anti-Replay Protection

The ESP receiver maintains a **sliding anti-replay window** for each inbound SA:

1. The sender increments the ESP sequence number as it sends packets.
2. The receiver checks whether a received sequence number is new and within the acceptable window.
3. Once integrity verification succeeds, the receiver updates its replay state.
4. Duplicate packets and packets that fall too far behind the window are discarded.

The window accommodates a limited amount of packet reordering. Excessive reordering can cause **replay drops even when the packets were legitimately sent**. A packet with valid encryption is not automatically accepted if its replay check fails.

---

## ESP Transport Mode Versus Tunnel Mode

An **IPsec transform set** selects `mode transport` or `mode tunnel`. This is **not the same setting** as `tunnel mode gre multipoint` on a DMVPN tunnel interface.

- `tunnel mode gre multipoint` determines how the **GRE tunnel** encapsulates traffic and handles multiple peers.
- `mode transport` or `mode tunnel` determines how **ESP** protects the GRE packet.

Both modes can protect GRE traffic, but their packet formats differ.

### GRE Without IPsec

Spoke1 (`198.51.100.9`) sends a packet to Spoke2 (`198.51.100.13`):

```text
+-----------------------------+-------------+-----------------------+
| Outer IPv4 header           | GRE header  | Original IP packet    |
| 198.51.100.9 -> .13         | Protocol 47 | 10.11.11.x -> 10.12.x |
+-----------------------------+-------------+-----------------------+
```

The outer IP header is used for underlay forwarding, but the GRE header and original packet are not encrypted.

### ESP Transport Mode

In **ESP transport mode**, IPsec retains the GRE packet's original outer IP header and protects its payload, including the **GRE header and encapsulated original IP packet**.

```text
+----------------------+---------------+-----------------------------+------------+
| Outer IPv4 header    | ESP header    | Encrypted:                  | ESP auth   |
| 198.51.100.9 -> .13  | SPI, Seq      | GRE + original IP + trailer | tag / ICV  |
| Protocol 50          |               |                             |            |
+----------------------+---------------+-----------------------------+------------+
```

The IP header that originally identified the GRE peers continues to identify the same endpoints. Its Protocol field identifies ESP rather than GRE (when ESP is carried directly over IP).

**Transport mode is typically preferred for DMVPN GRE-over-IPsec** because GRE has already supplied the IP encapsulation needed to carry the original packet. ESP transport mode therefore avoids adding another complete outer IP header solely for IPsec.

### ESP Tunnel Mode

In **ESP tunnel mode**, ESP protects the **entire original GRE IP packet**, including its original outer IP header. IPsec then adds a new outer IP header for underlay forwarding.

```text
+--------------------+---------------+----------------------------------------+------------+
| New outer IP header| ESP header    | Encrypted:                             | ESP auth   |
| Spoke1 -> Spoke2   | SPI, Seq      | Original GRE IP header + GRE header + | tag / ICV  |
| Protocol 50        |               | original IP packet + ESP trailer      |            |
+--------------------+---------------+----------------------------------------+------------+
```

The original GRE outer IP header is now **inside the encrypted ESP payload**. The new outer IP header remains visible to the underlay. In this example, the peer addresses can be the same in both headers, but they are distinct headers.

| Characteristic | ESP transport mode | ESP tunnel mode |
| --- | --- | --- |
| Protects the GRE header | Yes | Yes |
| Protects the original passenger IP packet | Yes | Yes |
| Encrypts the GRE packet's original outer IP header | No | Yes |
| Adds a separate outer IP header for ESP | No | Yes |
| Typically used for DMVPN GRE-over-IPsec | **Yes** | Possible, with additional overhead |

**Transport mode does not mean that the user's original IP packet is exposed.** The original packet is already inside GRE and is encrypted as part of the GRE payload protected by ESP.

---

## IPsec Transform Sets

An **IPsec transform set** defines the algorithms used by ESP to protect data and the ESP encapsulation mode. Its parameters are negotiated as part of the **IPsec Child SA**, rather than the IKE SA.

### AES Encryption and HMAC Integrity

A transform set can use **separate encryption and integrity algorithms**:

```cisco
crypto ipsec transform-set DMVPN-ESP esp-aes 256 esp-sha256-hmac
 mode transport
```

| Command | Purpose |
| --- | --- |
| `esp-aes 256` | Encrypts ESP data using AES with a 256-bit key. |
| `esp-sha256-hmac` | Provides ESP integrity/authentication using HMAC-SHA-256. |
| `mode transport` | Uses ESP transport mode rather than ESP tunnel mode. |

The two peers must negotiate **compatible ESP transforms and the same applicable encapsulation mode**. Matching IKEv2 proposals alone is not sufficient: the IKE SA can be established even when the IPsec Child SA negotiation fails.

### AES-GCM Authenticated Encryption

AES-GCM combines **encryption and integrity/authentication** into one algorithm:

```cisco
crypto ipsec transform-set DMVPN-GCM esp-gcm 256
 mode transport
```

`esp-gcm 256` specifies **AES-GCM with a 256-bit key**. Do **not** append a separate `esp-sha256-hmac` transform to the same AES-GCM transform set; GCM is an **AEAD (Authenticated Encryption with Associated Data)** algorithm.

The distinction between the two examples is:

| Transform set | Confidentiality | Integrity/authentication |
| --- | --- | --- |
| `esp-aes 256 esp-sha256-hmac` | AES-CBC encryption | Separate HMAC-SHA-256 |
| `esp-gcm 256` | AES-GCM encryption | Built into AES-GCM |

**IKEv2 proposal algorithms and ESP transform-set algorithms are selected independently.** For example, the IKE SA can use one encryption algorithm while the Child SA uses AES-GCM.

### Transport Mode Is Explicit

The default mode for a Cisco IPsec transform set is **tunnel mode**. Use `mode transport` explicitly for a transport-mode DMVPN design.

```cisco
crypto ipsec transform-set DMVPN-ESP esp-aes 256 esp-sha256-hmac
 mode transport
```

Changing a transform set does not retroactively change the parameters of an already established Child SA. New settings are used when the relevant IPsec SAs are negotiated or rekeyed.

---

## IPsec Profiles

An **IPsec profile** associates the data-plane protection settings with the IKEv2 peer authentication configuration. It can also control aspects of the negotiated IPsec SA, such as lifetimes and Perfect Forward Secrecy (PFS).

```cisco
crypto ipsec profile DMVPN-IPSEC
 set transform-set DMVPN-ESP
 set ikev2-profile DMVPN-IKEV2
```

| Command | Purpose |
| --- | --- |
| `crypto ipsec profile DMVPN-IPSEC` | Creates an IPsec profile. |
| `set transform-set DMVPN-ESP` | References the IPsec transform set used for ESP protection. |
| `set ikev2-profile DMVPN-IKEV2` | References the IKEv2 profile used for peer matching and authentication. |

The IKEv2 profile referenced here contains the peer identity matching, PSK authentication settings, and keyring reference. The **IKEv2 policy is not directly referenced by the IPsec profile**; its selection occurs separately as part of IKEv2 negotiation.

### Configuration Object Relationships

```text
    IKEv2 proposal <--------- IKEv2 policy
    (IKE SA algorithms)        (IKE proposal selection)

    IKEv2 keyring <---------- IKEv2 profile
    (preshared keys)           (peer matching and authentication)
                                     ^
                                     |
    IPsec transform set <----- IPsec profile
    (ESP algorithms/mode)      (IKEv2 profile, transforms,
                               PFS, Child SA lifetime)
                                     ^
                                     |
                               mGRE Tunnel0
                      (tunnel protection ipsec profile)
```

A **transform set** and **IPsec profile** are configuration objects. They do not themselves indicate that an ESP SA is up. The actual inbound and outbound ESP SAs are negotiated dynamically with individual peers.

### Multiple Transform Sets

An IPsec profile can list multiple acceptable transform sets, in preference order:

```cisco
crypto ipsec profile DMVPN-IPSEC
 set transform-set DMVPN-GCM DMVPN-ESP
 set ikev2-profile DMVPN-IKEV2
```

The peers negotiate a compatible set of ESP algorithms and mode. The ordering expresses local preference; the negotiated result must also be supported by the other peer.

For a simple DMVPN lab, **one consistent transform set on all hubs and spokes** makes mismatches easier to identify.

---

## IPsec Child SA Lifetimes and Perfect Forward Secrecy

IKE SAs and IPsec Child SAs have **separate lifetimes and rekey operations**. This section concerns the **Child SA**, which protects GRE data.

### Child SA Lifetime

An IPsec profile can override the Child SA lifetime:

```cisco
crypto ipsec profile DMVPN-IPSEC
 set security-association lifetime seconds 3600
```

The lifetime can also be controlled by a **traffic-volume limit**:

```cisco
crypto ipsec profile DMVPN-IPSEC
 set security-association lifetime kilobytes 4608000
```

**3600 seconds and 4,608,000 KB** are commonly documented Cisco IOS XE default IPsec SA lifetime values. Rekeying can occur before expiration so that traffic can continue using a replacement SA.

Changing a Child SA lifetime does **not** change the IKE SA lifetime. Do not confuse IPsec SA lifetime settings with the IKEv2 policy or IKEv2 profile.

### Perfect Forward Secrecy (PFS)

**PFS** uses a fresh Diffie-Hellman exchange when creating or rekeying applicable Child SAs, rather than relying solely on keying material inherited from an existing IKE SA.

```cisco
crypto ipsec profile DMVPN-IPSEC
 set pfs group14
```

`group14` selects the **2048-bit MODP Diffie-Hellman group** for PFS.

- The initial Child SA established during **IKE_AUTH** normally derives keys from the initial IKEv2 exchange; it does **not** require a separate CREATE_CHILD_SA exchange.
- Later **CREATE_CHILD_SA** exchanges can perform fresh Diffie-Hellman key exchange for new or rekeyed Child SAs when PFS is negotiated/required.
- PFS settings should be compatible between peers for the relevant Child SA negotiations.

**The IKEv2 proposal's DH group and the IPsec profile's PFS group are separate settings.** The former participates in IKE SA establishment; the latter concerns fresh DH key exchange for applicable Child SAs.

---

## Applying Tunnel Protection to DMVPN

The **tunnel interface** applies the IPsec profile using `tunnel protection ipsec profile`:

```cisco
interface Tunnel0
 tunnel source Ethernet0/0
 tunnel mode gre multipoint
 tunnel protection ipsec profile DMVPN-IPSEC
```

`DMVPN-IPSEC` must be the name of a previously configured **IPsec profile**; it is not the name of an IKEv2 profile or transform set.

The important distinction is:

- **`tunnel mode gre multipoint`** enables the multipoint GRE tunnel.
- **`tunnel protection ipsec profile DMVPN-IPSEC`** applies IPsec protection to GRE traffic on that tunnel.

There is **no fixed `tunnel destination`** on an mGRE interface. NHRP provides remote NBMA addresses as necessary, and IPsec can negotiate SAs with the corresponding peers.

### Why a Static Crypto Map Is Not Needed

Traditional policy-based IPsec commonly uses **crypto maps** and crypto ACLs to select which traffic to protect and which peer to use. DMVPN instead uses a **GRE tunnel with an attached IPsec profile**:

```text
          Tunnel0 (mGRE)
                 |
                 +---- NHRP --> Hub1 NBMA --> IPsec SA to Hub1
                 |
                 +---- NHRP --> Hub2 NBMA --> IPsec SA to Hub2
                 |
                 +---- NHRP --> Spoke2 NBMA --> IPsec SA to Spoke2
```

A single tunnel interface can therefore have **multiple peer-specific IPsec SAs**, even though it references just **one IPsec profile**.

With tunnel protection, IPsec protects the **GRE-encapsulated packets**, not independently selected LAN-to-LAN flows based on a user-configured crypto ACL. The IPsec selector is associated with the GRE traffic between the tunnel endpoints (IP protocol 47), rather than simply the original hosts' LAN subnets.

### Dynamic Spoke-to-Spoke ESP SAs

In DMVPN Phase 3, a typical protected spoke-to-spoke sequence is:

1. **Routing** initially sends traffic toward a hub.
2. **NHRP** discovers a direct shortcut to a remote spoke, including its NBMA address.
3. When traffic is sent to the newly selected peer, **IKEv2** establishes the required IKE/Child SA state if suitable state does not already exist.
4. **ESP** protects GRE traffic between the two spoke NBMA addresses.
5. Further packets can traverse the direct protected spoke-to-spoke path while the shortcut and security state remain valid.

The **NHRP mapping**, **IKE SA**, **Child SA**, and **overlay route** are different pieces of state with different purposes and lifetimes. They need not appear or expire simultaneously.

---

## Shared Tunnel Protection and Dual-Cloud DMVPN

A dual-cloud DMVPN design can have **two mGRE tunnel interfaces using the same WAN source interface**. This introduces a special case: multiple GRE tunnels may carry traffic between the **same local and remote NBMA addresses**, distinguished only by their GRE tunnel keys.

Without appropriate SA sharing, IPsec may have difficulty assigning received protected GRE traffic to the correct tunnel's IPsec security association database (SADB). Cisco provides the **`shared`** keyword for this scenario.

```cisco
interface Tunnel0
 tunnel source Ethernet0/0
 tunnel mode gre multipoint
 tunnel protection ipsec profile DMVPN-IPSEC shared
!
interface Tunnel1
 tunnel source Ethernet0/0
 tunnel mode gre multipoint
 tunnel protection ipsec profile DMVPN-IPSEC shared
```

These are **tunnel protection excerpts only**, not a complete dual-cloud configuration. Each tunnel also requires its own overlay addressing, NHRP settings, and appropriate GRE tunnel key.

The `shared` keyword allows the tunnel interfaces using the **same tunnel source and IPsec profile** to share an IPsec **SADB**, so applicable GRE sessions to the same peer can use the same IPsec SA set rather than establishing conflicting SAs.

Important constraints for this shared design:

- Configure `tunnel source` as an **interface**, not a literal source IP address.
- Use the **same IPsec profile** and `shared` keyword consistently on the tunnel interfaces in the sharing group.
- `shared` refers to **sharing IPsec security state**, not combining the two DMVPN clouds, tunnel subnets, NHRP domains, or routing tables.
- The feature is relevant when the same underlay source participates in multiple GRE/mGRE tunnels; it is **not required solely because a DMVPN has two hubs**.

For a **single-cloud dual-hub** design using one `Tunnel0`, ordinary `tunnel protection ipsec profile DMVPN-IPSEC` is sufficient. The more complex SA-sharing considerations arise when **multiple tunnel interfaces** coexist.

---

## Encapsulation Overhead, MTU, and NAT Traversal

GRE plus ESP adds overhead beyond the original IP packet. The exact size depends on the IP version, ESP transport/tunnel mode, encryption algorithm, padding, and whether NAT traversal is in use.

### MTU and MSS

```text
Original IP packet
     + GRE / outer IP encapsulation
     + ESP header, trailer, cryptographic overhead
     + optional NAT-T UDP encapsulation
     = larger packet on the underlay
```

A packet that fits the original LAN MTU might exceed the WAN path MTU after encapsulation. Symptoms include **small pings succeeding while larger packets fail**, IP fragmentation, PMTUD issues, or TCP sessions that stall for larger transfers.

An illustrative tunnel configuration is:

```cisco
interface Tunnel0
 ip mtu 1400
 ip tcp adjust-mss 1360
```

- `ip mtu` limits the **IP packet size carried over the tunnel**.
- `ip tcp adjust-mss` adjusts TCP MSS advertised in **TCP SYN** packets traversing the interface, helping avoid excessive TCP packet sizes.

**1400/1360 are illustrative values, not mandatory DMVPN defaults.** The appropriate values depend on the actual path and encapsulation overhead. With a standard 20-byte IPv4 header and 20-byte TCP header, an IP MTU of 1400 corresponds to a TCP MSS of 1360 for IPv4 TCP without options.

### NAT Traversal (NAT-T)

When NAT traversal is required, IKEv2 and ESP can use **UDP encapsulation on port 4500**:

| Traffic | On-the-wire protocol |
| --- | --- |
| Initial IKEv2 negotiation | UDP/500 (commonly) |
| IKEv2 after NAT-T port switching | UDP/4500 |
| UDP-encapsulated ESP (NAT-T) | UDP/4500 |
| Native ESP without NAT-T | IP protocol 50 |
| Unprotected GRE, for comparison | IP protocol 47 |

NAT-T adds UDP encapsulation overhead. In protected DMVPN, the underlay normally sees **ESP or UDP-encapsulated ESP**, rather than bare GRE traffic. IKEv2 remains a separate control-plane protocol.

---

## ESP Verification Essentials

The detailed troubleshooting workflow appears in [IKEv2/IPsec Verification and Troubleshooting](troubleshooting.md). At this stage, distinguish **IKE control-plane state** from **ESP data-plane state**.

```cisco
show crypto ikev2 sa
show crypto ipsec sa
show crypto ipsec transform-set
show dmvpn
show ip nhrp
```

`show crypto ipsec sa` is especially important. Inspect:

| Output | Meaning |
| --- | --- |
| **Local/remote crypto endpoints** | NBMA addresses of the protected IPsec peers. |
| **Inbound ESP SA / outbound ESP SA** | The unidirectional ESP SAs used by each traffic direction. |
| **SPI values** | Identify the negotiated ESP SAs. |
| **`#pkts encaps` / `#pkts encrypt`** | Outbound packets being protected. |
| **`#pkts decaps` / `#pkts decrypt`** | Inbound packets being processed successfully. |
| **Transform and mode** | Actual negotiated ESP protection and transport/tunnel mode. |
| **Replay/drop/error counters** | Potential anti-replay, integrity, or other IPsec processing failures. |

An **established IKEv2 SA** does not by itself prove that the Child SAs are established or that user traffic is passing. Similarly, an **established Child SA** does not prove that the NHRP shortcut, overlay routing, or end-to-end applications work correctly.

When testing a path, generate traffic and check **both directions' ESP counters**. Outbound encryption increasing without corresponding inbound decryption can indicate a path, peer, or return-traffic problem.

---

## Key Points

- **IKEv2 negotiates** security associations and keys; **ESP protects** the GRE data packets.
- **ESP uses IP protocol 50** when not encapsulated in UDP for NAT traversal.
- Each **Child SA** normally consists of **two unidirectional ESP SAs**, each with its own **32-bit SPI**.
- **ESP transport mode** encrypts the GRE header and original passenger packet while retaining the original outer GRE IP header.
- **ESP tunnel mode** encrypts the complete original GRE IP packet and adds a new outer IP header.
- An **IPsec transform set** selects ESP algorithms and transport/tunnel mode; an **IPsec profile** references the transform set and IKEv2 profile.
- `tunnel protection ipsec profile` attaches the IPsec profile to the mGRE interface; it does **not** select the GRE tunnel mode.
- One **mGRE tunnel interface** can dynamically establish **peer-specific IPsec SAs** to multiple hub and spoke NBMA addresses.
- The **`shared`** keyword allows suitable multiple tunnel interfaces with the same tunnel source to share an IPsec SADB; it does not merge DMVPN clouds.
- **PFS, Child SA lifetimes, anti-replay, MTU/MSS, and NAT-T** are important when configuring or troubleshooting protected DMVPN traffic.

---

## References

- [Cisco: Dynamic Multipoint VPN](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/security-vpn/security-vpn/m_sec-conn-dmvpn-dmvpn-0.html)
- [Cisco: Configuring Security for VPNs with IPsec](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/security-vpn/security-vpn/m_sec-cfg-vpn-ipsec-0.html)
- [Cisco: Sharing IPsec with Tunnel Protection](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/sec_conn_dmvpn/configuration/xe-16/sec-conn-dmvpn-xe-16-book/sec-conn-dmvpn-share-ipsec-w-tun-protect.html)
- [Cisco: Troubleshoot DMVPN Issues](https://www.cisco.com/c/en/us/support/docs/security/dynamic-multipoint-vpn-dmvpn/111976-dmvpn-troubleshoot-00.html)
- [RFC 4303: IP Encapsulating Security Payload (ESP)](https://www.rfc-editor.org/rfc/rfc4303.html)
