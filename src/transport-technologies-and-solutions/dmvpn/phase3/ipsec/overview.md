# 3.3.a (ii) IPsec/IKEv2 Using Preshared Key

**DMVPN** uses **GRE/mGRE** to encapsulate traffic and **NHRP** to discover tunnel endpoints. Neither GRE nor NHRP encrypts the traffic carried across the underlay. **IPsec** can be added to protect the GRE packets, while **IKEv2** authenticates peers and negotiates the security associations (SAs) required by IPsec.

This section focuses on **DMVPN Phase 3 using IKEv2 with preshared-key (PSK) authentication**, as specified in the CCIE Enterprise Infrastructure blueprint. The goal is to understand the relevant configuration objects, how they relate to each other, and how they protect hub-to-spoke and dynamically established spoke-to-spoke traffic.

---

## How IKEv2 and IPsec Fit into DMVPN

DMVPN's tunneling, endpoint discovery, and security mechanisms have separate responsibilities:

| Component | Responsibility |
| --- | --- |
| **GRE/mGRE** | Encapsulates overlay traffic and allows one tunnel interface to communicate with multiple peers. |
| **NHRP** | Resolves tunnel destinations to underlay (NBMA) addresses and enables Phase 3 spoke-to-spoke shortcuts. |
| **IKEv2** | Negotiates cryptographic parameters, authenticates peers using preshared keys, and establishes keying material and SAs. |
| **IPsec ESP** | Encrypts and provides integrity protection for GRE traffic sent between IPsec peers. |

```text
            DMVPN forwarding                     DMVPN security

    Overlay routing + NHRP determine       IKEv2 authenticates the peer
    which remote NBMA address to use       and negotiates IPsec protection
                    |                                    |
                    v                                    v
                 GRE packet --------------------> IPsec ESP
                                                       |
                                                       v
                                             Protected packet sent
                                                over the underlay
```

**IKEv2 is the key-management protocol; ESP protects the actual GRE packets.**

---

## Configuration Objects at a Glance

An IKEv2/IPsec configuration uses several different objects. Each object has a specific job, and some explicitly reference others.

| Configuration object | IOS XE configuration command | Purpose |
| --- | --- | --- |
| **IKEv2 proposal** | `crypto ikev2 proposal` | Defines acceptable algorithms for protecting the **IKE SA**, including encryption, integrity, PRF, and Diffie-Hellman (DH) parameters. |
| **IKEv2 policy** | `crypto ikev2 policy` | Selects one or more IKEv2 proposals for negotiation, optionally using criteria such as the local address or front-door VRF. |
| **IKEv2 keyring** | `crypto ikev2 keyring` | Stores **preshared keys** and the peer entries used to select them. |
| **IKEv2 profile** | `crypto ikev2 profile` | Defines peer identity matching, local and remote authentication methods, and the keyring used for PSK authentication. |
| **IPsec transform set** | `crypto ipsec transform-set` | Defines how **ESP** protects data traffic, including the ESP algorithm(s) and transport or tunnel mode. |
| **IPsec profile** | `crypto ipsec profile` | Associates an IPsec transform set with an IKEv2 profile and can specify additional IPsec SA parameters. |
| **DMVPN tunnel interface** | `interface Tunnel0` | Uses `tunnel protection ipsec profile` to apply the IPsec profile to GRE traffic sent through the tunnel. |

The key distinction is between **IKEv2 settings**, which protect the negotiation and authenticate peers, and **IPsec settings**, which protect the GRE traffic after negotiation.

### How the Objects Reference Each Other

```text
       IKEv2 proposal <------- IKEv2 policy
          (IKE SA             (selects acceptable
          algorithms)          IKE SA algorithms)

       IKEv2 keyring <-------- IKEv2 profile
           (PSKs)              (peer matching and
                                authentication)
                                      ^
                                      |
                                      |
       IPsec transform set <--- IPsec profile
          (ESP algorithms        (combines IKEv2
           and mode)             and IPsec settings)
                                      ^
                                      |
                               Tunnel0 interface
                      (tunnel protection ipsec profile)
```

The arrows show **configuration references**, not packet flow:

- An **IKEv2 policy** references an **IKEv2 proposal**.
- An **IKEv2 profile** references an **IKEv2 keyring**.
- An **IPsec profile** references both an **IKEv2 profile** and an **IPsec transform set**.
- The **DMVPN tunnel interface** references the **IPsec profile**.

**There is no direct configuration reference from the IKEv2 profile to the IKEv2 policy.** IKEv2 policy selection and IKEv2 profile selection are separate parts of the negotiation.

The following excerpt shows only the commands that connect these objects. It is **not a complete configuration**:

```cisco
crypto ikev2 policy IKE-POLICY
 proposal IKE-PROPOSAL
!
crypto ikev2 profile IKE-PROFILE
 keyring local IKE-KEYRING
!
crypto ipsec profile DMVPN-IPSEC
 set transform-set ESP-SET
 set ikev2-profile IKE-PROFILE
!
interface Tunnel0
 tunnel protection ipsec profile DMVPN-IPSEC
```

The objects themselves must also be defined with suitable cryptographic parameters, peer matching, authentication, and PSKs. The complete configurations are covered in [IKEv2/IPsec Configuration for DMVPN](configuration.md).

---

## IKE SAs and IPsec SAs

The configuration objects above are **templates and settings**. They are not the same as the **security associations (SAs)** negotiated between peers.

An SA contains the parameters and keys used for an active security relationship. With IKEv2, two related types are important:

| SA | Purpose |
| --- | --- |
| **IKE SA** | Provides a protected channel for IKEv2 control-plane exchanges, including authentication and management of Child SAs. |
| **Child SA (IPsec SA)** | Provides the negotiated ESP protection for the actual GRE traffic. IPsec SAs are unidirectional, so bidirectional traffic uses an inbound and an outbound SA. |

At a high level, the peers:

1. Exchange IKEv2 proposals and establish cryptographic parameters for the IKE SA.
2. Authenticate each other using their configured **PSKs**.
3. Negotiate a **Child SA** to protect GRE traffic with ESP.
4. Use the negotiated IPsec SAs to protect traffic in both directions, renewing SAs when necessary.

The preshared key is used for **peer authentication**. ESP data traffic is protected using session keys derived through the IKEv2 negotiation, not by directly encrypting packets with the configured PSK.

See [IKEv2 Negotiation and Preshared Key Authentication](ikev2.md) for the IKE_SA_INIT and IKE_AUTH exchanges and the distinction between IKE SAs and Child SAs.

---

## What IPsec Protects

Without IPsec, a GRE packet sent across an IPv4 underlay contains an outer IPv4 header, a GRE header, and the original packet:

```text
Without IPsec:

[ Outer IPv4 ][ GRE ][ Original IP packet ]
```

With **ESP transport mode**, ESP protects the GRE header and its encapsulated packet while retaining the original outer IPv4 header:

```text
With ESP transport mode:

[ Outer IPv4 ][ ESP header ][ Encrypted: GRE + original packet + ESP trailer ][ ESP auth tag ]
```

The underlay still sees the outer source and destination IP addresses so that it can forward the packet. It cannot read the protected GRE payload.

**ESP tunnel mode** is also possible. It encapsulates and protects the entire GRE IP packet inside an additional outer IP header. Transport mode avoids this extra outer header and is commonly used for GRE-over-IPsec DMVPN.

The **IPsec transform set** determines the ESP algorithms and mode. The **IKEv2 proposal** determines the algorithms for the IKE SA instead. These are separate negotiations and must not be confused.

For IPv4 underlays, IKEv2 commonly uses UDP port **500**, ESP uses IP protocol **50**, and NAT traversal can encapsulate IPsec traffic in UDP port **4500**. These are underlay protocol considerations, separate from the GRE tunnel IP addresses.

See [IPsec ESP and Tunnel Protection](esp.md) for encapsulation, transform sets, modes, and tunnel protection.

---

## Dynamic IPsec Protection in DMVPN

A DMVPN router can communicate with multiple remote NBMA addresses using one **mGRE tunnel interface**. Applying an IPsec profile to that interface does **not** mean every remote peer shares a single IPsec SA.

Instead, IPsec protection is negotiated **between the actual communicating peers**:

- A spoke establishes IKEv2/IPsec security associations with a hub to protect hub-to-spoke traffic.
- In Phase 3, NHRP can discover a direct path to another spoke.
- If Spoke1 begins communicating directly with Spoke2, they can establish their **own IKEv2/IPsec SAs**, without routing the protected data through the hub.
- The same tunnel interface and IPsec profile can be used to protect traffic to multiple dynamically discovered peers.

```text
                         Hub1
                        /    \
              IKEv2/IPsec    IKEv2/IPsec
                      /        \
                  Spoke1      Spoke2
                      \        /
                       \______/
                       IKEv2/IPsec
                     Direct spoke path
```

The diagram represents **separate peer-to-peer security relationships**, not separate tunnel interfaces on each spoke.

In the dual-hub design, security associations to Hub1 and Hub2 are also distinct. Working IKEv2/IPsec connectivity to one hub does not prove that protection or reachability to the other hub is working.

---

## Different Keys and Authentication Mechanisms

DMVPN configurations may contain several commands that appear to provide authentication or use the word *key*. They have different purposes:

| Configuration | Meaning |
| --- | --- |
| `ip nhrp authentication` | An NHRP authentication string used to validate participation in the NHRP network. It does **not** provide cryptographic protection for the tunnel. |
| `tunnel key` | A GRE key field used to identify or distinguish GRE tunnels. It is **not an encryption key**. |
| IKEv2 `pre-shared-key` | A cryptographic secret used by IKEv2 to authenticate IPsec peers. |

**An NHRP authentication string and an IKEv2 preshared key are not interchangeable.** Configuring NHRP authentication alone does not encrypt DMVPN traffic.

---

## Verification Approach

When IPsec protection is added to an existing DMVPN tunnel, troubleshoot the functions separately:

1. **Underlay:** Verify IP reachability between the relevant NBMA addresses.
2. **IKEv2:** Verify that the peers authenticate and establish an IKE SA with `show crypto ikev2 sa`.
3. **IPsec:** Verify Child SAs and encrypt/decrypt packet counters with `show crypto ipsec sa`.
4. **DMVPN:** Verify NHRP registrations, spoke-to-spoke shortcuts, and overlay routing independently.

A successfully negotiated **IKE SA does not, by itself, prove that ESP is passing traffic**. Likewise, a working NHRP mapping does not prove that the corresponding peer-to-peer IPsec SAs are established.

The [IKEv2/IPsec Verification and Troubleshooting](troubleshooting.md) page covers these checks in detail.

---

## Key Points

- **IKEv2 negotiates and authenticates; IPsec ESP protects GRE traffic.** Neither replaces mGRE, NHRP, or overlay routing.
- **IKEv2 proposal/policy:** Define and select the algorithms used for the IKE SA.
- **IKEv2 keyring/profile:** Store PSKs, match peers, and define authentication.
- **IPsec transform set:** Defines ESP protection for data traffic, independently of the IKE SA algorithms.
- **IPsec profile:** References the IKEv2 profile and IPsec transform set; `tunnel protection ipsec profile` applies it to the DMVPN tunnel.
- **SAs are negotiated per peer relationship**, including dynamically discovered spoke-to-spoke connections, rather than being one shared SA for the entire multipoint tunnel.
- **IKE SA and Child SA are different:** The former protects IKEv2 exchanges; the latter protects GRE traffic with ESP.
- **NHRP authentication, GRE tunnel keys, and IKEv2 PSKs serve different purposes.**

---

## References

- [Cisco: Configuring Internet Key Exchange Version 2](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/security-vpn/security-vpn/m_sec-cfg-ikev2-flex.html)
- [Cisco: Configure BGP over DMVPN Phase 3](https://www.cisco.com/c/en/us/support/docs/security/dynamic-multipoint-vpn-dmvpn/222585-configure-bgp-over-dmvpn-phase-3.html)
- [Cisco: Configuring Security for VPNs with IPsec](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/security-vpn/security-vpn/m_sec-cfg-vpn-ipsec-0.html)
- [Cisco: Sharing IPsec with Tunnel Protection](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/security-vpn/security-vpn/m_sec-conn-dmvpn-share-ipsec-w-tun-protect-0.html)
