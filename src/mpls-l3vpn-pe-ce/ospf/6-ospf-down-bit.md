# Down Bit

The **Down Bit** is an OSPF loop prevention mechanism used with MPLS L3VPN PE-CE OSPF.

It is also called the **DN bit**.

It is used when a PE advertises a route learned from MP-BGP VPNv4 back into OSPF.

The basic idea is:

```text
This route came down from the MPLS VPN backbone.
Do not send it back up into the MPLS VPN backbone again.
```

The Down Bit helps prevent routing loops between OSPF and MP-BGP.

---

## Why the Down Bit Exists

In PE-CE OSPF, routes move between OSPF and MP-BGP.

Example:

```text
CE1
 ↓ OSPF
PE1
 ↓ MP-BGP VPNv4
PE2
 ↓ OSPF
CE2
```

This route flow is normal.

But in some topologies, the same route may be advertised back toward another PE through OSPF.

Example:

```text
CE1 --- PE1 === MPLS VPN === PE2 --- CE2
                  ||                 /
                  ||              CE3
                  PE3______________/
```

Route flow:

```text
1. CE1 advertises 10.10.10.0/24 to PE1 using OSPF.
2. PE1 exports the route into MP-BGP VPNv4.
3. PE2 imports the VPNv4 route.
4. PE2 advertises the route into OSPF toward CE2.
5. CE2 or another customer router advertises the route through OSPF.
6. PE3 receives the route from OSPF.
```

Without loop prevention, PE3 might think the route originated from the local customer site and export it back into MP-BGP.

That would be wrong.

The route originally came from the MPLS VPN backbone.

The Down Bit prevents this.

---

## Basic Rule

When a PE advertises a VPN-learned route into OSPF, it sets the Down Bit in the LSA.

```text
MP-BGP VPNv4 route
  ↓
PE redistributes into OSPF
  ↓
OSPF LSA with Down Bit set
```

If another PE later receives that LSA from a CE, it recognizes the Down Bit.

The receiving PE does not use that LSA for OSPF route calculation.

Because the route is not used, it is not installed as an OSPF route in the VRF.

Because it is not installed as an OSPF route, it is not redistributed back into MP-BGP VPNv4.

```text
PE receives OSPF LSA with Down Bit set
  ↓
Do not use it for route calculation
  ↓
Do not install it as an OSPF route
  ↓
Do not export it back into VPNv4
```

This prevents the route from looping between OSPF and MP-BGP.

---

## Where the Down Bit Is Set

The Down Bit is set in OSPF LSAs originated by a PE for routes learned from the VPN backbone.

For OSPFv2, this applies to LSAs such as:

```text
Type 3 Summary LSAs
Type 5 External LSAs
Type 7 NSSA External LSAs
```

Type 3 LSAs are used when the remote route is treated as an inter-area route.

Type 5 LSAs are used when the remote route is treated as an external route in a normal area.

Type 7 LSAs are used when the remote route is treated as an external route in an NSSA area.

Example:

```text
Same Domain ID:
VPN route is advertised as Type 3 LSA with Down Bit set.

Different Domain ID:
VPN route is advertised as Type 5 LSA with Down Bit set.

NSSA PE-CE area:
VPN route may be advertised as Type 7 LSA with Down Bit set.
```

The Down Bit is for PE routers that may receive the route later from OSPF.

---

## Down Bit with Same Domain ID

Assume CE1 advertises an internal OSPF route to PE1.

```text
CE1 route:
O 10.10.10.0/24
```

PE1 exports the route into VPNv4.

PE2 imports the VPNv4 route.

If the Domain IDs match, PE2 advertises the route to CE2 as an inter-area route.

```text
CE2# show ip route ospf
O IA     10.10.10.0/24 via 192.168.2.1
```

Behind this route, PE2 originated a Type 3 Summary LSA.

That Type 3 LSA has the Down Bit set.

```text
VPNv4 route received by PE2
  ↓
PE2 originates Type 3 Summary LSA
  ↓
Down Bit is set
  ↓
CE2 installs O IA route
```

If that LSA later reaches another PE, the other PE ignores the route (although the LSA is kept in the LSDB).

---

## Down Bit with Different Domain ID

Now assume PE1 and PE2 use different OSPF Domain IDs.

```text
PE1 Domain ID: 0.0.0.10
PE2 Domain ID: 0.0.0.20
```

PE2 receives the VPNv4 route from PE1.

Because the Domain IDs do not match, PE2 treats the route as external.

```text
CE2# show ip route ospf
O E2     10.10.10.0/24 via 192.168.2.1
```

Behind this route, PE2 originated a Type 5 External LSA.

That Type 5 LSA has the Down Bit set.

It may also have a Domain Tag.

```text
Different Domain ID:
Type 5 External LSA
Down Bit set
Domain Tag used for external route loop prevention
```

The Down Bit and Domain Tag are closely related, but they are not the same thing.

```text
Down Bit:
A bit in the OSPF LSA/prefix options.

Domain Tag:
A 32-bit route tag used with external OSPF routes.
```

The Domain Tag is covered in the next page.

---

## Down Bit and CE routers

The Down Bit does not mean the route is unusable to the CE.

A CE may still install and use the route.

Example:

```text
CE2# show ip route ospf
O IA     10.10.10.0/24 [110/21] via 192.168.2.1
```

The CE sees a normal OSPF route.

The Down Bit is a signal for PE routers.

If another PE receives the same LSA through OSPF, that PE ignores that LSA for route calculation,
and therefore does not redistribute the router back into VPNv4.

```text
CE behavior:
Use the route normally.

PE behavior:
If LSA has Down Bit set, ignore the route.
```

---

## Example Loop Prevention

Use this topology:

```text
CE1 --- PE1 === MPLS VPN === PE2 --- CE2
                  ||            \    /
                  ||              CE3
                  PE3______________/
```

CE1 advertises `10.10.10.0/24` to PE1.

PE1 exports it into VPNv4.

PE2 imports it and advertises it to CE2 in OSPF.

The LSA has the Down Bit set.

CE2 advertises the route through OSPF toward PE3.

PE3 receives the LSA.

Because the Down Bit is set, PE3 ignores the LSA for route calculation.

PE3 does not export the route into VPNv4.

```text
PE3 receives LSA with Down Bit
  ↓
PE3 does not install route as OSPF
  ↓
PE3 does not redistribute it into BGP
  ↓
Loop prevented
```

---

## Verifying the Down Bit

Start with the route on the CE.

```text
CE2# show ip route ospf
O IA     10.10.10.0 [110/21] via 192.168.2.1, 00:16:12, Ethernet0/0
```

Then check the OSPF database.

For an inter-area route, check the summary LSA.

```text
CE2# show ip ospf database summary 10.10.10.0

            OSPF Router with ID (11.11.11.1) (Process ID 1)

                Summary Net Link States (Area 0)

  LS age: 989
  Options: (No TOS-capability, DC, Downward)
  LS Type: Summary Links(Network)
  Link State ID: 10.10.10.0 (summary Network Number)
  Advertising Router: 192.168.2.1
  LS Seq Number: 80000001
  Checksum: 0xE640
  Length: 28
  Network Mask: /24
        MTID: 0         Metric: 11 

  LS age: 142
  Options: (No TOS-capability, DC, Downward)
  LS Type: Summary Links(Network)
  Link State ID: 10.10.10.0 (summary Network Number)
  Advertising Router: 192.168.3.1
  LS Seq Number: 80000003
  Checksum: 0xDB48
  Length: 28
  Network Mask: /24
        MTID: 0         Metric: 11 
```

For an external route, check the external LSA.

```text
CE2# show ip ospf database external 10.10.10.0
```

`Downward` in the output indicates the Down Bit.

The important verification logic is:

```text
Route came from VPNv4.
PE advertised it into OSPF.
The resulting LSA should be marked with the Down Bit.
Another PE should not use that LSA if it receives it from OSPF.
```

---

## Verifying on the PE

On the PE, check whether the OSPF process is connected to the MPLS VPN superbackbone.

```text
PE2# show ip ospf 10
 Routing Process "ospf 10" with ID 192.168.2.1
   Domain ID type 0x0005, value 0.0.0.10
 Start time: 00:04:20.068, Time elapsed: 04:36:57.000
 Supports only single TOS(TOS0) routes
 Supports opaque LSA
 Supports Link-local Signaling (LLS)
 Supports area transit capability
 Supports NSSA (compatible with RFC 3101)
 Supports Database Exchange Summary List Optimization (RFC 5243)
 Connected to MPLS VPN Superbackbone, VRF CUSTOMER_A
```

This tells you the router is treating the VRF OSPF process as PE-CE OSPF for MPLS VPN.

That is the mode where Down Bit and other MPLS VPN OSPF checks apply.

Also verify the VPNv4 route.

```text
PE2# show bgp vpnv4 unicast vrf CUSTOMER_A 10.10.10.0/24
BGP routing table entry for 65000:1:10.10.10.0/24, version 77
Paths: (1 available, best #1, table CUSTOMER_A)
  Not advertised to any peer
  Refresh Epoch 1
  Local
    1.1.1.1 (metric 31) (via default) from 1.1.1.1 (1.1.1.1)
      Origin incomplete, metric 11, localpref 100, valid, internal, best
      Extended Community: RT:1:1 OSPF DOMAIN ID:0x0005:0x0000000A0200 
        OSPF RT:0.0.0.0:2:0 OSPF ROUTER ID:192.168.1.1:0
      mpls labels in/out nolabel/21
      rx pathid: 0, tx pathid: 0x0
      Updated on Jun 17 2026 23:19:57 UTC
```

The VPNv4 route carries OSPF information.

When PE2 advertises this route into OSPF, it marks the generated LSA with the Down Bit.


```
PE2#show ip ospf 10 database summary 10.10.10.0

            OSPF Router with ID (192.168.2.1) (Process ID 10)

                Summary Net Link States (Area 0)

  LS age: 1119
  Options: (No TOS-capability, DC, Downward)
  LS Type: Summary Links(Network)
  Link State ID: 10.10.10.0 (summary Network Number)
...
```

---

## Down Bit and Sham Links

A sham link changes how routes are exchanged between PEs.

With a sham link, the PEs form an OSPF adjacency inside the customer VRF and exchange LSAs directly.

```text
PE1 --- sham link --- PE2
```

Routes learned across the sham link can be intra-area routes.

The sham link is different from normal superbackbone redistribution.

```text
Normal superbackbone behavior:
VPNv4 route is imported and advertised into OSPF.
Down Bit is used on generated LSAs.

Sham link behavior:
PEs form an OSPF adjacency and exchange LSAs over the sham link.
The VPN path can appear as an intra-area OSPF path.
```

The Down Bit page and sham link page are related, but they solve different problems.

```text
Down Bit:
Prevents VPN-learned OSPF routes from being re-exported into VPNv4.

Sham link:
Makes the VPN path appear intra-area so it can compete with a backdoor link.
```

---

## capability vrf-lite

The `capability vrf-lite` command is important, but it is often misunderstood.

Example:

```text
router ospf 10 vrf CUSTOMER_A
 capability vrf-lite
```

This command tells IOS XE that the OSPF process is being used for multi-VRF / VRF-lite behavior, not as a PE connected to the MPLS VPN superbackbone.

After this command is configured, the OSPF process is no longer shown as connected to the MPLS VPN superbackbone.

Before:

```text
PE1# show ip ospf 10
 Connected to MPLS VPN Superbackbone
```

After:

```text
PE1# show ip ospf 10
```

The `Connected to MPLS VPN Superbackbone` line is no longer present.

In a normal MPLS L3VPN PE-CE OSPF design, do not configure `capability vrf-lite` unless the lab specifically requires it.

It suppresses PE checks that are needed for MPLS VPN OSPF loop prevention.

Use it for VRF-lite-style designs, or special cases where the router is not acting as an MPLS VPN PE for that OSPF process.

---

## Key Points

* The Down Bit is an OSPF loop prevention mechanism for MPLS L3VPN PE-CE OSPF.
* A PE sets the Down Bit when advertising VPN-learned routes into OSPF.
* If another PE receives an LSA with the Down Bit set from a CE, it does not use the LSA for route calculation.
* This prevents VPN-learned OSPF routes from being exported back into MP-BGP VPNv4.
* For OSPFv2, the Down Bit is used in Type 3, Type 5, and Type 7 LSAs generated from VPN routes.
* Type 5 and Type 7 external routes also use Domain Tag behavior.
* The CE can still install and use routes with the Down Bit set.
* The Down Bit is mainly a signal to PE routers.
* `capability vrf-lite` suppresses MPLS VPN PE behavior and should not be used in normal MPLS L3VPN PE-CE OSPF unless the design requires it.
