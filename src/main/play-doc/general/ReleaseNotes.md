# ConductR release notes

### 1.1.10 October 12th, 2016

1. Various monitor dashboard improvements.
2. There was a problem using the ConductR roles functionality from within the developer sandbox.
3. Various internal dependency updates including Akka 2.4.10.
4. Unreachable/reachable member states can now be observed via the /members/events endpoint.
5. The Visualizer has been updated to render unreachable/reachable member states.
6. A sporadic failure could occur when loading a bundle.
7. A bundle could start and then stop quickly due to a race condition between the sources of detection of each.

### 1.1.9 September 9th, 2016

1. Improve sandbox logging. The sandbox logs have been improved such that they are not confused with the logging associated with the bootstrap's initialisation.
2. Improve Grafana dashboards for monitoring feature in sandbox. Fix elasticsearch datasource configuration. JVM metrics include application filter, and memory pool stats are fixed. Default dashboard time range is now 15 minutes, rather than 1 hour.

### 1.1.8 August 29th, 2016

1. Exit codes of 143 signify that a bundle has been terminated - which is correct in that ConductR terminates them when a stop request is received. However these non-zero exit codes caused an error indicator to appear when performing a `conduct info`. Given that 143 is not technically an error (SIGTERM is a graceful shutdown condition), this code is now interpreted in the same was as an exit code of 0.
2. Bundle logs are now also directed to ConductR's file system based logs. This can help to debug ConductR by the ConductR team in particular.
3. PID files were removed prematurely upon a bundle stop request having been received. This could result in orphaned bundle processes that could not be subsequently terminated when ConductR restarted. Further to the change made to correct this, the process handling of bundles has been improved. A side effect of this latter change is that stopping bundles is now much faster.
4. Now handles failures in launching bundles much faster than before. Prior to this release up to 10 minutes (by default) could be spent waiting for confirmation on the success of a launch. Changes to this release propogate failures immediately thus allow the internal scaling mechanism to move on to the next bundle of the same system (if there is one).
5. Role matching is now disabled by default as this is what we have found people to prefer; thinking about roles upfront can be tedious.
6. Fixes a potential issue where the launching of processes could become blocked and even result in taking several minutes.
7. The reporting of scaling requests failing has been particularly improved in terms of formatting.
8. The reporting of all ConductR events has been improved in terms of formatting. All console written logs additionally now contain mapped data context information (meta data regarding a log event).
9. There was a proxy configuration issue with the developer sandbox that could result in a race condition when the Docker container was restarted; as is the case with the sbt-conductr `install` command. The problem would sometimes manifest itself with `conduct logs` returning a `502` error. This issue does *not* affect production releases i.e. RPM and Debian releases.
10. Prior to this release docker build errors during bundle startup were suppressed. This was due to an old issue where Docker used to write incorrectly to stderr. Docker now only writes its errors to stderr thus making it usable and capture problems that otherwise hide themselves (for example if docker authentication is failing).
11. Improved bundle loading resiliency for the developer sandbox.
12. Bundle configuration parsing issues are now improved by citing the parsing error when loading a bundle.
13. Due to a race condition it was possible for `conduct info` to show bundles having errors even though they have loaded and run correctly.
14. Grafana is now the basis of the monitoring feature instead of Takipi.
15. Prevents a benign error from being reported in conductr-haproxy after about 8MB of events have been received from ConductR.

### 1.1.7 June 15, 2016

1. An internal change has been made to handling of `StatusService` messages. These changes are concerned with scalability and resiliency.

2. When the ConductR service terminates gracefully it now informs the cluster so that its data can be cleaned up and thus avoid ConductR making poor decisions based on stale data. Secondly, in the event that ConductR abnormally exits a cluster then it shall ensure that any host related data is removed during startup.

3. conductr-haproxy has been improved so that bundle state changes are handled while HAProxy is in the process of being updated. There was a chance previously where these state changes could have been missed.

4. ConductR dependencies have been refreshed in order to take advantage of the most recent Akka release and thereby benefiting from some resiliency improvements around Akka clustering.

5. The latest Reactive Monitoring has been incorporated.

6. RPM distributions suffered a regression for 1.1.6. This problem has now been fixed.

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
