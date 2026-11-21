# BGP Route Refresh

**BGP Route Refresh** allows a BGP router to request that a neighbor readvertise routing information without resetting the BGP session.

It is defined in **RFC 2918** and is primarily used to reapply inbound routing policies after configuration changes.

For example, if an administrator modifies an inbound prefix list, Route Refresh allows the router to request the neighbor's routes again and process them using the updated policy.

The primary benefits are:

- **Non-disruptive policy changes:** Inbound policies can be reapplied without terminating the BGP session.
- **Reduced memory consumption:** The router does not need to maintain separate, unmodified copies of received routes.
- **Reevaluation of rejected routes:** Routes previously rejected by inbound policies can be reconsidered.
- **Support for multiple address families:** Route Refresh operates on a per-AFI/SAFI basis.

Route Refresh is supported by modern Cisco IOS XE BGP implementations and is generally preferred over traditional inbound soft reconfiguration.

---

## Why Route Refresh Is Needed

Consider the following topology:

```text
        AS 65001                       AS 65002

           R1 --------------------------- R2
        10.0.12.1                     10.0.12.2
                        EBGP
```

R2 advertises three prefixes:

```text
192.0.2.0/24
198.51.100.0/24
203.0.113.0/24
```

R1 initially accepts only `192.0.2.0/24`.

```text
R1(config)# ip prefix-list FROM-R2 permit 192.0.2.0/24

R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 prefix-list FROM-R2 in
```

The result is:

| Prefix | Inbound Policy | Result |
|---|---|---|
| 192.0.2.0/24 | Permit | Accepted |
| 198.51.100.0/24 | Implicit deny | Rejected |
| 203.0.113.0/24 | Implicit deny | Rejected |

The rejected routes are not available for R1's normal BGP best-path selection.

### Changing the Inbound Policy

Suppose R1's administrator decides to accept `198.51.100.0/24` as well.

```text
R1(config)# ip prefix-list FROM-R2 permit 198.51.100.0/24
```

The prefix list now permits two prefixes.

However, R1 does not necessarily have the original information for `198.51.100.0/24` available locally.

The route was previously rejected, so R1 needs to obtain or reprocess it.

Without Route Refresh or traditional inbound soft reconfiguration, a hard reset may be required.

Route Refresh provides a better solution by allowing R1 to request the routing information again without terminating the session.

---

## Route Refresh Operation

Route Refresh uses a dedicated **ROUTE-REFRESH message** to request routing information from a BGP neighbor.

When a router receives a valid ROUTE-REFRESH request, it readvertises the applicable routing information to the requesting neighbor.

### Message Exchange

Using the previous topology:

```text
       R1                               R2
        |                                |
        |       BGP Established          |
        |<------------------------------>|
        |                                |
        |   Inbound Policy Modified      |
        |                                |
        |------ ROUTE-REFRESH ---------->|
        |                                |
        |<----------- UPDATE ------------|
        |<----------- UPDATE ------------|
        |<----------- UPDATE ------------|
        |                                |
        |   Apply Updated Inbound Policy |
        |                                |
```

The process is:

1. R1 modifies its inbound routing policy.
2. R1 initiates an inbound soft reset.
3. R1 sends a ROUTE-REFRESH message to R2.
4. R2 reexamines the routes eligible for advertisement to R1.
5. R2 readvertises the applicable routes using BGP UPDATE messages.
6. R1 processes the received information using its current inbound policy.
7. The BGP session remains Established.

**Route Refresh does not terminate the BGP TCP connection or renegotiate the BGP session.**

### What Routes Are Readvertised?

RFC 2918 specifies that the responding BGP speaker readvertises the applicable contents of its **Adj-RIB-Out**, subject to its outbound routing policies.

This is an important distinction.

Route Refresh does not cause the neighbor to advertise every route in its BGP table.

It readvertises routes that are eligible for advertisement to the requesting neighbor.

For example, if R2 has an outbound prefix list that denies `203.0.113.0/24`, a Route Refresh request from R1 does not bypass that outbound filter.

**Route Refresh reprocesses inbound routing information on the requesting router. It does not bypass the advertising router's outbound policies.**

---

## BGP Route Refresh Capability

Before Route Refresh can be used, the routers must negotiate support for the feature.

BGP capabilities are advertised using **Capability Optional Parameters** in BGP OPEN messages.

The standard Route Refresh capability is defined by RFC 2918.

| Field | Value |
|---|---|
| Capability | Route Refresh |
| Capability Code | 2 |
| Capability Length | 0 |
| Defined in | RFC 2918 |

The capability contains no additional capability-specific data.

### Capability Negotiation

When R1 and R2 establish their BGP session, they exchange OPEN messages.

```text
        R1                            R2
         |                             |
         |---------- OPEN ------------>|
         |      Route Refresh          |
         |       Capability 2          |
         |                             |
         |<--------- OPEN -------------|
         |      Route Refresh          |
         |       Capability 2          |
         |                             |
         |       Established           |
```

By advertising the Route Refresh capability, a router indicates that it can **receive and process ROUTE-REFRESH requests** from its neighbor.

For R1 to send a ROUTE-REFRESH request to R2, R1 must have received the Route Refresh capability from R2.

### Verification

On R1:

```text
R1# show ip bgp neighbors 10.0.12.2
```

Look for output similar to:

```text
Neighbor capabilities:
  Route refresh: advertised and received(new)
```

The meanings are:

- **advertised:** R1 advertised Route Refresh support to R2.
- **received:** R1 received Route Refresh support from R2.

The `(new)` indication identifies the standard Route Refresh capability.

If R2 does not advertise Route Refresh support, the output might instead show:

```text
Neighbor capabilities:
  Route refresh: advertised
```

In this case, R1 cannot use standard Route Refresh to request routes from R2.

**The important requirement for sending a ROUTE-REFRESH request is that the remote neighbor has advertised the capability.**

### Enabling Route Refresh

On modern Cisco IOS XE platforms, standard Route Refresh support is normally enabled automatically.

No neighbor-specific command is required to enable the capability.

For example:

```text
R1(config)# router bgp 65001
R1(config-router)# neighbor 10.0.12.2 remote-as 65002
```

Assuming normal capability negotiation, R1 advertises its supported BGP capabilities during session establishment.

The neighbor's capabilities determine whether Route Refresh can be used.

---

## ROUTE-REFRESH Message Format

RFC 2918 defines the **ROUTE-REFRESH message** as BGP message type 5.

| BGP Message | Type |
|---|---|
| OPEN | 1 |
| UPDATE | 2 |
| NOTIFICATION | 3 |
| KEEPALIVE | 4 |
| ROUTE-REFRESH | 5 |

A ROUTE-REFRESH message contains the standard 19-byte BGP message header followed by a 4-byte payload.

The minimum message length is therefore **23 bytes**.

### Message Structure

```text
+--------------------------------+
| BGP Message Header (19 bytes)  |
+--------------------------------+
| AFI                  (2 bytes) |
+--------------------------------+
| Reserved/Subtype     (1 byte)  |
+--------------------------------+
| SAFI                 (1 byte)  |
+--------------------------------+
```

The fields are:

| Field | Purpose |
|---|---|
| AFI | Identifies the Address Family |
| Reserved/Subtype | Reserved in RFC 2918; interpreted as a subtype by RFC 7313 |
| SAFI | Identifies the Subsequent Address Family |

For a normal Route Refresh request, the Reserved/Subtype field has a value of 0.

### AFI and SAFI

Route Refresh operates on a per-address-family basis.

For example:

| Address Family | AFI | SAFI |
|---|---:|---:|
| IPv4 Unicast | 1 | 1 |
| IPv6 Unicast | 2 | 1 |
| VPNv4 Unicast | 1 | 128 |
| VPNv6 Unicast | 2 | 128 |

A Route Refresh request for IPv4 unicast does not inherently request routes belonging to IPv6 unicast or VPNv4.

This allows policies to be reapplied for one address family without refreshing unrelated address families.

**The requested AFI/SAFI must be supported by the relevant BGP session.**

---

## Initiating Route Refresh on Cisco IOS XE

The most common reason to initiate Route Refresh is to apply updated inbound routing policies.

### Inbound Soft Reset

Use:

```text
R1# clear ip bgp 10.0.12.2 soft in
```

When the neighbor supports Route Refresh and traditional inbound route storage is not being used, IOS XE sends a ROUTE-REFRESH request.

The neighbor readvertises its routing information, and the local router processes it using its current inbound policies.

The BGP session remains Established.

### Alternative Syntax

Cisco IOS XE also supports:

```text
R1# clear ip bgp 10.0.12.2 in
```

When Route Refresh is supported, the `in` keyword can initiate a dynamic inbound soft reset without explicitly specifying `soft`.

Cisco documents this behavior in the BGP command reference.

For clarity, this book generally uses:

```text
clear ip bgp <NEIGHBOR> soft in
```

The `soft` keyword explicitly identifies the intended operation as a soft reset.

### Address-Family-Specific Commands

For IPv4 unicast:

```text
R1# clear ip bgp ipv4 unicast 10.0.12.2 soft in
```

For IPv6 unicast:

```text
R1# clear bgp ipv6 unicast 2001:db8:12::2 soft in
```

Exact command syntax varies by address family and IOS XE release.

### Important Distinction

The command:

```text
R1# clear ip bgp 10.0.12.2
```

Performs a **hard reset**.

The command:

```text
R1# clear ip bgp 10.0.12.2 soft in
```

Performs an **inbound soft reset**.

These operations have very different effects.

See [BGP Session Reset and Clearing](session-reset.md) for a detailed comparison.

---

## Route Refresh Configuration Example

Consider the following topology:

```text
       AS 65001                      AS 65002

          R1 -------------------------- R2
      10.0.12.1                    10.0.12.2
                      EBGP
```

R2 advertises:

```text
2.2.2.2/32
22.22.22.22/32
```

R1 initially permits only `2.2.2.2/32`.

### R1 Configuration

```text
R1(config)# ip prefix-list FROM-R2 permit 2.2.2.2/32

R1(config)# router bgp 65001
R1(config-router)# neighbor 10.0.12.2 remote-as 65002
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 prefix-list FROM-R2 in
```

### Initial Verification

Check the accepted routes:

```text
R1# show ip bgp neighbors 10.0.12.2 routes
```

The output contains only the permitted prefix:

```text
2.2.2.2/32
```

The route `22.22.22.22/32` was rejected by the inbound prefix list.

### Modify the Inbound Policy

Permit the second prefix:

```text
R1(config)# ip prefix-list FROM-R2 permit 22.22.22.22/32
```

Initiate an inbound soft reset:

```text
R1# clear ip bgp 10.0.12.2 soft in
```

Provided R2 supports Route Refresh, R1 requests the IPv4 unicast routing information again.

R2 readvertises its eligible routes.

R1 processes the received information using the modified prefix list.

### Verify the Result

```text
R1# show ip bgp neighbors 10.0.12.2 routes
```

Both prefixes should now be accepted:

```text
2.2.2.2/32
22.22.22.22/32
```

Check the neighbor state:

```text
R1# show ip bgp summary
```

The BGP session should remain Established throughout the operation.

**No hard reset or TCP session reestablishment is required.**

---

## Route Refresh vs. Traditional Inbound Soft Reconfiguration

Traditional inbound soft reconfiguration and Route Refresh allow inbound policies to be reapplied without terminating the BGP session.

The difference is how the router obtains the original routing information.

| Feature | Traditional Inbound Soft Reconfiguration | Route Refresh |
|---|---|---|
| Stores unmodified received routes locally | Yes | No additional stored copy required |
| Requires Route Refresh capability | No | Yes |
| Requests routes again from neighbor | No, when stored information is available | Yes |
| Reapplies inbound policies | Yes | Yes |
| Can reconsider previously rejected routes | Yes, if stored | Yes |
| Resets BGP session | No | No |
| Additional memory for received routes | Yes | Not required |
| Preferred in modern networks | Usually unnecessary | Generally preferred |

### With Traditional Inbound Soft Reconfiguration

Configure:

```text
R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 soft-reconfiguration inbound
```

The router stores unmodified received route information.

When an inbound soft reset is performed, the stored routes can be reprocessed locally.

### With Route Refresh

No traditional inbound soft reconfiguration configuration is required.

When an inbound soft reset is performed, the router requests the routing information from its neighbor.

**Route Refresh avoids the need to store an additional copy of received routes solely for policy reprocessing.**

See [BGP Soft Reconfiguration](soft-reconfiguration.md) for details.

---

## Route Refresh and Soft Reconfiguration Backup

Cisco IOS XE supports the `bgp soft-reconfig-backup` command, which allows traditional inbound soft reconfiguration to be used when Route Refresh is unavailable.

For example:

```text
R1(config)# router bgp 65001
R1(config-router)# bgp soft-reconfig-backup
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 soft-reconfiguration inbound
```

When both commands are configured, the router can select the appropriate mechanism based on the neighbor's Route Refresh capability.

| Route Refresh Supported? | Behavior |
|---|---|
| Yes | Use Route Refresh without maintaining additional unmodified received-route copies |
| No | Store unmodified received routes for traditional inbound soft reconfiguration |

On the tested Cisco IOS XE image, both commands were necessary to enable the selective backup behavior.

### Verification

If the neighbor supports Route Refresh, executing:

```text
R1# show ip bgp neighbors 10.0.12.2 received-routes
```

Can return:

```text
% Inbound soft reconfiguration is in backup mode on 10.0.12.2
```

This indicates that traditional inbound route storage is not active because Route Refresh is available.

If the neighbor does not support Route Refresh, the router stores the unmodified received routes, and the command can display them.

### Configuration Comparison

| Configuration | Behavior |
|---|---|
| Neither command configured | Use Route Refresh if supported; no traditional inbound route storage |
| `soft-reconfiguration inbound` only | Store unmodified received routes, even if Route Refresh is supported |
| `bgp soft-reconfig-backup` only | Does not enable traditional route storage for the neighbor on the tested image |
| Both commands configured | Store unmodified received routes only when Route Refresh is unavailable |

The primary purpose of `bgp soft-reconfig-backup` is to **reduce memory consumption while preserving compatibility with legacy neighbors**.

See [BGP Soft Reconfiguration](soft-reconfiguration.md) for detailed configuration and verification.

---

## Enhanced Route Refresh

**Enhanced Route Refresh**, defined in **RFC 7313**, extends standard BGP Route Refresh.

Standard Route Refresh allows a router to request routing information again, but it does not provide explicit markers identifying the beginning and end of the refreshed route information.

Enhanced Route Refresh introduces these markers.

### Standard vs. Enhanced Route Refresh

| Feature | Standard Route Refresh | Enhanced Route Refresh |
|---|---|---|
| RFC | 2918 | 7313 |
| Capability Code | 2 | 70 |
| Requests route readvertisement | Yes | Uses standard Route Refresh requests |
| Explicit beginning marker | No | Yes |
| Explicit ending marker | No | Yes |
| Supports reconciliation of stale routes | No explicit refresh-boundary mechanism | Yes |

Enhanced Route Refresh is an extension of standard Route Refresh, not a replacement for it.

### Beginning and End Markers

RFC 7313 defines three ROUTE-REFRESH message subtypes.

| Subtype | Name | Purpose |
|---|---|---|
| 0 | Normal Route Refresh | Request routing information |
| 1 | Beginning-of-Route-Refresh (BoRR) | Mark the beginning of refreshed routing information |
| 2 | End-of-Route-Refresh (EoRR) | Mark the end of refreshed routing information |

### Enhanced Route Refresh Operation

When Enhanced Route Refresh has been negotiated:

```text
       R1                              R2
        |                               |
        |------ ROUTE-REFRESH --------->|
        |         Subtype 0             |
        |                               |
        |<--------- BoRR ---------------|
        |         Subtype 1             |
        |                               |
        |<--------- UPDATE -------------|
        |<--------- UPDATE -------------|
        |<--------- UPDATE -------------|
        |                               |
        |<--------- EoRR ---------------|
        |         Subtype 2             |
        |                               |
```

The process is:

1. R1 sends a normal ROUTE-REFRESH request to R2.
2. R2 sends a BoRR message indicating that refreshed advertisements are beginning.
3. R1 marks the relevant previously received routes as stale for the refresh operation.
4. R2 readvertises the applicable routing information.
5. R1 updates the corresponding route entries as the refreshed information arrives.
6. R2 sends an EoRR message when the refresh is complete.
7. R1 removes any routes that remain stale because they were not readvertised.

This makes it possible to identify routes that are no longer present in the neighbor's current advertisements.

### Why Is Enhanced Route Refresh Useful?

Suppose R1 previously learned:

```text
192.0.2.0/24
198.51.100.0/24
203.0.113.0/24
```

During an Enhanced Route Refresh, R2 readvertises only:

```text
192.0.2.0/24
198.51.100.0/24
```

When R1 receives the EoRR marker, it can identify `203.0.113.0/24` as stale because it was not readvertised.

The route can then be removed from the relevant received routing information.

This supports more reliable synchronization of routing information during refresh operations.

**Important:** EoRR is different from the End-of-RIB (EoR) marker used by BGP Graceful Restart.

---

## Disabling Enhanced Route Refresh

On Cisco IOS XE images supporting the hidden `dont-capability-negotiate` command, Enhanced Route Refresh capability advertisement can be selectively suppressed.

For example:

```text
R1(config)# router bgp 65001
R1(config-router)# neighbor 10.0.12.2 dont-capability-negotiate enhanced-refresh
```

This suppresses **Enhanced Route Refresh** capability advertisement to the specified neighbor.

It does not disable standard Route Refresh.

The router can continue to advertise standard Route Refresh capability code 2.

### Disabling All Capability Negotiation

The hidden command can also be used without additional keywords:

```text
R1(config-router)# neighbor 10.0.12.2 dont-capability-negotiate
```

This suppresses BGP capability negotiation with the specified neighbor, including standard Route Refresh.

This can be useful in a lab when testing interoperability with legacy BGP implementations.

However, it also affects other BGP capabilities, including MP-BGP and 4-byte ASN support.

**Do not use this command in production without understanding its consequences.** Availability and behavior vary by Cisco software release.

After changing capability negotiation, the BGP session must be reestablished for the updated OPEN message capabilities to take effect.

---

## Route Refresh and Outbound Route Filtering

BGP Route Refresh is also associated with **Outbound Route Filtering (ORF)**.

ORF allows a BGP router to communicate certain inbound filtering requirements to its neighbor.

The neighbor can then apply the corresponding filters before advertising routing information.

For example:

1. R1 wants to receive only specific prefixes from R2.
2. R1 communicates the filtering requirements using ORF.
3. R2 applies the requested filtering rules to its outbound advertisements toward R1.
4. Route Refresh can be used to request advertisements consistent with the updated filtering requirements.

This reduces the amount of unwanted routing information transmitted between neighbors.

However, **ORF and Route Refresh are separate capabilities**.

Route Refresh does not require ORF, and Route Refresh by itself does not communicate inbound prefix-list rules to a neighbor.

See [BGP Outbound Route Filtering](../routing-policies/orf.md) for ORF operation and configuration.

---

## Route Refresh vs. Other BGP Operations

Route Refresh is specifically designed to obtain routing information again without resetting the session.

It should not be confused with other BGP operations.

| Operation | Effect |
|---|---|
| Route Refresh | Requests routing information again for an address family |
| Traditional inbound soft reconfiguration | Reprocesses locally stored received routes |
| Outbound soft reset | Reevaluates and readvertises routes toward a neighbor |
| Hard reset | Terminates and reestablishes the BGP session |
| BGP Graceful Restart | Helps preserve forwarding during certain control-plane restarts |

### Route Refresh Does Not Reset the Session

During a normal Route Refresh:

- The existing TCP connection remains established.
- The BGP neighbor relationship remains Established.
- The negotiated BGP capabilities remain unchanged.
- Routing information is readvertised for the requested address family.

Route Refresh is therefore particularly useful when applying new inbound filters or attribute manipulation policies.

### Route Refresh Does Not Renegotiate Capabilities

BGP capabilities are normally negotiated when the BGP session is established.

A Route Refresh request does not renegotiate these capabilities.

For example, changing a neighbor's capability advertisement configuration cannot be accomplished through Route Refresh alone.

The session must be reestablished for the new OPEN message capabilities to be exchanged.

---

## Verification and Troubleshooting

### Verify Route Refresh Support

```text
R1# show ip bgp neighbors 10.0.12.2
```

Look for:

```text
Neighbor capabilities:
  Route refresh: advertised and received(new)
```

This confirms that Route Refresh support was exchanged during session establishment.

### Verify Accepted Routes

```text
R1# show ip bgp neighbors 10.0.12.2 routes
```

This displays routes accepted from the neighbor after inbound policy processing.

### Verify the BGP Session

```text
R1# show ip bgp summary
```

The session should remain Established after an inbound soft reset.

### Inspect Route Refresh Messages

In a lab environment, a packet capture of the BGP session can be used to inspect Route Refresh operation.

For example, Wireshark can identify:

- BGP OPEN messages and advertised capabilities.
- ROUTE-REFRESH messages (type 5).
- Requested AFI/SAFI values.
- BoRR and EoRR messages when Enhanced Route Refresh is used.
- BGP UPDATE messages sent in response to a refresh request.

A capture is particularly useful for confirming that an inbound soft reset uses Route Refresh rather than locally stored received routes.

### Troubleshooting Table

| Symptom | Possible Cause | Solution |
|---|---|---|
| Route Refresh capability not received | Neighbor does not advertise support | Check neighbor capabilities |
| Inbound policy changes are not reflected | Existing routes have not been reprocessed | Perform an inbound soft reset |
| Previously rejected routes remain unavailable | Route Refresh not supported, and no stored routes exist | Use traditional soft reconfiguration or reestablish the session |
| `received-routes` command fails | Traditional inbound route storage is not enabled | Use `routes` for accepted paths or enable storage if required |
| `received-routes` reports backup mode | Route Refresh is available, so backup storage is inactive | This is expected behavior |
| Route Refresh does not produce an expected prefix | Remote outbound policy does not advertise it | Check the neighbor's outbound filtering |
| Route Refresh works, but Enhanced Route Refresh markers are absent | Enhanced capability was not negotiated | Verify Enhanced Route Refresh capability support |
| Capabilities do not reflect a recent configuration change | BGP session has not been reestablished | Reestablish the session to renegotiate capabilities |

---

## Key Points

- **BGP Route Refresh** is defined in RFC 2918.
- It allows a router to request routing information again without terminating the BGP session.
- The standard Route Refresh capability uses **Capability Code 2**.
- The ROUTE-REFRESH message is **BGP Message Type 5**.
- Route Refresh support is negotiated through BGP OPEN messages.
- A router can send a ROUTE-REFRESH request only if the neighbor advertised support for receiving it.
- Route Refresh is **AFI/SAFI-specific**.
- The responding router readvertises eligible routing information according to its outbound policies.
- `clear ip bgp <NEIGHBOR> soft in` can initiate Route Refresh when the capability is supported.
- Route Refresh avoids storing additional unmodified copies of received routes.
- Traditional inbound soft reconfiguration can be used when Route Refresh is unavailable.
- `bgp soft-reconfig-backup` makes traditional inbound route storage conditional on Route Refresh availability for neighbors configured with `soft-reconfiguration inbound`.
- **Enhanced Route Refresh**, defined in RFC 7313, uses Capability Code 70 and introduces BoRR and EoRR markers.
- Disabling Enhanced Route Refresh does not disable standard Route Refresh.
- Route Refresh does not renegotiate BGP capabilities or reset the TCP session.

---

## References

- [RFC 2918: Route Refresh Capability for BGP-4](https://www.rfc-editor.org/rfc/rfc2918)
- [RFC 7313: Enhanced Route Refresh Capability for BGP-4](https://www.rfc-editor.org/rfc/rfc7313)
- [Cisco: BGP 4 Soft Configuration (IOS XE 17.x)](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-routing/b-ip-routing/m_irg-soft-config.html)
- [Cisco: BGP Command Reference, clear ip bgp](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_bgp/command/irg-cr-book/bgp-c1.html)
- [Cisco: Configuring a Basic BGP Network (IOS XE 17.x)](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-routing/b-ip-routing/m_irg-basic-net-0.html)
