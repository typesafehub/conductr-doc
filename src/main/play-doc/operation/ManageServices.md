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

To perform a per node upgrade, introduce new cluster members to the cluster. The new nodes could be created with updated versions of Conductr, Java, the Linux operating system or any other component installed during provisioning. As old members are removed from the cluster, bundles will be relocated to the new resources. It is critical to allow sufficient time for bundle relocation and replication before removing old members. In addition to ConductR replicating bundles to new nodes, stateful bundles may require additional time to replicate application data. Removing a cluster member before application data has fully replicated can result in application data loss. Elasticsearch bundle provided along with ConductR, for example, requires [verification](#Elasticsearch Verification) to ensure the data is transferred into the new member.

Application specific data replication should also be monitored during an upgrade to prevent data loss.


Be certain to ensure that sufficient resources for all roles are provisioned. Stopping the sole member providing a role leaves no members for bundles requiring that role to relocate to. With no nodes to replicate to and run on, affected bundles would require re-loading of the bundle after the role was re-provisioned in order to run again.

To perform a per cluster upgrade, build a new cluster in isolation from the current running cluster. Once the new cluster is fully prepared, cut-over traffic using DNS, load balancers, routers, etc. Per cluster upgrades may require more complicated strategies for migrating data storage managed by the cluster.

## Elasticsearch Verification

To perform deployment on a running cluster without downtime, Elasticsearch requires the new node to join the Elasticsearch cluster, the primary shard(s) is relocated from the old node to the new node, and Elasticsearch master is re-elected if required. Elasticsearch has the means to perform automatic relocation of primary shard(s) from the old node to the new node.

Firstly, we will use Elasticsearch cluster health endpoint to ensure the cluster is in a good health after the new node joined.

```bash
curl -XGET 'http://localhost:9200/_cluster/health?pretty=true'
```

The fields of interest are `status`, `number_of_nodes`, and `unassigned_shards`. The healthy cluster is indicated by the `status` having `green` value.

During the transfer between old node to the new node, the cluster health will transition from `green` to `yellow`, and back to `green` once the new node successfully joined.

It is crucial to wait for this transition to occur successfully.

Once this transition has occurred successfully, the `number_of_nodes` should display the number of running Elasticsearch nodes, and `unassigned_shards` should have the value of `0`. The `unassigned_shards` having the value of `0` means the primary shards has been successfully allocated to all members of the cluster, including the new node.

Next we will use Elasticsearch endpoint to ensure master has been elected. This endpoint will display the name and IP address of the elected master within the Elasticsearch cluster.

```bash
curl -XGET 'http://localhost:9200/_cat/master'
```

Once these steps has been performed with successful result, Elasticsearch cluster should be in a good working order.

## Recovering from Red Elasticsearch Cluster

Should Elasticsearch cluster endpoint `status` has the `red` value, this means one or more shard has not been allocated to the member of the cluster.

The recovery process involves allocating the unassigned shards to the member of the cluster.

First, we will need to identify which shards are not assigned.

```bash
curl -XGET 'http://localhost:9200/_cat/shards'
```

Here is an example output which displays unassigned shards.

```
conductr 2 p UNASSIGNED
conductr 2 r UNASSIGNED
conductr 2 r UNASSIGNED
conductr 0 p UNASSIGNED
conductr 0 r UNASSIGNED
conductr 0 r UNASSIGNED
conductr 3 p STARTED    199989 91.1mb 10.0.3.39  Spectra
conductr 3 r STARTED    199989 91.2mb 10.0.1.176 Thin Man
conductr 3 r STARTED    199989   91mb 10.0.5.83  Sam Wilson
conductr 1 p STARTED    200045 91.7mb 10.0.3.39  Spectra
conductr 1 r STARTED    200045 91.8mb 10.0.1.176 Thin Man
conductr 1 r STARTED    200045 91.9mb 10.0.5.83  Sam Wilson
conductr 4 p UNASSIGNED
conductr 4 r UNASSIGNED
conductr 4 r UNASSIGNED
```

In the example output we can see primary shard `0`, `2`, and `4` are not assigned. We can also see primary shard `1` and `3` are assigned to the node called `Spectra`, while the node called `Thin Man` and `Sam Wilson` does not have any primary shards assigned.

We will distribute shard `0`, `2`, and `4` between `Thin Man` and `Sam Wilson` to resolve this situation.

To distribute shard `0` to `Thin Man`, we will invoke the cluster reroute endpoint.

```bash
curl -XPOST 'localhost:9200/_cluster/reroute' -d '{
     "commands" : [ {
           "allocate" : {
               "index" : "conductr",
               "shard" : 0,
               "node" : "Thin Man",
               "allow_primary" : true
           }
         }
     ]
 }'
```

Ensure shard is allocated successfully by re-checking the shard allocation.
```bash
curl -XGET 'http://localhost:9200/_cat/shards'
```

The shard `0` should now be allocated to the node called `Thin Man`. Repeat these steps for each of the unassigned shards.

When allocating shards, ensure the shards are distributed as evenly as possible across all nodes of the Elasticsearch cluster. This will improve the resiliency of the cluster.

Once all the shards has been reallocated, the cluster health endpoint `status` should be back to `green` and the Elasticsearch cluster should be back in a working order.
