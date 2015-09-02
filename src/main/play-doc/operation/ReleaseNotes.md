# ConductR release notes


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
