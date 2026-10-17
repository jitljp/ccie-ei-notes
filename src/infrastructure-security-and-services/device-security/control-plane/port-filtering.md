# CPPr Port Filtering and Queue Thresholding

**Control Plane Protection (CPPr)** provides two additional protection mechanisms beyond ordinary Control Plane Policing (CoPP):

- **Port Filtering:** Drops packets directed toward selected or closed TCP/UDP ports before they reach the corresponding control-plane processes.
- **Queue Thresholding:** Limits the number of unprocessed packets that particular protocols can have in the control-plane IP input queue.

Both features use specialized **Modular QoS CLI (MQC)** class maps and policy maps.

Unlike ordinary CoPP policies, these features can only be applied as **input policies on the control-plane host subinterface** in Cisco's documented CPPr implementation.

| Feature | Port Filtering | Queue Thresholding |
|---|---|---|
| Primary purpose | Drop unwanted TCP/UDP traffic | Prevent a protocol from overwhelming the input queue |
| Classification | TCP/UDP ports, including closed ports | Supported application protocols |
| Policy action | `drop` | `queue-limit` |
| Uses CIR or Bc | No | No |
| Attachment point | `control-plane host` | `control-plane host` |
| Direction | Input only | Input only |

These are specialized CPPr features, not ordinary policing mechanisms.

---

## CPPr Port Filtering

**Port filtering** protects the control plane by dropping unwanted packets destined for TCP or UDP ports on the router.

It is particularly useful for traffic directed toward **closed or nonlistened ports**.

For example, an attacker might send large numbers of TCP SYN packets to ports on which the router has no active services.

Without port filtering, these packets may require control-plane processing, even though no application is listening on the destination ports.

Port filtering can discard them earlier.

```text
Incoming TCP/UDP Packet
          |
          v
   CPPr Port Filter
          |
          +-- Matches Drop Class --> Drop
          |
          +-- Does Not Match -----> Continue Processing
```

Port filtering does not use token buckets or policing rates.

**A matching packet is dropped regardless of the traffic rate.**

### Open-Port Database

CPPr maintains a dynamic database of TCP and UDP ports registered by applications running on the router.

The database can include:

- TCP ports on which services are listening.
- UDP ports registered by active services.
- Dynamically opened application ports.
- Ephemeral ports associated with certain application sessions.

For example, enabling SSH may cause TCP port 22 to be registered as an open port.

Port filtering can use this database to determine whether a packet is directed toward a closed port.

However, **not every legitimate IOS service is necessarily registered in the port-filter database**.

Some services may need to be explicitly excluded from closed-port filtering.

### IOS XE Configuration: Drop Closed Ports

The basic configuration consists of three steps.

**Step 1: Create a Port-Filter Class Map**

```text
R1(config)# class-map type port-filter match-all CPPR-CLOSED-PORTS
R1(config-cmap)# match closed-ports
```

The `match closed-ports` command matches traffic directed toward TCP or UDP ports that are not registered as open in the port-filter database.

**Step 2: Create a Port-Filter Policy Map**

```text
R1(config)# policy-map type port-filter CPPR-PORT-FILTER
R1(config-pmap)# class CPPR-CLOSED-PORTS
R1(config-pmap-c)# drop
```

The `drop` command discards packets matching the class.

**Step 3: Apply the Policy to the Host Subinterface**

```text
R1(config)# control-plane host
R1(config-cp-host)# service-policy type port-filter input CPPR-PORT-FILTER
```

The `type port-filter` keyword distinguishes this policy from an ordinary CoPP policing policy.

The resulting behavior is:

| Incoming Packet | Port Status | Action |
|---|---|---|
| TCP SYN to port 22 | Open, SSH listening | Not matched by `closed-ports` |
| TCP SYN to port 23 | Closed, Telnet disabled | Drop |
| UDP packet to port 9999 | Closed | Drop |
| UDP packet to port 161 | Open, SNMP registered | Not matched by `closed-ports` |

The actual open-port state depends on the services running on the router and their registration with CPPr.

---

## Excluding Ports from Closed-Port Filtering

Some legitimate services may not appear in the CPPr open-port database.

For these services, `match closed-ports` could incorrectly classify their traffic as unwanted.

To exclude a port, use:

```text
match not port {tcp | udp} <port-number>
```

For example, exclude NTP on UDP port 123:

```text
R1(config)# class-map type port-filter match-all CPPR-CLOSED-PORTS
R1(config-cmap)# match closed-ports
R1(config-cmap)# match not port udp 123
```

The class now matches packets satisfying **both** conditions:

1. The destination TCP/UDP port is classified as closed.
2. The packet is not directed toward UDP port 123.

| Packet | Closed Port? | UDP Port 123? | Class Match? |
|---|---|---|---|
| TCP destination port 23 (closed) | Yes | No | Yes |
| UDP destination port 9999 (closed) | Yes | No | Yes |
| UDP destination port 123 (closed) | Yes | Yes | No |
| TCP destination port 22 (open) | No | No | No |

**The `match-all` keyword is important.**

Using `match-any` would change the logic to OR, potentially matching and dropping traffic that should be permitted.

### Example: Excluding DHCP

If the router obtains its IPv4 address using DHCP, DHCP traffic might need to be excluded from the closed-port filter.

For example:

```text
R1(config)# class-map type port-filter match-all CPPR-CLOSED-PORTS
R1(config-cmap)# match closed-ports
R1(config-cmap)# match not port udp 67 68
```

This excludes DHCP-related UDP ports 67 and 68 from the drop class on implementations supporting the multiple-port syntax.

The exact exclusion requirements depend on which services are registered as open and how the platform processes their traffic.

> A `match not port` statement excludes traffic from the **drop class**. It does not explicitly permit traffic through all other security controls or policies.

---

## Explicit Port Filtering

Port filtering can also match a specified port directly, without checking whether the port is open.

For example, to drop traffic directed toward TCP port 23 (Telnet):

```text
R1(config)# class-map type port-filter match-all CPPR-TELNET
R1(config-cmap)# match port tcp 23
```

Create the policy:

```text
R1(config)# policy-map type port-filter CPPR-BLOCK-TELNET
R1(config-pmap)# class CPPR-TELNET
R1(config-pmap-c)# drop
```

Apply it:

```text
R1(config)# control-plane host
R1(config-cp-host)# service-policy type port-filter input CPPR-BLOCK-TELNET
```

This drops matching traffic even if the router has an active Telnet service.

Compare the two match criteria:

| Match Criterion | Behavior |
|---|---|
| `match closed-ports` | Matches ports not registered as open |
| `match port tcp 23` | Matches TCP destination port 23 regardless of whether it is open |
| `match not port udp 123` | Excludes UDP port 123 from the class's match criteria |

Unlike an interface ACL, a port-filter policy has no general-purpose `permit` action.

The documented port-filter action is **`drop`**.

---

## Port-Filter Restrictions

The following restrictions apply to the documented CPPr implementation:

- Port filtering is supported only on the **host subinterface**.
- Only **input** port-filter policies are supported.
- Classification is limited to supported TCP/UDP port criteria.
- The port-filter policy action is `drop`, not `police`.
- Ordinary ACL-based MQC class maps cannot substitute directly for `type port-filter` class maps.
- Some legitimate services may not be automatically recognized as open.
- Traditional CPPr port filtering is documented for the **IPv4 input path**.

Port filtering supplements, rather than replaces, other management-plane security controls.

For example, VTY access restrictions and AAA should still be used to secure SSH administration.

---

## CPPr Queue Thresholding

**Queue thresholding** limits the number of unprocessed packets associated with a protocol that can be admitted to the control-plane IP input queue.

Its purpose is to prevent one protocol from occupying too much of the queue and interfering with other control-plane traffic.

Consider a router receiving large numbers of SNMP requests.

Without a per-protocol queue limit, SNMP packets might consume substantial queue resources.

Queue thresholding allows a separate maximum number of queued packets for SNMP.

```text
          Incoming Host Traffic
                   |
                   v
             Protocol Class
                   |
                   v
              Queue Limit
                   |
         +---------+---------+
         |                   |
    Below Limit          Limit Reached
         |                   |
         v                   v
   Admit to Queue          Drop
         |
         v
     CPU Processing
```

### Queue Thresholding vs. Policing

Queue thresholding differs fundamentally from rate-based policing.

| Feature | Policing | Queue Thresholding |
|---|---|---|
| Main parameter | CIR, Bc, and optionally PIR/Be | Queue limit |
| Measures | Traffic rate and burst allowance | Number of unprocessed packets |
| Example command | `police cir 1000000 bc 31250` | `queue-limit 50` |
| Primary protection | Excessive traffic rate | Excessive queue occupancy |
| Uses token buckets | Yes | No |

For example:

```text
queue-limit 50
```

Means that the matched protocol is limited to **50 unprocessed packets in the relevant control-plane input queue**.

It does **not** mean:

- 50 packets per second.
- A burst of 50 packets per second.
- A maximum of 50 packets per TCP connection.

The limit concerns queue occupancy, not the rate at which packets arrive.

---

## Queue-Threshold Classification

Queue thresholding uses special MQC class maps:

```text
class-map type queue-threshold <class-name>
```

The documented implementation supports a restricted set of protocol matches.

| Protocol | Match Command |
|---|---|
| BGP | `match protocol bgp` |
| DNS | `match protocol dns` |
| FTP | `match protocol ftp` |
| HTTP | `match protocol http` |
| IGMP | `match protocol igmp` |
| SNMP | `match protocol snmp` |
| SSH | `match protocol ssh` |
| Syslog | `match protocol syslog` |
| Telnet | `match protocol telnet` |
| TFTP | `match protocol tftp` |

The command `match host-protocols` is also documented for matching other open TCP/UDP host protocols.

The available protocol keywords can vary by platform.

Unlike ordinary MQC class maps, queue-threshold class maps do not support arbitrary ACL-based matching.

---

## IOS XE Configuration: Queue Thresholding

Suppose R1 should enforce the following queue limits:

| Protocol | Queue Limit |
|---|---|
| SSH | 50 packets |
| SNMP | 25 packets |
| Other supported host protocols | 150 packets |

These limits are illustrative and should be validated against actual traffic and platform behavior.

### Step 1: Create Queue-Threshold Class Maps

```text
R1(config)# class-map type queue-threshold CPPR-SSH
R1(config-cmap)# match protocol ssh

R1(config)# class-map type queue-threshold CPPR-SNMP
R1(config-cmap)# match protocol snmp

R1(config)# class-map type queue-threshold CPPR-OTHER
R1(config-cmap)# match host-protocols
```

The first two classes match specific protocols.

The third class provides a broader match for other registered TCP/UDP host protocols.

### Step 2: Create a Queue-Threshold Policy Map

```text
R1(config)# policy-map type queue-threshold CPPR-QUEUE
R1(config-pmap)# class CPPR-SSH
R1(config-pmap-c)# queue-limit 50
R1(config-pmap-c)# exit
R1(config-pmap)# class CPPR-SNMP
R1(config-pmap-c)# queue-limit 25
R1(config-pmap-c)# exit
R1(config-pmap)# class CPPR-OTHER
R1(config-pmap-c)# queue-limit 150
```

The `queue-limit` command sets the maximum number of outstanding packets permitted for the matching protocol category.

In Cisco's documented implementation, the configurable range is **0 to 255 packets**.

### Step 3: Apply the Policy to the Host Subinterface

```text
R1(config)# control-plane host
R1(config-cp-host)# service-policy type queue-threshold input CPPR-QUEUE
```

This activates queue thresholding on the host processing path.

**The `type queue-threshold` keyword is required** to apply this specialized policy type.

### Example: SNMP Queue Limit

Suppose the SNMP queue limit is 25 packets.

If 20 SNMP packets are waiting for processing, additional packets may be admitted until the configured limit is reached.

Once the queue reaches its limit, further matching packets are dropped rather than admitted to that queue.

As queued packets are processed, space becomes available for new packets.

The mechanism therefore limits **simultaneous outstanding packets**, rather than imposing a fixed packets-per-second rate.

---

## Queue-Threshold Restrictions

In the documented CPPr implementation:

- Queue thresholding is supported only on the **host subinterface**.
- Only **input** queue-threshold policies are supported.
- Classification is limited to supported protocol matches.
- The documented policy action is `queue-limit`.
- Queue limits are measured in **packets**, not bytes.
- The documented configuration range is **0 to 255**.
- Traditional CPPr queue thresholding is limited to the **IPv4 input path**.

A queue limit that is too low can cause legitimate protocol packets to be dropped during bursts.

For example, restricting BGP or management traffic too aggressively can interfere with protocol operation.

---

## Combining CoPP, Port Filtering, and Queue Thresholding

On platforms supporting all three features, separate policies can be applied to the same host subinterface.

For example:

```text
R1(config)# control-plane host
R1(config-cp-host)# service-policy input CPPR-HOST-POLICING
R1(config-cp-host)# service-policy type port-filter input CPPR-PORT-FILTER
R1(config-cp-host)# service-policy type queue-threshold input CPPR-QUEUE
```

Each policy performs a different function:

| Policy | Function |
|---|---|
| Ordinary MQC policing | Limits traffic rate using a token bucket |
| Port filtering | Drops traffic matching unwanted TCP/UDP port criteria |
| Queue thresholding | Restricts the number of outstanding packets for particular protocols |

These features complement one another.

For example, an incoming SSH packet might be subject to host policing and queue thresholding, while a packet destined for a closed TCP port might be discarded by the port-filter policy.

> Availability of simultaneous policy attachments and the precise feature processing order are platform-dependent. Do not assume that every IOS XE platform supports all three.

---

## IOS XE Verification

### Display the Registered Open Ports

```text
R1# show control-plane host open-ports
```

This displays the TCP and UDP ports registered in the port-filter database.

Example output:

```text
Active internet connections (servers and established)

Port    Local Address    Foreign Address    Service       State
tcp     *:22             *:0                SSH           LISTEN
tcp     *:179            *:0                BGP           LISTEN
udp     *:161            *:0                SNMP          LISTEN
```

This output is illustrative.

**This is an important command when troubleshooting unexpected port-filter drops.**

If a legitimate service is missing from the registered open-port database, traffic for that service could match `closed-ports`.

### Display Port-Filter Class Maps

```text
R1# show class-map type port-filter
```

Displays the configured port-filter classes and matching criteria.

### Display Queue-Threshold Class Maps

```text
R1# show class-map type queue-threshold
```

Displays the configured protocol classification criteria.

### Display Port-Filter Policy Maps

```text
R1# show policy-map type port-filter
```

Displays port-filter policy definitions.

### Display Queue-Threshold Policy Maps

```text
R1# show policy-map type queue-threshold
```

Displays queue-threshold policy definitions.

### Display Applied Control-Plane Policies

```text
R1# show policy-map control-plane all
```

Displays available statistics for the aggregate control-plane policy and subinterface policies.

On platforms supporting the relevant command options, the policy type can be specified:

```text
R1# show policy-map control-plane type port-filter host
R1# show policy-map control-plane type queue-threshold host
```

The exact supported syntax and counters vary by platform.

### Verify the Running Configuration

```text
R1# show running-config | section control-plane
```

Check that the expected policies are attached to the host subinterface.

---

## Platform Support and Practical Considerations

The traditional CPPr port-filter and queue-threshold commands are documented in Cisco IOS control-plane protection guides.

**These features are not universally available on modern IOS XE platforms.**

For example, CSR1000v and ASR1000-family devices may expose a `control-plane host` configuration mode without supporting the full set of traditional CPPr port-filter and queue-threshold commands.

Similarly, Catalyst 9000 switches use a different CPU protection architecture based on **System CoPP**.

The availability of `control-plane host` alone does not guarantee support for:

```text
class-map type port-filter
class-map type queue-threshold
service-policy type port-filter input
service-policy type queue-threshold input
```

Check the target platform's CLI and feature documentation before relying on these features.

For the CCIE EI exam, the important concepts are how these mechanisms differ from ordinary policing, their specialized commands, and their restrictions.

---

## Key Points

- **Port filtering** drops traffic matching selected TCP/UDP ports or ports classified as closed.
- Port filtering uses `class-map type port-filter` and `policy-map type port-filter`.
- Use **`match closed-ports`** to match ports not registered as open.
- Use **`match port tcp/udp <port>`** to match a specific port.
- Use **`match not port tcp/udp <port>`** to exclude a port from a drop class.
- Use **`match-all`** when combining `match closed-ports` with port exclusions.
- The documented port-filter policy action is **`drop`**.
- Use **`show control-plane host open-ports`** to inspect the registered open-port database.
- Not all legitimate services are necessarily detected as open, so explicit exclusions may be required.
- **Queue thresholding** limits the number of unprocessed packets for a protocol in the control-plane input queue.
- Queue thresholding uses `class-map type queue-threshold` and `policy-map type queue-threshold`.
- Use **`match protocol <protocol>`** to classify supported protocols.
- Use **`queue-limit <number>`** to define the maximum number of outstanding packets.
- The documented queue-limit range is **0 to 255 packets**.
- Queue thresholding is **not** equivalent to policing in packets per second.
- Both features are supported only on the **host subinterface** and only in the **input direction** in the documented CPPr architecture.
- Port filtering and queue thresholding are **platform-dependent**, and support should not be assumed merely because `control-plane host` is available.

---

## References

- [Cisco - Control Plane Protection](https://www.cisco.com/en/US/docs/ios-xml/ios/qos_plcshp/configuration/15-1mt/qos-plcshp-cpp.html)
- [Cisco IOS QoS Command Reference - show control-plane host open-ports](https://www.cisco.com/c/en/us/td/docs/ios/qos/command/reference/qos_book/qos_s2.html)
- [Cisco IOS QoS Command Reference - show policy-map control-plane](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/qos/command/qos-cr-book/qos-s4.html)
- [Cisco IOS XE 17.x - Control Plane Policing](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/qos/b-quality-of-service/m_qos-plcshp-ctrl-pln-plc-0.html)