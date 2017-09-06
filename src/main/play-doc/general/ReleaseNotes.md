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

## 2.1.7 - Wednesday September 6th, 2017
* There was a problem in relation to file descriptors not being closed by the new process manager. This could limit the number of bundles loaded/run.
* The new process manager was not cleaning up all of its resources when a process exited, which could cause a memory leak. This has now been repaired.

## 2.1.6 - Wednesday August 30th, 2017
* Fixed a configuration problem such that multiple ConductR clusters on DC/OS were not assigning their own sets of ports.
* The ConductR framework id on DC/OS now embeds the ConductR version thus permitting an easier upgrade strategy.
* There was an intermittent issue with cgroup housekeeping due to a race condition.

## 2.1.5 - Wednesday August 23rd, 2017

* A ConductR Agent no longer requires 1.9 cpus on DC/OS. The new agent runs with system defaults for cgroup `cpu.shares`. The agent's CPU requirement has been reduced to that of Marathon's lowest permissible value (0.01). This particular change should free up a considerable amount of CPU reservations on DC/OS for ConductR agents.
* A fix has been applied so that the ConductR Agent logs to Elasticsearch when run on DC/OS.
* There is further resiliency around avoiding agent's being left orphaned in a situation where both the agent and the member it connects to are restarted around the same time.
* The ConductR agent has been greatly enhanced in the area of process management. Significantly less memory and improved CPU performance has been attained through using a non-blocking/asynchronous process manager in place of the JDK's.
* Considerable profiling and tuning has been performed on the ConductR Agent in order to cater for a large volume of logs being output by bundles. Logs are now throttled to a burst level of 50 messages per second (configurable).
* Akka security patches have been applied.
* OCI: we now ensure that cpu.share cgroups are correctly set.

## 2.1.4 - Thursday August 10th, 2017

* The ConductR agent's port allocator has been modified to issue ports in sequence, similar to how PIDs are issued by most operating systems. This ensures that ports will not be reused by other processes immediately after being deallocated, further improving the chances of Akka cluster processes to start.
* There was a recent and accidental limitation on the number of bundles that the ConductR agent would run, caused by a thread pool misconfiguration. The agent should now be able to run at least 200 bundles on its node with no trouble using its default configuration.
* Various strengthening improvements have been made to the bundle execution reconciliation process between ConductR agents and ConductR cores. It should now be less likely for an agent and core to become out of sync.
* `conduct logs` was returning an empty result when Elasticsearch v.5.x was used. ConductR continues to support earlier versions of Elasticsearch as well as v.5.x.
* Various improvements have been made to the ConductR agent's process management logic. These improvements further increase the resiliency around handling processes (bundles) that die unexpectedly.
* Various improvements have been made to eliminate the number of log messages that were considered pollution and unecessary.
* A small issue was found where the system version wasn't being considered when scaling multiple bundles of the same system, but distinct system versions. The change is to treat the combination of system and system version distinctly which results in faster system startup for these bundles.
* The ConductR agent's reliability in terms of conveying resource information has been strengthened.

## 2.1.3 - Wednesday July 19th, 2017

* Update [DNS service locator](https://github.com/typesafehub/service-locator-dns) to `1.0.2`. This will ensure the hostname of an SRV record is properly forwarded.

## 2.1.2 - Tuesday July 11th, 2017

* Upgrade to Akka HTTP `10.0.8`.
* Ensure that Control Protocol API routing is sealed thus preserving rejections.
* Move allocated port range in DCOS mode to avoid conflict with Marathon LB.
* Improve bundle termination when Agent is unexpectedly terminated.

## 2.1.1 - Wednesday June 21st, 2017

* There was an ancient bug in scheduling where resource offers were skipped given an ineligible offer followed by an eligible one.
* Agents now shutdown bundles gracefully when terminating. Prior to this, bundles received SIGKILL on an agent restart/shutdown.
* There was a problem that manifested in the Visualizer being unable to receive server sent events. This problem was recently introduced by Akka http disallowing `charset` not being accepted on all event endpoints.
* There was a problem where conductr-haproxy would render a backend element for inactive bundles.

## 2.1.0 - Thursday June 13th, 2017

* OCI dependency updates
* Akka updates which fixes a customer-reported remoting bug along with various other bug fixes

## 2.1.0-rc.1 - Wednesday June 7th, 2017

* The /agents endpoint now returns a field named `resourceAvailable` where such information is available. The field describes the resources available to the agent's machine.
* A `hostName` is now also returned when performing a service locator lookup - if there is one. This is done so that TLS connections can be correctly established with external services.
* OCI cgroups functionality such that cpu and memory constraints are automaticallly applied.
* Graceful agent shutdown now occurs when a Mesos executor is terminated, thus freeing executors from Mesos correctly.

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
