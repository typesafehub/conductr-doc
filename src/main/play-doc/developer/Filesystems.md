# Filesystems

File systems in distributed systems should be regarded as emphemeral. Your application or service can come and go for many reasons e.g. as the result of scaling up and down, a network partition occuring and then a ConductR node being ejected from the cluster and so forth.

When the sbt-native-packager is used i.e. when using sbt-bundle, and the default script for starting your application or service is invoked, it will see that the current working directory as that of your bundle. For example, a bundle may have the following layout:

```
bundle.conf
myapp
  bin
  conf
  lib
```

In the above case, `bundle.conf` and the `myapp` directory will be within the current working directory for any component that runs. The JDK `user.dir` property correctly yields this location. Therefore if you need to access files within your bundle you can do so relative to `user.dir` e.g.:

```scala
new File(sys.props("user.dir"))
```

When ConductR starts your bundle's components it will unzip into a temporary location on the disk of ConductR's host. When your bundle's components are entirely stopped, then that location will be removed from disk. Do not rely on data written being available from there again.

Your bundle's components may write to other areas of your host's file system unless it is running within Docker. If your bundle is running within a Docker container then the file system your bundle component sees is the one that Docker provides (which will certainly disappear between invocations).

Bundle components are started under the `conductr:conductr` group/user when run outside of Docker (Docker launches your component as root by default). Therefore you must ensure that `conductr:conductr` has the correct permissions to access the host file system. When writing to your host file system we encourage you to use `/var` as per the [conventions on Unix style systems](https://en.wikipedia.org/wiki/Filesystem_Hierarchy_Standard).

## Persisting State

In a distributed application, there should be an emphasis on replicating state between multiple instances. To this end, writing to the file system should become less important. For example, take the [postgres-bdr](http://2ndquadrant.com/en/resources/bdr/) project. postgres-bdr will certainly write to the local file system, but it will also replicate database state across the number of instances of it that are running. Thus the file system of one becomes less important given that the data is replicated.

akka-cluster can facilitate sharing data also, particularly given the forthcoming akka-distributed-data project which is currently available for Akka 2.3 as [akka-data-replication] (https://github.com/patriknw/akka-data-replication).
