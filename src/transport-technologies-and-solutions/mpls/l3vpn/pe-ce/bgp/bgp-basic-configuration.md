# Basic PE-CE BGP Configuration

This page shows a basic **PE-CE BGP** configuration for an MPLS L3VPN.

The goal is simple:

```text
CE1 LAN 10.10.10.0/24 should reach CE2 LAN 11.11.11.0/24.
CE2 LAN 11.11.11.0/24 should reach CE1 LAN 10.10.10.0/24.
```

The customer uses BGP between the CE routers and the provider PE routers.

> This page only covers the absolute basics of PE-CE BGP configuration.
>
> More specific configurations are covered in other pages of this section.
>
> General BGP configuration (not specific to L3VPN) is covered in the BGP section of this book.

## Topology

```text
             MPLS Provider Core
        MP-BGP VPNv4 + MPLS forwarding

        PE1 ================= PE2
         |                     |
      EBGP PE-CE           EBGP PE-CE
         |                     |
        CE1                   CE2
         |                     |
   10.10.10.0/24         11.11.11.0/24
```

## Addressing

```text
PE1-CE1 link: 192.168.1.0/24
PE1:          192.168.1.1
CE1:          192.168.1.2

PE2-CE2 link: 192.168.2.0/24
PE2:          192.168.2.1
CE2:          192.168.2.2

CE1 LAN:      10.10.10.0/24
CE2 LAN:      11.11.11.0/24
```

## AS Numbers

```text
Provider AS: 65000

CE1 AS:      65100
CE2 AS:      65200
```

Each CE site uses a different customer AS.

This is the simplest PE-CE BGP design.

```text
CE1 AS 65100 --- PE1 AS 65000 --- MPLS VPN --- PE2 AS 65000 --- CE2 AS 65200
```

## VRF Configuration on the PE Routers

The PE routers need a VRF for the customer.

### PE1

```text
vrf definition CUSTOMER_A
 rd 65000:1
 address-family ipv4
  route-target export 1:1
  route-target import 1:1
```

Apply the VRF to the PE-CE interface.

```text
interface Ethernet01
 description PE1-to-CE1
 vrf forwarding CUSTOMER_A
 ip address 192.168.1.1 255.255.255.0
 no shutdown
```

### PE2

```text
vrf definition CUSTOMER_A
 rd 65000:1
 address-family ipv4
  route-target export 1:1
  route-target import 1:1
```

Apply the VRF to the PE-CE interface.

```text
interface Ethernet0/1
 description PE2-to-CE2
 vrf forwarding CUSTOMER_A
 ip address 192.168.20.1 255.255.255.0
 no shutdown
```

## PE-CE BGP Configuration

PE-CE BGP is configured under the VRF address family.

The PE router has one global BGP process.

The CE neighbor is configured under:

```text
address-family ipv4 vrf CUSTOMER_A
```

### PE1

```text
router bgp 65000
 address-family ipv4 vrf CUSTOMER_A
  neighbor 192.168.1.2 remote-as 65100
  neighbor 192.168.1.2 activate
```

PE1 peers with CE1 inside VRF `CUSTOMER_A`.

Routes learned from CE1 are installed in the VRF routing table.

They are also converted into VPNv4 routes and advertised to the other PE routers by MP-BGP.

### PE2

```text
router bgp 65000
 address-family ipv4 vrf CUSTOMER_A
  neighbor 192.168.2.2 remote-as 65200
  neighbor 192.168.2.2 activate
```

PE2 peers with CE2 inside VRF `CUSTOMER_A`.

Routes learned from CE2 are installed in the VRF routing table.

They are also converted into VPNv4 routes and advertised to the other PE routers by MP-BGP.

> PE-CE BGP is somewhat simpler than OSPF or EIGRP because there is no need for redistribution on the PE routers.
>
> BGP routes learned from the CE routers are automatically exported as VPNv4 routes.

## CE Router Configuration

The CE routers run normal IPv4 BGP.

- They do not need VRF configuration.
- They do not run MP-BGP.
- They do not need to know about MPLS labels, route distinguishers, or route targets.

### CE1

```text
interface Ethernet0/0
 description CE1-to-PE1
 ip address 192.168.1.2 255.255.255.0
 no shutdown

interface Loopback0
 description CE1-LAN
 ip address 10.10.10.1 255.255.255.0
 no shutdown
```

```text
router bgp 65100
 neighbor 192.168.1.1 remote-as 65000
 network 10.10.10.0 mask 255.255.255.0
```

CE1 advertises its LAN route, `10.10.10.0/24`, to PE1.

### CE2

```text
interface Ethernet0/0
 description CE2-to-PE2
 ip address 192.168.2.2 255.255.255.0
 no shutdown

interface Loopback0
 description CE2-LAN
 ip address 11.11.11.1 255.255.255.0
 no shutdown
```

```text
router bgp 65200
 neighbor 192.168.2.1 remote-as 65000
 network 11.11.11.0 mask 255.255.255.0
```

CE2 advertises its LAN route, `11.11.11.0/24`, to PE2.

## Control Plane Process

The control plane works like this:

```text
1. CE1 advertises 10.10.10.0/24 to PE1 using EBGP.

2. PE1 installs 10.10.10.0/24 in the CUSTOMER_A VRF.

3. PE1 converts the route into a VPNv4 route.

4. PE1 advertises the VPNv4 route to PE2 using MP-BGP.

5. PE2 imports the route into the CUSTOMER_A VRF.

6. PE2 advertises 10.10.10.0/24 to CE2 using eBGP.
```

The same process happens in the opposite direction for CE2's LAN.

```text
1. CE2 advertises 11.11.11.0/24 to PE2 using EBGP.

2. PE2 installs 11.11.11.0/24 in the CUSTOMER_A VRF.

3. PE2 converts the route into a VPNv4 route.

4. PE2 advertises the VPNv4 route to PE1 using MP-BGP.

5. PE1 imports the route into the CUSTOMER_A VRF.

6. PE1 advertises 11.11.11.0/24 to CE1 using eBGP.
```

## Data Plane Process

The data plane uses MPLS inside the provider core.

For example, traffic from CE1 to CE2 follows this path:

```text
CE1 sends packet to PE1.

Source:      10.10.10.1
Destination: 11.11.11.1
```

PE1 performs a VRF lookup.

PE1 sees that `11.11.11.0/24` is reachable through PE2.

PE1 pushes the required MPLS labels and forwards the packet through the provider core.

```text
CE1 -> PE1 -> MPLS core -> PE2 -> CE2
```

CE1 and CE2 do not see the MPLS labels.

MPLS is only used inside the provider network.

## Verification on the PE Routers

### Check the PE-CE BGP Neighbor

On PE1:

```text
PE1# show bgp vrf CUSTOMER_A vpnv4 unicast summary 

Neighbor        V           AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
192.168.1.2     4        65100       9       9        3    0    0 00:04:19        1
```

On PE2:

```text
PE2# show bgp vrf CUSTOMER_A vpnv4 unicast summary 

Neighbor        V           AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
192.168.2.2     4        65200       5       5        3    0    0 00:00:16        1
```

## Check the VRF BGP Table

On PE1:

```text
PE1# show bgp vrf CUSTOMER_A vpnv4 unicast 

     Network          Next Hop            Metric LocPrf Weight Path
Route Distinguisher: 65000:1 (default for vrf CUSTOMER_A)
 *>   10.10.10.0/24    192.168.1.2              0             0 65100 i
 *>i  11.11.11.0/24    4.4.4.4                  0    100      0 65200 i
```

PE1 should have both customer LAN routes.

```text
10.10.10.0/24 is learned from CE1.
11.11.11.0/24 is learned through the MPLS VPN.
```

On PE2:

```text
PE2# show bgp vrf CUSTOMER_A vpnv4 unicast                                  

     Network          Next Hop            Metric LocPrf Weight Path
Route Distinguisher: 65000:1 (default for vrf CUSTOMER_A)
 *>i  10.10.10.0/24    1.1.1.1                  0    100      0 65100 i
 *>   11.11.11.0/24    192.168.2.2              0             0 65200 i
```

PE2 should also have both customer LAN routes.

```text
10.10.10.0/24 is learned through the MPLS VPN.
11.11.11.0/24 is learned from CE2.
```

## Check the VRF Routing Table

On PE1:

```text
PE1# show ip route vrf CUSTOMER_A

      10.0.0.0/24 is subnetted, 1 subnets
B        10.10.10.0 [20/0] via 192.168.1.2, 00:12:04
      11.0.0.0/24 is subnetted, 1 subnets
B        11.11.11.0 [200/0] via 4.4.4.4, 00:01:56
```

On PE2:

```text
PE2# show ip route vrf CUSTOMER_A

      10.0.0.0/24 is subnetted, 1 subnets
B        10.10.10.0 [200/0] via 1.1.1.1, 00:02:15
      11.0.0.0/24 is subnetted, 1 subnets
B        11.11.11.0 [20/0] via 192.168.2.2, 00:07:08
```

The route from the local CE is an EBGP route.

The route from the remote PE is an IBGP VPN route.

## Verification on the CE Routers

On CE1:

```text
CE1# show ip bgp

     Network          Next Hop            Metric LocPrf Weight Path
 *>   10.10.10.0/24    0.0.0.0                  0         32768 i
 *>   11.11.11.0/24    192.168.1.1                            0 65000 65200 i
```

CE1 should learn CE2's LAN route.

On CE2:

```text
CE2# show ip bgp

     Network          Next Hop            Metric LocPrf Weight Path
 *>   10.10.10.0/24    192.168.2.1                            0 65000 65100 i
 *>   11.11.11.0/24    0.0.0.0                  0         32768 i
```

CE2 should learn CE1's LAN route.

## Ping Test

From CE1, ping CE2's LAN interface.

```text
CE1# ping 11.11.11.1 source l0
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 11.11.11.1, timeout is 2 seconds:
Packet sent with a source address of 10.10.10.1 
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/2/3 ms
```

Success! As you can see, PE-CE BGP configuration is fairly simple.

But there are a few important considerations that will be covered in the following pages.

---

## Basic PE-CE BGP Summary

PE-CE BGP is used to exchange customer routes between the CE and PE routers.

The CE router runs normal IPv4 BGP.

The PE router runs BGP inside the customer VRF.

```text
CE side:
router bgp <customer-as>
 neighbor <pe-address> remote-as <provider-as>
 network <customer-lan> mask <mask>

PE side:
router bgp <provider-as>
 address-family ipv4 vrf <vrf-name>
  neighbor <ce-address> remote-as <customer-as>
  neighbor <ce-address> activate
```

The PE router is responsible for converting the customer IPv4 route into a VPNv4 route.

The CE router does not need to know anything about MPLS VPN internals.

## Key Points

* PE-CE BGP is normal IPv4 BGP between the PE and CE.
* The CE does not run MPLS.
* The CE does not run MP-BGP.
* The PE-CE neighbor is configured under the VRF address family on the PE.
* The CE router is configured with normal IPv4 BGP.
* The PE converts customer IPv4 routes into VPNv4 routes.
* MP-BGP carries the VPNv4 routes between PE routers.
* MPLS labels are used only inside the provider core.
