# Bundle configuration

ConductR has a bundle format in order for components to be described. In general there is a one-to-one correlation between a bundle and a component.

Bundles provide ConductR with some basic knowledge about components in a *bundle descriptor*; in particular, what is required in order to load and run a component. The following is an example of a `bundle.conf` descriptor:
([Typesafe configuration](https://github.com/typesafehub/config) is used):

```
version               = "1"
name                  = "simple-test"
compatibility-version = "1"
tags                  = ["1.0.0-beta.1"]
system                = "simple-test"
system-version        = "1"
nr-of-cpus            = 0.1
memory                = 134217728
disk-space            = 10485760
roles                 = ["web"]
annotations           = {
  "com.mycomany.region" = "us-east"
}
components = {
  "angular-seed-play" = {
    description      = "angular-seed-play"
    file-system-type = "universal"
    start-command    = ["angular-seed-play/bin/angular-seed-play", "-Xms=67108864", "-Xmx=67108864"]
    endpoints        = {
      "angular-seed-play" = {
        protocol      = "http"
        bind-port     = 0
        service-name  = "angular-seed-play"
        acls          = [
          {
            http = {
              requests = [
                {
                  path-beg = "/"
                }
              ]
            }
          }
        ]
      }
    }
  }
}
```

The following table describes each property:

Name                 | Description
---------------------|-------------
acls     			 | Discussed [below](#Endpoints).
annotations          | A [HOCON](https://github.com/typesafehub/config/blob/master/HOCON.md) value describing any additional metadata that ConductR itself is not concerned with. Annotation keys should be namespaced as per the [OCI image specification](https://github.com/opencontainers/image-spec/blob/master/annotations.md).
bind-port			 | Discussed [below](#Endpoints).
compatibility-version| A versioning scheme that will be associated with a bundle that describes the level of compatibility with the bundle that went before it. ConductR can use this property to reason about the compatibility of one bundle to another given the same bundle name. By default we take the major version component of a version as defined by <http://semver.org/>. However you can make this mean anything that you need it to mean in relation to the bundle produced prior to it. We take the notion of a compatibility version from <http://ometer.com/parallel.html>.
components			 | Each bundle has at least one component, and generally just one. A bundle component contains all that is required to run itself in terms of its directory on disk when the bundle is expanded. This section describes the meta data required for each bundle component.
description			 | A human readable description of the bundle component.
disk-space           | The amount of disk space required to host an expanded bundle and configuration.
endpoints            | Discussed [below](#Endpoints).
file-system-type	 | Discussed [below](#File-System-Types).
memory               | The amount of resident memory required to run the bundle. Use the UNIX `top` command to determine this value by observing the RES and rounding up to the nearest 10MiB.
name				 | The human readable name of the bundle. This name appears often in operational output such as the CLI.
nr-of-cpus           | The minimum number of cpus required to run the bundle (can be fractions thereby expressing a portion of CPU). This value is considered when starting a bundle on a node. If the specified CPUs exceeds the available CPUs on a node, then this node is not considered for scaling the bundle. Once running, the application is not restricted to the given value and tries to use all available CPUs on the node. Required.
protocol			 | Discussed [below](#Endpoints).
roles                | The types of node in the cluster that this bundle can be deployed to. Defaults to "web".
service-name	     | Discussed [below](#Endpoints).
start-command        | Command line args required to start the component. Paths are expressed relative to the component's bin folder. The default is to use the bash script in the bin folder. Arguments can be passed to a Docker container run command via a special `dockerArgs` command should additional args be required: `start-command = ["dockerArgs","-v","/var/lib/postgresql/data:/var/lib/postgresql/data"]`. See also [Java memory](CreatingBundles#More-on-memory)
system               | The name of a cluster system that the bundle belongs to. Cluster systems are strings that may be used by a number of bundles in order to associate them. ConductR provides a guarantee that bundles belonging to the same cluster system are started with the first one in isolation to the starting of the rest. This behavior can be leverage to form clusters where the first node must form the cluster and other nodes may then join it.
system-version       | A version to associate with a system. This setting defaults to the value of compatibilityVersion.
tags                 | A list of tags that may be used to provide more meaning to a bundle's name. Just as with a name, these strings are intended for human consumption and ConductR makes no assumptions about their value - see `compatibility-version` for semantically relevant versioning. Tags commonly represent version numbers. Tags may be used when selecting a bundle by bundle name, and may be specified by appending a `:` to the name followed by the tag itself e.g. "mybundle:1.0.0-beta.1" where "1.0.0-beta.1" is the tag.
version				 | The version of the bundle.conf file. Should be set to `1`.

ConductR application bundles should not contain deployment specific configuration information such keys, passwords or secrets. Deployment target specific configuration and secrets should instead be set in a [configuration bundle](#Configuration-Bundles). The configuration bundle is deployed together with the application bundle. This enables a single application bundle to be deployed to multiple environments such test, staging and production by changing only the configuration bundle it is paired with at deployment instead of rebuilding the application bundle.

## Endpoints

Understanding endpoint declarations is important in order for your bundle to be able to become available within ConductR.

A bundle's component may be run either within a container or on the ConductR host. In either circumstance the bind interface and bind port provided by ConductR need to be used by a component in order to successfully bind and start listening for traffic. These are made available as `name_BIND_IP` and `name_BIND_PORT` environment variables respectively (see the "Standard Environment Variables" section toward the bottom of this document for a reference to all environment variables).

Because multiple bundles may run on the same host, and that their respective components may bind to the same port, we have a means of avoiding them clash.

The following port definitions are used:

Name         | Description
-------------|------------
service-port | The port number to be used as the public-facing port. It is proxied to the host-port.
host-port    | This is not declared but is dynamically allocated if bundle is running in a docker container. Otherwise it has the same value as bind-port.
bind-port    | The port the bundle component's application or service actually binds to. When this is 0 it will be dynamically allocated (which is the default).

Endpoints are declared using an `endpoint` setting using a Map of `endpoint-name` -> `Endpoint(bindProtocol, bindPort, serviceName, acls)` pairs.

The bind-port allocated to your bundle will be available as an environment variable to your component. For example, given the default settings where an endpoint named "web" is declared that has a dynamically allocated port, an environment variable named `WEB_BIND_PORT` will become available. `WEB_BIND_IP` is also available and should be used as the interface to bind to.  

Service names are declared through `service-name` property for each endpoint. A service name is used to address the service when performing a service lookup. [[Resolving services|ResolvingServices]] describes service resolution in a greater detail.

The request acls is declared through the `acls` property, and allows declaration of HTTP, TCP, and UDP based endpoints. [[ACL configuration|AclConfiguration]] describes how to configure request acls in a greater detail.

### File System Types

A component must declare its `file-system-type`. See the table below for a reference of available types and their behavior.

Type        | Description
------------|------------
universal   | The bundle component will be run outside of a container. The Host environment will therefore be available, including a Java runtime.
oci-image   | The bundle component contains a valid OCI image specification. These can be created from Docker images and contain everything needed to run your component without the need to resolve or build images at runtime. It will be run inside a container by `runc` for process isolation but will share and retain access to the host network. When running `oci-image` components in the Developer Sandbox, they are executed inside Docker containers to ensure a consistent development experience across operating systems.
oci-bundle  | The bundle component contains a valid OCI runtime specification. It will be run inside a container by `runc` for process isolation but will share and retain access to the host network. When running `oci-bundle` components in the Developer Sandbox, they are executed inside Docker containers to ensure a consistent development experience across operating systems.
docker      | The bundle component must contain a `Dockerfile` that will be built and run at the time of the bundle being run.

### Docker Containers and ports

When your component will run within a Docker container you may alternatively declare the bind port to be whatever it may be. Taking our Play example again, we can set the bind port with no problem of it clashing with another port given that it is run within a container:

```json
    endpoints        = {
      "angular-seed-play" = {
        protocol      = "http"
        bind-port     = 9000
        service-name  = "angular-seed-play"
        acls          = [
          {
            http = {
              requests = [
                {
                  path-beg = "/"
                }
              ]
            }
          }
        ]
      }
    }
```

### Legacy & third party bundles


#### When you cannot change your source

It is sometimes not possible or practical to change source code in order to signal successful startup or have it use the environment variables that ConductR provides.

As a quick example, suppose that you wish to bundle [ActiveMQ](http://activemq.apache.org/) as a Docker component with a `Dockerfile`. Suppose you have a bundle saved in `my-active-mq.zip` with a  `bundle.conf` defined as follows:

```
version               = "1"
name                  = "jms-docker"
compatibility-version = "1"

system         = "jms-docker"
system-version = "1"

nr-of-cpus   = 0.1
memory     = 67108864
disk-space = 10485760
roles      = ["jms"]

components = {
  "jms" = {
    description      = "A Docker container for Active/MQ"
    file-system-type = "docker"
    start-command    = []
    endpoints        = {
      "jms" = {
        bind-protocol = "tcp"
        bind-port     = 61616
        service-name  = "jms"
        acls          = [
          {
            tcp = {
              requests = [61616]
            }
          }
        ]
      }
    }
  }
}
```

ConductR provides a `check` command that bundle components may use to poll a tcp endpoint until it becomes available. The following example will add a `check` component that will correctly signal ConductR when your bundle has started.

```bash
conduct load my-active-mq.zip --check 'docker+$JMS_HOST'
```

`docker` instructs `check` to wait for all Docker components of this bundle to start and `JMS_HOST` is a [standard environment variable](https://github.com/sbt/sbt-bundle#standard-environment-variables) that will be provided at runtime given the `"jms"` endpoint declaration; it is a URI describing the JMS endpoint. You can similarly poll http endpoints and wait for them to become available. Note in this examples that the check parameter `docker+$JMS_HOST` is specific to the endpoint `jms.` An endpoint of `webserver` would use `docker+$WEBSERVER_HOST` instead.

#### To Containerize or Not

The [Open Container Initiative](https://www.opencontainers.org/) established an open standard for container formats and runtimes. Containers can be leveraged to provide process isolation and security gains as well as runtime dependency packaging.

Most Lightbend Reactive Platform (Lightbend RP) based programs should not require to be containerized as the host OS's Java Runtime Environment 8 (JRE 8) environment should be sufficient. Bundles generally contain all that is required for a Lightbend RP program to run, with exception to the Host OS and the host JRE. Lightbend RP, and being JVM based in general, bundles will start faster and incur less system resources when used without Docker.

Containers becomes relevant when there are specific runtime dependencies that are different to ConductR's host OS environment or when you wish to leverage the enhanced security capabilities provided by containers. In particular if a binary program that does not use the JVM is required to be launched from a bundle then it becomes more likely to benefit from using a container.

> For more container bundle options including OCI and Docker, see "[Creating Bundles](Creating Bundles)".

## Configuration Bundles

Configuration bundles are bundles containing only configuration values such as API keys and secrets. Configuration bundles are deployed together with application bundles. This keeps the configuration out of the application code and enables application bundles to be deployed to various environments without repackaging the application bundle.

To create a configuration bundle, run [`bndl`](https://github.com/typesafehub/conductr-cli/blob/master/README.rst#bndl) on a directory containing a `runtime-config.sh` and/or `bundle.conf` file. The `runtime-config.sh` file should export any environment variables to be set. The `bundle.conf` file should contain any bundle key settings to be overridden. Only the values to be overridden need to be specified. Bundle key values already defined the application bundle do not need to be redefined.

For example to set an application secret and override the application's declared role, simply create the configuration files into a directory and run `bndl` on the directory containing the files. You'll also need to provide a name for your bundle configuration file.

```bash
mkdir test-config
echo 'export "APPLICATION_SECRET=thisismyapplicationsecret-pleasedonttellanyone"'> test-config/runtime-config.sh
echo 'roles      = [partner-frontend]' > test-config/bundle.conf
bndl -f configuration -o test-config.zip test-config
```

The resultant bundle, i.e. `test-config.zip` is then loaded together with the application.

```bash
conduct load webserver-015f73613aa48d397b0dbab6d7f96d687c56d72a275a5ea43d7da44a21c27482.zip test-config.zip
```

Where `webserver-015f73613aa48d397b0dbab6d7f96d687c56d72a275a5ea43d7da44a21c27482.zip` is the application bundle.

Alternatively, you can also provide the configuration directory directly to `conduct load` if that better fits your workflow. In this case, the CLI will automatically create a configuration bundle for you.

```bash
conduct load webserver-015f73613aa48d397b0dbab6d7f96d687c56d72a275a5ea43d7da44a21c27482.zip test-config
```

### Configuration Bundle Files

It is also possible to include files in addition to the `runtime-config.sh` and/or `bundle.conf` file(s). To do this, create one or more files or sub folders with files alongside these files. Your `runtime-config.sh` script is then responsible for copying the additional files to the location that your application or service requires.

The script below demonstrates how to copy a config file to a target location.

```bash
BUNDLE_CONFIG_DIR="$( cd $( dirname "${BASH_SOURCE[0]}" ) && pwd )"
TARGET="/opt/ourApp/initConfig.cfg"

cp "${BUNDLE_CONFIG_DIR}"/someFile.cfg "${TARGET}"
```

If you needed to copy somewhere within the associated bundle component you could, for example:

```
TARGET="my-component/conf"
```

...where `my-component/conf` is the configuration folder of the associated bundle with a component named `my-component`.

### Dynamic Configuration Bundles

ConductR's CLI includes the ability to automatically generate a configuration bundle for you based on provided arguments. This allows you to specify `bundle.conf` parameters and environment variable values without explicitly creating any files. For example, the following loads a bundle and specifies roles and environment variables from the command line:

```bash
conduct load webserver-015f73613aa48d397b0dbab6d7f96d687c56d72a275a5ea43d7da44a21c27482.zip \
    --env APPLICATION_SECRET=thisismyapplicationsecret-pleasedonttellanyone \
    --roles partner-frontend
```

If you use these arguments in conjunction with an explicit configuration bundle, they will be merged into your `bundle.conf` and `runtime-config.sh`.

For a complete listing of arguments, be sure to consult `conduct load -h`.