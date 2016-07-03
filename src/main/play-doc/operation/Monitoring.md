# Lightbend Monitoring

[Lightbend Monitoring](http://www.lightbend.com/products/monitoring) is a suite of insight tools for monitoring Lightbend Platform applications. These tools gain visibility into ConductR and its bundles — to respond early to changes that could indicate problems, to tune a system, or to track down the cause of unexpected behavior.

This guide discusses ConductR specifics regarding Lightbend Monitoring. For a comprehensive description please visit [Lightbend Monitoring's documentation](http://monitoring.lightbend.com/docs/latest/home.html).

## Monitoring ConductR

ConductR is a Lightbend Platform based application and has been configured to support Lightbend Monitoring by enabling some settings within its `conf/application.ini`.

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

The same also goes for conductr-haproxy's `application.ini` and service.

Note also that the Takipi agent will require installation at each ConductR node.

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
