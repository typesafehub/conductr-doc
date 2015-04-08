# Typesafe ConductR %PLAY_VERSION%

## Introduction

This document first provides an operator quickstart, with the second part explaining an developer quickstart. Feel free to read both sections, but you may read one or the other depending on your bias.

## Operator Quickstart

ConductR provides REST API which allows you as the operator to:

* query the information on loaded bundles and running services
* manage the lifecycle of bundles (load, run, stop, unload)

The API can be used by any HTTP client, but ConductR comes with CLI tool implemented in Python. To get the latest version of the CLI install it locally as a pip package.

### New CLI installation

Firstly `pip3` is required:

```bash
sudo apt-get install python3-setuptools 
sudo easy_install3 -U pip
```

...then:

```bash
pip3 install --user typesafe-conductr-cli
```

`pip3` will install the CLI in your home folder under the .local folder (at least on Ubuntu). Ensure that your path captures this:

```bash
# Local bin files
PATH=$HOME/.local/bin:$PATH
```

To verify the installation type:

```bash
conduct -h
```

...you should then see output similar to the following:

```bash
usage: conduct [-h] {version,info,services,load,run,stop,unload} ...

optional arguments:
  -h, --help            show this help message and exit

subcommands:
  valid subcommands                                                                                       depl

  {version,info,services,load,run,stop,unload}
                        help for subcommands
    version             print version
    info                print bundle information
    services            print service information
    load                load a bundle
    run                 run a bundle
    stop                stop a abundle
    unload              unload a bundle
```

### Upgrading the CLI

```bash
pip3 install --user --upgrade typesafe-conductr-cli
```

### Packaging configuration

In addition to consuming services provided by ConductR, the CLI also provides a quick way of packaging custom configuration to a bundle. We will go through most of the CLI features by deploying the Visualizer bundle to ConductR that comes together with the ConductR installation. The Visualizer can be found in the `/usr/share/conductr/samples` directory.

Visualizer is a sample Play Framework application that queries ConductR API endpoints and visualizes the ConductR cluster together with deployed and running bundles.

![scope](visualizer.png)

Some applications require additional configuration when deployed to different environments. Visualizer allows setting `POLL_INTERVAL` environment variable which controls how quickly ConductR is polled after receiving events that state has changed.

A strong feature of ConductR is that configuration may be coupled with a bundle at the time of loading. Thus a bundle can be associated with different configurations; perhaps one configuration for a test environment, and another for production. Furthermore, different versions of a bundle may be associated with the same configuration. Whatever the combination is, the bundle and its configuration that is loaded may always be distinguished from others given that unique identifiers are returned for them.

Configuration files are executed just before a bundle starts. Let's create a configuration file that exports a `POLL_INTERVAL` environment variable containing a shorter than default poll interval (which is 100ms):

```bash
cd #back to your home folder so we know where we're playing
echo "export POLL_INTERVAL=500ms" >> ./visualizer-poll-interval.sh
```

Once the configuration file is ready, we need to package it up to a bundle. We can use the `shazar` command provided by the CLI which zips configuration files and appends secure digest of the archive to the created bundle files.

```bash
shazar ./visualizer-poll-interval.sh
```

The configuration is now ready to be loaded along with a bundle. Configuration is always provided as the second param to the `conduct load` command.

### Deploying bundles

The standard bundle lifecycle in the ConductR is:

1. Load. Bundle is loaded into a ConductR cluster and replicated among cluster nodes.
2. Run. Bundle is started by running one or more instances of all defined bundle components.
3. Stop. Bundle is stopped by stopping all instances of all defined bundle components.
4. Unload. Bundle is unloaded from a ConductR cluster and all bundle replicas are removed from cluster nodes.

We use `conduct` command provided by the CLI to load a Visualizer bundle together with configuration to the ConductR and then run it. Every `conduct` command that is communicating with a ConductR needs to be given `--ip` and `--port` parameters (these can be omitted if ConductR is running on the same machine and on the default port). Alternatively CLI can use `CONDUCTR_IP` and `CONDUCTR_PORT` environment variables for corresponding parameters.

This data is used by the ConductR when making bundle scheduling decisions. Load the Visualizer bundle by executing:

```bash
conduct load --ip 172.17.0.1 \
             /usr/share/conductr/samples/visualizer-...zip \
             ./visualizer-poll-interval.sh-...zip
```

Use `conduct info` command to list all loaded bundles. You should see Visualizer replicated but not running (note that the example below shows 3 replications - you'll only get that if you have 3 or more nodes as the bundle cannot replicate beyond the cluster size).

```bash
conduct info --ip 172.17.0.1
```
...will yield something like:

```bash
ID               NAME              #REP  #STR  #RUN
23391d4-3cc322b  visualizer-0.1.0  3     0     0
```

Run Visualizer by executing:

```bash
conduct run --ip 172.17.0.1 visualizer
```

Whenever you need to refer to a bundle you can use a prefix of or a full bundle id/name. Run `conduct info` once again, to see that the bundle has been successfully started.

``` bash
ID               NAME              #REP  #STR  #RUN
23391d4-3cc322b  visualizer-0.1.0  3     0     1
```

### Accessing services provided by bundles

Access to services in ConductR is proxied for high availability and load balancing. To list all the currently running services in the ConductR execute the `conduct services` command.

```bash
conduct services --ip 172.17.0.1
```

You should see that there is one service called `9000/visualizer` provided by the Visualizer bundle.

```bash
SERVICE       BUNDLE ID       BUNDLE NAME       STATUS
http://:9999  23391d4-3cc322b visualizer-0.1.0  Running
```

To access Visualizer point your browser to any ConductR node and add the name of the service to the URL, e.g. `http://172.17.0.1:9999`. Alternatively, if you can only access ConductR nodes using SSH, create a SSH tunnel that tunnels local port from your machine to the Visualizer service `ssh -L 8080:172.17.0.1:9999 172.17.0.1` (don't forget to substitute the `172.17.0.1`) and then access Visualizer by pointing your browser to `http://localhost:9999`.

Visualizer shows ConductR nodes as small blue circles with IP addresses next to them. Green circles denote bundles, and spinning circle means that a bundle is running. You should see one instance of bundle running. Try starting Visualizer on more nodes by executing:

```bash
conduct run --ip 172.17.0.1 --scale 2 visualizer
```

You should see another green circle start spinning, which means that another instance of Visualizer was started. Play around with more `conduct` commands and see how it affects ConductR cluster visualization.

Our aim is to make using Typesafe ConductR by operators akin to using Play by developers; a joyful and productive experience! ConductR starts to shine when used in the context of managing more than 2 nodes; a common scenario for reactive applications. Go and spin those nodes up!

## Developer Quickstart

ConductR provides two services for you as the developer:

* the ability to signal when your application or service is ready to start processing; and
* the ability to resolve a URI describing where another service is located.

In addition ConductR provides a convenient packaging mechanism known as a bundle. Bundles compose components where a component is the application or service that you have developed. There is typically a one-to-one correlation between a bundle and a component with the ability to have multiple components for some interesting use-cases such as co-locating a persistence store with a microservice.

We provide an [sbt](http://www.scala-sbt.org/) plugin that conveniently bundles your application or service, and permits you to manage its lifecycle in ConductR (we also provide non-sbt tooling for this, as explained under _Operator Quickstart_ above).

### Signalling Ready

Your application or service should tell ConductR when it has completed its initialization and is ready for work. We have a fall-back strategy for where this is not possible or practical (more on that later), but for now, let's assume that you can modify your source code.

For a [Play](https://www.playframework.com/) application or service, signalling successful startup may be when its http endpoint is online having declared a connection to a database. For an [Akka](http://akka.io/) based application then it may be when your actor system has been initialized and you have connected to some other service.

ConductR's only requirement of you is to use a library and call a single function as `StatusService.signalStarted()`. Note that if you call this function outside of running within ConductR then it does nothing so you can continue to develop and debug as you have always done.

The library comes in multiple flavours: pure Scala/JDK, Akka and Play. To add Scala the library to your dependencies using `build.sbt`:

```scala
resolvers += "typesafe-releases" at "http://repo.typesafe.com/typesafe/maven-releases"

libraryDependencies += "com.typesafe.conductr" %% "scala-conductr-bundle-lib" % "0.6.1"
```

`scala-conductr-bundle-lib` has no dependencies other than the JDK and as such, a blocking implementation is used for its http calls (the JDK offers no non-blocking APIs for this). However when using Akka or Play then substitute `"akka-conductr-bundle-lib"` or `"play-conductr-bundle-lib"` respectively. Doing so will ensure that the types used are consistent with Akka and Play, and that non-blocking implementations using akka-http and Play.WS are used.

When you are reasonably sure that your code is ready to start processing (it doesn't have to be exactly at that time - ConductR tolerates services being unavailable), call the `signalStartedOrExit` function. Here's a complete example using Play and Scala having included a dependency for `"play-conductr-bundle-lib"` instead of `"scala-conductr-bundle-lib"`:

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

Where you have non-HTTP services to locate, you can use the ServiceLocator API directly. The following example attempts to locate a fictitious JMS broker service:

```scala
// This will require an implicit ConnectionContext to
// hold a Scala ExecutionContext. There are different
// ConnectionContexts depending on which flavor of the
// library is being used. For the Scala flavor, a Scala
// ExecutionContext is composed. The ExecutionContext
// is needed as "service" is returned as a Future.
// For convenience, we provide a global ConnectionContext
// that may be imported.
import com.typesafe.conductr.bundlelib.scala.ConnectionContext.Implicits.global

// We also provide a cache designed specifically to
// hold location values. Caches are optional to the
// lookup, but they are encouraged.
val locationCache = LocationCache()

// ...and the lookup itself...
val jmsBroker = LocationService.lookup("/jms", locationCache)
```

`jmsBroker` is typed `Future[Option[String]]` meaning that an optional response with the resolved URI will be returned at some time in the future. Supposing that this lookup is made during the initialisation of your program, the service you're looking for may not exist. However calling the same function later may yield the service. This is because services can come and go.

#### Non HTTP Static service lookup

Some bundle components cannot proceed with their initialisation unless the service can be located. We encourage you to re-factor these components so that they look up services at the time when they are required, given that services can come and go. However if you are somehow stuck with this style of code then you may consider the following blocking code as a temporary measure:

```scala
val resultUri = Await.result(
  LocationService.lookup("/someservice"),
  sometimeout)
val serviceUri = resultUri.getOrElse {
  if (Env.isRunByConductR) System.exit(70)
  "http://127.0.0.1:9000"
}
```

In the above, the program will exit if a service cannot be located at the time the program initializes; unless the program has not been started by ConductR in which case an alternate URI is provided. Instead of blocking you may also consider using an Akka actor:

```scala
// bundlelib types are imported from com.typesafe.conductr.bundlelib.akka

// ImplicitConnectionContext is a convenience that we provide for obtaining
// a connection context within an actor.

class MyService extends Actor with ImplicitConnectionContext {

  import context.dispatcher

  override def preStart(): Unit =
    LocationService.lookup("/someservice").pipeTo(self)

  override def receive: Receive =
    initial

  private def initial: Receive = {
    case Some(someService: String) =>
      // We now have the service

      context.become(service(someService))

    case None =>
      self ! (if (Env.isRunByConductR) PoisonPill else Some("http://127.0.0.1:9000"))
  }

  private def service(someService: String): Receive = {
    // Regular actor receive handling goes here given that we have a service URI now.
    ...
  }
}
```

This type of actor is used to handle service processing and should only receive service oriented messages once its dependent service URI is known. This is an improvement on the blocking example provided before, as it will not block. However it still has the requirement that `someservice` must be running at the point of initialisation, and that it continues to run. Neither of these requirements may always be satisfied with a distributed system.

### Bundling your Application or Service

A bundle is the fundamental denomination of currency for ConductR. Bundles are a composition of components. There is typically just the one component which is your application or service. You can think of bundles in J2EE parlance as an EAR given that it constitutes an application or service's compile-time dependencies, but that is probably where the similarity ends.

We offer two methods of building a bundle:

* using an [sbt](http://www.scala-sbt.org/) plugin with your build; and/or
* using `shazar` (we invented that name!)

#### sbt-bundle

The [sbt-native-packager](https://github.com/sbt/sbt-native-packager#sbt-native-packager) has been extended with a plugin named [sbt-bundle](https://github.com/sbt/sbt-bundle#typesafe-conductr-bundle-plugin). You can declare your native packager to build a Universal or Docker package and have that become what is bundled.

Note that the description here is just to provide a feel of how `sbt-bundle` is used. Please refer to [its documentation](https://github.com/sbt/sbt-bundle#typesafe-conductr-bundle-plugin) as there are some small considerations when dealing with the pre 1.0 `sbt-native-packager` e.g. the one used with Play 2.3.

Firstly add the sbt plugin, typically to your project's `project/plugins.sbt` file (check [here](https://github.com/sbt/sbt-bundle#usage) for the latest release of sbt-bundle):

```scala
addSbtPlugin("com.typesafe.sbt" % "sbt-bundle" % "0.19.1")
```

You will then need to declare what are known as "scheduling parameters" for ConductR. These parameters effectively describe what resources are used by your application or service and are used to determine which machine they will run on. Here's a minimum set of parameter specifying that 1 cpu, 64MiB memory and 5MB of disk space is required when your application or service runs:

```scala
import ByteConversions._

BundleKeys.nrOfCpus := 1.0
BundleKeys.memory := 64.MiB
BundleKeys.diskSpace := 5.MB
```

(You'll note that the international standards for supporting [binary prefixes](http://en.wikipedia.org/wiki/Binary_prefix), as in `MiB`, is supported).

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

#### When you cannot change your source

On the subject of bundling, it is sometimes not possible or practical to change source code in order to signal successful startup or have it use the environment variables that ConductR provides.

We provide a [CLI](https://github.com/typesafehub/typesafe-conductr-cli#command-line-interface-cli-for-typesafe-conductr) command named [`shazar`](https://github.com/typesafehub/typesafe-conductr-cli#shazar) for bundling the contents of any folder. You can therefore hand-craft a `bundle.conf` and its component folders and use `shazar` to bundle it. See [our documentation](TODO) for a full description on the layout of a bundle including its `bundle.conf` file.

As a quick example, suppose that you wish to bundle [ActiveMQ](http://activemq.apache.org/) as a Docker component with a `Dockerfile`. You can do something like this (btw: we appreciate that you cannot change the world in one go and don't always have the luxury of using Akka for messaging!):

```
components = {
  "jms" = {
    description      = "A Docker container for Active/MQ"
    file-system-type = "docker"
    start-command    = []
    endpoints        = {
      "jms" = {
        protocol     = "tcp"
        bind-port    = 61616
        services     = ["tcp://:61616"]
      }
    }
  }
  "jms-status" = {
    description      = "Status check for the jms component"
    file-system-type = "universal"
    start-command    = ["check", "docker+$JMS_HOST"]
    endpoints        = {}
  }
}
```

The declaration of interest is the `jms-status` component. ConductR provides a `check` command that bundle components may use to poll a tcp endpoint until it becomes available. `docker` instructs `check` to wait for all Docker components of this bundle to start and `JMS_HOST` is a [standard environment variable](https://github.com/sbt/sbt-bundle#standard-environment-variables) that will be provided at runtime given the `"jms"` endpoint declaration; it is a URI describing the JMS endpoint. You can similarly poll http endpoints and wait for them to become available. [Consult `check`'s documentation](TODO) for more information.

#### To Docker or Not

[Docker](https://www.docker.com/) is a technology that provides containers for your application or service. Most Typesafe Reactive Platform (Typesafe RP) based programs should not require Docker as the host OS's Java Runtime Environment 8 (JRE 8) environment should be sufficient. Bundles generally contain all that is required for a Typesafe RP program to run, with exception to the Host OS and the host JRE. Typesafe RP bundles will start faster and incur less system resources when used without Docker.

Docker becomes relevant when there are specific runtime dependencies that are different to ConductR's host OS environment. In particular if a binary program that does not use the JVM is required to be launched from a bundle then it becomes more likely to benefit from using a Docker container.

### Loading and Running your Application or Service

The fun part!

Once you've created a bundle you can deploy it using ConductR's RESTful API, even using [curl](http://curl.haxx.se/) if you want. However we've made it a little easier than that. You can of course use ConductR's CLI as discussed in the _Operator Quickstart_ above. Alternatively given that we've been discussing bundling your application or service mostly from an sbt perspective, you can use another plugin named [`sbt-typesafe-conductr`](https://github.com/sbt/sbt-typesafe-conductr#sbt-typesafe-conductr).

The following description is intended to provide a taste of what `sbt-typesafe-conductr` can do for you. Please refer to [its documentation](https://github.com/sbt/sbt-typesafe-conductr/blob/master/README.md) as there are some small considerations when dealing with the pre 1.0 `sbt-native-packager` e.g. the one used with Play 2.3.

To use `sbt-typesafe-conductr` first add the plugin your build (typically your `project/plugins.sbt` file); be sure to check at [the plugin's website](https://github.com/sbt/sbt-typesafe-conductr#sbt-typesafe-conductr) for the latest version to use:

```scala
addSbtPlugin("com.typesafe.conductr" % "sbt-typesafe-conductr" % "0.27.0")
```

Note that if you add this plugin as above, you do not need to have an explicit declaration for `sbt-bundle`. `sbt-bundle` will be automatically added as a dependency of `sbt-typesafe-conductr`.

The `sbt-typesafe-conductr` plugin must then be enabled for your project. Supposing that your project has one module that will use the plugin which is the root of the sbt project (the most typical situation for a single `build.sbt`):

```scala
lazy val root = project
  .in(file("."))
  .enablePlugins(SbtTypesafeConductR, <your other plugins go here>)
```

With your declarations out of the way, you can produce a bundle by typing:

```bash
bundle:dist
```

A bundle will be produced from the native packager settings of this project. A bundle effectively wraps a native
packager distribution and includes some component configuration. To load the bundle first declare the location of ConductR (supposing that ConductR is running on `172.14.0.1`:

```bash
conductr:controlServer 172.14.0.1
```

...and then load:

```bash
conductr:load <HIT THE TAB KEY AND THEN RETURN>
```

Using the tab completion feature of sbt will produce a URI representing the location of the last distribution
produced by the native packager.

Hitting return will cause the bundle to be uploaded. On successfully uploading the bundle the plugin will report
the `BundleId` to use for subsequent commands on that bundle.

You can also run, stop and unload bundles by using this plugin. This may be useful to support your development lifecycle without having to jump into the operator's CLI.

That is all that is required in essence, but as stated, you should read [`sbt-typesafe-conductr`'s documentation](https://github.com/sbt/sbt-typesafe-conductr/blob/master/README.md) as there are a few additional requirements, particularly if you are managing a Play 2.3 application.

Now go and develop reactive applications or services for ConductR!
