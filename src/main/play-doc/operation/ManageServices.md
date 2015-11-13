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

## Configuring ConductR

The application.ini file, located in /usr/share/conductr/conf, is the primary configuration file for the ConductR service. This file is used to specify ConductR service settings, such as '-Dconductr.ip' used during installation. See the comments section of the application.ini file for more examples.

Akka module configuration can also be set using this file. For example, to assign a ConductR node the roles of `megaIOPS` and `muchMem` instead of the default, `web`, set `akka.cluster.roles` in application.ini:

```bash
 -Dakka.cluster.roles.0=megaIOPS
 -Dakka.cluster.roles.1=muchMem
```
 
With this setting the node would offer the roles `megaIOPS` and `muchMem`.Only bundles with a `BundleKeys.roles` of `megaIOPS,` `muchMem` or both `megaIOPS` and `muchMem` will be loaded and run on this node.

The ConductR service must be restarted for changes to this file to take effect.

## Roles

Roles allow machines to be targetted for specific purposes. Some machines may have greater IO capabilities than others, some may have more CPU, memory and other resources. Some may also be required to maintain a volume that holds a database.

When getting started with ConductR it is reasonable to have each ConductR service rely on its default role of `web`. However when moving into a production scenario you should plan and assign roles for your ConductR cluster.

When a bundle is to be scheduled for loading or scaling, a check is made to first see whether a resource offer's roles intersect with the roles that the bundle requires. If it does then it is eligible. If no resource offers provide the roles required by the bundle, the bundle cannot be loaded or scaled. Bundles will only be loaded to member nodes providing the bundle required roles. If no members of the cluster provide those roles, the bundle will fail to load.

#### Using Roles

Roles can be leveraged in varying levels of specificity as needed to achieve the desired results. Small clusters running multiple apps will generally need few roles. Bundles need to be able to relocated to other nodes in the event of failure. Overly dividing a small cluster into small sub-sets reduces relience.  Smaller clusters therefore will generally use few roles to create a few sub-sets of nodes.

Larger clusters on the other hand will generally want more specialization and therefore benefit from further use of roles. Bundles with specific needs, such as resource intensive and data storage applications, will generally want exclusive use of a subset of nodes by using highly specific roles.

## Service Monitoring

For best resilience, the ConductR service daemons should be monitored and restarted in the event of failure. [sbt-native-packager](https://github.com/sbt/sbt-native-packager) has experimental [systemd support](http://www.scala-sbt.org/sbt-native-packager/archetypes/java_server/customize.html#systemd-support). As systemd support matures, ConductR will made available as a package manged by systemd with restart on failure enabled. Until that time, third party daemon monitors can be utilized to provide restart on failure.

# Upgrading ConductR

A running cluster can be upgraded without downtime either at the node or cluster level. On a per node basis, new updated nodes are introduced to the cluster while the old nodes are removed from the cluster. On a per cluster basis, incoming requests are directed from the old cluster to the new cluster using load balancer membership or DNS changes. Per node upgrades are generally easier, however they cannot be performed across ConductR releases that are marked as not compatible with previous releases.

To perform a per node upgrade, introduce new cluster members to the cluster. The new nodes could be created with updated versions of Conductr, Java, the Linux operating system or any other component installed during provisioning. As old members are removed from the cluster, bundles will be relocated to the new resources. It is critical to allow sufficient time for bundle relocation and replication before removing old members. In addition to ConductR replicating bundles to new nodes, stateful bundles may require additional time to replicate application data. Removing a cluster member before application data has fully replicated can result in application data loss. Elasticsearch, for example, provides a cluster health endpoint for checking Elasticsearch cluster health.
```bash
curl -XGET 'http://localhost:9200/_cluster/health?pretty=true
```
Application specific data replication should also be monitored during an upgrade to prevent data loss.


Be certain to ensure that sufficient resources for all roles are provisioned. Stopping the sole member providing a role leaves no members for bundles requiring that role to relocate to. With no nodes to replicate to and run on, affected bundles would require re-loading of the bundle after the role was re-provisioned in order to run again.

To perform a per cluster upgrade, build a new cluster in isolation from the current running cluster. Once the new cluster is fully prepared, cut-over traffic using DNS, load balancers, routers, etc. Per cluster upgrades may require more complicated strategies for migrating data storage managed by the cluster.
