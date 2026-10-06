# IGMPv2 Report Suppression

**Report suppression** prevents every host that belongs to the same multicast group from responding to an IGMP Query.

The router only needs to know that **at least one receiver exists** for a group on an interface,
so it knows to forward that group's packets out of that interface.
It does not need a Report from every receiver.

## Report Suppression Process

When hosts receive an applicable Query, each host starts a random report delay timer.

For example, three hosts belong to `239.1.1.1`:

```text
R1 -----------------------> LAN
          General Query

PC1: 239.1.1.1 → 2.1 seconds
PC2: 239.1.1.1 → 5.4 seconds
PC3: 239.1.1.1 → 8.7 seconds
```

PC1's timer expires first, so it sends a Membership Report:

```text
PC1 ----------------------> 239.1.1.1
        Membership Report
```

Because the Report is sent to `239.1.1.1`, PC2 and PC3 also receive it.

If another host's Report is received while a host has a report timer running for that group, the host:

1. Stops the report timer.
2. Does not send its own Report.
3. Clears its **last-reporter flag** for the group.

```text
PC1 timer expires
        ↓
PC1 sends Report
        ↓
PC2 hears Report → cancels timer
PC3 hears Report → cancels timer
        ↓
Only one Report is sent
```

This is **report suppression**.

## Suppression Is Per Group

Report suppression operates independently for each multicast group.

For example, if hosts belong to:

```text
239.1.1.1
239.2.2.2
```

a Report for `239.1.1.1` suppresses pending Reports only for `239.1.1.1`.

It does **not** suppress a pending Report for `239.2.2.2`.

Therefore, a General Query can still result in multiple Reports:

```text
General Query
     ↓
One Report for 239.1.1.1
One Report for 239.2.2.2
One Report for 239.3.3.3
...
```

The goal is **one Report per active group**, not one Report per Query. Additional reports would be redundant and are unnecessary.

## IGMPv1 and IGMPv2 Compatibility

An IGMPv2 host allows its pending Report to be suppressed by either:

```text
IGMPv1 Membership Report → Type 0x12
IGMPv2 Membership Report → Type 0x16
```

This allows IGMPv1 and IGMPv2 hosts to coexist on the same subnet.

## Unsolicited Reports

Report suppression can also affect the retransmission of an **unsolicited Membership Report**.

When a host first joins a group, it immediately sends a Report and starts a timer for a possible repeated Report.

If it hears another host report the same group before that timer expires, the additional Report can be suppressed.

```text
PC1 joins 239.1.1.1
      ↓
PC1 sends unsolicited Report
      ↓
PC1 starts retransmission timer
      ↓
PC2 sends Report for 239.1.1.1
      ↓
PC1 cancels its pending Report
```

## Effect on Router State

Because of report suppression, the router does not necessarily know how many receivers exist.

For example:

```text
PC1 ──┐
PC2 ──┼── Ethernet0/0 ── R1
PC3 ──┘
```

All three hosts may receive `239.1.1.1`, while R1 receives a Report from only PC1.

`show ip igmp groups` might therefore show:

```text
Group Address    Interface      Last Reporter
239.1.1.1        Ethernet0/0    10.1.1.10
```

The **Last Reporter** is simply the host whose Report the router most recently received.

It does **not** mean that host is the only receiver.

This is why traditional IGMPv2 router state is essentially:

```text
Group G has at least one receiver on this interface
```

rather than:

```text
Group G has receivers PC1, PC2, and PC3
```

## IGMPv3 Difference

Traditional host report suppression is used by **IGMPv1 and IGMPv2**.

IGMPv3 removes this behavior and uses a different reporting model in which hosts report their own group and source-filter state.

IGMPv3 reporting is covered later.
