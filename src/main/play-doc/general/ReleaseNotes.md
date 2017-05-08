# ConductR release notes

## 2.1

This release broadly consists of the following functionality:

* [Open Container Initiative (OCI) support](#OCI-Support);
* [a licensing mechanism to enable free usage](#Free-Licensing); and
* [Tagging & Annotations](#Tagging-And-Annotations).

### OCI Support

We are proud to announce ConductR as being the world's first cluster manager to support the OCI Image and Runtime specifications. This helps to avoid “vendor lock-in” that is common with container software. Another key benefit of this support is that it is now possible to run Docker images directly within ConductR. More information about these OCI specifications and their benefits are available here: https://www.opencontainers.org/faq#faq5.

### Free Licensing

The new licensing mechanism permits a single ConductR Agent to be used absolutely free in production and requiring no registration. If you register with us then you will receive a license and then be able to use up to 3 ConductR Agents in production. If you are a customer then life will continue as normal.

Please visit the [migration guide](MigrationGuide#Production_Suite_Licensing) for additional instructions.

### Tagging and Annotations

#### Tagging

When selecting a bundle by its bundle name, you may now also use one of a number of bundle "tags" to qualify the name. Bundle tags are similar to Docker tags in that they help you qualify a bundle with data often equating to version numbers. When selecting a bundle by its name, appending a `:` and then one of the bundle's tags will further constrain the selection e.g. "mybundle:1.0.0-beta.1".

#### Annotations

The notion of metadata with containers is quite popular noting that bundles also describe containers. The use-cases for metadata with containers has been popularised with Docker labels, Kubernetes annotations and OCI annotations. As of 2.1, bundles may now hold metadata in the form of annotations. These annotations are very similar to [OCI image annotations](https://github.com/opencontainers/image-spec/blob/master/annotations.md) with the extension that annotation values are expressed using HOCON, specifically [Typesafe Config](https://github.com/typesafehub/config). 

The contents of annotations are generally outside of the scope of what ConductR itself is concerned with.

## 2.1.0-alpha.4 - Wednesday May 8th, 2017

* Certain annotations may now be explicitly declared in order to be shared across core nodes.
* Volume support for OCI images.
* `conductr-haproxy` has been updated so it does not generate backend declarations where there are no actual endpoints - this could cause unexpected 503 errors.
* Agent re-balancing on the loss of core nodes has been enhanced to catch a situation that would result in "orphaned agents" i.e. agents appearing in the list of agents that should no longer be there.

## 2.1.0-alpha.3 - Wednesday April 26th, 2017

* Enhanced licensing to exclude counting proxy style agents in the license count e.g. agents with a role of "haproxy". This then prevents proxy style agents from being counted as licensed agents and blocking the potential of non-proxy agents to receive resource offers; particularly on Mesos.

## 2.1.0-alpha.2

* Bad release process - ignore.

## 2.1.0-alpha.1 - Saturday April 8th, 2017

* Improvements to the reporting of bundles failing given a lack of resources.
* The control protocol has been enhanced so that bundles may now be scaled given a signed offset.
* A bundle's `isStarted` flag is now deprecated in favour of a new `isActive` field. `isActive` is `false` during the startup of a bundle, and now also the stopping of a bundle. A bundle now stops over the period of a couple of seconds in order to provide proxies with enough time to drain any existing connections. This permits a cleaner serving of connections to the outside world during a rolling upgrade scenario, particular short-lived ones such as http.
* The "services" endpoint has been removed in favor of ACLs. Service endpoints were deprecated in 2.0.
* The bundle info endpoint now associates a process ID with a bundle execution.
* When scaling down, bundles are now stopped with the newest ones first. This should help with situations where Akka Cluster Singleton is being used by your service. Akka Cluster Singleton retains the singleton at the oldest cluster member.
* ConductR instrumentation now includes Akka clustering, SBR, remoting and more. In addition, DogStatsD reporting is used so that integration with OpsClarity can be achieved.
* Process management within the ConductR Agent has been overhauled and strengthened considerably. The chance of orphaned processes with ConductR should now be quite minimal, particularly around SIGTERM/SIGKILL handling.
* ConductR now includes support for OCI Image and Runtime specifications. These are standard container formats that have been developed by the [Open Container Initiative](https://www.opencontainers.org/). For more information, reference [Producing an OCI Bundle with Docker](CreatingBundles#producing-an-oci-bundle-with-docker).
