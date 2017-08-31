# Restarting ConductR Cluster

Restarting the entire ConductR cluster should only be considered in a face of disaster where all nodes have been lost. 
Choose one of the following sections depending on the ConductR mode:

* [Standalone mode](#Standalone-mode)
* [DC/OS mode](#DC/OS-mode)

## Standalone mode

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

### Notes on the ConductR Core seed-nodes file

* ConductR Core will keep the address of last known members in the cluster in what we call seed nodes file.
* The file is located at `/usr/share/conductr/conf/seed-nodes`.
* By default ConductR Core keeps the latest 3 reachable members in the cluster in this file.

### Notes on the ConductR Agent core-nodes file

* ConductR Agent will keep the address of last known ConductR Core nodes in what we call core nodes file.
* The file is located at `/usr/share/conductr-agent/conf/core-nodes`.
* By default ConductR Agent keeps the latest 3 reachable ConductR Core nodes in this file.

## DC/OS mode

To restart the cluster in DC/OS mode, follow the steps below. You'll also want to follow these steps if you are deploying
a configuration change. Note that upon cluster start, your bundles will need to be reloaded.

> Do not destroy the ConductR cluster directly from the UI without following the steps below.

* Stop the ConductR service via the DC/OS Services UI page:

```
http://dcos-host/#services >> Services >> conductr >> More >> Suspend >> Suspend Service
```

* Wait for all of the related tasks in the DC/OS UI to stop. This takes a couple minutes. When complete, there will be
  0 active tasks in the service group.

* Once the service is indicated as Suspended, make any configuration changes if necessary.

```
http://dcos-host/#services >> Services >> conductr >> More >> Edit >> Review & Run >> Run Service
```

* Finally, start the service again, choosing an odd number of instances (3 recommended). This will start the ConductR framework instances and the executors on the Mesos agent:

```
http://dcos-host/#services >> Services >> conductr >> More >> Resume >> Resume Service
```
