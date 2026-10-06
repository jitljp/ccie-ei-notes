# IPv4 Multicast MAC Addresses

When an IPv4 multicast packet is sent over Ethernet, the multicast destination IP address is mapped to an Ethernet multicast destination MAC address.

IPv4 multicast MAC addresses use the prefix:

```text
01:00:5E
```

Which is 24 bits, plus the 25th bit is always set to 0.

This gives a complete range of:

```text
01:00:5E:00:00:00
through
01:00:5E:7F:FF:FF
```

## IPv4-to-MAC Mapping

An IPv4 multicast address contains **28 multicast group bits**:

```text
1110xxxx.xxxxxxxx.xxxxxxxx.xxxxxxxx
    <--------- 28 bits ----------->
```

The Ethernet multicast MAC address contains only **23 bits** for the multicast group:

```text
01:00:5E:0xxxxxxx:xxxxxxxx:xxxxxxxx
         <--------- 23 bits ------->
```

Therefore, the mapping copies the **lowest 23 bits of the IPv4 multicast address** into the lowest 23 bits of the MAC address.

The other bits are fixed:

```text
01:00:5E:0
```

The most significant bit of the fourth MAC octet is always `0`.

## Example

Consider:

```text
239.1.1.1
```

The lower 23 bits are copied into the multicast MAC address:

```text
IPv4:  239.1.1.1
MAC:   01:00:5E:01:01:01
```

Another common example is the OSPF AllSPFRouters address:

```text
224.0.0.5
        ↓
01:00:5E:00:00:05
```

## 32 IPv4 Groups per MAC Address

IPv4 multicast provides **28 group bits**, but Ethernet carries only **23 of them**.

Therefore:

```text
28 - 23 = 5 bits lost

2^5 = 32
```

This means that **32 different IPv4 multicast addresses map to the same Ethernet multicast MAC address**.

For example:

```text
224.1.1.1
225.1.1.1
```

both map to:

```text
01:00:5E:01:01:01
```

This mapping is therefore **not one-to-one**.

A host may receive an Ethernet frame whose destination MAC matches a multicast group it is listening for, even though the destination IP address is for a different multicast group. The IP layer can then discard the unwanted packet.

## Source and Destination MAC Addresses

The multicast mapping applies only to the **destination MAC address**.

For example:

```text
Source IP:       10.1.1.10
Destination IP:  239.1.1.1

Source MAC:      Sender's unicast MAC
Destination MAC: 01:00:5E:01:01:01
```

The sender does not use a multicast source MAC address.

## Key Point

Remember the mapping as:

```text
IPv4 multicast MAC prefix:
01:00:5E

+ a 0 bit
+ lowest 23 bits of the multicast IP address
```

Because five multicast IP bits are discarded, **32 IPv4 multicast groups can map to the same Ethernet MAC address**.