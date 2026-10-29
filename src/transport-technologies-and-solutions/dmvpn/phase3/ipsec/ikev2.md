# IKEv2 Negotiation and Preshared Key Authentication

**Internet Key Exchange version 2 (IKEv2)** is the control-plane protocol that authenticates IPsec peers and negotiates the cryptographic keys and **security associations (SAs)** used to protect traffic. In DMVPN, **mGRE and NHRP discover and forward to tunnel endpoints**, while **IKEv2 establishes the IPsec security needed to communicate with those endpoints**.

This page focuses on **IKEv2 using preshared keys (PSKs)** for DMVPN Phase 3.

It covers the exchanges, configuration objects, peer selection, and SA lifecycle. ESP encapsulation and transform sets are covered in [IPsec ESP and Tunnel Protection](esp.md); complete hub/spoke configurations are covered in [IKEv2/IPsec Configuration for DMVPN](configuration.md).

---

## IKEv2, IPsec, and DMVPN

IKEv2 does not carry the GRE-encapsulated user traffic. It negotiates and maintains the SAs used by **IPsec ESP**, which protects that traffic across the underlay.

| Component | Responsibility |
| --- | --- |
| **mGRE** | Encapsulates overlay traffic and permits multiple remote tunnel endpoints on one interface. |
| **NHRP** | Learns and resolves overlay tunnel addresses to remote NBMA addresses. |
| **IKEv2** | Negotiates cryptographic parameters, authenticates peers, and manages SAs and keys. |
| **IPsec ESP** | Encrypts and authenticates the protected GRE traffic using negotiated IPsec SAs. |

For example, **Spoke1** may resolve **Spoke2's** NBMA address using NHRP. That resolution does not itself authenticate Spoke2 as an IKEv2 peer or establish encryption keys. When protected traffic needs to flow to Spoke2, IKEv2 can negotiate security directly between the two spokes.

```text
            Control-plane functions                Data-plane functions

       NHRP: discover remote NBMA address
                     |
                     v
       IKEv2: authenticate remote peer
              and establish SAs
                     |
                     v
       GRE: encapsulate overlay packet
                     |
                     v
       ESP: protect GRE packet using IPsec SA
                     |
                     v
       Underlay: forward toward remote NBMA address
```

**An established NHRP mapping does not prove that IKEv2 or IPsec is working.** Conversely, an IKEv2 SA by itself does not prove that overlay routing, NHRP shortcuts, or data-plane forwarding is working.

---

## IKEv2 SA and IPsec Child SAs

IKEv2 establishes two different kinds of security association:

| SA | Role | Protection |
| --- | --- | --- |
| **IKEv2 SA (IKE SA)** | Secure control channel between two IKEv2 peers. | Protects subsequent IKEv2 messages, including negotiation and maintenance of Child SAs. |
| **IPsec SA (Child SA)** | Security association for protected data traffic. | Provides the keys and algorithms used by ESP. |

An **IKEv2 SA** is established between a particular pair of IKEv2 peers. It can manage **one or more Child SAs**. Each IPsec Child SA represents **a pair of unidirectional IPsec SAs**, one for traffic in each direction.

```text
                    Spoke1 <---------> Spoke2

                      IKEv2 SA
               (protected IKE control channel)
                            |
                 +----------+----------+
                 |                     |
             Child SA 1             Child SA 2
          (ESP traffic pair)     (another traffic pair,
                                  if required)
                 |
       +---------+---------+
       |                   |
  Spoke1 -> Spoke2    Spoke2 -> Spoke1
   outbound ESP        inbound ESP
```

**Security associations are negotiated state, not the same thing as configuration profiles.** An `IKEv2 profile` is a configuration object; an **IKEv2 SA** is a negotiated relationship visible in operational output. Likewise, an `IPsec profile` is a configuration object; a **Child SA** consists of negotiated IPsec state.

### Security Parameter Index (SPI)

An **SPI** identifies a security association:

- The **IKEv2 SA** uses two eight-byte SPIs, one chosen by each endpoint, in IKEv2 headers.
- Each **unidirectional ESP SA** has its own 32-bit SPI. The receiver uses the ESP SPI, together with other SA lookup context, to select the correct inbound SA.

When inspecting `show crypto ipsec sa`, do not expect the inbound and outbound ESP SPIs to be identical. They refer to separate unidirectional SAs.

---

## IKEv2 Negotiation Exchanges

IKEv2 normally creates an IKE SA and its **first Child SA with four messages**, organized into two request/response exchanges:

1. **IKE_SA_INIT:** Negotiate IKE SA cryptographic parameters and establish shared keying material through Diffie-Hellman.
2. **IKE_AUTH:** Exchange identities, authenticate the peers, and normally negotiate the first IPsec Child SA.

Two further exchange types operate **after** the initial negotiation:

3. **CREATE_CHILD_SA:** Create additional Child SAs or rekey existing Child SAs or the IKE SA.
4. **INFORMATIONAL:** Exchange status, errors, deletion requests, and other control information; it can be used to check peer liveness.

```text
             Initiator                             Responder
                 |                                     |
                 |----- IKE_SA_INIT Request ---------->|
                 |      SA proposals, DH value, nonce  |
                 |<---- IKE_SA_INIT Response ----------|
                 |     Selected proposal, DH, nonce    |
                 |                                     |
                 |    Shared IKE keying material       |
                 |          is now derived             |
                 |                                     |
                 |----- IKE_AUTH Request ------------->|
                 |     Identity, AUTH, Child SA offer  |
                 |<--- IKE_AUTH Response --------------|
                 |     Identity, AUTH, Child SA choice |
                 |                                     |
                 |        IKE SA established           |
                 |     First Child SA established      |
                 |                                     |
                 |<==== Protected GRE traffic (ESP) ==>| 
                 |                                     |
                 |---- CREATE_CHILD_SA (as needed) --->|
                 |<--- CREATE_CHILD_SA Response -------|
                 |                                     |
                 |----- INFORMATIONAL (as needed) ---->|
                 |<--- INFORMATIONAL Response ---------|
```

The initial exchanges use **IKEv2 over UDP port 500**, or **UDP port 4500** when UDP encapsulation/NAT traversal applies. These are IKEv2 control messages, not ESP user data.

### 1. IKE_SA_INIT: Negotiate and Derive IKE Keys

The peers exchange:

- **SA proposals:** Supported encryption, integrity, pseudorandom function (PRF), and Diffie-Hellman (DH) algorithms for the IKE SA.
- **DH public values:** Material used to independently compute a shared secret without sending the secret across the network.
- **Nonces (`Ni` and `Nr`):** Fresh random values used in key derivation and to prevent reuse of old negotiation material.
- **Optional notifications:** For example, NAT-detection information.

The responder selects mutually acceptable cryptographic parameters. Each side derives the IKE SA's keying material from the DH shared secret and exchanged nonces, using the negotiated PRF.

**IKE_SA_INIT does not authenticate the peers.** At this point, they have negotiated algorithms and established keying material, but have not yet proved their identities. Authentication occurs in IKE_AUTH.

### 2. IKE_AUTH: Authenticate the Peers and Create the First Child SA

The peers exchange **identities (`IDi`, `IDr`)** and **authentication payloads (`AUTH`)**. With PSK authentication, each peer demonstrates possession of the appropriate preshared key through the AUTH calculation. **The PSK itself is not transmitted.**

The IKE_AUTH messages are **encrypted and integrity-protected using keys derived during IKE_SA_INIT**. In the normal case, these messages also negotiate the first Child SA's ESP parameters and traffic selectors.

The successful result is:

- An **authenticated IKEv2 SA** for subsequent IKE control traffic.
- The **first IPsec Child SA** for protecting GRE traffic.

A failure to create a Child SA does not necessarily mean the IKE SA failed: the IKE SA may be established while the initial Child SA negotiation fails. This distinction matters when troubleshooting.

### 3. CREATE_CHILD_SA: New SAs and Rekeying

The `CREATE_CHILD_SA` exchange is used when a peer needs to:

- Create an **additional Child SA** under an established IKE SA.
- **Rekey an existing Child SA**, replacing the IPsec keys and SPIs.
- **Rekey the IKE SA**, replacing its keying material.

A new Child SA negotiated with `CREATE_CHILD_SA` can include a fresh DH exchange to provide fresh DH material for that Child SA's keys. **A new DH exchange is not automatically performed for every Child SA**, and PFS behavior depends on the negotiated settings.

**The first Child SA is normally negotiated in IKE_AUTH, not in a separate CREATE_CHILD_SA exchange.**

### 4. INFORMATIONAL: Maintenance and Liveness

`INFORMATIONAL` exchanges carry protected notifications and administrative messages, including deletion of SAs and error reporting. They can also be used for **Dead Peer Detection (DPD)**, which helps identify unreachable peers.

IKEv2 request/response exchanges provide retransmission and sequencing. A loss of IKE control connectivity can eventually result in cleanup of the IKE SA and its Child SAs. A tunnel or overlay route might therefore remain configured even though the required IPsec state is missing.

---

## IKEv2 Configuration Objects

For IOS XE DMVPN using PSKs, the principal IKEv2 objects are the **proposal**, **policy**, **keyring**, and **profile**. The IPsec profile and transform set connect IKEv2 to ESP and the tunnel interface.

| Object | Configuration mode | Purpose |
| --- | --- | --- |
| **IKEv2 proposal** | `crypto ikev2 proposal` | Defines acceptable algorithms for protecting the IKEv2 SA. |
| **IKEv2 policy** | `crypto ikev2 policy` | Selects which IKEv2 proposal(s) apply to negotiation, based on the local address and FVRF. |
| **IKEv2 keyring** | `crypto ikev2 keyring` | Stores the PSKs associated with peers or peer groups. |
| **IKEv2 profile** | `crypto ikev2 profile` | Defines peer identity matching, local/remote authentication methods, and the keyring to use. |
| **IPsec transform set** | `crypto ipsec transform-set` | Defines ESP algorithms and the IPsec mode used for data protection. |
| **IPsec profile** | `crypto ipsec profile` | References the IPsec transform set and IKEv2 profile; it is applied to the DMVPN tunnel. |

These are **two related configuration paths**, not a single chain in which every object points to the next:

```text
IKE SA algorithm selection:

   IKEv2 proposal <------ IKEv2 policy
      encryption           references proposal
      integrity            selects using local address / FVRF
      PRF
      DH group

IKEv2 authentication and tunnel protection:

   IKEv2 keyring <------- IKEv2 profile
      peer / PSK           keyring local ...
                           remote identity match
                           local / remote auth methods
                                 ^
                                 |
                                 | set ikev2-profile ...
                                 |
   IPsec transform set <-- IPsec profile
      ESP settings          set transform-set ...
                                 ^
                                 |
                      tunnel protection ipsec profile ...
                                 |
                         DMVPN Tunnel interface
```

**The IKEv2 profile does not reference the IKEv2 policy.** The policy is selected independently during IKE_SA_INIT. An IPsec profile references the IKEv2 profile, which in turn references the keyring.

---

## IKEv2 Proposal: Cryptographic Negotiation

An **IKEv2 proposal** defines acceptable transform algorithms for the **IKE SA**, not the data-plane ESP SA. It contains four relevant categories:

| Transform | Purpose | Example |
| --- | --- | --- |
| **Encryption** | Confidentiality of subsequent IKEv2 exchanges. | AES-CBC-256 |
| **Integrity** | Detect modification of IKEv2 messages when using a non-AEAD cipher. | SHA-256 HMAC |
| **PRF** | Derive IKEv2 key material. | HMAC-SHA-256 |
| **DH group** | Specify the key-agreement group used to establish fresh shared keying material. | Group 14 or 19 |

An illustrative proposal is:

```cisco
crypto ikev2 proposal DMVPN-IKE
 encryption aes-cbc-256
 integrity sha256
 prf sha256
 group 14
```

This example allows **AES-256-CBC**, **SHA-256**, **HMAC-SHA-256 as the PRF**, and **DH group 14**. The `encryption`, `integrity`, and `prf` settings here protect and derive keys for the **IKEv2 SA**; they do **not** determine the ESP cipher for the Child SA.

### Proposal Matching and Algorithm Choice

An IKEv2 proposal can offer **multiple choices within a transform category**. For example:

```cisco
crypto ikev2 proposal DMVPN-IKE
 encryption aes-cbc-256 aes-cbc-128
 integrity sha256
 prf sha256
 group 14 19
```

The endpoints must negotiate a compatible **combination**. If they cannot agree, negotiation fails during **IKE_SA_INIT**, before PSK authentication begins.

### AES-GCM and the Integrity Transform

**AES-GCM** is an **authenticated-encryption (AEAD)** algorithm that provides encryption and integrity protection together. When AES-GCM is configured for IKEv2, the proposal does **not** specify a separate `integrity` transform, but it **does** specify a PRF:

```cisco
crypto ikev2 proposal DMVPN-IKE-GCM
 encryption aes-gcm-256
 prf sha256
 group 19
```

**The IKEv2 proposal and IPsec transform set are independent.** For example, IKEv2 might use AES-CBC-256 while the Child SA uses ESP AES-GCM. What matters is that the peers agree on the relevant algorithms for **each** SA.

---

## IKEv2 Policy: Selecting the IKE Proposal

The **IKEv2 policy** references one or more IKEv2 proposals, and selects them based on the local IKE negotiation context.

```cisco
crypto ikev2 policy DMVPN-POLICY
 proposal DMVPN-IKE
```

The `proposal DMVPN-IKE` command **references the IKEv2 proposal by name**. In this example, without explicit `match` statements, the policy matches any local address in the **global underlay routing table** (global FVRF).

A policy can optionally select according to:

- `match address local ...`: The **local IKEv2 endpoint address**, such as the router's WAN interface address.
- `match fvrf ...`: The **Front Door VRF (FVRF)** used for underlay IKEv2 reachability.

For example:

```cisco
crypto ikev2 policy DMVPN-WAN1
 match address local 198.51.100.9
 proposal DMVPN-IKE
```

This policy applies to IKEv2 negotiation using the specified **local** endpoint address. It does **not** match the remote peer address or the remote IKE identity.

A policy can reference **multiple proposals**, which are considered in their configured order. Avoid overlapping policies that match the same negotiation context.

**Policy selection happens during IKE_SA_INIT.** It does not authenticate the peer or select the PSK. Those functions involve the IKEv2 **profile and keyring** in the authentication stage.

### Front Door VRF versus Overlay VRF

The **FVRF** is the routing table used for reachability to the remote **NBMA/IKE endpoint**. In DMVPN this can correspond to a tunnel's transport VRF (`tunnel vrf`).

The **inside VRF (IVRF)**, where used, is the routing table associated with the protected/overlay traffic. It is a separate concept and should not be confused with FVRF matching in an IKEv2 policy or profile.

The following examples use the **global routing table for underlay reachability**, so an explicit FVRF match is not required.

---

## IKEv2 Keyring: Preshared Keys for Peers

An **IKEv2 keyring** stores the PSKs used for IKEv2 authentication. It consists of one or more named `peer` entries, each identifying a peer or peer group and its authentication keys.

### A Specific Peer

```cisco
crypto ikev2 keyring DMVPN-KEYS
 peer HUB1
  address 198.51.100.1
  pre-shared-key Pass123!
```

This entry associates a PSK with an IKEv2 peer reachable at the given **underlay (NBMA) address**, not at its DMVPN tunnel address (`172.16.0.1`).

### Dynamic Peers: Wildcard Keyring Entry

A DMVPN hub may need to accept IKEv2 negotiations from spokes whose NBMA addresses are not individually configured. A wildcard peer entry can be used:

```cisco
crypto ikev2 keyring DMVPN-KEYS
 peer DMVPN-PEERS
  address 0.0.0.0 0.0.0.0
  pre-shared-key Pass123!
```

The address and mask `0.0.0.0 0.0.0.0` match **any IPv4 IKE peer endpoint address** for this keyring entry. This makes a common PSK convenient for dynamic hub/spoke and spoke-to-spoke connectivity.

**Security tradeoff:** A shared wildcard PSK means anyone who knows that PSK and can reach the IKE service may attempt authentication as a peer matching the profile. Where practical, use more restrictive peer matching and per-peer keys or identities rather than distributing one shared secret to every site.

### Symmetric and Asymmetric PSKs

The simple `pre-shared-key ...` command defines a **symmetric PSK**, used as the local and remote authentication secret in that peer entry.

IKEv2 also supports **asymmetric** PSKs:

```cisco
crypto ikev2 keyring DMVPN-KEYS
 peer HUB1
  address 198.51.100.1
  pre-shared-key local LOCAL_SECRET
  pre-shared-key remote REMOTE_SECRET
```

- **`local`** is the secret this router uses to authenticate **itself** to that peer.
- **`remote`** is the secret this router uses to verify **the peer's** authentication.

If Spoke1 uses `LOCAL_SECRET` to authenticate itself to Hub1, Hub1 must use that same secret to verify Spoke1 (its `remote` key for Spoke1). The opposite direction must likewise match.

**A PSK is not a data-encryption key.** It is an authentication secret; IKEv2 derives negotiated traffic keys for the IKE and Child SAs. Changing a keyring entry does not directly replace an already established Child SA's ESP keys.

### Peer Address versus IKE Identity

Three values must be kept separate:

| Value | Meaning | Example |
| --- | --- | --- |
| **NBMA / IKE peer address** | Address used to reach the remote IKEv2 endpoint. | `198.51.100.1` |
| **IKE identity** | Identity presented in `IDi` or `IDr` during IKE_AUTH. | IP address, FQDN, or key ID |
| **DMVPN tunnel address** | Address used inside the GRE overlay. | `172.16.0.1` |

An IKE identity **can** be an IP address, but it is not inherently the same as the packet's outer source IP (particularly when NAT is present), and it is not automatically the DMVPN tunnel address.

A keyring peer entry can also contain an `identity ...` match to distinguish peers based on the IKE identity. IOS XE uses different key-lookup information depending on whether it is acting as **initiator or responder**: the initiator primarily looks up the configured peer hostname/address, while the responder can use the received IKE identity/address.

---

## IKEv2 Profile: Matching and Authenticating Peers

The **IKEv2 profile** defines which peers it is intended to serve and how those peers authenticate. Unlike an IKEv2 proposal, its authentication and identity settings are **not negotiated algorithm choices**.

Example using a common DMVPN PSK:

```cisco
crypto ikev2 profile DMVPN-IKEV2
 match identity remote address 0.0.0.0
 authentication remote pre-share
 authentication local pre-share
 keyring local DMVPN-KEYS
```

| Command | Purpose |
| --- | --- |
| `match identity remote address 0.0.0.0` | Match any remote **IPv4 address-type IKE identity**. |
| `authentication remote pre-share` | Require the remote peer to authenticate using a PSK. |
| `authentication local pre-share` | Authenticate this router to the remote peer using a PSK. |
| `keyring local DMVPN-KEYS` | Reference the local IKEv2 keyring containing PSKs. |

An IKEv2 profile normally requires a remote-identity matching criterion to be usable. In a DMVPN deployment, a wildcard is convenient for dynamically discovered peers, but the wildcard shown here matches **address-type identities**, not every possible identity type (such as all FQDNs).

### Local and Remote Authentication Are Directional

The profile defines two separate directions:

- **Local authentication** (`authentication local ...`): How this router proves its identity to its peer.
- **Remote authentication** (`authentication remote ...`): How this router expects the peer to prove its identity.

For PSK-based DMVPN, both are usually `pre-share`. The **local/remote perspective is always relative to the router on which the commands are configured**, not relative to a permanent hub/spoke initiator/responder role.

### Profile Matching versus Policy Matching

| Question | IKEv2 policy | IKEv2 profile |
| --- | --- | --- |
| **What does it select?** | IKE SA cryptographic proposal(s). | Identity-matching and authentication settings. |
| **Key criteria** | Local IKE address, FVRF. | Remote IKE identity; optionally local address and FVRF. |
| **References** | IKEv2 proposal(s). | IKEv2 keyring. |
| **When relevant?** | IKE_SA_INIT. | IKE_AUTH / authenticated IKE session setup. |

**The IKEv2 profile does not reference the IKEv2 policy.** Configuring `match identity remote ...` in a profile does not change which IKEv2 proposal the policy offers.

### Local Identity

The `identity local ...` command can explicitly choose the identity that this router sends during IKE_AUTH:

```cisco
crypto ikev2 profile DMVPN-IKEV2
 identity local fqdn spoke1.example.com
```

A chosen local identity must be compatible with the remote peer's profile matching and keyring lookup. When using IP-address identities, an address change or NAT can complicate a too-specific peer match; stable FQDN or key-ID identities can help in designs that require them.

---

## How the Objects Work Together

The following is a **representative IKEv2 configuration for a DMVPN spoke** whose underlay uses the global routing table. It is intentionally limited to IKEv2; the ESP transform set, IPsec profile, and GRE tunnel protection are configured on the next pages.

```cisco
! 1. Algorithms for the IKE SA
crypto ikev2 proposal DMVPN-IKE
 encryption aes-cbc-256
 integrity sha256
 prf sha256
 group 14
!
! 2. Select the proposal for IKEv2 negotiation
crypto ikev2 policy DMVPN-POLICY
 proposal DMVPN-IKE
!
! 3. Preshared key for dynamic IKEv2 peers
crypto ikev2 keyring DMVPN-KEYS
 peer DMVPN-PEERS
  address 0.0.0.0 0.0.0.0
  pre-shared-key Pass123!
!
! 4. Peer identity matching and authentication
crypto ikev2 profile DMVPN-IKEV2
 match identity remote address 0.0.0.0
 authentication remote pre-share
 authentication local pre-share
 keyring local DMVPN-KEYS
```

**How a new negotiation uses these objects:**

1. During **IKE_SA_INIT**, the router selects an applicable **IKEv2 policy** using the local IKE endpoint context; the policy supplies its **IKEv2 proposal** for negotiating the IKE SA's algorithms.
2. During **IKE_AUTH**, the **IKEv2 profile** determines permitted identity/authentication settings for the peer; the profile's **keyring** supplies the appropriate PSK.
3. Once authenticated, the peers have an **IKEv2 SA**, and normally an initial **Child SA** for ESP.
4. The **IPsec profile**, when attached to the DMVPN tunnel, associates the IKEv2 profile and ESP transform-set settings with protected tunnel traffic.

The object names (`DMVPN-IKE`, `DMVPN-POLICY`, etc.) are **local configuration identifiers**. The two routers do not need identical names, but they must agree on compatible cryptographic algorithms, identities/authentication, and secrets.

---

## IKEv2 in a DMVPN Phase 3 Spoke-to-Spoke Shortcut

Consider the existing single-cloud lab:

| Device | Tunnel IP | NBMA IP |
| --- | --- | --- |
| Hub1 | `172.16.0.1` | `198.51.100.1` |
| Spoke1 | `172.16.0.11` | `198.51.100.9` |
| Spoke2 | `172.16.0.12` | `198.51.100.13` |

Assume Spoke1 already has a **protected hub connection**, but no direct shortcut to Spoke2:

1. **Spoke1 forwards traffic through Hub1.** The Spoke1-to-Hub1 GRE packets are protected by an existing Spoke1/Hub1 IPsec SA pair.
2. **Hub1 sends an NHRP Redirect.** Spoke1 performs NHRP resolution and learns how to reach Spoke2 at NBMA address `198.51.100.13`.
3. **Direct protected traffic triggers negotiation if necessary.** Spoke1 and Spoke2 establish their own IKEv2 SA and IPsec Child SA if suitable SAs do not already exist.
4. **Subsequent data takes the direct path.** The Spoke1-to-Spoke2 GRE traffic is protected by their **spoke-to-spoke IPsec SAs**, rather than reusing the hub's IPsec SAs.

```text
Before shortcut:

Spoke1 ==== ESP ==== Hub1 ==== ESP ==== Spoke2
        separate SA       separate SA

After shortcut:

Spoke1 ============ ESP =============> Spoke2
              spoke-to-spoke SA
```

**NHRP does not distribute PSKs or negotiate ESP SAs.** NHRP supplies routing/endpoint-resolution information; IKEv2 independently authenticates each relevant IPsec peer and negotiates security. The initial packets that trigger a new SA may be delayed or lost while negotiation completes.

For a **dual-hub** deployment, each hub/spoke security relationship is likewise independent. A second NHS or a backup DMVPN route does **not** automatically imply a working IKEv2/ESP SA with the backup hub.

---


## SA Lifetimes, Rekeying, and Peer Liveness

IKEv2 and Child SAs have **separate lifecycles**. Replacing one does not necessarily replace the other at the same time.

| Mechanism | Effect |
| --- | --- |
| **IKE SA lifetime/rekey** | Refreshes keys protecting IKE control traffic. |
| **Child SA lifetime/rekey** | Refreshes ESP keys and SPIs used to protect data traffic. |
| **DPD** | Checks whether an IKEv2 peer remains responsive. |
| **SA deletion** | Removes obsolete or invalid IKE/IPsec state. |

IKEv2 **rekeying** and **reauthentication** are not equivalent:

- **Rekeying** can replace IKE SA keys using `CREATE_CHILD_SA` without repeating PSK authentication.
- **Reauthentication** requires a new initial IKE negotiation, including IKE_SA_INIT and IKE_AUTH.

---

### Dead Peer Detection (DPD)

**Dead Peer Detection (DPD)** determines whether an established IKEv2 peer is still reachable and responsive.

DPD is useful because an IKE SA and its associated Child SAs can remain established even if the remote router or network path fails. Without timely failure detection, traffic may continue to be sent toward an unreachable peer.

In IKEv2, DPD uses an **INFORMATIONAL exchange**:

1. The local router sends an encrypted **INFORMATIONAL Request** with no informational payloads.
2. The remote peer responds with an encrypted **INFORMATIONAL Response**.
3. Receiving the response confirms that the peer is alive.
4. If the peer does not respond, the router retransmits according to its retry behavior.
5. If the peer is declared dead, the router deletes the failed IKE SA and its associated Child SAs.

```text
       Router A                         Router B
          |                                |
          |---- INFORMATIONAL Request ---->|
          |<--- INFORMATIONAL Response ----|
          |                                |
                Peer confirmed alive
```

DPD does not necessarily require a probe to confirm liveness. Receiving recent, valid, cryptographically protected traffic from a peer can also demonstrate that it is alive.


#### DPD Modes

Cisco IOS XE supports two DPD modes. The meaning of the configured **DPD interval** depends on the mode.

| Mode | Meaning of the DPD interval |
| --- | --- |
| **on-demand** | Inactivity threshold: If no incoming protected traffic has been received for the configured interval, DPD can be triggered when outbound traffic needs to be sent. |
| **periodic** | Probing interval: DPD probes are sent regularly at the configured interval, even when there is no outbound traffic. |

**On-demand DPD** reduces unnecessary control traffic. A completely idle connection may not detect a failed peer until communication is required.

**Periodic DPD** can detect a failed peer even when there is no user traffic to send, at the cost of additional control traffic.

---

#### DPD Configuration

DPD can be configured under an **IKEv2 profile**:

```cisco
crypto ikev2 profile DMVPN-IKEV2
 dpd 30 5 on-demand
```

The syntax is:

```cisco
dpd <interval> <retry-interval> {on-demand | periodic}
```

| Parameter | Purpose |
| --- | --- |
| **interval** | DPD probing interval (periodic) or incoming traffic inactivity threshold (on-demand), in seconds. |
| **retry-interval** | Time between retries when the peer does not respond, in seconds. |
| **on-demand** | Enables on-demand probing. |
| **periodic** | Enables periodic probing. |

Consider the following examples:

**On-demand:**

```cisco
dpd 30 5 on-demand
```

- If no incoming protected traffic has been received for **30 seconds**, a DPD probe can be triggered when outbound traffic needs to be sent.
- If there is no outbound traffic, the 30-second threshold alone does not trigger a DPD probe.
- If the peer does not respond, retries use a **5-second** interval.

**Periodic:**

```cisco
dpd 30 5 periodic
```

- DPD probes are sent at regular **30-second** intervals, including when there is no outbound user traffic.
- If the peer does not respond, retries use a **5-second** interval.

The second numeric argument is a **retry interval**, not a retry count.

DPD can also be configured globally:

```cisco
crypto ikev2 dpd 30 5 on-demand
```

The IKEv2 profile's DPD configuration **overrides the global configuration** for peers matching that profile.

DPD is disabled by default in Cisco IOS XE.

#### DPD in DMVPN

In DMVPN, DPD monitors **IKEv2 peer liveness**, not NHRP registration or routing-protocol reachability.

Consider a direct Spoke1-to-Spoke2 IPsec connection:

1. Spoke1 and Spoke2 establish an IKE SA and Child SA to protect their direct GRE traffic.
2. Spoke2 becomes unreachable.
3. Spoke1's DPD probes fail.
4. Spoke1 removes the failed IKE SA and associated Child SAs.
5. Subsequent traffic can trigger a new IKEv2 negotiation when the peer becomes reachable again.

**DPD does not directly perform NHRP shortcut removal, NHS failover, or routing convergence.** Those functions use their own mechanisms.

For dual-hub DMVPN, DPD helps remove stale security associations, but hub failover still depends on NHRP, routing, and other relevant failure-detection mechanisms.

---

### NAT Traversal (NAT-T)

IKEv2 can detect NAT while establishing the IKE SA. When NAT traversal is required, IKEv2 and encapsulated ESP traffic use **UDP port 4500** instead of relying on native ESP (IP protocol 50) across the translated path. This is distinct from the GRE header's IP protocol 47.

Understanding the port/protocol distinction is useful when troubleshooting underlay ACLs or firewalls:

| Traffic | Typical transport |
| --- | --- |
| IKEv2 initial negotiation | UDP/500 |
| NAT-T IKEv2 and UDP-encapsulated ESP | UDP/4500 |
| ESP without UDP encapsulation | IP protocol 50 |
| Unprotected GRE (for comparison) | IP protocol 47 |

For DMVPN, GRE traffic is typically encapsulated and then protected by ESP, so an underlay observer normally sees the relevant **ESP or UDP-encapsulated ESP** packet, not bare GRE.

---

## Verification and Failure Isolation

These commands identify the IKEv2 configuration in use and whether negotiation succeeded:

```cisco
show crypto ikev2 proposal
show crypto ikev2 policy
show crypto ikev2 profile
show crypto ikev2 sa
show crypto ikev2 sa detailed
show crypto ipsec sa
show crypto session detail
```

In `show crypto ikev2 sa detailed`, inspect:

- **Local and remote IKE endpoints:** They should correspond to the expected NBMA/underlay endpoints, accounting for NAT if present.
- **IKE SA state:** `READY` indicates that the IKEv2 SA negotiation has completed.
- **Negotiated algorithms:** Encryption, integrity/PRF, and DH group should match the intended IKEv2 policy/proposal.
- **Authentication:** Confirm that PSK authentication was used when expected.

`show crypto ipsec sa` is a separate check: an IKE SA may be established without usable ESP Child SAs. Also check **encapsulation and decapsulation counters** to confirm that traffic is actually protected and received.

### Map Failures to the Exchange or Object

| Symptom | Investigate first |
| --- | --- |
| No IKE negotiation reaches the peer | Underlay reachability, UDP/500 or UDP/4500, routing/NAT/firewall rules. |
| IKE_SA_INIT proposal mismatch | IKEv2 policy selection, IKEv2 proposal transforms, DH compatibility. |
| IKE_AUTH authentication failure | IKEv2 profile matching, local/remote identity, keyring peer selection, PSK. |
| IKE SA is ready, but no IPsec SA | ESP transform/proposal agreement, Child SA negotiation, IPsec profile and traffic selection. |
| Hub IPsec works, direct spoke-to-spoke IPsec does not | Spoke-to-spoke NBMA reachability, IKEv2 peer matching, PSK/keyring behavior, and direct SA establishment. |
| IKE and ESP SAs exist, but overlay traffic fails | NHRP mappings/shortcuts, overlay routing, tunnel protection, and ESP packet counters. |

**The operational sequence to remember is:** negotiate IKE algorithms, authenticate IKE peers, establish Child SAs, and then verify that the expected GRE traffic is being protected. The [IKEv2/IPsec Verification and Troubleshooting](troubleshooting.md) page develops these checks into a systematic workflow.

---

## Key Points

- **IKEv2 is the control plane**; **ESP is the IPsec data plane** that protects GRE traffic.
- The **IKEv2 SA** protects IKEv2 exchanges and manages **IPsec Child SAs**; each Child SA includes an inbound and outbound ESP SA.
- **IKE_SA_INIT** negotiates IKE cryptographic parameters and DH keying material; **IKE_AUTH** authenticates identities/PSKs and normally creates the first Child SA.
- **CREATE_CHILD_SA** adds or rekeys SAs; **INFORMATIONAL** handles status, deletion, and liveness-related messaging.
- An **IKEv2 proposal** defines algorithms for the IKE SA; an **IKEv2 policy** selects applicable proposal(s) using the local address/FVRF.
- An **IKEv2 keyring** stores PSKs; an **IKEv2 profile** matches peers, selects authentication methods, and references that keyring.
- **The IKEv2 profile does not reference the IKEv2 policy.** An **IPsec profile** references the IKEv2 profile and IPsec transform set, then attaches to the DMVPN tunnel.
- **PSKs authenticate peers; they are not the session keys used by ESP.** DH and IKEv2 key derivation generate fresh cryptographic keying material.
- Dynamic spoke-to-spoke DMVPN paths require their **own** IKEv2/IPsec security state when protected by IPsec.
- An **IKE SA in READY state is not sufficient** to prove ESP traffic is passing. Verify Child SAs and packet counters separately.

---

## References

- [Cisco: Configuring Internet Key Exchange Version 2 (IKEv2), IOS XE Security and VPN Configuration Guide](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/security-vpn/security-vpn/m_sec-cfg-ikev2-flex.html)
- [Cisco: Troubleshoot IOS IKEv2 Debugs for Site-to-Site VPN with PSKs](https://www.cisco.com/c/en/us/support/docs/ip/internet-key-exchange-ike/115934-technote-ikev2-00.html)
- [Cisco: Dynamic Multipoint IPsec VPNs](https://www.cisco.com/c/en/us/support/docs/security-vpn/ipsec-negotiation-ike-protocols/41940-dmvpn.html)
- [RFC 7296: Internet Key Exchange Protocol Version 2 (IKEv2)](https://www.rfc-editor.org/rfc/rfc7296.html)
