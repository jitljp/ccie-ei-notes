# PE-CE Routing Overview

In an MPLS L3VPN, the PE routers exchange VPN routes with each other using MP-BGP VPNv4.

However, the PE routers also need to exchange routes with the customer routers.

This is called **PE-CE routing**.

```text
CE1 --- PE1 === MPLS Core === PE2 --- CE2
 ^       ^                     ^      ^
 |-------|                     |------|
PE-CE routing                  PE-CE routing
```

The CE routers do not run MPLS.

They only run normal IP routing with the PE routers.

The PE routers are responsible for connecting the customer routing domain to the MPLS VPN control plane.

---

## Where PE-CE Routing Fits

An MPLS L3VPN has two major routing areas:

```text
Provider core routing:
PE loopbacks, P router loopbacks, core links, LDP transport.

Customer routing:
Customer prefixes learned from CE routers and placed into VRFs.
```

These customer routes are installed in the customer's VRF on the PE router.

Example:

```text
CE1 advertises 10.10.10.0/24 to PE1.
PE1 installs the route in VRF CUSTOMER_A.
PE1 exports the route into MP-BGP VPNv4.
PE2 imports the route into VRF CUSTOMER_A.
PE2 advertises the route to CE2.
```

---

## PE-CE Routing Is VRF-Aware

PE-CE routing runs inside a VRF on the PE.

The PE interface facing the CE is assigned to the customer VRF.

Example:

```text
interface Ethernet0/1
 description Link to CE1
 vrf forwarding CUSTOMER_A
 ip address 192.168.1.1 255.255.255.0
```

Because the interface is in `CUSTOMER_A`, routes learned over that interface are associated with the `CUSTOMER_A` VRF.

The PE may run a separate routing process or address family for that VRF.

Examples:

```text
Static route in a VRF
OSPF process with a VRF
EIGRP address family with a VRF
BGP address family ipv4 vrf CUSTOMER_A
```

The important point is:

```text
PE-CE routes belong to the customer VRF, not the global routing table.
```

---

## Basic Route Flow

The exact configuration depends on the PE-CE protocol, but the general route flow is similar.

For a route from CE1 to CE2:

```text
1. CE1 advertises a customer route to PE1.
2. PE1 installs the route in the correct VRF.
3. PE1 makes the route available to BGP under the VRF address family.
4. PE1 exports the route into MP-BGP VPNv4.
5. PE1 attaches the VRF's export route target.
6. PE1 advertises the VPNv4 route to PE2.
7. PE2 imports the VPNv4 route into its local VRF.
8. PE2 advertises the route to CE2 using the PE-CE protocol.
```

In the opposite direction, the same process happens from CE2 to CE1.

```text
CE2 route → PE2 VRF → VPNv4 → PE1 VRF → CE1
```

This ensures that the PE routers know all of the necessary customer routes,
and each CE router knows routes to destinations behind other CE routers.

---

## PE-CE Static Routing

Static routing is the simplest PE-CE option.

Example on PE1:

```text
ip route vrf CUSTOMER_A 10.10.10.0 255.255.255.0 192.168.1.2
```

This installs a static route in the `CUSTOMER_A` VRF.

Then BGP must be told to export static routes from the VRF into VPNv4.

Example:

```text
router bgp 65000
 address-family ipv4 vrf CUSTOMER_A
  redistribute static
```

Static routing is useful for simple sites, small labs, and basic L3VPN verification.

It is also the easiest PE-CE method to understand before adding dynamic routing protocols.

However, it is rarely the preferred choice in real networks due to the usual downsides of static routing.
- Manual configuration
- Lack of flexibility/adaptability

---

## PE-CE OSPF

OSPF is a common choice for PE-CE routing.

The PE and CE form a normal OSPF neighbor relationship.

However, routes that cross the MPLS VPN core are treated specially.

Important L3VPN-specific OSPF topics include:

```text
OSPF superbackbone
Domain ID
OSPF route type preservation
Down bit
Domain tag
Sham links
```

These topics exist because the MPLS VPN core behaves like a special OSPF backbone between customer sites.

From the CE's point of view, routes may appear as inter-area or external depending on the Domain ID and how the route was carried through the VPN.

OSPF also needs loop-prevention mechanisms so that routes learned from the MPLS VPN are not incorrectly sent back into the VPN.

That is where the Down bit and domain tag become important.

> We'll cover these topics in the PE-CE OSPF section.

---

## PE-CE EIGRP

EIGRP can also be used between PE and CE routers.

The PE runs EIGRP inside the customer VRF and exchanges routes with the CE.

Important L3VPN-specific EIGRP topics include:

```text
EIGRP internal and external route behavior
Metric preservation
Redistribution between EIGRP and BGP
Site of Origin
Loop prevention with multihomed sites
```

EIGRP is easier than OSPF in some ways because it does not have OSPF's superbackbone, Domain ID, Down bit, or sham link behavior.

When EIGRP routes are carried through MP-BGP VPNv4 and then reintroduced into EIGRP at the remote PE, the route should still have useful EIGRP metric information.

EIGRP Site of Origin is also important when the same customer site connects to more than one PE.

---

## PE-CE BGP

BGP is a very common PE-CE routing protocol in service provider networks.

The CE forms an EBGP session with the PE inside the customer VRF.

Example:

```text
router bgp 65000
 address-family ipv4 vrf CUSTOMER_A
  neighbor 192.168.1.2 remote-as 65100
  neighbor 192.168.1.2 activate
```

BGP is often clean for L3VPN designs because the route is already in BGP under the VRF address family.

That means the PE can export the route into VPNv4 without first redistributing it from another protocol into BGP.

Important L3VPN-specific BGP topics include:

```text
Basic PE-CE eBGP
Customer sites using the same AS
AS override
Allowas-in
Site of Origin
Multihoming
Loop prevention
```

Two common problems appear when the same customer AS is used at multiple sites:

```text
The CE rejects a route because its own AS is in the AS path.
The provider needs to manipulate AS path behavior for the VPN design.
```

`as-override` and `allowas-in` are two common tools used to solve this, depending on the design.

---

## Static vs OSPF vs EIGRP vs BGP

Each PE-CE option has different tradeoffs.

| PE-CE method | Main use | Main L3VPN concern |
| :--- | :--- | :--- |
| **Static** | Simple sites and labs | Redistribute static into BGP VRF address family |
| **OSPF** | Customers that use OSPF internally | Superbackbone, Domain ID, Down bit, sham links |
| **EIGRP** | Customers that use EIGRP internally | Metric preservation and SoO |
| **BGP** | Service provider style PE-CE routing | AS override, allowas-in, SoO, multihoming |

The choice depends on the customer design, operational requirements, and the provider's policy.

---

## Redistribution Is Key

For many PE-CE protocols, the route must move between the PE-CE protocol and BGP.

Example with static routes:

```text
Static route in VRF
  ↓
BGP ipv4 vrf CUSTOMER_A
  ↓
MP-BGP VPNv4
```

Example with OSPF:

```text
OSPF route in VRF
  ↓
BGP ipv4 vrf CUSTOMER_A
  ↓
MP-BGP VPNv4
  ↓
BGP ipv4 vrf CUSTOMER_A on remote PE
  ↓
OSPF toward remote CE
```

This is why PE-CE routing is not only about neighbor formation.

A PE-CE neighbor can be up, but routes may still fail to move across the VPN if redistribution or route import/export is incorrect.

---

## What This Section Covers

This PE-CE section focuses only on the L3VPN-specific behavior of each routing option.

It does not re-teach the full protocols from scratch.

The goal is to understand how each protocol interacts with:

```text
VRFs
MP-BGP VPNv4
Route targets
VPN labels
Redistribution
Loop prevention
Multihoming
```

The following pages build from simple to more complex:

```text
1. PE-CE static routing
2. PE-CE OSPF
3. OSPF L3VPN special behaviors
4. PE-CE EIGRP
5. EIGRP L3VPN special behaviors
6. PE-CE BGP
7. BGP L3VPN special behaviors
8. Troubleshooting for each protocol
```

---

## Key Points

* PE-CE routing exchanges customer routes between CE and PE routers.
* CE routers do not need MPLS.
* PE-CE routing runs inside a VRF on the PE.
* Customer routes are installed in the VRF, not in the provider global routing table.
* The PE exports VRF routes into MP-BGP VPNv4.
* The remote PE imports VPNv4 routes into its local VRF based on route targets.
* Static routing is a simple option for labs.
* OSPF has several L3VPN-specific behaviors, including the superbackbone, Domain ID, Down bit, domain tag, and sham links.
* EIGRP L3VPN behavior focuses mainly on route type, metric preservation, redistribution, and Site of Origin.
* BGP is common for PE-CE routing and often involves AS override, allowas-in, Site of Origin, and multihoming.
* PE-CE troubleshooting should follow the route from CE to PE VRF, then to VPNv4, then to the remote PE VRF, then to the remote CE.
