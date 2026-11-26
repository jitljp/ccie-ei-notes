# 4.3.a (iii) RESTCONF, NETCONF

**RESTCONF** and **Network Configuration Protocol (NETCONF)** provide programmatic interfaces for managing network devices.

Unlike traditional CLI-based management, NETCONF and RESTCONF provide structured access to device configuration and operational data using YANG data models.

Both protocols can be used to:

- Retrieve device configuration.
- Retrieve operational state information.
- Modify device configuration.
- Invoke device operations.
- Integrate network devices with automation tools and management applications.

NETCONF uses XML-encoded messages exchanged over a persistent session, typically using SSH.

RESTCONF uses HTTP requests and responses exchanged over HTTPS, with data encoded using XML or JSON.

## Model-Driven Management

Traditional CLI-based management requires administrators or automation tools to interact with a device using platform-specific commands.

For example:

```text
R1# show ip interface brief
Interface              IP-Address      OK? Method Status    Protocol
GigabitEthernet1       192.0.2.1       YES manual up        up
GigabitEthernet2       198.51.100.1    YES manual up        up
Loopback0              1.1.1.1         YES manual up        up
```

Although this output is easy for a human to interpret, an automation application must parse the text to extract specific information.

Changes in command syntax or output formatting can cause automation scripts to fail.

**Model-driven management** provides an alternative by defining the structure of device data using standardized data modeling languages.

An application can request specific data elements and receive a structured response.

```text
                       Automation Application
                                 |
                     +-----------+-----------+
                     |                       |
                  NETCONF                 RESTCONF
                  SSH/830                HTTPS/443
                     |                       |
                     +-----------+-----------+
                                 |
                      Model-Driven Interfaces
                                 |
                         YANG Data Models
                                 |
                   Configuration and State Data
                                 |
                              IOS XE
```

NETCONF and RESTCONF provide different mechanisms for accessing the underlying modeled data.

A configuration change made using one protocol can be retrieved using the other protocol or viewed through the CLI.

## YANG Data Models

**YANG** is a data modeling language used to define the structure and characteristics of data exposed by network devices.

YANG was originally standardized in RFC 6020. YANG version 1.1 is standardized in RFC 7950.

A YANG model can define:

- Configuration parameters.
- Operational state information.
- Hierarchical relationships between data elements.
- Data types.
- Permitted values and constraints.
- Default values.
- Remote procedure calls (RPCs).
- Notifications.

YANG defines the **schema** of the data rather than the actual values configured on a device.

For example, a simplified YANG model might define an interface as having the following characteristics:

```text
interface
  +-- name          string
  +-- description   string
  +-- enabled       boolean
  +-- mtu           uint16
```

The actual interface name, description, administrative state, and MTU are instances of data conforming to the model.

### YANG, XML, and JSON

YANG should not be confused with XML or JSON.

| Technology | Purpose |
|---|---|
| YANG | Defines the structure and characteristics of data |
| XML | Encodes structured data |
| JSON | Encodes structured data |
| NETCONF | Provides protocol operations for accessing and manipulating modeled data |
| RESTCONF | Provides HTTP-based access to modeled data |

For example, an interface's configuration could be represented using XML:

```xml
<interface>
  <name>GigabitEthernet1</name>
  <description>WAN Link</description>
  <enabled>true</enabled>
  <mtu>1500</mtu>
</interface>
```

The equivalent data could be represented using JSON:

```json
{
  "interface": {
    "name": "GigabitEthernet1",
    "description": "WAN Link",
    "enabled": true,
    "mtu": 1500
  }
}
```

These are simplified examples illustrating the difference between XML and JSON encoding.

**YANG is a data modeling language, not a data encoding format or management protocol.**

### YANG Model Types

Cisco IOS XE supports multiple categories of YANG models.

| Model Type | Description | Example |
|---|---|---|
| Cisco native | Models specific to Cisco platforms and software | `Cisco-IOS-XE-native` |
| IETF | Standardized models developed by the IETF | `ietf-interfaces` |
| OpenConfig | Vendor-neutral models developed by the OpenConfig project | `openconfig-interfaces` |

Cisco native models provide access to Cisco-specific configuration and operational information.

IETF and OpenConfig models facilitate automation across devices from different vendors.

However, support for individual models and model elements depends on the platform and software version.

The same device configuration can be exposed through multiple YANG models.

For example, an interface might be accessible using:

- A Cisco native interface model.
- The `ietf-interfaces` model.
- The `openconfig-interfaces` model.

Each model can represent the interface using a different hierarchy and terminology.

### Configuration and Operational Data

YANG distinguishes between configuration data and state data.

| Data Type | Description | Examples |
|---|---|---|
| Configuration | Parameters that can be configured | IP address, interface description, administrative state |
| State | Information representing the current condition of the device | Operational interface state, counters, learned routes |

YANG uses the `config` statement to distinguish between configurable and read-only data.

- `config true`: Configuration data.
- `config false`: State data.

Operational information such as an interface's packet counters cannot be modified directly as configuration data.

## Configuration Datastores

A **datastore** is a collection of configuration or operational data maintained by a network device.

NETCONF defines several configuration datastores.

| Datastore | Description |
|---|---|
| Running | The configuration currently active on the device |
| Startup | The configuration loaded when the device starts |
| Candidate | A staging area for configuration changes that have not yet been committed |

The running datastore is mandatory in NETCONF. Other configuration datastores depend on the capabilities supported by the server.

### Running Datastore

The running datastore represents the device's active configuration.

When a client directly modifies a writable running datastore, successfully applied changes take effect without requiring a separate commit operation.

This is conceptually similar to entering configuration commands through the IOS XE CLI.

```text
NETCONF Client
      |
      | Modify running configuration
      v
+-----------------------+
|   Running Datastore   |
+-----------------------+
      |
      v
 Active Device Configuration
```

Direct modification of the running datastore requires support for the NETCONF `:writable-running` capability.

### Candidate Datastore

The candidate datastore provides a staging area for configuration changes.

A client can modify the candidate configuration without immediately affecting the device's active configuration.

The changes are subsequently applied to the running configuration using a commit operation.

```text
NETCONF Client
      |
      | Edit configuration
      v
+-----------------------+
|  Candidate Datastore  |
+-----------------------+
      |
      | Commit
      v
+-----------------------+
|   Running Datastore   |
+-----------------------+
      |
      v
 Active Device Configuration
```

Using a candidate datastore allows a client to:

- Stage multiple configuration changes.
- Review proposed changes.
- Validate the candidate configuration, if supported.
- Apply the changes using a commit operation.
- Discard uncommitted changes.

The candidate datastore is not enabled by default on Cisco IOS XE.

It can be enabled using:

```text
R1(config)# netconf-yang feature candidate-datastore
```

Changing the candidate datastore configuration restarts the model-driven management services and terminates existing NETCONF and RESTCONF sessions.

### Startup Datastore

The startup configuration determines the device's configuration when it boots.

Modifying the running configuration does not necessarily save the changes to the startup configuration.

**Committing a candidate configuration and saving the running configuration are distinct operations.**

The CLI equivalent of saving the running configuration is:

```text
R1# copy running-config startup-config
```

Cisco IOS XE also provides a YANG-defined `save-config` RPC in the `cisco-ia` model.

The availability of standard NETCONF startup datastore operations depends on the capabilities advertised by the server.

## NETCONF Overview

The **Network Configuration Protocol (NETCONF)** is standardized in RFC 6241.

NETCONF provides mechanisms for retrieving and modifying device configuration and retrieving operational information.

NETCONF uses XML-encoded messages and is typically transported over SSH using **TCP port 830**.

```text
NETCONF Client                      NETCONF Server
192.0.2.100                         192.0.2.1
      |                                   |
      |          SSH (TCP/830)            |
      | --------------------------------> |
      |                                   |
      | <====== NETCONF Messages =======> |
      |                                   |
```

Although NETCONF and interactive SSH CLI sessions both use SSH, they provide different management interfaces.

| Service | Default TCP Port | Interface |
|---|---|---|
| SSH CLI | 22 | Interactive CLI |
| NETCONF over SSH | 830 | XML-based programmatic interface |

Enabling traditional SSH CLI access does not automatically enable NETCONF.

### NETCONF Protocol Layers

NETCONF can be divided into four conceptual layers.

| Layer | Function |
|---|---|
| Content | Configuration and operational data |
| Operations | NETCONF operations such as `get-config` and `edit-config` |
| Messages | XML messages such as `rpc` and `rpc-reply` |
| Secure Transport | Secure delivery of NETCONF messages, typically using SSH |

```text
+------------------------------------+
| Content                            |
| Configuration and State Data       |
+------------------------------------+
| Operations                         |
| get, get-config, edit-config, etc. |
+------------------------------------+
| Messages                           |
| RPC, RPC Reply, Notification       |
+------------------------------------+
| Secure Transport                   |
| SSH                                |
+------------------------------------+
```

YANG models define the content accessed through NETCONF.

### NETCONF Session Establishment

A NETCONF session involves several stages:

1. The client establishes a secure transport connection to the server.
2. The client requests the NETCONF SSH subsystem.
3. The client and server exchange `<hello>` messages.
4. The client and server determine the supported NETCONF capabilities.
5. The client sends RPC requests to the server.
6. The server processes the requests and returns RPC replies.
7. The session is eventually terminated.

```text
NETCONF Client                      NETCONF Server
      |                                   |
      | ---- SSH Connection ------------> |
      |                                   |
      | ---- <hello> -------------------> |
      | <--- <hello> -------------------- |
      |                                   |
      | ---- <rpc> ---------------------> |
      | <--- <rpc-reply> ---------------- |
      |                                   |
      | ---- <rpc> ---------------------> |
      | <--- <rpc-reply> ---------------- |
      |                                   |
      | ---- Close Session RPC ---------> |
      | <--- <rpc-reply> ---------------- |
      |                                   |
```

A single NETCONF session can be used to perform multiple operations.

### NETCONF Capabilities

NETCONF capabilities identify protocol features supported by a client or server.

Capabilities are exchanged using `<hello>` messages when a NETCONF session is established.

Some important NETCONF capabilities include:

| Capability | Purpose |
|---|---|
| `:writable-running` | Allows direct modification of the running datastore |
| `:candidate` | Provides a candidate configuration datastore |
| `:confirmed-commit` | Supports commits that must be confirmed to remain active |
| `:startup` | Provides a separate startup configuration datastore |
| `:validate` | Supports configuration validation |
| `:rollback-on-error` | Supports rolling back changes if an edit operation fails |
| `:xpath` | Supports XPath-based filtering |

A client should inspect the capabilities advertised by the server before attempting operations that depend on optional features.

### NETCONF Remote Procedure Calls

NETCONF uses a **Remote Procedure Call (RPC)** mechanism.

A client sends an XML-encoded `<rpc>` message containing an operation to be performed.

The server processes the operation and returns an `<rpc-reply>` message.

Each RPC request contains a `message-id` attribute that allows the client to associate the reply with the original request.

For example, the following RPC retrieves the device's configured hostname using the Cisco native YANG model:

```xml
<rpc message-id="101"
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

The server could return the following reply:

```xml
<rpc-reply message-id="101"
           xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <data>
    <native xmlns="http://cisco.com/ns/yang/Cisco-IOS-XE-native">
      <hostname>R1</hostname>
    </native>
  </data>
</rpc-reply>
```

Notice that the request and reply contain the same `message-id`.

The response contains structured data rather than CLI command output.

### NETCONF Operations

NETCONF defines several standard operations.

| Operation | Purpose |
|---|---|
| `<get>` | Retrieve configuration and state data |
| `<get-config>` | Retrieve configuration data from a specified datastore |
| `<edit-config>` | Modify configuration data |
| `<copy-config>` | Copy configuration data to a target datastore |
| `<delete-config>` | Delete a configuration datastore, where permitted |
| `<lock>` | Lock a configuration datastore |
| `<unlock>` | Release a configuration datastore lock |
| `<commit>` | Apply the candidate configuration to the running datastore |
| `<discard-changes>` | Revert the candidate configuration to the running configuration |
| `<validate>` | Validate configuration data |
| `<close-session>` | Gracefully terminate the current NETCONF session |
| `<kill-session>` | Forcibly terminate another NETCONF session |

Some operations require specific server capabilities.

NETCONF also supports additional operations defined by YANG models.

### NETCONF Filtering

Without filtering, a NETCONF retrieval operation can return a large amount of device data.

NETCONF supports two primary filtering mechanisms:

| Filter Type | Description |
|---|---|
| Subtree | Selects data using an XML-based hierarchical structure |
| XPath | Selects data using XPath expressions |

Subtree filtering is part of the base NETCONF protocol.

XPath filtering requires the server to advertise the `:xpath` capability.

Filtering allows an application to retrieve only the data required for a particular task.

### NETCONF Configuration Overview

NETCONF is enabled on Cisco IOS XE using:

```text
R1(config)# netconf-yang
```

The device must also have appropriate authentication and authorization configured.

For example, a NETCONF client can initiate a connection from a workstation using OpenSSH:

```bash
ssh -p 830 -s ADMIN@192.0.2.1 netconf
```

- `-p 830`: Specifies the destination TCP port.
- `-s`: Requests an SSH subsystem instead of an interactive shell.
- `netconf`: Specifies the NETCONF subsystem.

Alternatively, applications can use NETCONF client libraries such as Python's `ncclient`.

NETCONF configuration, verification, capabilities, and operations are covered in detail in [NETCONF Configuration and Operations](netconf.md).

## RESTCONF Overview

**RESTCONF** is standardized in RFC 8040.

RESTCONF provides an HTTP-based interface for accessing YANG-modeled data using the datastore concepts defined by NETCONF.

RESTCONF uses HTTPS for secure communication, typically using **TCP port 443**.

```text
RESTCONF Client                     RESTCONF Server
192.0.2.100                         192.0.2.1
      |                                   |
      |         HTTPS (TCP/443)           |
      | --------------------------------> |
      |                                   |
      | ---- HTTP Request --------------> |
      | <--- HTTP Response -------------- |
      |                                   |
```

RESTCONF supports both XML and JSON data encoding.

Unlike NETCONF, RESTCONF does not use a persistent management session in which capabilities are negotiated using `<hello>` messages.

RESTCONF uses a **stateless request-response communication model**.

Each request contains the information needed to process that request.

Stateless operation does not mean that the underlying TCP or TLS connection must be closed after every request. Multiple HTTP requests can reuse an existing connection.

### RESTCONF Resources

RESTCONF exposes device data as resources identified using Uniform Resource Identifiers (URIs).

For example:

```text
https://192.0.2.1/restconf/data/Cisco-IOS-XE-native:native/hostname
```

This URI identifies the device's configured hostname using the Cisco native YANG model.

| URI Component | Description |
|---|---|
| `https://` | Secure HTTP transport |
| `192.0.2.1` | RESTCONF server address |
| `/restconf` | RESTCONF API root on Cisco IOS XE |
| `/data` | Data resource |
| `Cisco-IOS-XE-native:native` | Top-level data node and its YANG module |
| `/hostname` | Requested child data node |

RESTCONF resource paths follow the hierarchy defined by the applicable YANG model.

The URI identifies the resource, while the HTTP method determines the operation performed on the resource.

### RESTCONF HTTP Methods

RESTCONF uses HTTP methods to perform operations on resources.

These operations are commonly described as **CRUD** operations.

- Create
- Read
- Update
- Delete

| HTTP Method | Purpose | CRUD Operation |
|---|---|---|
| GET | Retrieve a resource | Read |
| HEAD | Retrieve response headers without the response body | Read |
| POST | Create a subordinate resource or invoke an operation | Create |
| PUT | Create or replace a resource | Create/Update |
| PATCH | Modify an existing resource | Update |
| DELETE | Delete a resource | Delete |
| OPTIONS | Discover supported HTTP methods | N/A |

The distinction between PUT and PATCH is important.

- **PUT:** Creates or replaces the target resource.
- **PATCH:** Applies changes to the target resource without necessarily replacing the entire resource.

POST can also invoke YANG-defined RPC operations and actions rather than create data resources.

### RESTCONF HTTP Headers

RESTCONF uses HTTP headers to indicate the encoding of transmitted data.

Two important headers are `Accept` and `Content-Type`.

| Header | Purpose |
|---|---|
| `Accept` | Specifies the response media type accepted by the client |
| `Content-Type` | Identifies the media type of the request or response body |

RESTCONF defines specific media types for YANG-modeled data.

| Encoding | Media Type |
|---|---|
| JSON | `application/yang-data+json` |
| XML | `application/yang-data+xml` |

For example:

```http
Accept: application/yang-data+json
```

This header requests that the server return YANG-modeled data encoded using JSON.

For a request containing a JSON-encoded body:

```http
Content-Type: application/yang-data+json
```

### RESTCONF Request and Response

The following HTTP request retrieves R1's configured hostname:

```http
GET /restconf/data/Cisco-IOS-XE-native:native/hostname HTTP/1.1
Host: 192.0.2.1
Accept: application/yang-data+json
```

Authentication headers are omitted from this example.

The server could return:

```http
HTTP/1.1 200 OK
Content-Type: application/yang-data+json

{
  "Cisco-IOS-XE-native:hostname": "R1"
}
```

Unlike NETCONF, RESTCONF does not require the client to construct an XML-encoded RPC message to retrieve the hostname.

Instead, the client specifies the desired resource using a URI and performs a GET request.

### RESTCONF HTTP Status Codes

RESTCONF uses HTTP status codes to indicate the result of an operation.

| Status Code | Meaning |
|---|---|
| 200 OK | Request succeeded |
| 201 Created | A new resource was created |
| 204 No Content | Request succeeded without a response body |
| 400 Bad Request | Invalid or malformed request |
| 401 Unauthorized | Missing or invalid authentication credentials |
| 403 Forbidden | The client is not permitted to perform the operation |
| 404 Not Found | The requested resource does not exist |
| 405 Method Not Allowed | The HTTP method is not supported for the resource |
| 409 Conflict | The request conflicts with the current resource or datastore state |
| 415 Unsupported Media Type | The request body uses an unsupported media type |
| 500 Internal Server Error | The server encountered an internal error |
| 503 Service Unavailable | The service is temporarily unavailable |

RESTCONF errors can also include structured error information in the HTTP response body.

### RESTCONF Configuration Overview

RESTCONF is enabled on Cisco IOS XE using:

```text
R1(config)# restconf
R1(config)# ip http secure-server
```

- `restconf` enables the RESTCONF interface.
- `ip http secure-server` enables the HTTPS server.

The device must also have appropriate authentication and authorization configured.

RESTCONF does not require the unencrypted HTTP server to be enabled.

RESTCONF can be accessed using various HTTP clients, including:

- curl.
- Postman.
- Python's `requests` library.

For example, the following curl command retrieves the hostname of R1:

```bash
curl -k -u ADMIN \
  -H "Accept: application/yang-data+json" \
  "https://192.0.2.1/restconf/data/Cisco-IOS-XE-native:native/hostname"
```

The `-u ADMIN` option specifies the username and prompts for the password.

The `-k` option disables TLS server certificate verification. This is convenient in isolated lab environments using self-signed certificates, but should not be used in production environments.

**NETCONF and RESTCONF can be enabled independently of each other.**

Enabling NETCONF using `netconf-yang` is not a prerequisite for enabling RESTCONF.

RESTCONF configuration, verification, resource paths, HTTP methods, and practical examples are covered in detail in [RESTCONF Configuration and Operations](restconf.md).

## NETCONF and RESTCONF Comparison

NETCONF and RESTCONF provide access to YANG-modeled device data, but differ in their transport, encoding, operation mechanisms, and configuration management capabilities.

| Characteristic | NETCONF | RESTCONF |
|---|---|---|
| Primary standard | RFC 6241 | RFC 8040 |
| Data modeling language | YANG | YANG |
| Data encoding | XML | XML or JSON |
| Typical secure transport | SSH | HTTPS |
| Default TCP port on IOS XE | 830 | 443 |
| Operation mechanism | RPCs | HTTP methods |
| Communication model | Stateful session | Stateless requests |
| Resource selection | XML subtree or XPath filters | URI paths and query parameters |
| Capability exchange | `<hello>` messages | Resource and YANG library discovery |
| Configuration datastore selection | Explicit selection where applicable | Managed by the server |
| Explicit datastore locking | Supported | Not supported |
| Explicit candidate configuration | Supported with appropriate capabilities | Not exposed through standard RESTCONF operations |
| Explicit commit operation | Supported with appropriate capabilities | Not exposed through standard RESTCONF operations |
| Confirmed commit | Supported with appropriate capabilities | Not exposed through standard RESTCONF operations |
| Error reporting | `<rpc-error>` messages | HTTP status codes and structured errors |
| IOS XE enablement | `netconf-yang` | `restconf` and `ip http secure-server` |
| Example client tools | OpenSSH, ncclient | curl, Postman, Python requests |

### Equivalent Operations

Many operations performed using NETCONF can also be performed using RESTCONF.

| Task | NETCONF | RESTCONF |
|---|---|---|
| Retrieve configuration | `<get-config>` | GET with `content=config` |
| Retrieve configuration and state data | `<get>` | GET |
| Create configuration | `<edit-config>` | POST, PUT, or PATCH |
| Modify configuration | `<edit-config>` | PUT or PATCH |
| Delete a configuration element | `<edit-config>` with a delete operation | DELETE |
| Invoke a YANG-defined RPC | `<rpc>` | POST to an operation resource |
| Lock a datastore | `<lock>` | No direct equivalent |
| Commit staged configuration | `<commit>` | No direct equivalent |

These operations are conceptually equivalent, although their exact behavior and supported options differ.

### Configuration Transactions

One of the primary differences between NETCONF and RESTCONF is their approach to configuration transactions.

NETCONF can provide the client with explicit control over configuration datastores and the configuration change process.

For example, a NETCONF client can:

1. Lock the applicable configuration datastores.
2. Modify the candidate configuration.
3. Validate the proposed configuration.
4. Commit the candidate configuration.
5. Unlock the datastores.

RESTCONF does not expose an equivalent multi-request candidate and commit workflow.

A RESTCONF client submits an individual configuration modification request, and the server manages the underlying datastore operations.

RESTCONF is not completely independent of NETCONF datastore mechanisms. For example, a datastore locked through NETCONF can prevent a RESTCONF client from modifying the configuration.

## NETCONF and RESTCONF Security

NETCONF and RESTCONF provide powerful interfaces that can expose sensitive device information and modify critical configuration.

Access to these interfaces must be appropriately secured.

### Transport Security

Both protocols use encrypted communication.

| Security Function | NETCONF over SSH | RESTCONF over HTTPS |
|---|---|---|
| Confidentiality | SSH encryption | TLS encryption |
| Integrity | SSH integrity protection | TLS integrity protection |
| Server authentication | SSH host key | TLS server certificate |
| Common client authentication | Username/password or SSH public key | HTTP authentication over TLS |

Server identity verification is important for preventing man-in-the-middle attacks.

### Authentication and Authorization

Authentication determines the identity of the connecting user.

Authorization determines the operations that the user is permitted to perform.

Cisco IOS XE integrates model-driven management interfaces with AAA.

Authentication and authorization can use local credentials or supported centralized AAA services.

However, NETCONF and RESTCONF are not interactive VTY sessions.

Their authentication, authorization, and access control configurations must be considered separately from traditional SSH CLI line configuration.

### Network Access Restrictions

Access to NETCONF and RESTCONF should be restricted to trusted management systems.

Possible protection mechanisms include:

- Dedicated management networks.
- Management VRFs.
- Interface ACLs.
- Protocol-specific access restrictions.
- Control plane protection.
- Appropriate AAA configuration.

Enabling NETCONF or RESTCONF introduces an additional management service that must be protected.

Services that are not required should be disabled.

## Verification and Troubleshooting

Cisco IOS XE implements NETCONF and RESTCONF using model-driven management software processes.

Their status can be verified using:

```text
R1# show platform software yang-management process
```

Several additional commands are useful for troubleshooting model-driven management.

| Command | Purpose |
|---|---|
| `show platform software yang-management process` | Display model-driven management process status |
| `show netconf-yang sessions` | Display NETCONF and RESTCONF session information |
| `show netconf-yang sessions detail` | Display detailed session information |
| `show netconf-yang datastores` | Display NETCONF datastore and lock information |
| `show netconf-yang statistics` | Display NETCONF statistics |
| `show ip http server status` | Display HTTP and HTTPS server status |

Although some commands include the `netconf-yang` keyword, they can also provide information relevant to RESTCONF because the two interfaces share underlying software components.

### Common Problems

| Problem | Possible Cause |
|---|---|
| NETCONF connection refused | NETCONF is disabled or TCP port 830 is blocked |
| RESTCONF connection refused | HTTPS is disabled or TCP port 443 is blocked |
| Authentication failure | Incorrect credentials or AAA configuration |
| Authorization failure | Insufficient privileges or access restrictions |
| SSH connection fails | SSH negotiation or host key verification problem |
| HTTPS connection fails | TLS negotiation or certificate verification problem |
| Requested data is unavailable | Unsupported YANG model or model element |
| NETCONF returns an RPC error | Invalid operation, XML, namespace, or data |
| RESTCONF returns HTTP 404 | Incorrect resource path or nonexistent resource |
| RESTCONF returns HTTP 415 | Incorrect `Content-Type` header |
| Configuration modification fails | Invalid data, datastore lock, or model constraint |
| Management interface is enabled but unavailable | Model-driven management processes are still initializing or have failed |

A useful troubleshooting sequence is:

1. Verify IP connectivity to the device.
2. Verify that the appropriate TCP port is reachable.
3. Verify that the required management interface is enabled.
4. Verify the status of the model-driven management processes.
5. Verify authentication and authorization.
6. Verify that the required YANG model is supported.
7. Verify the request syntax, namespaces, or resource paths.
8. Examine the returned RPC errors or HTTP status codes.
9. Check for configuration datastore locks.

## Key Concepts

- NETCONF and RESTCONF provide programmatic access to network device configuration and operational data.
- YANG defines the structure, data types, and constraints of modeled data.
- XML and JSON are data encoding formats, not data modeling languages.
- Cisco IOS XE supports Cisco native, IETF, and OpenConfig YANG models.
- NETCONF uses XML-encoded RPC messages, typically transported over SSH using TCP port 830.
- RESTCONF uses HTTP methods and XML or JSON encoding over HTTPS, typically using TCP port 443.
- NETCONF uses a stateful session, while RESTCONF uses stateless requests.
- NETCONF clients and servers exchange capabilities using `<hello>` messages.
- NETCONF `<get>` retrieves configuration and state data.
- NETCONF `<get-config>` retrieves configuration data from a specified datastore.
- NETCONF `<edit-config>` modifies configuration data.
- The candidate datastore allows configuration changes to be staged before being committed.
- Committing a candidate configuration does not necessarily save it to the startup configuration.
- NETCONF supports explicit datastore locking and candidate configuration workflows.
- RESTCONF identifies resources using URIs based on YANG model hierarchies.
- RESTCONF uses HTTP methods to determine which operations to perform on resources.
- RESTCONF uses HTTP status codes to communicate operation results.
- The `Accept` header specifies the desired response media type.
- The `Content-Type` header identifies the media type of a message body.
- RESTCONF does not provide explicit datastore locking or candidate/commit operations.
- NETCONF is enabled using `netconf-yang`.
- RESTCONF is enabled using `restconf` and `ip http secure-server`.
- NETCONF and RESTCONF can be enabled independently.
- The `show platform software yang-management process` command verifies the status of model-driven management processes.

## Further Reading

- [NETCONF Configuration and Operations](netconf.md)
- [RESTCONF Configuration and Operations](restconf.md)

### Cisco Documentation

- [NETCONF Protocol, Cisco IOS XE 17.15.x](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/prog/configuration/1715/b_1715_programmability_cg/m_1715_prog_yang_netconf.html)
- [RESTCONF Protocol, Cisco IOS XE 17.15.x](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/prog/configuration/1715/b_1715_programmability_cg/m_1715_prog_restconf.html)
- [Cisco YANG Models](https://github.com/YangModels/yang/tree/main/vendor/cisco/xe)

### RFCs

- [RFC 6241 - Network Configuration Protocol (NETCONF)](https://www.rfc-editor.org/rfc/rfc6241.html)
- [RFC 6242 - Using the NETCONF Protocol over Secure Shell (SSH)](https://www.rfc-editor.org/rfc/rfc6242.html)
- [RFC 7950 - The YANG 1.1 Data Modeling Language](https://www.rfc-editor.org/rfc/rfc7950.html)
- [RFC 7951 - JSON Encoding of Data Modeled with YANG](https://www.rfc-editor.org/rfc/rfc7951.html)
- [RFC 8040 - RESTCONF Protocol](https://www.rfc-editor.org/rfc/rfc8040.html)