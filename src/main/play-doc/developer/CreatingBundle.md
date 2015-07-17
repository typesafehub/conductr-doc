# Creating application bundle

A bundle is the fundamental denomination of currency for ConductR. Bundles are a composition of components. There is typically just the one component which is your application or service. You can think of bundles in J2EE parlance as an EAR given that it constitutes an application or service's compile-time dependencies, but that is probably where the similarity ends.

We offer two methods of building a bundle:

* using an [sbt](http://www.scala-sbt.org/) plugin with your build; and/or
* using `shazar` (we invented that name!)

If you can make changes to the application or service that you bundle then `sbt-bundle` is what you will typically use. In fact you can even use sbt-bundle to produce bundles for other applications or services. However you may find yourself crafting a bundle from scratch and for the latter scenario. See the "legacy & third party bundles" section of the [bundles](Bundling-Existing.html) document for more information on that, and for a deep dive on bundles in general. For now, let's look at bundling a project that you have control of.

## sbt-bundle

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

Take a look in your `target/bundle` folder - you'll see your new bundle with a hash string, something like this:

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