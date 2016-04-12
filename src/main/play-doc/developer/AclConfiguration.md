# ACL configuration

ConductR provides a DSL for the application to declare request ACLs for HTTP, TCP, or UDP based endpoints.

These request ACLs are used as the instructions to expose the HTTP and TCP based endpoints via proxy during deployment time. Operations may choose to use the request ACLs supplied by the developer as-is, or provide their own customised configuration for these endpoints. This is discussed at a greater length in [[Dynamic proxy configuration|DynamicProxyConfiguration]].

The DSL is provided as part of the [sbt-conductr](https://github.com/typesafehub/sbt-conductr) plugin which is required when packaging the application as a bundle.

## Declaring HTTP-based request ACLs

HTTP based request ACLs allows exposing endpoints that matches HTTP request based on the following criteria.

### Exact path match

Looks for an exact match given a particular HTTP path. If the exact match is found, the request is then relayed from the proxy to the endpoint.

In this example the request `/health` will be relayed to the endpoint:

```
BundleKeys.endpoints := Map(
  "healthcheck" -> Endpoint("http", 0, "healthcheck",
    RequestAcl(
      Http(
        "/health"
      )
    )
  )
```

The criteria can be further refined to match against a particular HTTP method:

```
BundleKeys.endpoints := Map(
  "healthcheck" -> Endpoint("http", 0, "healthcheck",
    RequestAcl(
      Http(
        "GET" -> "/health"
      )
    )
  )
```


It is also possible to specify `rewrite`. In the following example, all the incoming request on `/health` will be rewritten to `/` before it's relayed to the endpoint.

```
BundleKeys.endpoints := Map(
  "healthcheck" -> Endpoint("http", 0, "healthcheck",
    RequestAcl(
      Http(
        "/health" -> "/"
      )
    )
  )
```

The HTTP method match can be combined with the `rewrite` as such:

```
BundleKeys.endpoints := Map(
  "healthcheck" -> Endpoint("http", 0, "healthcheck",
    RequestAcl(
      Http(
        "GET" -> "/health" -> "/"
      )
    )
  )
```


### Path begin match

Looks for a match the HTTP request that starts with a particular path. If the match is found, the request is then relayed from the proxy to the endpoint.


In this example all request that starts with the path `/orders` will be relayed to the endpoint:

```
BundleKeys.endpoints := Map(
  "order" -> Endpoint("http", 0, "order",
    RequestAcl(
      Http(
        "^/orders".r
      )
    )
  )
```

The criteria can be further refined to match against a particular HTTP method:

```
BundleKeys.endpoints := Map(
  "order" -> Endpoint("http", 0, "order",
    RequestAcl(
      Http(
        "GET" -> "^/orders".r
      )
    )
  )
```


It is also possible to specify `rewrite`. In the following example, all the incoming request on `/health` will be rewritten to `/` before it's relayed to the endpoint.

```
BundleKeys.endpoints := Map(
  "order" -> Endpoint("http", 0, "order",
    RequestAcl(
      Http(
        "^/orders".r -> "/"
      )
    )
  )
```


The HTTP method match can be combined with the `rewrite` as such:

```
BundleKeys.endpoints := Map(
  "order" -> Endpoint("http", 0, "order",
    RequestAcl(
      Http(
        "GET" -> "^/orders".r -> "/"
      )
    )
  )
```


### Regular expression path match

Looks for a match the HTTP request path that matches a certain regular expression pattern. If the match is found, the request is then relayed from the proxy to the endpoint.

In this example, the HTTP request path that matches `^/users/(.*)/friends/(.*)$` will be relayed to the endpoint:

```
BundleKeys.endpoints := Map(
  "friend" -> Endpoint("http", 0, "friend",
    RequestAcl(
      Http(
        "^/users/(.*)/friends/(.*)$".r
      )
    )
  )
```

The criteria can be further refined to match against a particular HTTP method:

```
BundleKeys.endpoints := Map(
  "friend" -> Endpoint("http", 0, "friend",
    RequestAcl(
      Http(
        "GET" -> "^/users/(.*)/friends/(.*)$".r
      )
    )
  )
```

It is also possible to specify `rewrite`:

```
BundleKeys.endpoints := Map(
  "friend" -> Endpoint("http", 0, "friend",
    RequestAcl(
      Http(
        "^/users/(.*)/friends/(.*)$".r -> "/user/\\1-\\2/friend"
      )
    )
  )
```

In the example above `\\1` and `\\2` refers to the 1st and 2nd matching regular expression respectively. As such, the http request path `/users/sally/friend/joe` will be rewritten as `/user/sally-joe/friend` before relayed to the endpoint.

The HTTP method match can be combined with the `rewrite` as such:

```
BundleKeys.endpoints := Map(
  "friend" -> Endpoint("http", 0, "friend",
    RequestAcl(
      Http(
        "GET" -> "^/users/(.*)/friends/(.*)$".r -> "/user/\\1-\\2/friend"
      )
    )
  )
```

## Declaring TCP-based request ACLs

TCP based request ACLs allows exposing TCP ports, e.g.

```
BundleKeys.endpoints := Map(
  "ptunnel" -> Endpoint("tcp", 0, "ptunnel",
    RequestAcl(
      Tcp(
        3303, 12101
      )
    )
  )
)

```

In the example above, the TCP port `3303` and `12101` will be exposed via the proxy and the requests to these ports will be relayed to the endpoint.

## Declaring UDP-based request ACLs

UDP based request ACLs allows declaring UDP ports, e.g.

```
BundleKeys.endpoints := Map(
  "binports" -> Endpoint("udp", 0, "binports",
    RequestAcl(
      Udp(
        3303, 12101
      )
    )
  )
)

```

UDP based request will not be exposed via proxy. ConductR makes use of HAProxy to proxy the endpoints, and at the time of writing HAProxy does not offer support for UDP.
