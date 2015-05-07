# Typesafe ConductR %PLAY_VERSION%


## The Bundle API


The [ConductR bundle library](https://github.com/typesafehub/conductr-bundle-lib#typesafe-conductr-bundle-library) provides a convenient Scala, Akka and Play API over an underlying REST API. This document details the underlying REST API.

Two services are covered by the bundle API:

* Location lookup service
* Status service

## The Location Lookup Service

Resolve a service name expressed as path to a URL. The URL returned is to a service's proxy thereby insulating the caller from the actual address of a service. A proxy URL can resolve to any of a number of actual service addresses e.g. a customer service can be backed by two instances witin the cluster.

### Request

```
GET {SERVICE_LOCATOR}{service-name}
```

Field            | Description
-----------------|------------
SERVICE\_LOCATOR | The environment variable value of the same name. This environment variable translates to an http address e.g. `http://10.0.1.22:9008` given that ConductR is running on `10.0.1.22` with a service locator port bound to 9008.
service-name     | The name of the service required and expressed as a path e.g. `/customers`. Only the first portion of a path is considered e.g. `/customers/123` means that only `/customers` will be looked up. This approach permits most HTTP clients to be used with very little code change when compared to not using the service locator.

### Responses

##### Success

```
HTTP/1.1 307 Temporary Redirect
Location: {location-url}
Cache-Control: max-age={max-age}
```

Field        | Description
-------------|------------
location-url | The location of the requested service including any trailing parts to the path requested e.g. `/customers/123` would result in `http://10.0.1.22:8080/customers/123` supposing that ConductR's proxy service's address is `10.0.1.22`, and the endpoint service port is `8080`. If the protocol for the service was TCP then that will be reflected in the returned location's protocol field. For example looking up `/jms` may yield `tcp://10.0.1.22:61616` supposing that the Active/MQ JMS broker's service is offered on port 61616.
max-age      | The Time-To-Live (TTL) seconds before it is recommended to retain any previous value returned by this service. You should also evict any cached value if any subsequent request on the `location-url` fails.

##### Failure

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
SERVICE\_LOCATOR | The environment variable value of the same name. This environment variable translates to an http address e.g. `http://10.0.1.22:9008` given that ConductR is running on `10.0.1.22` with a service locator port bound to 9008.
service-name     | The name of the service required and expressed as a path e.g. `/customers`. Only the first portion of a path is considered e.g. `/customers/123` means that only `/customers` will be looked up. This approach permits most HTTP clients to be used with very little code change when compared to not using the service locator.

### Responses

##### Success

```
HTTP/1.1 204 No Content
```

The bundle is successfully signalled if any status code within the 2xx series is returned.

##### Failure

All other types of response outside of the 2xx status code range are an error, including no response at all i.e. use a timeout.

In the case of an error or timeout then your application or service should terminate.