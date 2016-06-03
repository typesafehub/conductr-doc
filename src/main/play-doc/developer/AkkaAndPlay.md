# Bundle library flavors

The [ConductR bundle library](https://github.com/typesafehub/conductr-lib#typesafe-conductr-bundle-library) comes in multiple flavors depending on whether you need it for Lagom, Play or Akka development. It also supports different framework versions. The following flavors are supported by the library:

* Akka 2.4 for Java and Scala - [akka24-conductr-bundle-lib](#akka23|24-conductr-bundle-lib)
* Akka 2.3 for Java and Scala - [akka23-conductr-bundle-lib](#akka23|24-conductr-bundle-lib)
* Play 2.5 for Scala (including Akka 2.4) - [play25-conductr-bundle-lib](#play25-conductr-bundle-lib)
* Play 2.4 for Java and Scala (including Akka 2.3) - [play24-conductr-bundle-lib](#play23|24-conductr-bundle-lib)
* Play 2.3 for Java and Scala (including Akka 2.3) - [play23-conductr-bundle-lib](#play23|24-conductr-bundle-lib)


## akka[23|24]-conductr-bundle-lib

Please select the Akka 2.3 or 2.4 variant depending on whether you are using Akka 2.3 or Akka 2.4 respectively.

This library provides a reactive API using [Akka Http](http://akka.io/docs/) and should be used when you are using Akka. The library can be used for both Java and Scala.

As with the previous discussions on `conductr-bundle-lib` there are these two services:

* `com.typesafe.conductr.bundlelib.akka.LocationService`
* `com.typesafe.conductr.bundlelib.akka.StatusService`

and there is also another:

* `com.typesafe.conductr.bundlelib.akka.Env`

The `Env` one is discussed in the "Akka Clustering" section below.

Other than the `import`s for the types, the only difference in terms of API are usage is how a `ConnectionContext` is established. A `ConnectionContext` for Akka requires an implicit `ActorSystem` or `ActorContext` at a minimum e.g.:

```scala
 implicit val cc = ConnectionContext()
```

There is also a lower level method where the `HttpExt` and `ActorMaterializer` are passed in:

```scala
implicit val cc = ConnectionContext(httpExt, actorMaterializer)
```

When in the context of an actor, a convenient `ImplicitConnectionContext` trait may be mixed in to establish the `ConnectionContext`. The next section illustrates this in its sample `MyService` actor.

### Static Service Lookup

As a reminder, some bundle components cannot proceed with their initialization unless the service can be located. We encourage you to re-factor these components so that they look up services at the time when they are required, given that services can come and go. That said, here is a non-blocking improvement on the example provided for the `scala-conductr-bundle-lib`:


```scala
class MyService(cache: CacheLike) extends Actor with ImplicitConnectionContext {

  import context.dispatcher

  override def preStart(): Unit =
    LocationService.lookup("someservice", URI("http://127.0.0.1:9000"), cache).pipeTo(self)

  override def receive: Receive =
    initial

  private def initial: Receive = {
    case Some(someService: URI) =>
      // We now have the service

      context.become(service(someService))

    case None =>
      self ! PoisonPill
  }

  private def service(someService: URI): Receive = {
    // Regular actor receive handling goes here given that we have a service URI now.
    ...
  }
}
```

This type of actor is used to handle service processing and should only receive service oriented messages once its dependent service URI is known. This is an improvement on the blocking example provided before, as it will not block. However it still has the requirement that `someservice` must be running at the point of initialization, and that it continues to run. Neither of these requirements may always be satisfied with a distributed system.

### Java

The following example illustrates how status is signalled using the Akka Java API:

```java
ConnectionContext cc = ConnectionContext.create(system);
StatusService.getInstance().signalStartedOrExitWithContext(cc);
```

Similarly here is a service lookup:

```java
ConnectionContext cc = ConnectionContext.create(system);
LocationService.getInstance().lookupWithContext("whatever", URI("tcp://localhost:1234"), cache, cc)
```

### Akka Clustering

[Akka cluster](http://doc.akka.io/docs/akka/2.4.1/scala/cluster-usage.html) based applications or services have a requirement where the first node in a cluster must form the cluster, and the subsequent nodes join with any of the ones that come before them (seed nodes). Where bundles share the same `system` property in their `bundle.conf`, and have an intersection of endpoint names, then ConductR will ensure that only one bundle is started at a time. Thus the first bundle can determine whether it is the first bundle, and subsequent bundles can determine the IP and port numbers of the bundles that have started before them.

In order for an application or service to take advantage of this guarantee provided by ConductR, the following call is required to obtain configuration that will be used when establishing your actor system:

```scala
import com.typesafe.conductr.bundlelib.akka.Env
import com.typesafe.config.ConfigFactory

val config = Env.asConfig
val systemName = sys.env.getOrElse("BUNDLE_SYSTEM", "MyApp1")
val systemVersion = sys.env.getOrElse("BUNDLE_SYSTEM_VERSION", "1")
val app1 = ActorSystem(s"$systemName-$systemVersion", config.withFallback(ConfigFactory.load()))
```

Clusters will then be formed correctly. The above call of `Env.asConfig` looks for an endpoint named `akka-remote` by default. Therefore you must declare the Akka remoting port as a bundle endpoint. The following endpoint declaration within a `build.sbt` shows how:

```scala
BundleKeys.endpoints := Map("akka-remote" -> Endpoint("tcp"))
```

In the above, no declaration of `services` is required as akka remoting is an internal, cluster-wide TCP service.

> Note that conductr-bundle-lib will search for an endpoint named `akka-remote` by default. This should not ever need to be overridden and it is convenient that it is consistent across bundles sharing the same Akka cluster.

> The Akka cluster of your application or service is distinct to your applications and services given the bundle's `system` and `systemVersion` properties. `sbt-bundle` will set these properties to the name of your bundle and its `compatibilityVersion` by default. The `compatibilityVersion` is the major component of your project's version by default. All of these properties are able to be overridden using bundle keys. You can therefore associate multiple bundles into a single Akka cluster by using the same system and systemVersion. Akka remote ports are always dynamically allocated given the above endpoint declaration, and so you can even have multiple bundles with multiple systems all running alongside each other, some even sharing the same system but different versions.

#### Downing Strategies

[The Akka documentation does not recommend auto-downing](http://doc.akka.io/docs/akka/2.4.6/scala/cluster-usage.html#Automatic_vs__Manual_Downing) for use within your Akka cluster based application/service. Instead it recommends that you manually shutdown a non-responsive service or provide a more robust automated strategy, such as using the Reactive Platform based Split Brain Resolver (SBR) package.

As a subscriber to the Lightbend Reactive Platform you can use SBR as an automated downing mechanism. Note that when using SBR you must handle the `MemberRemoved` event if it is received for the current member - SBR will not shutdown your application/service automatically.

We recommend running ConductR's agents on separate machines to your ConductR core services. This will eliminate the potential for contention between the downing strategies of your application/service and that of ConductR's.

If you prefer to run the ConductR agent on the same machine as ConductR core, contending strategies may not be a problem depending on your application/service's availability requirements. Suppose that your application/service chooses "keep oldest" as a strategy, and that the oldest member is in the minority portion of ConductR's cluster. If ConductR's SBR's "keep majority" decides to shutdown your oldest member, it will re-schedule it for execution again, but in the majority side of the cluster. In the case of "keep-oldest" resulting in reducing the number of running instances to one, your system may suffer an outage in terms of the length of time it takes ConductR to reschedule it for execution on the majority side of the cluster (seconds). This problem can be further mitigated by using an SBR timeout for "keep oldest" that is greater than ConductR's. 

A reasonable start point when considering an SBR based strategy to use in conjunction with ConductR in general is to use the following configuration:

```
akka.cluster.split-brain-resolver.active-strategy = keep-majority
akka.cluster.down-removal-margin = 30 seconds
```

> Always consult the Akka SBR documentation for details on the above parameters and to determine the best strategy for your particular application/service.

With any downing strategy, you should consider the availability requirements of your application/service. If the application/service requires high availability then we suggest running at least 3 instances of it, and always an odd number in order to facilitate downing strategy decisions that rely on quorums (such as "keep majority").

## play25-conductr-bundle-lib

> If you are using Play 2.5 then this section is for you. Otherwise jump below to the [play[23|24]-conductr-bundle-lib](#play23|24-conductr-bundle-lib) section.

[sbt-conductr](https://github.com/typesafehub/sbt-conductr) is automatically adding this library to your Play project.

This library provides a reactive API using [Play WS](https://www.playframework.com/documentation/2.5.x/ScalaWS) and should be used when you are using Play. The library depends on `akka24-conductr-bundle-lib` and can be used for both Java and Scala. As per Play's conventions, `play.api` is used for the Scala API and just `play` is used for Java.

As with `conductr-bundle-lib` there are two services:

* `com.typesafe.conductr.bundlelib.play.LocationService` (Java) or `com.typesafe.conductr.bundlelib.play.api.LocationService` (Scala)
* `com.typesafe.conductr.bundlelib.play.StatusService` (Java) or `com.typesafe.conductr.bundlelib.play.api.StatusService` (Scala)

and there is also another:

* `com.typesafe.conductr.bundlelib.play.Env` (Java) or `com.typesafe.conductr.bundlelib.play.api.Env` (Scala)

Please read the section on `conductr-bundle-lib` and then `scala-conductr-bundle-lib` for an introduction to these services. The `Env` one is discussed in the section below. The major difference between the APIs for Play 2.5 and the other variants is that components are expected to be injected. For example, to use the `LocationService` in your controller (Scala):

```scala
class MyGreatController @Inject() (locationService: LocationService, locationCache: CacheLike) extends Controller {
  ...
  locationService.lookup("known", URI(""), locationCache)
  ...
}
```

The following components are available for injection:

* CacheLike
* ConnectionContext
* LocationService
* StatusService

Note that if you are using your own application loader then you should ensure that the Akka and Play ConductR-related properties are loaded. Here's a complete implementation (for Scala):

```scala
class MyCustomApplicationLoader extends ApplicationLoader {
  def load(context: ApplicationLoader.Context): Application = {
    val conductRConfig = Configuration(AkkaEnv.asConfig) ++ Configuration(PlayEnv.asConfig)
    val newConfig = context.initialConfiguration ++ conductRConfig
    val newContext = context.copy(initialConfiguration = newConfig)
    val prodEnv = Environment.simple(mode = Mode.Prod)
    (new GuiceApplicationLoader(new GuiceApplicationBuilder(environment = prodEnv))).load(newContext)
  }
}
```

### Controlling startup signalling

The Play flavors of conductr-bundle-lib automatically signal that your application has started with the `StatusService`. In the event that you want finer control over signalling then you can provide your own lifecycle handler. Use [the conductr-bundle-lib one[(https://github.com/typesafehub/conductr-lib/blob/master/play25-conductr-bundle-lib/src/main/scala/com/typesafe/conductr/bundlelib/play/api/ConductRLifecycle.scala)] as a reference. You must also disable the existing lifecycle module and of course register your own. To disable the existing one:

```
play.modules.disabled += "com.typesafe.conductr.bundlelib.play.api.ConductRLifecycleModule"
```

`play.modules.enabled` is used correspondly to declare your custom lifecycle module.

A common use-case when using Akka clustering with your Play application is to wait until a cluster is ready. In this case your lifecycle module should then subscribe to member up and signal to the `StatusService` within the handler:

```scala
Cluster(system).registerOnMemberUp(yourhandler)
```

## play[23|24]-conductr-bundle-lib

[sbt-conductr](https://github.com/typesafehub/sbt-conductr) is automatically adding this library to your Play project.

Please select the Play 2.3 or 2.4 variant depending on whether you are using Play 2.3 or Play 2.4 respectively. Bringing in this library automatically brings in Akka 2.3.

This library provides a reactive API using [Play WS](https://www.playframework.com/documentation/2.4.x/ScalaWS) and should be used when you are using Play. The library can be used for both Java and Scala.

As with `conductr-bundle-lib` there are two services:

* `com.typesafe.conductr.bundlelib.play.LocationService`
* `com.typesafe.conductr.bundlelib.play.StatusService`

and there is also another:

* `com.typesafe.conductr.bundlelib.play.Env`

The `Env` one is discussed in the section below. Other than the `import`s for the types, the only difference in terms of API are usage is how a `ConnectionContext` is established. A `ConnectionContext` for Play requires an `ExecutionContext` at a minimum. For convenience, we provide a default ConnectionContext using the default execution context. This may be imported e.g.:

```scala
  import com.typesafe.conductr.lib.play.ConnectionContext.Implicits.*
```

There is also a lower level method where the `ExecutionContext` is passed in:

```scala
implicit val cc = ConnectionContext(executionContext)
```

### Java

The following example illustrates how status is signalled using the Play Java API:

```java
ConnectionContext cc =
    ConnectionContext.create(HttpExecution.defaultContext());

  ...

StatusService.getInstance().signalStartedOrExitWithContext(cc);
```

Similarly here is a service lookup:

```java
ConnectionContext cc =
    ConnectionContext.create(HttpExecution.defaultContext());

  ...

LocationService.getInstance().lookupWithContext("/whatever", new URI("tcp://localhost:1234"), cache, cc)
```

In order for an application or service to take advantage of setting important Akka and Play related properties, the following is required in order to associate ConductR configuration with that of Play and Akka:

#### Play 2.4

Note that if you are using your own application loader then you should ensure that the Akka and Play ConductR-related properties are loaded. Here's a complete implementation:

```scala
import com.typesafe.conductr.bundlelib.akka.{ Env => AkkaEnv }
import com.typesafe.conductr.bundlelib.play.{ Env => PlayEnv }
import play.api.inject.guice.GuiceApplicationLoader
import play.api.{ Configuration, Application, ApplicationLoader }

class MyCustomApplicationLoader extends ApplicationLoader {
  def load(context: ApplicationLoader.Context): Application = {
    val conductRConfig = Configuration(AkkaEnv.asConfig) ++ Configuration(PlayEnv.asConfig)
    val newConfig = context.initialConfiguration ++ conductRConfig
    val newContext = context.copy(initialConfiguration = newConfig)
    (new GuiceApplicationLoader).load(newContext)
  }
}
```

#### Play 2.3

```scala
import play.api._
import com.typesafe.conductr.bundlelib.akka.{ Env => AkkaEnv }
import com.typesafe.conductr.bundlelib.play.{ Env => PlayEnv }

object Global extends GlobalSettings {
  val totalConfiguration = 
    super.configuration ++ Configuration(AkkaEnv.asConfig) ++ Configuration(PlayEnv.asConfig)

  override def configuration: Configuration =
    totalConfiguration
}
```

### Controlling startup signalling

In the case of Play 2.4, the Play flavors of conductr-bundle-lib automatically signal that your application has started with the `StatusService`. In the event that you want finer control over signalling then you can provide your own lifecycle handler. Use [the conductr-bundle-lib one[(https://github.com/typesafehub/conductr-lib/blob/master/play24-conductr-bundle-lib/src/main/scala/com/typesafe/conductr/bundlelib/play/api/ConductRLifecycle.scala)] as a reference. You must also disable the existing lifecycle module and of course register your own. To disable the existing one:

```
play.modules.disabled += "com.typesafe.conductr.bundlelib.play.api.ConductRLifecycleModule"
```

`play.modules.enabled` is used correspondly to declare your custom lifecycle module.

A common use-case when using Akka clustering with your Play application is to wait until a cluster is ready. In this case your lifecycle module should then subscribe to member up and signal to the `StatusService` within the handler:

```scala
Cluster(system).registerOnMemberUp(yourhandler)
```

## lagom10-conductr-bundle-lib

> If you are using Lagom 1.0.x then this section is for you.

[sbt-conductr](https://github.com/typesafehub/sbt-conductr) is automatically adding this library to your Lagom project. You don't need set any additional setting for your Lagom services.

Note that if you are using your own application loader then you should ensure that the Akka, Play and Lagom ConductR-related properties are loaded. Here's a complete implementation (in Scala):

```scala
class MyCustomApplicationLoader extends ApplicationLoader {
  def load(context: ApplicationLoader.Context): Application = {
    val conductRConfig = Configuration(AkkaEnv.asConfig) ++ Configuration(PlayEnv.asConfig) ++ Configuration(LagomEnv.asConfig)
    val newConfig = context.initialConfiguration ++ conductRConfig
    val newContext = context.copy(initialConfiguration = newConfig)
    val prodEnv = Environment.simple(mode = Mode.Prod)
    (new GuiceApplicationLoader(new GuiceApplicationBuilder(environment = prodEnv))).load(newContext)
  }
}
```

### Controlling startup signalling

The Lagom flavors of conductr-bundle-lib automatically signal that your application has started with the `StatusService`. In the event that you want finer control over signalling then you can provide your own lifecycle handler. Use [the conductr-bundle-lib one[(https://github.com/typesafehub/conductr-lib/blob/master/play25-conductr-bundle-lib/src/main/scala/com/typesafe/conductr/bundlelib/play/api/ConductRLifecycle.scala)] as a reference. You must also disable the existing lifecycle module and of course register your own. To disable the existing one:

```
play.modules.disabled += "com.typesafe.conductr.bundlelib.play.api.ConductRLifecycleModule"
```

`play.modules.enabled` is used correspondly to declare your custom lifecycle module.

A common use-case when using Akka clustering with your Play application is to wait until a cluster is ready. In this case your lifecycle module should then subscribe to member up and signal to the `StatusService` within the handler:

```scala
Cluster(system).registerOnMemberUp(yourhandler)
```