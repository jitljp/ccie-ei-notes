# IGMPv2 and IGMPv3

**Internet Group Management Protocol (IGMP)** is used by IPv4 hosts and multicast routers to manage multicast group membership on a local network.

IGMP allows:

- Hosts to indicate that they want to receive traffic for a multicast group
- Multicast routers to discover which multicast groups have receivers on a local subnet
- Routers to determine when multicast traffic no longer needs to be forwarded onto a subnet

IGMP operates only between hosts and multicast routers on the local network. It does not determine how multicast traffic is routed through the network; protocols such as PIM perform that function.

For IPv6, the equivalent protocol is **Multicast Listener Discovery (MLD)**.

## IGMP Versions

There are three major versions of IGMP:

| Version | Key Features |
|---|---|
| IGMPv1 | Basic group membership reporting |
| IGMPv2 | Leave messages, group-specific queries, querier election improvements |
| IGMPv3 | Source filtering and Source-Specific Multicast (SSM) support |

IGMP versions are backward compatible to allow hosts and routers running different versions to operate on the same network.

## IGMPv1

**IGMPv1** introduced the basic mechanisms used to discover multicast receivers on a subnet.

The multicast router periodically sends **Membership Queries**, and hosts respond with **Membership Reports** for the multicast groups they have joined.

IGMPv1 does not have an explicit leave mechanism. When a host leaves a group, it simply stops sending Membership Reports. The router eventually removes the group after the membership state times out.

IGMPv1 therefore provides the basic foundation for later IGMP versions, but IGMPv2 and IGMPv3 improve how membership changes are detected and communicated.

IGMPv1 is covered in more detail on the next page.

## IGMPv2

**IGMPv2** improves group membership management by adding mechanisms such as:

- **Leave Group** messages
- **Group-Specific Queries**
- Querier election based on IP address
- Faster detection that the last receiver for a group has left

These mechanisms allow multicast routers to remove unnecessary forwarding state more quickly than with IGMPv1.

The following sections examine IGMPv2 message types, queries, reports, report suppression, and the leave process.

## IGMPv3

**IGMPv3** extends IGMP by allowing receivers to specify not only the multicast group they want to receive, but also which multicast **sources** they want or do not want to receive traffic from.

This introduces **source filtering** using:

- **INCLUDE mode**
- **EXCLUDE mode**

IGMPv3 is especially important for **Source-Specific Multicast (SSM)**, where a receiver explicitly requests traffic from a particular source and multicast group `(S,G)`.

IGMPv3 also introduces a more detailed Membership Report format containing one or more **Group Records** that describe the receiver's multicast state.

The following sections examine IGMPv3 queries, reports, source filtering, Group Record types, and state changes.