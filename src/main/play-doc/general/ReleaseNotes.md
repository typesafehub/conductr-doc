# ConductR release notes

## 2.0

This is the first of a series of releases of ConductR 2.0. The main theme of ConductR 2.0 is support for a new core/agent topology. This topology leads to the potential for greater scale and for a reduced attack surface area. The topology was also driven by the need to support Mesos and its master/slave relationship between schedulers and executors. In ConductR parlence, core represents a Mesos scheduler and agent represents a Mesos executor. ConductR 2.0 therefore supports Mesos's notion of a custom executor.

When using 2.0 it is important that you revisit your bundle's cpu and memory settings. Cpu is specified as a fraction of the CPU available to a machine in total. By convention, 0.1 should be the starting point in order to correlate with a convention introduced by Mesos Marathon. Memory is very important to get right, and represents the total resident memory required by your service. In the case of Lagom and Play based applications, sbt-conductr will choose 384MiB for you if you have not specified it. Note that *this is not your app's heap memory size* as per JVM options, it is resident memory. If unsure make sure that memory is set to at least 384MiB when explicitly declaring it.

> Note: failure to size your bundle's memory as being large enough will cause the Linux OOM killer to terminate all bundles being managed by a conductr-agent on a given host. This is a [known issue with Mesos](https://issues.apache.org/jira/browse/MESOS-3333#). Unfortunately the problem is also difficult to diagnose. Please ensure that your app has plenty of resident memory declared. Think generally in terms of 128MiB increments when going beyond 384MiB.

## 2.0.0-beta.1 September 2nd, 2016

1. The ConductR Agent is presently using Akka cluster client in order to communicate with ConductR core. Akka cluster client leverages Akka remoting and therefore also requires a bind to port 2552 on the agent's host. We will be moving away from Akka cluster client for our agent's use case.
2. The ConductR Agent's set of reserved ports (10000 to 10999) are not presently being reserved with Mesos and could therefore clash with other processes requiring those ports. This will be addressed for the next beta.
3. Certain resiliency scenarios around the Mesos master terminating and then reconciling are not presently implemented. This will be addressed for the next beta.
4. When a ConductR core service terminates where it is connected to the Mesos master another ConductR core service will take over. However it will join as a new framework and cause an inconsistency with Mesos. Bundle execution state will then be compromised and you may have to restart ConductR from Marathon. This will be addressed for the next beta.
5. The Visualizer requires an enhancement to view agents i.e. there is no rendering of bundle information unless the agent is running on the same node as core.
6. The Visualizer URL is not presently accessible from the DC/OS GUI.
7. ConductR's control protocol and Visualizer are not presently available via the DC/OS admin gateway. We plan on making this so, and also updating the ConductR CLI so that they may all leverage DC/OS authentication and authorization.
8. Agent information is not presently available via ConductR's control protocol.
