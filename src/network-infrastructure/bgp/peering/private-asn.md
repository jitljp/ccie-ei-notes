# 1.5.a (vi) Private AS Numbers

**Private Autonomous System Numbers (ASNs)** are reserved for use within private networks and do not require assignment by a Regional Internet Registry (RIR).

Like private IPv4 addresses, private ASNs can be reused by different organizations because they are not globally unique.

Private ASNs are commonly used for:

- EBGP between an enterprise and its ISP.
- EBGP between customer sites and a service provider.
- Internal EBGP designs, including data center networks.
- Lab environments.

Private ASNs operate like public ASNs in BGP. They support normal EBGP peering, AS_PATH processing, route advertisement, and loop prevention.

However, **private ASNs must not appear in AS_PATH attributes advertised to the global Internet**.

---

## Private ASN Ranges

[RFC 6996](https://www.rfc-editor.org/rfc/rfc6996.html) defines two private ASN ranges.

| ASN Type | Private Range | Number of ASNs |
|---|---|---:|
| 2-byte | 64512-65534 | 1,023 |
| 4-byte | 4200000000-4294967294 | 94,967,295 |

The 4-byte private ASN range provides a significantly larger pool of ASNs for private BGP deployments.

### Important Reserved Values

Not every ASN outside the private ranges is publicly assignable.

| ASN or Range | Purpose |
|---|---|
| 0 | Reserved |
| 23456 | AS_TRANS, used for 4-byte ASN compatibility |
| 64496-64511 | Documentation ASNs |
| 64512-65534 | Private 2-byte ASNs |
| 65535 | Reserved |
| 65536-65551 | Documentation ASNs |
| 4200000000-4294967294 | Private 4-byte ASNs |
| 4294967295 | Reserved |

**65535 is not a private ASN.** It is reserved.

Likewise, **23456 (AS_TRANS)** is not a private ASN.

---

## Private ASNs in EBGP

A private ASN can be configured using the normal BGP commands.

Consider an enterprise using private AS 65001 and an ISP using public AS 100.

### Topology

```text
        Enterprise                     ISP
         AS 65001                    AS 100

           R1 ------------------------- R2
         E0/0                         E0/0
      10.0.12.1/30                 10.0.12.2/30

      Lo0: 192.0.2.1/24
```

### R1 Configuration

```text
R1(config)# interface Ethernet0/0
R1(config-if)# ip address 10.0.12.1 255.255.255.252
R1(config-if)# no shutdown

R1(config)# interface Loopback0
R1(config-if)# ip address 192.0.2.1 255.255.255.0

R1(config)# router bgp 65001
R1(config-router)# bgp router-id 1.1.1.1
R1(config-router)# neighbor 10.0.12.2 remote-as 100
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 activate
R1(config-router-af)# network 192.0.2.0 mask 255.255.255.0
```

### R2 Configuration

```text
R2(config)# interface Ethernet0/0
R2(config-if)# ip address 10.0.12.2 255.255.255.252
R2(config-if)# no shutdown

R2(config)# router bgp 100
R2(config-router)# bgp router-id 2.2.2.2
R2(config-router)# neighbor 10.0.12.1 remote-as 65001
R2(config-router)# address-family ipv4 unicast
R2(config-router-af)# neighbor 10.0.12.1 activate
```

The EBGP session establishes normally.

R2 learns `192.0.2.0/24` with:

```text
AS_PATH: 65001
```

**Private ASNs are not automatically removed when routes are learned through EBGP.**

They are included in AS_PATH just like public ASNs.

---

## Private ASNs and AS_PATH

When a route is advertised between autonomous systems, the advertising router normally prepends its local ASN to AS_PATH.

This applies to private and public ASNs.

For example:

```text
         AS 65001              AS 65002             AS 100

            R1 ------------------- R2 ----------------- R3

       192.0.2.0/24
```

R1 originates `192.0.2.0/24`.

R2 receives:

```text
AS_PATH: 65001
```

R3 receives:

```text
AS_PATH: 65002 65001
```

Both ASNs remain in the path, even though both are private.

This is normal BGP behavior.

### Loop Prevention

Private ASNs participate in AS_PATH loop prevention.

For example, if R2 in AS 65002 receives a route containing its own ASN:

```text
AS_PATH: 65001 65002 65003
```

R2 normally rejects the path because its own ASN appears in AS_PATH.

**Private ASNs do not disable or change normal AS_PATH loop prevention.**

---

## Private ASNs and the Global Internet

Private ASNs are not globally unique.

For example, two unrelated enterprises might both use AS 65001.

```text
    Enterprise A                                  Enterprise B
      AS 65001                                      AS 65001
          |                                             |
          R1                                            R4
          |                                             |
       AS 100                 Internet                AS 200
          R2 ------------------------------------------- R3
```

If both enterprises advertise routes containing AS 65001 across the Internet, AS_PATH loop prevention and routing policies could produce unexpected results.

For example, Enterprise B might receive a route with:

```text
AS_PATH: 200 100 65001
```

Because Enterprise B also uses AS 65001, its BGP router normally rejects the route.

Additionally, Internet providers commonly filter routes containing private ASNs.

For these reasons, RFC 6996 requires private ASNs to be removed from AS path attributes before routes are advertised to the global Internet.

**Private ASNs are valid within private BGP networks, but must not be exposed in global Internet AS_PATH advertisements.**

---

## Removing Private ASNs

Cisco IOS XE supports removing private ASNs from outgoing EBGP route advertisements.

The primary command is:

```text
R1(config-router)# neighbor <IP> remove-private-as
```

This is configured on the router advertising the route toward the external neighbor.

### Example

Consider:

```text
        Enterprise                  ISP                 Internet
         AS 65001                 AS 100                AS 200

            R1 --------------------- R2 ------------------- R3
         10.0.12.1              10.0.12.2
                                10.0.23.2           10.0.23.3
```

R1 originates `192.0.2.0/24`.

R2 receives:

```text
AS_PATH: 65001
```

Without private ASN removal, R3 receives:

```text
AS_PATH: 100 65001
```

To prevent the private ASN from appearing in advertisements to R3, configure R2:

```text
R2(config)# router bgp 100
R2(config-router)# address-family ipv4 unicast
R2(config-router-af)# neighbor 10.0.23.3 remove-private-as
```

R3 now receives:

```text
AS_PATH: 100
```

The private ASN `65001` has been removed.

**The command affects outbound EBGP advertisements toward the specified neighbor.**

It does not remove ASNs from R2's BGP table.

### Important Restrictions

Cisco documents the following restrictions:

- `remove-private-as` applies to EBGP neighbors, not IBGP neighbors.
- The advertising router must normally use a public ASN.
- A supported `neighbor local-as` configuration using a public ASN can provide an alternative in some designs.

For example, if R2 itself uses private AS 65002, it cannot simply remove its own AS from the normal EBGP advertisement using `remove-private-as`.

The feature is intended to remove private ASNs from paths advertised by a public-AS-facing router.

---

## Removing Private ASNs from Mixed AS_PATHs

A route might contain both public and private ASNs.

For example:

```text
AS_PATH: 65001 200 65002
```

Here:

- 65001 is private.
- 200 is public.
- 65002 is private.

Modern Cisco IOS XE supports removing private ASNs from mixed public/private AS_PATHs.

For example:

```text
R1(config-router-af)# neighbor 10.0.12.2 remove-private-as all
```

The received AS_PATH at the external neighbor would contain the advertising router's public ASN and the retained public ASNs.

Assuming R1 belongs to AS 100:

```text
Before outbound processing:
65001 200 65002

After removal and EBGP advertisement:
100 200
```

### Historical Behavior

Older Cisco IOS versions had restrictions on private ASN removal.

In particular, private ASNs were not removed if the AS_PATH contained both public and private ASNs.

Cisco IOS XE 3.1S and later introduced enhanced removal behavior, including support for mixed AS_PATHs.

**Do not assume the old mixed-AS_PATH restriction applies to modern IOS XE.**

---

## Replacing Private ASNs

Cisco IOS XE can replace private ASNs with the advertising router's local ASN instead of removing them entirely.

Configure:

```text
R1(config-router-af)# neighbor <IP> remove-private-as all replace-as
```

For example, R1 belongs to public AS 100.

The original AS_PATH is:

```text
65001 65002 65003
```

Using `remove-private-as all` removes the private ASNs:

```text
AS_PATH advertised: 100
```

Using `remove-private-as all replace-as` replaces each private ASN with the local ASN:

```text
AS_PATH advertised: 100 100 100 100
```

The first `100` represents the normal EBGP prepend of the advertising router's local ASN.

The other three occurrences replace the private ASNs.

**The `replace-as` option preserves AS_PATH length**, which may be useful when AS_PATH length affects route selection.

| Command | Effect |
|---|---|
| `remove-private-as` | Remove private ASNs from outgoing EBGP advertisements |
| `remove-private-as all` | Explicitly remove all private ASNs, including in mixed paths |
| `remove-private-as all replace-as` | Replace private ASNs with the local public ASN |

The detailed behavior, including configuration examples and AS_PATH manipulation scenarios, is covered in [Remove-Private-AS](../as-path/as-number-manipulation/remove-private-as.md).

---

## Private 4-Byte ASNs

The 4-byte private ASN range is:

```text
4200000000-4294967294
```

These ASNs provide a much larger address space for private networks.

They are particularly useful in environments with large numbers of independent autonomous systems, such as EBGP-based data center fabrics.

### Configuration Example

```text
R1(config)# router bgp 4200000001
R1(config-router)# neighbor 10.0.12.2 remote-as 4200000002
```

R2:

```text
R2(config)# router bgp 4200000002
R2(config-router)# neighbor 10.0.12.1 remote-as 4200000001
```

No additional command is needed to enable 4-byte ASN support on modern IOS XE.

### 4-Byte ASN Compatibility

When using private 4-byte ASNs:

- BGP speakers should support the 4-byte ASN capability defined in RFC 6793.
- Legacy BGP speakers may represent 4-byte ASNs using AS_TRANS.
- Private ASN removal implementations must recognize the 4-byte private range.
- AS4_PATH information must also be handled correctly when routes traverse legacy BGP speakers.

See [4-Byte AS Numbers](four-byte-asn.md) for details.

---

## Private ASNs in Enterprise and Data Center Networks

Private ASNs are frequently used in enterprise networks that run EBGP internally.

For example, an EBGP-based data center fabric might use separate private ASNs for spine and leaf switches.

```text
                AS 65000           AS 65001
                  SPINE1             SPINE2
                    | \             / |
                    |  \           /  |
                    |   \         /   |
                    |    \       /    |
                    |     \     /     |
                    |      \   /      |
                    |       \ /       |
                    |       / \       |
                    |      /   \      |
                    |     /     \     |
                    LEAF1        LEAF2
                   AS 65101     AS 65102
```

Each leaf can establish EBGP sessions with both spines.

For example, LEAF1 uses AS 65101 and peers with:

- SPINE1 in AS 65000.
- SPINE2 in AS 65001.

Private ASNs allow the fabric to use multiple autonomous systems without requiring globally assigned ASNs.

When advertising fabric routes outside the private network, border routers can remove private ASNs as appropriate.

---

## Verification

### Display the Local ASN and BGP Neighbors

```text
R1# show ip bgp summary
```

Verify the local ASN and remote neighbor ASNs.

### Display AS_PATH Information

```text
R1# show ip bgp
R1# show ip bgp 192.0.2.0
```

Check whether private ASNs appear in received routes.

For example:

```text
AS_PATH: 65002 65001
```

Both values belong to the private 2-byte range.

### Verify Outbound AS_PATH Manipulation

Use:

```text
R1# show ip bgp neighbors 10.0.12.2 advertised-routes
```

This displays routes advertised toward the neighbor.

To verify the exact AS_PATH received after private ASN removal, check the BGP table on the receiving router:

```text
R2# show ip bgp 192.0.2.0
```

This is important because the advertising router's local BGP table continues to retain the original received AS_PATH.

### Display Removal Configuration

```text
R1# show running-config | section router bgp
R1# show ip bgp neighbors 10.0.12.2
R1# show ip bgp update-group
```

The detailed neighbor and update-group outputs can help confirm whether private ASN removal or replacement is configured.

---

## Command Summary

| Command | Purpose |
|---|---|
| `router bgp <PRIVATE-ASN>` | Start BGP using a private ASN |
| `neighbor <IP> remote-as <PRIVATE-ASN>` | Configure a BGP neighbor in a private AS |
| `neighbor <IP> remove-private-as` | Remove private ASNs from outbound EBGP advertisements |
| `neighbor <IP> remove-private-as all` | Remove all private ASNs, including in mixed AS_PATHs |
| `neighbor <IP> remove-private-as all replace-as` | Replace private ASNs with the local public ASN |
| `no neighbor <IP> remove-private-as` | Disable private ASN removal |
| `show ip bgp summary` | Display local and neighbor ASNs |
| `show ip bgp <PREFIX>` | Inspect the AS_PATH of a route |
| `show ip bgp neighbors <IP> advertised-routes` | Display routes advertised to a neighbor |

---

## Key Points

- Private ASNs can be reused because they are **not globally unique**.
- The private 2-byte ASN range is **64512-65534**.
- The private 4-byte ASN range is **4200000000-4294967294**.
- **65535 is reserved**, not private.
- Private ASNs operate like public ASNs within BGP, including AS_PATH loop prevention.
- Private ASNs must be removed from AS_PATH attributes before routes are advertised to the global Internet.
- Cisco IOS XE supports `neighbor remove-private-as` for outbound EBGP advertisements.
- `remove-private-as all` supports removing private ASNs from mixed public/private paths.
- `remove-private-as all replace-as` replaces private ASNs with the local public ASN, preserving AS_PATH length.
- Private ASNs are frequently used for enterprise-to-ISP peering and internal EBGP designs.

---

## Cisco Documentation

- [Cisco IOS XE 17.x: Removing Private AS Numbers from the AS Path in BGP](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-routing/b-ip-routing/m_irg-remove-as-0.html)
- [Cisco IOS BGP Command Reference: neighbor remove-private-as](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_bgp/command/irg-cr-book/bgp-m1.html)
- [RFC 6996: Autonomous System (AS) Reservation for Private Use](https://www.rfc-editor.org/rfc/rfc6996.html)
- [IANA: Special-Purpose Autonomous System Numbers](https://www.iana.org/assignments/iana-as-numbers-special-registry/)
- [RFC 6793: BGP Support for Four-Octet ASN Space](https://www.rfc-editor.org/rfc/rfc6793.html)
