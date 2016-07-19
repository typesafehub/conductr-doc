# Cluster troubleshooting

## Cluster Split Brain Recovery

Cluster partition is a failure that may occur when running a cluster based application. To recover from this situation ConductR utilizes Akka SBR (Split Brain Recovery). ConductR uses Akka SBR version `1.0.0` which is a part of Akka's commercial extension.

The Akka SBR strategy adopted by ConductR is `keep-majority`. This strategy ensures when cluster split occurs:

* The partition which has the majority of the nodes are kept running
* Other partition(s) which has less than majority of the nodes will have their ConductR process terminated.

Upon cluster split, Akka SBR will terminate the ConductR processes within the partition that does not have majority number of nodes. ConductR will then restart itself and continually attempt to rejoin with the cluster that it had. ConductR is therefore designed to be self-healing.

## Restarting the ConductR Cluster

Restarting the entire ConductR cluster should only be considered in a face of disaster where all nodes have been lost. This restart need to be performed in a sequential order.

### Steps

* Remove the seed node and restart the `conductr` service first on one ConductR Core node:

```bash
sudo service conductr stop
sudo rm /usr/share/conductr/conf/seed-nodes
sudo service conductr start
```

*Ensure the ConductR Core process has been started successfully on this node.* This is achieved by issuing the following command where `NODE_IP_ADDRESS_OR_HOST` is the ip address or hostname of the node:

```bash
curl http://${NODE_IP_ADDRESS_OR_HOST}:9005/members
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

* If the service has been restarted successfully continue with the next node and repeat that until all ConductR Core services has been restarted.

* Restart the ConductR Agent process on all the ConductR Agent nodes:

```bash
sudo service conductr-agent restart
```

### Notes on the ConductR Core seed-nodes file

* ConductR Core will keep the address of last known members in the cluster in what we call seed nodes file.
* The file is located at `/usr/share/conductr/conf/seed-nodes`.
* By default ConductR Core keeps the latest 3 reachable members in the cluster in this file.

### Notes on the ConductR Agent core-nodes file

* ConductR Agent will keep the address of last known ConductR Core nodes in what we call core nodes file.
* The file is located at `/usr/share/conductr-agent/conf/core-nodes`.
* By default ConductR Agent keeps the latest 3 reachable ConductR Core nodes in this file.

## Bundle replication

### ConductR Core node termination

If ConductR Core process is lost due to shutdown or crash of the `conductr` service, the bundle will be automatically replicated on a different node within the ConductR cluster.

ConductR will strive to achieve configured number of replicas, and there are `3` replicas configured by default.

## Restarting Bundles

### ConductR Agent node termination

If a bundle dies due to a shutdown or crash of the `conductr-agent` service, the bundle will be automatically started on a different node within the ConductR cluster.

The application state is not automatically restored by ConductR. When deploying a stateful bundle ensure in your application that the state is getting replicated, e.g. with [Akka Distributed Data](http://doc.akka.io/docs/akka/snapshot/scala/distributed-data.html), [Akka Cluster Sharding](http://doc.akka.io/docs/akka/snapshot/scala/cluster-sharding.html) and [Persistence](http://doc.akka.io/docs/akka/snapshot/scala/persistence.html) or distributed databases.
     
### Bundle termination 
    
If a bundle dies of its own accord then ConductR will re-schedule it for execution elsewhere. ConductR strives to attain the scale that has been declared for a bundle (the desired scale is declared when running a bundle).
