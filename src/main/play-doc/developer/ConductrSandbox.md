# Managing ConductR sandbox cluster

sbt-conductr is using the ConductR sandbox image to spin up a local ConductR cluster. This is a docker image based on Ubuntu that includes ConductR which makes it simple to use ConductR in a development context. It can be also utilized by Continuous Integration (CI) and other automation to validate bundles within a cluster context. The sandbox is available to freely all developers.

> The docker image contains the full version of ConductR. However, it is not recommended to use this version in production because it is pre-configured for non production scenarios and because this ConductR version is started inside a Docker container.

## Starting ConductR

The `sandbox run` command starts a local ConductR cluster. In order to use this command you need to specify the ConductR version. Note that this version is the version of ConductR itself and not the version of the `sbt-conductr` plugin. Please visit the [ConductR Developer page](https://www.lightbend.com/product/conductr/developer) to pick up the latest ConductR version from the section **Quick Configuration**.

Afterwards, start the sandbox inside the sbt session with:

```scala
[my-app] sandbox run <CONDUCTR_VERSION>
[info] Running ConductR...
[info] Running container cond-0 exposing 192.168.59.103:9000...
```

> The ConductR sandbox will take a few seconds to become available and so any initial command that you send to it may not work immediately.

Given the above you will then have a ConductR process running in the background (there will be an initial download cost for Docker to download the conductr/conductr-dev image from the public Docker registry).

### ConductR features

The sandbox contains handy features which can be optionally enabled during startup by specifying the `--feature` option, e.g.:
    
```scala
[my-app] sandbox run <CONDUCTR_VERSION> --feature visualization
[info] Running ConductR...
[info] Running container cond-0 exposing 192.168.59.103:9000 192.168.59.103:9909...
```

The `visualization` feature provides a web interface to visualize the ConductR cluster together with deployed and running bundles. After enabling the feature, access it at http://{docker-host-ip}:9909. Replace `docker-host-ip` with your docker host ip address. For convience, the url of the visualizer app is displayed in the sbt session, e.g. http://192.168.59.103:9909.

[[images/visualizer_simple.png]]

To check out all available features head over to the [features](https://github.com/typesafehub/sbt-conductr#features) section of sbt-conductr.

### Stop ConductR

To stop the ConductR sandbox use:

```scala
sandbox stop
```

For more information about managing a ConductR cluster with sbt-conductr checkout its [documentation](https://github.com/typesafehub/sbt-conductr).
