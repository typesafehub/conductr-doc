# Configuring ConductR Cluster

How to configure a ConductR cluster depends on the ConductR mode. Choose one of the following sections depending on the mode:

* [Standalone mode](#Standalone-mode)
* [DC/OS mode](#DC/OS-mode)

## Standalone

During installation, ConductR registers two Linux services named `conductr` and `conductr-agent` for ConductR Core and ConductR Agent respectively. These services are started automatically during boot-up.

### ConductR Core service user

The `conductr` service runs as the daemon user `conductr` in the user group `conductr`. When the service is started for the first time it creates the user and group itself.

The `conductr` user executes the commands in the background without any shell. You can specify additional environment variables in `/etc/default/conductr`. This file will be sourced before the actual service gets started.

### ConductR Agent service user

The `conductr-agent` service runs as the daemon user `conductr-agent` in the user group `conductr-agent`. When the service is started the for first time it creates the user and group itself.

The `conductr-agent` user executes the commands in the background without any shell. You can specify additional environment variables in `/etc/default/conductr-agent`. This file will be sourced before the actual service gets started.

### Configuring ConductR Core

The configuration file `/usr/share/conductr/conf/conductr.ini` is the primary configuration file for the ConductR Core service. This file is used to specify Core ConductR settings, such as `-Dconductr.ip` used during installation. See the comments section of the `conductr.ini` file for more examples.

The ConductR Core service must be restarted after changes to the `conductr.ini` to take effect.

### Configuring ConductR Agent

The configuration file `/usr/share/conductr-agent/conf/conductr-agent.ini` is the primary configuration file for the ConductR Agent service. This file is used to specify ConductR Agent settings, such as `-Dconductr.agent.roles` used during installation. See the comments section of the `conductr-agent.ini` file for more examples.

Akka module configuration can also be set using this file. For example, to assign a ConductR node the roles of `megaIOPS` and `muchMem` instead of the default, `web`, set `akka.cluster.roles` in `conductr-agent.ini`:

```bash
echo -Dconductr.agent.roles.0=megaIOPS | sudo tee -a /usr/share/conductr-agent/conf/conductr-agent.ini
echo -Dconductr.agent.roles.1=GPU | sudo tee -a /usr/share/conductr-agent/conf/conductr-agent.ini
sudo service conductr-agent restart
```

With this setting, the agent would offer the roles `megaIOPS` and `GPU`. Only bundles with a `BundleKeys.roles` of `megaIOPS,` `GPU` or both `megaIOPS` and `GPU` will be loaded and run on this agent.

The ConductR Agent service must be restarted after changes to the `conductr-agent.ini` to take effect. [Role matching](#Roles) must also be enabled on the cluster core nodes for bundles to be scheduled by role.

### Roles

Roles allow machines to be targeted for specific purposes. Some machines may have greater IO capabilities than others, some may have more CPU, memory, and other resources. Some may also be required to maintain a volume that holds a database.

When getting started with ConductR, it is reasonable to start with ConductR not considering roles during scheduling. ConductR has the default settings `-Dconductr.resource-provider.match-offer-roles=off`.

However, when moving into a production scenario, you should plan and assign roles for your ConductR cluster. The roles can be enabled by modifying the settings `-Dconductr.resource-provider.match-offer-roles=on` as such.

```bash
echo \
  -Dconductr.resource-provider.match-offer-roles=on | \
  sudo tee -a /usr/share/conductr/conf/conductr.ini
sudo /etc/init.d/conductr restart
```

When a bundle is to be scheduled for loading or scaling, a check is made first to see whether a resource offer's roles intersect with the roles that the bundle requires. If it does, then the bundle is eligible. If no resource offers provide the roles required by the bundle, the bundle cannot be loaded or scaled. Bundles will only be loaded or scaled to member nodes providing the bundle required roles. If no members of the cluster provide those roles, the bundle will fail to load or scale.

#### Using Roles

Roles can be leveraged in varying levels of specificity as needed to achieve the desired results. Small clusters running multiple apps will generally need only a few roles. Bundles need to be able to relocated to other nodes in the event of failure. Overly dividing a small cluster into small sub-sets reduces resilience.  Smaller clusters therefore will generally use few roles to create a few subsets of nodes.

Larger clusters, on the other hand, will generally want more specialization and therefore benefit further from the use of roles. Bundles with specific needs, such as resource intensive and data storage applications, will generally want exclusive use of a subset of nodes by using highly specific roles.

### Service Monitoring

For best resilience, the ConductR service daemons should be monitored and restarted in the event of failure. [sbt-native-packager](https://github.com/sbt/sbt-native-packager) has experimental [systemd support](http://www.scala-sbt.org/sbt-native-packager/archetypes/java_server/customize.html#systemd-support). As systemd support matures, ConductR will be made available as a package managed by systemd with a restart on failure enabled. Until that time, third party daemon monitors can be utilized to provide restart on failure.

## DC/OS mode

On DC/OS, the configuration of ConductR Core and Agent is performed via the DC/OS Services UI page.

### Configuring ConductR Core

Open the `Edit Service` page of the ConductR service by navigating to:

```
http://dcos-host/#services >> Services >> conductr >> Edit
```

The section `General` contains a text field `Command` in which you can provide additional configuration by overriding ConductR Core settings and by specifying environment variables.

To override ConductR Core settings add an argument to the `Command` text field, e.g. changing the log level to debug with `-Dakka.loglevel=DEBUG`. For a full list of settings that can be overridden check out the [ConductR Core Configuration Reference](ConfigurationRef#ConductR-Core) page.

Providing additional environment variables can be useful if a ConductR Core setting defaults to an environment variable, e.g.

```
conductr {
  ip = ${?LIBPROCESS_IP}
}
```

If an environment variable is specified at the beginning of the `Command` text field, then the value will be used in the respective ConductR core setting.

```
export LIBPROCESS_IP="10.10.10.10"
```

Environment variables that are set in the context of Marathon, e.g. `MARATHON_APP_ID` are also sourced by ConductR Core.

To apply the configuration changes the ConductR cluster need to be restarted. Follow these [instructions](ClusterRestart#DC/OS-mode) to restart the ConductR cluster.

### Configuring ConductR Agent

Open the `Edit Service` page of the ConductR service by navigating to:

```
http://dcos-host/#services >> Services >> conductr >> Edit
```

The section `General` contains a text field `Command` in which you can provide additional configuration by overriding ConductR Agent settings and by specifying environment variables.

ConductR Agent executor tasks are started with the `conductr.mesos-scheduler-client.mesos-executor.start-command` command. On DC/OS, the default value is:

```
# The start command of a Mesos executor (the conduct agent) that gets invoked on executor startup.
conductr.mesos-scheduler-client.mesos-executor.start-command='GLOBIGNORE='"'"'*.tar.gz:*.tgz'"'"' && export JAVA_HOME=$(echo $(pwd)/jre*) && ./conductr-agent-*/bin/conductr-agent'
```

To specify additional ConductR Agent settings, add your custom setting to the start command in the `Command` text field of the `Edit Service` page:

```
conductr.mesos-scheduler-client.mesos-executor.start-command='GLOBIGNORE='"'"'*.tar.gz:*.tgz'"'"' && export JAVA_HOME=$(echo $(pwd)/jre*) && ./conductr-agent-*/bin/conductr-agent -Dakka.loglevel=DEBUG"'
```

For a full list of settings that can be overridden check out the [ConductR Agent Configuration Reference](ConfigurationRef#ConductR-Agent-Configuration) page.

Providing additional environment variables can be useful if a ConductR Agent setting defaults to an environment variable, e.g.

```
conductr.agent {
  ip = ${?LIBPROCESS_IP}
}
```

To declare an environment variable that is sourced by ConductR Agent, add the variable to the `Environment Variable` section of the DC/OS `Edit Service` page.

To apply the configuration changes, the ConductR cluster needs to be restarted. Follow these [instructions](ClusterRestart#DC/OS-mode) to restart the ConductR cluster.

### Roles

On DC/OS, role matching is by default activated with the option:

```
conductr.resource-provider.match-offer-roles = on
```

When a bundle is to be scheduled for loading or scaling, a check is made first to see whether a resource offer's roles intersect with the roles that the bundle requires. If it does, then the bundle is eligible. If no resource offers provide the roles required by the bundle, the bundle cannot be loaded or scaled. Bundles will only be loaded or scaled to member nodes providing the bundle required roles. If no members of the cluster provide those roles, the bundle will fail to load or scale.

By leveraging roles, you can decide on DC/OS whether your bundles run on a public or private agent node.

#### Private agent node

A private agent node does not allow ingress from outside of the cluster via the cluster’s infrastructure networking. The Mesos role on each private agent is `*`, meaning that by default every bundle is scheduled only on the private agents.

#### Public agent node

A public agent node is on a network that allows ingress from outside of the cluster via the cluster's infrastructure networking. The Mesos role on each public agent is `slave_public`. By default, only the `conductr-haproxy` bundle is configured to run on public nodes.

To load and schedule your bundle on a public node follow these steps:

1. Add a role to your `bundle.conf`, e.g. by setting `BundleKeys.roles` in the `build.sbt`:

    ```
    BundleKeys.roles := Set("my-public-role")
    ```
2. Provide role mapping configuration from the ConductR role to a Mesos role in the `Command` text field of the DC/OS `Edit Service page`:

    ```
    -Dconductr.mesos-scheduler-client.mesos-roles.bundle-role-mappings.cpus.my-public-role=slave_public \
    -Dconductr.mesos-scheduler-client.mesos-roles.bundle-role-mappings.mem.my-public-role=slave_public \
    -Dconductr.mesos-scheduler-client.mesos-roles.bundle-role-mappings.ports.my-public-role=slave_public \
    -Dconductr.mesos-scheduler-client.mesos-roles.bundle-role-mappings.disk.my-public-role=slave_public
    ```

 ConductR will then map the ConductR role specified in your bundle to the respective Mesos `slave_public` role. As a result, only resource offers from public agent nodes are eligible for this bundle, and therefore the bundle is loaded and scaled only on public agent nodes.
