# Deploying application bundles

Once you've created a bundle you can deploy it using ConductR's RESTful API, even using [curl](http://curl.haxx.se/) if you want. However we've made it a little easier than that. You can of course use ConductR's CLI as discussed in the [[Operations guide|CLI]]. Alternatively, given that we've been discussing bundling your application or service mostly from an sbt perspective, you can use [sbt-conductr](https://github.com/sbt/sbt-conductr#sbt-conductr). The sbt plugin makes the same commands available within the sbt session as the `conductr-cli` on the terminal.

The quickstart section introduced the `install` command so that you can conveniently load and run your entire project in ConductR. You can also control the loading and running of a bundle with `conduct` sub commands.

To load a bundle use the `conduct load` command:

```bash
conduct load <HIT THE TAB KEY AND THEN RETURN>
```

Using the tab completion feature of sbt will produce a URI representing the location of the last distribution
produced by the native packager. When using the `sandbox` the location of the ConductR cluster is automatically picked up.

To load a bundle to a remote ConductR cluster, use the `--ip` option:

```bash
conduct load --ip 172.14.0.1 <HIT THE TAB KEY AND THEN RETURN>
```

On successfully uploading the bundle the plugin will report the `BundleId` to use for subsequent commands on that bundle.

You can also run, stop and unload bundles by using this plugin. This may be useful to support your development lifecycle without having to jump into the operator's CLI.

> The host running sbt in this example must have access to the ConductR daemon ports. Please see  [[Cluster security considerations|ClusterSetupConsiderations#Cluster-security-considerations]] for further information on controlling cluster access.

That is all that is required in essence. For more information head out to the [documentation](https://github.com/sbt/sbt-conductr/blob/master/README.md) of `sbt-conductr`.

## Generating an installation script

An installation script can be conveniently generated for your project. Just like the `install` command, a `generateInstallationScript` command will introspect your project for bundles and associated configurations and then generate a bash based script. You are encouraged to copy this script and tailor it so that you can conveniently deploy your project's bundles to production and other environments.