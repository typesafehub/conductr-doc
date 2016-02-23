# Typesafe Monitoring

[Typesafe Monitoring](http://www.lightbend.com/products/typesafe-monitoring) is a suite of insight tools for monitoring Typesafe Reactive Platform (RP) applications. These tools gain visibility into ConductR and its bundles — to respond early to changes that could indicate problems, to tune a system, or to track down the cause of unexpected behavior.

This guide discusses ConductR specifics regarding Typesafe Monitoring. For a comprehensive description please visit [Typesafe Monitoring's documentation](http://resources.lightbend.com/monitoring/docs/home.html).

## Monitoring ConductR

ConductR is a Typesafe RP based application and has been configured to support Typesafe Monitoring by enabling some settings within its `conf/application.ini`.

### Enabling ConductR Monitoring for Takipi

To enable monitoring for ConductR with [Takipi](https://www.takipi.com/) you'll need an account by [signing up with them](https://app.takipi.com/). Once done you can enable ConductR (you'll need to do this for each node where ConductR runs):

``` bash
echo \
  -Dcinnamon.instrumentation=on \
  -J-agentlib:TakipiAgent \
  -Dtakipi.name="conductr" | \
  sudo tee -a /usr/share/conductr/conf/application.ini
sudo /etc/init.d/conductr restart
```

conductr-haproxy can also be monitored:

``` bash
echo \
  -Dcinnamon.instrumentation=on \
  -J-agentlib:TakipiAgent \
  -Dtakipi.name="conductr-haproxy" | \
  sudo tee -a /usr/share/conductr-haproxy/conf/application.ini
sudo /etc/init.d/conductr-haproxy restart
```

Note also that the Takipi agent will require installation at each ConductR node.

## Monitoring Bundles

You can also incorporate Typesafe Monitoring capabilities into your bundles. This section assumes you have [signed up with Takipi]((https://app.takipi.com/)) and installed its agent at each ConductR node.

### Enabling Bundle Monitoring for Takipi

Your developers declare additional properties for your bundle's configuration:

> Note that the following is only required if the agent and bundle name have not already been declared as properties for your developer's application or service. Monitoring configuration is typically left for deployment configuration though, so it may make more sense to use the following.

```scala
javaOptions in Bundle ++= Seq(
  "-J-agentlib:TakipiAgent", 
  s"""-Dtakipi.name="${(normalizedName in Bundle).value}""""
  )
```

Be sure to have your bundle developers become familiar with ["Cinnamon"](http://resources.lightbend.com/monitoring/docs/install/cinnamon.html) - Typesafe Monitoring's API.