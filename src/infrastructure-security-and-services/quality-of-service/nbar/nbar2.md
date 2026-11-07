# NBAR2 Protocol Packs and Custom Applications

**Next-Generation Network Based Application Recognition (NBAR2)** identifies applications using protocol signatures and information extracted from network traffic. It supports a broader application library and more flexible classification than matching traffic using IP addresses or TCP/UDP ports alone.

Two features extend NBAR2's recognition capabilities:
- **Protocol packs:** Cisco-provided packages that update the library of recognized applications and their signatures.
- **Custom applications:** Administrator-defined applications that NBAR2 can recognize using ports, addresses, payload patterns, or information from protocols such as HTTP, TLS/SSL, and DNS.

Both can be used for **application visibility** through Protocol Discovery or for **application-based QoS** with Modular QoS CLI (MQC).

---

## NBAR2 Application Recognition

NBAR2 uses a **protocol engine** and a **protocol library** to identify the application associated with a flow. It can use information such as:
- TCP or UDP ports and IP addresses.
- Protocol-specific signatures in packet payloads.
- HTTP hostnames and URLs.
- TLS/SSL server-name information, when visible.
- DNS queries and responses.
- Stateful inspection of multiple packets within a flow.

For example, several applications may use TCP port 443. Matching TCP port 443 identifies HTTPS-related traffic, but does not reliably distinguish a business application from video streaming or file sharing.

NBAR2 may distinguish those applications using additional information supplied by its signatures.

**NBAR2 classification identifies an application. It does not independently prioritize, police, shape, or mark traffic.** Those actions require an appropriate QoS policy.

The distinction between a **built-in application**, a **custom application**, and an **unknown application** is important:

| Classification | Meaning |
|---|---|
| **Built-in application** | An application recognized using signatures provided with the active protocol pack |
| **Custom application** | An application identified using an administrator-defined matching rule |
| **Unknown** | Traffic that has not yet been identified or could not be identified by NBAR2 |
| **Unknown final** | Traffic for which NBAR2 has reached a final unknown classification |

See [NBAR Operation and Protocol Discovery](protocol-discovery.md) for the recognition process and statistics, and [NBAR Classification with MQC](mqc-classification.md) for QoS policy integration.

---

## NBAR2 Protocol Packs

A **protocol pack** contains Cisco-provided application definitions, signatures, and associated application information used by NBAR2.

Cisco IOS XE includes a **built-in protocol pack**. As applications and their network behavior change, Cisco can release updated protocol packs.

An updated protocol pack can:
- Add recognition for additional applications.
- Update existing application signatures.
- Improve recognition when applications change their traffic patterns.
- Update application taxonomy and attributes.

**A protocol pack update does not require an IOS XE software upgrade.** This allows the application-recognition library to be updated independently of the router or switch software, provided the protocol pack is compatible with the installed NBAR engine.

### Protocol Pack vs. NBAR Engine

These are separate components:

| Component | Function | Updated by |
|---|---|---|
| **NBAR engine** | Executes application-recognition logic and processes signatures | IOS XE software releases |
| **Protocol pack** | Supplies application definitions and signatures used by the engine | Built-in package or compatible protocol-pack update |

For example, an IOS XE image may include an NBAR engine and a built-in protocol pack. Loading a newer compatible protocol pack updates the available signatures, but **does not replace the NBAR engine**.

Protocol packs are not universally interchangeable. Their compatibility depends on the NBAR engine, platform, and software release. Check the [Cisco NBAR2 Protocol Pack Library](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/qos_nbar/prot_lib/config_library/nbar-prot-pack-library.html) before selecting a pack. Additional application coverage and protocol-pack upgrades can also depend on licensing.

### Checking the Active Protocol Pack

Display NBAR software and loaded pack information:

```text
R1# show ip nbar version
R1# show ip nbar protocol-pack active
R1# show ip nbar protocol-pack active detail
```

The output identifies information such as:
- NBAR software/engine version.
- Active protocol-pack name and version.
- Publisher.
- Protocol-pack state.

The **active** protocol pack is the one currently used for application recognition.

### Loading an Updated Protocol Pack

First, download a compatible protocol pack from Cisco and copy it to persistent local storage on the device.

For example, suppose the file is stored in flash as `newProtocolPack`:

```text
R1(config)# ip nbar protocol-pack flash:newProtocolPack
```

Verify that it is active:

```text
R1# show ip nbar version
R1# show ip nbar protocol-pack active detail
```

The filename above is illustrative; use the actual pack filename and filesystem available on the device.

On supported IOS XE platforms, a compatible protocol-pack change can be applied **without reloading the device**. In a switch stack, the necessary pack file must be available on all relevant members before activation.

### Reverting to the Built-in Protocol Pack

To stop using an externally loaded pack and return to the one supplied with IOS XE:

```text
R1(config)# default ip nbar protocol-pack
```

Verify the active pack afterward:

```text
R1# show ip nbar protocol-pack active
```

### Loading an Older Protocol Pack

Cisco provides a `force` option for loading a lower-version protocol pack:

```text
R1(config)# ip nbar protocol-pack flash:olderProtocolPack force
```

**`force` can remove configuration that the older pack does not support.** Use it only when a downgrade is required, and verify any policies that reference applications or features removed by the downgrade.

### Viewing the Protocol Library and Taxonomy

The **taxonomy** describes the applications in the protocol pack, including their names and related classification information.

```text
R1# show ip nbar protocol-pack active taxonomy
```

This output can be lengthy. Cisco documents redirecting it to a file:

```text
R1# show ip nbar protocol-pack active taxonomy | redirect flash:nbar-taxonomy.xml
```

The taxonomy can help determine whether a particular application is defined in the active pack. It is separate from observing whether that application is currently present in live traffic.

---

## NBAR2 Custom Applications

A **custom application**, also called a **custom protocol**, defines recognition criteria for traffic that needs to be identified separately from the built-in applications.

Typical uses include:
- An internally developed application using a dedicated TCP/UDP port.
- A business application hosted at a particular domain name.
- Several services associated with the same organization or server name.
- Traffic with an identifiable string or pattern in its payload.

Custom applications are configured globally using **`ip nbar custom`**. After creation, they can be referenced by name in NBAR classification and monitoring.

NBAR2 supports several customization methods:

| Method | Recognition criteria | Example use |
|---|---|---|
| **Transport (Layer 3/4)** | TCP/UDP, IP addresses/subnets, ports, DSCP, direction | Internal application on TCP 6700 |
| **HTTP** | HTTP fields, including hostnames | Web application identified by HTTP Host |
| **TLS/SSL** | Visible server identity information, such as SNI | HTTPS application identified by hostname |
| **DNS** | Domain names in observed DNS traffic | Application associated with a DNS name |
| **Composite** | Related HTTP, TLS/SSL, and DNS server-name signatures | Group a service under one custom application |
| **Payload pattern** | Fixed byte/string patterns at specified offsets | Proprietary application with a known signature |

**Custom rules can take precedence over built-in recognition.** This makes them useful for distinguishing an internal application, but a broad rule may also override a more specific built-in classification.

### Custom Application IDs

Custom applications may be assigned a **selector ID** using the `id` keyword. The ID identifies the custom application in NBAR reporting.

For example:

```text
R1(config)# ip nbar custom INTERNAL-WEB http host *app.example.com id 10
```

In this example:
- `INTERNAL-WEB` is the custom application name.
- `http host *app.example.com` defines its matching criteria.
- `id 10` is its selector ID.

The application name can subsequently be referenced in an MQC class map. The selector ID is primarily useful when viewing or reporting application information.

---

## Transport-Based Custom Applications

A **transport-based custom application** identifies traffic using packet header fields, rather than requiring a built-in application signature.

For example, suppose an internal application uses **TCP port 6700**. Define an application named `INTERNAL-APP`:

```text
R1(config)# ip nbar custom INTERNAL-APP transport tcp id 100
R1(config-custom)# port 6700
R1(config-custom)# exit
```

This creates an NBAR custom application for TCP traffic that matches the configured port criterion.

Additional criteria can be configured in custom-application configuration mode:

| Command | Purpose |
|---|---|
| `port <port>` | Match a specified TCP or UDP port |
| `port range <start> <end>` | Match a range of ports |
| `ip address <address>` | Match specified IPv4 addresses |
| `ip subnet <subnet> <prefix-length>` | Match an IPv4 subnet |
| `ipv6 address <address>` | Match specified IPv6 addresses |
| `dscp <value>` | Include DSCP as a matching criterion |
| `direction <value>` | Specify the relevant traffic direction |

These criteria let NBAR2 identify a specific internal service more precisely than simply matching all traffic using a common port.

For example, the following custom application identifies TCP traffic associated with two particular addresses and a specific DSCP value:

```text
R1(config)# ip nbar custom INTERNAL-TRANSACTION transport tcp id 101
R1(config-custom)# ip address 192.0.2.10 192.0.2.11
R1(config-custom)# dscp af31
R1(config-custom)# exit
```

Transport-based recognition can classify a flow using its initial packet headers. Unlike deeper application identification, it does not need to wait for an HTTP request, TLS handshake, or other application data.

**Remember:** A port-based custom application recognizes traffic according to the rule you define. It does not prove that every matching packet actually belongs to the intended application.

---

## HTTP-Based Custom Applications

NBAR2 can define a custom application using fields from an **unencrypted HTTP request**.

For example, to recognize HTTP traffic with a matching Host field:

```text
R1(config)# ip nbar custom INTERNAL-HTTP http host *app.example.com id 102
```

This creates a custom application named `INTERNAL-HTTP` using an HTTP hostname criterion.

HTTP-based customization can use other HTTP information, such as URL-related fields, depending on the supported syntax.

Because these criteria depend on inspecting HTTP messages, **they do not reveal the HTTP request inside encrypted HTTPS traffic**. To classify HTTPS-related applications by visible server information, use the TLS/SSL or composite methods instead.

---

## TLS/SSL-Based Custom Applications

NBAR2 can also define a custom application using **TLS/SSL server-name information**.

For example:

```text
R1(config)# ip nbar custom INTERNAL-HTTPS ssl unique-name *app.example.com id 103
```

The `ssl unique-name` criterion can use identifying information such as:
- **Server Name Indication (SNI)** in a TLS ClientHello, if visible.
- The certificate's **Common Name (CN)** on connections where the certificate information is visible and supported for inspection.

This is useful because NBAR2 may distinguish HTTPS services **without decrypting application payloads**.

However, the result depends on what metadata is observable. For example, encrypted client hello mechanisms can hide the true SNI, and modern TLS versions encrypt certificate contents. The command does **not** decrypt the HTTPS session.

---

## DNS-Based Custom Applications

NBAR2 can identify applications using **DNS domain names**.

For example:

```text
R1(config)# ip nbar custom INTERNAL-DNS dns domain-name *app.example.com id 104
```

NBAR2 can examine observed DNS queries and responses and associate a returned IP address with the specified application. That information can help classify subsequent connections involving the learned address.

For example:

```text
Client                       DNS Server
  |                              |
  | Query app.example.com        |
  |----------------------------->|
  |  Response: 192.0.2.50        |
  |<-----------------------------|
  |                              
  |  Connection to 192.0.2.50    
  |------------------------------------> Application Server
```

The DNS response helps NBAR2 associate the server's address with the application.

This mechanism depends on observing and correlating the relevant DNS traffic. Its effectiveness can be reduced when DNS requests are encrypted, bypass the device, or when an address is shared by multiple unrelated applications.

---

## Composite Custom Applications

A **composite custom application** combines multiple underlying recognition methods under a single custom application name.

For example:

```text
R1(config)# ip nbar custom INTERNAL-SERVICE composite server-name *app.example.com id 105
```

This can identify related traffic using server-name information from **HTTP, TLS/SSL, and DNS**, instead of requiring separate custom applications for each.

| Method | Information examined |
|---|---|
| HTTP | Hostname-related information from HTTP |
| TLS/SSL | Visible server-name information |
| DNS | Domain names and associated DNS information |

A composite rule is useful when one service may be accessed using several protocols but should receive the same QoS classification.

For example, an enterprise can create one custom application for its internal collaboration platform and use that custom application in an MQC class map.

---

## Payload-Based Custom Applications

Some custom applications can be defined using a **known string or byte pattern** at a specified offset in the application data.

For example, Cisco documents the following form for an application sending the ASCII string `SALES` at offset 5 over TCP port 4567:

```text
R1(config)# ip nbar custom APP-SALES 5 ascii SALES source tcp 4567
```

The matching process examines the specified pattern, rather than relying solely on the port number.

The components include:
- `APP-SALES`: Name of the custom application.
- `5`: Offset used for the signature.
- `ascii SALES`: Expected text pattern.
- `source tcp 4567`: Source-oriented matching over the specified TCP port.

Payload matching is useful for proprietary protocols with known signatures. Encrypted payloads cannot be matched using plaintext patterns without access to their decrypted contents.

---

## Using Custom Applications with MQC

Once a custom application has been created, it can be used with the same **`match protocol`** command as a built-in NBAR application.

Suppose the goal is to recognize an internal TCP application on port 6700 and mark its incoming traffic as **AF31**.

### Step 1: Define the Custom Application

```text
R1(config)# ip nbar custom INTERNAL-APP transport tcp id 100
R1(config-custom)# port 6700
R1(config-custom)# exit
```

### Step 2: Define an MQC Class Map

```text
R1(config)# class-map match-all BUSINESS-APP
R1(config-cmap)# match protocol INTERNAL-APP
R1(config-cmap)# exit
```

The `BUSINESS-APP` class now matches traffic recognized as the custom NBAR application.

### Step 3: Configure the Marking Policy

```text
R1(config)# policy-map CLASSIFY-AND-MARK
R1(config-pmap)# class BUSINESS-APP
R1(config-pmap-c)# set dscp af31
R1(config-pmap-c)# exit
```

### Step 4: Attach the Policy

```text
R1(config)# interface GigabitEthernet1
R1(config-if)# service-policy input CLASSIFY-AND-MARK
```

The router can now identify the custom application and mark its traffic **AF31 (26)** on ingress.

Downstream devices can match the DSCP value without performing the same application inspection:

```text
R2(config)# class-map match-all BUSINESS
R2(config-cmap)# match dscp af31
```

This follows the DiffServ design of performing detailed application recognition near the network edge and simpler DSCP-based classification within the network.

**Platform detail:** On Catalyst 9000 wired AVC QoS, NBAR `match protocol` classification is used in ingress marking or policing policies. Configure egress queuing by matching the resulting DSCP value instead of directly using an NBAR match in an egress queuing policy.

See [NBAR Classification with MQC](mqc-classification.md) and [DSCP Marking and Remarking with MQC](../marking/mqc-marking.md).

---

## Application Attributes and Categories

NBAR2 assigns **attributes** to recognized applications. These attributes allow policies to match groups of applications rather than naming every application individually.

Examples include:

| Attribute | Meaning |
|---|---|
| `application-group` | Groups related applications from the same application family |
| `category` | Broad application category, such as email or streaming |
| `sub-category` | More specific group within an application category |
| `encrypted` | Indicates the application's assigned encryption-related classification |
| `tunnel` | Indicates whether an application is categorized as carrying other traffic in a tunnel |
| `p2p-technology` | Indicates peer-to-peer technology classification |
| `traffic-class` | Application's suggested traffic-class category |
| `business-relevance` | Application's business-relevance classification |

**These attributes are metadata assigned to the application definition.** For example, matching `business-relevance` does not mean the router has independently determined whether a particular flow is important to the organization.

### Matching Applications by Attribute

Instead of listing applications one by one, an MQC class can match an attribute:

```text
R1(config)# class-map match-all BUSINESS-RELEVANT
R1(config-cmap)# match protocol attribute business-relevance business-relevant
```

This matches applications assigned that business-relevance attribute.

Attributes can also be combined in a `match-all` class map:

```text
R1(config)# class-map match-all BUSINESS-VOICE
R1(config-cmap)# match protocol attribute traffic-class voip-telephony
R1(config-cmap)# match protocol attribute business-relevance business-relevant
```

The class requires **both attributes** to match.

Attribute matching reduces the need for a long list of separate `match protocol` statements when an entire application group should receive the same treatment.

### Viewing Application Attributes

Display an application's assigned attributes:

```text
R1# show ip nbar protocol-attribute cisco-jabber-im
```

On supporting platforms, the output contains fields such as `category`, `application-group`, `traffic-class`, and `business-relevance`.

### Custom Attribute Values

NBAR2 can also define custom values for selected attributes, including `application-group`, `category`, and `sub-category`:

```text
R1(config)# ip nbar attribute category custom internal-apps "Internal enterprise applications"
```

The administrator can organize application definitions using **attribute maps** and associate a map with an application:

```text
R1(config)# ip nbar attribute-map INTERNAL-ATTRIBUTES
R1(config-attribute-map)# attribute category internal-apps
R1(config-attribute-map)# exit
R1(config)# ip nbar attribute-set INTERNAL-APP INTERNAL-ATTRIBUTES
```

This associates the custom category with the `INTERNAL-APP` protocol, allowing attribute-based classification to reflect an enterprise's own groupings.

Verify defined attribute values and mappings:

```text
R1# show ip nbar attribute-custom
R1# show ip nbar attribute category
```

---

## Verifying NBAR2 Configuration

The following commands are useful when verifying protocol packs and custom applications:

| Command | Purpose |
|---|---|
| `show ip nbar version` | Display NBAR engine and loaded protocol-pack information |
| `show ip nbar protocol-pack active detail` | Display the currently active protocol pack |
| `show ip nbar protocol-pack active taxonomy` | Display application information from the active pack |
| `show ip nbar protocol-id` | Display application names, IDs, and types |
| `show ip nbar protocol-id <name>` | Inspect a particular application ID |
| `show ip nbar port-map` | Display NBAR's protocol-to-port mappings |
| `show ip nbar protocol-discovery` | Display observed application traffic statistics |
| `show ip nbar protocol-attribute <name>` | Display attributes assigned to an application |
| `show class-map` | Verify MQC class-map matching criteria |
| `show policy-map interface` | Check QoS policy matches and counters |

### Verify a Custom Application ID

For example:

```text
R1# show ip nbar protocol-id INTERNAL-APP
```

The output should identify the custom application and its selector ID.

To list applications identified as custom:

```text
R1# show ip nbar protocol-id | include Custom
```

### Verify Traffic Recognition

Enable Protocol Discovery on a relevant interface when application traffic statistics are needed:

```text
R1(config)# interface GigabitEthernet1
R1(config-if)# ip nbar protocol-discovery
```

Then generate traffic and inspect recognition results:

```text
R1# show ip nbar protocol-discovery interface GigabitEthernet1
```

**Protocol Discovery is for observing classified traffic.** It is not a prerequisite for a separately configured MQC `match protocol` policy to classify and act on packets.

### Verify the Applied QoS Policy

```text
R1# show policy-map interface GigabitEthernet1
```

Confirm that:
1. The expected service policy is applied in the correct direction.
2. The class map references the correct custom application name.
3. The class counter increases when matching traffic is generated.
4. The configured marking or policing action is shown for the class.

If Protocol Discovery recognizes an application but no QoS action occurs, inspect the **service policy and class-map matches** rather than assuming that Protocol Discovery applies a policy.

---

## Troubleshooting

| Symptom | What to check |
|---|---|
| Expected application is not recognized | Verify the active pack, application name, and whether the traffic exposes the expected signature |
| New protocol pack does not activate | Check platform and engine compatibility, license, and file location |
| Custom domain application does not match | Verify the exact hostname pattern and whether HTTP/TLS/DNS metadata is visible |
| Custom transport application matches too much traffic | Narrow the rule with addresses, ports, DSCP, or direction |
| Traffic appears as `unknown` | Check whether NBAR has enough packets to identify the flow and whether its signatures support the application |
| Custom application appears in NBAR but receives no QoS treatment | Check the class map, policy map, service-policy attachment, and match counters |
| Recognition changes after a protocol-pack update | Inspect updated application definitions and any affected class maps |

A protocol pack improves recognition only for applications and patterns the updated library can identify. It does not bypass encryption or guarantee visibility into every application.

---

## Key Points

- **NBAR2 protocol packs** provide Cisco-maintained application definitions and signatures independently of the IOS XE software image.
- **The NBAR engine and protocol pack are separate.** A pack must be compatible with the installed engine and platform.
- **`ip nbar protocol-pack <file>`** loads a compatible protocol pack; **`default ip nbar protocol-pack`** returns to the built-in pack.
- **`show ip nbar version`** and **`show ip nbar protocol-pack active detail`** verify the active pack.
- **`ip nbar custom`** defines custom applications using transport headers, HTTP, TLS/SSL, DNS, composite server-name information, or payload patterns.
- **Transport-based custom applications** can use packet header information from the start of a flow.
- **Composite custom applications** can associate HTTP, TLS/SSL, and DNS recognition with one custom application.
- **A custom application can be referenced using `match protocol <name>`** in an MQC class map.
- **`match protocol attribute`** classifies applications according to their assigned attributes rather than matching a list of application names.
- **Protocol Discovery displays application statistics, while MQC applies QoS actions.**
- A common design is to **identify and mark applications at ingress, then queue traffic based on DSCP at egress**.

---

## Further Reading

- [4.4.c Network Based Application Recognition (NBAR)](overview.md)
- [NBAR Operation and Protocol Discovery](protocol-discovery.md)
- [NBAR Classification with MQC](mqc-classification.md)
- [MQC Class Maps and Match Criteria](../classification/class-maps.md)
- [DSCP Marking and Remarking with MQC](../marking/mqc-marking.md)

### Cisco Documentation

- [Cisco IOS XE: NBAR2 Protocol Pack](https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/quality-of-service/quality-of-service/m_nbar-protocl-pack-1.html)
- [Cisco IOS XE 17.x: NBAR2 Custom Protocol](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/qos/b-quality-of-service/m_nbar2-custm-protocl.html)
- [Cisco IOS XE 17.x: NBAR Categorization and Attributes](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/qos/b-quality-of-service/m_nbar2-cat-attrib-0.html)
- [Cisco IOS XE: Application Visibility and Control QoS](https://www.cisco.com/c/en/us/td/docs/switches/lan/c9000/qos/quality-of-service-configuration-guide/application-visibility-and-control-qos.html)
- [Cisco NBAR2 Protocol Pack Library](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/qos_nbar/prot_lib/config_library/nbar-prot-pack-library.html)
