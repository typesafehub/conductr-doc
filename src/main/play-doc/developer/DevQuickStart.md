# Developer quick start

ConductR simplifies the deployment of applications with resilience and elasticity without disrupting the application development lifecycle. Developers continue to develop and test their applications as they normally would prior to deployment.

This guide describes how to setup and deploy a Play 2.5 Scala application on ConductR. In particular it describes how to:

* Signal that application has started
* Create an application bundle
* Start a ConductR cluster
* Deploy an application bundle to ConductR

This section focuses to get you up and running quickly. The full documentation of these parts are described in the sections:

* [[Signaling application state|SignalingApplicationState]]
* [[Creating bundles|CreatingBundles]]
* [[Deploying bundles|DeployingBundles]]
* [[ConductR sandbox cluster|ConductrSandbox]]

## Prerequisites

* [Docker](https://www.docker.com/) (when using Docker based bundles)
* [sbt](http://www.scala-sbt.org/download.html)
* [conductr-cli](CLI)

sbt is our interactive build tool. Reading the getting started guide for sbt is recommended.

The conductr-cli is used to communicate with the ConductR cluster.

## Adding sbt-conductr plugin

[sbt-conductr](https://github.com/typesafehub/sbt-conductr) is a sbt plugin that provides commands in sbt to:

* Produce a ConductR bundle
* Start and stop a local ConductR cluster
* Manage a ConductR cluster within a sbt session

To use `sbt-conductr` for your project add the plugin to your `project/plugins.sbt`:

```scala
addSbtPlugin("com.lightbend.conductr" % "sbt-conductr" % "2.3.2")
```

If your project is using Lagom 1.2.x or a previous version use:

```scala
addSbtPlugin("com.lightbend.conductr" % "sbt-conductr" % "2.2.7")
```

## Signaling application state

Your application should tell ConductR when it has completed its initialization and is ready for work. Fortunately, `sbt-conductr` does this for Play and Lagom application/services automatically.

## Starting ConductR cluster

Now, you can go ahead and start the ConductR cluster locally. For that you should use the ConductR sandbox so you can easily spin up multiple ConductR nodes on your local machine. The `sandbox run` command will pick up and run this ConductR docker image.

> Windows users are required to run the developer sandbox inside a Linux based Virtual Machine.

Afterwards, you can start the ConductR cluster by executing the following from within sbt:

```scala
sandbox run 2.0.0 --feature visualization
[info] Running ConductR...
[info] Running container cond-0 exposing 192.168.10.1:9999...
```

The `visualization` feature is simple Play application that visualizes the ConductR cluster state. Access the ConductR visualizer at `http://{sandbox-host-ip}:9909` where `sandbox-host-ip` is the host of your docker environment (`192.168.10.1` by default).

[[images/visualizer_simple.png]]

## Deploying application bundle to ConductR

From within sbt:

```scala
install
```

The above will introspect your project and any sub projects, generate "bundles" and their configuration, restart the sandbox to ensure a clean state and then load and run your application. You can then access your application at http://sandbox-host-ip:9000.

> Bundles and their configuration are tamperproof given a digest hash incorporated into their filename. ConductR will verify this hash against the one supplied in the filename when loading a bundle. With bundles and configuration then, you can roll releases forward and backward with a high degree of confidence.

In the visualizer web interface you should see now two bundles running, the visualizer bundle itself and your application bundle.

[[images/visualizer_with_app.png]]

That's it! You now have ConductR running with the visualizer and your own application. Head over to the next chapters to learn in greater detail how to setup, configure and run your applications on ConductR.

> Note that the `conduct` and `sandbox` commands are also available from the command line for usage outside of sbt.
