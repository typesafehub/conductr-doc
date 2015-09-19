# ConductR Migration Guide


This document describes what is required to move between version 1.0 to 1.1.

## Bundle versioning

`[sbt-bundle](https://github.com/sbt/sbt-bundle#conductr-bundle-plugin)` has been enhanced to support two new types of version:

* `compatibility-version`; and
* `system-version`

A `compatibility-version` is a versioning scheme that will be included in a bundle's name that describes the level of compatibility with bundles that go before it. By default we take the major version component of a version as defined by [http://semver.org/]. However you can make this mean anything that you need it to mean in relation to bundles produced prior to it. We take the notion of a compatibility version from [http://ometer.com/parallel.html]."

The `system-version` is a version to associate with a system. This setting normally correlates to the value of `compatibility-version`, but doesn't have to. Systems declared using the `system` property can technically span across different bundles. Again, the meaning given to the `system-version` can be anything you need it to mean. However for Akka cluster based services it declares the name of the Akka cluster to join and is compatible with (Akka cluster requires binary compatibility for your service's messages given serialization concerns).

When migrating we recommend that you update your `sbt-bundle`, `sbt-conductr` and `sbt-conductr-sandbox` plugins to their 1.1 versions. These plugins can continue to work with ConductR 1.0 as they are backward compatible. However in order for `sbt-conductr` to convey the new versioning scheme, you are required to enable the new API:

```scala
ConductRKeys.conductrApiVersion := "1.1"
```

Prior to 1.1 `sbt-bundle` encoded your project's version into a bundle's name and also any system name. These new version properties factor out that concern into their own properties.