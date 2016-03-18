# Developer quick start

ConductR simplifies the deployment of applications with resilience and elasticity without disrupting the application development lifecycle. Developers continue to develop and test their applications as they normally would prior to deployment.

This guide describes how to setup and deploy a Play 2.4 Scala application on ConductR. In particular it describes how to:

* Signal that application is started
* Create an application bundle
* Start a ConductR cluster
* Deploy an application bundle to ConductR

The focus of this section is to get started quickly. The full documentation of these parts are described in these sections:

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

## Signaling application state

First let us setup the Play 2.4 application for ConductR. Your application should tell ConductR when it has completed its initialization and is ready for work. For a Play 2.4 application add these dependency to your `build.sbt`:

```scala
resolvers += "typesafe-releases" at "http://repo.typesafe.com/typesafe/maven-releases"

libraryDependencies += "com.typesafe.conductr" %% "play24-conductr-bundle-lib" % "1.4.1"
```

Now you can add a guice module in the `application.conf`. This module tells ConductR when your application has been started and therefore ready to start processing requests:

```scala
play.application.loader = "com.typesafe.conductr.bundlelib.play.ConductRApplicationLoader"
```

## Creating application bundle

[sbt-conductr-sandbox](https://github.com/typesafehub/sbt-conductr-sandbox) is an sbt plugin to easily manage your application inside ConductR. This plugin includes [sbt-bundle](https://github.com/sbt/sbt-bundle#typesafe-conductr-bundle-plugin) and [sbt-conductr](https://github.com/sbt/sbt-conductr).

1. Add `sbt-conductr-sandbox` to the `project/plugins.sbt`:

    ```scala
    addSbtPlugin("com.typesafe.conductr" % "sbt-conductr-sandbox" % "1.4.2")
    ```
2. Specify `sbt-bundle` keys in the `build.sbt`:   

    ```scala
    import ByteConversions._
    BundleKeys.nrOfCpus := 1.0
    BundleKeys.memory := 64.MiB
    BundleKeys.diskSpace := 10.MB
    ```    
3. Reload the sbt session:

    ```scala
    reload
    ```     
4. Create the bundle inside you sbt session with:

    ```scala
    bundle:dist
    ```

The new bundle should be created in your `target/bundle` directory. The `sbt-bundle` effectively describe what resources are used by your application and are used to determine which machine they will run on in the ConductR cluster.

As you move through our documentation you will  come across references to ConductR's environment variables e.g. `MY_APP_BIND_PORT`. Please refer to [our documentation](BundleEnvironmentVariables) for information on the meaning of these environment variables should you need to.

## Starting ConductR cluster

Now we can go ahead an start the ConductR cluster locally.

1. Specify the ConductR Developer Sandbox version in the `build.sbt`. This version is available gratis during development with registration at lightbend.com. Please visit the [ConductR Developer page](http://www.lightbend.com/product/conductr/developer) to retrieve the current version:

    ```
    SandboxKeys.imageVersion in Global := "YOUR_CONDUCTR_SANDBOX_VERSION"
    ```
2. Reload the sbt session:

    ```scala
    reload
    ``` 
        
3. Start ConductR cluster with visualization feature:
    
    ```scala
    [my-app] sandbox run --withFeatures visualization
    [info] Running ConductR...
    [info] Running container cond-0 exposing 192.168.59.103:9909...
    ```
4. Access ConductR visualizer at `http://{docker-host-ip}:9909` where `docker-host-ip` is the host of your docker environment. For convenience, the url of the visualizer app is displayed in the sbt session, e.g. http://192.168.59.103:9909.

[[images/visualizer_simple.png]]

## Deploying application bundle to ConductR

1. Load your application bundle to ConductR:
    
    ```scala
    conduct load <HIT THE TAB KEY AND THEN RETURN>
    ```
2. Run bundle on one node:
    
    ```scala
    conduct run my-app
    ```

3. Access your application at http://docker-host-ip:9000.

In the visualizer web interface you should see now two bundles running, the visualizer bundle itself and your application bundle.

[[images/visualizer_with_app.png]]

That's it! You now have ConductR running with the visualizer and your own application. Head over to the next chapters to learn in greater detail how to setup, configure and run your applications on ConductR.
