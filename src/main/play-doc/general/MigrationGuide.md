# ConductR Migration Guide

This document describes what is required to move between version 2.0 and 2.1.

* [Cluster incompatibility](#Cluster_incompatibility);
* [Production Suite Licensing](#Production_Suite_Licensing); and
* [Bundle endpoints](#Bundle_endpoints).

## Cluster incompatibility

ConductR 2.1's cluster protocol is not compatible with 2.0 or 1.1. As such, you will need to install ConductR 2.1 on to a new set of machines.

## Production Suite Licensing

A license is now required to run ConductR 2.1 onwards. Not having a license will restrict you to just one ConductR agent.

### Obtain license

To obtain a license please download the latest CLI, type `conduct load-license` and follow the instructions it provides. Once authenticated at lightbend.com, `load-license` will download your license, connect to the ConductR cluster via either by setting the environment variable `CONDUCTR_HOST` or by providing the CLI option `--host` and then upload the license. You can invoke `conduct load-license` any number of times, changing the ConductR IP address to upload to accordingly.

### Verify license

To see the license you have, please type the `conduct info` command. This command has been upgraded to display license information. The command will first connect with your ConductR cluster via either the `CONDUCTR_HOST` or `--host`, and then display your license.

## Bundle endpoints

Bundle endpoints using `services` properties are no longer accepted. `services` were deprecated in 2.0 in favor of ACLs (Access Control Lists).

ACLs have been introduced so that you may have more control over how an endpoint is presented by a proxy. With the previous approach that used `services`, we infered service names for service locator lookups from the first path component, and introduced query parameters in order to declare how a path should be re-written if required. Over time, we found the `services` expression restrictive and unintuitive. The ACL approached (introduced in ConductR 2.0) provides a great deal of flexibility and is more explicit in its declarations.

### How to migrate from `services` to `acls`

Suppose you have an endpoint declared in your `bundle.conf` as:

```
    endpoints = {
      "hello" = {
        bind-protocol = "http"
        bind-port     = 0
        services      = ["http://:9000/hello", "http://:9000/api/hello?preservePath"]
      },
      "akka-remote" = {
        bind-protocol = "tcp"
        bind-port     = 0
        services      = []
      }
    }
```

... which is the equivalent in sbt:

```
BundleKeys.endpoints := Map(
  "hello" -> Endpoint("http", 0, Set(URI("http://:9000/hello"), URI("http://:9000/api/hello?preservePath"))),
  "akka-remote" -> Endpoint("tcp", 0)
)
```

There are three things that are required to be changed:

* service name;
* replace `services` expressions with `acls` expressions; and
* rewrite paths for Play and other web servers.

Firstly the service name is no longer inferred from the first component of the path. Therefore a `service-name` property is now required if you would you like your service to be discoverable by other services (which you also may not want). 

Secondly the `services` property is replaced with ACLs which are rules to be applied when creating a proxy configuration. Note that with HTTP, ports are no longer expressed here as they are considered an runtime concern, and not a build time one. By default, we have configured HAProxy to provide all HTTP traffic on port 9000. For more information on configuring ports and other proxy concerns please see the [Dynamic proxy configuration](DynamicProxyConfiguration.md) documentation.

Thirdly Play and some other web frameworks do not operate off a root path of anything other than `/`. In our above example `/hello` will pass through as just `/` given the way that `services` used to work (the first path component was always removed from the path delivered to the service unleass a `preservePath` option was provided). With ACLs, unless the re-write rule is provided then `/hello` will be passed through. Thus we must declare a re-write rule.
 
Following these rules, the above would be translated to:

```
    endpoints = {
      "hello" = {
        bind-protocol = "http"
        bind-port     = 0
        service-name  = "hello"
        acls          = [
          {
            http = {
              requests = [
                {
                  path = "/hello"
                  rewrite = "/"
                },
                {
                  path = "/api/hello"
                }
              ]
            }
          }
        ]
      },
      "akka-remote" = {
        bind-protocol = "tcp"
        bind-port     = 0
        services      = []
      }
    }
```

... or the equivalent in sbt as:

```
BundleKeys.endpoints := Map(
  "hello" -> Endpoint("http", 0, "hello",
    RequestAcl(
      Http(
        "/hello" -> "/",
        "/api/hello"
      )
    )
  ),
  "akka-remote" -> Endpoint("tcp", 0)
)
```

Please refer to the [ACL documentation](AclConfiguration) for more information on how to express ACLs.
