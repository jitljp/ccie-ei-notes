# NETCONF Configuration and Operations

The **Network Configuration Protocol (NETCONF)** provides programmatic access to network device configuration and operational data.

Cisco IOS XE implements NETCONF using XML-encoded messages transported over SSH, with YANG models defining the structure of the data.

This section covers NETCONF configuration, session establishment, capabilities, RPC operations, configuration datastores, and troubleshooting.

## Lab Topology

The following topology is used throughout this section.

```text
 Management Workstation
      192.0.2.100/24
             |
             | NETCONF (TCP/830)
             |
             | G1
        192.0.2.1/24
            R1
             |
          Loopback0
          1.1.1.1/32
```

Configure R1 as follows:

```text
R1(config)# interface GigabitEthernet1
R1(config-if)# ip address 192.0.2.1 255.255.255.0
R1(config-if)# no shutdown
!
R1(config)# interface Loopback0
R1(config-if)# ip address 1.1.1.1 255.255.255.255
```

The management workstation must have IP connectivity to R1.

A Linux workstation with an OpenSSH client is sufficient for the initial exercises. Python and the `ncclient` library can be used for subsequent exercises.

## Enabling NETCONF

NETCONF is disabled by default on Cisco IOS XE.

Enable NETCONF using the following global configuration command:

```text
R1(config)# netconf-yang
```

This enables the model-driven NETCONF interface.

By default, the NETCONF SSH server listens on **TCP port 830**.

Disable NETCONF using:

```text
R1(config)# no netconf-yang
```

The NETCONF server is distinct from the traditional IOS XE SSH server used for interactive CLI access.

| Service | Default TCP Port | Purpose |
|---|---|---|
| SSH CLI | 22 | Interactive CLI access |
| SCP | 22 | File transfers |
| NETCONF | 830 | Model-driven device management |
| RESTCONF | 443 | HTTP-based model-driven device management |

Enabling traditional SSH CLI access does not automatically enable NETCONF.

Similarly, enabling NETCONF does not require configuring SSH transport on the VTY lines.

### Basic Authentication Configuration

Create a local user with privilege level 15 and enable NETCONF:

```text
R1(config)# username ADMIN privilege 15 secret ADMIN-PASSWORD
R1(config)# netconf-yang
```

The NETCONF user must have sufficient administrative privileges.

For standard IOS XE NETCONF access, the user must have privilege level 15.

**Enabling AAA using `aaa new-model` is not mandatory for NETCONF operation.**

However, if AAA is enabled, appropriate AAA authentication and authorization must be configured.

### Authentication and Authorization with AAA

The following example configures NETCONF authentication and authorization using the local username database.

```text
R1(config)# username ADMIN privilege 15 secret ADMIN-PASSWORD
!
R1(config)# aaa new-model
R1(config)# aaa authentication login default local
R1(config)# aaa authorization exec default local
!
R1(config)# netconf-yang
```

The AAA configuration performs two distinct functions:

- `aaa authentication login default local`: Authenticates the connecting user against the local username database.
- `aaa authorization exec default local`: Determines the user's authorization and privilege level using the local username database.

NETCONF can also use centralized AAA services such as TACACS+ and RADIUS.

NETCONF is not an interactive VTY session. Therefore, configuring `login local` or `login authentication` under the VTY lines does not configure the NETCONF authentication method.

### Named AAA Method Lists

Prior to Cisco IOS XE 17.9.1, NETCONF supports only the default AAA authentication and authorization method lists.

Starting with Cisco IOS XE 17.9.1, named AAA method lists can be associated with the model-driven management interfaces.

For example:

```text
R1(config)# aaa new-model
R1(config)# aaa authentication login YANG-AUTHN local
R1(config)# aaa authorization exec YANG-AUTHZ local
!
R1(config)# yang-interfaces aaa authentication method-list YANG-AUTHN
R1(config)# yang-interfaces aaa authorization method-list YANG-AUTHZ
!
R1(config)# netconf-yang
```

The `yang-interfaces aaa` commands associate the named method lists with the model-driven management interfaces.

This allows their authentication and authorization policies to be configured separately from traditional CLI access.

## NETCONF Verification

After enabling NETCONF, verify that the required model-driven management processes are running.

```text
R1# show platform software yang-management process
```

Important processes include:

| Process | Function |
|---|---|
| `confd` | Configuration management daemon |
| `nesd` | Network element synchronization |
| `syncfd` | Synchronization from the IOS configuration |
| `ncsshd` | NETCONF SSH server |
| `dmiauthd` | Model-driven interface authentication |
| `opdatamgrd` | Operational data management |
| `nginx` | HTTP server used by RESTCONF |

The `nginx` process is required for RESTCONF but is not required for NETCONF operation.

The complete initialization of model-driven management processes can take up to approximately 90 seconds after NETCONF is enabled.

Additional verification commands include:

| Command | Purpose |
|---|---|
| `show netconf-yang sessions` | Display active sessions |
| `show netconf-yang sessions detail` | Display detailed session information |
| `show netconf-yang datastores` | Display datastores and lock information |
| `show netconf-yang statistics` | Display protocol statistics |
| `show netconf-yang diagnostics summary` | Display diagnostic information |
| `show platform software yang-management process` | Display management process status |

## Establishing a NETCONF Session

NETCONF clients can establish sessions using dedicated NETCONF applications or an SSH client capable of requesting the NETCONF subsystem.

From the management workstation:

```bash
ssh -p 830 -s ADMIN@192.0.2.1 netconf
```

The command includes the following components:

| Component | Purpose |
|---|---|
| `ssh` | Initiates an SSH connection |
| `-p 830` | Specifies the destination TCP port |
| `-s` | Requests an SSH subsystem |
| `ADMIN@192.0.2.1` | Specifies the username and server address |
| `netconf` | Specifies the NETCONF subsystem |

Unlike a traditional SSH CLI session, this connection does not provide an IOS XE command prompt.

Instead, the client and server exchange XML-encoded NETCONF messages.

### Session Establishment Process

A NETCONF session involves the following stages:

1. Establish a TCP connection.
2. Negotiate SSH parameters and authenticate the server.
3. Authenticate the user.
4. Request the NETCONF SSH subsystem.
5. Exchange NETCONF `<hello>` messages.
6. Exchange RPC requests and replies.

```text
NETCONF Client                         R1
      |                                |
      | ----- TCP Connection --------> |
      | <---- SSH Negotiation -------> |
      | <---- Authentication --------> |
      | ----- NETCONF Subsystem -----> |
      |                                |
      | ----- <hello> ---------------> |
      | <---- <hello> ---------------- |
      |                                |
      | ----- <rpc> -----------------> |
      | <---- <rpc-reply> ------------ |
      |                                |
```

Both peers send a `<hello>` message independently after the secure connection has been established.

The `<hello>` messages identify the capabilities supported by each peer.

## NETCONF Hello Messages and Capabilities

### Server Hello Message

After connecting to R1 using the OpenSSH client, R1 sends a `<hello>` message.

The following is a simplified example:

```xml
<hello xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <capabilities>
    <capability>
      urn:ietf:params:netconf:base:1.0
    </capability>
    <capability>
      urn:ietf:params:netconf:base:1.1
    </capability>
    <capability>
      urn:ietf:params:netconf:capability:writable-running:1.0
    </capability>
    <capability>
      urn:ietf:params:netconf:capability:xpath:1.0
    </capability>
  </capabilities>
  <session-id>42</session-id>
</hello>
]]>]]>
```

An actual IOS XE `<hello>` message typically contains many additional capabilities and YANG model identifiers.

The `<session-id>` element identifies the NETCONF session assigned by the server.

Only the server includes a session ID in its `<hello>` message.

### Client Hello Message

The client must also send a `<hello>` message before it can perform NETCONF operations.

For manual testing, advertise only NETCONF version 1.0:

```xml
<hello xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <capabilities>
    <capability>urn:ietf:params:netconf:base:1.0</capability>
  </capabilities>
</hello>
]]>]]>
```

Copy and paste this message into the SSH session.

The client and server can now exchange NETCONF RPC messages.

Advertising only version 1.0 simplifies manual testing because subsequent messages use an end-of-message delimiter rather than chunked framing.

### NETCONF Capabilities

Capabilities identify supported protocol features.

The following table lists several important capabilities.

| Capability | Function |
|---|---|
| `:base:1.0` | NETCONF version 1.0 |
| `:base:1.1` | NETCONF version 1.1 |
| `:writable-running` | Direct modification of the running datastore |
| `:candidate` | Candidate configuration datastore |
| `:confirmed-commit` | Commit operations with automatic rollback unless confirmed |
| `:startup` | Separate startup configuration datastore |
| `:validate` | Configuration validation |
| `:rollback-on-error` | Rollback of failed configuration edits |
| `:xpath` | XPath filtering |
| `:notification` | NETCONF event notifications |
| `:with-defaults` | Control over the retrieval of default-valued data |

These are abbreviated capability names. Actual capability identifiers are URIs.

For example:

```text
urn:ietf:params:netconf:capability:candidate:1.0
```

A NETCONF client should inspect the server's advertised capabilities rather than assume that every standardized operation is supported.

## NETCONF Message Framing

NETCONF requires a mechanism for identifying the boundaries between XML messages.

NETCONF over SSH supports two framing mechanisms.

| Version | Framing Mechanism |
|---|---|
| NETCONF 1.0 | End-of-message delimiter |
| NETCONF 1.1 | Chunked framing |

### NETCONF 1.0 Framing

NETCONF 1.0 uses the following end-of-message delimiter:

```text
]]>]]>
```

For example:

```xml
<rpc message-id="101"
     xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <get-config>
    <source>
      <running/>
    </source>
  </get-config>
</rpc>
]]>]]>
```

The delimiter indicates that the XML message is complete.

It is not part of the XML document itself.

If the delimiter is omitted during manual testing, the server may continue waiting for the remainder of the message.

### NETCONF 1.1 Framing

NETCONF 1.1 uses chunked framing.

Each chunk begins with a header specifying the length of the chunk in bytes. The end of the message is indicated using a separate end-of-chunks marker.

The general structure is:

```text
#<chunk-size>
<chunk-data>
##
```

Multiple chunks can be used to transport a single NETCONF message.

Both peers must advertise `:base:1.1` for chunked framing to be used.

**The initial `<hello>` messages always use the `]]>]]>` delimiter**, even when both peers support NETCONF 1.1.

After the exchange of `<hello>` messages:

- If both peers advertise NETCONF 1.1, chunked framing is used.
- Otherwise, if both support NETCONF 1.0, end-of-message framing is used.

NETCONF client libraries handle message framing automatically.

## NETCONF RPC Messages

NETCONF uses a Remote Procedure Call (RPC) mechanism.

The client sends an `<rpc>` message containing an operation, and the server returns an `<rpc-reply>` message.

For example:

```xml
<rpc message-id="101"
     xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <get-config>
    <source>
      <running/>
    </source>
  </get-config>
</rpc>
```

The `<rpc>` element contains two important attributes:

| Attribute | Purpose |
|---|---|
| `message-id` | Associates the request with its corresponding reply |
| `xmlns` | Specifies the XML namespace |

The server returns the same `message-id` in its reply.

For example:

```xml
<rpc-reply message-id="101"
           xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <data>
    <!-- Requested data -->
  </data>
</rpc-reply>
```

### RPC Reply Types

NETCONF RPC replies can contain different types of information.

| Element | Purpose |
|---|---|
| `<data>` | Data returned by a retrieval operation |
| `<ok/>` | Successful completion of an operation without returned data |
| `<rpc-error>` | Information about an unsuccessful operation |

YANG-defined RPCs can also return their own model-specific output.

For example, a successful configuration modification typically returns:

```xml
<rpc-reply message-id="102"
           xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <ok/>
</rpc-reply>
```

An unsuccessful operation might return:

```xml
<rpc-reply message-id="102"
           xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <rpc-error>
    <error-type>application</error-type>
    <error-tag>invalid-value</error-tag>
    <error-severity>error</error-severity>
    <error-message>Invalid configuration value</error-message>
  </rpc-error>
</rpc-reply>
```

The exact error information depends on the operation and the reason for its failure.

> The following RPC examples omit message framing unless explicitly shown. When sending them manually in a NETCONF 1.0 session, append `]]>]]>` after each complete XML message.

## XML Namespaces

XML namespaces distinguish elements belonging to different protocols and YANG models.

For example:

```xml
<rpc message-id="103"
     xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <get-config>
    <source>
      <running/>
    </source>
    <filter type="subtree">
      <native xmlns="http://cisco.com/ns/yang/Cisco-IOS-XE-native">
        <hostname/>
      </native>
    </filter>
  </get-config>
</rpc>
```

This example contains two namespaces.

| Namespace | Purpose |
|---|---|
| `urn:ietf:params:xml:ns:netconf:base:1.0` | NETCONF protocol elements |
| `http://cisco.com/ns/yang/Cisco-IOS-XE-native` | Cisco native YANG model elements |

Elements inherit the default namespace of their parent unless another namespace is specified.

The `<rpc>`, `<get-config>`, `<source>`, `<running>`, and `<filter>` elements belong to the NETCONF protocol namespace.

The `<native>` and `<hostname>` elements belong to the Cisco native YANG model namespace.

### Namespace Prefixes

XML namespaces can also be specified using prefixes.

For example:

```xml
<nc:rpc message-id="103"
        xmlns:nc="urn:ietf:params:xml:ns:netconf:base:1.0"
        xmlns:ios="http://cisco.com/ns/yang/Cisco-IOS-XE-native">
  <nc:get-config>
    <nc:source>
      <nc:running/>
    </nc:source>
    <nc:filter type="subtree">
      <ios:native>
        <ios:hostname/>
      </ios:native>
    </nc:filter>
  </nc:get-config>
</nc:rpc>
```

This request is equivalent to the previous request.

The prefix names are arbitrary. The namespace URIs determine which namespaces the elements belong to.

## Discovering YANG Models

NETCONF operations require knowledge of the structure and namespaces of the supported YANG models.

There are several ways to discover this information:

- Examine the server's advertised capabilities.
- Retrieve YANG library information.
- Retrieve YANG model definitions using `<get-schema>`, when supported.
- Examine Cisco's published YANG model files.
- Convert existing IOS XE configuration into XML.

### Cisco YANG Models

Cisco publishes YANG models for different IOS XE releases in the following repository:

[YangModels/yang](https://github.com/YangModels/yang/tree/main/vendor/cisco/xe)

It is important to examine the models corresponding to the software release running on the device.

Some commonly used models include:

| Model | Purpose |
|---|---|
| `Cisco-IOS-XE-native` | Native IOS XE configuration |
| `Cisco-IOS-XE-interfaces-oper` | Interface operational information |
| `Cisco-IOS-XE-bgp-oper` | BGP operational information |
| `Cisco-IOS-XE-routing-oper` | Routing operational information |
| `ietf-interfaces` | Standardized interface data |
| `ietf-netconf-monitoring` | NETCONF monitoring information |
| `ietf-yang-library` | Information about supported YANG modules |

### Retrieving a YANG Model

The `<get-schema>` operation is defined by the `ietf-netconf-monitoring` YANG model in RFC 6022.

It allows a client to retrieve the schema of a supported model from the NETCONF server.

For example:

```xml
<rpc message-id="104"
     xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <get-schema xmlns="urn:ietf:params:xml:ns:yang:ietf-netconf-monitoring">
    <identifier>Cisco-IOS-XE-native</identifier>
    <format>yang</format>
  </get-schema>
</rpc>
```

The response contains the requested YANG model definition.

A revision can also be specified using the `<version>` element.

### Displaying YANG Models as Trees

The `pyang` utility can convert YANG models into a more readable tree representation.

Install `pyang` in a Python environment:

```bash
pip install pyang
```

For example:

```bash
pyang -p ./models -f tree ./models/ietf-interfaces.yang
```

- `-p ./models`: Specifies the directory containing imported and included YANG modules.
- `-f tree`: Selects the tree output format.

A simplified model tree might appear as follows:

```text
module: ietf-interfaces
  +--rw interfaces
     +--rw interface* [name]
        +--rw name           string
        +--rw description?   string
        +--rw type           identityref
        +--rw enabled?       boolean
```

Common tree notation includes:

| Symbol | Meaning |
|---|---|
| `rw` | Configuration data |
| `ro` | Read-only state data |
| `*` | List or leaf-list |
| `?` | Optional node |
| `[name]` | List key |

Understanding the YANG tree helps identify the XML hierarchy required for NETCONF requests.

## Converting IOS XE Configuration to XML

Starting with Cisco IOS XE 17.7.1, IOS XE supports converting CLI configuration into structured NETCONF XML and RESTCONF JSON representations.

Display the running configuration in NETCONF XML format:

```text
R1# show running-config | format netconf-xml
```

Display the running configuration in RESTCONF JSON format:

```text
R1# show running-config | format restconf-json
```

These commands are particularly useful when constructing model-driven configuration requests.

Instead of manually examining a large YANG model, an administrator can:

1. Configure the desired feature using the CLI.
2. Display the configuration in NETCONF XML format.
3. Identify the relevant hierarchy and namespaces.
4. Use the appropriate XML elements in an RPC request.

For example, configure a loopback interface:

```text
R1(config)# interface Loopback0
R1(config-if)# description Router ID
R1(config-if)# ip address 1.1.1.1 255.255.255.255
```

Then display the structured configuration:

```text
R1# show running-config | format netconf-xml
```

The relevant portion of the XML representation resembles the following:

```xml
<native xmlns="http://cisco.com/ns/yang/Cisco-IOS-XE-native">
  <interface>
    <Loopback>
      <name>0</name>
      <description>Router ID</description>
      <ip>
        <address>
          <primary>
            <address>1.1.1.1</address>
            <mask>255.255.255.255</mask>
          </primary>
        </address>
      </ip>
    </Loopback>
  </interface>
</native>
```

The displayed XML can be used as a reference when constructing `<edit-config>` RPCs.

## Retrieving Configuration and Operational Data

NETCONF provides two primary data retrieval operations:

- `<get-config>`
- `<get>`

The distinction between these operations is important.

| Operation | Configuration Data | State Data | Explicit Datastore Selection |
|---|---|---|---|
| `<get-config>` | Yes | No | Yes |
| `<get>` | Yes | Yes | No |

### The get-config Operation

The `<get-config>` operation retrieves configuration data from a specified configuration datastore.

For example, retrieve the entire running configuration:

```xml
<rpc message-id="201"
     xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <get-config>
    <source>
      <running/>
    </source>
  </get-config>
</rpc>
```

The `<source>` element identifies the datastore to retrieve.

Depending on the server's capabilities, possible sources include:

- `<running/>`
- `<candidate/>`
- `<startup/>`

Without a filter, the server can return a large amount of configuration data.

### The get Operation

The `<get>` operation retrieves running configuration and state data.

For example:

```xml
<rpc message-id="202"
     xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <get/>
</rpc>
```

Unlike `<get-config>`, the `<get>` operation does not contain a `<source>` element.

It can return information that is unavailable through `<get-config>`, such as:

- Interface counters.
- Operational interface states.
- Learned routing information.
- Neighbor relationships.
- Protocol statistics.

Because the amount of returned information can be very large, filters should normally be used.

## NETCONF Filtering

NETCONF supports two primary filtering mechanisms:

- Subtree filtering.
- XPath filtering.

Filtering allows the client to retrieve specific data instead of retrieving an entire datastore.

### Subtree Filtering

Subtree filtering selects data based on an XML hierarchy.

It is supported by the base NETCONF protocol.

For example, retrieve only R1's hostname:

```xml
<rpc message-id="203"
     xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <get-config>
    <source>
      <running/>
    </source>
    <filter type="subtree">
      <native xmlns="http://cisco.com/ns/yang/Cisco-IOS-XE-native">
        <hostname/>
      </native>
    </filter>
  </get-config>
</rpc>
```

The response resembles the following:

```xml
<rpc-reply message-id="203"
           xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <data>
    <native xmlns="http://cisco.com/ns/yang/Cisco-IOS-XE-native">
      <hostname>R1</hostname>
    </native>
  </data>
</rpc-reply>
```

Only the requested hostname and its parent hierarchy are returned.

### Subtree Filtering Components

Subtree filters can contain several types of elements.

| Element Type | Purpose |
|---|---|
| Containment node | Identifies a location in the data hierarchy |
| Selection node | Selects a node and its associated subtree |
| Content match node | Matches a specific data value |

For example:

```xml
<filter type="subtree">
  <native xmlns="http://cisco.com/ns/yang/Cisco-IOS-XE-native">
    <interface>
      <Loopback>
        <name>0</name>
        <description/>
      </Loopback>
    </interface>
  </native>
</filter>
```

In this filter:

- `<native>`, `<interface>`, and `<Loopback>` identify the hierarchy.
- `<name>0</name>` matches Loopback0.
- `<description/>` selects the description.

The `<name>` element is a content match node because it contains a value.

The empty `<description/>` element is a selection node.

### Retrieving a Specific Interface Configuration

Retrieve the configuration of Loopback0:

```xml
<rpc message-id="204"
     xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <get-config>
    <source>
      <running/>
    </source>
    <filter type="subtree">
      <native xmlns="http://cisco.com/ns/yang/Cisco-IOS-XE-native">
        <interface>
          <Loopback>
            <name>0</name>
          </Loopback>
        </interface>
      </native>
    </filter>
  </get-config>
</rpc>
```

The `<name>0</name>` element matches the list entry representing Loopback0.

Because no sibling selection nodes are specified, the matching interface subtree is returned.

### Retrieving Operational Data

The following example uses the Cisco interface operational YANG model to retrieve operational information about GigabitEthernet1.

```xml
<rpc message-id="205"
     xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <get>
    <filter type="subtree">
      <interfaces xmlns="http://cisco.com/ns/yang/Cisco-IOS-XE-interfaces-oper">
        <interface>
          <name>GigabitEthernet1</name>
        </interface>
      </interfaces>
    </filter>
  </get>
</rpc>
```

The response can include information such as:

- Administrative state.
- Operational state.
- Interface type.
- Physical address.
- Speed.
- Interface statistics.

The `<get>` operation is used because the requested information includes read-only operational data.

### XPath Filtering

**XPath (XML Path Language)** provides expressions for selecting nodes from an XML data hierarchy.

NETCONF XPath filtering requires the server to advertise the `:xpath` capability.

For example, retrieve the configured hostname:

```xml
<rpc message-id="206"
     xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <get-config>
    <source>
      <running/>
    </source>
    <filter type="xpath"
            xmlns:ios="http://cisco.com/ns/yang/Cisco-IOS-XE-native"
            select="/ios:native/ios:hostname"/>
  </get-config>
</rpc>
```

The `select` attribute specifies the XPath expression.

An XPath filter can also use predicates to select specific list entries.

For example:

```xml
<filter type="xpath"
        xmlns:ios="http://cisco.com/ns/yang/Cisco-IOS-XE-native"
        select="/ios:native/ios:interface/ios:Loopback[ios:name='0']/ios:description"/>
```

The predicate selects the Loopback interface whose name is `0`.

### Subtree and XPath Comparison

| Characteristic | Subtree | XPath |
|---|---|---|
| Selection mechanism | XML hierarchy | XPath expression |
| Base NETCONF support | Yes | No |
| Required optional capability | None | `:xpath` |
| Syntax | Nested XML elements | Path expressions |
| Value matching | Content match nodes | Predicates |

Both filtering methods can be used with `<get>` and `<get-config>`.

## Modifying Configuration

The `<edit-config>` operation modifies a configuration datastore.

An `<edit-config>` request typically contains the following elements:

| Element | Purpose |
|---|---|
| `<target>` | Identifies the datastore to modify |
| `<default-operation>` | Specifies the default modification behavior |
| `<test-option>` | Controls configuration testing behavior |
| `<error-option>` | Controls behavior when an error occurs |
| `<config>` | Contains the configuration data |

Not all optional elements are required in every request.

Some options depend on advertised capabilities.

### Modifying an Interface Description

The following RPC configures the description of Loopback0.

```xml
<rpc message-id="301"
     xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <edit-config>
    <target>
      <running/>
    </target>
    <config>
      <native xmlns="http://cisco.com/ns/yang/Cisco-IOS-XE-native">
        <interface>
          <Loopback>
            <name>0</name>
            <description>Configured via NETCONF</description>
          </Loopback>
        </interface>
      </native>
    </config>
  </edit-config>
</rpc>
```

If the operation succeeds, the server returns:

```xml
<rpc-reply message-id="301"
           xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <ok/>
</rpc-reply>
```

Verify the configuration through the CLI:

```text
R1# show running-config interface Loopback0
```

The equivalent CLI configuration is:

```text
R1(config)# interface Loopback0
R1(config-if)# description Configured via NETCONF
```

Because the RPC targets the running datastore, the change takes effect immediately.

This example assumes the server advertises the `:writable-running` capability.

### Creating an Interface

The following RPC creates Loopback100 with an IPv4 address.

```xml
<rpc message-id="302"
     xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <edit-config>
    <target>
      <running/>
    </target>
    <config>
      <native xmlns="http://cisco.com/ns/yang/Cisco-IOS-XE-native">
        <interface>
          <Loopback>
            <name>100</name>
            <description>NETCONF Lab</description>
            <ip>
              <address>
                <primary>
                  <address>100.100.100.100</address>
                  <mask>255.255.255.255</mask>
                </primary>
              </address>
            </ip>
          </Loopback>
        </interface>
      </native>
    </config>
  </edit-config>
</rpc>
```

Verify the configuration:

```text
R1# show running-config interface Loopback100
R1# show ip interface brief
```

The equivalent CLI configuration is:

```text
R1(config)# interface Loopback100
R1(config-if)# description NETCONF Lab
R1(config-if)# ip address 100.100.100.100 255.255.255.255
```

## NETCONF Edit Operations

NETCONF defines five operations that can be applied to configuration elements within an `<edit-config>` request.

| Operation | Behavior |
|---|---|
| `merge` | Merge the supplied data with existing configuration |
| `replace` | Replace the targeted configuration with the supplied data |
| `create` | Create data only if it does not already exist |
| `delete` | Delete existing data; return an error if it does not exist |
| `remove` | Delete existing data; do not return an error if it does not exist |

These operations are specified using an XML attribute named `operation` in the NETCONF namespace.

### Default Operation

The `<default-operation>` element specifies the default operation applied to configuration data.

It supports three values:

| Value | Behavior |
|---|---|
| `merge` | Merge supplied configuration with existing configuration |
| `replace` | Replace corresponding configuration with supplied configuration |
| `none` | Do not implicitly modify configuration without an explicit operation attribute |

**The default operation is `merge`.**

For example:

```xml
<edit-config>
  <target>
    <running/>
  </target>
  <default-operation>merge</default-operation>
  <config>
    <!-- Configuration data -->
  </config>
</edit-config>
```

An explicit `operation` attribute overrides the default operation for the associated configuration element.

### Merge Operation

The `merge` operation combines the supplied configuration with existing configuration.

For example, assume Loopback100 has the following configuration:

```text
interface Loopback100
 description NETCONF Lab
 ip address 100.100.100.100 255.255.255.255
```

The following configuration data changes the description without removing the IP address:

```xml
<native xmlns="http://cisco.com/ns/yang/Cisco-IOS-XE-native">
  <interface>
    <Loopback>
      <name>100</name>
      <description>New Description</description>
    </Loopback>
  </interface>
</native>
```

The resulting configuration is:

```text
interface Loopback100
 description New Description
 ip address 100.100.100.100 255.255.255.255
```

Existing configuration that is not included in the request is preserved.

### Replace Operation

The `replace` operation replaces the targeted configuration subtree with the supplied data.

For example:

```xml
<rpc message-id="303"
     xmlns="urn:ietf:params:xml:ns:netconf:base:1.0"
     xmlns:nc="urn:ietf:params:xml:ns:netconf:base:1.0">
  <edit-config>
    <target>
      <running/>
    </target>
    <config>
      <native xmlns="http://cisco.com/ns/yang/Cisco-IOS-XE-native">
        <interface>
          <Loopback nc:operation="replace">
            <name>100</name>
            <description>Replacement Configuration</description>
          </Loopback>
        </interface>
      </native>
    </config>
  </edit-config>
</rpc>
```

Unlike a merge operation, a replace operation does not preserve omitted configuration elements within the replaced subtree.

In this example, the previous IPv4 address is not included in the replacement data.

Therefore, it is removed from the interface configuration, assuming the model and resulting configuration permit the operation.

Be careful when applying `replace` to large configuration subtrees.

### Create Operation

The `create` operation creates a configuration element only if it does not already exist.

For example:

```xml
<rpc message-id="304"
     xmlns="urn:ietf:params:xml:ns:netconf:base:1.0"
     xmlns:nc="urn:ietf:params:xml:ns:netconf:base:1.0">
  <edit-config>
    <target>
      <running/>
    </target>
    <config>
      <native xmlns="http://cisco.com/ns/yang/Cisco-IOS-XE-native">
        <interface>
          <Loopback nc:operation="create">
            <name>200</name>
            <description>Created via NETCONF</description>
          </Loopback>
        </interface>
      </native>
    </config>
  </edit-config>
</rpc>
```

If Loopback200 already exists, the server returns a `data-exists` error.

This differs from `merge`, which can create a nonexistent element or modify an existing element.

### Delete Operation

The `delete` operation removes an existing configuration element.

For example, delete Loopback200:

```xml
<rpc message-id="305"
     xmlns="urn:ietf:params:xml:ns:netconf:base:1.0"
     xmlns:nc="urn:ietf:params:xml:ns:netconf:base:1.0">
  <edit-config>
    <target>
      <running/>
    </target>
    <default-operation>none</default-operation>
    <config>
      <native xmlns="http://cisco.com/ns/yang/Cisco-IOS-XE-native">
        <interface>
          <Loopback nc:operation="delete">
            <name>200</name>
          </Loopback>
        </interface>
      </native>
    </config>
  </edit-config>
</rpc>
```

The equivalent CLI command is:

```text
R1(config)# no interface Loopback200
```

If Loopback200 does not exist, the server returns a `data-missing` error.

Using `<default-operation>none</default-operation>` avoids implicitly creating or merging parent configuration elements during the delete operation.

### Remove Operation

The `remove` operation is similar to `delete`.

However, it does not return an error merely because the requested element does not exist.

For example:

```xml
<Loopback nc:operation="remove">
  <name>200</name>
</Loopback>
```

The `remove` operation is useful when an application wants to ensure that a configuration element is absent, regardless of its initial state.

### Edit Operation Comparison

| Operation | If Element Exists | If Element Does Not Exist |
|---|---|---|
| `merge` | Merge configuration | Create element |
| `replace` | Replace configuration | Create element |
| `create` | Return `data-exists` error | Create element |
| `delete` | Delete element | Return `data-missing` error |
| `remove` | Delete element | No error due to absence |

These are the standardized NETCONF operation semantics. Individual YANG constraints and platform restrictions can still cause an operation to fail.

### Deleting an Individual Configuration Parameter

An operation attribute can be applied to an individual leaf rather than an entire list entry.

For example, remove the description from Loopback100 without deleting the interface:

```xml
<rpc message-id="306"
     xmlns="urn:ietf:params:xml:ns:netconf:base:1.0"
     xmlns:nc="urn:ietf:params:xml:ns:netconf:base:1.0">
  <edit-config>
    <target>
      <running/>
    </target>
    <default-operation>none</default-operation>
    <config>
      <native xmlns="http://cisco.com/ns/yang/Cisco-IOS-XE-native">
        <interface>
          <Loopback>
            <name>100</name>
            <description nc:operation="delete"/>
          </Loopback>
        </interface>
      </native>
    </config>
  </edit-config>
</rpc>
```

The equivalent CLI configuration is:

```text
R1(config)# interface Loopback100
R1(config-if)# no description
```

The location of the `operation` attribute determines the scope of the operation.

## Error Handling and Validation

### Error Options

The `<error-option>` element determines how the server handles errors during an `<edit-config>` operation.

NETCONF defines three error options.

| Error Option | Behavior |
|---|---|
| `stop-on-error` | Stop processing when an error occurs |
| `continue-on-error` | Continue processing despite errors |
| `rollback-on-error` | Restore the configuration to its state before the failed edit |

**The default error option is `stop-on-error`.**

Stopping on an error does not necessarily mean that all previously applied changes are reverted.

The `rollback-on-error` option requires the `:rollback-on-error` capability.

For example:

```xml
<edit-config>
  <target>
    <running/>
  </target>
  <error-option>rollback-on-error</error-option>
  <config>
    <!-- Configuration data -->
  </config>
</edit-config>
```

Check the server's advertised capabilities before using this option.

### Test Options

The `<test-option>` element controls validation behavior during an `<edit-config>` operation.

| Test Option | Behavior |
|---|---|
| `test-then-set` | Validate the configuration before applying it |
| `set` | Apply the configuration without a separate preliminary validation phase |
| `test-only` | Validate the configuration without applying it |

The default test option is `test-then-set` when the applicable validation capability is supported.

Support for the test options depends on the server's advertised validation capability.

The `test-only` option requires `:validate:1.1`.

### The validate Operation

The `<validate>` operation validates configuration data without modifying the running configuration.

For example, validate the candidate configuration:

```xml
<rpc message-id="307"
     xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <validate>
    <source>
      <candidate/>
    </source>
  </validate>
</rpc>
```

The server returns `<ok/>` if the configuration passes validation.

If validation fails, the server returns an `<rpc-error>`.

Validation can identify problems such as:

- Invalid data types.
- Missing mandatory elements.
- Invalid references.
- Violations of YANG constraints.

Successful validation does not guarantee that the resulting configuration will provide the intended network behavior.

## Configuration Datastore Locking

Multiple NETCONF clients can establish sessions with the same device.

Without coordination, different clients could modify the same configuration simultaneously.

NETCONF provides the `<lock>` and `<unlock>` operations to control concurrent configuration access.

### Locking the Running Datastore

The following RPC locks the running datastore:

```xml
<rpc message-id="401"
     xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <lock>
    <target>
      <running/>
    </target>
  </lock>
</rpc>
```

If the lock succeeds, the server returns `<ok/>`.

The NETCONF session that acquired the lock can modify the datastore.

Other sessions cannot modify the locked datastore until the lock is released.

On IOS XE, locking the running datastore also prevents conflicting configuration changes through other management mechanisms, including the CLI.

### Verifying Datastore Locks

Use the following commands:

```text
R1# show netconf-yang sessions
R1# show netconf-yang datastores
```

The session information identifies the NETCONF session holding the lock.

Common global lock indicators include:

| Indicator | Meaning |
|---|---|
| `R` | Running datastore lock |
| `C` | Candidate datastore lock |
| `S` | Startup datastore lock |
| `None` | No datastore lock |

### Unlocking a Datastore

Release the running datastore lock:

```xml
<rpc message-id="402"
     xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <unlock>
    <target>
      <running/>
    </target>
  </unlock>
</rpc>
```

Normally, only the session that acquired the lock can release it using `<unlock>`.

Datastore locks are also released when the owning NETCONF session terminates.

### Locking Workflow

A common direct configuration workflow is:

```text
         NETCONF Client
                |
                | 1. Lock running
                v
       +------------------+
       | Running Datastore|
       +------------------+
                ^
                | 2. Edit configuration
                |
                | 3. Verify configuration
                |
                | 4. Unlock running
                |
         NETCONF Client
```

Locking a datastore does not create a separate staging area.

When the running datastore is modified, the changes still take effect immediately.

## Candidate Configuration Datastore

By default, Cisco IOS XE allows NETCONF clients to modify the running configuration directly.

The **candidate datastore** provides an alternative configuration workflow in which changes are staged before being committed.

Enable the candidate datastore:

```text
R1(config)# netconf-yang feature candidate-datastore
```

Disable the candidate datastore:

```text
R1(config)# no netconf-yang feature candidate-datastore
```

Changing this setting reinitializes the model-driven management services and terminates existing NETCONF and RESTCONF sessions.

### Candidate Datastore and Writable Running

The NETCONF standard permits servers to support both the candidate and writable-running capabilities simultaneously.

However, Cisco IOS XE uses mutually exclusive configuration modes.

| IOS XE Configuration Mode | Writable Running | Candidate |
|---|---|---|
| Default | Yes | No |
| Candidate enabled | No | Yes |

**When the candidate datastore is enabled, the running datastore cannot be modified directly through NETCONF.**

NETCONF configuration changes must be applied to the candidate datastore and subsequently committed.

This restriction does not mean that the IOS XE CLI can no longer modify the running configuration.

### Candidate Datastore Workflow

```text
NETCONF Client
      |
      | edit-config
      v
+-----------------------+
|  Candidate Datastore  |
+-----------------------+
      |
      | commit
      v
+-----------------------+
|   Running Datastore   |
+-----------------------+
      |
      v
 Active Configuration
```

The candidate datastore is shared between NETCONF sessions.

It is not a private configuration workspace for each client.

Therefore, an administrator should lock the candidate datastore before modifying it.

### Locking the Candidate Datastore

```xml
<rpc message-id="501"
     xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <lock>
    <target>
      <candidate/>
    </target>
  </lock>
</rpc>
```

This prevents other NETCONF sessions from modifying the candidate datastore.

Locking the candidate datastore does not automatically lock the running datastore.

An application can lock both datastores when necessary to prevent conflicting configuration changes.

### Modifying the Candidate Datastore

The following RPC stages a description change for Loopback0.

```xml
<rpc message-id="502"
     xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <edit-config>
    <target>
      <candidate/>
    </target>
    <config>
      <native xmlns="http://cisco.com/ns/yang/Cisco-IOS-XE-native">
        <interface>
          <Loopback>
            <name>0</name>
            <description>Candidate Configuration</description>
          </Loopback>
        </interface>
      </native>
    </config>
  </edit-config>
</rpc>
```

The running configuration is not modified by this operation.

### Retrieving the Candidate Configuration

```xml
<rpc message-id="503"
     xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <get-config>
    <source>
      <candidate/>
    </source>
    <filter type="subtree">
      <native xmlns="http://cisco.com/ns/yang/Cisco-IOS-XE-native">
        <interface>
          <Loopback>
            <name>0</name>
          </Loopback>
        </interface>
      </native>
    </filter>
  </get-config>
</rpc>
```

Compare the result with the same request targeting the running datastore.

The candidate configuration contains the staged description change, while the running configuration retains its previous description.

### Committing the Candidate Configuration

Apply the candidate configuration to the running datastore using `<commit>`.

```xml
<rpc message-id="504"
     xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <commit/>
</rpc>
```

If successful, the server returns `<ok/>`.

The staged configuration becomes the active running configuration.

Verify the change:

```text
R1# show running-config interface Loopback0
```

A commit operation applies the candidate configuration to the running configuration.

**It does not necessarily save the running configuration to the startup configuration.**

### Discarding Candidate Configuration Changes

Uncommitted candidate configuration changes can be discarded using `<discard-changes>`.

```xml
<rpc message-id="505"
     xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <discard-changes/>
</rpc>
```

This operation resets the candidate configuration to the current running configuration.

It does not modify the running configuration.

It also does not undo changes that have already been committed.

### Unlocking the Candidate Datastore

```xml
<rpc message-id="506"
     xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <unlock>
    <target>
      <candidate/>
    </target>
  </unlock>
</rpc>
```

A typical candidate configuration workflow is:

1. Lock the candidate datastore.
2. Lock the running datastore if required.
3. Modify the candidate configuration.
4. Review the candidate configuration.
5. Validate the candidate configuration, if supported.
6. Commit or discard the candidate configuration.
7. Release the datastore locks.

## Confirmed Commit

A normal commit applies the candidate configuration to the running datastore.

A **confirmed commit** applies the candidate configuration temporarily and requires a subsequent confirmation.

If the commit is not confirmed within the specified timeout, the device automatically restores the previous configuration.

This feature requires the appropriate `:confirmed-commit` capability.

Confirmed commit is particularly useful when modifying configuration that could disrupt management connectivity.

For example:

- Management interface addressing.
- Routing configuration.
- Access control lists.
- AAA configuration.

### Confirmed Commit Workflow

```text
 Previous Running Configuration
                |
                | Confirmed Commit
                v
  Temporary Running Configuration
                |
                +-----------------------+
                |                       |
          Confirmation            Timer Expires
                |                       |
                v                       v
       Configuration Kept      Previous Configuration
                                      Restored
```

The default confirmation timeout is **600 seconds (10 minutes)**.

### Initiating a Confirmed Commit

The following RPC performs a confirmed commit with a confirmation timeout of 120 seconds.

```xml
<rpc message-id="601"
     xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <commit>
    <confirmed/>
    <confirm-timeout>120</confirm-timeout>
  </commit>
</rpc>
```

The candidate configuration becomes the active running configuration immediately.

However, the previous configuration is retained for possible rollback.

### Confirming the Commit

Before the timer expires, send a normal commit operation:

```xml
<rpc message-id="602"
     xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <commit/>
</rpc>
```

The automatic rollback is canceled, and the committed configuration remains active.

Saving the running configuration to the startup configuration is still a separate operation.

### Extending the Confirmation Timeout

A client can extend the confirmation period by issuing another confirmed commit before the current timer expires.

For example:

```xml
<rpc message-id="603"
     xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <commit>
    <confirmed/>
    <confirm-timeout>300</confirm-timeout>
  </commit>
</rpc>
```

### Canceling a Confirmed Commit

The `:confirmed-commit:1.1` capability defines the `<cancel-commit>` operation.

```xml
<rpc message-id="604"
     xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <cancel-commit/>
</rpc>
```

This operation cancels the outstanding confirmed commit and restores the previous configuration without waiting for the timeout to expire.

Support for this operation depends on the server's advertised capabilities.

### Persistent Confirmed Commit

The `:confirmed-commit:1.1` capability also defines persistent confirmed commits.

A normal nonpersistent confirmed commit is associated with the NETCONF session that initiated it.

If that session terminates before confirmation, the configuration is rolled back.

A persistent confirmed commit can survive termination of the originating NETCONF session.

For example:

```xml
<rpc message-id="605"
     xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <commit>
    <confirmed/>
    <confirm-timeout>120</confirm-timeout>
    <persist>LAB-TOKEN</persist>
  </commit>
</rpc>
```

A subsequent session can confirm the commit using the matching persistence identifier:

```xml
<rpc message-id="606"
     xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <commit>
    <persist-id>LAB-TOKEN</persist-id>
  </commit>
</rpc>
```

Persistent confirmed commit is a standardized NETCONF feature. Do not assume that a particular IOS XE release supports it without inspecting its advertised capabilities.

## Copying and Saving Configuration

### The copy-config Operation

The `<copy-config>` operation copies configuration data from a source to a target.

Unlike `<edit-config>`, which modifies configuration elements, `<copy-config>` replaces the target configuration with the source configuration.

For a server supporting the `:startup` capability, the following RPC copies running configuration to startup configuration:

```xml
<rpc message-id="701"
     xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <copy-config>
    <target>
      <startup/>
    </target>
    <source>
      <running/>
    </source>
  </copy-config>
</rpc>
```

The equivalent IOS XE CLI operation is:

```text
R1# copy running-config startup-config
```

The `<startup/>` datastore is available through standard NETCONF operations only if the server supports the `:startup` capability.

### The Cisco save-config RPC

Cisco IOS XE provides a `save-config` RPC defined by the `cisco-ia` YANG model.

```xml
<rpc message-id="702"
     xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <save-config xmlns="http://cisco.com/yang/cisco-ia"/>
</rpc>
```

This operation saves the running configuration to the startup configuration.

It is a Cisco-specific YANG-defined RPC rather than a base NETCONF protocol operation.

Verify the saved configuration using:

```text
R1# show startup-config
```

### The delete-config Operation

The `<delete-config>` operation deletes the contents of a configuration datastore, where permitted.

For example, on a server supporting a deletable startup datastore:

```xml
<rpc message-id="703"
     xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <delete-config>
    <target>
      <startup/>
    </target>
  </delete-config>
</rpc>
```

The running configuration datastore cannot be deleted using `<delete-config>`.

To remove individual configuration elements, use `<edit-config>` with the `delete` or `remove` operation.

## NETCONF Session Management

### Displaying Active Sessions

Use the following command to display active NETCONF sessions:

```text
R1# show netconf-yang sessions
```

The output identifies information such as:

- Session ID.
- Transport protocol.
- Username.
- Source IP address.
- Datastore locks.

For more detailed information:

```text
R1# show netconf-yang sessions detail
```

The detailed output also includes RPC counters, error counters, login time, and notification statistics.

### Closing a NETCONF Session

A client can gracefully terminate its own NETCONF session using `<close-session>`.

```xml
<rpc message-id="801"
     xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <close-session/>
</rpc>
```

The server returns `<ok/>` and closes the session.

Resources and datastore locks associated with the session are released.

### Terminating Another NETCONF Session

The `<kill-session>` operation allows an authorized client to terminate another NETCONF session.

For example:

```xml
<rpc message-id="802"
     xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <kill-session>
    <session-id>42</session-id>
  </kill-session>
</rpc>
```

This forcibly terminates NETCONF session 42.

The `<kill-session>` operation cannot be used to terminate the session from which the RPC itself is sent.

Use `<close-session>` to terminate the current session.

### Clearing Sessions Through the CLI

An administrator can also terminate NETCONF sessions using the IOS XE CLI.

For example:

```text
R1# clear netconf-yang session 42
```

This is useful when an unresponsive NETCONF client is holding a datastore lock.

Terminating the session releases its associated datastore locks.

A pending nonpersistent confirmed commit is rolled back if its owning session is terminated before confirmation.

## NETCONF Security

### NETCONF Access Control Lists

A service-level ACL can restrict the source addresses permitted to establish NETCONF connections.

For example, allow only the management workstation at 192.0.2.100:

```text
R1(config)# ip access-list standard NETCONF-CLIENTS
R1(config-std-nacl)# permit host 192.0.2.100
!
R1(config)# netconf-yang ssh ipv4 access-list name NETCONF-CLIENTS
```

The ACL is associated directly with the NETCONF SSH service.

This is distinct from applying an access class to the VTY lines.

A VTY access class is not a substitute for a NETCONF service-level ACL.

Remove the ACL association using:

```text
R1(config)# no netconf-yang ssh ipv4 access-list name NETCONF-CLIENTS
```

IPv6 NETCONF access can be restricted using the corresponding IPv6 service-level ACL configuration.

### Modifying the NETCONF TCP Port

The default NETCONF SSH port is TCP 830.

It can be modified using:

```text
R1(config)# netconf-yang ssh port 8300
```

The NETCONF client must specify the configured destination port.

For example:

```bash
ssh -p 8300 -s ADMIN@192.0.2.1 netconf
```

Restore the default port:

```text
R1(config)# no netconf-yang ssh port
```

Changing the NETCONF SSH port does not change the traditional SSH CLI port.

### SSH Public-Key Authentication

NETCONF can authenticate users using SSH public keys instead of passwords.

Cisco IOS XE uses the SSH public-key configuration associated with the local user.

The relevant configuration includes:

```text
R1(config)# ip ssh pubkey-chain
R1(conf-ssh-pubkey)# username ADMIN
R1(conf-ssh-pubkey-user)# key-string
```

Enter the client's public key in public-key data configuration mode.

For NETCONF public-key authentication, the traditional IOS SSH server must also be appropriately configured, and the user must receive appropriate authorization through the local authentication source.

The client's private key remains on the client.

SSH public-key configuration is covered in [SSH Authentication and Security Options](../ssh-scp/authentication.md).

## NETCONF Using Python

Although NETCONF messages can be exchanged manually using OpenSSH, dedicated client libraries simplify automation.

The Python `ncclient` library handles:

- SSH connections.
- NETCONF session establishment.
- Capability exchange.
- XML message framing.
- RPC construction.
- RPC replies and errors.

Install `ncclient` in a Python environment:

```bash
pip install ncclient
```

### Establishing a Connection

The following script establishes a NETCONF session with R1 and displays its advertised capabilities.

```python
from getpass import getpass
from ncclient import manager

with manager.connect(
    host="192.0.2.1",
    port=830,
    username="ADMIN",
    password=getpass("Password: "),
    hostkey_verify=True,
    device_params={"name": "iosxe"}
) as m:
    print("Session ID:", m.session_id)

    for capability in m.server_capabilities:
        print(capability)
```

The `with` statement ensures that the NETCONF session is closed when execution leaves the block.

The server's SSH host key must be trusted by the client.

In an isolated lab, `hostkey_verify=False` can be used to bypass SSH host-key verification. This should not be used in production.

### Retrieving Configuration

The following example retrieves R1's configured hostname.

```python
from getpass import getpass
from ncclient import manager

hostname_filter = """
<native xmlns="http://cisco.com/ns/yang/Cisco-IOS-XE-native">
    <hostname/>
</native>
"""

with manager.connect(
    host="192.0.2.1",
    port=830,
    username="ADMIN",
    password=getpass("Password: "),
    hostkey_verify=True,
    device_params={"name": "iosxe"}
) as m:
    reply = m.get_config(
        source="running",
        filter=("subtree", hostname_filter)
    )

    print(reply.data_xml)
```

The `get_config()` method generates a `<get-config>` RPC.

The `source` argument identifies the requested configuration datastore.

The `filter` argument specifies the filtering mechanism and filter content.

### Modifying Configuration

The following example changes the description of Loopback0.

```python
from getpass import getpass
from ncclient import manager

config = """
<config xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
    <native xmlns="http://cisco.com/ns/yang/Cisco-IOS-XE-native">
        <interface>
            <Loopback>
                <name>0</name>
                <description>Configured using ncclient</description>
            </Loopback>
        </interface>
    </native>
</config>
"""

with manager.connect(
    host="192.0.2.1",
    port=830,
    username="ADMIN",
    password=getpass("Password: "),
    hostkey_verify=True,
    device_params={"name": "iosxe"}
) as m:
    reply = m.edit_config(
        target="running",
        config=config
    )

    print(reply.xml)
```

The `edit_config()` method generates an `<edit-config>` RPC.

This example assumes the running datastore is writable.

### Using the Candidate Datastore

If the candidate datastore is enabled, the application must modify the candidate configuration and commit it.

For example, assuming `m` represents an established NETCONF connection and `config` contains the configuration data:

```python
with m.locked("candidate"):
    try:
        m.edit_config(
            target="candidate",
            config=config
        )

        m.commit()

    except Exception:
        m.discard_changes()
        raise
```

The `locked()` context manager acquires the datastore lock and releases it when execution leaves the block.

The `commit()` method generates a `<commit>` RPC.

The `discard_changes()` method generates a `<discard-changes>` RPC.

### Invoking a YANG-Defined RPC

The `dispatch()` method can invoke an RPC that does not have a dedicated `ncclient` method.

For example, save the running configuration:

```python
from ncclient.xml_ import to_ele

save_config = """
<save-config xmlns="http://cisco.com/yang/cisco-ia"/>
"""

reply = m.dispatch(to_ele(save_config))

print(reply.xml)
```

This invokes the Cisco-specific `save-config` RPC.

### Common ncclient Methods

| Python Method | NETCONF Operation |
|---|---|
| `get()` | `<get>` |
| `get_config()` | `<get-config>` |
| `edit_config()` | `<edit-config>` |
| `copy_config()` | `<copy-config>` |
| `delete_config()` | `<delete-config>` |
| `lock()` | `<lock>` |
| `unlock()` | `<unlock>` |
| `validate()` | `<validate>` |
| `commit()` | `<commit>` |
| `discard_changes()` | `<discard-changes>` |
| `close_session()` | `<close-session>` |
| `kill_session()` | `<kill-session>` |
| `get_schema()` | `<get-schema>` |
| `dispatch()` | Generic RPC invocation |

## NETCONF Error Messages

When an operation fails, the server returns an `<rpc-error>` element.

This element can include the following information:

| Element | Purpose |
|---|---|
| `<error-type>` | Identifies the layer associated with the error |
| `<error-tag>` | Identifies the standardized error condition |
| `<error-severity>` | Indicates the severity of the error |
| `<error-app-tag>` | Provides application-specific error identification |
| `<error-path>` | Identifies the data node associated with the error |
| `<error-message>` | Provides a human-readable explanation |
| `<error-info>` | Provides additional error information |

### Common Error Tags

| Error Tag | Possible Cause |
|---|---|
| `access-denied` | Insufficient authorization |
| `invalid-value` | Invalid configuration value |
| `unknown-element` | Unrecognized XML element |
| `unknown-namespace` | Incorrect or unsupported XML namespace |
| `missing-element` | Required XML element is missing |
| `missing-attribute` | Required XML attribute is missing |
| `data-exists` | A create operation targets existing data |
| `data-missing` | A delete operation targets nonexistent data |
| `lock-denied` | The requested datastore lock cannot be acquired |
| `in-use` | The requested resource is in use |
| `operation-not-supported` | The requested operation is unsupported |
| `operation-failed` | The operation failed |
| `malformed-message` | The NETCONF message is malformed |

A single RPC reply can contain multiple `<rpc-error>` elements.

Always examine the complete error information rather than relying exclusively on the error tag.

## Troubleshooting NETCONF

NETCONF troubleshooting can be divided into several layers.

### Transport Connectivity

Verify that the management workstation can reach the router.

```bash
ping 192.0.2.1
```

Test connectivity to TCP port 830:

```bash
nc -vz 192.0.2.1 830
```

If the TCP connection fails, verify:

- IP routing.
- Interface status.
- Management VRF configuration.
- Interface ACLs.
- NETCONF service-level ACLs.
- Control plane protection.
- The configured NETCONF port.

### SSH Connection

Use verbose SSH output to troubleshoot connection establishment.

```bash
ssh -vvv -p 830 -s ADMIN@192.0.2.1 netconf
```

This can help identify:

- SSH algorithm negotiation problems.
- Host-key verification problems.
- Authentication problems.
- Subsystem request failures.

### NETCONF Service Status

Verify that NETCONF is enabled:

```text
R1# show running-config | include netconf-yang
```

Verify that the model-driven management processes are running:

```text
R1# show platform software yang-management process
```

The `ncsshd`, `confd`, and other required processes must be operational.

### Authentication and Authorization

Verify the local user and AAA configuration:

```text
R1# show running-config | include username
R1# show running-config | section aaa
R1# show running-config | include yang-interfaces
```

A successful SSH CLI connection does not guarantee that NETCONF authentication and authorization will succeed.

Verify the appropriate AAA method lists and the privilege level assigned to the NETCONF user.

### Hello Message Exchange

If an SSH connection succeeds and the server sends a `<hello>` message but RPC operations do not work, verify:

- The client has sent a valid `<hello>` message.
- The client and server support a common NETCONF version.
- The correct message framing mechanism is being used.
- The complete XML message has been transmitted.

When manually using NETCONF 1.0, remember to append the `]]>]]>` delimiter to each message.

### YANG Model and RPC Errors

If a NETCONF session is established but operations fail, verify:

- The requested YANG model is supported.
- The XML namespaces are correct.
- The XML hierarchy matches the YANG model.
- Required list keys are included.
- Data values satisfy the model's constraints.
- The requested operation is supported.
- The target datastore is writable.
- The target datastore is not locked by another session.

A valid XML document is not necessarily a valid NETCONF request.

Similarly, a valid NETCONF request can contain invalid YANG-modeled configuration data.

### Session and Datastore Information

Use the following commands:

```text
R1# show netconf-yang sessions
R1# show netconf-yang sessions detail
R1# show netconf-yang datastores
R1# show netconf-yang statistics
```

These commands can identify:

- Active sessions.
- RPC requests and errors.
- Invalid hello messages.
- Dropped sessions.
- Datastore lock owners.

### Common Problems

| Problem | Possible Cause |
|---|---|
| TCP connection refused | NETCONF is disabled or the wrong TCP port is used |
| TCP connection times out | Routing, ACL, or firewall problem |
| SSH CLI works, but NETCONF fails | Separate NETCONF service or authentication problem |
| Authentication succeeds, but session fails | Insufficient authorization or privilege level |
| Server sends hello but does not respond to RPCs | Missing client hello or incorrect message framing |
| Retrieval returns empty data | Incorrect filter, namespace, or nonexistent data |
| `<get-config>` does not return interface counters | Counters are operational data; use `<get>` |
| `<edit-config>` targeting running fails | Candidate datastore is enabled or running is locked |
| Candidate changes do not appear in the CLI | The candidate configuration has not been committed |
| Commit fails | Invalid candidate configuration or conflicting datastore lock |
| Configuration disappears after a reboot | Running configuration was not saved |
| Configuration is automatically reverted | Confirmed commit was not confirmed |

## Practice Exercises

Use the lab topology at the beginning of this section to perform the following exercises.

### Exercise 1: Basic NETCONF Configuration

1. Configure a local user with privilege level 15.
2. Enable NETCONF.
3. Verify the model-driven management processes.
4. Establish a NETCONF session using OpenSSH.
5. Examine the server's advertised capabilities.
6. Send a client `<hello>` message advertising NETCONF 1.0.
7. Verify the session through the IOS XE CLI.

### Exercise 2: Retrieving Data

Use manually constructed RPC messages to retrieve:

1. The configured hostname.
2. The running configuration of Loopback0.
3. The descriptions of all configured loopback interfaces.
4. The operational information of GigabitEthernet1.
5. The hostname using an XPath filter.

### Exercise 3: Modifying Configuration

Use `<edit-config>` operations to:

1. Create Loopback100.
2. Assign an IPv4 address to Loopback100.
3. Configure a description without removing the IPv4 address.
4. Remove only the description.
5. Replace the interface configuration.
6. Delete Loopback100.
7. Attempt to delete Loopback100 again and examine the error.
8. Use `remove` instead of `delete` and compare the results.

### Exercise 4: Datastore Locking

1. Establish two NETCONF sessions.
2. Lock the running datastore using the first session.
3. Attempt to modify the running datastore using the second session.
4. Attempt to enter configuration mode through the IOS XE CLI.
5. Examine the session and datastore information.
6. Release the lock.
7. Repeat the previously unsuccessful configuration operation.

### Exercise 5: Candidate Configuration

1. Enable the candidate datastore.
2. Reestablish the NETCONF session.
3. Examine the changes in advertised capabilities.
4. Attempt to modify the running datastore directly.
5. Modify the candidate datastore instead.
6. Retrieve and compare the running and candidate configurations.
7. Discard the staged changes.
8. Stage another configuration change.
9. Commit the configuration.
10. Verify the running configuration through the CLI.

### Exercise 6: Confirmed Commit

1. Stage a configuration change in the candidate datastore.
2. Perform a confirmed commit with a 30-second timeout.
3. Verify that the running configuration has changed.
4. Allow the timeout to expire without confirming the commit.
5. Verify that the previous configuration has been restored.
6. Repeat the exercise, but confirm the commit before the timer expires.

### Exercise 7: Saving Configuration

1. Modify the running configuration using NETCONF.
2. Invoke the Cisco `save-config` RPC.
3. Verify that the startup configuration contains the changes.

### Exercise 8: NETCONF Security

1. Configure a NETCONF service-level ACL permitting the management workstation.
2. Verify that permitted sources can connect.
3. Verify that unauthorized sources cannot connect.
4. Change the NETCONF TCP port.
5. Establish a connection using the new TCP port.
6. Verify that traditional SSH CLI access is unaffected.

## Key Concepts

- NETCONF is enabled using `netconf-yang`.
- NETCONF over SSH uses TCP port 830 by default.
- NETCONF is separate from traditional SSH CLI access.
- NETCONF does not require enabling `aaa new-model`.
- If AAA is enabled, appropriate authentication and authorization must be configured.
- Named AAA method lists for NETCONF are supported starting with IOS XE 17.9.1.
- The NETCONF client and server exchange `<hello>` messages containing their capabilities.
- NETCONF 1.0 uses the `]]>]]>` end-of-message delimiter.
- NETCONF 1.1 uses chunked framing after the initial hello exchange.
- Both NETCONF versions use end-of-message framing for the initial hello messages.
- Each RPC request contains a `message-id` that is returned in the corresponding reply.
- XML namespaces distinguish NETCONF protocol elements from YANG model elements.
- `<get-config>` retrieves configuration data from a specified datastore.
- `<get>` retrieves running configuration and operational state data.
- Subtree filtering is supported by the base NETCONF protocol.
- XPath filtering requires the `:xpath` capability.
- `<edit-config>` modifies configuration data.
- The default configuration edit operation is `merge`.
- The `replace` operation replaces the targeted configuration subtree.
- The `create` operation fails if the targeted data already exists.
- The `delete` operation fails if the targeted data does not exist.
- The `remove` operation does not fail merely because the targeted data does not exist.
- The default error option is `stop-on-error`.
- `<lock>` and `<unlock>` control configuration datastore access.
- The candidate datastore is enabled using `netconf-yang feature candidate-datastore`.
- Enabling the candidate datastore on IOS XE disables direct NETCONF writes to the running datastore.
- The candidate datastore is shared between NETCONF sessions.
- `<commit>` applies the candidate configuration to the running datastore.
- `<discard-changes>` resets the candidate configuration to the running configuration.
- Confirmed commit automatically rolls back unconfirmed configuration changes.
- The default confirmed commit timeout is 600 seconds.
- Committing configuration and saving configuration are separate operations.
- The Cisco-specific `save-config` RPC saves the running configuration to the startup configuration.
- `<close-session>` terminates the current NETCONF session.
- `<kill-session>` terminates another NETCONF session.
- NETCONF service-level ACLs restrict access independently of VTY access classes.
- `show running-config | format netconf-xml` displays IOS XE configuration in NETCONF XML format.
- `ncclient` is a Python library that simplifies NETCONF communication.

## Further Reading

- [RESTCONF Configuration and Operations](restconf.md)
- [SSH Configuration and Verification](../ssh-scp/ssh.md)
- [SSH Authentication and Security Options](../ssh-scp/authentication.md)

### Cisco Documentation

- [NETCONF Protocol, Cisco IOS XE 17.15.x](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/prog/configuration/1715/b_1715_programmability_cg/netconf_protocol.html)
- [NETCONF and RESTCONF Service-Level ACLs, Cisco IOS XE 17.14.x](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/prog/configuration/1714/b_1714_programmability_cg/m_1714_prog_service-level_acls.html)
- [Programmability Command Reference, Cisco IOS XE 17.9.x](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/prog/command/179/b_179_programmability_cr/programmability.html)
- [Cisco IOS XE YANG Models](https://github.com/YangModels/yang/tree/main/vendor/cisco/xe)

### RFCs

- [RFC 6241 - Network Configuration Protocol (NETCONF)](https://www.rfc-editor.org/rfc/rfc6241.html)
- [RFC 6242 - Using the NETCONF Protocol over Secure Shell (SSH)](https://www.rfc-editor.org/rfc/rfc6242.html)
- [RFC 6022 - YANG Module for NETCONF Monitoring](https://www.rfc-editor.org/rfc/rfc6022.html)
- [RFC 6243 - With-defaults Capability for NETCONF](https://www.rfc-editor.org/rfc/rfc6243.html)
- [RFC 5277 - NETCONF Event Notifications](https://www.rfc-editor.org/rfc/rfc5277.html)
- [RFC 7950 - The YANG 1.1 Data Modeling Language](https://www.rfc-editor.org/rfc/rfc7950.html)

### Tools

- [ncclient](https://github.com/ncclient/ncclient)
- [ncclient Documentation](https://ncclient.readthedocs.io/)
- [pyang](https://github.com/mbj4668/pyang)