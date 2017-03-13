# ConductR release notes

## 2.0

This is the first of a series of releases of ConductR 2.0. The main theme of ConductR 2.0 is support for a new core/agent topology. This topology leads to the potential for greater scale and for a reduced attack surface area. The topology was also driven by the need to support Mesos and its master/slave relationship between schedulers and executors. In ConductR parlence, core represents a Mesos scheduler and agent represents a Mesos executor. ConductR 2.0 therefore supports Mesos's notion of a custom executor.

When using 2.0 it is important that you revisit your bundle's cpu and memory settings. Cpu is specified as a fraction of the CPU available to a machine in total. By convention, 0.1 should be the starting point in order to correlate with a convention introduced by Mesos Marathon. Memory is very important to get right, and represents the total resident memory required by your service. In the case of Lagom and Play based applications, sbt-conductr will choose 384MiB for you if you have not specified it. Note that *this is not your app's heap memory size* as per JVM options, it is resident memory. If unsure make sure that memory is set to at least 384MiB when explicitly declaring it.

> Note: failure to size your bundle's memory as being large enough will cause the Linux OOM killer to terminate all bundles being managed by a conductr-agent on a given host. This is a [known issue with Mesos](https://issues.apache.org/jira/browse/MESOS-3333#). Unfortunately the problem is also difficult to diagnose. Please ensure that your app has plenty of resident memory declared. Think generally in terms of 128MiB increments when going beyond 384MiB.

## 2.0.5 Monday, March 13th, 2017

1. The internal `check` command would still fill up the /tmp in some circumstances. We changed how we package this command to fully resolve the issue.

## 2.0.2 Tuesday, February 28th, 2017

1. There was an edge-case where service locator and other service information could become stale at the ConductR agent. This manifested itself with a bundle using an incorrect service locator i.e. one that may not be there anymore. The problem has been resolved, along with the algorithm improved in that the "nearest" service locator will always be available to a bundle.
2. The internal `check` command was filling up the /tmp folder in some circumstances, particularly when Docker based bundles failed to start for any reason. While this is a reported problem with the underlying Python technology (pyinstaller leaves files in /tmp on receiving a SIGTERM), we have implemented a work-around so that temporary files are cleaned up.

## 2.0.1 Thursday, February 23rd, 2017

1. The ConductR core and agent command lines have been tidied up by consolidating their library dependencies into just one. This means that Unix `ps` commands become more readable when inspecting the core and agent processes.
2. ConductR now reports to its conductr.log and conductr-agent.log files at millisecond resolution. This improves log analysis should that be required.
3. Core/Agent communications has been improved in terms of resilience and bugs have been addressed in this area. A race condition was able to occur during the launch of multiple bundles at one time, which then manifested itself as being unable to stop a bundle. There was also a race condition when adding agents to cores where resource offers could be missed, thus stalling the starting of a bundle for a number of minutes.
4. The time taken to stop multiple instances of a bundle has been reduced.
5. We now log a warning when an agent fails to start on DC/OS.
6. The Visualizer has been improved by incorporating a legend. In addition, clicking on a bundle name provides a fast means to filter it.
7. The agent's port allocation algorithm has been improved by testing whether a port is actually available. Ports are allocated to bundles so that they may safely bind to a host address without conflict with others. However a process cannot reserve ports and so there was a potential for a clash. The change here is to try another port if there is a clash, which improves the chances of a bundle starting up.
8. Many hardening improvements have been made to the Continuous Delivery service, which has also received new documentation.
9. Improved reporting around missed resource offer events for a bundle. Prior to this change, missed resource offers would not be reported due to a role mismatch.
10. The deprecated services URI method used to declare service names permitted partial matches e.g. "http://:9000/myservice-with-some-prefix" could be looked up as "myservice" and "myservice-with-some-prefix". Our requirement is that exact matches are always required when performing a service locator lookup. The more recent and preferred means of explicit declaring service names was unaffected.

## 2.0.0 Friday, January 27th, 2017

1. Initial release.
