# ConductR release notes

## 1.0.16

* Update to Reactive Platform 15v09
* Increase Visualizer request timeout to prevent reconnects and log noise
* Configure for Akka diagnostics reporting, [akka.diagnostics.recorder.enabled](http://doc.akka.io/docs/akka/rp-15v09p03/common/diagnostics-recorder.html)
* Add mappings for directories in installation location to be owned by daemon user

## 1.0.15

* RHEL 7 systemd support is now available as a new RPM.
* A small improvement was applied to the control protocol unload response where the bundle id is returned. This permits some new CLI functionality where the unload command can wait for completion before returning to the users (aids scripting).

## 1.0.14

* Bundles prior to this release could signal their exit to ConductR prematurely i.e. before they really have exited. Redis is an example where it performs a graceful shutdown on SIGTERM and therefore should not exit immediately. In this case though, ConductR was signalled completion during the bundle's shutdown. ConductR is now signalled later, when a bundle truly completes. This change should lead to increased reliability for bundle process management.
* Audited log levels and messages and tuned them for operational consumption. Also handled some exceptions were benign so that they do not pollute the logs.
* Improved the local log file format to output a UTC based time along with host information.
* The `-J-XX:+HeapDumpOnOutOfMemoryError` and `-server` flags are now declared on by default for ConductR's `application.ini`.

## 1.0.13


* The Docker based image now supports ConductR node role assignment via sbt-conductr-sandbox and the forthcoming CLI.
* Improved the Visualizer for iOS and general mobile operation.
* Fixed a problem where bundles that report no replications were not able to be unloaded.
* Fixed a problem introduced by 1.0.12 where the seed-nodes file was being written to during the shutdown of a ConductR member, and could thus write incorrect host data and prevent the service from restarting properly.
* Upgraded our usage of sbt-native-packager so that relocatable RPMs and symlinks now work properly.
* Fixed an issue where a stacktrace appeared given a condition where a single ConductR is yet to receive cluster member information. Although the effect was benign, the log file is a bit tidier.
* Leader membership changes weren't handled insofar as need to update bundle state. This has now been corrected.
* ConductR cluster changes and the effects on replication and scaling have been improved further.

## 1.0.12

* Improved the reliability of connectivity with the experimental Elasticsearch logging feature.
* Released a Docker based image of the full ConductR which may be used by the sandbox for development purposes only. The image name is "typesafe-docker-registry-for-subscribers-only.bintray.io/conductr/conductr" and is now the default for sbt-conductr-sandbox.
* Improved the Visualizer sample with a means to filter bundles.
* Corrected the index name used by ConductR for the experimental Elasticsearch functionality.
* Now permits the disablement of reading the conf/seed-nodes file on startup
  for non-production style environments. For more information please refer to the `--seed-node-file-disabled` option of your distribution's `conf/application.ini`.
* Seed nodes are now provided in an order of their distance from the current node. This provides ConductR with the best chance of seeding with networks that may exist on other availability zones or data centers.
* Fixed a bug where requests to load or scale a bundle could accumulate despite expiring and result in a situation where no further loads or scales would be accepted.

## 1.0.11

* The check command was not indicating a failure exit status when failures occurred
* Included the conductr-kibana bundle as part of ConductR project
* The Elasticsearch host required setting when used with the sandbox
* A file named runtime-config.sh is now favored when a configuration bundle is supplied thereby permitting other files to be a part of the configuration bundle

## 1.0.10

* Enabled RPM prefixes to be used with the RPM distribution of ConductR
* Slightly enhanced ConductR's Resource Provider so that it responds to more system stimuli
* Improved bundle load error checking

## 1.0.9

* Fixed the current working directory for the RPM packaging of ConductR
* Enhancements to the experimental Elasticsearch support for consolidated logging
* Improved bundle replication and scaling and tested them to be both reliable within larger clusters on EC2
* Bundles are now restarted when they fail. In the case of a bundle starting and not registering that it has started, ConductR will attempt 3 retries. If a bundle signals that it starts successfully and then fails with a non-zero exit code, ConductR will schedule it for a restart. If a bundle signals that it starts successfully and then exits with a zero exit code (success), ConductR will leave it alone.

## 1.0.8

* Bundle replication and scaling have again been improved given further testing. A 9 node cluster on EC2 with 6 bundles, replicated 9 times and each receiving a scale factor of 4 has shown to be resilient to 3 nodes (1 AZ) disappearing. All bundles are re-balanced on the other nodes. One important and helpful change here is the introduction of a configurable parameter for dampening the ConductR's reaction to cluster membership changes. By default each ConductR service will wait for 2 seconds post receiving any membership change event before acting upon it. This dampening can reduce jitter. Other areas have also been addressed where some of the assumptions about nodes being in a certain state have been removed.

## 1.0.7

* Improved bundle scaling reliability
* Corrected an issue where the name_OTHER_IPS was only being populated with one other IP (same goes for ports)
* The default max size of bundles accepted by ConductR is now 200MB
* Dropped the use of CEE cookies for syslog - structured data is used exclusively
* Arguments can now be provided for bundle components that use Docker

## 1.0.6

* Escaped bash chars in start command
* Split brains are now handled using Akka's new split brain resolver functionality

## 1.0.5

* Verified that ConductR is free from GPL code
* Use SystemV on RHEL6
* Added ability to redirect URLs capturing all information in the URI for service lookup
* Improved scaling reliability drammatically
* Improved the way entries are written to the seed-nodes file for greater success when restarting
* Improved the method by which bundles are terminated - they are now sent SIGTERM and then only SIGKILL if the former times out

## 1.0.4

* Improved the resiliency around starting up bundles
* Bundle scale query implemented for the control protocol
* Internal dependency update
* Improved logging resiliency

## 1.0.3

* Improved the resliency around service lookup within ConductR
* Corrected a problem where a bundle's dynamic ports were not being deallocated.
* Improved some log output at the debug level
* General akka updates

## 1.0.2

* Improved memory management for the ConductR service
* Corrected a problem with validating the loading of bundles
* Improved the debian package to depend on pip3

## 1.0.1

* Improved reliability around scaling bundles

## 1.0.0

Initial release.
