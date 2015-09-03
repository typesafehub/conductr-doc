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
