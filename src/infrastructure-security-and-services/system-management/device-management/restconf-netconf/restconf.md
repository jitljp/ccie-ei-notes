# RESTCONF Configuration and Operations

**RESTCONF** provides an HTTPS-based interface for accessing and manipulating YANG-modeled configuration and operational data.

This section covers RESTCONF configuration on Cisco IOS XE, resource discovery, URI construction, HTTP operations, and troubleshooting. The fundamental differences between RESTCONF and NETCONF are covered in the [overview](overview.md).

## Lab Topology

The following topology uses the same addressing as the NETCONF lab.

```text
 Management Workstation
      192.0.2.100/24
             |
             | RESTCONF (TCP/443)
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

The management workstation must have IP connectivity to R1. The examples use `curl` from a Linux workstation, but the same requests can be sent using Postman or Python.

## Enabling RESTCONF

Enable RESTCONF and the HTTPS server:

```text
R1(config)# restconf
R1(config)# ip http secure-server
```

- `restconf` enables the RESTCONF interface.
- `ip http secure-server` enables secure HTTP services.

RESTCONF uses **TCP port 443** by default. Neither `netconf-yang` nor the unencrypted HTTP server (`ip http server`) is required.

Disable RESTCONF without disabling other HTTPS-based services:

```text
R1(config)# no restconf
```

### Basic Authentication Configuration

When `aaa new-model` is not configured, RESTCONF can use a locally configured user. Configure a privilege level 15 account for full administrative access under the default model-based authorization policy:

```text
R1(config)# username ADMIN privilege 15 secret ADMIN-PASSWORD
R1(config)# restconf
R1(config)# ip http secure-server
```

RESTCONF commonly uses **HTTP Basic authentication** over HTTPS. Basic authentication encodes the username and password using Base64, which is not encryption. TLS protects the credentials in transit.

RESTCONF does not use the VTY lines. Commands such as `login local` and `transport input ssh` under the VTY lines are not prerequisites.

### Authentication and Authorization with AAA

If AAA is enabled, configure the appropriate authentication and authorization method lists. For example, use the local database for both:

```text
R1(config)# username ADMIN privilege 15 secret ADMIN-PASSWORD
!
R1(config)# aaa new-model
R1(config)# aaa authentication login default local
R1(config)# aaa authorization exec default local
!
R1(config)# restconf
R1(config)# ip http secure-server
```

The login method list authenticates the user. The EXEC authorization method list provides authorization information, including the privilege level used to determine model-based access.

On releases supporting named model-driven AAA method lists, the lists can be specified explicitly:

```text
R1(config)# aaa authentication login YANG-AUTHN local
R1(config)# aaa authorization exec YANG-AUTHZ local
R1(config)# yang-interfaces aaa authentication method-list YANG-AUTHN
R1(config)# yang-interfaces aaa authorization method-list YANG-AUTHZ
```

These commands are shared with NETCONF, rather than being specific to RESTCONF. Changing them can affect both protocols.

### Model-Based Authorization

IOS XE implements the **NETCONF Access Control Model (NACM)** for NETCONF and RESTCONF.

By default, users with privilege level 15 have full access to modeled data and operations. Custom NACM rules can grant more granular permissions to other users and groups.

Traditional IOS CLI command authorization is not equivalent to model-based authorization. A user's ability to enter a CLI command does not, by itself, determine whether the equivalent RESTCONF operation is permitted.

## RESTCONF Verification

Verify the HTTPS server:

```text
R1# show ip http server secure status
R1# show ip http server status
```

Verify the model-driven management processes:

```text
R1# show platform software yang-management process
```

For example, a device with RESTCONF enabled but NETCONF disabled can display the following process states (abbreviated):

```text
confd            : Running
nesd             : Running
syncfd           : Running
ncsshd           : Not Running
dmiauthd         : Running
opdatamgrd       : Running
nginx            : Running
```

- `nginx`: HTTP server used by RESTCONF.
- `ncsshd`: NETCONF SSH server. It is not required for RESTCONF.
- `dmiauthd`: Model-driven interface authentication.
- `confd`, `nesd`, `syncfd`, and `opdatamgrd`: Processes involved in accessing and synchronizing modeled data.

The presence of the `nginx` process alone does not prove that RESTCONF is enabled. It can be running because of other HTTP services.

Additional commands:

```text
R1# show platform software yang-management process monitor
R1# show netconf-yang sessions
R1# show netconf-yang sessions detail
R1# show netconf-yang datastores
```

The `show netconf-yang sessions` commands can display RESTCONF session information as well as NETCONF sessions. Because RESTCONF uses individual HTTP requests, there might not be a long-lived session to inspect after a request has completed.

## Sending RESTCONF Requests with curl

Retrieve the configured hostname of R1:

```bash
curl -k -i -u ADMIN \
  -H 'Accept: application/yang-data+json' \
  'https://192.0.2.1/restconf/data/Cisco-IOS-XE-native:native/hostname'
```

| Option | Purpose |
|---|---|
| `-k` | Skip TLS certificate verification |
| `-i` | Include HTTP response headers in the output |
| `-u ADMIN` | Authenticate as ADMIN and prompt for the password |
| `-H` | Specify an HTTP request header |
| `-X` | Specify an HTTP method explicitly |
| `-d` | Supply a request body |

With no request body or explicit method, curl sends a GET request.

An abbreviated example of a successful response:

```http
HTTP/1.1 200 OK
Content-Type: application/yang-data+json

{
  "Cisco-IOS-XE-native:hostname": "R1"
}
```

The `-k` option is used throughout the lab to accommodate a self-signed certificate. It disables server identity verification, not TLS encryption. In production, use a trusted certificate and a hostname that matches the certificate, rather than disabling verification.

For subsequent examples, define the following Bash variables and helper function:

```bash
RC='https://192.0.2.1/restconf'
NATIVE="$RC/data/Cisco-IOS-XE-native:native"
IFACE="$NATIVE/interface"
LO100="$IFACE/Loopback=100"
JSON='application/yang-data+json'

rc() {
  curl -k -i -u ADMIN -H "Accept: $JSON" "$@"
}
```

For example, the following command performs the same hostname GET request:

```bash
rc "$NATIVE/hostname"
```

## RESTCONF Resource Paths

The RESTCONF API root on IOS XE is `/restconf`. The two principal resource paths used in this lab are:

| Resource | Purpose |
|---|---|
| `/restconf/data` | Access modeled configuration and operational data |
| `/restconf/operations` | Discover and invoke YANG-defined RPCs |

### YANG Module Names and Data Nodes

Consider the following resource path:

```text
/restconf/data/Cisco-IOS-XE-native:native/interface/Loopback=0/description
```

```text
Cisco-IOS-XE-native:native
                    |
                interface
                    |
                Loopback (list)
                    |
                  name = 0
                    |
                description (leaf)
```

`Cisco-IOS-XE-native` is the YANG module name, and `native` is its top-level data node.

A node is qualified with its module name when it is the top-level node or belongs to a different module from its parent. RESTCONF URIs use **YANG module names**, not the arbitrary namespace prefixes used in XML documents or the prefixes declared in YANG files.

For example, the HTTP configuration is defined by an augmentation from `Cisco-IOS-XE-http`:

```bash
rc "$NATIVE/ip/Cisco-IOS-XE-http:http"
```

### YANG List Keys

A specific list entry is identified by appending `=key-value` to the list name.

| Resource | URI suffix under the native model |
|---|---|
| Interface configuration | `/interface` |
| Loopback0 | `/interface/Loopback=0` |
| Loopback0 description | `/interface/Loopback=0/description` |
| GigabitEthernet1 | `/interface/GigabitEthernet=1` |
| GigabitEthernet0/0/0 | `/interface/GigabitEthernet=0%2F0%2F0` |

A list with multiple keys uses comma-separated values in the order specified by the YANG model's `key` statement. For example, a list keyed by an IPv4 prefix and mask might use `=198.51.100.0,255.255.255.0`.

The exact list name, key names, key order, and data types depend on the model.

### URI Percent-Encoding

Characters that have special meaning in a URI must be percent-encoded when they occur within a key value.

| Character | Encoding |
|---|---|
| `/` | `%2F` |
| Space | `%20` |
| `,` | `%2C` |
| `=` | `%3D` |
| `%` | `%25` |

For example, the key for `GigabitEthernet0/0/0` in the native interface model is `0/0/0`.

```text
Incorrect: /interface/GigabitEthernet=0/0/0
Correct:   /interface/GigabitEthernet=0%2F0%2F0
```

Without encoding, the slashes are interpreted as resource path separators.

The IETF interface model uses a different hierarchy and a different key value: the complete interface name.

```text
/restconf/data/ietf-interfaces:interfaces/interface=GigabitEthernet0%2F0%2F0
```

Do not encode the delimiters that separate URI path segments or separate multiple list keys. Encode the individual key values.

## Discovering YANG Models and Resources

### YANG Library

A RESTCONF client does not receive a NETCONF-style `<hello>` containing model information. Instead, it can retrieve the YANG library.

IOS XE releases exposing the RFC 7895 YANG library support the following resource:

```bash
rc "$RC/data/ietf-yang-library:modules-state"
```

An abbreviated response can include information such as:

```json
{
  "ietf-yang-library:modules-state": {
    "module": [
      {
        "name": "Cisco-IOS-XE-native",
        "namespace": "http://cisco.com/ns/yang/Cisco-IOS-XE-native",
        "conformance-type": "implement"
      }
    ]
  }
}
```

The actual response includes the applicable module revisions and other metadata. A server implementing the newer RFC 8525 YANG library can expose `/restconf/data/ietf-yang-library:yang-library` instead or in addition.

### RESTCONF Capabilities

Where available, retrieve the capabilities resource:

```bash
rc "$RC/data/ietf-restconf-monitoring:restconf-state/capabilities"
```

This is useful for discovering optional RESTCONF features and query parameter support.

### Finding the Correct Data Hierarchy

The [Cisco IOS XE YANG models](https://github.com/YangModels/yang/tree/main/vendor/cisco/xe) can be downloaded and examined directly. Use the directory corresponding to the device's software release.

Another useful method is to configure a feature through the CLI and retrieve the appropriate parent resource using RESTCONF GET.

IOS XE 17.7.1 and later also support displaying modeled running configuration in JSON:

```text
R1# show running-config | format restconf-json
```

Cisco documents enabling `netconf-yang` to initialize the data model interfaces for this CLI conversion feature. This is separate from the prerequisites for RESTCONF itself.

Use this output to identify relevant nodes and payload structure. Do not blindly send the entire output as a RESTCONF request: the body must match the specific target resource and HTTP method.

## HTTP Methods

| Method | Operation | Target |
|---|---|---|
| GET | Read | Existing resource |
| HEAD | Read response metadata | Existing resource |
| POST | Create | Parent of the resource to create |
| PUT | Create or replace | Resource to create or replace |
| PATCH | Merge or apply edits | Existing target resource |
| DELETE | Delete | Resource to delete |
| OPTIONS | Discover supported methods | Target resource |

POST is also used to invoke YANG-defined RPCs.

### HTTP Headers

| Header | Function |
|---|---|
| `Accept` | Requests a particular response media type |
| `Content-Type` | Describes the encoding of the request body |
| `Authorization` | Carries authentication credentials |
| `Location` | Identifies a newly created resource in a successful POST response |
| `Allow` | Lists supported HTTP methods |
| `Accept-Patch` | Lists supported PATCH media types |

For ordinary JSON-encoded data requests, use `application/yang-data+json`. For XML, use `application/yang-data+xml`.

A GET request normally has no body, so it does not need a `Content-Type` header.

## GET: Retrieving Data

### Retrieving Configuration Data

Retrieve the entire configuration subtree represented by the Cisco native model:

```bash
rc "$NATIVE"
```

Retrieve only Loopback0:

```bash
rc "$IFACE/Loopback=0"
```

An abbreviated example response:

```json
{
  "Cisco-IOS-XE-native:Loopback": [
    {
      "name": 0,
      "ip": {
        "address": {
          "primary": {
            "address": "1.1.1.1",
            "mask": "255.255.255.255"
          }
        }
      }
    }
  ]
}
```

A YANG list is represented by a JSON array, even if only one list instance is returned.

### Retrieving Operational Data

The Cisco native configuration model is not the same as the operational interface model.

Retrieve GigabitEthernet1 operational information:

```bash
rc "$RC/data/Cisco-IOS-XE-interfaces-oper:interfaces/interface=GigabitEthernet1"
```

Retrieve its interface statistics:

```bash
rc "$RC/data/Cisco-IOS-XE-interfaces-oper:interfaces/interface=GigabitEthernet1/statistics"
```

These resources represent operational state, including counters. They are not writable configuration resources.

### Retrieving XML Instead of JSON

Specify the XML response media type:

```bash
curl -k -i -u ADMIN \
  -H 'Accept: application/yang-data+xml' \
  "$NATIVE/hostname"
```

Example response body:

```xml
<hostname xmlns="http://cisco.com/ns/yang/Cisco-IOS-XE-native">R1</hostname>
```

The resource URI is independent of the data encoding. Only the headers and representation change.

## GET Query Parameters

Query parameters follow `?` in the URI. Multiple parameters are separated by `&`.

| Parameter | Purpose |
|---|---|
| `content` | Select configuration data, state data, or both |
| `depth` | Limit the depth of returned data |
| `fields` | Select specific descendants of a resource |
| `with-defaults` | Control the reporting of default values, if supported |

Support for optional query parameters depends on the server and software release.

### content

| Value | Data Returned |
|---|---|
| `config` | Configuration data (`config true`) |
| `nonconfig` | Operational state data (`config false`) |
| `all` | Configuration and operational data (default) |

```bash
rc "$RC/data/ietf-interfaces:interfaces?content=config"
```

`content=config` is conceptually similar to the NETCONF `<get-config>` operation. `content=all` is conceptually similar to `<get>`.

A query parameter does not transform an operational model into a configuration model, or vice versa.

### depth

The target resource is considered depth 1. Limit the returned subtree to two levels:

```bash
rc "$IFACE?depth=2"
```

`depth=unbounded` removes the depth restriction.

### fields

Retrieve only the name and description of Loopback0:

```bash
rc "$IFACE/Loopback=0?fields=name;description"
```

Retrieve its IPv4 address and mask:

```bash
rc "$IFACE/Loopback=0?fields=ip/address/primary"
```

Combine query parameters:

```bash
rc "$IFACE/Loopback=0?content=config&fields=name;description;ip/address/primary"
```

Quote the URL in shell commands. Otherwise, shell metacharacters such as `&` and `;` can be interpreted as shell syntax.

### with-defaults

Where supported, the `with-defaults` query parameter selects how default-valued configuration nodes are reported.

| Value | Description |
|---|---|
| `report-all` | Include default values |
| `trim` | Omit nodes whose values equal their schema defaults |
| `explicit` | Report explicitly configured values, along with applicable non-default data |
| `report-all-tagged` | Include default values and identify default data using metadata |

For example:

```bash
rc "$IFACE/Loopback=0?with-defaults=report-all"
```

Not every server supports every reporting mode.

## POST: Creating a Resource

Use POST to create Loopback100. The request targets the parent of the new list entry (`interface`), not the URI of Loopback100 itself.

```bash
rc -X POST "$IFACE" \
  -H "Content-Type: $JSON" \
  -d '{
    "Cisco-IOS-XE-native:Loopback": [
      {
        "name": 100,
        "description": "Created using RESTCONF",
        "ip": {
          "address": {
            "primary": {
              "address": "198.51.100.100",
              "mask": "255.255.255.255"
            }
          }
        }
      }
    ]
  }'
```

The expected status code is `201 Created`. The `Location` response header identifies the newly created resource.

Verify the configuration:

```text
R1# show running-config interface Loopback100
interface Loopback100
 description Created using RESTCONF
 ip address 198.51.100.100 255.255.255.255
end
```

Alternatively, retrieve the new resource:

```bash
rc "$LO100"
```

Repeat the same POST request. The target data already exists, so the create operation fails, normally with `409 Conflict` and a `data-exists` error.

POST is not a general-purpose configuration merge operation.

## PATCH: Modifying a Resource

A plain PATCH request merges the supplied data into the target resource. Existing data that is not included in the payload is preserved.

Modify only Loopback100's description:

```bash
rc -X PATCH "$LO100" \
  -H "Content-Type: $JSON" \
  -d '{
    "Cisco-IOS-XE-native:Loopback": [
      {
        "name": 100,
        "description": "Modified using PATCH"
      }
    ]
  }'
```

The expected status code is `204 No Content` for a successful edit without a response body.

Verify the configuration:

```text
R1# show running-config interface Loopback100
interface Loopback100
 description Modified using PATCH
 ip address 198.51.100.100 255.255.255.255
end
```

The description has changed, but the IP address has been preserved.

The URI identifies the existing target resource. The payload can add previously absent child configuration to that resource, but plain PATCH cannot create the target resource itself if it does not exist.

Plain PATCH uses the ordinary YANG data media type (`application/yang-data+json` or `application/yang-data+xml`). It does not provide an explicit delete operation for individual child nodes. Use DELETE or YANG Patch for that purpose.

## PUT: Creating or Replacing a Resource

PUT targets the resource itself. If the resource exists, PUT replaces its configuration rather than merging with it.

Replace Loopback100's configuration:

```bash
rc -X PUT "$LO100" \
  -H "Content-Type: $JSON" \
  -d '{
    "Cisco-IOS-XE-native:Loopback": [
      {
        "name": 100,
        "description": "Replaced using PUT"
      }
    ]
  }'
```

The expected status code is `204 No Content` for replacement of an existing resource.

Verify the configuration:

```text
R1# show running-config interface Loopback100
interface Loopback100
 description Replaced using PUT
 no ip address
end
```

Because the IPv4 address was omitted from the replacement payload, the previously configured address is removed. Schema defaults can still apply to omitted data.

PUT can also create a nonexistent resource. In that case, the expected status is `201 Created`.

### Replacing a Leaf Instead of an Entire List Entry

The scope of a replacement depends on the URI, not just the body.

For example, replace only the description leaf:

```bash
rc -X PUT "$LO100/description" \
  -H "Content-Type: $JSON" \
  -d '{"Cisco-IOS-XE-native:description":"Leaf-level PUT"}'
```

This targets the description, not the interface configuration as a whole. Other interface configuration is unaffected.

A PUT request targeting a large subtree can unintentionally remove existing configuration. Target the narrowest appropriate resource.

### PUT vs. PATCH

Assume Loopback100 has an IPv4 address and a description, and the body specifies the list key and a new description only.

| Method and Target | Effect on Description | Effect on IPv4 Address |
|---|---|---|
| PATCH to Loopback100 | Modified | Preserved |
| PUT to Loopback100 | Modified | Removed |
| PUT to Loopback100's description leaf | Modified | Preserved |

The scope of the edit is particularly important when using PUT.

### POST vs. PUT

| Characteristic | POST to Parent | PUT to Target |
|---|---|---|
| Creates nonexistent resource | Yes | Yes |
| Replaces existing resource | No | Yes |
| Existing resource | Conflict | Replacement |
| URI for creating Loopback100 | `/interface` | `/interface/Loopback=100` |
| Idempotent by HTTP semantics | No | Yes |

Idempotence means that repeating an identical request has the same intended effect as executing it once. It does not mean that each request must return the same status code.

## DELETE: Removing a Resource

Delete only the description of Loopback100:

```bash
rc -X DELETE "$LO100/description"
```

The expected status code is `204 No Content`. The interface itself remains configured.

Delete the entire Loopback100 resource:

```bash
rc -X DELETE "$LO100"
```

Verify the deletion:

```text
R1# show ip interface brief
```

A subsequent GET targeting Loopback100 should fail because the resource no longer exists. Repeating the DELETE should also fail because the target no longer exists. Examine the returned status and error tag. RFC 8040 maps `data-missing` to `409 Conflict`; a server can also report a missing resource with `404 Not Found` and `invalid-value`.

DELETE has no request body in these examples.

## HEAD and OPTIONS

HEAD retrieves response headers without the response body:

```bash
curl -k -I -u ADMIN -H "Accept: $JSON" "$NATIVE/hostname"
```

OPTIONS can be used to discover the supported methods and PATCH encodings:

```bash
rc -X OPTIONS "$IFACE"
```

Examine the `Allow` and `Accept-Patch` response headers, if present. Support for particular methods can depend on the resource and the server implementation.

## Invoking YANG-Defined RPCs

The `/restconf/operations` resource exposes YANG-defined RPCs.

Retrieve the list of advertised operations:

```bash
rc "$RC/operations"
```

A POST request to a specific operation invokes the corresponding RPC.

For example, IOS XE exposes the `save-config` RPC in the `cisco-ia` module:

```bash
rc -X POST "$RC/operations/cisco-ia:save-config"
```

The operation does not require input parameters. Examine the response status and any returned RPC output.

Verify through the CLI:

```text
R1# show startup-config
```

This is the programmatic equivalent of saving the running configuration to the startup configuration.

If an RPC requires input parameters, the request body contains a YANG-defined `input` node qualified by its module name. If the RPC returns output parameters, the response contains the corresponding `output` node.

RESTCONF does not provide arbitrary CLI execution simply by POSTing an IOS command string. The requested operation and its inputs must be supported by an applicable YANG model.

## RESTCONF and Configuration Datastores

RESTCONF edits affect the active device configuration through the server's datastore handling.

Unlike NETCONF, ordinary RESTCONF operations do not expose an explicit client-controlled candidate configuration and commit workflow.

- Successful RESTCONF configuration edits take effect without a separate client-issued commit.
- Standard RESTCONF does not provide explicit datastore lock and unlock operations.
- RESTCONF does not support NETCONF confirmed commit.
- Configuration changes are not necessarily saved to startup configuration automatically on IOS XE.

A running configuration lock held by NETCONF or the CLI can interfere with RESTCONF writes. If a write fails despite valid credentials, paths, and payloads, check the datastore lock status.

```text
R1# show netconf-yang datastores
R1# show netconf-yang sessions detail
```

An individual RESTCONF edit request is processed as a configuration transaction. Multiple separate HTTP requests do not form a single atomic transaction simply because they reuse the same HTTP connection.

## YANG Patch

**YANG Patch** (RFC 8072) is an optional RESTCONF extension that uses the PATCH method with a different body format from plain PATCH.

| PATCH Type | JSON Media Type | XML Media Type |
|---|---|---|
| Plain PATCH | `application/yang-data+json` | `application/yang-data+xml` |
| YANG Patch | `application/yang-patch+json` | `application/yang-patch+xml` |

YANG Patch supports a sequence of edits, including `create`, `merge`, `replace`, `delete`, `remove`, `insert`, and `move`. The edits are evaluated in order and applied atomically. If an edit fails, the patch must not be partially applied.

For example, after deleting Loopback100 in the previous exercise, create it and configure its description in a single YANG Patch request:

```bash
rc -X PATCH "$IFACE" \
  -H 'Content-Type: application/yang-patch+xml' \
  -d '<yang-patch xmlns="urn:ietf:params:xml:ns:yang:ietf-yang-patch">
    <patch-id>loopback100</patch-id>
    <edit>
      <edit-id>create-loopback</edit-id>
      <operation>create</operation>
      <target>/Loopback=100</target>
      <value>
        <Loopback xmlns="http://cisco.com/ns/yang/Cisco-IOS-XE-native">
          <name>100</name>
        </Loopback>
      </value>
    </edit>
    <edit>
      <edit-id>set-description</edit-id>
      <operation>merge</operation>
      <target>/Loopback=100/description</target>
      <value>
        <description xmlns="http://cisco.com/ns/yang/Cisco-IOS-XE-native">YANG Patch</description>
      </value>
    </edit>
  </yang-patch>'
```

The target paths inside the edits are relative to the resource specified in the request URI. A successful YANG Patch returns a patch-status response identifying the patch and indicating success.

Check `Accept-Patch` and the device's supported capabilities before using this extension. Plain PATCH is sufficient for many configuration tasks.

## RESTCONF Security

### RESTCONF Service-Level ACLs

Restrict RESTCONF access to the management workstation:

```text
R1(config)# ip access-list standard RESTCONF-CLIENTS
R1(config-std-nacl)# permit host 192.0.2.100
R1(config-std-nacl)# exit
R1(config)# restconf ipv4 access-list name RESTCONF-CLIENTS
```

The ACL filters RESTCONF requests based on their source IP address. It is distinct from a VTY access class and from the NETCONF SSH service-level ACL.

Remove the association:

```text
R1(config)# no restconf ipv4 access-list name RESTCONF-CLIENTS
```

IPv6 clients can be restricted using an IPv6 ACL and the following command:

```text
R1(config)# restconf ipv6 access-list name RESTCONF-IPV6-CLIENTS
```

An IPv4 ACL does not restrict IPv6 management access.

### TLS Server Authentication

HTTPS provides encrypted transport and allows the client to verify the identity of the server using its certificate.

Verify the HTTPS server status and installed certificates:

```text
R1# show ip http server secure status
R1# show crypto pki certificates
```

For example, if R1 has a certificate valid for `r1.example.net`, signed by the CA in `lab-ca.pem`, and the workstation can resolve that name:

```bash
curl --cacert lab-ca.pem -u ADMIN \
  -H 'Accept: application/yang-data+json' \
  'https://r1.example.net/restconf/data/Cisco-IOS-XE-native:native/hostname'
```

This validates the certificate chain and server name instead of bypassing verification with `-k`.

## Using Postman

Postman provides a graphical interface for constructing RESTCONF requests.

For a GET request targeting R1's hostname, configure the following:

| Setting | Value |
|---|---|
| Method | GET |
| URL | `https://192.0.2.1/restconf/data/Cisco-IOS-XE-native:native/hostname` |
| Authorization type | Basic Auth |
| Username | `ADMIN` |
| Password | The configured local user's password |
| Header | `Accept: application/yang-data+json` |
| Body | None |

For POST, PUT, and PATCH requests, additionally configure the `Content-Type: application/yang-data+json` header and select a raw JSON body containing the modeled data.

Postman allows environment variables to be used in URLs. For example, `{{host}}` can represent the router's management address.

If a self-signed certificate is used, import the trusted CA or, for an isolated lab only, disable SSL certificate verification. Reenable verification when leaving the lab.

## Using Python requests

Install the `requests` library in a Python environment:

```bash
pip install requests
```

### Retrieving Configuration

The following example retrieves R1's hostname:

```python
from getpass import getpass
import requests

url = "https://192.0.2.1/restconf/data/Cisco-IOS-XE-native:native/hostname"

response = requests.get(
    url,
    auth=("ADMIN", getpass("Password: ")),
    headers={"Accept": "application/yang-data+json"},
    verify=False,
    timeout=30
)
response.raise_for_status()
print(response.json()["Cisco-IOS-XE-native:hostname"])
```

`verify=False` is the equivalent of curl's `-k`. Use a trusted server certificate in production, for example by specifying `verify="lab-ca.pem"` with a matching server hostname.

### Creating and Modifying Configuration

The following example creates or replaces a dedicated practice interface, Loopback101, and then updates its description without removing its IPv4 address:

```python
from getpass import getpass
import requests

url = (
    "https://192.0.2.1/restconf/data/"
    "Cisco-IOS-XE-native:native/interface/Loopback=101"
)
media_type = "application/yang-data+json"

with requests.Session() as session:
    session.auth = ("ADMIN", getpass("Password: "))
    session.headers.update({
        "Accept": media_type,
        "Content-Type": media_type
    })
    session.verify = False  # Isolated lab only

    configuration = {
        "Cisco-IOS-XE-native:Loopback": [
            {
                "name": 101,
                "description": "Created using Python",
                "ip": {
                    "address": {
                        "primary": {
                            "address": "198.51.100.101",
                            "mask": "255.255.255.255"
                        }
                    }
                }
            }
        ]
    }

    response = session.put(url, json=configuration, timeout=30)
    response.raise_for_status()
    print("PUT:", response.status_code)

    update = {
        "Cisco-IOS-XE-native:Loopback": [
            {"name": 101, "description": "Updated using Python"}
        ]
    }
    response = session.patch(url, json=update, timeout=30)
    response.raise_for_status()
    print("PATCH:", response.status_code)

    response = session.get(url, timeout=30)
    response.raise_for_status()
    print(response.json())
```

The `json=` argument serializes a Python dictionary into a JSON request body. The headers explicitly identify the RESTCONF media type, rather than the generic `application/json` media type.

`raise_for_status()` raises an exception for HTTP error status codes. Do not attempt to decode a `204 No Content` response using `response.json()`, because there is no response body.

A `requests.Session()` can reuse the underlying HTTP connection. It does not create a NETCONF-style configuration session.

## HTTP Status Codes and RESTCONF Errors

| Status Code | Meaning | Example |
|---|---|---|
| 200 OK | Successful request with a response body | GET |
| 201 Created | New resource created | POST or PUT |
| 204 No Content | Successful request without a body | PATCH or DELETE |
| 400 Bad Request | Invalid request | Invalid YANG value or malformed JSON |
| 401 Unauthorized | Authentication failure | Incorrect password |
| 403 Forbidden | Access denied | Insufficient authorization |
| 404 Not Found | Resource not found | Nonexistent interface |
| 405 Method Not Allowed | Unsupported method for resource | Write to read-only operational data |
| 406 Not Acceptable | Unsupported response representation | Invalid `Accept` header |
| 409 Conflict | Conflict with existing data or datastore | Duplicate POST or configuration lock |
| 412 Precondition Failed | Conditional request precondition not met | Stale `If-Match` value |
| 415 Unsupported Media Type | Unsupported request body representation | Incorrect `Content-Type` |
| 500 Internal Server Error | Server-side failure | Internal processing failure |
| 503 Service Unavailable | Service unavailable | Model-driven services not ready |

The exact status code returned can depend on the implementation and specific error condition. Examine the structured response body as well as the HTTP status.

For example, a duplicate create request can generate an error similar to the following (illustrative):

```json
{
  "ietf-restconf:errors": {
    "error": [
      {
        "error-type": "application",
        "error-tag": "data-exists",
        "error-message": "The requested resource already exists"
      }
    ]
  }
}
```

Other useful error tags include `invalid-value`, `unknown-element`, `data-missing`, `access-denied`, `in-use`, and `operation-not-supported`.

### Conditional Requests

A server can return an `ETag` header identifying a version of a resource. When supported, a client can send an `If-Match` header with that value on a subsequent edit. If the resource has changed since the earlier GET, the edit can fail with `412 Precondition Failed` rather than overwriting the newer configuration.

This provides optimistic concurrency control, not a datastore lock. Support for conditional requests and resource-specific validators should be verified on the target device.

## Troubleshooting RESTCONF

### Transport and TLS

Verify reachability and TCP port 443:

```bash
ping 192.0.2.1
nc -vz 192.0.2.1 443
```

Use verbose curl output to diagnose TLS and HTTP problems:

```bash
curl -v -u ADMIN \
  -H 'Accept: application/yang-data+json' \
  'https://192.0.2.1/restconf/data/Cisco-IOS-XE-native:native/hostname'
```

If certificate verification fails, verify the trust chain and server identity. A certificate verification error is a TLS-layer failure, not an HTTP 401 or 403 response.

### Service and Authentication Configuration

```text
R1# show running-config | include restconf
R1# show running-config | include ip http
R1# show ip http server secure status
R1# show platform software yang-management process
R1# show running-config | section aaa
R1# show running-config | include yang-interfaces
```

Verify RESTCONF and HTTPS enablement, process initialization, AAA method lists, user privileges, and RESTCONF service-level ACLs.

### Data Model and Payload Problems

If the server is reachable and authenticated but a request fails, verify:

- The YANG model is supported by the device.
- The path matches the model hierarchy and capitalization.
- The correct module names qualify the relevant nodes.
- All list keys appear in the required order and are correctly encoded.
- The HTTP method targets the correct resource.
- JSON arrays, objects, and value types match the YANG schema.
- The request and response media types are correct.
- The target data is writable and the datastore is not locked.

A valid JSON document is not necessarily valid YANG-modeled data.

| Problem | Likely Cause |
|---|---|
| TCP/443 is unreachable | Routing, ACL, firewall, or HTTPS service problem |
| HTTPS works but `/restconf/data` does not | RESTCONF disabled or model-driven processes not initialized |
| Certificate verification fails | Untrusted issuer, expired certificate, or name mismatch |
| GET succeeds but POST fails | Invalid payload, existing data, or insufficient write access |
| Interface with slashes in its name cannot be retrieved | Key value not percent-encoded |
| PATCH fails for a nonexistent target | Plain PATCH requires an existing target |
| PUT unexpectedly removes other configuration | Replacement applied to a broader subtree than intended |
| A successful write returns an empty body | Normal `204 No Content` response |
| Configuration is lost after reboot | Running configuration was not saved |

## Practice Exercises

### Exercise 1: RESTCONF Enablement and Verification

1. Configure a local privilege level 15 user.
2. Enable RESTCONF and HTTPS without enabling NETCONF or AAA.
3. Verify the model-driven management processes.
4. Retrieve R1's hostname using curl.
5. Enable AAA with local authentication and authorization and repeat the request.

### Exercise 2: Retrieval and Resource Paths

1. Retrieve Loopback0 using the Cisco native model.
2. Retrieve GigabitEthernet1 using the operational interface model.
3. Retrieve only the interface counters.
4. Retrieve the hostname as both XML and JSON.
5. Use `fields` to retrieve selected interface configuration elements.
6. Use `depth` to limit the returned data hierarchy.
7. Retrieve the YANG library and identify the supported interface models.

### Exercise 3: Configuration Operations

1. Create Loopback100 with an IPv4 address using POST.
2. Repeat the POST and examine the error.
3. Modify its description using PATCH and verify that the IPv4 address remains.
4. Replace the interface with PUT, specifying only the key and description.
5. Verify that the original IP address is removed.
6. Restore the IP address using PATCH.
7. Use PUT to replace only the description leaf and verify that the address remains.
8. Delete the description leaf using DELETE.
9. Delete the entire interface using DELETE.
10. Recreate the interface using PUT and verify the status code.

### Exercise 4: Saving Configuration and Datastore Locks

1. Modify the configuration using RESTCONF.
2. Invoke the Cisco `save-config` RPC.
3. Verify the startup configuration.
4. Enable NETCONF and lock the running datastore through NETCONF.
5. Attempt to modify the configuration using RESTCONF.
6. Release the lock and repeat the request.

### Exercise 5: Security

1. Configure a RESTCONF service-level ACL permitting only the management workstation.
2. Verify access from permitted and denied sources.
3. Compare the results of using correct and incorrect credentials.
4. Compare TLS certificate verification with and without curl's `-k` option.

### Exercise 6: Client Tools

1. Repeat a GET, POST, PATCH, and DELETE request using Postman.
2. Write a Python script that creates a loopback interface, verifies its configuration, and deletes it.
3. Handle HTTP error status codes and responses that do not contain a body.

## Key Concepts

- RESTCONF is enabled using `restconf` and `ip http secure-server`.
- RESTCONF uses HTTPS (TCP port 443 by default).
- Neither `netconf-yang` nor `ip http server` is required for RESTCONF.
- RESTCONF is not a VTY service.
- The `nginx` process supports RESTCONF; `ncsshd` is specific to NETCONF over SSH.
- RESTCONF uses YANG module names and list keys to construct resource URIs.
- Reserved characters within a list key must be percent-encoded.
- `/restconf/data` accesses modeled data; `/restconf/operations` accesses modeled RPCs.
- `Accept` specifies the desired response format; `Content-Type` describes the supplied request body.
- RESTCONF data uses `application/yang-data+json` or `application/yang-data+xml`.
- A YANG list is represented as a JSON array, including a single list instance.
- GET retrieves configuration or operational state data.
- POST creates a resource beneath the specified parent or invokes a modeled operation.
- PUT creates or replaces the targeted resource.
- Plain PATCH merges data into an existing targeted resource.
- DELETE removes the targeted resource.
- `content`, `fields`, and `depth` control the data returned by GET.
- RESTCONF does not expose a standard explicit candidate/commit workflow or confirmed commit.
- A successful RESTCONF edit is distinct from saving the configuration to startup configuration.
- The Cisco `cisco-ia:save-config` RPC can save the running configuration.
- `restconf ipv4 access-list name` applies a service-level IPv4 ACL.
- An HTTP status code and structured RESTCONF error body help diagnose failed requests.

## Further Reading

- [RESTCONF, NETCONF Overview](overview.md)
- [NETCONF Configuration and Operations](netconf.md)

### Cisco Documentation

- [RESTCONF Protocol, Cisco IOS XE 17.15.x](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/prog/configuration/1715/b_1715_programmability_cg/m_1715_prog_restconf.html)
- [NETCONF and RESTCONF Service-Level ACLs, Cisco IOS XE 17.15.x](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/prog/configuration/1715/b_1715_programmability_cg/m_1715_prog_service-level_acls.html)
- [Model Based AAA, Cisco IOS XE 17.18.x](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/prog/configuration/1718/b-1718-programmability-cg/model-based-aaa.html)
- [Cisco IOS XE YANG Models](https://github.com/YangModels/yang/tree/main/vendor/cisco/xe)

### RFCs

- [RFC 8040 - RESTCONF Protocol](https://www.rfc-editor.org/rfc/rfc8040.html)
- [RFC 7951 - JSON Encoding of Data Modeled with YANG](https://www.rfc-editor.org/rfc/rfc7951.html)
- [RFC 8072 - YANG Patch Media Type](https://www.rfc-editor.org/rfc/rfc8072.html)
- [RFC 7895 - YANG Module Library](https://www.rfc-editor.org/rfc/rfc7895.html)
- [RFC 8525 - YANG Library](https://www.rfc-editor.org/rfc/rfc8525.html)
- [RFC 8341 - Network Configuration Access Control Model](https://www.rfc-editor.org/rfc/rfc8341.html)

### Tools

- [curl](https://curl.se/docs/)
- [Postman](https://learning.postman.com/)
- [Python requests](https://requests.readthedocs.io/)
