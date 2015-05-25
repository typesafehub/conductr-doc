# Typesafe ConductR %PLAY_VERSION%


## The Control API

ConductR's control protocol is RESTful and has the following functional scope:

* [load a bundle and optionally its configuration](#Load-a-bundle)
* [scale a bundle i.e. starting it and possibly stopping it](#Scale-a-bundle)
* [unload a bundle](#Unload-a-bundle)
* [query bundle state](#Query-bundle-state)
* query member state
* subscribe to Server Sent Events (SSEs) regarding state changes to bundles and members

## Load a bundle

Request an upload a bundle and optionally, its configuration to ConductR. 

Here is an example of uploading a Visualizer bundle without any configuration. [Curl](http://curl.haxx.se/) is being used.

```
curl \
  --form system=sys \
  --form bundleName=http-server \
  --form nrOfCpus=2 \
  --form memory=104857600 \
  --form diskSpace=104857600 \
  --form roles=web-server \
  --form bundle=@visualizer/target/bundle/visualizer-0.1.0-cfe2a36795fd78507c4d2b5817152ae449e4acd9d5ea94d1f604d2c11417e40f.zip \
  http://localhost:9005/bundles
```

### Request

```
POST /bundles
```

The following fields are provided as multipart/form-data fields:

Field            | Description
-----------------|------------
system           | As per its equivalent property in [bundle.conf](Bundles.html)
bundleName       | As per its equivalent property in [bundle.conf](Bundles.html)
nrOfCpus         | As per its equivalent property in [bundle.conf](Bundles.html)
memory           | As per its equivalent property in [bundle.conf](Bundles.html)
roles            | As per its equivalent property in [bundle.conf](Bundles.html)
bundle           | The file that is the bundle. The filename is important with its hex digest string and is required to be consistent with the SHA-256 hash of the bundle's contents. Any inconsistency between the hashes will result in the load being rejected.
configuration    | Optional. Similar in form to the bundle, only that is the file that describes the configuration. Again any inconsistency between the hex digest string in the filename, and the SHA-256 digest of the actual contents will result in the load being rejected.

### Responses

#### Success

```
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
PUT /bundles/{bundleIdOrName}?scale={scale}
```

Field            | Description
-----------------|------------
bundleIdOrName   | An existing bundle identifier, a shortened version of it (min 7 characters) or a non-ambigious name given to the bundle during loading.
scale            | The number of instances of the bundle to start. A scale value of 0 indicates that all instances should be stopped.

### Responses

#### Success

```
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
DELETE /bundles/{bundleIdOrName}
```

Field            | Description
-----------------|------------
bundleIdOrName   | An existing bundle identifier, a shortened version of it (min 7 characters) or a non-ambigious name given to the bundle during loading.

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
GET /bundles
```

### Responses

#### Success

```
HTTP/1.1 200 OK
Content-Type: application/json

[]
```

There are no bundles JSON array is empty.

```
HTTP/1.1 200 OK
Content-Type: application/json

[
  {
    "bundleId": "{bundleId}",
    "bundleDigest": "{bundleDigest}",
    "configurationDigest", "{configurationDigest}",
    "attributes": {
      "system": "{system}",
      "nrOfCpus": {nrOfCpus},
      "memory": {memory},
      "diskSpace": {diskSpace},
      "roles": {roles},
      "bundleName": "{bundleName}"
    },
    "bundleConfig": {
      "endpoints": {
        "{endpoint-name}": {
          "bindProtocol": "{bindProtocol}",
          "services": {services}
        }
      }
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
bundleExecutions    | Bundle executions describes what is running and where in the context of a bundle and so will not be present when there are none running.
bundleInstallations | Bundle installations describe where the bundle has been replicated i.e. written to a file system. The array may be empty in the case where the first replication has not yet completed.
endpoints           | The endpoint declaration of a bundle component expressed as an object.

##### Fields

Field               | Description
--------------------|------------
address             | The location of a ConductR member.
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
isStarted           | `true` if the bundle has signalled that it has started successfully, `false` if it is in the process of starting up.
memory              | The amount of memory required when a bundle is running. Values are expressed in bytes.
nrOfCpus            | The number of cpus required when a bundle is running. This value can be expressed as a fraction.
roles               | An array of strings representing the roles that a bundle plays in a ConductR cluster. These roles are matched for eligibility with cluster members when searching for one to load and scale on. Only cluster members with matching roles will be selected.
services            | An array of string URIs providing the addresses for a given endpoint's proxying.
system              | The name of a system that the bundle belongs to. Systems are strings that may be used by a number of bundles in order to associate them. ConductR provides a guarantee that bundles belonging to the same system are started with the first one in isolation to the starting of the rest. This behavior can be leverage to form clusters where the first node must form the cluster and other nodes may then join it.
uid                 | The unique identifier of a ConductR member within an `address`
uniqueAddress       | An object describing the unique address of a member of ConductR's cluster.


#### Failure

Other non 2xx status codes should also be treated as a failure.

## Query member state

Retrieve the current state of ConductR cluster members.

### Request

```
GET /members
```

### Responses

#### Success

```
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
nodeUid             | The unique identifier of the member within the `node`.`
roles               | An array of strings representing the roles that the ConductR cluster member is able to handle. Role names are user-supplied with exception to `all-conductrs`. The latter means that the member will accept any role for the purposes of scheduling a bundle's loading or execution.
selfNode            | The host of the member serving this response.
selfNodeUid         | The unique identifier of the member within the `selfNode`.`
status              | `Joining`, `Up`, `Leaving`, `Exiting`, `Down` or `Removed` as per Akka Cluster's `MemberStatus`.


#### Failure

Other non 2xx status codes should also be treated as a failure.