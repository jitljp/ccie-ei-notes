# MPLS L2VPN: Ethernet over MPLS (EoMPLS)

**Ethernet over MPLS (EoMPLS)** extends an Ethernet connection across an MPLS network. It is a form of **MPLS Layer 2 VPN (L2VPN)** and **Any Transport over MPLS (AToM)**.

With EoMPLS, two customer devices can exchange Ethernet frames as though they were connected through the same Layer 2 link, even when an MPLS network separates them. The customer devices do **not** need to run MPLS.

## Key Terminology

| Term | Meaning |
|---|---|
| **CE (Customer Edge)** | Customer router or switch connected to the provider network. |
| **PE (Provider Edge)** | MPLS router that originates or terminates the L2VPN. |
| **Attachment Circuit (AC)** | Local Ethernet port or VLAN subinterface connecting a CE to a PE. |
| **Pseudowire (PW)** | Logical point-to-point connection that carries a Layer 2 service between PEs. |
| **Virtual Circuit (VC)** | Logical Layer 2 circuit emulated by the pseudowire. In basic EoMPLS, the terms *VC* and *pseudowire* are often used interchangeably. |
| **VC ID** | Identifier used by both PEs to associate the two endpoints of the same pseudowire. |
| **VPWS** | Virtual Private Wire Service: a point-to-point L2VPN service. |

A pseudowire is **not a physical wire**: it emulates a Layer 2 circuit by encapsulating frames and transporting them across a packet network. EoMPLS is one way to implement a pseudowire.

## How EoMPLS Works

Consider the following topology:

```text
       10.1.35.3/24                               10.1.35.5/24
       R3 (CE)                                       R5 (CE)
         | G0/2                                      G0/4 |
         |                                               |
       G0/3                                            G0/5
       R2 (PE) ------------ MPLS core --------------- R4 (PE)
       Lo0 1.1.1.2/32                              Lo0 1.1.1.4/32
                  <---- EoMPLS pseudowire ---->
                  <------ VC ID 35 ----------->

       R3 and R5 behave as though they share an Ethernet segment.
```

1. R3 sends an Ethernet frame toward R5 (for example, an OSPF Hello to `224.0.0.5`).
2. R2 receives the frame on its **attachment circuit**, G0/3, and associates it with pseudowire **VC 35**.
3. R2 encapsulates the frame with an MPLS **pseudowire/VC label**, plus a **transport label** when needed to reach R4 across the MPLS core.
4. MPLS routers forward the labeled packet toward R4.
5. R4 removes the MPLS encapsulation and sends the Ethernet frame out G0/5 toward R5.

R2 and R4 do **not** route the customer's IP packet, inspect its destination subnet to select a route, or need to participate in R3/R5's OSPF process. A point-to-point EoMPLS xconnect forwards traffic by attachment circuit and pseudowire, rather than by learning customer MAC addresses.

### MPLS Label Stack

A typical EoMPLS packet across a multi-hop MPLS core carries two labels:

```text
[ Transport label ][ Pseudowire (VC) label ][ Customer Ethernet frame ]
```

- **Transport label:** Delivers the packet toward the remote PE, using the MPLS LSP.
- **Pseudowire label:** Identifies the correct pseudowire/attachment circuit at the egress PE.

Both PEs allocate **their own local pseudowire labels** and signal them to the other PE. The label values do **not** have to match. The **VC ID** is a separate identifier, and it **must match** on both ends of this basic LDP-signaled pseudowire.

**Note:** The transport label might not be present on the final hop because of penultimate-hop popping (PHP). On a directly connected PE-to-PE MPLS link, the imposed stack may contain only the pseudowire label.

## Requirements

1. **Platform support:** Both PEs must support EoMPLS/L2VPN pseudowires in the chosen IOS XE image and interface type.
2. **IP reachability:** Each PE must have an IP route to the remote PE's pseudowire endpoint (normally a loopback `/32`).
3. **MPLS transport:** The core must support MPLS forwarding and have a usable labeled path between PEs.
4. **LDP signaling:** LDP must be available to signal pseudowire labels. A classic `xconnect` automatically initiates targeted LDP discovery toward the configured peer when necessary.
5. **Matching service:** The PEs must use compatible pseudowire types/encapsulation, the same VC ID, and compatible MTU/control-word settings.
6. **Attachment circuits:** The PE-facing customer ports must be suitable for Layer 2 xconnect operation and must not be used as routed IP interfaces at the same time.
7. **MTU:** Core links must accommodate the customer frame plus MPLS label overhead (normally 4 bytes per label, plus any optional pseudowire control word).

The core and pseudowire use LDP in different ways:

- **Ordinary LDP** distributes labels for transport FECs and helps establish the MPLS LSP.
- **Targeted LDP** enables pseudowire label signaling between PEs even when they are not directly connected. A directly connected pair of PEs may use one LDP session for both purposes.

An MPLS L3VPN, VRF, route target, or MP-BGP VPNv4 session is **not required** for this basic EoMPLS service.

## EoMPLS Configuration: Port Mode

This example uses **port mode**, which associates the entire CE-facing Ethernet port with one pseudowire. The R2–R4 transport network is `10.1.200.0/24`.

**Assumed transport addresses:** R2 G0/4 = `10.1.200.2/24`; R4 G0/2 = `10.1.200.4/24`. R2 and R4 use loopbacks `1.1.1.2/32` and `1.1.1.4/32` as LDP identifiers and pseudowire endpoints.

### R2 (PE)

```text
interface Loopback0
 ip address 1.1.1.2 255.255.255.255
!
mpls ldp router-id Loopback0 force
!
interface GigabitEthernet0/4
 ip address 10.1.200.2 255.255.255.0
 mpls ip
!
ip route 1.1.1.4 255.255.255.255 10.1.200.4
!
interface GigabitEthernet0/3
 no ip address
 no keepalive
 xconnect 1.1.1.4 35 encapsulation mpls
```

### R4 (PE)

```text
interface Loopback0
 ip address 1.1.1.4 255.255.255.255
!
mpls ldp router-id Loopback0 force
!
interface GigabitEthernet0/2
 ip address 10.1.200.4 255.255.255.0
 mpls ip
!
ip route 1.1.1.2 255.255.255.255 10.1.200.2
!
interface GigabitEthernet0/5
 no ip address
 no keepalive
 xconnect 1.1.1.2 35 encapsulation mpls
```

**Command notes:**

- `mpls ip` enables MPLS on the **core-facing** link. Do not configure it on the customer attachment circuit just to enable this pseudowire.
- `mpls ldp router-id Loopback0 force` selects the stable loopback as the LDP identifier. Changing it on a live router can reset LDP sessions.
- `ip route` gives each PE reachability to the other's loopback. An IGP could provide this reachability instead.
- `no ip address` keeps the attachment circuit as a Layer 2 service interface.
- `no keepalive` disables Ethernet keepalives; Cisco documents this for port-mode xconnect on certain platforms.
- `xconnect 1.1.1.4 35 encapsulation mpls` binds the local attachment circuit to an MPLS pseudowire toward **R4**, using **VC ID 35**. Configure the reverse peer address on R4.

If the CE-facing port is a switchport-capable interface, the platform may also require `no switchport` before configuring the xconnect. Commands and feature support vary by hardware/image; on some IOS XE platforms the newer `interface pseudowire` / `l2vpn xconnect context` configuration style is used instead.

### R3 and R5 (CEs)

R3 and R5 remain ordinary OSPF routers. **No MPLS or L2VPN configuration is required on the CEs.**

**R3**

```text
interface GigabitEthernet0/2
 ip address 10.1.35.3 255.255.255.0
 ip ospf 1 area 0
!
router ospf 1
 router-id 1.1.1.3
```

**R5**

```text
interface GigabitEthernet0/4
 ip address 10.1.35.5 255.255.255.0
 ip ospf 1 area 0
!
router ospf 1
 router-id 1.1.1.5
```

When the pseudowire is operational, OSPF Hellos can pass across it, and R3 and R5 can establish an adjacency (assuming their OSPF settings match). With the default OSPF broadcast network type, they perform normal DR/BDR election.

## Verification

### 1. Verify PE-to-PE IP and MPLS Transport

On R2 (reverse the destination for R4):

```text
R2# ping 1.1.1.4 source Loopback0
R2# show ip route 1.1.1.4
R2# show mpls interfaces
R2# show mpls ldp neighbor
R2# show mpls forwarding-table
```

Confirm that the remote PE loopback is reachable, MPLS is enabled on the core link, and LDP is operational.

### 2. Verify the Pseudowire

```text
R2# show mpls l2transport vc
R2# show mpls l2transport vc detail
R2# show xconnect all
```

On IOS XE versions using the newer command structure, use:

```text
R2# show l2vpn atom vc
R2# show l2vpn atom vc detail
R2# show l2vpn service
```

A working pseudowire should show **VC status: UP**. In detailed output, examine:

| Field | What to verify |
|---|---|
| **Destination address** | Correct remote PE loopback. |
| **VC ID** | Matches the remote PE's configuration. |
| **Local interface / circuit** | Correct attachment circuit, operationally up. |
| **Signaling protocol / LDP peer** | LDP signaling is operational; targeted Hello/session information may be shown. |
| **MPLS VC labels** | Both local and remote pseudowire labels are assigned. |
| **Imposed label stack** | Outgoing labels and forwarding toward remote PE. |
| **MTU / Control Word** | Compatible parameters at both ends. |

### 3. Verify OSPF on the CEs

```text
R3# show ip ospf neighbor
R3# show ip ospf interface GigabitEthernet0/2
R3# ping 10.1.35.5 source GigabitEthernet0/2
```

If the pseudowire is UP but OSPF is not FULL, investigate CE OSPF parameters, attachment-circuit state, and the platform's handling of multicast/control traffic.

## Port Mode vs. VLAN Mode

| Mode | Attachment circuit | Behavior |
|---|---|---|
| **Port mode** | Physical Ethernet interface | Transports the Ethernet service received on the whole port over one pseudowire. |
| **VLAN mode** | 802.1Q subinterface or Ethernet service instance | Transports traffic associated with a particular VLAN/service over a pseudowire. |

A basic VLAN-mode example, where supported:

```text
interface GigabitEthernet0/3.35
 encapsulation dot1Q 35
 xconnect 1.1.1.4 35 encapsulation mpls
```

The **802.1Q VLAN ID** (`35` in the subinterface encapsulation) identifies the local Ethernet service; the **VC ID** (`35` in `xconnect`) identifies the pseudowire. They are different concepts and need **not** have the same numeric value. VLAN-tag preservation or removal depends on the negotiated pseudowire type and platform behavior.

## Common Problems

| Symptom | Likely cause |
|---|---|
| Remote PE not reachable | Missing route to remote endpoint; transport interface down. |
| LDP not established | MPLS/LDP not enabled where required, or LDP ID/reachability problem. |
| VC DOWN | Incorrect peer/VC ID, unsupported PW type, attachment circuit down, label-signaling failure, or incompatible parameters. |
| VC UP, no customer traffic | MTU/drop issues, CE VLAN mismatch, incorrect AC, or platform forwarding limitations. |
| VC UP, OSPF not FULL | CE OSPF mismatch or customer multicast/control frames not passing correctly. |

## Key Points

- **EoMPLS is a point-to-point Layer 2 service over an MPLS network**, not an MPLS L3VPN.
- **The attachment circuit** is the local CE-to-PE connection; **the pseudowire** links the two PEs.
- The **VC ID must match** at both endpoints; the allocated **pseudowire labels do not**.
- An EoMPLS packet usually carries a **transport label** and **pseudowire label**, with PHP possibly removing the transport label.
- `xconnect <remote-PE-address> <vc-id> encapsulation mpls` is the classic IOS XE configuration command.
- PEs need **IP reachability, MPLS transport, and pseudowire signaling**; CEs just see an Ethernet link.
- Check both **transport LDP** and **pseudowire VC status** before troubleshooting a customer OSPF adjacency.

## Cisco References

- [MPLS Configuration Guide, Cisco IOS XE 17.x — Any Transport over MPLS](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/mpls/b-mpls/m_mp-any-transport-xe.html)
- [MPLS Configuration Guide, Cisco IOS XE 17.x — L2VPN Protocol-Based CLIs](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/mpls/b-mpls/m_l2vpn-prot-based.html)
- [Cisco IOS XE 17.18.x — Configuring Ethernet-over-MPLS (EoMPLS)](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9400/software/release/17-18/configuration_guide/mpls/b_1718_mpls_9400_cg/configuring_ethernet_over_mpls__eompls_.html)
