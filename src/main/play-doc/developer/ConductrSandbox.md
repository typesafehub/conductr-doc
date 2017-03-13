# Managing ConductR sandbox cluster

sbt-conductr uses the ConductR sandbox image to spin up a local ConductR cluster. The sandbox makes it simple to use ConductR in a development context. It can be also utilized by Continuous Integration (CI) and other automation to validate bundles within a cluster context. The sandbox is available freely to all developers.

## Starting ConductR

The `sandbox run` command starts a local ConductR cluster. In order to use this command you need to specify the ConductR version. Note that this version is the version of ConductR itself and not the version of the `sbt-conductr` plugin.

Afterwards, start the sandbox inside the sbt session with:

```scala
[my-app] sandbox run 2.0.5
[info] Running ConductR...
[info] Running container cond-0 exposing 192.168.59.103:9000...
```

Given the above you will then have a ConductR process running in the background (there will be an initial download cost  to download the conductr/conductr-dev image from our registry).

### ConductR features

The sandbox contains handy features which can be optionally enabled during startup by specifying the `--feature` option, e.g.:

```scala
[my-app] sandbox run 2.0.5 --feature visualization
[info] Running ConductR...
[info] Running container cond-0 exposing 192.168.59.103:9000 192.168.59.103:9909...
```

The `visualization` feature provides a web interface to visualize the ConductR cluster together with deployed and running bundles. After enabling the feature, access it at http://{sandbox-host-ip}:9999. Replace `sandbox-host-ip` with your sandbox host ip address; this http://192.168.10.1:9999 by default.

[[images/visualizer_simple.png]]

To check out all available features head over to the [features](https://github.com/typesafehub/sbt-conductr#features) section of sbt-conductr.

### Stop ConductR

To stop the ConductR sandbox use:

```scala
sandbox stop
```

For more information about managing a ConductR cluster with sbt-conductr checkout its [documentation](https://github.com/typesafehub/sbt-conductr).
