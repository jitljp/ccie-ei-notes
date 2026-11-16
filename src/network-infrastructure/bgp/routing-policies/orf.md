# 1.5.c (iii) Outbound Route Filtering

**BGP Outbound Route Filtering (ORF)** allows a BGP router to communicate its inbound prefix-filtering requirements to a neighbor, so that the neighbor can apply those filters **before advertising routes**.

Normally, inbound BGP filtering occurs after the receiving router has already received the BGP UPDATE messages.

With ORF, the receiving router can request that the advertising neighbor filter unwanted prefixes at the source.

This reduces:

- Unnecessary BGP UPDATE messages.
- Bandwidth consumed by unwanted route advertisements.
- Control-plane processing on the receiving router.
- Resources used to generate and process unwanted updates.

ORF is particularly useful when a router receives routing information from a provider but does not need the provider's complete routing table.

---

## Normal Inbound Filtering vs. ORF

### Normal Inbound Filtering

Consider two EBGP neighbors:

```text
            AS 65001                   AS 65002

               R1 ----------------------- R2
                     EBGP Session

        Inbound Filter            BGP Advertisements
```

R2 advertises 100,000 prefixes to R1.

R1 has an inbound prefix list that permits only 1,000 prefixes.

With normal inbound filtering:

1. R2 generates and sends UPDATE messages for all 100,000 prefixes.
2. R1 receives and processes the advertisements.
3. R1's inbound prefix list permits 1,000 prefixes.
4. The other 99,000 prefixes are rejected.

Although the unwanted routes are not accepted into R1's BGP table, they still consume bandwidth and processing resources.

### Outbound Route Filtering

With ORF, R1 can communicate its inbound prefix-list requirements to R2.

```text
            AS 65001                   AS 65002

               R1 ----------------------- R2
                     EBGP Session

               |------- ORF Filter ----->|
               |                         |
               |    R2 applies ORF       |
               |    outbound toward R1   |
               |                         |
               |<------ 1,000 routes ----|
```

Now:

1. R1 sends its inbound prefix-filtering requirements to R2.
2. R2 installs the received ORF as an outbound filter toward R1.
3. R2 generates advertisements only for permitted routes.
4. R1 receives the 1,000 permitted prefixes.

**The key difference is where filtering occurs.**

| Characteristic | Normal Inbound Filtering | ORF |
|---|---|---|
| Filter configured on | Receiving router | Receiving router |
| Filter executed on | Receiving router | Advertising router |
| Unwanted prefixes transmitted? | Yes | No, after ORF is applied |
| Neighbor cooperation required? | No | Yes |
| BGP capability negotiation required? | No | Yes |

**Important:** The receiving router continues applying its own inbound prefix list even after sending the corresponding ORF to its neighbor.

---

## ORF Send and Receive Capabilities

ORF requires compatible capabilities to be negotiated between BGP neighbors.

Cisco IOS XE supports three options:

```text
R1(config-router-af)# neighbor <IP> capability orf prefix-list send
R1(config-router-af)# neighbor <IP> capability orf prefix-list receive
R1(config-router-af)# neighbor <IP> capability orf prefix-list both
```

| Keyword | Meaning |
|---|---|
| `send` | Advertise the capability to send ORF filters to the neighbor |
| `receive` | Advertise the capability to receive ORF filters from the neighbor |
| `both` | Advertise both send and receive capabilities |

**Important:** The `send` and `receive` keywords refer to the direction of the **ORF filters**, not the direction of BGP route advertisements.

### ORF Sender

The ORF sender is the router that wants to restrict the routes it receives.

For example, on R1:

```text
R1(config-router-af)# neighbor 10.0.12.2 capability orf prefix-list send
```

R1 advertises its ability to send ORF filters to R2.

### ORF Receiver

The ORF receiver is the router that will apply the received filters to its outbound advertisements.

For example, on R2:

```text
R2(config-router-af)# neighbor 10.0.12.1 capability orf prefix-list receive
```

R2 advertises its ability to receive ORF filters from R1.

### Capability Compatibility

| R1 Capability | R2 Capability | Can R1 Send ORF to R2? |
|---|---|---|
| `send` | `receive` | Yes |
| `both` | `both` | Yes |
| `send` | `both` | Yes |
| `both` | `receive` | Yes |
| `receive` | `send` | No |
| `send` | `send` | No |
| `receive` | `receive` | No |

The last three combinations do not permit **R1 to send ORF to R2**. Other combinations may still permit ORF in the opposite direction.

ORF capabilities are advertised during BGP OPEN message exchange.

By default, ORF capabilities are not advertised unless configured.

If the configuration changes after the BGP session is established, the session may need to be reestablished to negotiate the updated capabilities.

---

## Configuring ORF

ORF is configured under the relevant BGP address family.

The basic configuration requires:

1. An inbound prefix list on the router sending ORF.
2. ORF `send` capability on that router.
3. ORF `receive` capability on its neighbor.
4. An ORF-aware inbound refresh to communicate the filter.

### Example Topology

```text
             AS 65001                   AS 65002

                R1 ----------------------- R2
           10.0.12.1                  10.0.12.2
              E0/0                       E0/0

                          EBGP
```

R2 originates three prefixes:

```text
2.2.2.2/32
22.22.22.22/32
33.33.33.33/32
```

R1 should receive only `2.2.2.2/32`.

Instead of receiving all three advertisements and filtering two locally, R1 will use ORF to request that R2 advertise only `2.2.2.2/32`.

---

## Complete ORF Configuration Example

### Step 1: Configure R1

Configure the interfaces:

```text
R1(config)# interface Ethernet0/0
R1(config-if)# ip address 10.0.12.1 255.255.255.252
R1(config-if)# no shutdown

R1(config)# interface Loopback0
R1(config-if)# ip address 1.1.1.1 255.255.255.255
```

Configure the inbound prefix list:

```text
R1(config)# ip prefix-list FROM-R2 permit 2.2.2.2/32
```

Configure BGP and ORF:

```text
R1(config)# router bgp 65001
R1(config-router)# neighbor 10.0.12.2 remote-as 65002
R1(config-router)# address-family ipv4 unicast
R1(config-router-af)# neighbor 10.0.12.2 activate
R1(config-router-af)# network 1.1.1.1 mask 255.255.255.255
R1(config-router-af)# neighbor 10.0.12.2 capability orf prefix-list send
R1(config-router-af)# neighbor 10.0.12.2 prefix-list FROM-R2 in
```

The two important commands are:

```text
neighbor 10.0.12.2 capability orf prefix-list send
neighbor 10.0.12.2 prefix-list FROM-R2 in
```

The first enables ORF filter transmission.

The second identifies the inbound filtering policy that R1 can communicate to R2.

**Important:** The prefix list must be applied in the `in` direction on R1, even though R2 will use the received ORF for outbound filtering.

### Step 2: Configure R2

Configure the interfaces:

```text
R2(config)# interface Ethernet0/0
R2(config-if)# ip address 10.0.12.2 255.255.255.252
R2(config-if)# no shutdown

R2(config)# interface Loopback0
R2(config-if)# ip address 2.2.2.2 255.255.255.255

R2(config)# interface Loopback1
R2(config-if)# ip address 22.22.22.22 255.255.255.255

R2(config)# interface Loopback2
R2(config-if)# ip address 33.33.33.33 255.255.255.255
```

Configure BGP:

```text
R2(config)# router bgp 65002
R2(config-router)# neighbor 10.0.12.1 remote-as 65001
R2(config-router)# address-family ipv4 unicast
R2(config-router-af)# neighbor 10.0.12.1 activate
R2(config-router-af)# network 2.2.2.2 mask 255.255.255.255
R2(config-router-af)# network 22.22.22.22 mask 255.255.255.255
R2(config-router-af)# network 33.33.33.33 mask 255.255.255.255
R2(config-router-af)# neighbor 10.0.12.1 capability orf prefix-list receive
```

The important command is:

```text
neighbor 10.0.12.1 capability orf prefix-list receive
```

R2 now advertises the capability to receive ORF filters from R1.

R2 does not need a locally configured outbound prefix list for this example.

### Step 3: Activate ORF

After establishing the EBGP session with compatible ORF capabilities, initiate an ORF-aware inbound refresh on R1:

```text
R1# clear ip bgp 10.0.12.2 in prefix-filter
```

This command initiates an inbound refresh and communicates the prefix-based ORF to R2.

R2 installs the received filter and applies it when advertising routes toward R1.

**Important:** Cisco documents this ORF-specific refresh command as the required activation step. Do not confuse it with an ordinary inbound soft clear.

### Step 4: Verify the Result

On R1:

```text
R1# show bgp ipv4 unicast
```

R1 should receive only:

```text
2.2.2.2/32
```

The other prefixes are filtered by R2 before advertisement.

Verify the received ORF on R2:

```text
R2# show ip bgp neighbors 10.0.12.1 received prefix-filter
```

This command displays the ORF prefix-list information received from the specified neighbor.

Verify advertisements:

```text
R2# show bgp ipv4 unicast neighbors 10.0.12.1 advertised-routes
```

After ORF is applied, only `2.2.2.2/32` should be advertised to R1.

R2 should still have all three locally originated prefixes in its BGP table.

---

## ORF and BGP Route Refresh

ORF works with the **BGP Route Refresh mechanism**.

However, ORF and ordinary Route Refresh serve different purposes.

### Ordinary Route Refresh

With ordinary Route Refresh:

1. R1 requests that R2 resend routing information.
2. R2 readvertises eligible routes according to its outbound policies.
3. R1 applies its inbound filtering policies.

For example:

```text
R1# clear ip bgp 10.0.12.2 soft in
```

This normally uses Route Refresh if supported.

However, ordinary Route Refresh does not itself communicate R1's inbound prefix list to R2.

### ORF-Aware Route Refresh

With ORF:

1. R1 communicates its inbound prefix-list requirements to R2.
2. R2 stores the received ORF.
3. R2 applies the ORF to its outbound advertisements toward R1.
4. R2 sends the routes permitted by the combined outbound policies.

The ORF-specific command is:

```text
R1# clear ip bgp 10.0.12.2 in prefix-filter
```

| Command | Purpose |
|---|---|
| `clear ip bgp <IP> soft in` | Reapply inbound policies using Route Refresh or retained routes |
| `clear ip bgp <IP> in prefix-filter` | Initiate ORF-aware inbound refresh and communicate prefix filtering |

ORF entries are carried in BGP ROUTE-REFRESH messages.

The protocol supports refreshing immediately or deferring the refresh until a subsequent Route Refresh request.

See [BGP Route Refresh](../other-features/route-refresh.md).

---

## Updating ORF Filters

Suppose R1 initially permits only:

```text
2.2.2.2/32
```

Later, R1 should also accept:

```text
22.22.22.22/32
```

Update the prefix list:

```text
R1(config)# ip prefix-list FROM-R2 permit 22.22.22.22/32
```

Then initiate another ORF-aware refresh:

```text
R1# clear ip bgp 10.0.12.2 in prefix-filter
```

R1 communicates the updated filter to R2.

R2 can now advertise both permitted prefixes.

The expected result is:

```text
2.2.2.2/32
22.22.22.22/32
```

The prefix `33.33.33.33/32` remains filtered.

ORF allows filtering requirements to change without requiring an administrator to manually configure matching outbound prefix lists on the remote router.

---

## ORF and Existing Outbound Policies

A received ORF does not replace the advertising router's locally configured outbound policies.

Instead, the remote ORF is applied **in addition to** those policies.

For example, suppose:

- R1's ORF permits `192.0.2.0/24` and `198.51.100.0/24`.
- R2's local outbound prefix list permits only `192.0.2.0/24`.

R2 can advertise only `192.0.2.0/24` to R1.

| Prefix | R1's ORF | R2's Outbound Policy | Advertised? |
|---|---|---|---|
| `192.0.2.0/24` | Permit | Permit | Yes |
| `198.51.100.0/24` | Permit | Deny | No |
| `203.0.113.0/24` | Deny | Deny | No |

**Both policies must permit a route for it to be advertised.**

ORF cannot force a neighbor to advertise routes that are denied by its own routing policies or otherwise ineligible for advertisement.

Likewise, ORF does not override normal BGP propagation rules or AS_PATH loop prevention.

---

## ORF Protocol Operation

ORF uses the BGP capability negotiation and Route Refresh mechanisms.

### Capability Negotiation

During session establishment, each router advertises its supported ORF capabilities in its BGP OPEN message.

The capabilities include:

- Address Family Identifier (AFI).
- Subsequent Address Family Identifier (SAFI).
- Supported ORF type.
- Send and/or receive capability.

The negotiated capabilities determine whether a router can send ORF information to its neighbor.

### Exchanging Filters

After capability negotiation, the ORF sender can transmit filter entries using BGP ROUTE-REFRESH messages.

The protocol supports the following operations:

| Operation | Purpose |
|---|---|
| ADD | Add an ORF entry |
| REMOVE | Remove a specific ORF entry |
| REMOVE-ALL | Remove all entries for the specified ORF type |

ORF entries can specify whether matching routes should be permitted or denied.

For address-prefix ORF, entries include prefix matching information such as prefix length and allowed length ranges.

### Applying Filters

The receiving router maintains the ORF information associated with the BGP session.

The ORF is then considered when generating advertisements to that neighbor.

Received ORF entries are associated with the session during which they were exchanged.

They are not permanent configuration statements in the receiving router's running configuration.

---

## ORF vs. Conditional Advertisement

Although both mechanisms control BGP advertisements, their purposes are different.

| Feature | ORF | Conditional Advertisement |
|---|---|---|
| Primary purpose | Prevent unwanted prefixes from being sent | Advertise routes when conditions are met |
| Configuration command | `neighbor ... capability orf prefix-list` | `neighbor ... advertise-map` |
| Filter requirements communicated to neighbor? | Yes | No |
| Requires BGP capability negotiation? | Yes | No additional ORF capability |
| Uses route maps for its main operation? | No, IOS XE prefix-based ORF uses prefix lists | Yes |
| Checks existence of unrelated BGP routes? | No | Yes |
| Reduces unnecessary UPDATE transmission? | Yes | Can, depending on the condition |

**ORF communicates filtering requirements between BGP peers.**

**Conditional advertisement controls whether selected routes are advertised based on specified conditions in the local BGP table.**

See [BGP Conditional Advertisement](conditional-advertisement.md).

---

## ORF Restrictions and Considerations

Cisco IOS XE documents several restrictions for prefix-based ORF.

### Prefix Lists Required

Prefix-based ORF uses **IP prefix lists**.

It does not directly transmit:

- AS-path access lists.
- Distribute lists based on ACLs.
- Route maps.
- Community lists.

A route map can still be used independently for local BGP policies, but its match/set logic is not automatically transmitted as a prefix-based ORF.

### EBGP Support

Cisco's documented IOS XE prefix-based ORF implementation is intended for **external BGP peering sessions**.

Do not assume that the same feature is supported for arbitrary IBGP sessions.

### Address-Family Scope

ORF operates on a per-address-family basis.

The relevant address family and ORF type must be supported by both neighbors.

The Cisco IOS XE configuration covered here focuses on IPv4 unicast. Support for other address families depends on the platform and release.

### Multicast

Cisco documents that this implementation of prefix-based ORF does not support BGP multicast route filtering.

### Negotiation Required

Both peers must advertise compatible ORF capabilities.

Configuring `send` on one router without a matching `receive` capability on the other does not establish operational ORF filtering.

### ORF Does Not Guarantee Prefix Reception

Even when a prefix is permitted by an ORF, it may still be rejected by other BGP filtering policies or fail normal path eligibility checks.

ORF controls advertisement, not guaranteed acceptance or RIB installation.

---

## Verification and Troubleshooting

### Verify ORF Capability Negotiation

```text
R1# show bgp ipv4 unicast neighbors 10.0.12.2
```

Inspect the neighbor capability information for advertised and received ORF support.

Look for the negotiated prefix-based ORF send/receive capabilities.

### Verify the Inbound Prefix List

```text
R1# show ip prefix-list FROM-R2
```

Confirm that the prefix list permits the intended prefixes and prefix lengths.

### Verify Received ORF Entries

On the router receiving ORF filters:

```text
R2# show ip bgp neighbors 10.0.12.1 received prefix-filter
```

This is a particularly important ORF-specific verification command.

It displays the prefix-filter information received from the neighbor, rather than merely displaying the locally configured prefix lists.

### Verify Advertised Routes

```text
R2# show bgp ipv4 unicast neighbors 10.0.12.1 advertised-routes
```

Verify that routes excluded by ORF are no longer advertised.

### Verify Accepted Routes

On the ORF sender:

```text
R1# show bgp ipv4 unicast neighbors 10.0.12.2 routes
```

This displays accepted routes from the neighbor.

### Reapply ORF

```text
R1# clear ip bgp 10.0.12.2 in prefix-filter
```

Use this command after changing the prefix-list requirements when an ORF-aware refresh is needed.

---

## Common Troubleshooting Scenarios

| Symptom | Possible Cause |
|---|---|
| ORF filter is not installed on the neighbor | Incompatible send/receive capabilities |
| ORF capability is absent from neighbor output | Capability was not configured or negotiated |
| ORF configured but has no filtering effect | ORF-aware inbound refresh has not occurred |
| Unexpected prefixes are advertised | Prefix-list entries permit them, or ORF was not applied |
| Expected prefixes are missing | Received ORF or local outbound policy denies them |
| BGP session is Established but ORF is unavailable | ORF capability was not negotiated |
| Changed prefix list does not affect remote advertisements | Updated ORF has not been communicated |
| `received prefix-filter` shows no entries | Neighbor has not sent an applicable ORF |

---

## Key Points

- **Outbound Route Filtering (ORF)** allows a BGP router to request that its neighbor filter outbound advertisements.
- ORF reduces unnecessary BGP UPDATE traffic by filtering routes at the advertising router.
- **`send`** means the router can send ORF filters.
- **`receive`** means the router can receive and apply ORF filters.
- **`both`** enables both ORF directions.
- The ORF sender configures its prefix list **inbound**.
- The ORF receiver applies the received filter **outbound** toward the sender.
- ORF capabilities are negotiated through BGP OPEN messages.
- ORF entries are exchanged using BGP ROUTE-REFRESH messages.
- `clear ip bgp <IP> in prefix-filter` initiates an ORF-aware inbound refresh.
- ORF is applied in addition to the advertising router's existing outbound policies.
- Cisco IOS XE prefix-based ORF uses prefix lists, not route maps or AS-path ACLs.
- ORF does not create routes, modify BGP best-path selection directly, or guarantee route installation.

---

## References

- [Cisco: Connecting to a Service Provider Using External BGP](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-routing/b-ip-routing/m_irg-external-sp-0.html)
- [Cisco: BGP Prefix-Based Outbound Route Filtering](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_bgp/configuration/xe-3e/irg-iproute-bgp-xe-3e-book/irg-oubound-route-filtering.html)
- [Cisco: BGP Command Reference, Commands M through N](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_bgp/command/irg-cr-book/bgp-m1.html)
- [Cisco: BGP Command Reference, Commands show ip through Z](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_bgp/command/irg-cr-book/bgp-s1.html)
- [RFC 5291: Outbound Route Filtering Capability for BGP-4](https://www.rfc-editor.org/rfc/rfc5291.html)
- [RFC 5292: Address-Prefix-Based Outbound Route Filter for BGP-4](https://www.rfc-editor.org/rfc/rfc5292.html)
