# Developer quick start

ConductR simplifies the deployment of applications with resilience and elasticity without disrupting the application development lifecycle. Developers continue to develop and test their applications as they normally would prior to deployment.

This guide describes how to use a Play 2.4 Scala application to:

* Signal that application is started
* Create application bundle
* Start ConductR cluster
* Deploy application bundle to ConductR

The focus of this section is to get started quickly. The full documentation of these parts are described in these sections:

* [[Signaling application state|SignalingApplicationState]]
* [[Creating bundles|CreatingBundles]]
* [[Deploying bundles|DeployingBundles]]
* [[ConductR sandbox cluster|ConductrSandbox]]

## Vendor lock-in

A quick word on [vendor lock-in](https://en.wikipedia.org/wiki/Vendor_lock-in) as a philosophical point. We think it is important for you to avoid vendor lock-in as much as is reasonable. Furthermore all of the interfaces that we provide we do so in an open and transparent manner so that you can understand what is going on at all times.

We additionally open source the application/client-side libraries that use various ConductR APIs. You may find it useful to reference the following projects for further comprehension:

* https://github.com/typesafehub/conductr-bundle-lib
* https://github.com/typesafehub/conductr-doc
* https://github.com/sbt/sbt-bundle
* https://github.com/sbt/sbt-conductr
* https://github.com/typesafehub/sbt-conductr-sandbox

## Signal application readiness

Your application should tell ConductR when it has completed its initialization and is ready for work. For a Play 2.4 application add these dependency to your `build.sbt`:

```scala
resolvers += "typesafe-releases" at "http://repo.typesafe.com/typesafe/maven-releases"

libraryDependencies += "com.typesafe.conductr" %% "play24-conductr-bundle-lib" % "1.0.1"
```

Now you can override the `onStart` method to tell ConductR that your application has been started and is ready to start processing requests:

```scala
import com.typesafe.conductr.bundlelib.play.StatusService
import play.api.{ Application, GlobalSettings }
import com.typesafe.conductr.bundlelib.play.ConnectionContext.Implicits.defaultContext

object Global extends GlobalSettings {
  override def onStart(app: Application): Unit = {
    StatusService.signalStartedOrExit()
  }
}
```

## Create application bundle

[sbt-conductr](https://github.com/sbt/sbt-conductr) is an sbt plugin to easily manage your application bundle inside ConductR. This plugin includes [sbt-bundle](https://github.com/sbt/sbt-bundle#typesafe-conductr-bundle-plugin) which we will use to create the application bundle for our Play application. 

1. Add `sbt-conductr` to the `project/plugins.sbt`:

    ```scala
    addSbtPlugin("com.typesafe.conductr" % "sbt-conductr" % "1.1.0")
    ```
2. Enable the plugin in the `build.sbt`:  

    ```scala
    lazy val root = project.in(file(".")).enablePlugins(PlayScala)
    ```
3. Specify `sbt-bundle` keys in the `build.sbt`:   

    ```scala
    import ByteConversions._
    BundleKeys.nrOfCpus := 1.0
    BundleKeys.memory := 64.MiB
    BundleKeys.diskSpace := 10.MB
    BundleKeys.roles := Set("web")
    BundleKeys.endpoints := Map("my-app" -> Endpoint("http", services = Set(URI("http://:9000"))))
    BundleKeys.startCommand += "-Dhttp.address=$MY_APP_BIND_IP -Dhttp.port=$MY_APP_BIND_PORT"
    ```
4. Reload the sbt session:

    ```scala
    reload
    ```     
5. Create the bundle inside you sbt session with:

    ```scala
    bundle:dist
    ```

The new bundle should be created in your `target/bundle` directory. The `sbt-bundle` effectively describe what resources are used by your application and are used to determine which machine they will run on in the ConductR cluster.

As you move through our documentation you will  come across references to ConductR's environment variables e.g. `MY_APP_BIND_PORT`. Please refer to [our documentation](BundleEnvironmentVariables) for information on the meaning of these environment variables should you need to.

## Start ConductR cluster

In order to manage a ConductR cluster we provide a sbt plugin [sbt-conductr-sandbox](https://github.com/typesafehub/sbt-conductr-sandbox). Follow these steps to start the ConductR cluster.


1. Add the sbt plugin to the `project/plugins.sbt` of your project (the plugin is automatically enabled):

    ```scala
    addSbtPlugin("com.typesafe.conductr" % "sbt-conductr-sandbox" % "1.1.0")
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

## Deploy application bundle to ConductR

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

That's it! You now have ConductR running with the visualizer and your own application.
