# Cluster troubleshooting

This section covers several troubleshooting scenarios:

* [Cluster Split Brain Recovery](#Cluster-Split-Brain-Recovery)
* [ConductR Core node termination](#ConductR-Core-node-termination)
* [ConductR Agent node termination](#ConductR-Agent-node-termination)
* [Bundle termination](#Bundle-termination)
* [Mesos Executor termination](#Mesos-Executor-termination)

## Cluster Split Brain Recovery

Cluster partition is a failure that may occur when running a cluster based application. To recover from this situation ConductR utilizes Akka SBR (Split Brain Recovery), which is a part of Akka's commercial extension.

The Akka SBR strategy adopted by ConductR is `keep-majority`. This strategy ensures when cluster split occurs:

* The partition which has the majority of the nodes are kept running
* Other partition(s) which has less than the majority of the nodes will have their ConductR process terminated.

Upon cluster split, Akka SBR will terminate the ConductR processes within the partition that do not have the majority number of nodes. ConductR will then restart itself and continually attempt to rejoin with the cluster that it had. ConductR is therefore designed to be self-healing.

## ConductR Core node termination

If ConductR Core process is lost due to shutdown or crash of the `conductr` service, the bundle will be automatically replicated on a different node within the ConductR cluster.

ConductR will strive to achieve configured number of replicas, and there are `3` replicas configured by default.

## ConductR Agent node termination

If a bundle dies due to a shutdown or crash of the `conductr-agent` service, the bundle will be automatically started on a different node within the ConductR cluster.

The application state is not automatically restored by ConductR. When deploying a stateful bundle ensure in your application that the state is getting replicated, e.g. with [Akka Distributed Data](http://doc.akka.io/docs/akka/snapshot/scala/distributed-data.html), [Akka Cluster Sharding](http://doc.akka.io/docs/akka/snapshot/scala/cluster-sharding.html) and [Persistence](http://doc.akka.io/docs/akka/snapshot/scala/persistence.html) or distributed databases.

## Bundle termination

If a bundle dies of its own accord then ConductR will re-schedule it for execution elsewhere. ConductR strives to attain the scale that has been declared for a bundle (the desired scale is declared when running a bundle).

## Mesos Executor termination

An orphaned ConductR executor on Mesos can be terminated by logging into the Mesos agent and killing the corresponding pid. This action should only be considered in a face of disaster where the ConductR framework instances have been already removed but the executors and associated tasks are still running. Perform the following steps to find and kill the executor pid:

* Retrieve the IP address of the Mesos agent on which the ConductR executor is running by using the Mesos UI:

```
http://dcos-host/mesos >> Find the executor with a task name "conductr-agent-bootstrapper" and select the <Host> address
```

* Open a terminal window on the selected host

* Find the pid of the ConductR executor:

```bash
ps aux | grep [c]onductr.agent.ConductrAgent | awk '{print $2}'
```

* Terminate the executor on the host by using the retrieved pid:

```bash
kill <pid>
```

* Verify that the associated tasks have been removed from the Mesos UI
