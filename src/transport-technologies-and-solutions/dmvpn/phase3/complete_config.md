# Complete DMVPN Phase 3 Configuration

This is a **CCIE-style configuration and verification lab** for a **dual-hub, dual-cloud DMVPN Phase 3** network. It combines the underlay, mGRE, NHRP, EIGRP, Phase 3 shortcuts, hub redundancy, and **IKEv2/IPsec with preshared keys** into one scenario.

The addressing, tunnel numbers, NHRP domains, and EIGRP AS match the earlier [Dual-Hub Phase 3 Configuration](dual-hub/configuration.md) and [Dual-Hub DMVPN Topologies](dual-hub/topologies.md) pages. This is the **dual-cloud** design, not the alternative single-cloud design.

**Lab method:** Read the requirements for each task before looking at its solution. Configurations are **cumulative**: apply each task's solution to the existing device configuration. The final IPsec task adds security to the working DMVPN, rather than replacing its GRE/NHRP or routing configuration.

---

## Scenario

A company has three branch routers (Spoke1, Spoke2, Spoke3) and two hub routers (Hub1, Hub2). All five routers connect to the same ISP router through independent Ethernet links.

Build two independent DMVPN clouds:

- **Cloud 100:** Hub1 provides the primary overlay path. It uses `Tunnel0` and `172.16.0.0/24`.
- **Cloud 200:** Hub2 provides a backup overlay path. It uses `Tunnel1` and `172.16.1.0/24`.
- **All spokes:** Participate in both clouds through `Tunnel0` and `Tunnel1` using the **same physical WAN interface**.
- **Overlay routing:** EIGRP AS 100, with Cloud 100 preferred under normal conditions.
- **Data forwarding:** Phase 3 NHRP Redirect/Shortcut must allow direct spoke-to-spoke forwarding in the selected cloud.
- **Security:** IKEv2 preshared-key authentication and ESP transport-mode protection must secure both clouds.

```text
                 Hub1                      Hub2
            NBMA 198.51.100.1        NBMA 198.51.100.5
            Tu0 172.16.0.1           Tu1 172.16.1.2
                  |                         |
          +----------------+       +----------------+
          | Cloud 100      |       | Cloud 200      |
          | 172.16.0.0/24  |       | 172.16.1.0/24  |
          | NHS: Hub1      |       | NHS: Hub2      |
          +----------------+       +----------------+
               |   |   |                |   |   |
               |   |   |                |   |   |
             Tu0 Tu0 Tu0              Tu1 Tu1 Tu1
              S1  S2  S3               S1  S2  S3
             .11 .12 .13              .11 .12 .13
```

The ISP connections are physical; the two boxes represent **logical overlays** across those connections. A direct spoke-to-spoke shortcut uses the selected cloud's tunnel interface and GRE key, even though the two clouds share the same physical ISP.

### Device and IP Addressing

| Device | WAN (`Ethernet0/0`) | ISP next hop | Loopback0 (LAN) | Cloud 100 | Cloud 200 |
| --- | --- | --- | --- | --- | --- |
| Hub1 | `198.51.100.1/30` | `198.51.100.2` | `10.1.1.1/24` | `Tunnel0`: `172.16.0.1/24` | Not used |
| Hub2 | `198.51.100.5/30` | `198.51.100.6` | `10.2.2.2/24` | Not used | `Tunnel1`: `172.16.1.2/24` |
| Spoke1 | `198.51.100.9/30` | `198.51.100.10` | `10.11.11.11/24` | `Tunnel0`: `172.16.0.11/24` | `Tunnel1`: `172.16.1.11/24` |
| Spoke2 | `198.51.100.13/30` | `198.51.100.14` | `10.12.12.12/24` | `Tunnel0`: `172.16.0.12/24` | `Tunnel1`: `172.16.1.12/24` |
| Spoke3 | `198.51.100.17/30` | `198.51.100.18` | `10.13.13.13/24` | `Tunnel0`: `172.16.0.13/24` | `Tunnel1`: `172.16.1.13/24` |

The ISP has these addresses on its five separate links: `198.51.100.2/30`, `.6/30`, `.10/30`, `.14/30`, and `.18/30`. Its interface **names** depend on how the virtual lab is cabled.

### Cloud-Specific Parameters

| Requirement | Cloud 100 | Cloud 200 |
| --- | --- | --- |
| NHS | Hub1, `172.16.0.1` | Hub2, `172.16.1.2` |
| NHRP network ID | `100` | `200` |
| NHRP authentication | `NHRP100` | `NHRP200` |
| GRE tunnel key | `100` | `200` |
| Hub multicast replication | Dynamic NHRP mappings | Dynamic NHRP mappings |
| Spoke multicast replication | Hub1 (`198.51.100.1`) | Hub2 (`198.51.100.5`) |
| EIGRP bandwidth | `100000` Kbit/s | `100000` Kbit/s |
| EIGRP delay on spokes | `1000` (tens of microseconds) | `5000` (tens of microseconds) |

**Scope:** The three spoke LANs must be mutually reachable and survive the failure of either hub's overlay. Each hub also has its own connected loopback. **There is no inter-hub core link** in this topology, so do not assume Hub1's LAN remains reachable through Hub2 (or vice versa). That would require an additional core/inter-hub routing design.

---

## Task 1: Build the IP Underlay

### Requirements

1. Configure the IP addresses on all five WAN interfaces and the five loopback interfaces.
2. Configure the five routed ISP interfaces using the addressing above. The ISP must **not** participate in overlay routing.
3. Configure an IPv4 default route on every hub and spoke pointing to its directly connected ISP address.
4. Verify reachability between **all WAN/NBMA addresses**, including spoke-to-spoke reachability. Do not configure any DMVPN tunnels yet.

### Solution

**ISP**

```cisco
hostname ISP
!
interface Ethernet0/0
 description TO-HUB1
 ip address 198.51.100.2 255.255.255.252
 no shutdown
!
interface Ethernet0/1
 description TO-HUB2
 ip address 198.51.100.6 255.255.255.252
 no shutdown
!
interface Ethernet0/2
 description TO-SPOKE1
 ip address 198.51.100.10 255.255.255.252
 no shutdown
!
interface Ethernet0/3
 description TO-SPOKE2
 ip address 198.51.100.14 255.255.255.252
 no shutdown
!
interface Ethernet1/0
 description TO-SPOKE3
 ip address 198.51.100.18 255.255.255.252
 no shutdown
```

**Hub1**

```cisco
hostname Hub1
!
interface Ethernet0/0
 ip address 198.51.100.1 255.255.255.252
 no shutdown
!
interface Loopback0
 ip address 10.1.1.1 255.255.255.0
!
ip route 0.0.0.0 0.0.0.0 198.51.100.2
```

**Hub2**

```cisco
hostname Hub2
!
interface Ethernet0/0
 ip address 198.51.100.5 255.255.255.252
 no shutdown
!
interface Loopback0
 ip address 10.2.2.2 255.255.255.0
!
ip route 0.0.0.0 0.0.0.0 198.51.100.6
```

**Spoke1**

```cisco
hostname Spoke1
!
interface Ethernet0/0
 ip address 198.51.100.9 255.255.255.252
 no shutdown
!
interface Loopback0
 ip address 10.11.11.11 255.255.255.0
!
ip route 0.0.0.0 0.0.0.0 198.51.100.10
```

**Spoke2**

```cisco
hostname Spoke2
!
interface Ethernet0/0
 ip address 198.51.100.13 255.255.255.252
 no shutdown
!
interface Loopback0
 ip address 10.12.12.12 255.255.255.0
!
ip route 0.0.0.0 0.0.0.0 198.51.100.14
```

**Spoke3**

```cisco
hostname Spoke3
!
interface Ethernet0/0
 ip address 198.51.100.17 255.255.255.252
 no shutdown
!
interface Loopback0
 ip address 10.13.13.13 255.255.255.0
!
ip route 0.0.0.0 0.0.0.0 198.51.100.18
```

### Verify

```cisco
Spoke1# ping 198.51.100.1 source Ethernet0/0
Spoke1# ping 198.51.100.5 source Ethernet0/0
Spoke1# ping 198.51.100.13 source Ethernet0/0
Spoke1# ping 198.51.100.17 source Ethernet0/0
Spoke1# show ip route 0.0.0.0
ISP# show ip route connected
```

All five WAN subnets must appear as connected routes on the ISP.

---

## Task 2: Configure Both mGRE/NHRP Clouds

### Requirements

1. Build **Cloud 100** on Hub1 `Tunnel0` and each spoke's `Tunnel0`.
2. Build **Cloud 200** on Hub2 `Tunnel1` and each spoke's `Tunnel1`.
3. Use mGRE with the specified GRE keys, NHRP network IDs, NHRP authentication strings, and IP addresses.
4. Configure each spoke to register with **Hub1 on Tunnel0** and **Hub2 on Tunnel1**, including multicast replication to the appropriate hub.
5. Allow the hubs to replicate multicast traffic to dynamically registered spokes.
6. Do not add IKEv2/IPsec protection yet.

### Solution

**Hub1: Cloud 100**

```cisco
interface Tunnel0
 ip address 172.16.0.1 255.255.255.0
 ip nhrp authentication NHRP100
 ip nhrp network-id 100
 ip nhrp map multicast dynamic
 tunnel source Ethernet0/0
 tunnel mode gre multipoint
 tunnel key 100
```

**Hub2: Cloud 200**

```cisco
interface Tunnel1
 ip address 172.16.1.2 255.255.255.0
 ip nhrp authentication NHRP200
 ip nhrp network-id 200
 ip nhrp map multicast dynamic
 tunnel source Ethernet0/0
 tunnel mode gre multipoint
 tunnel key 200
```

**Spoke1**

```cisco
interface Tunnel0
 ip address 172.16.0.11 255.255.255.0
 ip nhrp authentication NHRP100
 ip nhrp network-id 100
 ip nhrp nhs 172.16.0.1 nbma 198.51.100.1 multicast
 tunnel source Ethernet0/0
 tunnel mode gre multipoint
 tunnel key 100
!
interface Tunnel1
 ip address 172.16.1.11 255.255.255.0
 ip nhrp authentication NHRP200
 ip nhrp network-id 200
 ip nhrp nhs 172.16.1.2 nbma 198.51.100.5 multicast
 tunnel source Ethernet0/0
 tunnel mode gre multipoint
 tunnel key 200
```

**Spoke2**

```cisco
interface Tunnel0
 ip address 172.16.0.12 255.255.255.0
 ip nhrp authentication NHRP100
 ip nhrp network-id 100
 ip nhrp nhs 172.16.0.1 nbma 198.51.100.1 multicast
 tunnel source Ethernet0/0
 tunnel mode gre multipoint
 tunnel key 100
!
interface Tunnel1
 ip address 172.16.1.12 255.255.255.0
 ip nhrp authentication NHRP200
 ip nhrp network-id 200
 ip nhrp nhs 172.16.1.2 nbma 198.51.100.5 multicast
 tunnel source Ethernet0/0
 tunnel mode gre multipoint
 tunnel key 200
```

**Spoke3**

```cisco
interface Tunnel0
 ip address 172.16.0.13 255.255.255.0
 ip nhrp authentication NHRP100
 ip nhrp network-id 100
 ip nhrp nhs 172.16.0.1 nbma 198.51.100.1 multicast
 tunnel source Ethernet0/0
 tunnel mode gre multipoint
 tunnel key 100
!
interface Tunnel1
 ip address 172.16.1.13 255.255.255.0
 ip nhrp authentication NHRP200
 ip nhrp network-id 200
 ip nhrp nhs 172.16.1.2 nbma 198.51.100.5 multicast
 tunnel source Ethernet0/0
 tunnel mode gre multipoint
 tunnel key 200
```

### Verify

```cisco
Spoke1# show ip nhrp nhs
Spoke1# show dmvpn interface Tunnel0
Spoke1# show dmvpn interface Tunnel1
Hub1# show ip nhrp
Hub2# show ip nhrp
```

Expected relationships:

- Hub1 receives three **Cloud 100** NHRP registrations (`172.16.0.11`, `.12`, `.13`).
- Hub2 receives three **Cloud 200** NHRP registrations (`172.16.1.11`, `.12`, `.13`).
- Every spoke registers once in **each** cloud.
- Both tunnels on each spoke use `Ethernet0/0` as their source, but their **GRE keys differ**.

NHRP registration confirms basic tunnel endpoint resolution; it does not prove LAN routing or direct spoke-to-spoke forwarding.

---

## Task 3: Configure EIGRP Across Both Clouds

### Requirements

1. Run **EIGRP AS 100** between each spoke and **both hubs**.
2. Advertise each router's Loopback0 network into EIGRP without forming EIGRP neighbors on the loopback or WAN interface.
3. Permit the hubs to advertise spoke-learned routes back out their mGRE interfaces.
4. Configure spokes as **EIGRP stubs**, advertising only connected networks.
5. Set the EIGRP tunnel bandwidth to **100,000 Kbit/s** everywhere. On each spoke, use delay **1000** on Tunnel0 and **5000** on Tunnel1 to favor Cloud 100 when the competing route metrics are otherwise equal.
6. Use appropriate EIGRP **next-hop-self** behavior on the hubs for Phase 3.

### Solution

**Hub1**

```cisco
interface Tunnel0
 bandwidth 100000
 delay 1000
 no ip split-horizon eigrp 100
!
router eigrp 100
 passive-interface default
 no passive-interface Tunnel0
 network 172.16.0.0 0.0.0.255
 network 10.1.1.1 0.0.0.0
```

**Hub2**

```cisco
interface Tunnel1
 bandwidth 100000
 delay 1000
 no ip split-horizon eigrp 100
!
router eigrp 100
 passive-interface default
 no passive-interface Tunnel1
 network 172.16.1.0 0.0.0.255
 network 10.2.2.2 0.0.0.0
```

**Spoke1**

```cisco
interface Tunnel0
 bandwidth 100000
 delay 1000
!
interface Tunnel1
 bandwidth 100000
 delay 5000
!
router eigrp 100
 passive-interface default
 no passive-interface Tunnel0
 no passive-interface Tunnel1
 network 172.16.0.0 0.0.0.255
 network 172.16.1.0 0.0.0.255
 network 10.11.11.11 0.0.0.0
 eigrp stub connected
```

**Spoke2**

```cisco
interface Tunnel0
 bandwidth 100000
 delay 1000
!
interface Tunnel1
 bandwidth 100000
 delay 5000
!
router eigrp 100
 passive-interface default
 no passive-interface Tunnel0
 no passive-interface Tunnel1
 network 172.16.0.0 0.0.0.255
 network 172.16.1.0 0.0.0.255
 network 10.12.12.12 0.0.0.0
 eigrp stub connected
```

**Spoke3**

```cisco
interface Tunnel0
 bandwidth 100000
 delay 1000
!
interface Tunnel1
 bandwidth 100000
 delay 5000
!
router eigrp 100
 passive-interface default
 no passive-interface Tunnel0
 no passive-interface Tunnel1
 network 172.16.0.0 0.0.0.255
 network 172.16.1.0 0.0.0.255
 network 10.13.13.13 0.0.0.0
 eigrp stub connected
```

The hubs need `no ip split-horizon eigrp 100` because they learn spoke prefixes and advertise them to other spokes through the **same mGRE interface**. Do **not** configure `no ip next-hop-self eigrp 100` on the hubs; that is a typical Phase 2 requirement, not the default Phase 3 design.

### Verify

```cisco
Spoke1# show ip eigrp neighbors
Spoke1# show ip eigrp topology
Spoke1# show ip route 10.12.12.12
Hub1# show ip eigrp neighbors
Hub2# show ip eigrp neighbors
```

Each spoke should have **two EIGRP neighbors**: Hub1 on Tunnel0 and Hub2 on Tunnel1. Each hub should have **three** EIGRP neighbors. There should be **no spoke-to-spoke EIGRP adjacencies** and no hub-to-hub adjacency over the DMVPN clouds.

Before enabling shortcuts, test routed connectivity:

```cisco
Spoke1# ping 10.12.12.12 source Loopback0
Spoke1# ping 10.13.13.13 source Loopback0
Spoke2# ping 10.11.11.11 source Loopback0
```

The ordinary route to a remote spoke LAN should normally select **Hub1 / Tunnel0**. Inspect the actual metrics rather than assuming the chosen interface from delay alone.

---

## Task 4: Enable DMVPN Phase 3 Shortcuts and Summarization

### Requirements

1. Configure both hubs to generate **NHRP Redirects**.
2. Configure spokes to use **NHRP Shortcuts**.
3. Instead of advertising all individual spoke LANs to spokes, make each hub advertise the covering `10.8.0.0/13` summary toward its DMVPN cloud. This prefix covers all three spoke LANs but excludes the two hub LANs.
4. Verify that traffic can initially follow the selected hub and then use a **direct spoke-to-spoke shortcut**, even though the ordinary EIGRP next hop is the hub.
5. Keep shortcuts in their **respective clouds**. Cloud 100 and Cloud 200 must not share NHRP state.

### Solution

**Hub1**

```cisco
interface Tunnel0
 ip nhrp redirect
 ip summary-address eigrp 100 10.8.0.0 255.248.0.0
```

**Hub2**

```cisco
interface Tunnel1
 ip nhrp redirect
 ip summary-address eigrp 100 10.8.0.0 255.248.0.0
```

**All three spokes**

```cisco
interface Tunnel0
 ip nhrp shortcut
!
interface Tunnel1
 ip nhrp shortcut
```

The explicit `ip nhrp shortcut` commands show the required Phase 3 behavior; on modern IOS XE the command is enabled by default and won't appear in the running configuration.

The `10.8.0.0/13` summary covers `10.8.0.0` through `10.15.255.255`, including the spoke LANs `10.11.11.0/24`, `10.12.12.0/24`, and `10.13.13.0/24`. The summary is intentionally **not** an underlay route: it is advertised only over DMVPN.

### Verify

On Spoke1, before generating sustained traffic:

```cisco
Spoke1# show ip route 10.12.12.12
Spoke1# show ip eigrp topology
Spoke1# show ip nhrp shortcut
```

Then generate traffic and inspect the shortcut:

```cisco
Spoke1# ping 10.12.12.12 source Loopback0 repeat 20
Spoke1# show ip nhrp shortcut
Spoke1# show ip route next-hop-override
Spoke1# show ip cef 10.12.12.12 detail
Spoke1# show ip nhrp traffic
```

Expected behavior:

1. Spoke1's ordinary EIGRP route to `10.12.12.12` is covered by the hub-advertised `10.8.0.0/13` summary.
2. Initial traffic can traverse Hub1. Hub1 forwards toward Spoke2 and sends an NHRP Redirect.
3. Spoke1 resolves Spoke2's tunnel/NBMA mapping and installs shortcut forwarding information.
4. Subsequent matching traffic can reach Spoke2 **directly** over Cloud 100, using Spoke2's `172.16.0.12` tunnel address and `198.51.100.13` NBMA address.
5. The EIGRP route may **still point to Hub1**. The direct forwarding change is reflected in NHRP/CEF state, not necessarily in the ordinary EIGRP next hop.

A shortcut is not guaranteed to exist immediately after a single ping; give NHRP time to complete the redirect and resolution exchanges. The exact NHRP entry or next-hop override shown can vary with the resolved destination prefix.

---

## Task 5: Verify Dual-Hub Failover

### Requirements

1. With both hubs operational, verify that remote spoke LAN reachability prefers **Cloud 100**.
2. Simulate loss of Hub1's overlay without shutting down the spoke WAN interfaces or the ISP.
3. Confirm that spokes retain EIGRP reachability to each other's LANs **through Hub2 / Cloud 200**.
4. Demonstrate that Phase 3 can establish a direct shortcut in **Cloud 200** when routing selects that cloud.
5. Restore Hub1 and confirm that the preferred normal path returns when the relevant state converges.

### Solution and Verification

On each spoke, inspect the initial route and EIGRP neighbor state:

```cisco
Spoke1# show ip eigrp neighbors
Spoke1# show ip route 10.12.12.12
Spoke1# show ip nhrp shortcut
```

**Simulate Hub1 failure:**

```cisco
Hub1(config)# interface Tunnel0
Hub1(config-if)# shutdown
```

Check the alternate EIGRP path and test traffic between spoke LANs:

```cisco
Spoke1# show ip eigrp neighbors
Spoke1# show ip route 10.12.12.12
Spoke1# ping 10.12.12.12 source Loopback0 repeat 20
Spoke1# show ip nhrp shortcut
Spoke1# show ip cef 10.12.12.12 detail
```

With Hub1 unavailable, the ordinary EIGRP route should use **Hub2 / Tunnel1**. Any newly established Cloud 200 shortcut to Spoke2 should use Spoke2's **Cloud 200 tunnel IP** `172.16.1.12` and **NBMA** `198.51.100.13`.

**Important:** A preexisting direct spoke-to-spoke shortcut in Cloud 100 may remain usable after Hub1 fails because that data path bypasses the hub. A hub failure does not automatically imply that every active spoke-to-spoke shortcut immediately moves to Cloud 200. Judge **routing reconvergence and direct forwarding state separately**. For a clean Cloud 200 shortcut demonstration, test traffic without a previously installed Cloud 100 shortcut for that destination (or allow existing shortcut state to expire).

Restore Hub1:

```cisco
Hub1(config)# interface Tunnel0
Hub1(config-if)# no shutdown
```

Wait for NHRP registration and EIGRP adjacency recovery, then confirm that the preferred EIGRP route normally returns to Tunnel0. A direct shortcut learned earlier may remain until its own state changes.

**The purpose of this test is spoke LAN continuity.** The topology does not include a core path that would keep Hub1's connected `10.1.1.0/24` network reachable when Hub1 fails.

---

## Task 6: Protect Both DMVPN Clouds with IKEv2/IPsec

### Requirements

The GRE/NHRP/routing design is now functional. Add IKEv2 and IPsec **without changing the existing tunnel addresses, NHRP domains, GRE keys, EIGRP configuration, or underlay routes**.

Configure the following on **every hub and spoke**:

| Configuration object | Required setting |
| --- | --- |
| **IKEv2 proposal** | Name `DMVPN-IKE-PROP`; AES-CBC-256 encryption; SHA-256 integrity; SHA-256 PRF; DH group 14 |
| **IKEv2 policy** | Name `DMVPN-IKE-POLICY`; reference `DMVPN-IKE-PROP`; match all local addresses in the global routing table |
| **IKEv2 keyring** | Name `DMVPN-KEYRING`; one wildcard peer entry for dynamic peers; common lab PSK `CCIE-DMVPN-PSK` for local and remote authentication |
| **IKEv2 profile** | Name `DMVPN-IKE-PROFILE`; match any remote IPv4 IKE identity; local and remote PSK authentication; reference the keyring |
| **IKEv2 profile lifetime/DPD** | IKE SA lifetime **43,200 seconds**; DPD **30 seconds**, **5-second** retry, **on-demand** |
| **IPsec transform set** | Name `DMVPN-ESP-SET`; **ESP AES-256** encryption; **ESP HMAC-SHA-256** integrity; **transport mode** |
| **IPsec profile** | Name `DMVPN-IPSEC`; reference the transform set and IKEv2 profile; enable **PFS group 14**; Child SA time lifetime **1,800 seconds** |
| **Tunnel protection** | Apply the IPsec profile to every DMVPN tunnel. |

Use **one common set of cryptographic objects on each router**. A single wildcard key simplifies this closed lab with dynamically discovered peers; it is not a recommendation to use one shared secret for every site in production.

### Solution A: IKEv2 Proposal and Policy

**Configure identically on Hub1, Hub2, Spoke1, Spoke2, and Spoke3:**

```cisco
crypto ikev2 proposal DMVPN-IKE-PROP
 encryption aes-cbc-256
 integrity sha256
 prf sha256
 group 14
!
crypto ikev2 policy DMVPN-IKE-POLICY
 proposal DMVPN-IKE-PROP
```

- The **proposal** defines acceptable IKE SA cryptographic algorithms.
- The **policy** selects that proposal for eligible local IKEv2 negotiations.
- The lab uses the **global routing table for WAN reachability**. No `tunnel vrf` or `match fvrf` is required here. When introducing an FVRF, account for it in the IKEv2 policy and profile.

### Solution B: IKEv2 Keyring and Profile

**Configure identically on all five DMVPN routers:**

```cisco
crypto ikev2 keyring DMVPN-KEYRING
 peer DMVPN-PEERS
  address 0.0.0.0 0.0.0.0
  pre-shared-key local CCIE-DMVPN-PSK
  pre-shared-key remote CCIE-DMVPN-PSK
!
crypto ikev2 profile DMVPN-IKE-PROFILE
 match identity remote address 0.0.0.0 0.0.0.0
 authentication local pre-share
 authentication remote pre-share
 keyring local DMVPN-KEYRING
 lifetime 43200
 dpd 30 5 on-demand
```

- `address 0.0.0.0 0.0.0.0` in the **keyring** permits matching dynamic remote IPv4 IKE endpoints.
- `match identity remote address 0.0.0.0 0.0.0.0` in the **profile** matches remote IPv4 **IKE identities**. A peer's identity and packet source IP are distinct concepts.
- `authentication local` and `authentication remote` both use **preshared-key** authentication.
- `keyring local` links the IKEv2 profile to the keyring containing the PSKs.
- `lifetime 43200` configures the **IKE SA** lifetime, not the ESP Child SA lifetime.
- `dpd 30 5 on-demand` checks peer liveness when needed; it does not perform NHRP NHS failover.

### Solution C: IPsec Transform Set and Profile

**Configure identically on all five DMVPN routers:**

```cisco
crypto ipsec transform-set DMVPN-ESP-SET esp-aes 256 esp-sha256-hmac
 mode transport
!
crypto ipsec profile DMVPN-IPSEC
 set transform-set DMVPN-ESP-SET
 set ikev2-profile DMVPN-IKE-PROFILE
 set pfs group14
 set security-association lifetime seconds 1800
```

- The **transform set** defines the algorithms and encapsulation mode used for ESP protection.
- **Transport mode** avoids adding a second outer IP header around the GRE packet. The GRE packet is already encapsulated for the underlay.
- The **IPsec profile** references the transform set and IKEv2 profile; it also configures Child SA protection settings.
- `set pfs group14` requests an independent Diffie-Hellman exchange when creating/rekeying applicable Child SAs using `CREATE_CHILD_SA`. The first Child SA is normally created during `IKE_AUTH` without an additional Child SA DH exchange.
- The Child SA **1,800-second** time-based lifetime overrides the global IPsec SA time lifetime for SAs using this profile.

### Solution D: Apply Tunnel Protection

**Hub1:**

```cisco
interface Tunnel0
 tunnel protection ipsec profile DMVPN-IPSEC
```

**Hub2:**

```cisco
interface Tunnel1
 tunnel protection ipsec profile DMVPN-IPSEC
```

**Spoke1, Spoke2, and Spoke3 (same commands on each):**

```cisco
interface Tunnel0
 tunnel protection ipsec profile DMVPN-IPSEC shared
!
interface Tunnel1
 tunnel protection ipsec profile DMVPN-IPSEC shared
```

Each spoke's two mGRE tunnels use the same `tunnel source Ethernet0/0` and **the same IPsec profile**. The `shared` keyword allows shared IPsec security state for GRE sessions with the same source/destination endpoints. The **GRE tunnel keys** (100 and 200) still distinguish the two clouds after decryption.

Do **not** use `tunnel source <interface-ip>` with a literal IP address when using shared tunnel protection.

Hub1 and Hub2 each have only one mGRE tunnel interface and do not need `shared` for this lab.

### Verify IKEv2 and ESP

Generate traffic from a spoke to the hub through each cloud and between spokes:

```cisco
Spoke1# ping 172.16.0.1 source Tunnel0
Spoke1# ping 172.16.1.2 source Tunnel1
Spoke1# ping 10.12.12.12 source Loopback0 repeat 20
```

Inspect crypto state on the spoke and hubs:

```cisco
Spoke1# show crypto ikev2 sa
Spoke1# show crypto ikev2 sa detailed
Spoke1# show crypto ipsec sa
Spoke1# show crypto session detail
Spoke1# show dmvpn interface Tunnel0
Spoke1# show dmvpn interface Tunnel1
```

Verify these **independent outcomes**:

| Check | What to establish |
| --- | --- |
| IKEv2 SA | Peer authentication and IKE SA negotiation succeed for the applicable NBMA peers. |
| Child SA | ESP inbound and outbound SAs are installed with their own SPIs. |
| ESP transform | The negotiated ESP algorithms and **transport** mode match the requirements. |
| ESP counters | `#pkts encaps` / `#pkts encrypt` and `#pkts decaps` / `#pkts decrypt` increase as traffic flows. |
| Hub paths | Both Cloud 100 and Cloud 200 retain NHRP registration and EIGRP connectivity. |
| Shortcuts | Direct spoke-to-spoke forwarding can be established with protection enabled, not only over the hub path. |
| Shared source | Both spoke tunnels can coexist with IPsec while using the same WAN source interface. |

**A successful `show crypto ikev2 sa` result alone is not enough.** Confirm working ESP Child SAs and actual forwarding. Similarly, IPsec can be healthy while a separate NHRP or EIGRP problem prevents a shortcut.

---

## Final Validation Checklist

| Requirement | Useful command / test |
| --- | --- |
| All five WAN addresses mutually reachable | `ping ... source Ethernet0/0` |
| Correct NHRP registration in each cloud | `show ip nhrp nhs`, `show dmvpn` |
| Each spoke forms EIGRP with both hubs | `show ip eigrp neighbors` |
| Spoke LAN reachability via preferred cloud | `show ip route`, sourced `ping` |
| Hub summaries advertised to spokes | `show ip route 10.8.0.0`, `show ip eigrp topology` |
| Phase 3 Redirect/Shortcut works | `show ip nhrp shortcut`, `show ip nhrp traffic` |
| Direct path uses intended cloud | `show ip cef ... detail`, `show ip route next-hop-override` |
| Hub1 failure leaves viable Hub2 path | Repeat sourced pings after Hub1 Tunnel0 shutdown |
| IKEv2 SA establishes with PSK | `show crypto ikev2 sa detailed` |
| ESP protects both clouds | `show crypto ipsec sa` and traffic counters |
| Shared-source tunnel protection works | Both spoke tunnels up and protected, with distinct GRE keys |

### Troubleshooting Order

When a test fails, work from the bottom up:

1. **Underlay:** WAN interfaces, ISP forwarding, correct NBMA reachability, default routes.
2. **GRE/NHRP:** Tunnel source, tunnel key, NHRP network ID, authentication, NHS registration, multicast replication.
3. **Overlay routing:** EIGRP adjacency, stub setting, split horizon, summary advertisements, successor and feasible distance.
4. **Phase 3:** Redirect, resolution, shortcut/NHO, CEF forwarding.
5. **IKEv2:** Proposal/policy compatibility, profile/identity matching, keyring/PSK, IKE SA establishment.
6. **ESP:** Transform set, transport mode, IPsec profile, `shared` setting, Child SA installation, encrypt/decrypt counters.

When IPsec is enabled, **IKEv2/IPsec problems can prevent NHRP and EIGRP traffic from functioning**. Do not assume a failure at an upper layer originates there; check the security state as soon as a protected tunnel stops carrying traffic.

---

## Key Points

- The **two DMVPN clouds share the ISP underlay**, but have separate tunnel addresses, GRE keys, NHRP domains, and NHRP shortcut state.
- **Hub1 / Tunnel0** is the preferred EIGRP path; **Hub2 / Tunnel1** provides an alternative path for the three spoke LANs.
- **Phase 3** allows hubs to advertise a summary while spokes establish direct shortcuts without changing the EIGRP next hop.
- `no ip split-horizon eigrp 100` permits the hubs to advertise spoke-learned routes back out the mGRE interfaces. Normal EIGRP next-hop-self behavior remains enabled.
- **IKEv2 proposals/policies** control IKE SA algorithms; **IKEv2 keyrings/profiles** provide peer matching and PSK authentication.
- **IPsec transform sets/profiles** control ESP protection and Child SA settings; tunnel protection applies them to mGRE.
- Because each spoke has **two tunnels with the same source interface**, apply the **same IPsec profile** with `shared` to both tunnels.
- Examine **underlay, routing, NHRP, IKEv2, and ESP** independently during troubleshooting.

## References

- [Cisco: Dynamic Multipoint IPsec VPNs](https://www.cisco.com/c/en/us/support/docs/security-vpn/ipsec-negotiation-ike-protocols/41940-dmvpn.html)
- [Cisco: Sharing IPsec with Tunnel Protection](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/sec-vpn/b-security-vpn/m_sec-conn-dmvpn-share-ipsec-w-tun-protect-0.html)
- [Cisco: IKEv2 and Legacy VPNs (DMVPN IKEv2 Example)](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/security-vpn/security-vpn/m_sec-apx-ikev2-legacy.html)
- [Cisco: Configuring Internet Key Exchange Version 2](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/security-vpn/security-vpn/m_sec-cfg-ikev2-flex.html)
- [Cisco: Configure DMVPN Phase 3 Using IKEv2](https://www.cisco.com/c/en/us/support/docs/security/dynamic-multipoint-vpn-dmvpn/224281-configure-dmvpn-phase-3-using-ikev2.html)
- [RFC 7296: Internet Key Exchange Protocol Version 2](https://www.rfc-editor.org/rfc/rfc7296)
- [RFC 4303: IP Encapsulating Security Payload (ESP)](https://www.rfc-editor.org/rfc/rfc4303)
