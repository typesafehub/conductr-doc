# ConductR release notes


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
