# Upgrading ConductR Cluster

A running cluster can be upgraded without downtime either at the node or cluster level.

On a per node basis, new updated nodes are introduced to the cluster while the old nodes are removed from the cluster. The per node update strategy applies to both ConductR Core node or ConductR Agent node.

On a per-cluster basis, incoming requests are directed from the old cluster to the new cluster using load balancer membership or DNS changes.

## Per node upgrade

Per node upgrades are generally easier. However, they cannot be performed across ConductR releases that are marked as not compatible with previous releases.

To perform a per node upgrade for ConductR Core, introduce new cluster members to the cluster. The new nodes could be created with updated versions of ConductR, Java, the Linux operating system or any other component installed during provisioning. As old members are removed from the cluster, bundles will be replicated to the new resources.

To perform a per node upgrade for ConductR Agent, connect the new agent node to an existing cluster. As old agent nodes are removed from the cluster, bundles will be rescheduled for execution on the new agent nodes.

To perform a per node upgrade for ConductR Core and ConductR Agent installed on the same host, connect both the ConductR Core and ConductR agent to an existing cluster. As old hosts (containing both old ConductR Core and ConductR Agent) are removed from the cluster, bundles will be replicated to the new resources, and bundles will be rescheduled for execution on the new agent nodes.

It is critical to allow sufficient time for bundle relocation and replication before removing old members. In addition to ConductR replicating bundles to new nodes, stateful bundles may require additional time to replicate application data. Removing an agent node before application data has fully replicated can result in application data loss. Elasticsearch bundle provided along with ConductR, for example, requires [verification](#Elasticsearch-Verification) to ensure the data is transferred into the new agent node.

Application specific data replication should also be monitored during an upgrade to prevent data loss.

Be certain to ensure that sufficient resources for all roles are provisioned. Stopping the sole agent node providing a role leaves no nodes for bundles requiring that role to relocate to.

## Per cluster upgrade

To perform a per cluster upgrade, build a new cluster in isolation from the current running cluster. Once the new cluster is fully prepared, cut-over traffic using DNS, load balancers, routers, etc. Per cluster, upgrades may require more complicated strategies for migrating data storage managed by the cluster.

## Elasticsearch Verification

To perform deployment on a running cluster without downtime, Elasticsearch requires the new node to join the Elasticsearch cluster, the primary shard(s) is relocated from the old node to the new node, and Elasticsearch master is re-elected if required. Elasticsearch has the means to perform automatic relocation of primary shard(s) from the old node to the new node.

Firstly, we will use Elasticsearch cluster health endpoint to ensure the cluster is in good health after the new node joined. Elasticsearch endpoints are exposed via the proxy node via port `9200` under `/elastic-search` path. Replace `10.0.1.250` with the ip address of the proxy node.

```bash
curl -XGET 'http://10.0.1.250:9200/elastic-search/_cluster/health?pretty=true'
```

The fields of interest are `status`, `number_of_nodes`, and `unassigned_shards`. The healthy cluster is indicated by the `status` having `green` value.

During the transfer between old node to the new node, the cluster health will transition from `green` to `yellow`, and back to `green` once the new node successfully joined.

It is crucial to wait for this transition to occur successfully.

Once this transition has occurred successfully, the `number_of_nodes` should display the number of running Elasticsearch nodes, and `unassigned_shards` should have the value of `0`. The `unassigned_shards` having the value of `0` means the primary shards has been successfully allocated to all members of the cluster, including the new node.

Next, we will use Elasticsearch endpoint to ensure master has been elected. This endpoint will display the name and IP address of the elected master within the Elasticsearch cluster.

```bash
curl -XGET 'http://10.0.1.250:9200/elastic-search/_cat/master'
```

Once these steps have been performed successfully, Elasticsearch cluster should be in a good working order.

## Recovering from Red Elasticsearch Cluster

Should the Elasticsearch cluster endpoint `status` have the `red` value, this means one or more shards have not been allocated to the member of the cluster.

The recovery process involves allocating the unassigned shards to the member of the cluster.

First, we will need to identify which shards are not assigned.

```bash
curl -XGET 'http://10.0.1.250:9200/elastic-search/_cat/shards'
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

We will distribute the shard `0`, `2`, and `4` between `Thin Man` and `Sam Wilson` to resolve this situation.

To distribute the shard `0` to `Thin Man`, we will invoke the cluster reroute endpoint.

```bash
curl -XPOST '10.0.1.250:9200/elastic-search/_cluster/reroute' -d '{
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

Ensure that the shard is allocated successfully by re-checking the shard allocation.
```bash
curl -XGET 'http://10.0.1.250:9200/elastic-search/_cat/shards'
```

The shard `0` should now be allocated to the node called `Thin Man`. Repeat these steps for each of the unassigned shards.

When allocating shards, ensure the shards are distributed as evenly as possible across all nodes of the Elasticsearch cluster. This will improve the resiliency of the cluster.

Once all the shards have been reallocated, the cluster health endpoint `status` should be back to `green` and the Elasticsearch cluster should be back in working order.
