# Basic L3VPN Configuration

This page shows a basic MPLS L3VPN configuration using the same topology from the previous examples.

The goal is to bring together the pieces covered so far:

```text
VRFs
RDs and RTs
MP-BGP VPNv4
VPN labels
L3VPN packet forwarding
MPLS MTU
Route import and export
```

It uses:

```text
OSPF in the provider core
LDP for transport labels
MP-BGP VPNv4 between PE1 and PE2
Static PE-CE routing
One customer VRF: CUSTOMER_A
```

---

## Topology

```text
Customer A Site 1                    Customer A Site 2

10.10.10.0/24                        11.11.11.0/24
      |                                      |
     CE1                                    CE2
      |                                      |
     PE1 -------- P1 --------- P2 --------- PE2
    1.1.1.1     2.2.2.2      3.3.3.3      4.4.4.4
```

The provider core is:

```text
PE1 --- P1 --- P2 --- PE2
```

The customer-facing links are:

```text
CE1 --- PE1
CE2 --- PE2
```

CE routers do not run MPLS.

P routers do not have VRFs or customer routes.

PE routers have both:

```text
Global routing table:
Provider core routes.

VRF CUSTOMER_A:
Customer routes.
```

---

## Addressing

### Provider Core

| Device | Interface | IP Address | Description |
| :--- | :--- | :--- | :--- |
| PE1 | Loopback0 | 1.1.1.1/32 | PE1 router ID / BGP update source |
| P1 | Loopback0 | 2.2.2.2/32 | P1 router ID |
| P2 | Loopback0 | 3.3.3.3/32 | P2 router ID |
| PE2 | Loopback0 | 4.4.4.4/32 | PE2 router ID / BGP update source |
| PE1 | Ethernet0/0 | 10.1.1.1/24 | PE1 to P1 |
| P1 | Ethernet0/0 | 10.1.1.2/24 | P1 to PE1 |
| P1 | Ethernet0/1 | 10.1.2.1/24 | P1 to P2 |
| P2 | Ethernet0/0 | 10.1.2.2/24 | P2 to P1 |
| P2 | Ethernet0/1 | 10.2.2.1/24 | P2 to PE2 |
| PE2 | Ethernet0/0 | 10.2.2.2/24 | PE2 to P2 |

### Customer Links

| Device | Interface | IP Address | Description |
| :--- | :--- | :--- | :--- |
| CE1 | Loopback0 | 10.10.10.1/24 | Customer A Site 1 LAN |
| CE1 | Ethernet0/0 | 192.168.1.2/24 | CE1 to PE1 |
| PE1 | Ethernet0/1 | 192.168.1.1/24 | PE1 to CE1 in VRF CUSTOMER_A |
| CE2 | Loopback0 | 11.11.11.1/24 | Customer A Site 2 LAN |
| CE2 | Ethernet0/0 | 192.168.2.2/24 | CE2 to PE2 |
| PE2 | Ethernet0/1 | 192.168.2.1/24 | PE2 to CE2 in VRF CUSTOMER_A |

---

## Configuration Order

A clean order is:

```text
1. Configure core interface IP addresses.
2. Configure the provider IGP.
3. Enable MPLS and LDP in the core.
4. Configure MPLS MTU on core-facing interfaces.
5. Configure the customer VRFs on PE routers.
6. Place PE-CE interfaces into the VRF.
7. Configure PE-CE routing.
8. Configure MP-BGP VPNv4 between PE routers.
9. Redistribute customer routes into BGP under the VRF address family.
10. Verify end-to-end connectivity.
```

In this basic example, the PE-CE routing is static.

This keeps the focus on the MPLS L3VPN itself.

---

## CE1 Configuration

CE1 is a normal customer router.

It does not run MPLS.

It only needs:

```text
Customer LAN interface
Link to PE1
Route to the remote customer site
```

```text
hostname CE1
!
interface Loopback0
 ip address 10.10.10.1 255.255.255.0
!
interface Ethernet0/0
 description Link to PE1
 ip address 192.168.1.2 255.255.255.0
 no shutdown
!
ip route 11.11.11.0 255.255.255.0 192.168.1.1
```

CE1 sends traffic for `11.11.11.0/24` to PE1.

CE1 does not know that the provider uses MPLS.

---

## PE1 Configuration

PE1 has three main roles:

```text
1. Participate in the provider core.
2. Maintain the CUSTOMER_A VRF.
3. Exchange VPNv4 routes with PE2.
```

### PE1 Core Interfaces and IGP

```text
hostname PE1
!
ip cef
!
interface Loopback0
 ip address 1.1.1.1 255.255.255.255
!
interface Ethernet0/0
 description Core link to P1
 ip address 10.1.1.1 255.255.255.0
 mpls mtu override 1508
 mpls ip
 no shutdown
!
router ospf 1
 router-id 1.1.1.1
 network 1.1.1.1 0.0.0.0 area 0
 network 10.1.1.0 0.0.0.255 area 0
```

The provider IGP must advertise PE1's loopback.

That loopback will be used as the VPNv4 BGP next hop.

### PE1 LDP

```text
mpls ldp router-id Loopback0 force
```

LDP provides the transport labels to reach remote PE loopbacks.

PE1 needs a transport label to reach PE2's loopback `4.4.4.4`.

### PE1 VRF

```text
vrf definition CUSTOMER_A
 rd 65000:1
 !
 address-family ipv4
  route-target export 1:1
  route-target import 1:1
 exit-address-family
```

This creates the customer VRF.

```text
RD 65000:1:
Makes CUSTOMER_A routes unique in VPNv4.

Export RT 1:1:
Attached to routes exported from this VRF.

Import RT 1:1:
Imports received VPNv4 routes that carry RT 1:1.
```

### PE1 Customer-Facing Interface

```text
interface Ethernet0/1
 description Link to CE1
 vrf forwarding CUSTOMER_A
 ip address 192.168.1.1 255.255.255.0
 no shutdown
```

`vrf forwarding` must be configured before the IP address.

This interface belongs to `CUSTOMER_A`, not the global routing table.

### PE1 Static Route to CE1 LAN

```text
ip route vrf CUSTOMER_A 10.10.10.0 255.255.255.0 192.168.1.2
```

This installs the customer route into the `CUSTOMER_A` VRF.

But by itself, this does not export the route into VPNv4.

The route must be redistributed into BGP under the VRF address family.

### PE1 MP-BGP VPNv4

```text
router bgp 65000
 bgp log-neighbor-changes
 no bgp default ipv4-unicast
 neighbor 4.4.4.4 remote-as 65000
 neighbor 4.4.4.4 update-source Loopback0
 !
 address-family vpnv4
  neighbor 4.4.4.4 activate
  neighbor 4.4.4.4 send-community extended
 exit-address-family
 !
 address-family ipv4 vrf CUSTOMER_A
  redistribute static
 exit-address-family
```

The VPNv4 address family exchanges VPN routes between PE routers.

The IPv4 VRF address family controls routes inside `CUSTOMER_A`.

`redistribute static` takes the static route from the VRF routing table and places it into BGP for the VRF, making it eligible for VPNv4 export.

---

## P1 Configuration

P1 is a provider core router.

It does not have:

```text
VRFs
MP-BGP VPNv4
Customer routes
RDs
RTs
```

It only needs:

```text
Core IP reachability
MPLS forwarding
LDP
```

```text
hostname P1
!
ip cef
!
interface Loopback0
 ip address 2.2.2.2 255.255.255.255
!
interface Ethernet0/0
 description Core link to PE1
 ip address 10.1.1.2 255.255.255.0
 mpls mtu override 1508
 mpls ip
 no shutdown
!
interface Ethernet0/1
 description Core link to P2
 ip address 10.1.2.1 255.255.255.0
 mpls mtu override 1508
 mpls ip
 no shutdown
!
mpls ldp router-id Loopback0 force
!
router ospf 1
 router-id 2.2.2.2
 network 2.2.2.2 0.0.0.0 area 0
 network 10.1.1.0 0.0.0.255 area 0
 network 10.1.2.0 0.0.0.255 area 0
```

P1 forwards labeled packets based on the top transport label.

It does not know about `10.10.10.0/24` or `11.11.11.0/24`.

---

## P2 Configuration

P2 is also a provider core router.

```text
hostname P2
!
ip cef
!
interface Loopback0
 ip address 3.3.3.3 255.255.255.255
!
interface Ethernet0/0
 description Core link to P1
 ip address 10.1.2.2 255.255.255.0
 mpls mtu override 1508
 mpls ip
 no shutdown
!
interface Ethernet0/1
 description Core link to PE2
 ip address 10.2.2.1 255.255.255.0
 mpls mtu override 1508
 mpls ip
 no shutdown
!
mpls ldp router-id Loopback0 force
!
router ospf 1
 router-id 3.3.3.3
 network 3.3.3.3 0.0.0.0 area 0
 network 10.1.2.0 0.0.0.255 area 0
 network 10.2.2.0 0.0.0.255 area 0
```

---

## PE2 Configuration

PE2 mirrors PE1.

### PE2 Core Interfaces and IGP

```text
hostname PE2
!
ip cef
!
interface Loopback0
 ip address 4.4.4.4 255.255.255.255
!
interface Ethernet0/0
 description Core link to P2
 ip address 10.2.2.2 255.255.255.0
 mpls mtu override 1508
 mpls ip
 no shutdown
!
router ospf 1
 router-id 4.4.4.4
 network 4.4.4.4 0.0.0.0 area 0
 network 10.2.2.0 0.0.0.255 area 0
```

### PE2 LDP

```text
mpls ldp router-id Loopback0 force
```

### PE2 VRF

```text
vrf definition CUSTOMER_A
 rd 65000:1
 !
 address-family ipv4
  route-target export 1:1
  route-target import 1:1
 exit-address-family
```

This is the same simple full-mesh customer VPN design.

Both PEs export and import RT `1:1`.

### PE2 Customer-Facing Interface

```text
interface Ethernet0/1
 description Link to CE2
 vrf forwarding CUSTOMER_A
 ip address 192.168.2.1 255.255.255.0
 no shutdown
```

### PE2 Static Route to CE2 LAN

```text
ip route vrf CUSTOMER_A 11.11.11.0 255.255.255.0 192.168.2.2
```

### PE2 MP-BGP VPNv4

```text
router bgp 65000
 bgp log-neighbor-changes
 no bgp default ipv4-unicast
 neighbor 1.1.1.1 remote-as 65000
 neighbor 1.1.1.1 update-source Loopback0
 !
 address-family vpnv4
  neighbor 1.1.1.1 activate
  neighbor 1.1.1.1 send-community extended
 exit-address-family
 !
 address-family ipv4 vrf CUSTOMER_A
  redistribute static
 exit-address-family
```

PE2 exports `11.11.11.0/24` into VPNv4.

PE1 imports it into `CUSTOMER_A` because the route carries RT `1:1`.

---

## CE2 Configuration

CE2 is a normal customer router.

```text
hostname CE2
!
interface Loopback0
 ip address 11.11.11.1 255.255.255.0
!
interface Ethernet0/0
 description Link to PE2
 ip address 192.168.2.2 255.255.255.0
 no shutdown
!
ip route 10.10.10.0 255.255.255.0 192.168.2.1
```

CE2 sends traffic for `10.10.10.0/24` to PE2.

---

## Complete PE1 Configuration

For reference, here is the full PE1 configuration together.

```text
hostname PE1
!
ip cef
!
vrf definition CUSTOMER_A
 rd 65000:1
 !
 address-family ipv4
  route-target export 1:1
  route-target import 1:1
 exit-address-family
!
interface Loopback0
 ip address 1.1.1.1 255.255.255.255
!
interface Ethernet0/0
 description Core link to P1
 ip address 10.1.1.1 255.255.255.0
 mpls mtu override 1508
 mpls ip
 no shutdown
!
interface Ethernet0/1
 description Link to CE1
 vrf forwarding CUSTOMER_A
 ip address 192.168.1.1 255.255.255.0
 no shutdown
!
mpls ldp router-id Loopback0 force
!
router ospf 1
 router-id 1.1.1.1
 network 1.1.1.1 0.0.0.0 area 0
 network 10.1.1.0 0.0.0.255 area 0
!
ip route vrf CUSTOMER_A 10.10.10.0 255.255.255.0 192.168.1.2
!
router bgp 65000
 bgp log-neighbor-changes
 no bgp default ipv4-unicast
 neighbor 4.4.4.4 remote-as 65000
 neighbor 4.4.4.4 update-source Loopback0
 !
 address-family vpnv4
  neighbor 4.4.4.4 activate
  neighbor 4.4.4.4 send-community extended
 exit-address-family
 !
 address-family ipv4 vrf CUSTOMER_A
  redistribute static
 exit-address-family
```

---

## Complete PE2 Configuration

```text
hostname PE2
!
ip cef
!
vrf definition CUSTOMER_A
 rd 65000:1
 !
 address-family ipv4
  route-target export 1:1
  route-target import 1:1
 exit-address-family
!
interface Loopback0
 ip address 4.4.4.4 255.255.255.255
!
interface Ethernet0/0
 description Core link to P2
 ip address 10.2.2.2 255.255.255.0
 mpls mtu override 1508
 mpls ip
 no shutdown
!
interface Ethernet0/1
 description Link to CE2
 vrf forwarding CUSTOMER_A
 ip address 192.168.2.1 255.255.255.0
 no shutdown
!
mpls ldp router-id Loopback0 force
!
router ospf 1
 router-id 4.4.4.4
 network 4.4.4.4 0.0.0.0 area 0
 network 10.2.2.0 0.0.0.255 area 0
!
ip route vrf CUSTOMER_A 11.11.11.0 255.255.255.0 192.168.2.2
!
router bgp 65000
 bgp log-neighbor-changes
 no bgp default ipv4-unicast
 neighbor 1.1.1.1 remote-as 65000
 neighbor 1.1.1.1 update-source Loopback0
 !
 address-family vpnv4
  neighbor 1.1.1.1 activate
  neighbor 1.1.1.1 send-community extended
 exit-address-family
 !
 address-family ipv4 vrf CUSTOMER_A
  redistribute static
 exit-address-family
```

---

## What Each Protocol Does

| Component | Where | Purpose |
| :--- | :--- | :--- |
| OSPF | PE/P core | Provides reachability to PE loopbacks |
| LDP | PE/P core | Provides transport labels to PE loopbacks |
| VRF | PE routers | Separates customer routes |
| Static routes | PE-CE | Installs customer LAN routes into the VRF |
| MP-BGP VPNv4 | PE to PE | Exchanges VPN routes and VPN labels |
| RT import/export | PE VRF | Controls which VPN routes enter and leave the VRF |
| MPLS MTU | Core links | Allows labeled packets to fit across the core |

---

## Verification Step 1: Provider IGP

Check OSPF neighbors in the core.

PE1 should have an OSPF neighbor with P1:

```text
PE1# show ip ospf neighbor

Neighbor ID     Pri   State           Dead Time   Address         Interface
2.2.2.2           0   FULL/  -        00:00:35    10.1.1.2        Ethernet0/0
```

PE2 should have an OSPF neighbor with P2:

```text
PE2# show ip ospf neighbor

Neighbor ID     Pri   State           Dead Time   Address         Interface
3.3.3.3           0   FULL/  -        00:00:36    10.2.2.1        Ethernet0/0
```

Check that PE loopbacks are reachable in the global table.

On PE1:

```text
PE1# show ip route 4.4.4.4
Routing entry for 4.4.4.4/32
  Known via "ospf 1", distance 110, metric 31, type intra area
  Last update from 10.1.1.2 on Ethernet0/0, 01:36:48 ago
  Routing Descriptor Blocks:
  * 10.1.1.2, from 4.4.4.4, 01:36:48 ago, via Ethernet0/0
      Route metric is 31, traffic share count is 1

PE1# ping 4.4.4.4
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 4.4.4.4, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/1/2 ms
```

On PE2:

```text
PE2# show ip route 1.1.1.1
Routing entry for 1.1.1.1/32
  Known via "ospf 1", distance 110, metric 31, type intra area
  Last update from 10.2.2.1 on Ethernet0/0, 01:37:03 ago
  Routing Descriptor Blocks:
  * 10.2.2.1, from 1.1.1.1, 01:37:03 ago, via Ethernet0/0
      Route metric is 31, traffic share count is 1

PE2# ping 1.1.1.1
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 1.1.1.1, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/2/3 ms
```

This must work before MP-BGP VPNv4 can work reliably.

---

## Verification Step 2: LDP and Transport Labels

Check LDP neighbors.

```text
PE1# show mpls ldp neighbor 
    Peer LDP Ident: 2.2.2.2:0; Local LDP Ident 1.1.1.1:0
        TCP connection: 2.2.2.2.51251 - 1.1.1.1.646
        State: Oper; Msgs sent/rcvd: 122/121; Downstream
        Up time: 01:37:36
        LDP discovery sources:
          Ethernet0/0, Src IP addr: 10.1.1.2
        Addresses bound to peer LDP Ident:
          10.1.1.2        10.1.2.1        2.2.2.2  
```

Check that PE1 has a label path to PE2's loopback.

```text
PE1# show mpls forwarding-table 4.4.4.4
Local      Outgoing   Prefix           Bytes Label   Outgoing   Next Hop    
Label      Label      or Tunnel Id     Switched      interface              
16         17         4.4.4.4/32       0             Et0/0      10.1.1.2    
```

Check the reverse direction on PE2.

```text
PE2# show mpls forwarding-table 1.1.1.1
Local      Outgoing   Prefix           Bytes Label   Outgoing   Next Hop    
Label      Label      or Tunnel Id     Switched      interface              
19         18         1.1.1.1/32       0             Et0/0      10.2.2.1  
```

If there is no transport label to the remote PE loopback, the L3VPN data plane will fail.

---

## Verification Step 3: VPNv4 BGP Session

Check the VPNv4 BGP session.

```text
PE1# show bgp vpnv4 unicast all summary
BGP router identifier 1.1.1.1, local AS number 65000
BGP table version is 14, main routing table version 14
2 network entries using 528 bytes of memory
2 path entries using 272 bytes of memory
2/2 BGP path/bestpath attribute entries using 624 bytes of memory
1 BGP extended community entries using 24 bytes of memory
0 BGP route-map cache entries using 0 bytes of memory
0 BGP filter-list cache entries using 0 bytes of memory
BGP using 1448 total bytes of memory
BGP activity 6/4 prefixes, 6/4 paths, scan interval 60 secs
6 networks peaked at 21:35:16 Jun 16 2026 UTC (01:23:50.429 ago)

Neighbor        V           AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
4.4.4.4         4        65000     122     119       14    0    0 01:38:12        1
```

On PE2:

```text
PE2# show bgp vpnv4 unicast all summary
BGP router identifier 4.4.4.4, local AS number 65000
BGP table version is 35, main routing table version 35
2 network entries using 528 bytes of memory
2 path entries using 272 bytes of memory
2/2 BGP path/bestpath attribute entries using 624 bytes of memory
1 BGP extended community entries using 24 bytes of memory
0 BGP route-map cache entries using 0 bytes of memory
0 BGP filter-list cache entries using 0 bytes of memory
BGP using 1448 total bytes of memory
BGP activity 12/10 prefixes, 12/10 paths, scan interval 60 secs
6 networks peaked at 21:35:16 Jun 16 2026 UTC (01:24:08.736 ago)

Neighbor        V           AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
1.1.1.1         4        65000     120     123       35    0    0 01:38:30        1
```

---

## Verification Step 4: VRF Routes

PE1 should have:

```text
Local static route:
10.10.10.0/24 via 192.168.1.2

Remote BGP VPN route:
11.11.11.0/24 via 4.4.4.4
```

Check PE1:

```text
PE1# show ip route vrf CUSTOMER_A

      10.0.0.0/24 is subnetted, 1 subnets
S        10.10.10.0 [1/0] via 192.168.1.2
      11.0.0.0/24 is subnetted, 1 subnets
B        11.11.11.0 [200/0] via 4.4.4.4, 01:38:54
      192.168.1.0/24 is variably subnetted, 2 subnets, 2 masks
C        192.168.1.0/24 is directly connected, Ethernet0/1
L        192.168.1.1/32 is directly connected, Ethernet0/1
```

PE2 should have the reverse:

```text
PE2#show ip route vrf CUSTOMER_A

      10.0.0.0/24 is subnetted, 1 subnets
B        10.10.10.0 [200/0] via 1.1.1.1, 00:22:47
      11.0.0.0/24 is subnetted, 1 subnets
S        11.11.11.0 [1/0] via 192.168.2.2
      192.168.2.0/24 is variably subnetted, 2 subnets, 2 masks
C        192.168.2.0/24 is directly connected, Ethernet0/1
L        192.168.2.1/32 is directly connected, Ethernet0/1
```

---

## Verification Step 5: VPNv4 Routes and Labels

Check VPNv4 routes and labels.

On PE1:

```text
PE1# show bgp vpnv4 unicast all labels
   Network          Next Hop      In label/Out label
Route Distinguisher: 65000:1 (CUSTOMER_A)
   10.10.10.0/24    192.168.1.2     21/nolabel
   11.11.11.0/24    4.4.4.4         nolabel/21
```

From PE1's point of view:

```text
10.10.10.0/24:
Local route.
PE1 assigned an in label and advertises it to PE2.

11.11.11.0/24:
Remote route.
PE1 learned the out label from PE2.
```

The exact labels will vary. The important point is that the remote route has an out label.

---

## Verification Step 6: CEF Label Stack

Check the actual label stack PE1 imposes for traffic to CE2.

```text
PE1# show ip cef vrf CUSTOMER_A 11.11.11.0/24 detail
11.11.11.0/24, epoch 0, flags [rib defined all labels]
  recursive via 4.4.4.4 label 21
    nexthop 10.1.1.2 Ethernet0/0 label 17-(local:16)
```

This means PE1 imposes:

```text
Top label:    17
Bottom label: 21
```

The top label gets the packet to PE2.

The bottom label tells PE2 how to forward the packet in the VPN.

---

## Verification Step 7: End-to-End Ping

From CE1, ping CE2's loopback.

```text
CE1# ping 11.11.11.1 source Loopback0 size 1500 df-bit 
Type escape sequence to abort.
Sending 5, 1500-byte ICMP Echos to 11.11.11.1, timeout is 2 seconds:
Packet sent with a source address of 10.10.10.1 
Packet sent with the DF bit set
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/2/7 ms
```

---

## Key Points

* CE routers do not run MPLS in a normal MPLS L3VPN.
* P routers do not have VRFs, MP-BGP VPNv4, or customer routes.
* PE routers connect the customer VRF to the MPLS VPN control plane.
* The provider IGP must advertise PE loopbacks.
* LDP must provide transport labels to PE loopbacks.
* MP-BGP VPNv4 exchanges customer VPN routes and VPN labels between PEs.
* `route-target export` controls which RT is attached to exported routes.
* `route-target import` controls which received VPNv4 routes enter the VRF.
* Static PE-CE routes must be redistributed under `address-family ipv4 vrf`.
* The ingress PE imposes the transport label and VPN label.
* The P routers switch using only the top transport label.
* The egress PE uses the VPN label to forward the packet toward the CE.
* Verify the control plane first, then verify the label stack, then test end-to-end traffic.
