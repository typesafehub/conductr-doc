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

## Service Monitoring

For best resilience, the ConductR service daemons should be monitored and restarted in the event of failure. [sbt-native-packager](https://github.com/sbt/sbt-native-packager) has experimental [systemd support](http://www.scala-sbt.org/sbt-native-packager/archetypes/java_server/customize.html#systemd-support). As systemd support matures, ConductR will made available as a package manged by systemd with restart on failure enabled. Until that time, third party daemon monitors can be utilized to provide restart on failure.
