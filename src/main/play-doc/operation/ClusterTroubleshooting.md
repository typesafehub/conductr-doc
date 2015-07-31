# Cluster Troubleshooting

Cluster partition is a failure that may occur when running a cluster based application. To recover from this situation ConductR utilizes Akka SBR (Split Brain Recovery). Akka SBR is a part of Akka in the [Typesafe Reactive Platform](http://www.typesafe.com/products/typesafe-reactive-platform) version `15v01p05`.

The Akka SBR strategy adopted by ConductR is `keep-majority`.

The `keep-majority` strategy ensures when cluster split occurs:

* The partition which has the majority of the nodes are kept running
* Other partition(s) which has less than majority of the nodes will have their ConductR process terminated.

## Cluster Split Recovery

Upon cluster split, Akka SBR will terminate the ConductR processes within the partition that does not have majority number of nodes.

### Recovery Steps

* Obtain the list of the nodes where ConductR process have been terminated.
* Restart the ConductR process on one of the nodes in the list (`sudo service conductr restart`).
* Upon restart, the ConductR processes in these nodes will attempt to reconnect to last known members of the cluster based on the information stored in the `seed-nodes` file. This will effectively allow the restarted node to rejoin the cluster.
* If the service has been restarted successfully continue with the next node and repeat that until all services has been restarted.

### Notes on `seed-nodes` file

* The address of last known members in the cluster are kept in what we call seed nodes file.
* The file is located at `/usr/share/conductr/conf/seed-nodes`.
* By default ConductR keeps the latest 3 reachable members in the cluster in the seed nodes file.

## Whole Cluster Termination Recovery

Should the cluster split occur in such way that each partition does not have majority of the node, all ConductR process within the cluster will be terminated. Deploying a cluster across at least 3 network partitions improves the chances of maintaining a majority and preventing cluster shutdown. See [Cluster Setup Considerations](ClusterSetupConsiderations) for more information.

An example of this would be a cluster with 3 nodes, and the split occurs in such a way that none of the nodes are able to see each other. In this situation, each partition will have 1 node each, causing all ConductR process to be terminated.

The recovery in this situation can be achieved by restarting the whole ConductR cluster.

## Restarting the ConductR Cluster

Restarting the entire ConductR cluster should only be considered in a face of disaster where all nodes have been lost. This restart need to be performed in a sequential order.

### Steps

* Remove the seed node and restart the `conductr` service first on one node:

```bash
sudo service conductr stop
sudo rm /usr/share/conductr/conf/seed-nodes
sudo service conductr start
```

*Ensure the ConductR process has been started successfully on this node.* This is achieved by issuing the following command where `NODE_IP_ADDRESS_OR_HOST` is the ip address or hostname of the node:

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
                "all-conductrs"
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

* Restart the next node:

```bash
sudo service conductr restart
```

* Upon restart, this node will attempt to reconnect to last known members of the cluster based on the information stored in the `seed-nodes` file. The node will persist with the reconnection attempt until they are successful.

* If the service has been restarted successfully continue with the next node and repeat that until all services has been restarted.
