# Creating application bundles

Once the application is ready for deployment, developers can view ConductR bundles as just yet another deployment target. We offer a few methods of building a bundle:

1. Using an [sbt](http://www.scala-sbt.org/) plugin with your build
2. Using `bndl` to create bundles from Docker images
2. Using `shazar` (we invented that name!)

If you can make changes to the application or service then `sbt-conductr` is what you will typically use to produce a bundle. In fact you can even use sbt-conductr to produce bundles for other applications or services. However you may find yourself crafting a bundle from scratch and for the latter scenario. See the "legacy & third party bundles" section of the [bundles](BundleConfiguration#Legacy-&-third-party-bundles) document for more information on that, and for a deep dive on bundles in general. For now, let's look at bundling a project that you have control of.

ConductR supports several different types of bundle components.
  * OCI bundle components (introduced in 2.1) package an entire container in the bundle. They can be produced from Docker images. They run inside a container using `runc`.
  * Universal bundle components run directly on the host.
  * Docker bundle components build a provided `Dockerfile` and run the resulting image
  
In addition, Docker images can be run from "universal" bundles. The following sections cover each of these scenarios:

* [Producing an OCI bundle with Docker](#producing-an-oci-bundle-with-docker)
* [Producing a universal bundle without Docker](#producing-a-universal-bundle-without-docker)
* [Producing a universal bundle that runs a Docker image](#producing-a-universal-bundle-that-runs-a-docker-image)
* [Producing a docker bundle that uses a Dockerfile](#producing-a-docker-bundle-that-uses-a-dockerfile)

## Producing an OCI bundle with Docker
ConductR is capable of running images that have been exported from Docker. To do this, you use the `bndl` tool to convert a docker image into a ConductR bundle. Behind the scenes, we're using the standard OCI Image format so you can be sure your images are free of vendor lock-in. Additionally, this approach doesn't require Docker to be installed in production; ConductR provides everything necessary to run these images.

The following command will fetch the image `dockercloud/hello-world` with tag `stdout` from DockerHub and load it directly into ConductR. If the tag is omitted, `latest` is used.

```bash
conduct load dockercloud/hello-world:stdout
```

Private registries are also supported using traditional notation.

```bash
conduct load some-registry.bintray.io/some-project/some-image:some-tag
```



If you'd rather save the resulting bundle to a file, you can do this using `docker save` and `bndl -o <filename>`.
```bash
docker pull dockercloud/hello-world:stdout
docker save dockercloud/hello-world:stdout | bndl -o dockercloud-hello-world.zip
```


We attempt to extract as much information from the your docker image as possible. Any `EXPOSE` directives in your `Dockerfile` will automatically create corresponding service endpoints in `bundle.conf`. Support for persistent mounting of `VOLUME` directives is planned.

Docker images can vary in size significantly and this can affect development/deployment time and consume additional system resources. Because of this, we recommend that you build your images ontop of a light-weight base image. [openjdk/8-jre-alpine](https://hub.docker.com/_/openjdk/) is a great choice as it's only ~80MB in size (~50MB when compressed). 

## Producing a universal bundle without Docker

With [sbt-conductr](https://github.com/typesafehub/sbt-conductr) you can produce a ConductR bundle for your application. sbt-conductr extends the [sbt-native-packager](https://github.com/sbt/sbt-native-packager#sbt-native-packager). Just as developers might use `sbt dist` or `sbt debian:packageBin` to produce production binaries of an application, the sbt `bundle:dist` task from sbt-conductr is used to produce ConductR application bundles.

Note that the description here is just to provide a feel of how `sbt-conductr` is used. Please refer to [its documentation](https://github.com/typesafehub/sbt-conductr#sbt-conductr) as there are some small considerations when dealing with the pre 1.0 `sbt-native-packager` e.g. the one used with Play 2.3.

`sbt-conductr` is triggered for each project that enables a native packager plugin in the `build.sbt`, e.g.:

```scala
lazy val root = (project in file(".")).enablePlugins(JavaAppPackaging)
```

Note that Lagom or Play users can enable one of the following plugins instead:

| Project            | Description                                                                         |
|--------------------|-------------------------------------------------------------------------------------|
| Lagom Java         | `lazy val myService = (project in file(".")).enablePlugins(LagomJava)`              |
| Play Java in Lagom | `lazy val myService = (project in file(".")).enablePlugins(LagomPlay)`              |
| Play Scala 2.4+    | `lazy val root = (project in file(".")).enablePlugins(PlayScala)`                   |
| Play Scala 2.3     | `lazy val root = (project in file(".")).enablePlugins(JavaAppPackaging, PlayScala)` |
| Play Java 2.4+     | `lazy val root = (project in file(".")).enablePlugins(PlayJava)`                    |
| Play Java 2.3      | `lazy val root = (project in file(".")).enablePlugins(JavaAppPackaging, PlayJava)`  |

You will then need to declare what are known as "scheduling parameters" for ConductR. These parameters effectively describe what resources are used by your application or service and are used to determine which machine they will run on.

The Play and Lagom bundle plugins provide [default scheduling parameters](https://github.com/typesafehub/sbt-conductr/blob/master/README.md#scheduling-parameters), i.e. it is not mandatory to declare scheduling parameters for these kind of applications. However, we recommend to define custom settings for each of your application.  

In the following example we specify that 0.1 CPU, 64 MiB JVM heap memory, 384 MiB resident memory and 5 MB of disk space is required when your application or service runs:

```scala
import ByteConversions._

javaOptions in Universal := Seq(
  "-J-Xmx64m",
  "-J-Xms64m"
)

BundleKeys.nrOfCpus := 0.1
BundleKeys.memory := 384.MiB
BundleKeys.diskSpace := 5.MB
```

You'll note that the international standards for supporting [binary prefixes](http://en.wikipedia.org/wiki/Binary_prefix), as in `MiB`, is supported.

Now create a bundle:

```scala
bundle:dist
```

Take a look in your `target/bundle` folder - you'll see your new bundle with a hash string, something like this:

```
visualizer-1.0.0-b07601b2bd015c94de0514ad698760d0396cb6f95881396d37981b271c0e7142.zip
```

i.e. the name of your project (`visualizer`), its version `1.1` and the hash value representing its contents (`b07601b2bd015c94de0514ad698760d0396cb6f95881396d37981b271c0e7142`).

Your bundle is now ready for loading into ConductR. What happened there is a few sensible defaults were chosen for you, a `bundle.conf` was generated, and a zip was built containing the output of a Universal build. The plugin then generated a [secure hash](http://en.wikipedia.org/wiki/Secure_Hash_Algorithm) of the zip contents and encoded it in the filename. The secure hash provides a version of your bundle that ConductR is able to verify and thus assure you of its integrity. You can derive a reasonable level of confidence that version xyz of your application or service is always going to be xyz; something that makes you happy if you ever need to reliably rollback to an older version of your software!

Your application or service must be told what interface and port it should bind to when it runs. The Lagom and Play bundle plugins do that automatically.

ConductR provides a specific ip address for the application to bind. Binding the provided private address enables the application to limit its network exposure to only the required interface. Furthermore, the port that ConductR provides for binding to is guaranteed to not clash with other bundle components on the same machine, and so it is important to use.

Here is a Akka example one where [Typesafe config](https://github.com/typesafehub/config#overview) is used to determine the ip and port that your service requires:

```
customer-service {
  ip = "127.0.0.1"
  ip = ${?CUSTOMER_SERVICE_BIND_IP}
  port = 9000
  port = ${?CUSTOMER_SERVICE_BIND_PORT}
}

```

`CUSTOMER_SERVICE_BIND_IP` and `CUSTOMER_SERVICE_BIND_PORT` are environment variables provided by ConductR. ConductR will generate a few useful environment variables for your bundle component. Check out the [bundle settings](https://github.com/typesafehub/sbt-conductr#bundle-settings) of sbt-conductr for a comprehensive statement of its capabilities and settings, particularly around configuring your service's endpoints.

Typesafe config provides the ability to substitute environment variables if they exist. With the above, an ip of 127.0.0.1 and a port of 9000 will be used if ConductR has not been used to start your application. `CUSTOMER_SERVICE` was declared as the name of the endpoint using sbt configuration e.g.:


```scala
BundleKeys.endpoints := Map("customer-service" -> Endpoint("http", 0, "customers", RequestAcl(Http("^/customers".r))))
```

* The endpoint name is `customer-service`.
* The endpoint exposes `http` based service.
* The port is declared as `0`, meaning it will be assigned with a port number picked by ConductR.
* The service name is `customers` which can be used to resolve the endpoint by service name. Refer to [[Resolving services|ResolvingServices]] for further details.
* The endpoint will accept any http request that starts with the path `/customers`. Refer to [[ACL configuration
|RequestAclConfiguration]] for further details.

With the above Typesafe config you can then resolve the ip address and port within your application. The following example additionally uses Akka HTTP to start a HTTP server:

```scala
val ip = config.getString("customer-service.ip")
val port = config.getInt("customer-service.port")
Http(system).bind(ip, port) // ... and so forth
```

### More on memory

The `javaOptions` values declare the maximum and minimum heap size for your application respectively. Profiling your application under load will help you determine an optimal heap size. We recommend declaring the `BundleKeys.memory` value to be approximately twice that of the heap size. `BundleKeys.memory` represents the resident memory size of your application, which includes the heap, thread stacks, code caches, the code itself and so forth. On Unix, use the top command and observe the resident memory column (RES) with your application under load.

`BundleKeys.memory` is used for locating machines with enough resources to run your application, and so it is particularly important to size it before you go to production.

## Producing a universal bundle that runs a Docker image

If you'd prefer, or indeed require to run a Docker image directly then you can set up one via sbt. Here is a section of `conductr-kibana`'s build file:

```scala
import ByteConversions._

name := "conductr-kibana"

mappings in Universal := Seq.empty

BundleKeys.conductrTargetVersion := ConductrVersion.V2
BundleKeys.compatibilityVersion := "1"
BundleKeys.nrOfCpus := 0.1
BundleKeys.memory := 128.MiB
BundleKeys.minMemoryCheckValue := 128.MiB
BundleKeys.diskSpace := 25.MB
BundleKeys.roles := Set("kibana")
BundleKeys.endpoints := Map(
  "kibana" -> Endpoint("http", 0, "kibana", RequestAcl(Http("^/".r)))
)
BundleKeys.startCommand := Seq(
  "docker",
  "run",
  "--name", "$BUNDLE_ID-$BUNDLE_NAME",
  "--env", "ELASTICSEARCH_URL=$(curl -s -o /dev/null -w %{redirect_url} $SERVICE_LOCATOR/elastic-search)",
  "--publish", "$BUNDLE_HOST_IP:$KIBANA_HOST_PORT:5601",
  "--rm",
  "kibana:4.1"
)
BundleKeys.checks := Seq(uri("docker+http://$BUNDLE_HOST_IP:$KIBANA_HOST_PORT?retry-count=10&retry-delay=3"))
```

The `startCommand` and `checks` settings are particulary important when running existing Docker images. The [environment variables](BundleEnvironmentVariables) available to ConductR bundles are also very important. As a convention, we name the Docker container in accordance with its unique identifier and add the bundle's name for human consumption. We also remove a bundle's mappings given that sbt-native-packager populates this setting by default (and these files have no use here).

The kibana image (we're using the [official one](https://hub.docker.com/_/kibana/)) requires that an `ELASTICSEARCH_URL` environment variable be used to declare the location of Elasticsearch. Given that Kibana has no service discovery mechanism then we attempt to locate Elasticsearch prior to starting Kibana. We use ConductR's [service locator HTTP redirect API](BundleAPI#The-Location-Lookup-Service) to locate Elasticsearch. Note that if there is no Elasticsearch service then the bundle will fail, and ConductR will retry to start it indefinitely. Therefore when Elasticsearch does become available then the Kibana bundle will succeed (of course, if Elasticsearch goes away subsequently, and appears at a new URL then Kibana will have a problem - arguably the best way to fix that properly is to introduce service discovery into Kibana itself, but you can stop/start Kibana's bundle as a work-around).

So that Kibana is accessible from the outside world, we `publish` the bundle's host IP/Kibana host port at the proxy, again using the information provided by bundle environment variables. This time, the environment variables are derived from the ACL settings. ACLs are used to declare how to make a service available at a proxy.

Finally, as part of the start command, we ensure that the container is removed once it terminates. This is important otherwise containers will accumulate within Docker and consume disk space.

The `checks` command is used to signal ConductR that Kibana has started successfully. Once the bind IP/port has become available then the start is signalled.

## Producing a docker bundle that uses a Dockerfile

When wanting to create Docker bundles with Dockerfiles you leverage the sbt-native-packager's ability to generate a Dockerfile and then let sbt-conductr know about this being the desired target. An example [build.sbt configuration for postgres-bdr](https://github.com/huntc/postgres-bdr/blob/master/build.sbt) is shown below:

```scala
// Docker specifics for the native packager

dockerCommands := Seq(
  Cmd("FROM", "agios/postgres-bdr"),
  Cmd("ADD", "/opt/docker/bin/init-database.sh /docker-entrypoint-initdb.d/")
)

// The following setting instructs sbt-bundle to use Docker packaging
// i.e. Dockerfile based installations

BundleKeys.bundleType := Docker

// Regular sbt-bundle configuration

BundleKeys.nrOfCpus := 4.0
BundleKeys.memory := 2.GB
BundleKeys.diskSpace := 10.GB
BundleKeys.roles := Set("postgres94")
BundleKeys.endpoints := Map(
  "postgres" -> Endpoint("tcp", 5432, "postgres", RequestAcls(Tcp(5432)))
)

// Additional args for the docker run can be supplied here (see the sbt-bundle
// README) - otherwise the startCommand is empty

BundleKeys.startCommand := Seq.empty

// The following check creates a bundle component that waits for the
// Docker build to complete and then test the postgres-bdr port. If
// the port is open then ConductR is signalled that the bundle is ready.

BundleKeys.checks := Seq(uri("docker+$POSTGRES_HOST"))
```

> When deploying Docker bundles to the sandbox they won't work. The sandbox is using Docker itself and you cannot run Docker within Docker. For development purposes setup a single VM and configure it as per the regular Linux installation along with Docker. You'll then be able to test your Docker bundles locally.

## Publishing bundles

Bundles are able to be published to repositories such that they can be pulled down and used by others at a later stage. At the time of writing [Bintray](https://bintray.com/) is a supported provider of repositories. Bintray supports the publishing of bundles to both private and public repositories. Once published, the `conduct load` command can be used to pull down the latest or a specific version of your bundle with ease. For example, to pull down Cassandra from our organization's standard repository and load it into ConductR:

```
conduct load cassandra
```

The `cassandra` bundle will be resolved against supported repositories, the latest one determined, and it is then pulled down and loaded into ConductR. See [Deploying Bundles](DeployingBundlesOps) for more information on the `load` command's syntax when needing to specifying repositories other than our standard one. It generally boils down to something similar to `conduct load mygreatorg/mygreatbundle`.

The [sbt-bintray-bundle](https://github.com/sbt/sbt-bintray-bundle) is used to provide Bintray bundle publishing capabilities for your project. This plugin uses the [bintray sbt plugin](https://github.com/softprops/bintray-sbt#bintray-sbt) where most of its settings are supported.

To add the Bintray bundle plugin to your `plugins.sbt`, or equivalent (check the [plugin's site](https://github.com/sbt/sbt-bintray-bundle) for the latest version):

```scala
addSbtPlugin("com.typesafe.sbt" % "sbt-bintray-bundle" % "1.2.0")
```

Once the plugin is added you need to declare some settings for your build. Here are some taken from [a sample Lagom based application](https://github.com/lagom/activator-lagom-java-chirper):

```scala
licenses in ThisBuild := Seq("Apache-2.0" -> url("http://www.apache.org/licenses/LICENSE-2.0"))
bintrayVcsUrl in Bundle in ThisBuild := Some("https://github.com/lagom/activator-lagom-java-chirper")
bintrayOrganization in Bundle in ThisBuild := Some("typesafe")
bintrayReleaseOnPublish in Bundle in ThisBuild := true
```

Each publication to Bintray requires the declaration of a license file and a URL back to the source code repository. The `bintrayOrganization` will reflect your Bintray organization. `bintrayReleaseOnPublish` declares that a bundle will become available for downloading immediately, rather than reside in a "staging" state where it will then require publishing at Bintray. The bundle repository used by default is named "bundle". If you want to use another repository then you must use the `bintrayRepository` setting to declare it.

Other bundle repositories will be supported in the future and ConductR's forthcoming support for Continuous Delivery will leverage webhook events from Bintray in order to automatically deploy published bundles.
