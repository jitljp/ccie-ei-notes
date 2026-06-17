# VPN Labels

MPLS L3VPN packets usually use two labels.

```text
Top label:    Transport label
Bottom label: VPN label
```

The **transport label** gets the packet across the provider core to the remote PE.

The **VPN label** tells the egress PE how to forward the packet after it reaches the egress PE.

```text
CE1 → PE1 → P1 → P2 → PE2 → CE2
      ↑                 ↑
      ingress PE        egress PE
```

The ingress PE imposes the label stack.

The P routers switch the packet based on the top label.

The egress PE uses the VPN label to forward the packet toward the correct CE or VRF route.

---

## Why VPN Labels Are Needed

In an MPLS L3VPN, the P routers do not know customer routes.
They only know provider routes, such as PE loopbacks.

The transport label solves the core forwarding problem, getting the packet to the remote PE.

But when the packet reaches the remote PE, the remote PE still needs to know what to do with the customer packet.
That is the job of the VPN label.

```text
Transport label:
Get this packet to PE2.

VPN label:
Once it reaches PE2, forward it using the correct VRF/route.
```

---

## Who Allocates the VPN Label?

The **egress PE** allocates the VPN label.

Then it advertises that label with the VPNv4 route using MP-BGP.

Example:

```text
PE2 has route 11.11.11.0/24 in VRF CUSTOMER_A.
PE2 allocates VPN label 24 for the route.
PE2 advertises the VPNv4 route and label to PE1.
```

As shown before, this VPN label is advertised via MP-BGP:

```text
VPNv4 route: 65000:1:11.11.11.0/24
Next hop:    5.5.5.5
RT:          65000:100
VPN label:   24
```

PE1 learns that traffic for `11.11.11.0/24` should be sent toward PE2 with VPN label `24`.

---

## Local Significance

VPN labels have local significance to the PE that allocated them.

If PE2 allocates label `24`, that label is meaningful to PE2.

```text
Label 24 on PE2:
Means PE2 knows how to forward the packet for that VPN route.
```

Other routers do not independently decide what label `24` means. They simply use it because PE2 advertised it.

> This is similar to LDP labels.

```text
LDP transport labels:
Locally significant to the LSR that advertises them.

VPN labels:
Locally significant to the PE that advertises them.
```

---

## Ingress PE vs Egress PE

For traffic from CE1 to CE2:

```text
CE1 → PE1 → MPLS core → PE2 → CE2
```

PE1 is the ingress PE.

PE2 is the egress PE.

PE2 advertises the VPN label to PE1.

PE1 imposes that VPN label when sending traffic to PE2.

```text
PE2 advertises label 24.
PE1 uses label 24.
PE2 receives label 24.
PE2 understands label 24.
```

For traffic in the reverse direction, PE1 allocates and advertises its own VPN label.

```text
CE2 → PE2 → MPLS core → PE1 → CE1
```

Now PE2 is the ingress PE.

PE1 is the egress PE.

PE1 advertises a VPN label to PE2 for PE1's local customer route.

---

## Two Labels in the Data Plane

Assume CE1 sends traffic to `11.11.11.1` behind CE2.

PE1 receives the packet in VRF `CUSTOMER_A`.

PE1 performs a VRF lookup and finds a VPNv4 route learned from PE2.

The route has:

```text
BGP next hop: 5.5.5.5
VPN label:    24
```

PE1 also has an LDP transport label to reach `5.5.5.5`.

Example:

```text
Transport label to 5.5.5.5: 17
VPN label for 11.11.11.0/24: 24
```

PE1 imposes both labels.

```text
Top label:    17
Bottom label: 24
Payload:      Customer IP packet
```

---

## PHP and VPN Labels

Penultimate Hop Popping can remove the transport label before the packet reaches the egress PE.

Example path:

```text
PE1 → P1 → P2 → PE2
```

P2 is the penultimate hop before PE2.

If PE2 advertises implicit null for its loopback FEC, P2 pops the transport label.

Before PHP:

```text
Top label:    Transport label to PE2
Bottom label: VPN label
Payload:      Customer IP packet
```

After PHP:

```text
Top label:    VPN label
Payload:      Customer IP packet
```

The VPN label is not popped by the penultimate P router.

The VPN label must reach the egress PE because the egress PE uses it to forward the customer packet.

This is why an L3VPN packet may arrive at the egress PE with only the VPN label remaining.

---

## Local Labels on the Egress PE

A PE allocates local VPN labels for its own VRF routes.

Example on PE1:

```text
PE1# show mpls forwarding-table
Local      Outgoing   Prefix           Bytes Label   Outgoing   Next Hop
Label      Label      or Tunnel Id     Switched      interface
18         No Label   10.10.10.0/24[V] 570           Et0/1      192.168.1.2
```

This means PE1 allocated local label `18` for the VPN route `10.10.10.0/24`.

The `[V]` indicates a VPN/VRF route.

The outgoing label is `No Label` because PE1 is the egress PE for this route.
After PE1 receives a packet with VPN label `18`, it forwards the packet as a normal IP packet toward CE1.

PE1 advertises label `18` to other PE routers using MP-BGP VPNv4.

---

## Remote VPN Labels on the Ingress PE

Remote VPN labels are learned through MP-BGP.

Use BGP or CEF commands to see them from the ingress PE's point of view.

Example on PE1:

```text
PE1# show bgp vpnv4 unicast all labels
   Network          Next Hop      In label/Out label
Route Distinguisher: 65000:1 (CUSTOMER_A)
   10.10.10.0/24    192.168.1.2     18/nolabel
   11.11.11.0/24    5.5.5.5         nolabel/24
```

From PE1's point of view:

```text
10.10.10.0/24:
In label 18
PE1 allocated this local VPN label.
PE1 advertises this label to remote PEs.

11.11.11.0/24:
Out label 24
PE1 learned this remote VPN label from PE2.
PE1 uses this label when sending traffic to PE2.
```

The remote VPN label may not appear in `show mpls forwarding-table` as a local LFIB entry.

That command mainly shows labels that the local router owns or uses as incoming labels.

To see remote labels used for imposition, `show bgp vpnv4 unicast all labels`.

---

## In Label and Out Label

The `show bgp vpnv4 unicast all labels` output is from the perspective of the router where the command is run.

```text
In label:
Local VPN label assigned by this PE.

Out label:
Remote VPN label learned from another PE.
```

Example:

```text
10.10.10.0/24    192.168.1.2     18/nolabel
```

This route is local to PE1.

PE1 allocated label `18`.

PE1 does not impose a VPN label to reach this route because the route is behind PE1.

So the out label is `nolabel`.

Example:

```text
11.11.11.0/24    5.5.5.5         nolabel/24
```

This route is remote from PE1's point of view.

PE1 does not allocate a local incoming label for it.

PE1 learned remote label `24` from PE2 and uses it as the outgoing VPN label.

---

## Seeing the Imposed Label Stack

To see the actual label stack PE1 imposes for traffic to a remote customer prefix, check CEF in the VRF.

Example:

```text
PE1#show ip cef vrf CUSTOMER_A 11.11.11.0/24 detail 
11.11.11.0/24, epoch 0, flags [rib defined all labels]
  recursive via 4.4.4.4 label 24
    nexthop 10.1.1.2 Ethernet0/0 label 20-(local:23)
```

This means the label stack will look like this as PE1 sends the packet:

```text
Transport label: 20
VPN label:       24
```

---

## VPN Label Allocation Modes

A PE must advertise a VPN label with VPNv4 routes.

There are different ways to allocate those labels. Two common methods are:

```text
Per-prefix label allocation
Per-VRF label allocation
```

### Per-Prefix Label Allocation

With per-prefix allocation, the PE allocates a different VPN label for each VPN route.

Example:

```text
10.10.10.0/24 → label 18
10.10.20.0/24 → label 19
10.10.30.0/24 → label 20
```

The label directly identifies the route or forwarding entry on the egress PE.

This makes forwarding simple, but it consumes more labels.

Per-prefix label mode is the default for local routes in IOS XE due to this default command:

```
PE1# show run all | i label mode 
mpls label mode all-vrfs protocol all-afs per-prefix
```

### Per-VRF Label Allocation

With per-VRF allocation, the PE uses one VPN label for all local routes in a VRF.

Example:

```text
VRF CUSTOMER_A → label 33

10.10.10.0/24 → label 33
10.10.20.0/24 → label 33
10.10.30.0/24 → label 33
```

When the egress PE receives the packet, the VPN label identifies the VRF.

Then the PE performs an IP lookup in that VRF to forward the packet.

This uses fewer labels, but it requires an additional VRF IP lookup on the egress PE.

This is not the complete command syntax, but here are some of the options:

```text
mpls label mode {vrf <vrf-name> | all-vrfs} protocol bgp-vpnv4 {per-prefix | per-vrf}
```

Example:

```text
PE1(config)# mpls label mode all-vrfs protocol bgp-vpnv4 per-vrf
```

I restarted PE1 to clear the labels, and now this is what it looks like:

```
PE1#show mpls forwarding-table 
Local      Outgoing   Prefix           Bytes Label   Outgoing   Next Hop    
Label      Label      or Tunnel Id     Switched      interface              
16         20         4.4.4.4/32       0             Et0/0      10.1.1.2    
17         21         3.3.3.3/32       0             Et0/0      10.1.1.2    
18         Pop Label  2.2.2.2/32       0             Et0/0      10.1.1.2    
19         18         10.2.2.0/24      0             Et0/0      10.1.1.2    
20         Pop Label  10.1.2.0/24      0             Et0/0      10.1.1.2    
21         No Label   IPv4 VRF[V]      0             aggregate/CUSTOMER_A 
```

There is now a label (`21`) for the CUSTOMER_A VRF, rather than for any specific prefix in that VRF.

---

## VPN Labels Are Not Route Targets

Do not confuse VPN labels with route targets.

```text
Route target:
BGP extended community used for route import/export.

VPN label:
MPLS label used by the egress PE in the data plane.
```

The RT controls whether the route is imported into a VRF.

The VPN label is used after the route has been imported and traffic is forwarded.

```text
RT:
Should this route enter this VRF?

VPN label:
How should the egress PE forward packets for this route?
```

---

## VPN Labels vs Transport Labels

They are both MPLS labels, but do not confuse VPN labels with transport labels.

```text
Transport label:
Usually learned via LDP.
Gets the packet to the remote PE loopback.

VPN label:
Learned via MP-BGP.
Tells the egress PE how to forward the customer packet.
```

The transport label changes hop by hop.

The VPN label is advertised by the egress PE and is carried across the core until the packet reaches that PE.

Example:

```text
PE1 → P1:
Transport label 17, VPN label 24

P1 → P2:
Transport label 20, VPN label 24

P2 → PE2:
VPN label 24 only, if PHP removed the transport label
```

The transport label may change at each P router.

The VPN label remains the same until the egress PE receives it.

---

## Checking Local VPN Labels

Use the MPLS forwarding table to see local VPN labels.

Example:

```text
PE1# show mpls forwarding-table
```

Look for VRF routes marked with `[V]`.

```text
Local      Outgoing   Prefix           Bytes Label   Outgoing   Next Hop
Label      Label      or Tunnel Id     Switched      interface
18         No Label   10.10.10.0/24[V] 570           Et0/1      192.168.1.2
```

This tells you that PE1 allocated label `18` for `10.10.10.0/24`.

That is the label remote PEs should use when sending traffic to that route through PE1.

---

## Checking Remote VPN Labels

Use VPNv4 BGP label output to see labels learned from remote PEs.

Example:

```text
PE1# show bgp vpnv4 unicast all labels
```

For a remote route, look at the out label.

```text
11.11.11.0/24    5.5.5.5         nolabel/24
```

This tells you that PE1 will use VPN label `24` when sending traffic to `11.11.11.0/24`.

Then check CEF to confirm the full imposed stack.

```text
PE1# show ip cef vrf CUSTOMER_A 11.11.11.0/24 detail
11.11.11.0/24, epoch 0, flags [rib defined all labels]
  recursive via 4.4.4.4 label 21
    nexthop 10.1.1.2 Ethernet0/0 label 17-(local:18)
```

The full forwarding state must include:

```text
Transport label to the remote PE (21)
VPN label advertised by the remote PE (17)
```

---

## Key Points

* VPN labels are advertised by MP-BGP VPNv4 (or VPNv6).
* The egress PE allocates the VPN label for its local VPN route.
* The ingress PE uses the remote VPN label as the bottom label in the label stack.
* The transport label gets the packet to the remote PE.
* The VPN label tells the egress PE how to forward the packet after it arrives.
* P routers switch based on the top transport label.
* P routers do not need customer routes, VRFs, or MP-BGP.
* PHP may remove the transport label before the egress PE, but the VPN label remains.
* `show mpls forwarding-table` shows local VPN labels as `[V]` entries.
* `show bgp vpnv4 unicast all labels` shows local in VPN labels and remote out VPN labels.
* `show ip cef vrf <vrf> <prefix> detail` is useful for checking the imposed label stack.
* Per-prefix labels identify specific VPN routes or forwarding entries.
* Per-VRF labels identify the VRF, then the egress PE performs a VRF IP lookup.
* VPN labels are data-plane MPLS labels, different fromt route targets and route distinguishers.
