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

Akka module configuration can also be set using this file. For example, to assign a ConductR node the role of `megaIOPS` instead of the default, `all-conductrs`, set `akka.cluster.roles` in application.ini:

```bash
 -Dakka.cluster.roles.0=“megaIOPS"
 ```
With this setting only bundles with a `BundleKeys.roles` of `megaIOPS` will be scheduled to execute on this node.

The ConductR service must be restarted for changes to this file to take effect.

## Roles

Roles allow machines to be targetted for specific purposes. Some machines may have greater IO capabilities than others, some may have more CPU, memory and other resources. Some may also be required to maintain a volume that holds a database.

When getting started with ConductR it is reasonable to have each ConductR service rely on its default role of "all-conductrs". However when moving into a production scenario you should plan and assign roles for your ConductR cluster.

When a bundle is to be scheduled for loading or scaling, a check is made to first see whether a resource offer has the "all-conductrs" role. If it does then it is eligible. However if it does not then it must have a roles that intersect with the roles that a bundle requires. Note that you should not assign the "all-conductrs" role to a member that has other roles declared as "all-conductrs" will supercede the others.


## Service Monitoring

For best resilience, the ConductR service daemons should be monitored and restarted in the event of failure. [sbt-native-packager](https://github.com/sbt/sbt-native-packager) has experimental [systemd support](http://www.scala-sbt.org/sbt-native-packager/archetypes/java_server/customize.html#systemd-support). As systemd support matures, ConductR will made available as a package manged by systemd with restart on failure enabled. Until that time, third party daemon monitors can be utilized to provide restart on failure.
