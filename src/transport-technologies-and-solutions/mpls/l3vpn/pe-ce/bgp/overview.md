# PE-CE BGP

BGP is one of the most common routing protocols used between a **PE** router and a **CE** router in an MPLS Layer 3 VPN.

The CE uses normal IPv4 BGP.

The PE uses BGP inside a VRF.

The MPLS provider core uses MP-BGP to carry the customer routes between PE routers.

```text
        Customer Site A                 MPLS VPN                 Customer Site B

        CE1 -------- PE1 ================================= PE2 -------- CE2
             BGP          MP-BGP VPNv4 / MPLS forwarding        BGP
```

The important point is that there are two different BGP roles:

| BGP Session | Purpose |
| --- | --- |
| PE-CE BGP | Exchanges customer IPv4 routes between the PE and CE |
| PE-PE MP-BGP | Exchanges VPNv4 routes between PE routers |

The CE does not need to understand MPLS, VPNv4, route distinguishers, route targets, or VPN labels.

The CE just exchanges IPv4 routes with the PE.

---

## PE-CE BGP vs PE-PE MP-BGP

PE-CE BGP is normal IPv4 BGP.

PE-PE MP-BGP is used to carry VPN routes across the provider network.

```text
CE1 advertises 10.10.10.0/24 to PE1.

PE1 learns the route in the CUSTOMER_A VRF.

PE1 converts the route into a VPNv4 route.

PE1 advertises the VPNv4 route to PE2.

PE2 imports the route into the CUSTOMER_A VRF.

PE2 advertises 10.10.10.0/24 to CE2 as a normal IPv4 BGP route.
```

From the CE's point of view, this looks like normal BGP routing.

From the PE's point of view, the route must be associated with a VRF and then exported into MP-BGP.

---

## Route Flow

### CE to PE

When the CE advertises a route to the PE, the PE learns the route inside the VRF.

```text
CE1 -> PE1

Prefix: 10.10.10.0/24
Protocol: BGP
VRF: CUSTOMER_A
```

The PE can then export the route into MP-BGP VPNv4.

To do that, the PE adds VPN information to the route.

```text
IPv4 route:
10.10.10.0/24

VPNv4 route:
RD + 10.10.10.0/24

Additional VPN information:
Route Target
VPN label
BGP next hop
```

The RD makes the route unique in the provider network.

The RT controls which VRFs import the route.

The VPN label tells the remote PE which VRF should receive the packet.

---

### PE to CE

When a remote PE receives a VPNv4 route, it checks the route targets.

If the route target matches the local VRF import policy, the route is imported into the VRF.

The PE then advertises the route to the CE as a normal IPv4 BGP route.

```text
PE2 -> CE2

Prefix: 10.10.10.0/24
Protocol: BGP
VRF: CUSTOMER_A
```

The CE does not see the RD, RT, or VPN label.

---

## Example

Topology:

```text
Customer Site A                        Customer Site B

10.10.10.0/24                            11.11.11.0/24
     |                                      |
    CE1                                    CE2
     |                                      |
    PE1 ============== MPLS VPN ========== PE2
```

CE1 advertises `10.10.10.0/24` to PE1.

CE2 advertises `11.11.11.0/24` to PE2.

The provider network carries both routes using MP-BGP VPNv4.

After the routes are imported into the correct VRFs:

```text
CE1 learns 11.11.11.0/24 from PE1.

CE2 learns 10.10.10.0/24 from PE2.
```

The customer sites can now communicate through the MPLS VPN.

---

## BGP Configuration Location

On the PE, the CE neighbor is configured under the VRF IPv4 address family.

Example:

```text
router bgp 65000
 address-family ipv4 vrf CUSTOMER_A
  neighbor 192.168.1.2 remote-as 65100
  neighbor 192.168.1.2 activate
```

On the CE, the neighbor is usually configured as a normal IPv4 BGP neighbor.

Example:

```text
router bgp 65100
 neighbor 192.168.1.1 remote-as 65000
```

The PE-CE link belongs to the customer VRF on the PE.

```text
interface Ethernet0/1
 ip vrf forwarding CUSTOMER_A
 ip address 192.168.1.1 255.255.255.252
```

The CE does not configure a VRF unless the customer is also using VRFs internally.

---

## Common AS Designs

There are a few common PE-CE BGP AS designs.

### Different Customer AS per Site

Each customer site uses a different AS number.

```text
CE1: AS 65100
CE2: AS 65200
PEs: AS 65000
```

This is simple because the remote CE does not see its own AS in the AS path.

```text
CE2 receives 10.10.10.0/24 with AS path:

65000 65100
```

CE2 is in AS 65200, so it accepts the route.

---

### Same Customer AS at Multiple Sites

The customer uses the same AS number at multiple sites.

```text
CE1: AS 65100
CE2: AS 65100
PEs: AS 65000
```

This can cause a problem.

When CE2 receives a route that originated from CE1, the AS path may include AS 65100.

```text
CE2 receives 10.10.10.0/24 with AS path:

65000 65100
```

CE2 sees its own AS in the AS path and rejects the route.

This is normal BGP loop prevention.

There are two common solutions:

| Feature | Where It Is Configured | Basic Idea |
| --- | --- | --- |
| AS Override | PE | PE replaces the customer's AS in the AS path |
| Allowas-in | CE | CE allows routes that contain its own AS |

These are covered in their own page.

---

## PE-CE Multihoming

A customer site may connect to more than one PE.

```text
                 MPLS VPN
             PE2 ========= PE3
              |             |
             CE2 --------- CE3
                   Site B
```

This improves redundancy, but it can also introduce routing loop problems.

For example, a route learned from Site A through one PE might be advertised through the MPLS VPN and then sent back to Site A through another PE.

BGP has AS path loop prevention, but AS path alone is not always enough in MPLS L3VPN multihoming designs.

This is where **Site of Origin** is useful.

---

## Site of Origin

Site of Origin, or SoO, is a BGP extended community used to identify the site where a route came from.

The same SoO value is configured on all PE-CE links that connect to the same customer site.

Example:

```text
Site A SoO: 65000:100
Site B SoO: 65000:200
```

If a route is learned from Site A, the PE tags it with the Site A SoO value.

A PE should not advertise a route back toward a CE if the route has the same SoO value as that PE-CE link.

This prevents a site from relearning its own routes through the MPLS VPN.

SoO is especially important when:

- A customer site is multihomed to multiple PEs
- A customer site has an internal or backdoor link
- AS override or allowas-in overrides normal AS path loop prevention

SoO is covered in detail later in its own page.

---

## What the CE Sees

The CE only sees normal BGP information.

For example:

```text
Prefix: 10.10.10.0/24
Next hop: PE address on the PE-CE link
AS path: 65000 65100
```

The CE does not see:

- The route distinguisher
- The route target
- The VPN label
- The MPLS transport label
- The provider core topology

This is one of the benefits of MPLS L3VPN.

The customer routing table is separated from the provider core.

---

## What the PE Sees

The PE sees both sides of the VPN.

On the customer-facing side, the PE sees normal IPv4 routes inside the VRF.

```text
VRF CUSTOMER_A:
10.10.10.0/24 learned from CE1
```

On the provider-facing side, the PE advertises VPNv4 routes to other PEs.

```text
VPNv4:
RD 65000:1:10.10.10.0/24
RT 1:1
VPN label 24001
Next hop PE1 loopback
```

The PE connects the customer routing domain to the provider MP-BGP control plane.

---

## Control Plane Summary

```text
1. CE advertises IPv4 route to PE.

2. PE installs the route in the customer VRF.

3. PE converts the route into a VPNv4 route.

4. PE advertises the VPNv4 route to remote PEs using MP-BGP.

5. Remote PE imports the VPNv4 route into the correct VRF.

6. Remote PE advertises the route to the remote CE using IPv4 BGP.
```

---

## Data Plane Summary

The data plane uses MPLS labels across the provider core.

When traffic enters the MPLS VPN from a CE:

```text
CE1 sends an IP packet to PE1.

PE1 pushes:
- Transport label
- VPN label

Core routers forward based on the transport label.

PE2 receives the packet.

PE2 uses the VPN label to identify the correct VRF.

PE2 forwards the packet to CE2.
```

The CE sends and receives normal IP packets. Only the provider routers use MPLS labels.

This is standard MPLS L3VPN forwarding.

---

## Common Issues

PE-CE BGP troubleshooting often involves checking both the VRF side and the VPNv4 side.

Common issues include:

- The PE-CE interface is not in the correct VRF
- The PE-CE BGP neighbor is not configured under the VRF address family
- The CE is not advertising the expected prefixes
- The PE learns the route but does not export it with the correct route target
- The remote PE receives the VPNv4 route but does not import it
- The route is rejected because the CE sees its own AS in the AS path
- AS override or allowas-in is missing when the same customer AS is used at multiple sites
- SoO blocks a route because the route appears to have originated from the same site
- The VPN label or transport label is missing
- The remote PE does not advertise the imported route to the CE

I'll cover basic PE-CE configuration on the next page.

---

## Key Points

- PE-CE BGP exchanges customer IPv4 routes between the PE and CE.
- PE-PE MP-BGP exchanges VPNv4 routes between PE routers.
- The CE does not participate in MPLS or MP-BGP VPNv4.
- The PE converts customer IPv4 routes into VPNv4 routes.
- Route targets control which VRFs import the routes.
- VPN labels identify the correct VRF on the egress PE.
- If the same customer AS is used at multiple sites, AS path loop prevention can block routes.
- AS override and allowas-in are common solutions for same-AS customer designs.
- Site of Origin helps prevent routing loops in multihomed MPLS VPN sites.
