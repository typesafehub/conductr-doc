# Restarting ConductR Cluster

Restarting the entire ConductR cluster should only be considered in a face of disaster where all nodes have been lost. Choose one of the following sections depending on the ConductR mode:

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

> Do not Scale to 0, Suspend, or Destroy ConductR cluster directly from the Marathon UI without following the steps below. Doing so without following the steps below will cause the ConductR to be de-activated from DC/OS and forcefully detached from its still running tasks. These tasks will be orphaned and can't be terminated as it's no longer visible from DC/OS CLI's and Marathon's perspective.

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

* The ConductR framework instances are still running. Stop them by suspending the ConductR service via the DC/OS Services UI page:

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
