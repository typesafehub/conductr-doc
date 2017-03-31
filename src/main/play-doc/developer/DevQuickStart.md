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

* [Docker](https://www.docker.com/)
* [sbt](http://www.scala-sbt.org/download.html)
* [conductr-cli](CLI)

Docker is required so that you can run the ConductR cluster as if it were running on a number of machines in your network. You won't need to understand much about Docker for ConductR other than installing it as described in its "Get Started" section. If you are on Windows or Mac then you will become familiar with `docker-machine` which is a utility that controls a virtual machine for the purposes of running Docker.

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

Your application should tell ConductR when it has completed its initialization and is ready for work. Fortunately, `sbt-conductr` does this for a Play application automatically.

## Starting ConductR cluster

Now, you can go ahead and start the ConductR cluster locally. For that you should use the ConductR sandbox which is a docker image based on Ubuntu that includes ConductR. With this docker image you can easily spin up multiple ConductR nodes on your local machine. The `sandbox run` command will pick up and run this ConductR docker image. In order to use this command we need to specify the ConductR version. Please visit the [ConductR Developer page](https://www.lightbend.com/product/conductr/developer) to pick up the latest ConductR version from the section **Quick Configuration**.

Afterwards, you can start the ConductR cluster by executing the following from within sbt:

```scala
sandbox run <CONDUCTR_VERSION> --feature visualization
[info] Running ConductR...
[info] Running container cond-0 exposing 192.168.59.103:9909...
```

The `visualization` feature is simple Play application that visualizes the ConductR cluster state. Access the ConductR visualizer at `http://{docker-host-ip}:9909` where `docker-host-ip` is the host of your docker environment. For convenience, the url of the visualizer app is displayed in the sbt session, e.g. http://192.168.59.103:9909.

[[images/visualizer_simple.png]]

## Deploying application bundle to ConductR

From within sbt:

```scala
install
```

The above will introspect your project and any sub projects, generate "bundles" and their configuration, restart the sandbox to ensure a clean state and then load and run your application. You can then access your application at http://docker-host-ip:9000.

> Bundles and their configuration are tamperproof given a digest hash incorporated into their filename. ConductR will verify this hash against the one supplied in the filename when loading a bundle. With bundles and configuration then, you can roll releases forward and backward with a high degree of confidence.

In the visualizer web interface you should see now two bundles running, the visualizer bundle itself and your application bundle.

[[images/visualizer_with_app.png]]

That's it! You now have ConductR running with the visualizer and your own application. Head over to the next chapters to learn in greater detail how to setup, configure and run your applications on ConductR.

> Note that the `conduct` and `sandbox` commands are also available from the command line for usage outside of sbt.
