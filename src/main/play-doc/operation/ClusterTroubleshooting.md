# Cluster troubleshooting

This section covers several troubleshooting scenarios:

* [Cluster Split Brain Recovery](#Cluster-Split-Brain-Recovery)
* [Restarting ConductR Cluster](#Restarting-ConductR-Cluster)
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

## Restarting ConductR Cluster

Restarting the entire ConductR cluster should only be considered in a face of disaster where all nodes have been lost. Choose one of the following sections depending on the ConductR mode:

* [Standalone](#Standalone-mode)
* [DC/OS](#DC/OS-mode)

### Standalone mode

The following steps need to be performed per node in a sequential order:

* Remove the seed node and restart the `conductr` service first on one ConductR Core node:

```bash
sudo service conductr stop
sudo rm /usr/share/conductr/conf/seed-nodes
sudo service conductr start
```

*Ensure the ConductR Core process has been started successfully on this node.* This is achieved by issuing the following command where `NODE_IP_ADDRESS_OR_HOST` is the ip address or hostname of the node:

```bash
curl http://${NODE_IP_ADDRESS_OR_HOST}:9005/v2/members
```

* When the node has been started successfully, the endpoint will return the following json structure with `members` field in the payload populated with at least one entry:

```
{
    "members": [
        {
            "node": "akka.tcp://conductr@172.17.3.158:9004",
            "nodeUid": "-1306973826",
            "roles": [
                "web"
            ],
            "status": "Up"
        }
    ],
    "selfNode": "akka.tcp://conductr@172.17.3.158:9004",
    "selfNodeUid": "-1306973826",
    "unreachable": []
}
```

* Upon restart, the node will not find any existing seed node due to the fact the `seed-nodes` file has been removed. Therefore the node will register itself as a seed node in a new cluster.

* Restart the next ConductR Core node:

```bash
sudo service conductr restart
```

* Upon restart, this node will attempt to reconnect to last known members of the cluster based on the information stored in the `seed-nodes` file. The node will persist with the reconnection attempt until they are successful.

* If the service has been restarted successfully continue with the next node and repeat that until all ConductR Core services have been restarted.

* Restart the ConductR Agent process on all the ConductR Agent nodes:

```bash
sudo service conductr-agent restart
```

#### Notes on the ConductR Core seed-nodes file

* ConductR Core will keep the address of last known members in the cluster in what we call seed nodes file.
* The file is located at `/usr/share/conductr/conf/seed-nodes`.
* By default ConductR Core keeps the latest 3 reachable members in the cluster in this file.

#### Notes on the ConductR Agent core-nodes file

* ConductR Agent will keep the address of last known ConductR Core nodes in what we call core nodes file.
* The file is located at `/usr/share/conductr-agent/conf/core-nodes`.
* By default ConductR Agent keeps the latest 3 reachable ConductR Core nodes in this file.

### DC/OS mode

* Retrieve the service id of the ConductR service:

```bash
dcos service
NAME       HOST        ACTIVE  TASKS  CPU     MEM      DISK      ID
conductr   10.0.1.118  True    0      11.8    42760.0  120804.0  my-conductr
```

* Shutdown the ConductR service by using the `dcos service shutdown`. The command shuts down all ConductR executors on the Mesos agents and with it all tasks and bundles that have been created by ConductR. On Mesos master, the service is also marked as removed:

```bash
dcos service shutdown my-conductr
```

* The ConductR framework instances are still running. Stop them by suspending the ConductR service via the DC/OS Services UI page

```
http://dcos-host/#services >> Services >> conductr >> More >> Suspend >> Suspend Service
```

* Once a service has been removed the id cannot be reused. Therefore, rename your service by editing the service configuration:

```
http://dcos-host/#services >> Services >> conductr >> Edit >> <Change ID> >> Deploy Changes
```

* The previous step does not rename the existing service but instead create an additional service with the new id. Remove the old service:

```
http://dcos-host/#services >> Services >> conductr >> More >> Destroy >> Destroy Service
```

* Navigate to the new service and start it. This will start the ConductR framework instances and the executors on the Mesos agent:

```
http://dcos-host/#services >> Services >> conductr >> Scale >> <Select number of ConductR framework instances> >> Scale Service
```

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
ps aux | grep [c]onductr.agent.ConductRAgent | awk '{print $2}'
```

* Terminate the executor on the host by using the retrieved pid:

```bash
kill <pid>
```

* Verify that the associated tasks have been removed from the Mesos UI
