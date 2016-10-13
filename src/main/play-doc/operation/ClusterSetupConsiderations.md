# Setting up for non-production

Sometimes it is useful to setup a Linux environment that is close to production, but is not production, for example a QA environment. While production environments are configured to stay up indefinitely, non-production environments may come and go when not in use. This is often the case in order to save money, say, with EC2.

When setting up a non-production environment be sure to disable a ConductR node's ability to automatically re-form the cluster it had prior to being shutdown. While this feature is useful for individual production nodes (it provides self-healing after a split-brain style scenario where a minority of nodes are restarted automatically), for non-production environments it can prevent an entire cluster from re-forming given a completely fresh restart. Again, fresh restarts are typical for non-production style environments.

In order to prevent ConductR from attempting to recover its former cluster, uncomment the `--seed-node-file-disabled` option within the `application.ini` of each ConductR installation.

# Cluster resiliency considerations

In order to achieve cluster resilience, it’s recommended for the cluster to be separated into at least 3 distinct parts, e.g.

* If you are using Amazon Web Services or AWS EC2, this would be spreading the number of nodes evenly in 3 different availability zones.
* If you are hosting the machines yourself, spread the number of nodes evenly in 3 different networks if it’s possible to do so.
* Or if you only have 1 network, try to split the nodes among 3 different physical machine.

Separating the number of nodes evenly into 3 distinct parts provides the best chance of recovery when cluster partition does occur.

# Cluster security considerations

Access to the cluster is controlled by network port access. The ConductR daemon ports of 9004-9008 must only be opened to trusted networks and hosts. In general, the cluster nodes should reside in their own Virtual LAN (VLAN). For AWS EC2 deployments this VLAN is the node security group, which we have called SG-Nodes.

The ability to deploy and manage bundles thus requires the ability to access the ConductR node VLAN. This can be achieved by enabling access to the node VLAN from a trusted network or by establishing a VPN connection into the node VLAN. When using a trusted network, a [bastion host](https://en.wikipedia.org/wiki/Bastion_host) placed in the trusted network can provide operators easy, secure access to cluster management tools. Only authorized operators should be given access to the bastion host.

For EC2 deployments, the bastion host will be placed in a different security group from the nodes. We'll call this SG-Ops. SG-Ops must be in the same VPC as SG-Nodes. SG-Ops should only allow select public port access. Most commonly only SSH port 22 will be opened. The SG-Nodes security group can then be further restricted to allow access from the bastion SG-Ops and the load balancer SG-ELB, with no direct public access.

The bastion host can be also be used as the controller host for installation and maintenance. Deployment bundles can be uploaded to the bastion host from build servers to provide a final gated 'one command' deployment step to a continuous deployment pipeline.

# Managing Ports and Paths

It is generally preferred to expose as few ports as required into the cluster from the public network. Ports should be limited to `80` and `443` in general. Most all cluster bundles are thus differentiated on the internet by host name or path. Exceptions to this include internal service ports, e.g. `5444`.  These ports might be used for services only to be used internally, with an internal facing load balancer, that is not exposed to the internet.

We'll also generally use few edge routers. Management of the deployment, from TLS certificates, to DNS and other domain specific functions can provide compelling reason to deploy over multiple edge routers. For example, an ELB for `domain1` and another ELB for `domain2`.

Differentiation of bundle location by path is particularly useful in larger systems wishing to expose fewer hostnames. Developers should note the ability to [preserve paths](CreatingBundles#Preserving-paths-at-the-proxy) at the proxy when applications require such.

Use the `conduct services` command from the [[ConductR CLI|CLI]] to see what host name or port and path currently deployed bundles provide. Note that bundles without named endpoints are not listed by `conduct services`.
