# ConductR Migration Guide


This document describes what is required to move between version 1.1 to 2.0.

Here is a list of major areas to be considered when migrating:

* [Binary incompatibility](#Binary_incompatibility)
* [Topology change](#Topology_change)
* [Proxying](#Proxying)


## Binary incompatibility

_ConductR 1.1 and 2.0 are not binary compatible._

As such, a new cluster of ConductR 2.0 must be built in isolation from the version 1.1 of the cluster. Once the version 2.0 of the cluster is formed successfully and the application data is migrated over, cut-over traffic using DNS, load balancers, routers, etc.

## Topology change

ConductR 2.0 introduces the notion of ConductR Core and ConductR Agent.

ConductR Core holds the state of the cluster, and is responsible for decision making as to which ConductR Agent gets to run which bundles based on the resource available on the agent. The physical bundle files are stored on the ConductR Core's filesystem and replicated across the cluster.

ConductR Agent is responsible for running bundle processes. As such, the bundle roles which are previously specified are now to be specified within ConductR Agent.

The ConductR Core and ConductR Agent is available as 2 different packages. The [[installation instructions|Install]] describes the steps of installing ConductR Core and ConductR Agent on the same host. These steps is also applicable should ConductR Core and ConductR Agent installation on different host is desired.

## Node Peer Access

This distributed topology enables Agents to be managed by remote Cores. This requires nodes to be able to reach each other on ports 2552, 9005, 9007 and 9008 in addition the 9004 and 9006 ports previously required with 1.1. See also the port table in [[Subnets and Security Groups|Install#Subnets-and-Security-Groups]].

## Proxying

conductr-haproxy is now provided as a bundle with the "haproxy" role. When running conductr-haproxy you should specify a scale for the number of HAProxy services in your cluster. In addition the nodes that run the HAProxy service must have the "haproxy" role from a ConductR perspective.

### Amazon's ELB

Prior to 1.2, we recommended that you configure the ELB to poll the /bundles endpoint of the control protocol. Given conductr-haproxy you should now configured the ELB to poll the http://:9009/status. This endpoint will return an HTTP OK status when HAProxy has been correctly configured therefore leading to a more reliable configuration of the ELB. See [the cluster setup considerations document](ClusterSetupConsiderations#Cluster security considerations) for more information.

## Syslog

In additional to disabling Elasticsearch via configuration, a service lookup must now also be disabled. ConductR's default behavior is to locate the service endpoint for logging when required. Supposing that you are using RSYSLOG locally then ConductR Core's `conductr.ini` should contain the following:

```
  -Dcontrail.syslog.server.host=127.0.0.1
  -Dcontrail.syslog.server.port=514
  -Dcontrail.syslog.server.elasticsearch.enabled=off 
  -Dcontrail.syslog.server.service-locator.enabled=off
```

The same also goes for ConductR Agent's `conductr-agent.ini`.

## Service endpoint declaration

The services endpoint declaration defines the protocol, port, and/or path under which your service will be addressed to the outside world.

**From 1.2 onwards, the service endpoint declaration is deprecated in lieu of [[request ACL|AclConfiguration]].**

The [[request ACL|AclConfiguration]] in conjunction with [[dynamic proxy configuration|DynamicProxyConfiguration]] provides additional proxy mapping flexibility on top of what was previously provided by the service ports. This will allow operations to have a complete control on the proxy configuration.

### Migrating HTTP-based services

Some HTTP based service definition can be migrated by supplying its [[request ACL|AclConfiguration]] equivalent, while others require a customized HAProxy configuration.

#### Migrating services definition

The following HTTP based service definition can be migrated by supplying its [[request ACL|AclConfiguration]] equivalent.

##### Services mapped to root path

Given the following example where the service URI was "http://my-service", ConductR will interpret the first path component (`my-service`) as the service name for the purposes of service lookup. By default ConductR will then remove the first path component when rewriting the request. This means that your application or service will be deployed as root context or `/`.

###### Migrating build.sbt

Modify the `Endpoint` declared within the `BundleKeys.endpoints`.

From:

```
BundleKeys.endpoints := Map(
  "endpoint-label" -> Endpoint("http", 0, Set(URI("http://my-service")))
)
```

To:

```
BundleKeys.endpoints := Map(
  "endpoint-label" -> Endpoint("http", 0, "my-service",
    RequestAcl(Http("^/my-service".r -> "/"))
  )
)
```

###### Migrating bundle.conf

Modify the `endpoints` section declared within the `bundle.conf` as such.

From:

```
    endpoints         = {
      "endpoint-label" = {
        bind-protocol = "http"
        bind-port     = 0
        services      = ["http://my-service]
      }
    }
```

To:

```
    endpoints         = {
      "endpoint-label" = {
        bind-protocol = "http"
        bind-port     = 0
        service-name  = "my-service"
        acls          = [
          {
            http = {
              requests = [
                {
                  path-beg = "/my-service"
                  rewrite = "/"
                }
              ]
            }
          }
        ]
      }
    }
```



##### Services mapped with preserved path

The `preservePath` query parameter on a service declaration indicated that the context root path should be preserved when passing through ConductR's proxy. Given the service URI `http://my-service?preservePath`, the service is accessible from the outside world on HTTP port allocated by ConductR under `/my-sevice` context.

This behaviour can now be declared with a request ACL.

###### Migrating build.sbt

Modify the `Endpoint` declared within the `BundleKeys.endpoints`.

From:

```
BundleKeys.endpoints := Map(
  "endpoint-label" -> Endpoint("http", 0, Set(URI("http://my-service?preservePath")))
)
```

To:

```
BundleKeys.endpoints := Map(
  "endpoint-label" -> Endpoint("http", 0, "my-service",
    RequestAcl(Http("^/my-service".r))
  )
)
```

###### Migrating bundle.conf

Modify the `endpoints` section declared within the `bundle.conf` as such.

From:

```
    endpoints         = {
      "endpoint-label" = {
        bind-protocol = "http"
        bind-port     = 0
        services      = ["http://my-service?preservePath"]
      }
    }
```

To:

```
    endpoints         = {
      "endpoint-label" = {
        bind-protocol = "http"
        bind-port     = 0
        service-name  = "my-service"
        acls          = [
          {
            http = {
              requests = [
                {
                  path-beg = "/my-service"
                }
              ]
            }
          }
        ]
      }
    }
```

#### Supplying customized HAProxy template configuration

The following services definition will require a customized HAProxy configuration.

The [[Dynamic proxy configuration|DynamicProxyConfiguration]] describes the steps required to generate a customized HAProxy configuration.

Generating a customized HAProxy configuration will rely on the [[ifAcl|DynamicProxyConfiguration#ifAcl]] directive which accepts `system`, `systemVersion`, and `endpointLabel` as input argument.

When declaring bundle settings via `build.sbt`, unless declared explicitly, the `system` and `systemVersion` settings may fallback to a default value as described by [sbt-conductr settings](https://github.com/typesafehub/sbt-conductr#bundle-settings).


##### Customized port number

A service may declare an endpoint exposed on a custom port number. Given the service URI `http://:6789/my-service`, the service is accessible from the outside world on HTTP port `6789` under `/my-service` context.

###### Migrating build.sbt

Modify the `Endpoint` declared within the `BundleKeys.endpoints`.

From:

```
name := "my-bundle"
BundleKeys.compatibilityVersion := "1.1"
BundleKeys.endpoints := Map(
  "endpoint-label" -> Endpoint("http", 0, Set(URI("http://:6789/my-service")))
)

```

To:

```
BundleKeys.endpoints := Map(
  "endpoint-label" -> Endpoint("http", 0, "my-service",
    RequestAcl(Http("^/my-service".r -> "/"))
  )
)
```


###### Migrating bundle.conf

Modify the `endpoints` section declared within the `bundle.conf` as such.

From:

```
    system             = "my-bundle"
    systemVersion      = "1.1"
    endpoints          = {
      "endpoint-label" = {
        bind-protocol  = "http"
        bind-port      = 0
        services       = ["http://:6789/my-service"]
      }
    }
```

To:

```
    endpoints         = {
      "endpoint-label" = {
        bind-protocol = "http"
        bind-port     = 0
        service-name  = "my-service"
        acls          = [
          {
            http = {
              requests = [
                {
                  path-beg = "/my-service"
                  rewrite = "/"
                }
              ]
            }
          }
        ]
      }
    }
```

###### Supplying customized HAProxy configuration

The [[Dynamic proxy configuration|DynamicProxyConfiguration]] describes the steps required to generate a customized HAProxy configuration.

Populate the "Custom Endpoint Configuration" section described in the [[populate custom HAProxy configuration template|DynamicProxyConfiguration#Populate-the-custom-HAProxy-configuration-template]] step as such:

```
{{#eachAcls bundles defaultHttpPort=9443}}
  {{#ifAcl 'my-bundle' '1.1' 'endpoint-label'}}
frontend my_www_frontend
  bind {{haproxyHost}}:6789
  mode http
  acl my_www_path_match path_beg /my-service
  use_backend my_www_backend if my_www_path_match

backend my_www_backend
  mode http
  reqrep ^([^\ :]*)\ /my-service/?(.*) \1\ /\2
    {{#eachBackendServer}}
  server {{serverName}} {{host}}:{{port}} maxconn 1024
    {{/eachBackendServer}}
  {{/ifAcl}}
{{/eachAcls}}

```

The snippet above will expose other HTTP endpoint at port `9443`, this value can be replaced with any valid port number accordingly.

##### Host header match

A service may declare an endpoint exposed on a particular host address. Given the service URI `http://www.acme.com`, the service is accessible from the outside world on HTTP port `80` when the HTTP host header is `www.acme.com`.

###### Migrating build.sbt

Modify the `Endpoint` declared within the `BundleKeys.endpoints`.

From:

```
name := "my-bundle"
BundleKeys.compatibilityVersion := "1.1"
BundleKeys.endpoints := Map(
  "endpoint-label" -> Endpoint("http", 0, Set(URI("http://www.acme.com")))
)

```

To:

```
BundleKeys.endpoints := Map(
  "endpoint-label" -> Endpoint("http", 0, "my-service",
    RequestAcl(Http("^/".r))
  )
)
```

Where `my-service` is the service name for the endpoint for the purpose of service lookup.

###### Migrating bundle.conf

Modify the `endpoints` section declared within the `bundle.conf` as such.

From:

```
    system             = "my-bundle"
    systemVersion      = "1.1"
    endpoints          = {
      "endpoint-label" = {
        bind-protocol  = "http"
        bind-port      = 0
        services       = ["http://www.acme.com"]
      }
    }
```

To:

```
    endpoints         = {
      "endpoint-label" = {
        bind-protocol = "http"
        bind-port     = 0
        service-name  = "my-service"
        acls          = [
          {
            http = {
              requests = [
                {
                  path-beg = "/"
                }
              ]
            }
          }
        ]
      }
    }
```

Where `my-service` is the service name for the endpoint for the purpose of service lookup.


###### Supplying customized HAProxy configuration

The [[Dynamic proxy configuration|DynamicProxyConfiguration]] describes the steps required to generate a customized HAProxy configuration.

Populate the "Custom Endpoint Configuration" section described in the [[populate custom HAProxy configuration template|DynamicProxyConfiguration#Populate-the-custom-HAProxy-configuration-template]] step as such:

```
{{#eachAcls bundles defaultHttpPort=9443}}
  {{#ifAcl 'my-bundle' '1.1' 'endpoint-label'}}
frontend my_www_frontend
  bind {{haproxyHost}}:80
  mode http
  acl my_www_host_match hdr(host) -i www.acme.com
  use_backend my_www_backend if my_www_host_match

backend my_www_backend
  mode http
    {{#eachBackendServer}}
  server {{serverName}} {{host}}:{{port}} maxconn 1024
    {{/eachBackendServer}}
  {{/ifAcl}}
{{/eachAcls}}

```

The snippet above will expose other HTTP endpoint at port `9443`, this value can be replaced with any valid port number accordingly.

### Migrating TCP-based services

TCP based service can be migrated by supplying its [[request ACL|AclConfiguration]] equivalent.

#### Migrating build.sbt

Modify the `Endpoint` declared within the `BundleKeys.endpoints`.

From:

```
BundleKeys.endpoints := Map(
  "logbroker" -> Endpoint("tcp", 0, Set(URI("http://:6379/redis")))
)
```

To:

```
BundleKeys.endpoints := Map(
  "logbroker" -> Endpoint("tcp", 0, "redis", RequestAcl(Tcp(6379)))
)
```

#### Migrating bundle.conf

Modify the `endpoints` section declared within the `bundle.conf` as such.

From:

```
    endpoints         = {
      "logbroker" = {
        bind-protocol = "tcp"
        bind-port     = 0
        services      = ["http://:6379/redis"]
      }
    }
```

To:

```
    endpoints         = {
      "logbroker" = {
        bind-protocol = "tcp"
        bind-port     = 0
        service-name  = "redis"
        acls          = [
          {
            tcp = {
              requests = [6379]
            }
          }
        ]
      }
    }
```
