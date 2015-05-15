# Typesafe ConductR %PLAY_VERSION%

## Developer Quickstart

ConductR provides two services for you as the developer:

* the ability to signal when your application or service is ready to start processing; and
* the ability to resolve a URI describing where another service is located.

In addition ConductR provides a convenient packaging mechanism known as a bundle. Bundles compose components where a component is the application or service that you have developed. There is typically a one-to-one correlation between a bundle and a component with the ability to have multiple components for some interesting use-cases such as co-locating a persistence store with a microservice.

We provide an [sbt](http://www.scala-sbt.org/) plugin that conveniently bundles your application or service, and permits you to manage its lifecycle in ConductR (we also provide non-sbt tooling for this, as explained under _Operator Quickstart_ above).

### Signalling Ready

Your application or service should tell ConductR when it has completed its initialization and is ready for work. We have a fall-back strategy for where this is not possible or practical (more on that later), but for now, let's assume that you can modify your source code.

For a [Play](https://www.playframework.com/) application or service, signalling successful startup may be when its http endpoint is online having declared a connection to a database. For an [Akka](http://akka.io/) based application then it may be when your actor system has been initialized and you have connected to some other service.

ConductR's only requirement of you is to use a library and call a single function as `StatusService.signalStartedOrExit()`. Note that if you call this function outside of running within ConductR then it does nothing so you can continue to develop and debug as you have always done.

The library comes in multiple flavours: 
- Scala / JDK: `scala-conductr-bundle-lib`
- Akka: `akka23-conductr-bundle-lib`
- Play 2.3.x: `play23-conductr-bundle-lib`
- Play 2.4.x: `play24-conductr-bundle-lib`

To use it add one of the libraries as a dependency to your `build.sbt`:

```scala
resolvers += "typesafe-releases" at "http://repo.typesafe.com/typesafe/maven-releases"

libraryDependencies += "com.typesafe.conductr" %% "scala-conductr-bundle-lib" % "0.13.0"
```

The Scala / JDK library has no dependencies other than the JDK and as such, a blocking implementation is used for its HTTP calls (the JDK offers no non-blocking APIs for this). Using the Akka or Play library will ensure that the library is consistent with the respective Akka or Play application and that non-blocking implementations are used:
- Akka: akka-http
- Play: Play.WS

When you are reasonably sure that your code is ready to start processing (it doesn't have to be exactly at that time - ConductR tolerates services being unavailable), call the `signalStartedOrExit` function. 

**Scala Example**
Library: `scala-conductr-bundle-lib`

```scala
import com.typesafe.conductr.bundlelib.scala.ConnectionContext.Implicits.global

...

StatusService.signalStartedOrExit()
```

**Play example**
Library: `play23-conductr-bundle-lib`

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

Simple!

After reading the following sections, you may also wish to refer to [the reference documentation for conductr-bundle-lib](https://github.com/typesafehub/conductr-bundle-lib#typesafe-conductr-bundle-library).

### Resolving other bundle services

When you create a bundle (we'll show you how to do that in the next section) you can declare service names to ConductR. You can resolve a URL to the service if the service is running. For example if you have a RESTful `/accounts` service you can resolve it from within your application or service as using Play WS.

Firstly establish the URL for locating the accounts service. Note that your code should look toward factoring out this type of behavior so that you do not hard-code host addresses or ports. For this example though, we'll relax that recommendation for the sake of clarity:

```Scala
val accounts = LocationService.getLookupUrl("/accounts", "http://127.0.0.1:9000/accounts")
```

If the above code is run with your component has been started by ConductR then the `accounts` value will contain a URL to ConductR's service locator. Otherwise, when not running via ConductR then a service running on the loopback interface will be used; this latter one being something generally useful to you when in development mode, or an address for when running your component outside of ConductR.

You can then make a request:

```scala
WS
  .url(accounts)
  .withFollowRedirects(follow = true)
  .get()
  .map { response =>
    ...
  }
```

How easy is that!

Where you have non-HTTP services to locate, you can use the ServiceLocator API directly. See the section on [TCP and UDP service lookups](TCP-Lookups.html) for more information on that.

### Bundling your Application or Service

A bundle is the fundamental denomination of currency for ConductR. Bundles are a composition of components. There is typically just the one component which is your application or service. You can think of bundles in J2EE parlance as an EAR given that it constitutes an application or service's compile-time dependencies, but that is probably where the similarity ends.

We offer two methods of building a bundle:

* using an [sbt](http://www.scala-sbt.org/) plugin with your build; and/or
* using `shazar` (we invented that name!)

If you can make changes to the application or service that you bundle then `sbt-bundle` is what you will typically use. In fact you can even use sbt-bundle to produce bundles for other applications or services. However you may find yourself crafting a bundle from scratch and for the latter scenario. See the "legacy & third party bundles" section of the [bundles](Bundling-Existing.html) document for more information on that, and for a deep dive on bundles in general. For now, let's look at bundling a project that you have control of.

#### sbt-bundle

The [sbt-native-packager](https://github.com/sbt/sbt-native-packager#sbt-native-packager) has been extended with a plugin named [sbt-bundle](https://github.com/sbt/sbt-bundle#typesafe-conductr-bundle-plugin). You can declare your native packager to build a Universal or Docker package and have that become what is bundled.

Note that the description here is just to provide a feel of how `sbt-bundle` is used. Please refer to [its documentation](https://github.com/sbt/sbt-bundle#typesafe-conductr-bundle-plugin) as there are some small considerations when dealing with the pre 1.0 `sbt-native-packager` e.g. the one used with Play 2.3.

Firstly add the sbt plugin, typically to your project's `project/plugins.sbt` file (check [here](https://github.com/sbt/sbt-bundle#usage) for the latest release of sbt-bundle):

```scala
addSbtPlugin("com.typesafe.sbt" % "sbt-bundle" % "0.23.0")
```

You will then need to declare what are known as "scheduling parameters" for ConductR. These parameters effectively describe what resources are used by your application or service and are used to determine which machine they will run on. Here's a minimum set of parameter specifying that 1 cpu, 64MiB memory and 5MB of disk space is required when your application or service runs:

```scala
import ByteConversions._

BundleKeys.nrOfCpus := 1.0
BundleKeys.memory := 64.MiB
BundleKeys.diskSpace := 5.MB
```

You'll note that the international standards for supporting [binary prefixes](http://en.wikipedia.org/wiki/Binary_prefix), as in `MiB`, is supported.

Also note that while `BundleKeys.memory` is honored, the other two are ignored by ConductR when running in standalone mode i.e. when not running on a resource provider such as [Mesos](http://mesos.apache.org/). However it is good practice to specify these parameters, and future ConductR releases may indeed honor them.

Now create a bundle:

```scala
sbt bundle:dist
```

Take a look in your `target/typesafe-conductr` folder - you'll see your new bundle with a hash string, something like this:

```
visualizer-0.1.0-b07601b2bd015c94de0514ad698760d0396cb6f95881396d37981b271c0e7142.zip
```

i.e. the name of your project (`visualizer`), its version `0.1.0` and the hash value representing its contents (`b07601b2bd015c94de0514ad698760d0396cb6f95881396d37981b271c0e7142`).

Your bundle is now ready for loading into ConductR. What happened there is a few sensible defaults were chosen for you, a `bundle.conf` was generated, and a zip was built containing the output of a Universal build. The plugin then generated a [secure hash](http://en.wikipedia.org/wiki/Secure_Hash_Algorithm) of the zip contents and encoded it in the filename. The secure hash provides a version of your bundle that ConductR is able to verify and thus assure you of its integrity. You can derive a reasonable level of confidence that version xyz of your application or service is always going to be xyz; something that makes you happy if you ever need to reliably rollback to an older version of your software!

Your application or service must be told what interface and port it should bind to when it runs. *You should not assume these binding details.*

ConductR provides a specific ip address for the application to bind. Binding the provided private address enables the application to limit its network exposure to only the required interface. Furthermore, the port that ConductR provides for binding to is guaranteed to not clash with other bundle components on the same machine, and so it is important to use.

Here is a Play example of declaring the interface and port to bind to:

```scala
BundleKeys.startCommand += "-Dhttp.address=$WEB_BIND_IP -Dhttp.port=$WEB_BIND_PORT"
```

`WEB_BIND_IP` and `WEB_BIND_PORT` are environment variables provided by ConductR. ConductR will generate a few useful environment variables for your bundle component. Check out [sbt-bundle's documentation](https://github.com/sbt/sbt-bundle#typesafe-conductr-bundle-plugin) for a comprehensive statement of its capabilities and settings, particularly around configuring your service's endpoints.

By default, sbt-bundle will assume that you have a Play application and it will expose port 9000.

Here's another example, this time of a non-Play application such as an Akka one where [Typesafe config](https://github.com/typesafehub/config#overview) is used to determine the ip and port that your service requires:

```
customer-service {
  ip = "127.0.0.1"
  ip = ${?CUSTOMER_SERVICE_BIND_IP}
  port = 9000
  port = ${?CUSTOMER_SERVICE_BIND_PORT}
}

```

Typesafe config provides the ability to substitute environment variables if they exist. With the above, an ip of 127.0.0.1 and a port of 9000 will be used if ConductR has not been used to start your application. `CUSTOMER_SERVICE` was declared as the name of the endpoint using sbt configuration e.g.:


```scala
BundleKeys.endpoints := Map("customer-service" -> Endpoint("http", services = Set(URI("http://:5444/customers"))))
```

With the above Typesafe config you can then access the host and ip to use from within your application using code along the lines of following and using akka-http as an example:

```scala
  val ip = config.getString("customer-service.ip")
  val port = config.getInt("customer-service.port")
  Http(system).bind(ip, port) // ... and so forth

```
### Loading and Running your Application or Service

The fun part!

Once you've created a bundle you can deploy it using ConductR's RESTful API, even using [curl](http://curl.haxx.se/) if you want. However we've made it a little easier than that. You can of course use ConductR's CLI as discussed in the _Operator Quickstart_ above. Alternatively given that we've been discussing bundling your application or service mostly from an sbt perspective, you can use another plugin named [`sbt-conductr`](https://github.com/sbt/sbt-conductr#sbt-conductr).

The following description is intended to provide a taste of what `sbt-conductr` can do for you. Please refer to [its documentation](https://github.com/sbt/sbt-conductr/blob/master/README.md) as there are some small considerations when dealing with the pre 1.0 `sbt-native-packager` e.g. the one used with Play 2.3.

To use `sbt-conductr` first add the plugin your build (typically your `project/plugins.sbt` file); be sure to check at [the plugin's website](https://github.com/sbt/sbt-conductr#sbt-conductr) for the latest version to use:

```scala
addSbtPlugin("com.typesafe.conductr" % "sbt-conductr" % "0.36.0")
```

Note that if you add this plugin as above, you do not need to have an explicit declaration for `sbt-bundle`. `sbt-bundle` will be automatically added as a dependency of `sbt-conductr`.

The `sbt-conductr` plugin must then be enabled for your project. Supposing that your project has one module that will use the plugin which is the root of the sbt project (the most typical situation for a single `build.sbt`):

```scala
lazy val root = project
  .in(file("."))
  .enablePlugins(ConductRPlugin, <your other plugins go here>)
```

With your declarations out of the way, you can produce a bundle by typing:

```bash
bundle:dist
```

A bundle will be produced from the native packager settings of this project. A bundle effectively wraps a native
packager distribution and includes some component configuration. To load the bundle first declare the location of ConductR (supposing that ConductR is running on `172.14.0.1`:

```bash
controlServer 172.14.0.1
```

...and then load:

```bash
conduct load <HIT THE TAB KEY AND THEN RETURN>
```

Using the tab completion feature of sbt will produce a URI representing the location of the last distribution
produced by the native packager.

Hitting return will cause the bundle to be uploaded. On successfully uploading the bundle the plugin will report
the `BundleId` to use for subsequent commands on that bundle.

You can also run, stop and unload bundles by using this plugin. This may be useful to support your development lifecycle without having to jump into the operator's CLI.

That is all that is required in essence, but as stated, you should read [`sbt-conductr`'s documentation](https://github.com/sbt/sbt-conductr/blob/master/README.md) as there are a few additional requirements, particularly if you are managing a Play 2.3 application.

Now go and develop reactive applications or services for ConductR!
