# Setup sbt-conductr

[sbt-conductr](https://github.com/typesafehub/sbt-conductr) is a sbt plugin that provides commands in sbt to:

* Produce a ConductR bundle and bundle configuration
* Start and stop a local ConductR cluster
* Manage a ConductR cluster within a sbt session

## Prerequisites

* [Docker](https://www.docker.com/)
* [conductr-cli](CLI#New-CLI-installation)

Docker is required so that you can run the ConductR cluster as if it were running on a number of machines in your network. You won't need to understand much about Docker for ConductR other than installing it as described in its "Get Started" section. If you are on Windows or Mac then you will become familiar with `docker-machine` which is a utility that controls a virtual machine for the purposes of running Docker.

The conductr-cli is used to communicate with the ConductR cluster.

## Setup

Add sbt-conductr to your `project/plugins.sbt`:

```scala
addSbtPlugin("com.lightbend.conductr" % "sbt-conductr" % "2.1.16")
```

This makes several commands available within sbt. We'll use these commands on the next pages.
