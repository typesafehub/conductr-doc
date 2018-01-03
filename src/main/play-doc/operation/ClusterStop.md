# Stopping ConductR Cluster

To shutdown ConductR cluster properly choose one of the following sections depending on the ConductR mode:

* [Standalone mode](#Standalone-mode)
* [DC/OS mode](#DC/OS-mode)

## Standalone mode

The following steps need to be performed per node in a sequential order:

* Remove the seed node and stop the `conductr` service first on one ConductR Core node:

```bash
sudo service conductr stop
sudo rm /usr/share/conductr/conf/seed-nodes
```

* The removal of `seed-nodes` file is done in preparation for the future ConductR cluster start. The removal of the `seed-nodes` file
 will ensure the node where the file is removed will be able to perform as the ConductR cluster leader upon the issue of the cluster startup.

* Stop the next ConductR Core node:

```bash
sudo service conductr stop
```

* Stop the ConductR Agent process on all the ConductR Agent nodes:

```bash
sudo service conductr-agent stop
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

> Do not destroy the ConductR cluster directly from the UI without following the steps below.

* Stop the ConductR service via the DC/OS Services UI page:

```
http://dcos-host/#services >> Services >> conductr >> More >> Suspend >> Suspend Service
```

* Wait for all of the related tasks in the DC/OS UI to stop. This takes a couple minutes. When complete, there will be
0 active tasks in the service group.

* If reconfiguring, edit its JSON in the DC/OS UI and start the service again.

* If uninstalling, destroy the service:

#### DC/OS 1.8, 1.9

```
http://dcos-host/#services >> Services >> conductr >> More >> Destroy >> Destroy Service
```

#### DC/OS 1.10

Starting with DC/OS 1.10, you must use the `dcos marathon app list` and `dcos marathon app remove` commands to
uninstall ConductR.

```
# Use dcos marathon app list to find the framework ID
dcos marathon app list
ID               MEM   CPUS  TASKS  HEALTH  DEPLOYMENT  WAITING  CONTAINER  CMD
/conductr-2.1.9  1024   1     1/1    1/1       ---      False       N/A     GLOBIGNORE='*.tar.gz:*.tgz';...

# Use dcos marathon app remove to remove ConductR
dcos marathon app remove /conductr-2.1.9
```
