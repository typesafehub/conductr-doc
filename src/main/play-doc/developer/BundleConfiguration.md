# Bundle configuration

ConductR has a bundle format in order for components to be described. In general there is a one-to-one correlation between a bundle and a component.

Bundles provide ConductR with some basic knowledge about components in a *bundle descriptor*; in particular, what is required in order to load and run a component. The following is an example of a `bundle.conf` descriptor:
([Typesafe configuration](https://github.com/typesafehub/config) is used):

```
version               = "1"
name                  = "simple-test"
compatibilityVersion  = "1"
system                = "simple-test"
systemVersion         = "1"
nrOfCpus              = 1.0
memory                = 67108864
diskSpace             = 10485760
roles                 = ["web"]
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
bind-port			 | Discussed [below](#Endpoints).
compatibilityVersion | A versioning scheme that will be included in a bundle's name that describes the level of compatibility with bundles that go before it. By default we take the major version component of a version as defined by [http://semver.org/]. However you can make this mean anything that you need it to mean in relation to bundles produced prior to it. We take the notion of a compatibility version from [http://ometer.com/parallel.html].
components			 | Each bundle has at least one component, and generally just one. A bundle component contains all that is required to run itself in terms of its directory on disk when the bundle is expanded. This section describes the meta data required for each bundle component.
description			 | A human readable description of the bundle component.
diskSpace            | The amount of disk space required to host an expanded bundle and configuration.
endpoints            | Discussed [below](#Endpoints).
file-system-type	 | Describes the type of the bundle and can be either "universal" or "docker". A universal type means that this bundle copmonent will be run outside of a container. The Host environment will therefore be available, including a Java runtime. Docker types expect a Dockerfile to reside within a component. The Docker component will be built and run at the time of the bundle being run.
memory               | The total amount of system memory required to run the bundle.
name				 | The human readable name of the bundle. This name appears often in operational output such as the CLI.
nrOfCpus             | The number of cpus required to run the bundle (can be fractions thereby expressing a portion of CPU). Required.
protocol			 | Discussed [below](#Endpoints).
roles                | The types of node in the cluster that this bundle can be deployed to. Defaults to "web".
service-name	     | Discussed [below](#Endpoints).
acls     			 | Discussed [below](#Endpoints).
start-command        | Command line args required to start the component. Paths are expressed relative to the component's bin folder. The default is to use the bash script in the bin folder. Arguments can be passed to a Docker container run command via a special `dockerArgs` command should additional args be required: `start-command = ["dockerArgs","-v","/var/lib/postgresql/data:/var/lib/postgresql/data"]`.
system               | A logical name that can be used to associate multiple bundles with each other. This could be an application or service association e.g. myapp. Defaults to the package name.
systemVersion        | A version to associate with a system. This setting defaults to the value of compatibilityVersion.
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
host-port    | This is not declared but is dynamically allocated if bundle is running in a container. Otherwise it has the same value as bind-port.
bind-port    | The port the bundle component's application or service actually binds to. When this is 0 it will be dynamically allocated (which is the default).

Endpoints are declared using an `endpoint` setting using a Map of `endpoint-name` -> `Endpoint(bindProtocol, bindPort, serviceName, acls)` pairs.

The bind-port allocated to your bundle will be available as an environment variable to your component. For example, given the default settings where an endpoint named "web" is declared that has a dynamically allocated port, an environment variable named `WEB_BIND_PORT` will become available. `WEB_BIND_IP` is also available and should be used as the interface to bind to.  

Service names are declared through `serviceName` property for each endpoint. A service name is used address the service when performing a service lookup. [[Resolving services|ResolvingServices]] describes service resolution in a greater detail.

The request acls is declared through the `acls` property, and allows declaration of HTTP, TCP, and UDP based endpoints. [[ACL configuration|AclConfiguration]] describes how to configure request acls in a greater detail.

### Docker Containers and ports

When your component will run within a container you may alternatively declare the bind port to be whatever it may be. Taking our Play example again, we can set the bind port with no problem of it clashing with another port given that it is run within a container:

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

We provide a [CLI](https://github.com/typesafehub/typesafe-conductr-cli#command-line-interface-cli-for-typesafe-conductr) command named [`shazar`](https://github.com/typesafehub/typesafe-conductr-cli#shazar) for bundling the contents of any folder. You can therefore hand-craft a `bundle.conf` and its component folders and use `shazar` to bundle it.

As a quick example, suppose that you wish to bundle [ActiveMQ](http://activemq.apache.org/) as a Docker component with a `Dockerfile`. You can do something like this (btw: we appreciate that you cannot change the world in one go and don't always have the luxury of using Akka for messaging!):

```
version               = "1"
name                  = "jms-docker"
compatibilityVersion  = "1"

system         = "jms-docker"
systemVersion  = "1"

nrOfCpus   = 1.0
memory     = 67108864
diskSpace  = 10485760
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
  "jms-status" = {
    description      = "Status check for the jms component"
    file-system-type = "universal"
    start-command    = ["check", "docker+$JMS_HOST"]
    endpoints        = {}
  }
}
```

The declaration of interest is the `jms-status` component. ConductR provides a `check` command that bundle components may use to poll a tcp endpoint until it becomes available. `docker` instructs `check` to wait for all Docker components of this bundle to start and `JMS_HOST` is a [standard environment variable](https://github.com/sbt/sbt-bundle#standard-environment-variables) that will be provided at runtime given the `"jms"` endpoint declaration; it is a URI describing the JMS endpoint. You can similarly poll http endpoints and wait for them to become available. Note in this examples that the check parameter `docker+$JMS_HOST` is specific to the endpoint `jms.` An endpoint of `webserver` would use `docker+$WEBSERVER_HOST` instead.

#### To Docker or Not

[Docker](https://www.docker.com/) is a technology that provides containers for your application or service. Most Typesafe Reactive Platform (Typesafe RP) based programs should not require Docker as the host OS's Java Runtime Environment 8 (JRE 8) environment should be sufficient. Bundles generally contain all that is required for a Typesafe RP program to run, with exception to the Host OS and the host JRE. Typesafe RP bundles will start faster and incur less system resources when used without Docker.

Docker becomes relevant when there are specific runtime dependencies that are different to ConductR's host OS environment. In particular if a binary program that does not use the JVM is required to be launched from a bundle then it becomes more likely to benefit from using a Docker container.

#### When you cannot Dockerize existing services - a Redis example

If you cannot run a Linux service in Docker but would still like the services to be managed by ConductR, it may be possible to run services without a container. To do this, create an application bundle of the full service installation folder and a configuration bundle containing any scripting needed to control the service. 

For example, to a run a single instance of [Redis](http://redis.io/) under ConductR without a container, start with a built version of redis. For this example `redis/redis-stable` will be the top level folder of the Redis installation containing the Redis README, MANIFESTO and COPYING. Redis will be executing natively on the member node and must be built for the target node's distribution. We will use roles to ensure that this bundle only runs on specific nodes.

In the `redis` folder, add the following as a file named `bundle.conf`.

```bash
version               = "1"
name                  = "redis-broker"
compatibilityVersion  = "3"
system                = "redis"
systemVersion         = "3"
nrOfCpus              = 2.0
memory                = "1342701568"
diskSpace             = "10737418240"
roles                 = [db, redis]

components = {
  "logbroker" = {
    description       = "redis-broker"
    file-system-type  = "universal"
    start-command     = ["./redis.sh"]
    endpoints         = {
      "logbroker" = {
        bind-protocol  = "tcp"
        bind-port     = 0
        service-name  = "redis"
        acls          = [
          {
            tcp = {
              requests = [6379]
            }
          }
        ]
      }
    }
  }
  "logbroker-status" = {
   description      = "Status check for the redis logbroker component"
   file-system-type = "universal"
   start-command    = ["check", "$LOGBROKER_HOST"]
   endpoints        = {}
  }
}
```

Also in the `redis` folder, create the executable file `redis.sh` specified in the `start-command` as follows. In addition to starting Redis, the script uses `redis-cli` to shutdown Redis upon a shutdown signal.

```bash
#!/usr/bin/env bash

shutdown() {
   redis-stable/src/redis-cli shutdown -h $LOGBROKER_BIND_IP -p $LOGBROKER_BIND_PORT
}

trap ‘shutdown’ SIGTERM SIGINT SIGHUP

redis-stable/src/redis-server redis-stable/redis.conf
```

In `redis-stable/redis.conf` remove the `port 6379` directive. We want HAProxy to bind 6379, not Redis. We will not be daemonizing Redis, specifying a bind address or a logfile in `redis-conf`. 

Create the application bundle
```bash
shazar redis
```

In a new folder namned `redis-config`, create an executable file named `runtime-config.sh` containing the following:

```bash
echo bind $LOGBROKER_BIND_IP | tee -a redis-stable/redis.conf
echo port $LOGBROKER_BIND_PORT | tee -a redis-stable/redis.conf
```

This obtains the ConductR assigned IP address and bind port and appends them to the redis.conf. Create the configuration bundle.

```bash
shazar redis-config
```
Load the application and configuration bundle together and run the bundle. To constrain which nodes Redis will run on, our `bundle.conf` contains the roles `db` and `redis`. Only member nodes providing at least these two roles will be eligable to run Redis.

## Configuration Bundles

Configuration bundles are bundles containing only configuration values such as API keys and secrets. Configuration bundles are deployed together with application bundles. This keeps the configuration out of the application code and enables application bundles to be deployed to various environments without repackaging the application bundle.

To create a configuration bundle, run [`shazar`](https://github.com/typesafehub/typesafe-conductr-cli#shazar) on a directory containing a `runtime-config.sh` and/or `bundle.conf` file. The `runtime-config.sh` file should export any environment variables to be set. The `bundle.conf` file should contain any bundle key settings to be overridden. Only the values to be overridden need to be specified. Bundle key values already defined the application bundle do not need to be redefined.

For example to set an application secret and override the application's declared role, simply create the configuration files into a directory and shazar the directory containing the files. The configuration bundle will take its name from the directory name.

```bash
mkdir test-config
echo 'export "APPLICATION_SECRET=thisismyapplicationsecret-pleasedonttellanyone"'> test-config/runtime-config.sh
echo 'roles      = [partner-frontend]' > test-config/bundle.conf
shazar test-config
```

The resultant bundle, i.e. `test-config-2ddf3c2453ad16589b7bfae316e6ec746418491292f874b72236741bbd8f84ab.zip` is then loaded together with the application.

```bash
conduct load webserver-015f73613aa48d397b0dbab6d7f96d687c56d72a275a5ea43d7da44a21c27482.zip test-config-2ddf3c2453ad16589b7bfae316e6ec746418491292f874b72236741bbd8f84ab.zip
```

Where `webserver-015f73613aa48d397b0dbab6d7f96d687c56d72a275a5ea43d7da44a21c27482.zip` is the application bundle.

### Configuration Bundle Files

It is also possible to include files in addition to the `runtime-config.sh` and/or `bundle.conf` file(s). To do this, create one or more files or sub folders with files alongside these files. Your `runtime-config.sh` script is then responsible for copying the additional files to the location that your application or service requires.

The use of the special parameter `$0` is recommended in configuration scripts. The script below demonstrates the use of `$0` to copy a config file to a target location.

```bash
SCRIPT_DIR="`dirname \"$0\"`"
TARGET="/opt/ourApp/initConfig.cfg"

cp "${SCRIPT_DIR}"/someFile.cfg ${TARGET}
```

> `runtime-config.sh` is the name preferred by ConductR when searching for a script to execute within a configuration bundle. However you can name the configuration script anything that you like. In the case of ambiguity though, for example where you have `a.sh` and `b.sh`, ConductR does not know which is the one to select, and it can select either depending on your JDK. In most cases then, you're better to stick with `runtime-config.sh`
