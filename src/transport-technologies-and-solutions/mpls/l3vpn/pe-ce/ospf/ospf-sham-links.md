# OSPF Sham Links

An **OSPF sham link** is used in MPLS L3VPN PE-CE OSPF designs when customer sites have both:

```text
1. An MPLS VPN path through the provider network
2. A direct OSPF backdoor link between customer sites
```

Example:

```text
            MPLS L3VPN
        PE1 ========== PE2
         |              |
        CE1 ---------- CE2
          Backdoor link
```

The sham link makes the MPLS VPN path appear as an intra-area OSPF path.

This allows OSPF to compare the MPLS VPN path and the backdoor path using normal OSPF cost.

Without a sham link, OSPF route type preference can make the backdoor link preferred even when its cost is worse.

---

## The Problem Sham Links Solve

Assume CE1 and CE2 are in the same OSPF area.

```text
CE1 --- PE1 === MPLS VPN === PE2 --- CE2
```

Without a backdoor link, routes can be exchanged through the MPLS VPN.

Example:

```text
CE1 LAN:
10.10.10.0/24

CE2 LAN:
11.11.11.0/24
```

CE2 may learn CE1's LAN as an inter-area route:

```text
CE2# show ip route ospf
O IA     10.10.10.0 [110/21] via 192.168.2.1, 00:00:01, Ethernet0/0
```

That is normal PE-CE OSPF behavior across the MPLS VPN superbackbone.

> Notice the cost of `21`.

Now add a direct OSPF backdoor link between CE1 and CE2:

```text
        MPLS VPN path
CE1 --- PE1 === PE2 --- CE2
 |                      |
 +----- backdoor -------+


CE1# show run int e0/1
!
interface Ethernet0/1
 ip address 12.12.12.1 255.255.255.0
 ip ospf network point-to-point
 ip ospf 1 area 0
 ip ospf cost 100

CE2# show run int e0/1
!
interface Ethernet0/1
 ip address 12.12.12.2 255.255.255.0
 ip ospf network point-to-point
 ip ospf 1 area 0
 ip ospf cost 100
```

CE2 now learns CE1's LAN as an intra-area route over the backdoor link:

```text
CE2# show ip route ospf
O        10.10.10.0 [110/101] via 12.12.12.1, 00:00:00, Ethernet0/1
```

OSPF prefers the backdoor link, even though the cost is `101` (vs `21`).

That is due to OSPF's route type preference:

```
Intra-area > inter-area > external
```

So CE2 prefers the backdoor path even if the MPLS VPN path has a lower metric.

This is the key issue.

```text
Without sham link:
MPLS VPN path appears as O IA.
Backdoor path appears as O.
OSPF prefers the backdoor path because intra-area beats inter-area.
```

A sham link fixes this by making the MPLS VPN path appear as an intra-area path too.

---

## OSPF Route Preference Problem

OSPF route selection prefers route types in this order:

```text
1. Intra-area routes
2. Inter-area routes
3. External type 1 routes
4. External type 2 routes
```

So this route is preferred:

```text
O  10.10.10.0 [110/101]
```

over this route:

```text
O IA  10.10.10.0 [110/21]
```

Even though the `O IA` path has a better cost, CE2 still prefers the intra-area route:

The route type wins before the metric comparison.

A sham link changes the MPLS VPN path from an inter-area route to an intra-area route.

Then OSPF can compare the two paths by cost.

---

## What a Sham Link Does

A sham link is a logical OSPF link between two PE routers.

It is configured inside the customer VRF OSPF process.

Example:

```text
PE1 --- sham link --- PE2
```

The sham link belongs to a customer OSPF area.

Example:

```text
area 0 sham-link 172.16.1.1 172.16.2.2
```

The logical connection looks like this:

```text
CE1 --- PE1 --- sham link --- PE2 --- CE2
```

This makes the MPLS VPN path participate in the same OSPF area as the customer sites.

Routes that would otherwise appear as inter-area can appear as intra-area.

```text
Without sham link:
CE2 learns CE1 LAN as O IA.

With sham link:
CE2 learns CE1 LAN as O.
```

Now the CE can choose between the MPLS VPN path and the backdoor link based on OSPF cost.

### Key point

A sham link creates a logical point-to-point OSPF link between the PE routers inside the customer VRF.

The PEs form an OSPF adjacency over this link and exchange LSAs directly.

Because os this, routes learned across it can remain intra-area
instead of being represented as inter-area routes through the MPLS VPN superbackbone.

This is what allows the MPLS VPN path to compete with a customer backdoor link using normal OSPF intra-area cost comparison.

---

## A Sham Link Is Not a Data-Plane Tunnel

A sham link is not a tunnel.

It is not a separate forwarding path for customer packets.

The sham link affects the OSPF control plane.

Customer data traffic still follows normal MPLS L3VPN forwarding:

```text
Ingress PE:
VRF lookup.

Core:
Transport label switching.

Egress PE:
VPN label disposition.
```

The sham link changes how OSPF sees the VPN path.

It does not replace the MPLS VPN data plane.

---

## Sham Link Requirements

A sham link has several important requirements.

### The customer sites must be in the same OSPF area

Sham links are used when the VPN path and the backdoor path need to appear in the same OSPF area.

Typical example:

```text
CE1 area 0
CE2 area 0
Backdoor link area 0
Sham link area 0
```

The area in the sham-link command must match the area where you want the logical OSPF connection.

```text
router ospf 10 vrf CUSTOMER_A
 area 0 sham-link 172.16.1.1 172.16.2.2
```

### The sham link is configured between PE routers

The sham link is configured on the PEs, not on the CEs.

```text
PE1:
area 0 sham-link 172.16.1.1 172.16.2.2

PE2:
area 0 sham-link 172.16.2.2 172.16.1.1
```

### The sham link endpoints must be in the VRF

Each PE needs a sham-link endpoint address inside the customer VRF.

A common method is to create a loopback interface in the VRF.

PE1:

```text
interface Loopback100
 vrf forwarding CUSTOMER_A
 ip address 172.16.1.1 255.255.255.255
```

PE2:

```text
interface Loopback100
 vrf forwarding CUSTOMER_A
 ip address 172.16.2.2 255.255.255.255
```

These addresses are the sham-link source and destination.

### The sham link endpoints must be reachable through the VPN

PE1 must be able to reach PE2's sham-link endpoint through the MPLS VPN.

PE2 must be able to reach PE1's sham-link endpoint through the MPLS VPN.

The endpoint routes should be carried through MP-BGP VPNv4.

The sham link is supposed to run over the VPN path, so the endpoint /32 routes should be in BGP/VPNv4.

---

## Basic Sham Link Topology

Use this example:

```text
Customer A Site 1                    Customer A Site 2

10.10.10.0/24                        11.11.11.0/24
      |                                      |
     CE1 ---------- backdoor link --------- CE2
      |                                      |
     PE1 ----------- MPLS Core ------------ PE2
```

OSPF area:

```text
CE1-to-PE1 link:
Area 0

CE2-to-PE2 link:
Area 0

CE1-to-CE2 backdoor link:
Area 0

PE1-to-PE2 sham link:
Area 0
```

Sham-link endpoints:

```text
PE1 Loopback100 in CUSTOMER_A:
172.16.1.1/32

PE2 Loopback100 in CUSTOMER_A:
172.16.2.2/32
```

Goal:

```text
Prefer the MPLS VPN path.

Use the backdoor link only as backup.
```

---

## Configuring the Sham-Link Endpoint Routes

Create a VRF loopback on each PE.

PE1:

```text
interface Loopback100
 vrf forwarding CUSTOMER_A
 ip address 172.16.1.1 255.255.255.255
```

PE2:

```text
interface Loopback100
 vrf forwarding CUSTOMER_A
 ip address 172.16.2.2 255.255.255.255
```

Advertise these /32 routes through VPNv4.

One simple method is to redistribute connected under the BGP VRF address family.

PE1:

```text
router bgp 65000
 address-family ipv4 vrf CUSTOMER_A
  redistribute connected
```

PE2:

```text
router bgp 65000
 address-family ipv4 vrf CUSTOMER_A
  redistribute connected
```

In a real design, avoid blindly redistributing all connected routes unless that is intended.

You can use a route map to allow only the sham-link endpoint loopbacks.

Example on PE1:

```text
route-map CONNECTED-TO-BGP permit 10
 match interface loopback100

router bgp 65000
 address-family ipv4 vrf CUSTOMER_A
  redistribute connected route-map CONNECTED-TO-BGP
```

PE2 would use a similar policy.

The important point:

```text
The remote sham-link endpoint must be reachable in the VRF through BGP/VPNv4.
```

Verify on PE1:

```text
PE1# show ip route vrf CUSTOMER_A
B        172.16.2.2 [200/0] via 4.4.4.4, 00:00:55
```

Verify on PE2:

```text
PE2# show ip route vrf CUSTOMER_A
B        172.16.1.1 [200/0] via 1.1.1.1, 00:00:51
```

---

## Configuring the Sham Link

Configure the sham link under the OSPF process for the VRF.

PE1:

```text
router ospf 10 vrf CUSTOMER_A
 area 0 sham-link 172.16.1.1 172.16.2.2
```

PE2:

```text
router ospf 10 vrf CUSTOMER_A
 area 0 sham-link 172.16.2.2 172.16.1.1
```

The command format is:

```text
area <area-id> sham-link <local-address> <remote-address> [cost <cost>]
```

The local address is the local PE's sham-link endpoint.

The remote address is the remote PE's sham-link endpoint.

The cost controls the OSPF metric across the sham link.

> The default cost of a sham link is **1**.

If the goal is for MPLS VPN to be primary and the backdoor link to be backup, make sure the sham link's cost is lower than the backdoor path cost.

---

## Before and After Sham Link

Assume CE1's LAN is `10.10.10.0/24`.

Before the sham link, CE2 learns the route over the backdoor link as intra-area:

```text
CE2# show ip route ospf
O        10.10.10.0 [110/101] via 12.12.12.1, 00:00:00, Ethernet0/1
```

CE2 may also have the VPN path available as an inter-area route, but OSPF prefers the intra-area route.

After the sham link comes up, the MPLS VPN path can be seen as intra-area too.

```text
CE2# show ip route ospf
O        10.10.10.0 [110/22] via 192.168.2.1, 00:01:12, Ethernet0/0
```

Now both paths are intra-area.

OSPF can compare the cost.

```text
Backdoor:
O, cost 101

MPLS VPN with sham link:
O, cost 22
```

So the path through the service provider is preferred.

---

## Verification

Verify sham links with this command:

```text
PE1# show ip ospf sham-links 
Sham Link OSPF_SL0 to address 172.16.2.2 is up
Area 0 source address 172.16.1.1
  Run as demand circuit
  DoNotAge LSA allowed. Cost of using 1 State POINT_TO_POINT,
  Timer intervals configured, Hello 10, Dead 40, Wait 40,
    Hello due in 00:00:03
    Adjacency State FULL (Hello suppressed)
    Index 1/2/2, retransmission queue length 0, number of retransmission 0
    First 0x0(0)/0x0(0)/0x0(0) Next 0x0(0)/0x0(0)/0x0(0)
    Last retransmission scan length is 0, maximum is 0
    Last retransmission scan time is 0 msec, maximum is 0 msec
```

Additionally, verify that the CE routers are now receiving **intra-area** routes (type `O`).

---

## Common Problems

### The sham link does not come up

Check reachability to the remote sham-link endpoint.

```text
show ip route vrf CUSTOMER_A
ping vrf CUSTOMER_A <remote-sham-endpoint> source <local-sham-endpoint>
```

The remote endpoint must be reachable through the VPN.

### The sham link is up, but traffic still uses the backdoor link

Check OSPF cost.

The sham link makes the VPN path intra-area, but OSPF still chooses the lowest-cost intra-area path.

If the backdoor path has a lower cost, it will still be preferred.

Increase the backdoor link cost or lower the sham-link cost.

### The CE still sees `O IA`

If the CE still sees the remote route as inter-area, the sham link may not be in the correct area or may not be up.

The sham link must belong to the same OSPF area as the customer sites that need intra-area connectivity.

---

## Sham Link vs Virtual Link

Do not confuse sham links with OSPF virtual links.

| | Sham Link | Virtual Link |
| :--- | :--- | :--- |
| **Used for** | MPLS L3VPN PE-CE OSPF with backdoor links | Connecting to area 0 through another area |
| **Configured on** | PE routers | ABRs |
| **Runs over** | MPLS VPN inside the VRF | Transit OSPF area |
| **Main purpose** | Make VPN path appear intra-area | Extend area 0 connectivity |
| **Common command** | `area <area> sham-link ...` | `area <area> virtual-link ...` |

A sham link is an L3VPN-specific OSPF tool.

A virtual link is a normal OSPF backbone repair tool.

---

## When Sham Links Are Not Needed

You do not need a sham link just because OSPF is used as the PE-CE protocol.

A sham link is normally needed only when:

```text
The same customer OSPF area exists at multiple sites.

There is a backdoor OSPF link between those sites.

The backdoor route is preferred because it is intra-area.

You want the MPLS VPN path to compete as an intra-area path.
```

If there is no backdoor link, do not configure a sham link.

If the backdoor route is not preferred, you may not need a sham link.

If the customer is fine using the backdoor link as primary, you may not need a sham link.

---

## Key Points

* A sham link is used with MPLS L3VPN PE-CE OSPF when customer sites also have an OSPF backdoor link.
* Without a sham link, the VPN path may appear as `O IA` while the backdoor path appears as `O`.
* OSPF prefers intra-area routes over inter-area routes, regardless of cost.
* A sham link makes the VPN path appear as an intra-area path.
* After the sham link is up, OSPF can compare the VPN path and backdoor path using cost.
* A sham link is configured between PE routers under the customer VRF OSPF process.
* The sham link belongs to an OSPF area.
* Sham-link endpoints are usually /32 loopbacks inside the customer VRF.
* The remote sham-link endpoint must be reachable through MP-BGP VPNv4.
* The sham-link endpoint route should not be learned through OSPF over the backdoor link.
* The CE routers do not configure the sham link.
* A sham link affects the OSPF control plane, not the MPLS data-plane forwarding mechanism.
* A sham link is different from an OSPF virtual link.
