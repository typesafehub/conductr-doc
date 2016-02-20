# Advanced sbt sandbox

> Note that the ConductR sandbox is available as both an sbt plugin, and also as a command line tool. This document describes the sbt plugin variant and its position in terms of your development activities.

A Developer Sandbox Docker image is available so that developers can validate and debug ConductR applications without needing to run a full multi-node cluster. The sbt plugin [sbt-conductr-sandbox](https://github.com/typesafehub/sbt-conductr-sandbox) is using this Docker image and can be utilized locally by developers as well as by Continous Integration (CI) and other automation to validate bundles within a cluster context.

The ConductR Developer Sandbox is available to freely all developers. To request access to the sandbox, login to lightbend.com to visit the [ConductR Developer page](https://www.lightbend.com/product/conductr/developer).

> The Docker image of `sbt-conductr-sandbox` contains the full version of ConductR. However, it is not recommended to use this version in production because it is pre-configured for non production scenarios and because this ConductR version is started inside a Docker container.

The following description is intended to provide a taste of what `sbt-conductr-sandbox` can do for you. Please refer to [its documentation](https://github.com/typesafehub/sbt-conductr-sandbox) for more details.

## Setting up sbt-conductr-sandbox

1. Add the sbt plugin to the `project/plugins.sbt` of your project:

    ```scala
    addSbtPlugin("com.typesafe.conductr" % "sbt-conductr-sandbox" % "1.2.1")
    ```
2. Configure the `build.sbt` as described by the [ConductR download page](https://www.lightbend.com/product/conductr/developer).

3. Reload the sbt session:

    ```scala
    reload
    ```    

The plugin is then enabled automatically for your entire project. Additionally it includes `sbt-conductr` and `sbt-bundle` so these plugins doesn't need to be added separately. The `conduct` commands such as `conduct info` will automatically communicate with the Docker cluster managed by the sandbox. There is no need to set ConductR's ip address with `controlServer` manually.

## Using sbt-conductr-sandbox

### Starting ConductR

Start the sandbox inside the sbt session with:

```scala
[my-app] sandbox run
[info] Running ConductR...
[info] Running container cond-0 exposing 192.168.59.103:9000...
```

> Note that the ConductR sandbox will take a few seconds to become available and so any initial command that you send to it may not work immediately.

Given the above you will then have a ConductR process running in the background (there will be an initial download cost for Docker to download the conductr/conductr-dev image from the public Docker registry).

#### ConductR features

> In order to use the following features then you should ensure that the machine that runs Docker has enough memory, typically at least 2GB. VM configurations such as those provided via `boot2docker` and Oracle's VirtualBox can be configured like so:
>
>     boot2docker down
>     VBoxManage modifyvm boot2docker-vm --memory 2048
>     boot2docker up

ConductR gets shipped with features which can be optionally enabled during startup by specifying the `--withFeatures` option, e.g.:
    
```scala
[my-app] sandbox run --withFeatures visualization logging
[info] Running ConductR...
[info] Running container cond-0 exposing 192.168.59.103:9000 192.168.59.103:9909...
```

The `visualization` feature provides a web interface to visualize the ConductR cluster together with deployed and running bundles. After enabling the feature, access it at http://{docker-host-ip}:9909. Replace `docker-host-ip` with your docker host ip address. For convience, the url of the visualizer app is displayed in the sbt session, e.g. http://192.168.59.103:9909.

[[images/visualizer_simple.png]]

The `logging` feature consolidates the logging output of ConductR itself and the bundles that it executes. To view the consolidated log messsages run:

```scala
conduct logs conductr-elasticsearch
```

For more information refer to the [[Logging|Logging]] section.

### Debugging ConductR

It is possible to debug your application inside of the ConductR sandbox:

1. Start ConductR sandbox in debug mode:

    ```scala
    sandbox debug
    ```
2. This exposes a debug port to docker and the ConductR container. By default the debug port is set to `5005` which is the default port for remote debugging in IDEs such as IntelliJ. If you are happy with the default setting skip this step. Otherwise specify another port in the `build.sbt`:

    ```scala
    SandboxKeys.debugPort := 5432
    ```    
3. Create the application bundle. The JVM argument `-jvm-debug $debugPort` is automatically added to the bundle configuration when using:

    ```scala
    bundle:dist
    ```
4. Load and run your bundle to the sandbox, e.g. by using [sbt-conductr](https://github.com/sbt/sbt-conductr):

    ```scala
    conduct load <HIT THE TAB KEY AND THEN RETURN>
    conduct run my-app
    ```

You should be now able to remotely debug the bundle inside the ConductRs sandbox with your favorite IDE.

**IntelliJ**

1. In IntelliJ navigate to `Run` -> `Edit Configurations`
2. Create a new `Remote` configuration with these options:
    - Transport: Socket
    - Debugger mode: Attach
    - Host: Your docker host ip, e.g. 192.168.59.103
    - Port: `debugPort` of your application, e.g. 5005

    [[images/conductr_sandbox_debugging_intellij.png]]
3. Save this configuration and start remote debugging with `Run` -> `Debug` and then select your configuration.

**Eclipse**

1. In Eclipse navigate to `Run` -> `Debug Configurations..`
2. Create a new `Remote Java Application` with these options:
    - Connection Type: Standard (Socket Attach)
    - Host: Your docker host ip, e.g. 192.168.59.103
    - Port: `debugPort` of your application, e.g. 5005

    [[images/conductr_sandbox_debugging_eclipse.png]]
3. Select `Debug` to start the remote debug session.

### Stop ConductR

To stop the ConductR sandbox use:

```scala
sandbox stop
```