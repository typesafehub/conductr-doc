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

* The removal of `seed-nodes` file is done in preparation for the future ConductR cluster start. The removal of the `seed-nodes` file will ensure the node where the file is removed will be able to perform as the ConductR cluster leader upon the issue of the cluster startup.

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

* To completely remove ConductR framework from DC/OS destroy the service:

```
http://dcos-host/#services >> Services >> conductr >> More >> Destroy >> Destroy Service
```
