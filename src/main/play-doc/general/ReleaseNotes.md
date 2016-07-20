# ConductR release notes


## 2.0.0 Alpha

### Completed

* Separate replication/scaling concerns into ConductR Core and bundle executions concerns into ConductR Agent.
* Package and deploy ConductR as a framework within Mesos or DC/OS context.
* Implement support for loading, running and scaling of ConductR JVM bundles when deployed within Mesos or DC/OS context.
* Consolidated logging of the ConductR bundles when deployed within Mesos or DC/OS context.


### Work in progress

* Hardening when in the face of various failure scenario, i.e. loss of Core node or Agent node in both Mesos and Standalone context.
* Enhance Visualizer bundle to better display bundles running on the location separate from ConductR Core node.

### Work in progress - Mesos

* Enable proxying solution currently provided by ConductR on DC/OS.
* Consolidate definition of roles provided by ConductR with the roles provided by Mesos.
* Implement support for loading, running and scaling of ConductR Docker bundles when deployed within Mesos or DC/OS context.

### Work in progress - Standalone

* Bundle affinity functionality

