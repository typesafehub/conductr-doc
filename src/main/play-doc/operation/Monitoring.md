# Lightbend Monitoring

[Lightbend Monitoring](http://www.lightbend.com/products/monitoring) is a suite of insight tools for monitoring Lightbend Platform applications. These tools gain visibility into ConductR and its bundles — to respond early to changes that could indicate problems, to tune a system, or to track down the cause of unexpected behavior.

This guide discusses ConductR specifics regarding Lightbend Monitoring. For a comprehensive description please visit [Lightbend Monitoring's documentation](http://monitoring.lightbend.com/docs/latest/home.html).

## Monitoring ConductR

ConductR is a Lightbend Platform based application and has been configured to support Lightbend Monitoring by enabling some settings within its configuration files.

### Enabling ConductR Monitoring for Takipi

To enable monitoring for ConductR with [Takipi](https://www.takipi.com/) you'll need an account by [signing up with them](https://app.takipi.com/). Once done, you can enable it for ConductR Core and Agent:

#### Standalone mode

Change the ConductR Core configuration (you'll need to do this for each node where ConductR Core runs):

```bash
echo \
  -Dcinnamon.instrumentation=on \
  -J-agentlib:TakipiAgent \
  -Dtakipi.name="conductr" | \
  sudo tee -a /usr/share/conductr/conf/conductr.ini
sudo /etc/init.d/conductr restart
```

Additionally, change the ConductR Agent configuration (you'll need to do this for each node where ConductR Agent runs):

```bash
echo \
  -Dcinnamon.instrumentation=on \
  -J-agentlib:TakipiAgent \
  -Dtakipi.name="conductr" | \
  sudo tee -a /usr/share/conductr-agent/conf/conductr-agent.ini
sudo /etc/init.d/conductr-agent restart
```

Note also that the Takipi agent will require installation at each node where ConductR Core and ConductR Agent is running.

#### DC/OS mode

Change the ConductR Core and Agent configuration by adding the following settings via the DC/OS Services UI page.

```bash
-Dcinnamon.instrumentation=on \
-J-agentlib:TakipiAgent \
-Dtakipi.name="conductr" \
-Dconductr.mesos-scheduler-client.mesos-executor.start-command='GLOBIGNORE='"'"'*.tar.gz:*.tgz'"'"' && export JAVA_HOME=$(echo $(pwd)/jre*) && ./conductr-agent-*/bin/conductr-agent -Dcinnamon.instrumentation=on -J-agentlib:TakipiAgent -Dtakipi.name=conductr"'
```

For more information how to specify custom configuration in DC/OS go to [Cluster Configuration in DC/OS].

Note also that the Takipi agent will require installation at each node where ConductR Core and ConductR Agent is running.


## Monitoring Bundles

You can also incorporate Lightbend Monitoring capabilities into your bundles. This section assumes you have [signed up with Takipi]((https://app.takipi.com/)) and installed its agent at each ConductR node.

### Enabling Bundle Monitoring for Takipi

Your developers declare additional properties for your bundle's configuration:

> Note that the following is only required if the agent and bundle name have not already been declared as properties for your developer's application or service. Monitoring configuration is typically left for deployment configuration though, so it may make more sense to use the following.

```scala
javaOptions in Bundle ++= Seq(
  "-J-agentlib:TakipiAgent",
  s"""-Dtakipi.name="${(normalizedName in Bundle).value}""""
  )
```

Be sure to have your bundle developers become familiar with ["Cinnamon"](http://monitoring.lightbend.com/docs/latest/home.html) - Lightbend Monitoring's API.
