# ConductR Migration Guide


This document describes what is required to move between version 1.0 to 1.1. Here is a list of major areas to be considered when migrating:

* [Bundle versioning](#Bundle-versioning)
* [Logging](#Logging)

## Bundle versioning

Prior to 1.1 `sbt-bundle` encoded your project's version into a bundle's name and also any system name. These new version properties factor out that concern into their own properties. The major implication of this is that if you have provided a `bundle.conf` in a bundle's optional configuration then you must attend to any component name being overridden. Here's an example of a pre 1.1 bundle overiding a value inside an optional configuration's `bundle.conf`:

```
system = "doc-renderer-cluster"
components {
  "project-doc-1.0-SNAPSHOT" {
    endpoints.web.services = ["http://milo.lightbend.com"]
  }
}
```

For 1.1 this should now look like:

```
system = "doc-renderer-cluster"
components {
  "project-doc" {
    endpoints.web.services = ["http://milo.lightbend.com"]
  }
}
```

i.e. note that the encoding of a version has been removed from the component's name.

### Rationale

`[sbt-bundle](https://github.com/sbt/sbt-bundle#conductr-bundle-plugin)` has been enhanced to support two new types of version so that version information can be retained and reasoned with reliably. The versions are:

* `compatibilityVersion`; and
* `systemVersion`

A `compatibilityVersion` is a versioning scheme that will be included in a bundle's name that describes the level of compatibility with bundles that go before it. By default we take the major version component of a version as defined by [http://semver.org/]. However you can make this mean anything that you need it to mean in relation to bundles produced prior to it. We take the notion of a compatibility version from [http://ometer.com/parallel.html]."

The `systemVersion` is a version to associate with a system. This setting normally correlates to the value of `compatibilityVersion`, but doesn't have to. Systems declared using the `system` property can technically span across different bundles. Again, the meaning given to the `systemVersion` can be anything you need it to mean. However for Akka cluster based services it declares the name of the Akka cluster to join and is compatible with (Akka cluster requires binary compatibility for your service's messages given serialization concerns).

When migrating we recommend that you update your `sbt-bundle`, `sbt-conductr` and `sbt-conductr-sandbox` plugins to their 1.1 versions. These plugins can continue to work with ConductR 1.0 as they are backward compatible. However in order for `sbt-conductr` to convey the new versioning scheme, you are required to enable the new API:

```scala
ConductRKeys.conductrApiVersion := "1.1"
```

## Logging

The use of Elasticsearch for collecting event and log data is no longer considered experimental and is therefore enabled by default. If you wish to continue using a syslog receiver then please apply to the following settings in your `application.ini` (the following relates to connectivity with a locally installed rsyslog):

```
-Dcontrail.syslog.server.host=127.0.0.1 
-Dcontrail.syslog.server.port=514 
-Dcontrail.syslog.server.elasticsearch.enabled=off
```

## Reloading HAProxy

ConductR-HAProxy now provides a `reloadHAProxy.sh` script to handle the reloading of HAProxy across systemv and systemd platforms. If a more specific reload sequence is required, a custom reload script can be specified using the CONDUCTR_RELOADHAPROXY_SCRIPT environment variable in a configuration bundle.

Node installation has changed to use `/usr/bin/reloadHAProxy.sh` in `sudoers` instead of calling `/etc/init.d/haproxy reload.`
