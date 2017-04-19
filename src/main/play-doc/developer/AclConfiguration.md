# ACL configuration

ConductR Access Control Lists (ACLs) for HTTP, TCP, or UDP based endpoints for the purposes of conveying how an endpoint should be presented at a proxy.

These request ACLs are used as the instructions to expose the HTTP and TCP based endpoints via proxy during deployment time. Operations may choose to use the request ACLs supplied by the developer as-is, or provide their own customized configuration for these endpoints. This is discussed at a greater length in [[Dynamic proxy configuration|DynamicProxyConfiguration]].

## Declaring HTTP-based request ACLs

HTTP based request ACLs allows exposing endpoints that matches HTTP request based on the following criteria.

### Exact path match

Looks for an exact match given a particular HTTP path. If the exact match is found, the request is then relayed from the proxy to the endpoint.

In this example the request `/health` will be relayed to the endpoint:

```
    endpoints = {
      "healthcheck" = {
        bind-protocol = "http"
        bind-port     = 0
        service-name  = "healthcheck"
        acls          = [
          {
            http = {
              requests = [
                {
                  path = "/health"
                }
              ]
            }
          }
        ]
      }
    }
```

...or when using the sbt-conductr plugin:

```
BundleKeys.endpoints := Map(
  "healthcheck" -> Endpoint("http", 0, "healthcheck",
    RequestAcl(
      Http(
        "/health"
      )
    )
  )
)
```

The criteria can be further refined to match against a particular HTTP method:

```
    endpoints = {
      "healthcheck" = {
        bind-protocol = "http"
        bind-port     = 0
        service-name  = "healthcheck"
        acls          = [
          {
            http = {
              requests = [
                {
                  path = "/health"
                  method = "GET"
                }
              ]
            }
          }
        ]
      }
    }
```

...or when using the sbt-conductr plugin:

```
BundleKeys.endpoints := Map(
  "healthcheck" -> Endpoint("http", 0, "healthcheck",
    RequestAcl(
      Http(
        "GET" -> "/health"
      )
    )
  )
)
```


It is also possible to specify `rewrite`. In the following example, all the incoming request on `/health` will be rewritten to `/` before it's relayed to the endpoint.

```
    endpoints = {
      "healthcheck" = {
        bind-protocol = "http"
        bind-port     = 0
        service-name  = "healthcheck"
        acls          = [
          {
            http = {
              requests = [
                {
                  path = "/health"
                  rewrite = "/"
                }
              ]
            }
          }
        ]
      }
    }
```

...or when using the sbt-conductr plugin:

```
BundleKeys.endpoints := Map(
  "healthcheck" -> Endpoint("http", 0, "healthcheck",
    RequestAcl(
      Http(
        "/health" -> "/"
      )
    )
  )
)
```

The HTTP method match can be combined with the `rewrite` as such:

```
    endpoints = {
      "healthcheck" = {
        bind-protocol = "http"
        bind-port     = 0
        service-name  = "healthcheck"
        acls          = [
          {
            http = {
              requests = [
                {
                  path = "/health"
                  method = "GET"
                  rewrite = "/"
                }
              ]
            }
          }
        ]
      }
    }
```

...or when using the sbt-conductr plugin:

```
BundleKeys.endpoints := Map(
  "healthcheck" -> Endpoint("http", 0, "healthcheck",
    RequestAcl(
      Http(
        "GET" -> "/health" -> "/"
      )
    )
  )
)
```

### Path begin match

Looks for a match the HTTP request that starts with a particular path. If the match is found, the request is then relayed from the proxy to the endpoint.


In this example all request that starts with the path `/orders` will be relayed to the endpoint:

```
    endpoints = {
      "order" = {
        bind-protocol = "http"
        bind-port     = 0
        service-name  = "order"
        acls          = [
          {
            http = {
              requests = [
                {
                  path-beg = "/orders"
                }
              ]
            }
          }
        ]
      }
    }
```

...or when using the sbt-conductr plugin:

```
BundleKeys.endpoints := Map(
  "order" -> Endpoint("http", 0, "order",
    RequestAcl(
      Http(
        "^/orders".r
      )
    )
  )
)
```

### Regular expression path match

Looks for a match the HTTP request path that matches a certain regular expression pattern. If the match is found, the request is then relayed from the proxy to the endpoint.

In this example, the HTTP request path that matches `^/users/(.*)/friends/(.*)$` will be relayed to the endpoint:

```
    endpoints = {
      "friend" = {
        bind-protocol = "http"
        bind-port     = 0
        service-name  = "friend"
        acls          = [
          {
            http = {
              requests = [
                {
                  path-regex = "^/users/(.*)/friends/(.*)$"
                }
              ]
            }
          }
        ]
      }
    }
```

...or when using the sbt-conductr plugin:

```
BundleKeys.endpoints := Map(
  "friend" -> Endpoint("http", 0, "friend",
    RequestAcl(
      Http(
        "^/users/(.*)/friends/(.*)$".r
      )
    )
  )
)
```

When rewriting paths, captures may be used:

```
    endpoints = {
      "friend" = {
        bind-protocol = "http"
        bind-port     = 0
        service-name  = "friend"
        acls          = [
          {
            http = {
              requests = [
                {
                  path-regex = "^/users/(.*)/friends/(.*)$"
                  rewrite = "/user/\1-\2/friend"
                }
              ]
            }
          }
        ]
      }
    }
```

...or when using the sbt-conductr plugin:

```
BundleKeys.endpoints := Map(
  "friend" -> Endpoint("http", 0, "friend",
    RequestAcl(
      Http(
        "^/users/(.*)/friends/(.*)$".r -> """/user/\1-\2/friend"""
      )
    )
  )
)
```

In the example above `\1` and `\2` refers to the 1st and 2nd matching regular expression respectively. As such, the http request path `/users/sally/friend/joe` will be rewritten as `/user/sally-joe/friend` before relayed to the endpoint.

## Declaring TCP-based request ACLs

TCP based request ACLs allows exposing TCP ports, e.g.

```
    endpoints = {
      "ptunnel" = {
        bind-protocol = "tcp"
        bind-port     = 0
        service-name  = "ptunnel"
        acls          = [
          {
            tcp = {
              requests = [3303, 12101]
            }
          }
        ]
      }
    }
```

...or when using the sbt-conductr plugin:

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

> Note that while UDP endpoints may be declared for proxying, HAProxy does not support proxying at this point.

UDP based request ACLs allows declaring UDP ports, e.g.

```
    endpoints = {
      "binports" = {
        bind-protocol = "udp"
        bind-port     = 0
        service-name  = "binports"
        acls          = [
          {
            udp = {
              requests = [3303, 12101]
            }
          }
        ]
      }
    }
```

...or when using the sbt-conductr plugin:

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

## Multiple request paths

Multiple request paths may also be provided for each protocol family (each will be routed to the same endpoint at your service). Here's an example using Http:

```
    endpoints = {
      "healthcheck" = {
        bind-protocol = "http"
        bind-port     = 0
        service-name  = "healthcheck"
        acls          = [
          {
            http = {
              requests = [
                {
                  path = "/health"
                },
                {
                  path = "/v2/health"
                }
              ]
            }
          }
        ]
      }
    }
```

...or when using the sbt-conductr plugin:

```
BundleKeys.endpoints := Map(
  "healthcheck" -> Endpoint("http", 0, "healthcheck",
    RequestAcl(
      Http(
        "/health",
        "/v2/health"
      )
    )
  )
)
```
