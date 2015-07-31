# Cluster Resiliency Considerations

In order to achieve cluster resilience, it’s recommended for the cluster to be separated into at least 3 distinct parts, e.g.

* If you are using Amazon Web Services or AWS EC2, this would be spreading the number of nodes evenly in 3 different availability zones.
* If you are hosting the machines yourself, spread the number of nodes evenly in 3 different networks if it’s possible to do so.
* Or if you only have 1 network, try to split the nodes among 3 different physical machine.

Separating the number of nodes evenly into 3 distinct parts provides the best chance of recovery when cluster partition does occur.
