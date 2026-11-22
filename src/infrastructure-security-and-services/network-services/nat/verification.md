# NAT Translation Tables and Verification

Cisco IOS XE maintains a **NAT translation table** to track how addresses and, when required, transport-layer ports or other protocol identifiers are translated.

The two most important verification commands are:

```text
R1# show ip nat translations
R1# show ip nat statistics
```

- **`show ip nat translations`** displays individual mappings and active flows.
- **`show ip nat statistics`** summarizes translation counts, NAT interfaces, table lookups, and dynamic mappings.
- **`show ip nat translations verbose`** adds aging and other internal information to individual entries.

A NAT configuration can be correct even when the translation table is empty. Most **dynamic** translations are created only when matching traffic passes through the router. **Static** mappings are normally present independently of traffic.

This page focuses on interpreting the NAT table, distinguishing **simple** and **extended** entries, configuring **aging timers**, clearing translations, and troubleshooting translation failures.

---

## Reading the NAT Translation Table

Consider an inside host making a TCP connection to an outside web server:

```text
Inside host                 R1                    Outside server
10.1.1.10                NAT/PAT                 203.0.113.20
TCP 50000                    |                     TCP 443
     |                       |                         |
     +-----------------------+-------------------------+
                             |
                 Inside global address:
                       198.51.100.1
```

Suppose R1 changes the source address and source port as follows:

```text
Before NAT: 10.1.1.10:50000     -> 203.0.113.20:443
After NAT:  198.51.100.1:40001  -> 203.0.113.20:443
```

The corresponding translation appears as:

```text
R1# show ip nat translations
Pro  Inside global         Inside local          Outside local         Outside global
 tcp 198.51.100.1:40001    10.1.1.10:50000       203.0.113.20:443     203.0.113.20:443
```

The four address columns mean:

| Column | Meaning in this example |
|---|---|
| `Inside global` | Address and port representing the inside host to the outside network |
| `Inside local` | Original address and port of the inside host |
| `Outside local` | Address and port used to represent the outside host to the inside network |
| `Outside global` | Outside host's actual address and port |

**Important:** The `Outside local` and `Outside global` columns are identical here because R1 is not translating the outside host's address. They do **not** have to differ for a valid NAT translation to exist.

### What Does `Pro` Mean?

The `Pro` column indicates the protocol, such as `tcp`, `udp`, or `icmp`. A value of `---` indicates an address-only entry with no particular protocol identified.

For TCP and UDP, the number after the colon is a **port**. For ICMP, a similar number commonly represents an **ICMP identifier**, not a TCP/UDP port.

The `show ip nat translations` output is a NAT mapping view, not a general-purpose TCP connection table. It does not necessarily show a complete TCP state machine or prove that the remote application is responding.

---

## Simple and Extended Translation Entries

Cisco distinguishes two important translation entry formats:

| Entry type | What it identifies | Typical appearance |
|---|---|---|
| **Simple** | A translated IP-address binding | `---` in `Pro`, no ports or outside flow endpoints |
| **Extended** | A translation associated with a specific protocol/flow | `tcp`, `udp`, or `icmp`, with ports or protocol identifiers and outside addresses |

**Simple versus extended describes the entry's contents, not whether the NAT rule is static or dynamic.** An extended entry is not necessarily a PAT entry.

### Simple Entries

Consider static inside source NAT:

```text
R1(config)# ip nat inside source static 10.1.1.10 198.51.100.10
```

The translation table can show:

```text
R1# show ip nat translations
Pro  Inside global         Inside local          Outside local         Outside global
---  198.51.100.10         10.1.1.10            ---                   ---
```

This is a **simple entry**:

- It maps one inside address to another.
- It is not tied to a particular TCP/UDP port or remote host.
- The `---` entries do not mean an error has occurred.
- Because the rule is static, the mapping normally remains until its configuration is removed.

Simple entries can also be **dynamic**, as when a host receives an address from a dynamic NAT pool without overloading.

### Extended Entries

For the TCP connection introduced above:

```text
R1# show ip nat translations
Pro  Inside global         Inside local          Outside local         Outside global
 tcp 198.51.100.1:40001    10.1.1.10:50000       203.0.113.20:443     203.0.113.20:443
```

This is an **extended entry** because it contains protocol-specific information and identifies a particular flow.

An extended entry gives the router enough information to distinguish flows that might otherwise share translated addresses. For example, two inside hosts can use the same inside global IP address under PAT as long as their resulting translation keys do not conflict.

### Simple Parent and Extended Child Entries

A particularly important distinction is that **dynamic NAT without `overload` can produce both types of entries**.

For example:

```text
R1(config)# ip nat pool PUBLIC 198.51.100.10 198.51.100.20 netmask 255.255.255.0
R1(config)# access-list 10 permit 10.1.1.0 0.0.0.255
R1(config)# ip nat inside source list 10 pool PUBLIC
```

Suppose `10.1.1.10` is assigned `198.51.100.10` and opens a TCP connection and sends an ICMP Echo Request. Depending on the IOS XE platform and implementation, the table can resemble:

```text
R1# show ip nat translations
Pro  Inside global         Inside local          Outside local         Outside global
 tcp 198.51.100.10:50000   10.1.1.10:50000       203.0.113.20:443     203.0.113.20:443
icmp 198.51.100.10:7       10.1.1.10:7           203.0.113.30:7       203.0.113.30:7
---  198.51.100.10         10.1.1.10            ---                   ---
```

The last row is the **simple parent entry**, sometimes called a **half-entry**. It represents the address assignment:

```text
10.1.1.10 <-> 198.51.100.10
```

The TCP and ICMP rows are **extended child entries**, representing individual flows using that address assignment.

```text
           Simple parent / half-entry
           10.1.1.10 -> 198.51.100.10
                       |
             +---------+---------+
             |                   |
         TCP child            ICMP child
       Port 50000             ID 7
```

The important implications are:

1. An address-only binding can remain after a particular TCP, UDP, or ICMP flow ends.
2. A parent binding cannot normally be removed while it still has active child translations.
3. **Extended child entries do not imply that PAT is being used.** In this example, the TCP port is unchanged and the dynamic NAT pool is configured without `overload`.

**CCIE takeaway:** Distinguish **address allocation** from **individual flow translations**. This matters for aging, address-pool exhaustion, and clearing dynamic entries.

### PAT Does Not Always Have a Visible Simple Parent

With interface-based PAT, such as:

```text
R1(config)# ip nat inside source list 10 interface GigabitEthernet2 overload
```

A router may show extended entries directly, without a separate simple address-binding row for each inside host.

The absence of a `---` parent row does not indicate a problem. The exact display depends on the NAT feature, platform, and software version.

### Static PAT Entries

A static PAT mapping is also protocol-specific:

```text
R1(config)# ip nat inside source static tcp 10.1.1.10 443 198.51.100.10 8443
```

An entry may resemble:

```text
Pro  Inside global         Inside local          Outside local         Outside global
 tcp 198.51.100.10:8443    10.1.1.10:443         ---                   ---
```

Despite the `tcp` protocol and port numbers, this is a **static** mapping, not a dynamically allocated session. The outside columns can remain blank because the static rule is not restricted to a particular outside host.

---

## Static and Dynamic Entry Lifetimes

**Static mappings** are provisioned by configuration. They normally do not age out merely because no traffic uses them.

**Dynamic mappings** are created as traffic is processed, and ordinarily expire after a period of inactivity or after a protocol-specific termination condition.

| Mapping or entry | Typical lifetime |
|---|---|
| Static address mapping | Remains while configured |
| Static port mapping | Remains while configured |
| Dynamic simple address binding | Uses the general NAT translation/half-entry aging timer |
| Dynamic TCP extended entry | Uses TCP or TCP-state-specific aging |
| Dynamic UDP extended entry | Uses UDP aging, with possible DNS-specific handling |
| Dynamic ICMP extended entry | Uses ICMP aging |

Timers are **idle timers**, not necessarily maximum connection durations. Qualifying traffic generally refreshes a dynamic entry's activity state. Some implementations update aging information asynchronously, so `show` output may not change at exactly the moment a packet is forwarded.

---

## NAT Aging Timers and Defaults

Cisco provides different aging timers because a one-to-one address binding and an individual TCP, UDP, or ICMP flow have different lifecycles.

The following are common defaults documented in the Cisco NAT command reference:

| Timer | Command | Common documented default |
|---|---|---|
| Dynamic simple binding / half-entry | `ip nat translation timeout` | **86,400 seconds (24 hours)** |
| TCP | `ip nat translation tcp-timeout` | **86,400 seconds (24 hours)** in the command reference |
| UDP | `ip nat translation udp-timeout` | **300 seconds (5 minutes)** |
| DNS | `ip nat translation dns-timeout` | **60 seconds** |
| ICMP | `ip nat translation icmp-timeout` | **60 seconds** |
| TCP FIN/RST aging | `ip nat translation finrst-timeout` | **60 seconds** |
| TCP SYN-stage aging | `ip nat translation syn-timeout` | **60 seconds** |

**IOS XE version caveat:** Cisco's generic IOS XE 17.x NAT configuration guide lists a **TCP default of 7,440 seconds**, whereas Cisco's IOS IP Addressing Services command reference lists **86,400 seconds**. Do not assume either value universally applies to every IOS XE platform or release. Check the documentation for the exact image, or verify its effective behavior with `show ip nat translations verbose`. For labs, you can explicitly configure the timers to avoid ambiguity.

### General Timeout: `timeout`

The general timeout is particularly important for **dynamic address bindings and half-entries**:

```text
R1(config)# ip nat translation timeout 900
```

This example sets the general timeout to **900 seconds (15 minutes)**.

A Cisco employee's NAT half-entry walkthrough demonstrates that this command controls the parent binding and that a parent may remain even after its timer expires if it still has live child translations. Cisco's IOS XE configuration guide also documents interactions between the general timeout and default TCP/UDP timers, so use the specific `tcp-timeout` and `udp-timeout` commands when you need deterministic protocol timer settings.

**Do not assume** that shortening `ip nat translation timeout` immediately deletes every extended TCP/UDP entry or releases an address that still has active flows.

### TCP Timeout

```text
R1(config)# ip nat translation tcp-timeout 3600
```

This sets the general TCP NAT inactivity timeout to **one hour**. A long-lived TCP connection that periodically transmits qualifying traffic can remain active longer than one hour because its idle timer is refreshed.

The TCP timeout is not identical to the TCP retransmission timeout or a host's application-level timeout.

### TCP FIN/RST Timeout

```text
R1(config)# ip nat translation finrst-timeout 30
```

This sets the shorter aging interval associated with TCP connection termination or reset handling.

NAT attempts to reclaim translations more quickly after connection teardown rather than waiting for the full TCP idle timeout.

### TCP SYN Timeout

```text
R1(config)# ip nat translation syn-timeout 45
```

This configures aging for entries associated with the initial TCP SYN stage, when a connection has not yet been fully established.

This is useful to distinguish from the longer idle timeout of an established TCP flow.

### UDP and DNS Timeouts

```text
R1(config)# ip nat translation udp-timeout 120
R1(config)# ip nat translation dns-timeout 30
```

UDP does not have a TCP-style connection teardown handshake. NAT therefore relies heavily on inactivity timers to reclaim UDP flow translations.

DNS may have a shorter protocol-specific timer than other UDP traffic. A short DNS timeout helps reclaim translations created by brief request-response exchanges.

### ICMP Timeout

```text
R1(config)# ip nat translation icmp-timeout 30
```

ICMP Echo Requests and Replies can use an **ICMP identifier** as part of the translation key. The number displayed after the colon in an ICMP NAT entry is not a TCP or UDP port.

### Per-Port and Route-Map Timers

On platforms that support these command options, Cisco also documents:

```text
R1(config)# ip nat translation port-timeout tcp 443 600
R1(config)# ip nat translation routemap-entry-timeout 300
```

- **`port-timeout tcp 443 600`** configures a TCP port-specific timeout.
- **`routemap-entry-timeout 300`** configures aging for route-map-created half-entries.

Availability and behavior of these options are platform and release dependent; check the router's contextual CLI help before relying on them.

### Restoring Default Timers

To remove a custom timer and return to the device's default behavior:

```text
R1(config)# no ip nat translation timeout
R1(config)# no ip nat translation tcp-timeout
R1(config)# no ip nat translation udp-timeout
R1(config)# no ip nat translation icmp-timeout
```

**Avoid `never` for dynamic entries unless there is a clear requirement.** Non-expiring translations can consume addresses, ports, or table capacity indefinitely.

---

## Inspecting Timers with `verbose`

Use:

```text
R1# show ip nat translations verbose
```

For example, an extended UDP entry might display information similar to:

```text
Pro  Inside global         Inside local          Outside local         Outside global
 udp 198.51.100.1:40001    10.1.1.10:50000       203.0.113.53:53      203.0.113.53:53
     create 00:01:20, use 00:00:10, left 00:04:50
     flags: extended, Map-Id(In): 1
```

This is **illustrative**, since field names, format, and flags vary by IOS XE platform and release.

| Field | What to look for |
|---|---|
| `create` | When, or how long ago, the entry was created |
| `use` | When, or how long ago, matching traffic last used the entry |
| `left` | Remaining aging interval on implementations that display it |
| `timeout` | Configured/effective timeout shown for the entry; some implementations display this value in milliseconds |
| `flags` | Internal entry characteristics, including `extended` or `static` |
| `Map-Id(In)` | NAT mapping identifier associated with the entry |
| `use_count` | Internal reference/use count; **not** a packet counter |
| `entry-id` | Internal identifier that can assist platform-level troubleshooting |

**CCIE takeaway:** A parent half-entry can reach its timeout but remain until its child entries disappear. On some software, verbose output marks such an entry as `timing-out`.

---

## `show ip nat statistics`

The statistics command provides a system-level view:

```text
R1# show ip nat statistics
Total active translations: 3 (1 static, 2 dynamic; 2 extended)
Outside interfaces:
  GigabitEthernet2
Inside interfaces:
  GigabitEthernet1
Hits: 105 Misses: 2
Expired translations: 7
Dynamic mappings:
-- Inside Source
[Id: 1] access-list 10 interface GigabitEthernet2 refcount 2
```

*Illustrative output. Exact counters and additional hardware fields vary.*

| Field | Interpretation |
|---|---|
| `Total active translations` | Current entries, broken down into static and dynamic; the extended count is a further classification, not an additional amount to add to the total |
| `Outside interfaces` | Interfaces with `ip nat outside` |
| `Inside interfaces` | Interfaces with `ip nat inside` |
| `Hits` | Successful NAT table lookups |
| `Misses` | NAT lookups that did not find an existing translation and may require creating a new one |
| `Expired translations` | Accumulated dynamic translations that have aged out |
| `Dynamic mappings` | ACLs, pools, or interfaces used by configured dynamic NAT rules |
| `refcount` | References associated with a dynamic mapping; interpretation can depend on implementation |

### A NAT Miss Is Not Necessarily a Drop

A **miss** can be entirely normal for the first packet of a new flow: no matching translation exists yet, so NAT must try to create one.

A miss is **not automatically a dropped packet**. However, on some hardware platforms Cisco also exposes counters for translation-creation failures and allocation failures. Check those alongside table capacity and forwarding state when sessions are failing.

### NAT Address Pool Statistics

With dynamic address pools, `show ip nat statistics` may show:

- Pool start and end addresses.
- Total and currently allocated addresses.
- Pool allocation failures or misses.
- The NAT ACL or route map associated with the pool.

If new hosts cannot obtain translations, verify whether the pool is exhausted or an existing half-entry is still reserving an address.

---

## Filtering the Translation Table

On supported IOS XE platforms, useful forms include:

```text
R1# show ip nat translations tcp
R1# show ip nat translations udp
R1# show ip nat translations icmp
R1# show ip nat translations inside 198.51.100.10
R1# show ip nat translations vrf BLUE
R1# show ip nat translations verbose
```

The `inside` filter uses the **inside global address** in Cisco's command-reference syntax, not the inside local address. For example, when an inside host `10.1.1.10` is translated to `198.51.100.10`, filter on `198.51.100.10`.

`show ip nat translations vrf BLUE` is especially useful when overlapping address ranges exist in different VRFs. Make sure you are examining the correct VRF rather than assuming the global NAT table tells the whole story.

**Note:** Supported filters vary across platforms. Use `show ip nat translations ?` on the lab router to confirm available options.

---

## Clearing NAT Translation Entries

Dynamic entries can be cleared without removing the NAT configuration:

```text
R1# clear ip nat translation *
```

This clears dynamic NAT translations on the supported platform. It **does not remove the configured static NAT rules**, so static translation entries ordinarily remain visible.

**Caution:** Clearing active translations can disrupt ongoing TCP/UDP sessions and force applications to establish new flows. Use this carefully on production networks.

### Clear a Specific Simple Dynamic Entry

For a dynamic inside mapping:

```text
Inside local:  10.1.1.10
Inside global: 198.51.100.10
```

Use the **inside global address first**, then the **inside local address**:

```text
R1# clear ip nat translation inside 198.51.100.10 10.1.1.10
```

The ordinary clear may fail if that parent has active child translations.

On IOS XE platforms supporting forced clearing:

```text
R1# clear ip nat translation inside 198.51.100.10 10.1.1.10 forced
```

This can remove the half-entry and its children. Use `forced` only when you intentionally want to terminate those translations.

### Clear a Specific Extended Entry

Cisco also documents a form that specifies protocol, addresses, and ports:

```text
clear ip nat translation <protocol> inside <inside-global> <global-port> <inside-local> <local-port> outside <outside-local> <local-port> <outside-global> <global-port>
```

For a TCP entry:

```text
R1# clear ip nat translation tcp inside 198.51.100.1 40001 10.1.1.10 50000 outside 203.0.113.20 443 203.0.113.20 443
```

The command identifies a **specific extended translation**, avoiding the broader impact of clearing the entire table.

### Changing a NAT Rule While Entries Exist

When changing a dynamic NAT pool, overload rule, ACL, or route map, existing translations may continue using their previously selected mapping until they age out or are cleared.

A practical lab sequence is:

1. Record existing translations and statistics.
2. Change the NAT configuration.
3. Clear relevant **dynamic** entries if the old bindings would interfere with testing.
4. Initiate a **new** flow.
5. Verify the new mapping and return traffic.

Do not assume that changing the configuration retroactively rewrites every active translation.

---

## Configuring NAT Translation Limits

Cisco supports NAT table limits on applicable platforms:

```text
R1(config)# ip nat translation max-entries 10000
```

This configures a maximum translation-entry count in supported implementations.

Some platforms also support more targeted limits, such as per-VRF or per-host limits. For example, Cisco documents syntax forms resembling:

```text
R1(config)# ip nat translation max-entries vrf BLUE 2000
R1(config)# ip nat translation max-entries host 10.1.1.10 100
```

Check platform support and the precise syntax in the relevant release. Table limits can protect router resources, but setting them too low may prevent new sessions from obtaining translations.

If a limit is reached, reducing translation timers may eventually recover capacity, but it will not necessarily remove active sessions immediately. Find and correct the cause of excessive translation creation.

---

## Verifying NAT with a Lab

A simple way to observe the translation table is to configure interface-overload PAT:

```text
R1(config)# interface GigabitEthernet1
R1(config-if)# ip address 10.1.1.1 255.255.255.0
R1(config-if)# ip nat inside
R1(config-if)# exit
R1(config)# interface GigabitEthernet2
R1(config-if)# ip address 198.51.100.1 255.255.255.0
R1(config-if)# ip nat outside
R1(config-if)# exit
R1(config)# access-list 10 permit 10.1.1.0 0.0.0.255
R1(config)# ip nat inside source list 10 interface GigabitEthernet2 overload
```

Ensure the inside host has a route to R1 and R1 has a route toward the outside test destination. An outside server should have a suitable return path to R1's translated address.

Then, from an inside host such as `10.1.1.10`:

1. Open a TCP connection to an outside server.
2. Run `show ip nat translations` and identify the translated source address and port.
3. Run `show ip nat translations verbose` and inspect `create`, `use`, and remaining lifetime.
4. Run `show ip nat statistics` and observe active translations and hits/misses.
5. Stop generating traffic. Observe the entry aging and eventually expiring.
6. Configure a shorter UDP/ICMP timeout for a separate test, rather than shortening every timer at once.

**Note:** These addresses are documentation examples. The topology requires a functioning IP path between the inside host, R1, and the outside server.

---

## NAT Troubleshooting Workflow

When NAT is not behaving as expected, work through the following checks.

### 1. Verify the NAT Configuration

```text
R1# show running-config | include ^ip nat|^access-list
R1# show ip interface GigabitEthernet1
R1# show ip interface GigabitEthernet2
```

Confirm that the correct interfaces are marked `ip nat inside` and `ip nat outside`, the source matches the NAT ACL or route map, and the intended pool or interface is referenced.

### 2. Verify Routing and the Forwarding Path

```text
R1# show ip route 203.0.113.20
R1# show ip cef 203.0.113.20 detail
```

For traditional inside-to-outside inside source NAT, the initial destination route determines the outgoing interface before source translation. A correct NAT rule cannot make an otherwise unroutable destination reachable.

For outside source NAT, the inside-to-outside routing lookup can instead concern the **outside local** address. The relevant address depends on the type and direction of translation.

### 3. Inspect the Translation Table

```text
R1# show ip nat translations
R1# show ip nat translations verbose
```

Ask:

- Was any entry created?
- Is its inside global address correct?
- Is the expected TCP/UDP port being translated?
- Are the outside local and global addresses what you expect?
- Is there a stale entry that continues to use an old pool or ISP address?
- Is a parent entry reserving a pool address after a particular child flow expires?

### 4. Check Statistics and Resource Limits

```text
R1# show ip nat statistics
```

Check NAT hits and misses, pool usage, expired translations, and any available allocation/drop counters. If translation creation fails, check both address availability and translation table limits.

### 5. Verify the Return Path and ACLs

Even if outbound packets are translated correctly, replies must return to the translated address and reach the NAT device. Examine routes, ACLs, and policies on both sides.

Remember the traditional NAT order of operations for a common inside-source deployment:

- **Inside-to-outside:** Inbound interface ACL, then routing, then NAT, then outbound interface ACL.
- **Outside-to-inside:** Inbound interface ACL, then NAT, then routing, then outbound interface ACL.

Actual feature interactions can be platform dependent. See [NAT Packet Processing and Order of Operations](order-of-operations.md).

### 6. Debug Carefully

Some IOS XE platforms support:

```text
R1# debug ip nat
R1# undebug all
```

**Warning:** NAT debugging can be very noisy and resource-intensive. Avoid enabling broad debugs on busy production routers. Prefer translation tables, statistics, and supported platform-specific diagnostics first.

---

## Key Points

- **Simple** NAT entries represent address bindings. **Extended** entries identify particular protocol flows.
- **Extended does not mean PAT**. Dynamic NAT without overload can create extended child entries beneath a simple parent binding.
- Static NAT mappings normally remain until the rule is removed. Dynamic entries age out after inactivity or protocol-specific termination.
- The **general translation timeout** is important for dynamic simple bindings and half-entries; protocol-specific timers control TCP, UDP, DNS, and ICMP aging.
- A **parent half-entry may remain after its timer expires** until its child entries have expired.
- `show ip nat translations verbose` exposes creation, usage, and timeout information.
- `show ip nat statistics` helps identify table usage, NAT interfaces, translation lookup activity, and dynamic pool problems.
- `clear ip nat translation *` clears dynamic translations but does not remove configured static mappings.
- A NAT **miss** does not necessarily mean a packet was dropped.

---

## Cisco References

- [IP Addressing Configuration Guide, Cisco IOS XE 17.x: Configuring NAT for IP Address Conservation](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-addressing/b-ip-addressing/m_iadnat-addr-consv-xe.html)
- [IP Addressing Configuration Guide, Cisco IOS XE 17.x: Monitoring and Maintaining NAT](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-addressing/b-ip-addressing/m_iadnat-monmain-xe.html)
- [Cisco IOS IP Addressing Services Command Reference: `ip nat translation`](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/ipaddr/command/ipaddr-cr-book/ipaddr-i4.html)
- [Cisco IOS IP Addressing Services Command Reference: `show ip nat translations`](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/ipaddr/command/ipaddr-cr-book/ipaddr-cr-book_chapter_01000.html)
- [Cisco Community, Cisco Employee: `ip nat translation timeout` and Half-Entries](https://community.cisco.com/t5/networking-knowledge-base/quot-ip-nat-translation-timeout-quot-command/ta-p/3137012)
