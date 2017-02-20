# Setup sbt-conductr

[sbt-conductr](https://github.com/typesafehub/sbt-conductr) is a sbt plugin that provides commands in sbt to:

* Produce a ConductR bundle and bundle configuration
* Start and stop a local ConductR cluster
* Manage a ConductR cluster within a sbt session

## Prerequisites

* [Docker](https://www.docker.com/) (when using Docker based bundles)
* [conductr-cli](CLI#New-CLI-installation)

The conductr-cli is used to communicate with the ConductR cluster. The CLI also includes a "developer sandbox" so that you may test your bundles in a production-like environment, but locally on your machine.

> Note that Windows users will require a Linux based virtual machine to run the developer sandbox.

## Setup

Add sbt-conductr to your `project/plugins.sbt`:

```scala
addSbtPlugin("com.lightbend.conductr" % "sbt-conductr" % "2.2.4")
```

This makes several commands available within sbt. We'll use these commands on the next pages.
