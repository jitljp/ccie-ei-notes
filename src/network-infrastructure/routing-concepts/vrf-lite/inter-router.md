# VRF-Lite Inter-Router Connectivity

VRF-Lite allows multiple routers or Layer 3 switches to extend separate routing domains across a network without using MPLS.

For example, R1 and R2 can each have a `BLUE` VRF and a `RED` VRF. Traffic in `BLUE` on R1 can reach `BLUE` on R2, while traffic in `RED` remains separate.

There are two common ways to connect the VRFs:

1. **Dedicated physical interfaces:** Use a separate routed link for each VRF.
2. **802.1Q subinterfaces:** Carry multiple VRFs over one physical link, using a separate VLAN for each VRF.

## Dedicated Physical Interfaces

The simplest method uses a separate physical link for each VRF.

```text
              VRF BLUE (10.10.10.0/30)
        E0/0 10.10.10.1     10.10.10.2 E0/0
     R1 ------------------------------- R2
        E0/1 10.20.20.1     10.20.20.2 E0/1
              VRF RED (10.20.20.0/30)
```

- `E0/0` belongs to `BLUE` on both routers.
- `E0/1` belongs to `RED` on both routers.
- Each link carries traffic for only its assigned VRF.

### R1 Configuration

Create the VRFs and assign the physical interfaces.

```text
R1(config)# vrf definition BLUE
R1(config-vrf)# address-family ipv4
R1(config-vrf-af)# exit-address-family
R1(config-vrf)# exit
R1(config)# vrf definition RED
R1(config-vrf)# address-family ipv4
R1(config-vrf-af)# exit-address-family
R1(config-vrf)# exit

R1(config)# interface Ethernet0/0
R1(config-if)# vrf forwarding BLUE
R1(config-if)# ip address 10.10.10.1 255.255.255.252
R1(config-if)# no shutdown

R1(config)# interface Ethernet0/1
R1(config-if)# vrf forwarding RED
R1(config-if)# ip address 10.20.20.1 255.255.255.252
R1(config-if)# no shutdown
```

### R2 Configuration

R2 needs its own VRFs and corresponding interface assignments.

```text
R2(config)# vrf definition BLUE
R2(config-vrf)# address-family ipv4
R2(config-vrf-af)# exit-address-family
R2(config-vrf)# exit
R2(config)# vrf definition RED
R2(config-vrf)# address-family ipv4
R2(config-vrf-af)# exit-address-family
R2(config-vrf)# exit

R2(config)# interface Ethernet0/0
R2(config-if)# vrf forwarding BLUE
R2(config-if)# ip address 10.10.10.2 255.255.255.252
R2(config-if)# no shutdown

R2(config)# interface Ethernet0/1
R2(config-if)# vrf forwarding RED
R2(config-if)# ip address 10.20.20.2 255.255.255.252
R2(config-if)# no shutdown
```

The two routers now have separate Layer 3 connections for `BLUE` and `RED`.

Verify directly connected reachability from R1:

```text
R1# ping vrf BLUE 10.10.10.2
R1# ping vrf RED 10.20.20.2
```

**VRF names are locally significant.** The names do not have to match between R1 and R2. What matters is that the connected interfaces use the intended routing domains and compatible IP addressing.

## Routing Between the Routers

Connecting two interfaces in the same routing domain allows communication across the directly connected subnet. To reach remote networks, each VRF must also learn routes to those networks.

For example, configure the following loopbacks:

| Router | VRF | Loopback | IP Address |
|---|---|---|---|
| R1 | BLUE | Loopback10 | 192.168.10.1/32 |
| R2 | BLUE | Loopback10 | 192.168.10.2/32 |
| R1 | RED | Loopback20 | 192.168.20.1/32 |
| R2 | RED | Loopback20 | 192.168.20.2/32 |

Configure the loopbacks on both routers:

```text
R1(config)# interface Loopback10
R1(config-if)# vrf forwarding BLUE
R1(config-if)# ip address 192.168.10.1 255.255.255.255
R1(config)# interface Loopback20
R1(config-if)# vrf forwarding RED
R1(config-if)# ip address 192.168.20.1 255.255.255.255

R2(config)# interface Loopback10
R2(config-if)# vrf forwarding BLUE
R2(config-if)# ip address 192.168.10.2 255.255.255.255
R2(config)# interface Loopback20
R2(config-if)# vrf forwarding RED
R2(config-if)# ip address 192.168.20.2 255.255.255.255
```

For this example, use VRF-aware static routes:

```text
R1(config)# ip route vrf BLUE 192.168.10.2 255.255.255.255 10.10.10.2
R1(config)# ip route vrf RED 192.168.20.2 255.255.255.255 10.20.20.2

R2(config)# ip route vrf BLUE 192.168.10.1 255.255.255.255 10.10.10.1
R2(config)# ip route vrf RED 192.168.20.1 255.255.255.255 10.20.20.1
```

Each static route is installed in its specified VRF, and its next hop is reached through the matching inter-router interface.

```text
R1# ping vrf BLUE 192.168.10.2 source Loopback10
R1# ping vrf RED 192.168.20.2 source Loopback20
```

You could use OSPF, EIGRP, or BGP instead of static routes. The routing protocols would operate in their respective VRFs and would **not** automatically exchange routes between `BLUE` and `RED`.

Detailed protocol configuration is covered under [VRF-Aware Routing](../vrf-aware-routing/overview.md).

## 802.1Q Subinterfaces

Using a separate physical connection for every VRF becomes impractical as the number of VRFs increases.

The following configuration is an **alternative** to the dedicated-link design, not an addition to it. Remove the original `E0/1` routed-link configuration if converting that topology to subinterfaces.

Instead, multiple VRFs can share one Ethernet connection using **802.1Q VLAN tagging**.

```text
                         Single physical link
                        Ethernet0/0 <-> E0/0
                    +---------------------------+
                    |                           |
                   R1                           R2
                    |                           |
                    +-- VLAN 10: VRF BLUE ------+
                    |                           |
                    +-- VLAN 20: VRF RED -------+

VLAN 10: 10.10.10.0/30
VLAN 20: 10.20.20.0/30
```

Each router uses a Layer 3 subinterface for each VLAN:

| VRF | VLAN | R1 Subinterface | R2 Subinterface |
|---|---|---|---|
| BLUE | 10 | E0/0.10 | E0/0.10 |
| RED | 20 | E0/0.20 | E0/0.20 |

The VLAN tag identifies the subinterface receiving an Ethernet frame. The **receiving subinterface's VRF assignment** determines which routing table is used for the IP lookup.

### R1 Configuration

Keep the `BLUE` and `RED` VRF definitions from the physical-interface example. Replace the two routed inter-router interfaces with these subinterfaces:

```text
R1(config)# interface Ethernet0/0
R1(config-if)# no ip address
R1(config-if)# no shutdown

R1(config)# interface Ethernet0/0.10
R1(config-subif)# encapsulation dot1Q 10
R1(config-subif)# vrf forwarding BLUE
R1(config-subif)# ip address 10.10.10.1 255.255.255.252

R1(config)# interface Ethernet0/0.20
R1(config-subif)# encapsulation dot1Q 20
R1(config-subif)# vrf forwarding RED
R1(config-subif)# ip address 10.20.20.1 255.255.255.252
```

### R2 Configuration

```text
R2(config)# interface Ethernet0/0
R2(config-if)# no ip address
R2(config-if)# no shutdown

R2(config)# interface Ethernet0/0.10
R2(config-subif)# encapsulation dot1Q 10
R2(config-subif)# vrf forwarding BLUE
R2(config-subif)# ip address 10.10.10.2 255.255.255.252

R2(config)# interface Ethernet0/0.20
R2(config-subif)# encapsulation dot1Q 20
R2(config-subif)# vrf forwarding RED
R2(config-subif)# ip address 10.20.20.2 255.255.255.252
```

The loopback and static-route configurations from the previous example can remain unchanged. The next-hop addresses are the same, but they are now reached through the subinterfaces.

On routers using routed 802.1Q subinterfaces, `encapsulation dot1Q` is configured **on each subinterface**. You do not configure `switchport mode trunk` on these routed subinterfaces.

### Forwarding Example

Suppose R1 forwards a packet from its `BLUE` loopback to `192.168.10.2`:

1. R1 looks up the destination in the `BLUE` routing table.
2. The static route identifies `10.10.10.2` as the next hop through `E0/0.10`.
3. R1 sends the packet with an 802.1Q tag for **VLAN 10**.
4. R2 receives the frame on `E0/0.10`, which belongs to `BLUE`.
5. R2 performs the IP lookup in its `BLUE` routing table and delivers the packet to `Loopback10`.

Traffic in `RED` follows the same process using VLAN 20 and `E0/0.20`.

**The VLAN tag does not contain a VRF identifier.** Each device independently maps its local subinterface to a VRF.

## VRF-Lite with Switch SVIs

On supported Layer 3 switches, another common option is to carry the VLANs over a Layer 2 trunk and assign **SVIs** to the VRFs rather than configuring routed 802.1Q subinterfaces.

For example, on R1 as a Layer 3 switch (with IP routing enabled):

```text
R1(config)# ip routing
R1(config)# vlan 10
R1(config-vlan)# exit
R1(config)# vlan 20
R1(config-vlan)# exit

R1(config)# interface Ethernet0/0
R1(config-if)# switchport mode trunk
R1(config-if)# switchport trunk allowed vlan 10,20
R1(config-if)# no shutdown

R1(config)# interface Vlan10
R1(config-if)# vrf forwarding BLUE
R1(config-if)# ip address 10.10.10.1 255.255.255.252
R1(config-if)# no shutdown

R1(config)# interface Vlan20
R1(config-if)# vrf forwarding RED
R1(config-if)# ip address 10.20.20.1 255.255.255.252
R1(config-if)# no shutdown
```

Configure R2 with corresponding VLANs and SVIs using `.2` addresses.

The distinction is where Layer 3 forwarding occurs:

| Method | Layer 3 Interface | Inter-Router Transport |
|---|---|---|
| Dedicated routed links | Physical interface | Separate untagged links |
| Routed 802.1Q subinterfaces | Subinterface | One tagged physical link |
| Switch SVIs | VLAN interface | Layer 2 trunk carrying the VLANs |

Support for routed subinterfaces and SVIs depends on the device and platform.

## Verification and Troubleshooting

Common problems include:

- **Incorrect VRF membership:** The receiving interface or subinterface belongs to the wrong VRF or the global routing table.
- **Incorrect VLAN tags:** The two endpoints use different VLAN IDs, or an intermediate switch does not carry the required VLANs.
- **Missing routes:** Directly connected next hops are reachable, but remote prefixes are not installed in the correct VRF.
- **Missing return routes:** One router has a route to the remote network, but the other does not have a return route.
- **Incorrect interface configuration:** The IP address was removed when `vrf forwarding` was applied, or the parent interface is shut down.

## VRF-Lite vs EVN Trunking

Traditional VRF-Lite requires an explicit interface, subinterface, or SVI mapping for each routed VRF connection.

**Easy Virtual Network (EVN)** can simplify this model with `vnet tag` and `vnet trunk`. EVN trunking uses 802.1Q tagging and can interoperate on the wire with traditional VRF-Lite subinterfaces on supported platforms.

EVN is covered in the [next page](evn.md).

## Key Points

- VRF-Lite can extend multiple isolated routing domains between routers without MPLS.
- Dedicated physical interfaces provide one routed connection per VRF.
- 802.1Q subinterfaces allow multiple VRFs to share a single physical link.
- Each subinterface belongs to one VRF, and its VLAN tag distinguishes traffic on the link.
- On supported Layer 3 switches, SVIs assigned to VRFs can connect over a Layer 2 VLAN trunk.
- VRF names are locally significant. Matching VRF names on neighboring routers are not required.
- Remote networks require static routes or VRF-aware dynamic routing protocols.
- Extending a VRF between routers does **not** leak routes between different VRFs.
- EVN trunking can simplify traditional VRF-Lite subinterface configurations on supported platforms.
