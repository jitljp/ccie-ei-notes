# Easy Virtual Network (EVN)

**Easy Virtual Network (EVN)** is a Cisco feature that simplifies extending multiple VRFs across routers or Layer 3 switches.

EVN builds on VRF-Lite and IEEE 802.1Q encapsulation. It does not introduce a new tunneling or routing protocol, and it does not require MPLS.

With traditional VRF-Lite, an administrator typically creates an 802.1Q subinterface for each VRF on every inter-router link. EVN can **automatically create and configure these subinterfaces** using a **VNET trunk**.

EVN was designed to simplify multi-VRF campus networks. Support is platform- and software-dependent, so verify feature availability on the IOS XE image used in your lab.

## EVN vs Traditional VRF-Lite

Consider two routers that carry the `BLUE` and `RED` VRFs over a single Ethernet link.

Traditional VRF-Lite requires manually configured subinterfaces:

```text
R1                                  R2
E0/0.10 -------- VLAN 10 -------- E0/0.10    VRF BLUE
E0/0.20 -------- VLAN 20 -------- E0/0.20    VRF RED
```

Each subinterface must be configured with its own 802.1Q encapsulation, VRF assignment, and IP address.

With EVN, each router instead configures the parent interface as a **VNET trunk**:

```text
R1                                  R2
E0/0 ========= VNET trunk ======== E0/0
                |        |
             Tag 10   Tag 20
              BLUE      RED
```

EVN automatically creates the corresponding VLAN-tagged subinterfaces.

| Feature | Traditional VRF-Lite | EVN |
|---|---|---|
| Separate VRF routing tables | Yes | Yes |
| Inter-router 802.1Q tagging | Yes | Yes |
| Manually configure subinterfaces | Yes | No, derived automatically |
| VRF-to-tag mapping | Configured per subinterface | `vnet tag` under the VRF |
| Common trunk settings | Repeated as needed | Inherited from the parent interface |
| Per-VRF settings | Configured on each subinterface | `vnet name` overrides |
| MPLS required | No | No |

EVN is **wire-compatible with traditional 802.1Q-based VRF-Lite**. One device can use an EVN trunk while the other uses manually configured subinterfaces with matching VLAN tags and addressing.

## VNET Tags

A **VNET tag** identifies a virtual network on an EVN trunk.

For Ethernet trunks, the VNET tag is carried in the **802.1Q VLAN ID field**.

Configure a tag under each VRF:

```text
R1(config)# vrf definition BLUE
R1(config-vrf)# vnet tag 10
R1(config-vrf)# address-family ipv4

R1(config)# vrf definition RED
R1(config-vrf)# vnet tag 20
R1(config-vrf)# address-family ipv4
```

This configuration maps:

| VRF | VNET Tag | 802.1Q VLAN ID |
|---|---|---|
| BLUE | 10 | 10 |
| RED | 20 | 20 |

On the receiving router, the tag determines which virtual network receives the frame.

VNET tags have a documented range of **2 through 4094**. Use consistent tags for the same virtual network on connected devices, and avoid VLAN ID conflicts along the trunk path. Cisco documents some platform-specific tag restrictions.

**A VNET tag is not a Route Distinguisher (RD) or Route Target (RT).** A basic EVN trunk does not require either an RD or an RT.

## VNET Trunks

A **VNET trunk** carries multiple virtual networks on one routed Ethernet interface.

Configure it with:

```text
R1(config)# interface Ethernet0/0
R1(config-if)# ip address 10.10.10.1 255.255.255.252
R1(config-if)# vnet trunk
R1(config-if)# no shutdown
```

By default, EVN creates a derived subinterface for every VNET-tagged VRF included on the trunk.

For the `BLUE` and `RED` definitions above, the resulting logical interfaces are:

```text
Ethernet0/0         Global routing table, untagged
Ethernet0/0.10      VRF BLUE, VLAN 10
Ethernet0/0.20      VRF RED, VLAN 20
```

The base interface is part of **vnet global**, which represents the global routing table. On an EVN trunk, it carries **untagged traffic**; the named VNETs carry tagged traffic.

### Automatically Derived Subinterfaces

EVN creates the VRF subinterfaces internally. They are not manually configured in the usual interface configuration.

For example:

```text
R1# show derived-config interface Ethernet0/0.10
```

The relevant derived configuration resembles:

```text
interface Ethernet0/0.10
 vrf forwarding BLUE
 encapsulation dot1Q 10
 ip address 10.10.10.1 255.255.255.252
```

The `RED` subinterface similarly uses `Ethernet0/0.20`, `vrf forwarding RED`, and `encapsulation dot1Q 20`.

**The derived subinterfaces inherit the parent interface's IP address by default.** Therefore, the same subnet and local address may appear in the global routing table and in multiple VRFs. This works because each VRF has an independent routing context.

Do not confuse these automatically derived subinterfaces with manually created `interface Ethernet0/0.10` or `interface Ethernet0/0.20` configurations.

## EVN Configuration Example

The following example connects two routers over an EVN trunk. Each router also has a loopback interface in each VRF.

```text
                    E0/0                     E0/0
              R1 10.10.10.1/30 -------- 10.10.10.2/30 R2
                           EVN trunk
                     VLAN 10: VRF BLUE
                     VLAN 20: VRF RED

R1 Loopback10 (BLUE): 192.168.10.1/32
R2 Loopback10 (BLUE): 192.168.10.2/32
R1 Loopback20 (RED):  192.168.20.1/32
R2 Loopback20 (RED):  192.168.20.2/32
```

### R1 Configuration

```text
R1(config)# vrf definition BLUE
R1(config-vrf)# vnet tag 10
R1(config-vrf)# address-family ipv4
R1(config-vrf-af)# exit-address-family
R1(config-vrf)# exit

R1(config)# vrf definition RED
R1(config-vrf)# vnet tag 20
R1(config-vrf)# address-family ipv4
R1(config-vrf-af)# exit-address-family
R1(config-vrf)# exit

R1(config)# interface Ethernet0/0
R1(config-if)# ip address 10.10.10.1 255.255.255.252
R1(config-if)# vnet trunk
R1(config-if)# no shutdown

R1(config)# interface Loopback10
R1(config-if)# vrf forwarding BLUE
R1(config-if)# ip address 192.168.10.1 255.255.255.255

R1(config)# interface Loopback20
R1(config-if)# vrf forwarding RED
R1(config-if)# ip address 192.168.20.1 255.255.255.255
```

### R2 Configuration

```text
R2(config)# vrf definition BLUE
R2(config-vrf)# vnet tag 10
R2(config-vrf)# address-family ipv4
R2(config-vrf-af)# exit-address-family
R2(config-vrf)# exit

R2(config)# vrf definition RED
R2(config-vrf)# vnet tag 20
R2(config-vrf)# address-family ipv4
R2(config-vrf-af)# exit-address-family
R2(config-vrf)# exit

R2(config)# interface Ethernet0/0
R2(config-if)# ip address 10.10.10.2 255.255.255.252
R2(config-if)# vnet trunk
R2(config-if)# no shutdown

R2(config)# interface Loopback10
R2(config-if)# vrf forwarding BLUE
R2(config-if)# ip address 192.168.10.2 255.255.255.255

R2(config)# interface Loopback20
R2(config-if)# vrf forwarding RED
R2(config-if)# ip address 192.168.20.2 255.255.255.255
```

The following tests verify reachability across the directly connected VNETs:

```text
R1# ping vrf BLUE 10.10.10.2
R1# ping vrf RED 10.10.10.2
```

The destination is identical, but each ping uses a different VRF and VLAN tag.

These configurations alone do **not** provide routes to the remote loopbacks. Those routes must be configured statically or learned through a VRF-aware routing protocol.

## Routing Protocols Over EVN Trunks

EVN does not automatically exchange routes between routers or between VRFs.

Routing protocols operate independently within the virtual networks, just as they do with traditional VRF-Lite. For example, R1 can run one OSPFv2 process in `BLUE` and another in `RED`:

```text
R1(config)# router ospf 10 vrf BLUE
R1(config-router)# router-id 1.1.1.10
R1(config-router)# network 10.10.10.0 0.0.0.3 area 0
R1(config-router)# network 192.168.10.1 0.0.0.0 area 0

R1(config)# router ospf 20 vrf RED
R1(config-router)# router-id 1.1.1.20
R1(config-router)# network 10.10.10.0 0.0.0.3 area 0
R1(config-router)# network 192.168.20.1 0.0.0.0 area 0
```

On R2:

```text
R2(config)# router ospf 10 vrf BLUE
R2(config-router)# router-id 2.2.2.10
R2(config-router)# network 10.10.10.0 0.0.0.3 area 0
R2(config-router)# network 192.168.10.2 0.0.0.0 area 0

R2(config)# router ospf 20 vrf RED
R2(config-router)# router-id 2.2.2.20
R2(config-router)# network 10.10.10.0 0.0.0.3 area 0
R2(config-router)# network 192.168.20.2 0.0.0.0 area 0
```

The OSPF process associated with `BLUE` activates on the `BLUE` derived interface, while the process associated with `RED` activates on the `RED` derived interface.

Verify the separate routing tables and OSPF neighbors:

```text
R1# show ip route vrf BLUE ospf
R1# show ip route vrf RED ospf
R1# show ip ospf vrf BLUE neighbor
R1# show ip ospf vrf RED neighbor
```

Other supported EVN routing configurations include static routing and EIGRP. Detailed routing-protocol configuration is covered under [VRF-Aware Routing](../vrf-aware-routing/overview.md).

## Restricting VRFs on a VNET Trunk

By default, a VNET trunk carries all eligible VNET-tagged VRFs on the router.

To restrict a trunk to particular VRFs, first create a **VRF list**:

```text
R1(config)# vrf list ONLY_BLUE
R1(config-vrf-list)# member BLUE
R1(config-vrf-list)# exit-vrf-list
```

Apply the list to the trunk:

```text
R1(config)# interface Ethernet0/0
R1(config-if)# vnet trunk list ONLY_BLUE
```

Now only `BLUE` is carried as a tagged VNET on this trunk. `RED` is excluded, while the untagged **vnet global** remains present.

To carry both VRFs, create a list containing both members or restore the unrestricted command:

```text
R1(config)# interface Ethernet0/0
R1(config-if)# vnet trunk
```

Check the list:

```text
R1# show vrf list
```

## Per-VRF Interface Overrides

Many interface settings configured on the parent VNET trunk are **inherited by all derived subinterfaces**.

For example, an OSPF interface cost on the parent may be inherited by both VNETs.

Use `vnet name` to enter a per-VNET interface configuration mode and override an eligible setting:

```text
R1(config)# interface Ethernet0/0
R1(config-if)# vnet name BLUE
R1(config-if-vnet)# ip ospf cost 100
R1(config-if-vnet)# exit-if-vnet
```

This applies the OSPF interface cost of `100` to the derived interface for `BLUE`, without applying it to `RED`.

You can inspect the resulting configuration:

```text
R1# show derived-config interface Ethernet0/0.10
```

`vnet name` **does not assign the physical interface to a VRF**. It selects one VNET's derived interface for configuration overrides. Only supported interface commands are available in this mode.

## EVN and Route Replication

EVN also introduced a simpler mechanism for sharing selected routes between VRFs: **route replication**.

For example, routes from a shared-services VRF can be made available in another VRF without using MP-BGP VPNv4 route import/export.

The command is configured under the destination VRF's address family:

```text
R1(config)# vrf definition BLUE
R1(config-vrf)# address-family ipv4
R1(config-vrf-af)# route-replicate from vrf SERVICES unicast connected
```

This makes eligible connected routes from `SERVICES` available for forwarding in `BLUE`, subject to platform support and route selection behavior.

**Route replication is separate from VNET trunking.** It can be relevant even when no EVN trunk is configured. It also does not automatically advertise replicated routes to remote routers through a routing protocol.

See [Route Replication](../route-leaking/route-replication.md) for details, including filtering, return-path requirements, and route redistribution.

## Verification and Troubleshooting

Useful EVN-specific commands include:

| Command | Purpose |
|---|---|
| `show vnet tag` | Displays VNET tag assignments |
| `show vnet detail` | Displays EVN and VRF information |
| `show vnet counters` | Displays configured VNET, trunk, and subinterface counts |
| `show vrf list` | Displays configured VRF lists |
| `show derived-config interface Ethernet0/0.10` | Displays a derived EVN subinterface's configuration |
| `show ip route vrf BLUE` | Displays routes in `BLUE` |
| `show ip cef vrf BLUE` | Displays the VRF forwarding table |

If a VNET is not working, check that:

1. The feature and commands are supported by the device and software image.
2. Both routers have the intended VRFs and corresponding VNET tags.
3. The parent interfaces are operational and configured as VNET trunks.
4. Any applied VRF list includes the required VRF.
5. The trunk's VLAN tags are carried correctly across intermediate switches.
6. The routing protocol or static routes are configured in the correct VRF.
7. Any per-VNET overrides have not introduced inconsistent settings.

## EVN Restrictions

EVN is **not synonymous with all VRF-Lite features**. Cisco documents restrictions for its EVN implementation, including:

- A maximum of **32 virtual networks** in the documented design.
- EVN trunk support depends on the hardware, line cards, and software version.
- An EVN trunk cannot simultaneously use conventional, manually configured VRF-Lite subinterfaces on the same interface.
- **OSPFv2** is supported, but **OSPFv3** is not supported by the documented EVN feature.
- **IPv6 traffic is restricted to vnet global** on the documented EVN trunk implementation.
- Features such as ACLs, NAT, and NetFlow are not supported directly on EVN trunk interfaces in Cisco's documented restrictions.
- The documented EVN implementation does not support IS-IS or RIP, and does not support BGP-based route replication.

These restrictions concern **EVN**, not VRF-Lite in general. Verify actual support before designing or labbing EVN on a particular platform.

## Key Points

- **EVN (Easy Virtual Network)** simplifies inter-router VRF-Lite connectivity using 802.1Q trunking.
- `vnet tag` maps a VRF to a tag carried in the Ethernet VLAN ID field.
- `vnet trunk` automatically creates **derived, VLAN-tagged subinterfaces** for the eligible VNETs.
- Derived subinterfaces inherit the parent interface's IP address and supported interface settings by default.
- **vnet global** uses the global routing table and carries **untagged** traffic on the trunk.
- `vrf list` and `vnet trunk list` restrict the tagged VRFs carried on a particular trunk.
- `vnet name` configures per-VNET interface overrides.
- Use `show derived-config` to examine automatically generated subinterface settings.
- EVN does **not** automatically leak routes between VRFs or advertise remote routes; those require separate routing configurations.
- EVN route replication is a separate capability from EVN trunking.
- EVN support and restrictions are platform-dependent.

## References

- [Cisco: Overview of Easy Virtual Network](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/ip-addressing/ip-addressing/m_evn-overview.html)
- [Cisco: Configuring Easy Virtual Network](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/ip-addressing/ip-addressing/m_evn-confg.html)
- [Cisco: Configure Easy Virtual Network](https://www.cisco.com/c/en/us/support/docs/ip/ip-routing/117974-configure-evn-00.html)
