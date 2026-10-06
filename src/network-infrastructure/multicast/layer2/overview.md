# Layer 2 Multicast

At Layer 2, switches must determine **which ports should receive multicast traffic**.

Unlike unicast traffic, multicast destination MAC addresses do not identify a single receiving device. Without additional mechanisms, a switch will simply **flood multicast traffic** throughout the VLAN.

Layer 2 multicast optimization relies primarily on the following technologies:

- **IGMP (Internet Group Management Protocol)**: IPv4 hosts use IGMP to tell multicast routers which IPv4 multicast groups they want to receive.
- **MLD (Multicast Listener Discovery)**: The IPv6 equivalent of IGMP, used by IPv6 hosts to signal multicast group membership.
- **IGMP Snooping**: A switch examines IGMP messages exchanged between hosts and multicast routers and builds Layer 2 multicast forwarding state.
- **MLD Snooping**: The IPv6 equivalent of IGMP snooping.
- **IGMP/MLD Snooping Querier**: Allows a switch to generate membership queries when no multicast router is available to perform that function.

Although **IGMP and MLD are Layer 3 protocols**, switches can inspect their messages to determine where multicast receivers are located.

> The idea is similar to DHCP Snooping, which enables a switch to examine and make decisions based on information above Layer 2.

For example, if only Host A has joined multicast group **239.1.1.1**, IGMP snooping allows the switch to forward traffic for that group only toward Host A rather than flooding it to every port in the VLAN.

```text
                     Multicast Router
                            |
                           SW1
                       /    |    \
                    Host A Host B Host C
                      |
                 Joined 239.1.1.1
```

Without IGMP snooping:

```text
239.1.1.1 → Host A, Host B, Host C
Host A receives and processes the message.
Hosts B and C receive and discard the message.
```

With IGMP snooping:

```text
239.1.1.1 → Host A
```

The following sections examine **IGMP and MLD operation**, how switches learn multicast receiver and router ports through **snooping**, and the Layer 2 forwarding behavior used to deliver multicast traffic efficiently.