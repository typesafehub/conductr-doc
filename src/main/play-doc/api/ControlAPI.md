# Control API

ConductR's control protocol is RESTful and has the following functional scope:

* [Load a bundle and optionally its configuration](#Load-a-bundle)
* [Scale a bundle i.e. starting it and possibly stopping it](#Scale-a-bundle)
* [Unload a bundle](#Unload-a-bundle)
* [Query bundle state](#Query-bundle-state)
* [Receive bundle state events](#Receive-bundle-state-events)
* [Query member state](#Query-member-state)
* [Query agent state](#Query-agent-state)
* [Query logs by bundle](#Query-logs-by-bundle)
* [Query events by bundle](#Query-events-by-bundle)

## Load a bundle

Request an upload a bundle and optionally, its configuration to ConductR.

Here is an example of uploading a Visualizer bundle without any configuration. [Curl](http://curl.haxx.se/) is being used.

```bash
curl \
  --form bundleConf=@visualizer/target/bundle/bundle/tmp/bundle.conf \
  --form bundle=@visualizer/target/bundle/visualizer-v1.1-cfe2a36795fd78507c4d2b5817152ae449e4acd9d5ea94d1f604d2c11417e40f.zip \
  http://localhost:9005/v2/bundles
```

### Request

```
POST /v2/bundles
```

The following fields are provided as multipart/form-data fields:

Field                | Description
---------------------|------------
bundleConf           | The file that is the [bundle.conf](BundleConfiguration) of the bundle being uploaded. Typically this will be the [bundle.conf](BundleConfiguration) contained within the bundle file.
bundleConfOverlay    | Optional. The file that is the [bundle.conf](BundleConfiguration) containing the overridden configuration values. Typically specified to provide deployment target specific configuration. Alternatively, this will be [bundle.conf](BundleConfiguration) that is optionally contained within the [configuration bundle](#Configuration-Bundles).
bundle               | The file that is the bundle. The filename is important with its hex digest string and is required to be consistent with the SHA-256 hash of the bundle's contents. Any inconsistency between the hashes will result in the load being rejected.
configuration        | Optional. Similar in form to the bundle, only that is the file that describes the configuration. Again any inconsistency between the hex digest string in the filename, and the SHA-256 digest of the actual contents will result in the load being rejected.

### Responses

#### Success

```json
HTTP/1.1 200 OK
Content-Type: application/json

{
  "requestId": "{request-id}",
  "bundleId": "{bundleId}"
}
```

Field        | Description
-------------|------------
requestId    | A UUID for this request.
bundleId     | An identifier directly related to the digest of a bundle and, if supplied, the digest of its configuration. This identifier is guaranteed to remain related to both the bundle and its optional configuration. It is used for subsequent distinctive references by the API.

#### Failure

```
HTTP/1.1 400 Bad Request
```

The request was badly formed.

Other non 2xx status codes should also be treated as a failure.

## Scale a bundle

Request a scale (run) of a bundle to the value provided. A scale value of 0 is interpreted as a stop of all running instances.

### Request

```
PUT /v2/bundles/{bundleIdOrName}?scale={scale}&affinity={bundleIdOrName}
PUT /v2/bundles/{bundleIdOrName}?scaleOffset={scaleOffset}&affinity={bundleIdOrName}
```

Field            | Description
-----------------|------------
affinity         | Optional. Identifier to other bundle. If specified, the current bundle will be run on the same host where the specified bundle is currently running.
bundleIdOrName   | An existing bundle identifier, a shortened version of it (min 7 characters) or a non-ambigious name given to the bundle during loading. Names may be qualified with a tag in order to further distinguish them. A tag is delimited by a `:` e.g. `mybundle:1.0.0-beta.1` is a valid name where `mybundle` is the name and `1.0.0-beta.1` is the tag to select (in this case a version number).
scale            | The number of instances of the bundle to start. A scale value of 0 indicates that all instances should be stopped.
scaleOffset      | An offset relative to the existing number that have been scaled e.g. a `scaleOffset` of `-1` indicates that the current scale should be reduced by one.

### Responses

#### Success

```json
HTTP/1.1 200 OK
Content-Type: application/json

{
  "requestId": "{request-id}",
  "bundleId": "{bundleId}"
}
```

Field        | Description
-------------|------------
requestId    | A UUID for this request.

#### Failure

```
HTTP/1.1 404 Not Found
```

The supplied bundleIdOrName cannot be resolved to a bundle that has been loaded.

```
HTTP/1.1 300 Multiple choices

Specified Bundle ID/name: {bundleIdOrName} resulted in multiple Bundle IDs: '{commaDelimBundleIds}'
```

The shortened id or name cannot be resolved to a single bundle id. Try providing a more specific one.

Other non 2xx status codes should also be treated as a failure.

## Unload a bundle

Request to unload a bundle that has been stopped.

### Request

```
DELETE /v2/bundles/{bundleIdOrName}
```

Field            | Description
-----------------|------------
bundleIdOrName   | An existing bundle identifier, a shortened version of it (min 7 characters) or a non-ambigious name given to the bundle during loading. Names may be qualified with a tag in order to further distinguish them. A tag is delimited by a `:` e.g. `mybundle:1.0.0-beta.1` is a valid name where `mybundle` is the name and `1.0.0-beta.1` is the tag to select (in this case a version number).

### Responses

#### Success

```
HTTP/1.1 200 OK
Content-Type: application/json

{
  "requestId": "{request-id}"
}
```

Field        | Description
-------------|------------
requestId    | A UUID for this request.

#### Failure

```
HTTP/1.1 404 Not Found
```

The supplied bundleIdOrName cannot be resolved to a bundle that has been loaded.

```
HTTP/1.1 300 Multiple choices

Specified Bundle ID/name: {bundleIdOrName} resulted in multiple Bundle IDs: '{commaDelimBundleIds}'
```

The shortened id or name cannot be resolved to a single bundle id. Try providing a more specific one.

Other non 2xx status codes should also be treated as a failure.

## Query bundle state

Retrieve the current state of bundles within ConductR.

### Request

```
GET /v2/bundles
```

### Responses

#### Success

```
HTTP/1.1 200 OK
Content-Type: application/json

[]
```

There are no bundles JSON array is empty.

```json
HTTP/1.1 200 OK
Content-Type: application/json

[
  {
    "bundleId": "{bundleId}",
    "bundleDigest": "{bundleDigest}",
    "configurationDigest": "{configurationDigest}",
    "attributes": {
      "system": "{system}",
      "nrOfCpus": {nrOfCpus},
      "memory": {memory},
      "diskSpace": {diskSpace},
      "roles": {roles},
      "bundleName": "{bundleName}",
      "tags": {tags}
    },
    "bundleConfig": {
      "endpoints": {
        "{endpoint-name}": {
          "bindProtocol": "{bindProtocol}",
          "services": {services}
        }
      }
    },
    "bundleScale": {
      "scale": {scale},
      "affinity": "{otherBundleId}"
    },
    "bundleExecutions": [
      {
        "host": "{host}",
        "endpoints": {
          "{endpoint-name}": {
            "bindPort": {bindPort},
            "hostPort": {hostPort}
        }
      },
      "isStarted": {isStarted}
      "isActive": {isActive}
    }
  ],
  "bundleInstallations": [
    {
      "uniqueAddress": {
        "address": "{address}",
        "uid": {uid}
      },
      "bundleFile": "{bundleFile}",
      "configurationFile": "{configurationFile}"
    }
  ],
  "hasError": {hasError}
  }
]
```

##### Sections

Section             | Description
--------------------|------------
attributes          | Attributes provide meta information about the bundle that ConductR requires in order to make various decisions such as scheduling the replication of a bundle. The meta information is provided so that a bundle's configuration need not need to be extracted from the bundle or configuration archive. Indeed, the archive(s) may not have been fully streamed at the point where this meta data is required hence it being provided in addition to them.
bundleConfig        | Bundle configuration provides information that has been extracted from a bundle configuration and so will not exist if a bundle is not yet running within the ConductR cluster.
bundleScale         | Provides the information of scale requested and so will not exist if a bundle has not been requested to run within ConductR cluster.
bundleExecutions    | Bundle executions describes what is running and where in the context of a bundle and so will not be present when there are none running.
bundleInstallations | Bundle installations describe where the bundle has been replicated i.e. written to a file system. The array may be empty in the case where the first replication has not yet completed.
endpoints           | The endpoint declaration of a bundle component expressed as an object.

##### Fields

Field               | Description
--------------------|------------
address             | The location of a ConductR member.
affinity            | The bundle identifier that references a different bundle. If specified, the current bundle will be run on the same host where the specified bundle is currently running.
bindPort            | The network port that is used by a bundle component to bind to an interface. This may be the same value as the `hostPort` when running outside of a container.
bindProtocol        | The network protocol that is used by a bundle component to bind to an interface.
bundleDigest        | The hex form of a digest representing the contents of the bundle.
bundleFile          | The location of a bundle archive on disk.
bundleId            | The bundle identifier.
bundleName          | A human-readable name representing the bundle.
configurationDigest | The hex form of a digest representing the contents of the configuration. This field is optional and provided only when there is a configuration.
configurationFile   | The location of a configuration archive on disk. This field is optional and provided only when there is a configuration.
diskSpace           | The amount of disk required when a bundle is running. Values are expressed in bytes.
endpoint-name       | The name of an endpoint. Endpoint names are distinct across all components of a bundle.
hasError            | `true` if some event in ConductR has raised an error, `false` otherwise.
host                | The ip address of the machine hosting the bundle's components.
hostPort            | The port of the machine hosting the bundle component that will be mapped to the `bindPort`. `hostPort` is distinct across all services running on a host.
isActive            | `true` if the bundle has signalled that it has started successfully, `false` if it is in the process of starting up or shutting down.
isStarted           | Deprecated. Use `isActive` instead.
memory              | The amount of resident memory required to run the bundle. Values are expressed in bytes.
nrOfCpus            | The minimum number of cpus required to run the bundle (can be fractions thereby expressing a portion of CPU). This value is considered when starting a bundle on a node. If the specified CPUs exceeds the available CPUs on a node, then this node is not considered for scaling the bundle. Once running, the application is not restricted to the given value and tries to use all available CPUs on the node.
roles               | An array of strings representing the roles that a bundle plays in a ConductR cluster. These roles are matched for eligibility with cluster members when searching for one to load and scale on. Only cluster members with matching roles will be selected.
scale               | The requested number of instance(s) of the bundle to be started and run.
services            | An array of string URIs providing the addresses for a given endpoint's proxying.
system              | The name of a system that the bundle belongs to. Systems are strings that may be used by a number of bundles in order to associate them. ConductR provides a guarantee that bundles belonging to the same system are started with the first one in isolation to the starting of the rest. This behavior can be leverage to form clusters where the first node must form the cluster and other nodes may then join it.
tags                | An array of strings that can be used to further qualify a bundle name. These are often represented as versions e.g. "1.0.0-beta.1", "version1" etc.
uid                 | The unique identifier of a ConductR member within an `address`
uniqueAddress       | An object describing the unique address of a member of ConductR's cluster.


#### Failure

Other non 2xx status codes should also be treated as a failure.

## Receive bundle state events

Receive Server Sent Events (SSE) in relation to the bundle information changing.


### Request

```
GET /v2/bundles/events?events={event-name}
```

Field            | Description
-----------------|------------
event-type       | Optional. The type of the event to be filtered. Valid values see below.

The following event types can be specified as a query parameter to receive to the following events:

Event Type       | Events
-----------------|------------
config           | bundleConfigAdded, bundleConfigRemoved
error            | bundleErrorOccurred
execution        | bundleExecutionAdded, bundleExecutionChanged, bundleExecutionRemoved
info             | bundleInfoAdded, bundleInfoRemoved
installation     | bundleInstallationAdded, bundleInstallationChanged, bundleInstallationRemoved

These filtering parameters are also case insensitive.

#### Success

```
HTTP/1.1 200 OK
Content-Type: text/event-stream

data:73595ecbdad1f01a05db5304046b4ad5
event:bundleExecutionAdded

data:73595ecbdad1f01a05db5304046b4ad5
event:bundleExecutionRemoved
```

Heartbeat events are also emitted so that the connection is kept alive through http proxies. SSE heartbeats are empty lines.

#### Failure

Other status codes should also be treated as a failure.

## Query member state

Retrieve the current state of ConductR cluster members (otherwise known as the "core").

### Request

```
GET /v2/members
```

### Responses

#### Success

```json
HTTP/1.1 200 OK
Content-Type: application/json

{
  "selfNode": "{selfNode}",
  "selfNodeUid": "{selfNodeUid}",
  "members": [
    {
      "node": "{node}",
      "nodeUid": "{nodeUid}",
      "status": "{status}",
      "roles": {roles}
    }
  ],
  "unreachable": [ ]
}
```


##### Sections

Section             | Description
--------------------|------------
members             | The ConductR cluster members that are seen to be up.
unreachable         | The members who are seen to be unreachable.

##### Fields

Field               | Description
--------------------|------------
node                | The host of the member.
nodeUid             | The unique identifier of the member within the `node`.
roles               | An array of strings representing the roles that the ConductR cluster member is able to handle. Role names are user-supplied with exception to `web`.
selfNode            | The host of the member serving this response.
selfNodeUid         | The unique identifier of the member within the `selfNode`.
status              | `Joining`, `Up`, `Leaving`, `Exiting`, `Down` or `Removed` as per Akka Cluster's `MemberStatus`.


#### Failure

Other non 2xx status codes should also be treated as a failure.

### Events

A `GET` on the `/members/events` endpoint can be used to receive server sent events in relation to the above member information changing. The following event types are possible:

* `memberUp`
* `memberReachable`
* `memberUnreachable`
* `memberDown`

The "data" of the event represents a node.

Heartbeat events are also emitted so that the connection is kept alive through http proxies. SSE heartbeats are empty lines.


## Query agent state

Retrieve the current state of ConductR cluster agents.

### Request

```
GET /v2/agents
```

### Responses

#### Success

```json
HTTP/1.1 200 OK
Content-Type: application/json

[
  {
    "address": "{address}",
    "roles": {roles},
    "observedBy": {observed-by}
  }
]
```

##### Fields

Field               | Description
--------------------|------------
address             | A uri describing the agent node.
roles               | An array of strings representing the roles that the ConductR cluster agent is able to handle. Role names are user-supplied with exception to `web`.
observedBy          | An array of ConductR core members that the agent has connected to. An agent is generally connected to just one core member at a time, but it is possible for there to be zero, or more than one for a short period of time during re-connects.


#### Failure

Other non 2xx status codes should also be treated as a failure.

### Events

A `GET` on the `/agents/events` endpoint can be used to receive server sent events in relation to the above agent information changing. The following event types are possible:

* `agentAdded`
* `agentRemoved`

The "data" of the event represents an agent node.

Heartbeat events are also emitted so that the connection is kept alive through http proxies. SSE heartbeats are empty lines.



## Query logs by bundle

Request the log messages of a bundle. Log messages with the latest timestamp are going to be returned in a "tail" like fashion.

### Request

```
GET /v2/bundles/{bundleIdOrName}/logs?count={count}
```

Field            | Description
-----------------|------------
bundleIdOrName   | An existing bundle identifier, a shortened version of it (min 7 characters) or a non-ambigious name given to the bundle during loading.
count            | The number of log messages to return. Allowed values are 1 to 100.

### Responses

#### Success

```json
HTTP/1.1 200 OK
Content-Type: application/json

[
  {
    "timestamp": "2015-09-24T14:18:32.573Z",
    "host": "a1b29b193a22",
    "message": "[info] application - Signalled start to ConductR"
  }
]  
```

Field     | Description
----------|------------
timestamp | Timestamp of log messages expressed according to ISO 8601.
host      | Host name in which the messages has been logged.
message   | The log message.

#### Failure

```
HTTP/1.1 400 Bad Request

Invalid fetch size of {count} where the max is 100 for bundle '{bundleIdOrName}'
```

The given `count` parameter is not valid number between 1 and 100.

```
HTTP/1.1 404 Not found

No bundle found by the specified Bundle ID/name: '{bundleIdOrName}'
```

The supplied bundleIdOrName cannot be resolved to a bundle that has been loaded.

```
HTTP/1.1 500 Internal Server Error
```

An internal server error has occured while retrieving the log messages. Check if the logging infrastructure is up and running, e.g. if the bundle `conductr-elasticsearch` has been loaded and scaled to at least one instance.

## Query events by bundle

Request the events of a bundle. Events with the latest timestamp are going to be returned in a "tail" like fashion.

### Request

```
GET /v2/bundles/{bundleIdOrName}/events?count={count}
```

Field            | Description
-----------------|------------
bundleIdOrName   | An existing bundle identifier, a shortened version of it (min 7 characters) or a non-ambigious name given to the bundle during loading.
count            | The number of events to return.

### Responses

#### Success

```json
HTTP/1.1 200 OK
Content-Type: application/json

[
  {
    "timestamp": "2015-09-24T14:38:21.365Z",
    "event": "conductr.loadExecutor.bundleWritten",
    "description": "Bundle written: requestId=f77ac3df-8d40-48d5-bd48-c545dae7a5a2, bundleName=visualizer, bundleId=3e124250d0367abce72e68ee3c7f93b1"
  }
]  
```

Field       | Description
------------|------------
timestamp   | Timestamp of log messages expressed according to ISO 8601.
event       | Event name
description | Event description

#### Failure

The given `count` parameter is not valid number between 1 and 100.

```
HTTP/1.1 404 Not found

No bundle found by the specified Bundle ID/name: '{bundleIdOrName}'
```

The supplied bundleIdOrName cannot be resolved to a bundle that has been loaded.

```
HTTP/1.1 500 Internal Server Error
```

An internal server error has occured while retrieving the events. Check if the logging infrastructure is up and running, e.g. if the bundle `conductr-elasticsearch` has been loaded and scaled to at least one instance.
