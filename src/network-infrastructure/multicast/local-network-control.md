# Local Network Control Multicast

The IPv4 **Local Network Control Block** is:

```text
224.0.0.0/24
```

It is reserved for routing protocols, multicast control protocols, and other protocols that operate on the local network.

## Link-Local Scope

Packets destined for `224.0.0.0/24` are **not forwarded by multicast routers to another network segment**, regardless of their TTL.

```text
LAN 1                         LAN 2

Host ---- SW1 ---- R1 -------- SW2 ---- Host
                    X
              224.0.0.0/24
```

The traffic can travel through Layer 2 switches within the same VLAN, but it does not cross a Layer 3 boundary.

The scope is determined by the **destination address**, not simply by using a TTL of 1. Some protocols use other TTL values while still using a link-local multicast destination.

## Important Addresses

| Address | Purpose |
| --- | --- |
| `224.0.0.0` | Reserved base address |
| `224.0.0.1` | All Systems on this Subnet |
| `224.0.0.2` | All Routers on this Subnet |
| `224.0.0.5` | OSPF AllSPFRouters |
| `224.0.0.6` | OSPF AllDRouters |
| `224.0.0.9` | RIPv2 Routers |
| `224.0.0.10` | EIGRP Routers |
| `224.0.0.13` | All PIM Routers |
| `224.0.0.18` | VRRP |
| `224.0.0.22` | IGMPv3 Membership Reports |
| `224.0.0.102` | HSRPv2, GLBP |

### 224.0.0.1 — All Systems

`224.0.0.1` represents all systems on the local subnet.

### 224.0.0.2 — All Routers

`224.0.0.2` represents all routers on the local subnet.

Some protocols also use this address for their own control traffic. For example, **HSRPv1** sends hello messages to `224.0.0.2`.

HSRPv2 instead uses `224.0.0.102`.

### 224.0.0.22 — IGMPv3 Routers

IGMPv3 Membership Reports are sent to `224.0.0.22`.

All IGMPv3-capable multicast routers on the local network listen for this address.

Other IGMPv3 hosts do not listen to this group, so do not hear each others' messages.

## Key Point

`224.0.0.0/24` is for **local control traffic**, not general multicast applications.

Even if multicast routing is enabled, traffic destined for this block remains on the local link.