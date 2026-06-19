# Domain Tag

The **OSPF Domain Tag** is a loop prevention mechanism used with OSPF external routes in MPLS L3VPN PE-CE designs.

It is also called the **VPN Route Tag**.

The Domain Tag is used when a PE advertises a VPN-learned route into OSPF as an external route.

It applies to:

```text
Type 5 external LSAs
Type 7 NSSA external LSAs
```

It does not apply to Type 3 summary LSAs.

I'll use the same topology as in the Down Bit page:

```
CE1 --- PE1 === MPLS VPN === PE2 --- CE2
                  ||            \    /
                  ||              CE3
                  PE3______________/
```

---

## Why the Domain Tag Exists

In MPLS L3VPN PE-CE OSPF, a route can move between OSPF and MP-BGP.

Example:

```text
CE1
  ↓ OSPF
PE1
  ↓ VPNv4
PE2
  ↓ OSPF
CE2
```

If the customer network has multiple PE connections, the same route might return to another PE through OSPF.

Example:

```text
1. CE1 advertises an external route to PE1.
2. PE1 exports the route into VPNv4.
3. PE2 imports the VPNv4 route.
4. PE2 advertises it into OSPF toward CE2 as a Type 5 LSA.
5. CE2 floods the Type 5 LSA through the customer OSPF domain.
6. PE3 receives the Type 5 LSA from the CE side.
```

Without loop prevention, PE3 might think the route is a new customer OSPF external route and advertise it back into VPNv4.

That could create a routing loop.

The Domain Tag prevents this.

```text
PE-originated external LSA:
Set the VPN Route Tag.

Another PE receives that LSA from the CE side:
If the tag matches the local VPN Route Tag, ignore it for route calculation.
```

---

## Domain Tag vs Down Bit

The Down Bit and Domain Tag are both loop-prevention mechanisms.

| Mechanism | Where it is carried | Main use |
| :--- | :--- | :--- |
| **Down Bit** | OSPF LSA Options field | Loop prevention for VPN-learned LSAs |
| **Domain Tag / VPN Route Tag** | OSPF external route tag field | Loop prevention for Type 5 and Type 7 external LSAs |

The Down Bit can be set on PE-originated Type 3, Type 5, and Type 7 LSAs.

The Domain Tag is only relevant for external LSAs.

```text
Type 3:
Uses Down Bit.

Type 5:
Uses Down Bit and Domain Tag.

Type 7:
Uses Down Bit and Domain Tag.
```

The Domain Tag exists partly for backward compatibility with older implementations that did not set the Down Bit on Type 5 LSAs.

```text
Down Bit:
General OSPF L3VPN loop prevention bit.

Domain Tag:
External-route loop prevention tag.
```

---

## Where the Domain Tag Is Seen

The Domain Tag is seen in the OSPF database as the external LSA tag.

Example:

```text
CE2# show ip ospf database | beg External
                Type-5 AS External Link States

Link ID         ADV Router      Age         Seq#       Checksum Tag
10.10.20.0      192.168.2.1     931         0x80000004 0x00BD89 3489725928
10.10.20.0      192.168.3.1     62          0x80000004 0x00B68F 3489725928
```

10.10.20.0/24 is an external route advertised by CE1.

CE2 learns the route from two PEs:
- PE2 (192.168.2.1)
- PE3 (192.168.3.1)

The important field is:

```text
Tag:
3489725928
```

That tag identifies the route as having been originated into OSPF by a PE after being learned from the VPN backbone.

The CE may still install and use the route normally.

The loop-prevention behavior is on PE routers.

---

## How the PE Uses the Domain Tag

When a PE receives a Type 5 LSA from a CE, it checks the external route tag.

If the tag matches the local VPN Route Tag, the PE does not use that LSA for route calculation.

```text
PE receives Type 5 LSA from CE.
↓
LSA tag matches the VPN Route Tag.
↓
PE does not use the LSA for OSPF route calculation.
↓
Route is not installed from that LSA into the VRF routing table.
↓
Route is not redistributed back into VPNv4.
```

This is similar to the Down Bit behavior.

The LSA can still be visible in the LSDB.

But the PE must not use that LSA as a usable route source.

---

## Example Loop Prevention Flow

CE1 originates a Type 5 LSA:

```text
Link ID: 10.10.20.0
Adv Router: 10.10.10.1
Tag: 0
```

PE1 learns the route and exports it into VPNv4.

PE2 imports the VPNv4 route and advertises it into OSPF as a Type 5 LSA.

PE2 sets the Domain Tag.

```text
Link ID: 10.10.20.0
Adv Router: 192.168.2.1
Tag: 3489725928
```

If PE3 later receives this LSA from a CE, PE3 sees the tag.

```text
Received Type 5 LSA tag:
3489725928

Local VPN Route Tag:
3489725928
```

Result:

```text
Tag matches.
Do not use the LSA for route calculation.
Do not re-export it into VPNv4.
```

This prevents the route from looping back into the VPN.

---

## Default Domain Tag

On Cisco IOS XE, the Domain Tag can be configured manually.

If it is not configured, IOS XE can generate one automatically.

Example:

```text
Tag:
3489725928
```

In hexadecimal:

```text
3489725928 = 0xD000FDE8
```

For BGP AS 65000:

```text
65000 = 0xFDE8
```

So the automatically generated domain tag was derived from the PE routers' BGP ASN.

```text
0xD000FDE8
      FDE8 = 65000
```

This is why the same tag appears by default on PE-originated Type 5 LSAs in the same provider AS.

---

## Configuring the Domain Tag

You can manually configure the Domain Tag under the OSPF process.

Example on PE2:

```text
router ospf 10 vrf CUSTOMER_A
 domain-tag 12345
```

The configured tag is used when the PE originates Type 5 or Type 7 LSAs for VPN-learned external routes.

Example:

```text
PE2 receives VPNv4 external route.
PE2 redistributes it into OSPF.
PE2 originates Type 5 LSA.
PE2 sets route tag 12345.
```

Verification:

```text
CE2# show ip ospf database | beg External
                Type-5 AS External Link States

Link ID         ADV Router      Age         Seq#       Checksum Tag
10.10.20.0      192.168.2.1     20          0x80000005 0x007A1A 12345     
10.10.20.0      192.168.3.1     521         0x80000004 0x00B68F 3489725928
```

PE2 has advertised the LSA with the configured tag `12345`.

> This does **not normally break loop prevention** on modern PE routers, because the DN bit is still set on the PE-originated Type 5 LSA.
>
> However, the Domain Tag check only works if the receiving PE compares the LSA tag to the same VPN Route Tag value.
>
> If PE2 uses `domain-tag 12345` but PE3 uses the default tag, PE3 can still ignore PE2's LSA because of the DN bit. But PE3 would not ignore it based on the Domain Tag alone.
>
> For a clean design, use the same Domain Tag on all PEs in the same VPN/OSPF domain.


---

## When to Configure a Domain Tag Manually

Configure the Domain Tag manually when you need predictable tag values, such as when the lab specifies it as a requirement.

The Domain Tag must be distinct from normal customer route tags, if the customer tags their external LSAs.

If a customer-originated external route uses the same tag as the PE's VPN Route Tag, a PE might ignore it.

That would cause the PE to not install the route from OSPF and not export it into VPNv4.

Example: Redistribute 10.10.20.0/24 on CE1 with a route tag:

```
CE1(config-router)#redistribute connected metric-type 1 tag 3489725928
```

PE1 receives the LSA but does not add it to VPNv4:

```
PE1# sh ip ospf database | beg External 
                Type-5 AS External Link States

Link ID         ADV Router      Age         Seq#       Checksum Tag
10.10.20.0      10.10.10.1      71          0x8000000B 0x00FB1C 3489725928

PE1# sh bgp vpnv4 unicast all 10.10.20.0
% Network not in table
```

As a result it is not advertised to other PEs, can CE2 does not learn the route:

```
CE2# show ip route 10.10.20.0
% Subnet not in table
```

---

## Domain Tag and Domain ID

Do not confuse Domain Tag with Domain ID.

| Feature | Purpose |
| :--- | :--- |
| **Domain ID** | Determines whether a route came from the same OSPF domain |
| **Domain Tag** | Prevents external VPN-learned routes from being re-used by another PE |

The Domain ID affects how the PE represents routes when advertising them into OSPF.

Example:

```text
Same Domain ID:
Internal routes can be advertised as inter-area.

Different Domain ID:
Internal routes are advertised as external.
```

The Domain Tag is loop prevention for external LSAs.

Example:

```text
PE-originated Type 5 or Type 7 LSA:
Set the VPN Route Tag.

Another PE receives it from OSPF:
Ignore it for route calculation.
```

---

## Common Problems

### Customer route tag matches the VPN Route Tag

If a customer external route uses the same tag as the PE's VPN Route Tag, the PE may ignore it.

Example:

```text
Customer external LSA tag:
3489725928

PE VPN Route Tag:
3489725928
```

Result:

```text
PE treats the route as VPN-originated.
PE does not use it for route calculation.
PE does not export it into VPNv4.
```

Fix:

```text
Use a different customer route tag.

or

Configure a different domain-tag on the PE.
```

### External route is in the LSDB but not in the PE routing table

Check whether the route has the VPN Route Tag.

```text
show ip ospf database external <prefix>
```

If the tag matches the PE's VPN Route Tag, the PE ignores the route for route calculation.

### Manually configured domain tags do not match

If you configure the Domain Tag manually, use a consistent design.

Different PEs in the same VPN should normally use the same VPN Route Tag for the same customer OSPF domain.

If PE2 tags VPN external routes with `12345`, but PE3 expects `54321`, PE3 may not recognize the route as VPN-originated.

---

## Key Points

* The Domain Tag is also called the VPN Route Tag.
* It is used with OSPF external routes in MPLS L3VPN PE-CE designs.
* It applies to Type 5 and Type 7 LSAs.
* It does not apply to Type 3 LSAs.
* PE-originated external LSAs for VPN-learned routes should include the VPN Route Tag.
* If another PE receives a Type 5 or Type 7 LSA from a CE with the VPN Route Tag, it ignores that LSA for route calculation.
* The LSA may still be visible in the LSDB.
* The route is not installed from that LSA into the VRF routing table.
* Because the route is not installed from that LSA, it is not exported back into VPNv4.
* The default Cisco-generated tag is derived from the BGP AS number.
* The Domain Tag is different from the Domain ID.
* The Domain ID affects route type translation.
* The Domain Tag prevents loops for external OSPF routes.
