
# Advanced MPLS L3VPN Topics

This section covers additional MPLS topics related to **VPN route selection** and **IPv6 transport over an MPLS backbone**.

The three topics are:

- **VPN Import Path Selection:** Controls which VPNv4 paths are imported into a VRF's BGP table.
- **6PE:** Provides global IPv6 connectivity over an IPv4 MPLS backbone.
- **6VPE:** Provides IPv6 MPLS L3VPN connectivity over an IPv4 MPLS backbone.

---

## VPN Import Path Selection

In an MPLS L3VPN, a PE may receive multiple VPNv4 paths for the same destination.

By default, BGP best-path selection determines which VPNv4 paths are eligible for import into a VRF.

The **BGP import path** feature provides additional control over this process.

```text
MP-BGP VPNv4 table
        |
        | Import path selection
        v
VRF BGP table
        |
        | BGP multipath selection
        v
VRF routing table
```

This is particularly useful when multiple PEs advertise routes for the same customer destination.

Important distinctions:

- **Route Targets:** Determine whether a VPN route is eligible for import into a VRF.
- **Import Path Selection:** Determines which eligible VPNv4 paths are imported into the VRF's BGP table.
- **BGP Maximum Paths:** Determines whether multiple eligible BGP paths can be installed in the VRF routing table.

See [VPN Import Path Selection](import-path.md).

---

## IPv6 over an IPv4 MPLS Backbone

An MPLS provider can transport IPv6 traffic without enabling IPv6 routing on its core P routers.

Two technologies provide this capability:

- **6PE:** IPv6 Provider Edge
- **6VPE:** IPv6 VPN Provider Edge

Both use an IPv4 MPLS backbone to transport IPv6 packets between PE routers.

```text
IPv6 Site                             IPv6 Site
    |                                     |
   CE1                                   CE2
    |                                     |
   PE1 -------- P1 -------- P2 -------- PE2
            IPv4 MPLS Backbone
```

The P routers only need IPv4 routing and MPLS forwarding.

### 6PE

6PE provides **global IPv6 reachability** over an IPv4 MPLS backbone.

It uses:

- The global IPv6 routing table on PE routers.
- MP-BGP IPv6 labeled unicast to exchange IPv6 routes and labels.
- MPLS transport across the IPv4 backbone.

6PE is **not an MPLS L3VPN**. It does not use customer VRFs, RDs, or RTs.

See [6PE](6pe.md).

### 6VPE

6VPE extends MPLS L3VPN services to IPv6 customers.

It uses:

- IPv6 routing tables within customer VRFs.
- MP-BGP VPNv6 to exchange customer IPv6 VPN routes.
- RDs and RTs to distinguish routes and control VPN membership.
- MPLS transport across the IPv4 backbone.

6VPE provides the same fundamental customer separation as IPv4 MPLS L3VPNs.

See [6VPE](6vpe.md).

---

## 6PE vs 6VPE

| Feature | 6PE | 6VPE |
| :--- | :--- | :--- |
| Purpose | Global IPv6 connectivity | IPv6 L3VPN connectivity |
| IPv6 routing table | Global | VRF |
| BGP address family | IPv6 labeled unicast | VPNv6 |
| Route Distinguisher | No | Yes |
| Route Target | No | Yes |
| Customer VPN separation | No | Yes |
| IPv6 required on P routers | No | No |
| MPLS transport | Yes | Yes |

---

## Key Points

- VPN Import Path Selection controls which VPNv4 paths are imported into a VRF's BGP table.
- Import Path Selection and BGP multipath are separate mechanisms.
- Both 6PE and 6VPE allow IPv6 traffic to traverse an IPv4 MPLS backbone.
- Neither requires IPv6 routing on the P routers.
- **6PE** provides global IPv6 connectivity without VRFs, RDs, or RTs.
- **6VPE** provides IPv6 MPLS L3VPN connectivity using VRFs, RDs, RTs, and VPNv6.
- The fundamental distinction between 6PE and 6VPE is whether IPv6 traffic belongs to a customer VPN.
