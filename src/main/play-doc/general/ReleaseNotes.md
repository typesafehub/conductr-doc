# ConductR release notes

## 2.1

This release broadly consists of the following functionality:

* [Open Container Initiative (OCI) support](#OCI-Support);
* [a licensing mechanism to enable free usage](#Free-Licensing); and
* [external service registration](#External-Service-Registration); and
* [Tagging & Annotations](#Tagging-And-Annotations).

### OCI Support

We are proud to announce ConductR as being the world's first cluster manager to support the OCI Image and Runtime specifications. This helps to avoid “vendor lock-in” that is common with container software. Another key benefit of this support is that it is now possible to run Docker images directly within ConductR. More information about these OCI specifications and their benefits are available here: https://www.opencontainers.org/faq#faq5.

### Free Licensing

The new licensing mechanism permits a single ConductR Agent to be used absolutely free in production and requiring no registration. If you register with us then you will receive a license and then be able to use up to 3 ConductR Agents in production. If you are a customer then life will continue as normal.

Please visit the [migration guide](MigrationGuide#Production_Suite_Licensing) for additional instructions.

### External Service Registration

Since 2.0, ConductR has supported a DNS SRV record fallback mechanism for service discovery i.e. if a service cannot be discovered as being hosted by ConductR, then DNS SRV lookups can be performed in order to resolve the service's location. This was disabled for standalone mode, but enabled for DC/OS installation so that Elasticsearch lookups can utilize the DC/OS hosted service.

As of 2.1, ConductR replaces the 2.0 mechanism with something considerably more flexible. Any service name may be registered via ConductR's configuration to exist as an external service given one of the following mappings:

* an explicit IP and port
* DNS and port
* DNS SRV host 

One service name may have many mappings. A mapping may also contain user credentials (http basic auth at this point) and a root context path.

The mechanism's most significant advantage comes with not having to re-configure your services to locate services outside of those hosted by ConductR. Therefore Kafka, Elasticsearch, Cassandra and others can be readily located with no code change for your services.

For more information see the [chapter on external service discovery](ExternalServices) in order to determine how this new external service registration feature may be utilized.

### Tagging and Annotations

#### Tagging

When selecting a bundle by its bundle name, you may now also use one of a number of bundle "tags" to qualify the name. Bundle tags are similar to Docker tags in that they help you qualify a bundle with data often equating to version numbers. When selecting a bundle by its name, appending a `:` and then one of the bundle's tags will further constrain the selection e.g. "mybundle:1.0.0-beta.1".

#### Annotations

The notion of metadata with containers is quite popular noting that bundles also describe containers. The use-cases for metadata with containers has been popularised with Docker labels, Kubernetes annotations and OCI annotations. As of 2.1, bundles may now hold metadata in the form of annotations. These annotations are very similar to [OCI image annotations](https://github.com/opencontainers/image-spec/blob/master/annotations.md) with the extension that annotation values are expressed using HOCON, specifically [Typesafe Config](https://github.com/typesafehub/config). 

The contents of annotations are generally outside of the scope of what ConductR itself is concerned with.

## 2.1.0-beta.2 - ???

* The /agents endpoint now returns a field named `resourceAvailable` where such information is available. The field describes the resources available to the agent's machine.
* A `hostName` is now also returned when performing a service locator lookup - if there is one. This is done so that TLS connections can be correctly established with external services.

## 2.1.0-beta.1 - Wednesday May 24th, 2017

* External service lookup support
* Rename of "production suite" to "enterprise suite".
* Various improvements to OCI including volume support.
* `runtime-config.sh` files can now exist with regular bundles as well as configuration bundles.
* Improved `check` command given a `--any-addresses` flag that permits any one of a given number of addresses being satisfied to cause the signalling of a successful startup.
* Fixed two important issues around agent monitoring. One was another edge case around orphaned agents, and the other was around an Akka cluster client issue.
* There was a problem now fixed when subscribing to /service-hosts/<bundle>/events where no stop events were coming through.
* Volume support for OCI and Docker containers
* Fixed a problem in relation to scaling when running OCI within Docker (sandbox/OCI use cases involing a scale of more than one).

## 2.1.0-alpha.4 - Wednesday May 8th, 2017

* Certain annotations may now be explicitly declared in order to be shared across core nodes.
* Volume support for OCI images.
* `conductr-haproxy` has been updated so it does not generate backend declarations where there are no actual endpoints - this could cause unexpected 503 errors.
* Agent re-balancing on the loss of core nodes has been enhanced to catch a situation that would result in "orphaned agents" i.e. agents appearing in the list of agents that should no longer be there.
* ***Note**: This release has a defect in the `.rpm` and `.deb` agent packages that prevents the agent from starting. As a workaround, run the following command:* 
    ```bash
    sudo mkdir -p /home/conductr-agent && \
    sudo chown conductr-agent:conductr-agent /home/conductr-agent && \
    sudo systemctl restart conductr-agent
    ```

## 2.1.0-alpha.3 - Wednesday April 26th, 2017

* Enhanced licensing to exclude counting proxy style agents in the license count e.g. agents with a role of "haproxy". This then prevents proxy style agents from being counted as licensed agents and blocking the potential of non-proxy agents to receive resource offers; particularly on Mesos. A consequence of this change is that agent roles are now correctly reported when ConductR is run on DC/OS.

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
