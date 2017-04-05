# Configuring ConductR Cluster

During installation, ConductR registers a linux service named `conductr` and `conductr-agent` for ConductR Core and ConductR Agent respectively. The service is started automatically during boot-up.

## ConductR Core service user

The `conductr` service runs as the daemon user `conductr` in the user group `conductr`. When the service is started the first time it creates the user and group itself.

The `conductr` user executes the commands in the background without any shell. You can specify additional environment variables in `/etc/default/conductr`. This file will be sourced before the actual service gets started.

## ConductR Agent service user

The `conductr-agent` service runs as the daemon user `conductr-agent` in the user group `conductr-agent`. When the service is started the first time it creates the user and group itself.

The `conductr-agent` user executes the commands in the background without any shell. You can specify additional environment variables in `/etc/default/conductr-agent`. This file will be sourced before the actual service gets started.

## Configuring ConductR Core

The configuration file `/usr/share/conductr/conf/conductr.ini` is the primary configuration file for the ConductR Core service. This file is used to specify Core ConductR service settings, such as `-Dconductr.ip` used during installation. See the comments section of the `conductr.ini` file for more examples.

The ConductR Core service must be restarted after changes to the `conductr.ini` to take effect.

## Configuring ConductR Agent

The configuration file `/usr/share/conductr-agent/conf/conductr-agent.ini` is the primary configuration file for the ConductR Agent service. This file is used to specify ConductR Agent service settings, such as `-Dconductr.agent.roles` used during installation. See the comments section of the `conductr-agent.ini` file for more examples.

Akka module configuration can also be set using this file. For example, to assign a ConductR node the roles of `megaIOPS` and `muchMem` instead of the default, `web`, set `akka.cluster.roles` in `conductr-agent.ini`:

```bash
echo -Dconductr.agent.roles.0=megaIOPS | sudo tee -a /usr/share/conductr-agent/conf/conductr-agent.ini
echo -Dconductr.agent.roles.1=GPU | sudo tee -a /usr/share/conductr-agent/conf/conductr-agent.ini
sudo service conductr-agent restart
```

With this setting, the node would offer the roles `megaIOPS` and `GPU`. Only bundles with a `BundleKeys.roles` of `megaIOPS,` `GPU` or both `megaIOPS` and `GPU` will be loaded and run on this node.

The ConductR Agent service must be restarted after changes to the `conductr-agent.ini` to take effect. Role matching must also be enabled on the cluster core nodes for bundles to be scheduled by role.

## Roles

Roles allow machines to be targeted for specific purposes. Some machines may have greater IO capabilities than others, some may have more CPU, memory, and other resources. Some may also be required to maintain a volume that holds a database.

When getting started with ConductR, it is reasonable to start with ConductR not considering roles during scheduling. ConductR has the default settings `-Dconductr.resource-provider.match-offer-roles=off`.

However, when moving into a production scenario, you should plan and assign roles for your ConductR cluster. The roles can be enabled by modifying the settings `-Dconductr.resource-provider.match-offer-roles=on` as such.

```bash
echo \
  -Dconductr.resource-provider.match-offer-roles=on | \
  sudo tee -a /usr/share/conductr/conf/conductr.ini
sudo /etc/init.d/conductr restart
```

When a bundle is to be scheduled for loading or scaling, a check is made first to see whether a resource offer's roles intersect with the roles that the bundle requires. If it does, then it is eligible. If no resource offers provide the roles required by the bundle, the bundle cannot be loaded or scaled. Bundles will only be loaded to member nodes providing the bundle required roles. If no members of the cluster provide those roles, the bundle will fail to load.

### Using Roles

Roles can be leveraged in varying levels of specificity as needed to achieve the desired results. Small clusters running multiple apps will generally need only a few roles. Bundles need to be able to relocated to other nodes in the event of failure. Overly dividing a small cluster into small sub-sets reduces resilience.  Smaller clusters therefore will generally use few roles to create a few subsets of nodes.

Larger clusters, on the other hand, will generally want more specialization and therefore benefit further from the use of roles. Bundles with specific needs, such as resource intensive and data storage applications, will generally want exclusive use of a subset of nodes by using highly specific roles.

## Service Monitoring

For best resilience, the ConductR service daemons should be monitored and restarted in the event of failure. [sbt-native-packager](https://github.com/sbt/sbt-native-packager) has experimental [systemd support](http://www.scala-sbt.org/sbt-native-packager/archetypes/java_server/customize.html#systemd-support). As systemd support matures, ConductR will be made available as a package managed by systemd with a restart on failure enabled. Until that time, third party daemon monitors can be utilized to provide restart on failure.
