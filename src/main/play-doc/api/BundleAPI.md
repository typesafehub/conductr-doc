# Lightbend ConductR %PLAY_VERSION%


## The Bundle API


The [ConductR bundle library](https://github.com/typesafehub/conductr-lib#typesafe-conductr-bundle-library) provides a convenient Scala, Akka and Play API over an underlying REST API. This document details the underlying REST API.

The following services are covered by the bundle API:

* [Location lookup service](#The-Location-Lookup-Service)
* [List hosts service](#The-List-Hosts-Service)
* [Find host service](#The-Find-Host-Service)
* [Service Server Sent Events](#The-Service-Server-Sent-Event)
* [Status service](#The-Status-Service)

## The Location Lookup Service

Resolve a service name expressed as path to a URL. The URL returned is to the nearest service of the cluster in relation to the caller's IP. Service lookup insulates the caller from the actual address of a service, and therefore allows these other services to move around.

### Request

```
GET {SERVICE_LOCATOR}/services/{service-name}
```

Field            | Description
-----------------|------------
SERVICE\_LOCATOR | The environment variable value of the same name. This environment variable translates to an http address e.g. `http://10.0.1.22:9008` given that ConductR is running on `10.0.1.22` with a service locator port bound to 9008.
service-name     | The name of the service required and expressed as a path e.g. `/customers`. Only the first portion of a path is considered e.g. `/customers/123` means that only `/customers` will be used as the service name. This approach permits most HTTP clients to be used with very little code change when compared to not using the service locator.

### Responses

#### Success

```
HTTP/1.1 307 Temporary Redirect
Location: {location-url}
Cache-Control: max-age={max-age}
```

Field        | Description
-------------|------------
location-url | The location of the requested service less the first component which is the service name, but including any trailing parts to the path requested e.g. `/customers/123` would result in `http://10.0.1.22:10121/123` supposing that service's host address is `10.0.1.22`, and the endpoint service host port is `10121`. If the protocol for the service was TCP then that will be reflected in the returned location's protocol field. For example looking up `/jms` may yield `tcp://10.0.1.22:10121`.
max-age      | The Time-To-Live (TTL) seconds before it is recommended to retain any previous value returned by this service. You should also evict any cached value if any subsequent request on the `location-url` fails.

#### Failure

```
HTTP/1.1 404 Not Found
```

The service requested cannot be found as it is unknown to ConductR.

Other status codes should also be treated as a failure.

## The List Hosts Service

Returns a list of host and port of the running service given a service name.

### Request

```
GET {SERVICE_LOCATOR}/service-hosts{service-name}
```

Field            | Description
-----------------|------------
SERVICE\_LOCATOR | The environment variable value of the same name. This environment variable translates to an http address e.g. `http://10.0.1.22:9008` given that ConductR is running on `10.0.1.22` with a service locator port bound to 9008.
service-name     | The name of the service required and expressed as a path e.g. `/customers`.

#### Success

```json
HTTP/1.1 200 OK
Content-Type: application/json

[
  "10.11.23.22:7001",
  "10.11.23.22:7002"
]
```

Where `10.11.23.22:7001` and `10.11.23.22:7002` are the host and host port of the service which is currently running.

#### Success - Empty Result

```json
HTTP/1.1 200 OK
Content-Type: application/json

[]
```

Either the service has not been started, or the service requested cannot be found as it is unknown to ConductR.

#### Failure

Other status codes should also be treated as a failure.

### Events

A `GET` on the `/service-hosts{service-name}/events` endpoint can be used to receive server sent events in relation to the service execution state changing. The following event types are possible:
 
* `running`
* `stopped`

The "data" of the event represents the host and port where the service has stopped or is running.

An `event` parameter may be supplied to as a repeated set of query parameters with the value representing the filtering of specific event types. Partial event type names are permitted e.g.: `GET` `/service-hosts{service-name}/events?event=running` would filter the emission of `running` only. These filtering parameters are also case insensitive.

Heartbeat events are also emitted so that the connection is kept alive through http proxies. SSE heartbeats are empty lines.

## The Find Host Service

Returns the host and port of the running service given a service name and host ip.

### Request

```
GET {SERVICE_LOCATOR}/service-hosts{service-name}/{host-ip}
```

Field            | Description
-----------------|------------
SERVICE\_LOCATOR | The environment variable value of the same name. This environment variable translates to an http address e.g. `http://10.0.1.22:9008` given that ConductR is running on `10.0.1.22` with a service locator port bound to 9008.
service-name     | The name of the service required and expressed as a path e.g. `/customers`.
host-ip          | The ip address of the service to be queried.

#### Success

```json
HTTP/1.1 200 OK
Content-Type: application/json

"10.11.23.22:7001"
```

Where `10.11.23.22:7001` is the host and host port of the service which is currently running.

#### Failure

```
HTTP/1.1 404 Not Found
```

Either the service is not started, or the service requested cannot be found as it is unknown to ConductR.

Other status codes should also be treated as a failure.

## The Service Server Sent Event

Returns stream of Server Sent Event given a service name. The Server Sent Event will be invoked when a particular service is running or stopped.

### Request

```
GET {SERVICE_LOCATOR}/service-hosts{service-name}/events?event={event-name}
```

Field            | Description
-----------------|------------
SERVICE\_LOCATOR | The environment variable value of the same name. This environment variable translates to an http address e.g. `http://10.0.1.22:9008` given that ConductR is running on `10.0.1.22` with a service locator port bound to 9008.
service-name     | The name of the service required and expressed as a path e.g. `/customers`.
event-name       | Optional. The name of the event to be filtered. Valid values are either `running` or `stopped`.

#### Success

```
HTTP/1.1 200 OK
Content-Type: text/event-stream

event:running
data:10.11.23.22:7001

event:stopped
data:10.11.23.22:7001
```

Where the two events above indicates the service being running and stopped on `10.11.23.22:7001`.

#### Failure

```
HTTP/1.1 404 Not Found
```

The service requested cannot be found as it is unknown to ConductR.

Other status codes should also be treated as a failure.

## The Status Service

When your application or service has performed its initialization and is satisfied that it is ready to operate, it needs to signal ConductR of this. ConductR will then proceed with other activities such as scaling, proxying and so forth.

### Request

```
PUT {CONDUCTR_STATUS}/bundles/{bundle-id}?isStarted=true
```

Field            | Description
-----------------|------------
CONDUCTR\_STATUS | The environment variable value of the same name. This environment variable translates to an http address e.g. `http://10.0.1.22:9008` given that ConductR is running on `10.0.1.22` with a service locator port bound to 9008.
bundle-id        | The id of the bundle to report.

### Responses

#### Success

```
HTTP/1.1 204 No Content
```

The bundle is successfully signalled if any status code within the 2xx series is returned.

#### Failure

All other types of response outside of the 2xx status code range are an error, including no response at all i.e. use a timeout.

In the case of an error or timeout then your application or service should terminate.
