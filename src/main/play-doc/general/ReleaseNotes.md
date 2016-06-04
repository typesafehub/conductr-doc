# ConductR release notes

### 1.1.6 June 6, 2016

1. Changes to syslog connectivity in order to avoid CLOSED_WAIT states.

2. A native library is now used in order to collect metrics so that resource offers may be created reliably. In addition the wrong metric was being used to determine the amount of free memory available to a host. This could prevent bundles with memory requirements exceeding 4GiB from being run. Changes in this area should result in the correct offering of memory allocations, including a fairer sharing of resources across the cluster.

3. Elasticsearch 2.x compatibility has been incorporated when using it as an external log collector i.e. when not using the Elasticsearch bundle shipped with ConductR. Prior to this release, only Elasticsearch 1.x was supported.

4. When a bundle is failed to be matched against resource offers during scaling, the `conduct events` command will now report more information on why. The last 10 distinctly declined resource offers received will be reported. This new reporting mechanism should assist in the diagnosis of resource shortages of a ConductR cluster.

5. A serialisation issue was fixed that could prevent a bundle from scaling correctly.

6. Some default timeout settings have been improved to reflect slower clusters such as the developer sandbox, but perhaps also slower clusters in general.

### 1.1.5 May 17, 2016

1. A problem was fixed when there was a bundle belonging to a system shared by other bundles, and that it also had multiple endpoints, causing the xxx_OTHER_xxx series of environment variables to become empty when there was no other instance of the same bundle running. This problem could prevent bundles from joining an established Akka cluster.

2. A clearer message and HTTP status is now returned by ConductR when there is no Elasticsearch running and the `conduct logs` command (or its API) is used. An HTTP "Service Unavailable" status is returned.


### 1.1.4 May 4, 2016

1. The Resource Provider now uses committed virtual memory size for the value of available memory in resource offers. This has been demonstrated to provide a better measure of how much memory can be allocated for a new bundle process over free memory and swap calculated values.

2. The bundleInstallationChanged event has been added for bundle installation changes and is now available to server-sent events(SSE) clients. This new event is used by the ResourceProvider to request resources on all installation change events. This enables bundle scaling to be more reliable under stress, such as scaling when resources are highly constrained.

4. Ensure that bundles are always written to a single volume of a node. This ensures the ability to atomically move a bundle when using an operator specified bundle location.

5. Adds bundle id and configuration id to ESlite service. Improves ESlite tests.

6. Improvements to ConductR tests including upgrading Scala test versions, using Trireme for sbt-mocha and concurrency restrictions to address inconsistent test issues.

### 1.1.3 - Apr 15, 2016

1. The Resource Provider now outputs its view of what resources it believes are available so that operators can better diagnose failure to obtain suitable resource offers.

2. The service locator should always remove the first path component when replying with a redirect and only use the first path component to identify the service name in a redirect style scenario, enabling seamless usage with HTTP clients.

3. Get and load bundles endpoint should only respond to `/bundles` or `/bundles/` paths, and should not response to any other subpaths underneath.
Prior this release, calling HTTP get on `/bundles/some/random/path` will return the bundles response. This is incorrect as 404 should be returned instead

4. Introduces Eslite, a basic Elasticsearch behaving bundle that doesn't require a GiB of memory to run. Eslite is ideal for localhost testing and will be used for the default logging option by the developer sandbox.

5. Updates ConductR's use of the Reactive Platform to rp16s01p05.

6. Syslog connectivity was broken. This has now been repaired.

### 1.1.2 - Mar 3, 2016

1. Preserve the first component of a path that is rewritten at the proxy.

2. Only retrieve bundle info after the events connection is established to prevent potential loss of events.

3. Introduce new conductr-haproxy integration tests.

4. Integrate StateEvents and StateQueries actors to provide a consistent view of bundle state between the two to prevent possible race condition.

## 1.1.1 - Feb 25, 2016

1. When a load or a scale fails due to no resources being available then ConductR now reports this. For loading this means that the initial http load request is responded with a `Forbidden` http status`, along with a message. For all other scenarios, an error event is raised on the bundle and `conduct events` can be used to discover what happened.
2. Bundle replication would sometimes fail due to there being insufficient resources. In this scenario CPU, memory and disk space were being checked. However replication only requires a check in terms of disk space, and so this is now the case. Replication should therefore be more reliable in scenarios where there is low CPU or memory availability.

## 1.1.0 - Feb 14, 2016

Initial release.
