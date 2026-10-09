# IGMP Querier

An **IGMP querier** is responsible for periodically sending **IGMP Membership Queries** on an IPv4 subnet.

These Queries prompt multicast receivers to send **IGMP Membership Reports**, allowing multicast routers and IGMP snooping switches to maintain accurate multicast group membership information.

Normally, a multicast router performs the querier function. However, a Layer 2 switch can also act as an **IGMP snooping querier** when no multicast router is present.

## Why an IGMP Querier Is Required

When a host joins a multicast group, it sends an unsolicited IGMP Membership Report.

For example:

```text
                 R1
                  |
                 SW1
                /   \
              PC1   PC2

PC1 joins 239.1.1.1
```

SW1 learns that PC1's port has an interested receiver for `239.1.1.1`, while R1 learns that the subnet has a receiver for the group.

However, what happens if PC1 disconnects without sending an IGMP Leave message?

Neither R1 nor SW1 can assume that the receiver is still present indefinitely.

The **IGMP querier** solves this problem by periodically sending Queries, allowing active receivers to refresh their memberships.

```text
IGMP General Query
        |
        v
Receivers send Membership Reports
        |
        v
Membership state is refreshed
        |
        v
Inactive memberships eventually expire
```

Without periodic Queries, dynamically learned membership information may expire even while receivers remain interested in a group.

For IGMP snooping switches, this can result in multicast traffic being flooded, restricted, or dropped, depending on the platform and configuration.

## Basic IGMP Querier Operation

Consider the following topology:

```text
                 R1
             IGMP Querier
             10.10.10.1
                  |
                  |
                 SW1
              VLAN 10
              /     \
             /       \
           PC1       PC2
```

PC1 has joined multicast group `239.1.1.1`, but PC2 has not.

### Step 1: General Query

R1 periodically sends an **IGMP General Query** to `224.0.0.1`, the All Systems multicast address.

```text
                 R1
                  |
            General Query
             224.0.0.1
                  |
                  v
                 SW1
              /       \
             v         v
           PC1         PC2
```

The Query asks receivers to report their current multicast group memberships.

### Step 2: Membership Report

PC1 responds with a Membership Report for `239.1.1.1`.

PC2 has not joined any multicast groups, so it does not report membership for `239.1.1.1`.

```text
           PC1
            |
      Membership Report
        239.1.1.1
            |
            v
           SW1
            |
            v
            R1
```

### Step 3: Membership State Is Refreshed

R1 uses the Report to maintain its Layer 3 IGMP membership state.

SW1 examines the Report to refresh its Layer 2 IGMP snooping entry.

```text
R1:
239.1.1.1 → Active receivers on VLAN 10

SW1:
239.1.1.1 → PC1's switch port
```

If no receivers continue reporting membership for a group, the corresponding dynamic membership state eventually expires.

This allows multicast forwarding information to remain accurate as receivers join, leave, or disappear from the network.


## IGMP Querier Election

Multiple multicast routers may exist on the same subnet, but normally only **one router acts as the active IGMP querier**.

The selection mechanism differs between IGMP versions.

### IGMPv1

IGMPv1 does **not** have its own querier election mechanism.

Instead, when PIM is used, the elected **PIM Designated Router (DR)** typically performs the IGMP querier function.

PIM DR election prefers the **highest DR priority**, followed by the **highest IP address**.

### IGMPv2 and IGMPv3

IGMPv2 introduced an independent querier election mechanism, which IGMPv3 also uses.

The router with the **lowest IPv4 address** on the subnet becomes the querier.

For example:

```text
            R1               R2
        10.10.10.1       10.10.10.2
             \              /
              \            /
                   SW1
                  /   \
                PC1   PC2
```

Assuming equal PIM DR priorities:

```text
IGMPv1:
R2 → PIM DR and IGMP Querier

IGMPv2/v3:
R1 → IGMP Querier
R2 → Non-Querier
```

The non-querier continues processing IGMP Reports but does not send periodic General Queries while the active querier is present.

If the active querier fails, another router can eventually take over.

> **The IGMP querier and PIM DR are separate roles in IGMPv2/v3**, and the same router is not necessarily elected for both.
> Due to their different election processes (PIM DR = highest IP, IGMP querier = lowest IP), they will default to different routers.

The election process, failure detection, and relevant timers are covered in **IGMP Querier Operation and Configuration**.


## IGMP Snooping Querier

A multicast router is not always present in a Layer 2 network.

For example, consider two hosts communicating using multicast within the same VLAN:

```text
                 SW1
               VLAN 10
                /    \
               /      \
            Source   Receiver
```

SW1 can use IGMP snooping to learn which ports have interested receivers.

However, without a multicast router, there may be no device sending periodic IGMP Queries.

Although the receiver initially sends an unsolicited Membership Report, its dynamically learned snooping entry may eventually expire.

To prevent this, SW1 can be configured as an **IGMP snooping querier**.

```text
                 SW1
           Snooping Querier
                VLAN 10
                /    \
               /      \
            Source   Receiver
```

SW1 now generates IGMP Queries itself.

Receivers respond with Membership Reports, allowing SW1 to maintain its snooping entries.

Importantly, **an IGMP snooping querier does not require Layer 3 multicast routing**.

It generates the Queries necessary to maintain Layer 2 group membership information without routing multicast traffic between subnets.

## IGMP Querier vs IGMP Snooping Querier

Although both can generate IGMP Queries, their roles are different.

| Feature | Router IGMP Querier | IGMP Snooping Querier |
|---|---|---|
| Typical device | Multicast router / Layer 3 switch | Layer 2 switch |
| Sends IGMP Queries | Yes | Yes |
| Maintains Layer 3 IGMP membership state | Yes | No |
| Requires multicast routing | Normally used with multicast routing | No |
| Primary purpose | Learn receiver interest for Layer 3 multicast forwarding | Maintain Layer 2 snooping information |

An IGMP snooping switch does not need to act as a querier if another device is already providing the necessary Queries.

## Section Overview

The following sections examine:

**IGMP Querier Operation and Configuration**
- IGMPv1, IGMPv2, and IGMPv3 querier behavior
- Querier election and failure detection
- Query intervals and other relevant timers
- IOS XE configuration and verification

**IGMP Snooping Querier**
- Layer 2 querier operation
- Configuration without multicast routing
- Query source IP addressing
- Interaction with an existing multicast router
- IOS XE configuration and verification
