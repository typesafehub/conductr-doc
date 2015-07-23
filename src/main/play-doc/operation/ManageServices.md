# Managing ConductR services

During installation ConductR registers a linux service named `conductr`. If HAProxy is installed, the service `conductr-haproxy` is registered as well. Both services are started automatically during boot-up.

## Service user
The `conductr` service runs as the daemon user `conductr` in the user group `conductr`. When the service is started the first time it creates the user and group itself.

The `conductr` user executes the commands in the background without any shell. You can specify additional environment variables in `/etc/default/conductr`. This file will be sourced before the actual service gets started.


## Change service state

In order to start, stop or restart ConductR on one node change the state of the service.

**sysvinit**

```bash
sudo service conductr start
sudo service conductr stop
sudo service conductr restart
```

### Restart cluster

Restarting the entire ConductR cluster should only be considered in a face of disaster where all nodes have been lost. This restart need to be performed in a sequential order. This is necessary to ensure that at least one seed node is always up and running. Restarting all nodes at the same time will stop all nodes first. If now a node is starting again, it will not find any existing seed node and therefor it will register itself as a seed node in a new cluster. This means that each node starting in parallel will form its own cluster. To avoid this scenario restart the `conductr` service first on one node. If the service has been restarted successfully continue with the next node and repeat that until all services has been restarted.

**Troubleshooting**
In case a node has accidentally formed its own cluster you can stop this node and remove the seed node information. Afterwards start this node again.

```bash
sudo service conductr stop
sudo rm /usr/share/conductr/conf/seed-nodes
sudo service conductr start
```

During startup the node will now join another running cluster as a cluster member.
