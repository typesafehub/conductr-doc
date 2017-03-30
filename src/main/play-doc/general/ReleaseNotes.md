# ConductR release notes

## 2.1

This release broadly consists of the following functionality:

* [Open Container Initiative (OCI) image support](#OCI_Image_Support);
* [a licensing mechanism to enable free usage](#Free_Licensing); and
* [Tagging](#Tagging) (!).

### OCI Image Support

We are proud to announce ConductR as being the world's first cluster manager to support the OCI image format. OCI image support equates to running Docker images directly within ConductR. The key benefit to the you is the avoidance of “vendor lock-in”. The benefits of OCI images are described well here: https://www.opencontainers.org/faq#faq5.

### Free Licensing

The new licensing mechanism permits a single ConductR Agent to be used absolutely free in production and requiring no registration. If you register with us then you will receive a license and then be able to use up to 3 ConductR Agents in production. If you are a customer then life will continue as normal.

Please visit the [migration guide](MigrationGuide#Production_Suite_Licensing) for additional instructions.

### Tagging

When selecting a bundle by its bundle name, you may now also use one of a number of bundle "tags" to qualify the name. Bundle tags are similar to Docker tags in that they help you qualify a bundle with data often equating to version numbers. When selecting a bundle by its name, appending a `:` and then one of the bundle's tags will further constrain the selection e.g. "mybundle:1.0.0-beta.1".

## 2.1.0-alpha.1 - Tuesday March 28th, 2017 (FIXME)

* Improvements to the reporting of bundles failing given a lack of resources.
* The control protocol has been enhanced so that bundles may now be scaled given a signed offset.
* A bundle's `isStarted` flag is now deprecated in favour of a new `isActive` field. `isActive` is `false` during the startup of a bundle, and now also the stopping of a bundle. A bundle now stops over the period of a couple of seconds in order to provide proxies with enough time to drain any existing connections. This permits a cleaner serving of connections to the outside world during a rolling upgrade scenario, particular short-lived ones such as http.
* The "services" endpoint has been removed in favor of ACLs. Service endpoints were deprecated in 2.0.
* The bundle info endpoint now associates a process ID with a bundle execution.