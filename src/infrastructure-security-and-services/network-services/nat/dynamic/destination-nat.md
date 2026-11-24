# Inside Destination NAT and Load Balancing

**Inside destination NAT** allows outside clients to access multiple inside servers using a single **virtual IP address (VIP)**. The NAT router selects a real server for each new TCP connection and translates the destination IP address to that server's address.

Cisco calls this feature **TCP load distribution**. It uses a NAT pool of `type rotary` to distribute new connections in round-robin order.

> Inside destination NAT supports **only TCP**. It does NOT support UDP, ICMP, etc.

Unlike dynamic inside source NAT, which allocates inside global addresses to inside hosts, inside destination NAT selects **inside local addresses** from a pool when outside clients initiate connections.

---

## Configuration Syntax

```text
Router(config)# ip nat pool <name> <start-ip> <end-ip> prefix-length <length> type rotary
Router(config)# access-list <number> permit host <virtual-ip>
Router(config)# ip nat inside destination list <number> pool <name>
```

| Component | Purpose |
|---|---|
| Standard ACL | Identifies the inside global address (VIP) to which outside clients connect |
| Rotary pool | Contains the inside local addresses of the real servers |
| `ip nat inside destination` | Associates the virtual address selection ACL with the real server pool |

**Although a standard ACL normally matches source addresses, in this NAT configuration it identifies the destination address of incoming TCP traffic.**

The `overload` keyword is not needed. The virtual IP address is shared by multiple real servers, but this is not ordinary dynamic PAT: the destination TCP port is preserved.

---

## Comparison with Other Dynamic NAT Types

| NAT type | ACL identifies | Pool contains | Initial traffic creating the mapping |
|---|---|---|---|
| Inside source | Inside local addresses | Inside global addresses | Inside to outside |
| Outside source | Outside global addresses | Outside local addresses | Outside to inside |
| Inside destination | Inside global addresses | Inside local addresses | Outside to inside (TCP connections) |

Inside source NAT and inside destination NAT both translate inside hosts' addresses. The distinction is how the dynamic mapping is established and which address is allocated from a pool.

---

## Lab Topology

We'll reuse the topology from the previous NAT lessons. LH5 and LH6 now act as TCP servers, while RH7 and RH8 act as outside clients.

```text
       INSIDE NETWORK                         OUTSIDE NETWORK

  Local_Host_5 (LH5)
     10.1.1.5/24
          |
          |                   Local_Router                    ISP1
          +---- LAN ---------- E0/0    E0/1 ---------------- E0/1
          |                10.1.1.1   7.7.7.2              7.7.7.1
          |                NAT inside NAT outside              |
  Local_Host_6 (LH6)                                           |
     10.1.1.6/24                                   E0/0: 3.3.3.1/24
                                                               |
                                                          Outside LAN
                                                               |
                                                        +------+------+
                                                        |             |
                                                Remote_Host_7  Remote_Host_8
                                                 3.3.3.7/24    3.3.3.8/24
```

For this exercise, use a separate routed virtual address, `203.0.113.50`, rather than an address on the outside transit subnet. This avoids dependence on NAT-generated ARP responses for the VIP.

| Address | Role |
|---|---|
| `203.0.113.50` | Virtual IP address (inside global) |
| `10.1.1.5` | First real server (inside local) |
| `10.1.1.6` | Second real server (inside local) |
| `3.3.3.7` | Outside client RH7 |
| `3.3.3.8` | Outside client RH8 |

The VIP is not configured on either server. The existing public lab ranges are for isolated lab use only.

---

## Configure Inside Destination NAT

Remove conflicting NAT rules from earlier exercises and clear their dynamic translations before starting.

### Step 1: Configure Interface Roles and Routing

On Local_Router:

```text
interface Ethernet0/0
 ip address 10.1.1.1 255.255.255.0
 ip nat inside
 no shutdown
!
interface Ethernet0/1
 ip address 7.7.7.2 255.255.255.0
 ip nat outside
 no shutdown
!
ip route 0.0.0.0 0.0.0.0 7.7.7.1
```

On ISP1:

```text
interface Ethernet0/1
 ip address 7.7.7.1 255.255.255.0
 no shutdown
!
interface Ethernet0/0
 ip address 3.3.3.1 255.255.255.0
 no shutdown
!
ip route 203.0.113.50 255.255.255.255 7.7.7.2
```

LH5 and LH6 use `10.1.1.1` as their default gateway. RH7 and RH8 use `3.3.3.1`.

ISP1 does not need a route to `10.1.1.0/24` for this translated service. It needs a route to the VIP. The NAT router has connected routes to the real servers and a route back to the clients.

### Step 2: Configure a Rotary Pool

```text
Local_Router(config)# ip nat pool SERVERS 10.1.1.5 10.1.1.6 prefix-length 24 type rotary
```

The pool contains the actual addresses of the two inside servers, not their outside-facing representations.

`type rotary` enables cyclic selection of pool addresses for new connections. Unlike an ordinary address-only source NAT pool, the same real server address can be selected for many simultaneous connections.

### Step 3: Identify the Virtual IP Address

```text
Local_Router(config)# access-list 2 permit host 203.0.113.50
```

The ACL identifies the **inside global address** to match as the destination of outside-to-inside traffic. It does not select outside clients or identify the inside server subnet.

### Step 4: Enable Dynamic Inside Destination Translation

```text
Local_Router(config)# ip nat inside destination list 2 pool SERVERS
```

The association is now configured. A connection to the VIP can create a dynamic translation to either server.

### Step 5: Prepare the Real Servers

Both servers must listen on the same TCP port. For an isolated IOS lab, enable Telnet on LH5 and LH6 with distinct hostnames and a lab-only password:

```text
line vty 0 4
 password LABONLY
 login
 transport input telnet
```

Telnet is used here solely for convenient testing in an isolated lab. Alternatively, use HTTP servers on both hosts with distinguishable response content. For Linux hosts, for example, run `python3 -m http.server 8080 --bind 0.0.0.0` in separate directories containing different index pages.

---

## TCP Connection Allocation

Suppose RH7 initiates a connection to `203.0.113.50` on TCP port 23 using source port 50001. Assume the rotary allocator selects LH5.

| Point in path | Source | Destination |
|---|---|---|
| Outside request | `3.3.3.7:50001` | `203.0.113.50:23` |
| Inside request | `3.3.3.7:50001` | `10.1.1.5:23` |
| Inside reply | `10.1.1.5:23` | `3.3.3.7:50001` |
| Outside reply | `203.0.113.50:23` | `3.3.3.7:50001` |

The destination is translated before the outside-to-inside routing lookup, so Local_Router routes the packet to the selected server. On the reply, the source is translated to the VIP, so the outside client sees the expected TCP endpoint.

Neither TCP port changes. The outside client's address also remains unchanged.

For subsequent new connections, the allocator cycles through the real servers. For example:

| New TCP connection | Outside client | Selected inside server |
|---|---|---|
| 1 | RH7 | LH5 |
| 2 | RH7 | LH6 |
| 3 | RH8 | LH5 |
| 4 | RH8 | LH6 |

The starting position in this example is illustrative. Existing connections continue using their original server. Retransmissions and subsequent packets matching an existing translation do not select a different server.

**Load distribution is per TCP connection, not per packet, per client, or per application request.** A single persistent HTTP connection can carry many requests without another server selection.

---

## Verification

### Verify the Configuration

```text
Local_Router# show running-config | include ip nat
Local_Router# show access-lists 2
Local_Router# show ip nat statistics
```

Confirm the inside/outside roles, ACL, dynamic destination mapping, and rotary pool. The pool should contain two addresses.

Before any connections, `show ip nat translations` can be empty. A rotary mapping is not a permanent static mapping for the VIP.

### Generate Multiple Connections

From RH7, open a Telnet connection:

```text
Remote_Host_7# telnet 203.0.113.50
```

Log in and check the hostname to identify the selected server. Open another independent connection (from RH8, or by suspending the first session on RH7 and opening a second one).

For HTTP testing, make separate TCP connections, rather than issuing multiple requests over one persistent connection:

```bash
curl http://203.0.113.50:8080/
curl http://203.0.113.50:8080/
```

### Inspect the Translation Table

Representative output after two simultaneous TCP connections from RH7:

```text
Local_Router# show ip nat translations
Pro Inside global         Inside local       Outside local      Outside global
 tcp 203.0.113.50:23       10.1.1.5:23        3.3.3.7:50001      3.3.3.7:50001
 tcp 203.0.113.50:23       10.1.1.6:23        3.3.3.7:50002      3.3.3.7:50002
```

Notice that the inside global address and port are identical in both rows. The different outside client endpoints distinguish the connections. These are **extended dynamic translations**, even though no PAT overload is configured and no port is changed.

A third connection can use `10.1.1.5` again while the first remains active. The pool size is not a limit on the number of TCP connections.

### Verify Routing and Debug Translation

```text
ISP1# show ip route 203.0.113.50
Local_Router# show ip route 10.1.1.5
Local_Router# show ip route 10.1.1.6
Local_Router# show ip route 3.3.3.7
Local_Router# show ip nat translations verbose
```

In an isolated lab, use `debug ip nat` to inspect translation activity and `undebug all` when finished. Packet captures on both sides can verify destination translation on requests and source translation on replies.

---

## Routing and ARP for the Virtual Address

The outside network must deliver packets for the VIP to the NAT router. The rotary configuration alone is not a substitute for routing.

In this exercise, the ISP's `/32` route points to the actual outside interface address (`7.7.7.2`). ISP1 resolves that next hop through ARP, not the VIP itself.

Unlike static inside source NAT, **dynamic inside destination NAT does not create an IP alias for the inside global address (VIP), even if it is on-link**.

If you choose `7.7.7.50` as the VIP, ISP1 considers it directly connected and sends ARP Requests for it. However, Local_Router does not answer these ARP Requests on behalf of the VIP by virtue of the NAT configuration.

An explicit host route on ISP1 solves this problem:

```text
ISP1(config)# ip route 7.7.7.50 255.255.255.255 7.7.7.2
```

The `/32` static route takes precedence over the connected `/24` route. ISP1 resolves `7.7.7.2` through ARP instead of attempting to resolve the VIP directly.

Change ACL 2 to match the new VIP if performing this alternative test. Clear old translations and remove the old ACL entry.

**Local_Router itself does not need a route to the VIP.** Outside-to-inside NAT occurs before routing, so the routing lookup uses the translated destination address (the inside local address of the selected server).

Both directions of the connection must traverse the NAT router. If a server replies through another gateway that lacks the translation, the outside client can receive traffic from an unexpected source or no reply at all.

---

## Limitations

### TCP Only

The feature is designed for TCP connection distribution. Cisco's IOS XE 17.x configuration guide describes non-TCP traffic as untranslated by this rule. A separate Cisco support article describes non-TCP traffic being directed to the first pool member instead. Do not rely on either behavior without testing your image, and do not use ping to verify round-robin TCP distribution.

### No Port Redirection

A connection to TCP port 8080 on the VIP is forwarded to TCP port 8080 on the selected real server. Unlike static PAT, this mechanism does not let you configure different external and internal ports.

### No Server Health Checks or Load Awareness

Rotary selection does not measure server load or test whether a server's service is responding. If LH6 fails, new connections may still be assigned to LH6. Round-robin connection counts also do not guarantee equal bandwidth or CPU usage.

To demonstrate this, stop the TCP service on LH6 while leaving it in the pool. Generate several new connections and observe which succeed. Restore the service afterward.

This feature is basic Layer 4 connection distribution, not a replacement for a dedicated application load balancer with health checks, persistence policies, or application-aware routing.

### Translation Lifetime

The translations are dynamic and are subject to NAT aging and TCP session-related timeouts. Closing a connection does not necessarily remove the displayed translation immediately.

```text
Local_Router# clear ip nat translation *
```

Clearing dynamic translations can interrupt active connections and affects other dynamic NAT rules as well. Use this only in the lab or during an appropriate maintenance window. See [NAT Translation Tables and Verification](../verification.md) for timeout and clearing details.

---

## Troubleshooting

| Symptom | Possible cause |
|---|---|
| No translations after pinging the VIP | Ping does not test TCP rotary allocation |
| No translations after a TCP connection attempt | Incorrect ACL, missing VIP route, wrong NAT interface roles, conflicting rules, or unsupported feature |
| Translation exists but TCP connection fails | Backend service unavailable, ACL filtering, incorrect server gateway, or return-path issue |
| All requests reach one server | Requests reuse a single TCP connection, existing translation state, or incorrect pool range |
| Every other connection fails | One server is unavailable or not listening on the destination port |
| ISP cannot deliver packets to an on-link VIP | Missing ARP resolution; verify or use a routed VIP |
| Wrong TCP port on backend | Rotary destination NAT preserves the destination port |

---

## Key Takeaways

- `ip nat inside destination list <acl> pool <pool>` enables dynamic inside destination NAT.
- The ACL identifies **inside global addresses**; the pool contains **inside local addresses**.
- The pool must use `type rotary` for TCP round-robin distribution.
- A new outside-initiated TCP connection selects a real server; existing connections retain their mapping.
- Outside-to-inside traffic has its destination translated; return traffic has its source translated to the VIP.
- TCP ports and the outside client's address are unchanged by this rule.
- Several active connections can use the same real server.
- The NAT router must receive traffic for the VIP and see the return path.
- The feature does not perform server health checks, port redirection, or load-aware selection.

---

## References

- [Cisco IOS XE 17.x: Configuring NAT for IP Address Conservation](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-addressing/b-ip-addressing/m_iadnat-addr-consv-xe.html)
- [Cisco IOS IP Addressing Services Command Reference: ip nat inside destination](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/ipaddr/command/ipaddr-cr-book/ipaddr-i3.html)
- [Cisco: Configure Server Load Balancing Using Dynamic NAT](https://www.cisco.com/c/en/us/support/docs/ip/network-address-translation-nat/200608-Server-Load-Balancing-Using-Dynamic-NAT.html)
