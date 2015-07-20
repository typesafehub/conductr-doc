# Deploying application bundle

Once you've created a bundle you can deploy it using ConductR's RESTful API, even using [curl](http://curl.haxx.se/) if you want. However we've made it a little easier than that. You can of course use ConductR's CLI as discussed in the _Operator Quickstart_ above. Alternatively given that we've been discussing bundling your application or service mostly from an sbt perspective, you can use another plugin named [`sbt-conductr`](https://github.com/sbt/sbt-conductr#sbt-conductr).

The following description is intended to provide a taste of what `sbt-conductr` can do for you. Please refer to [its documentation](https://github.com/sbt/sbt-conductr/blob/master/README.md) as there are some small considerations when dealing with the pre 1.0 `sbt-native-packager` e.g. the one used with Play 2.3.

To use `sbt-conductr` first add the plugin your build (typically your `project/plugins.sbt` file); be sure to check at [the plugin's website](https://github.com/sbt/sbt-conductr#sbt-conductr) for the latest version to use:

```scala
addSbtPlugin("com.typesafe.conductr" % "sbt-conductr" % "0.36.0")
```

Note that if you add this plugin as above, you do not need to have an explicit declaration for `sbt-bundle`. `sbt-bundle` will be automatically added as a dependency of `sbt-conductr`.

The `sbt-conductr` plugin must then be enabled for your project. Supposing that your project has one module that will use the plugin which is the root of the sbt project (the most typical situation for a single `build.sbt`):

```scala
lazy val root = project
  .in(file("."))
  .enablePlugins(ConductRPlugin, <your other plugins go here>)
```

With your declarations out of the way, you can produce a bundle by typing:

```bash
bundle:dist
```

A bundle will be produced from the native packager settings of this project. A bundle effectively wraps a native
packager distribution and includes some component configuration. To load the bundle first declare the location of ConductR (supposing that ConductR is running on `172.14.0.1`:

```bash
controlServer 172.14.0.1
```

...and then load:

```bash
conduct load <HIT THE TAB KEY AND THEN RETURN>
```

Using the tab completion feature of sbt will produce a URI representing the location of the last distribution
produced by the native packager.

Hitting return will cause the bundle to be uploaded. On successfully uploading the bundle the plugin will report
the `BundleId` to use for subsequent commands on that bundle.

You can also run, stop and unload bundles by using this plugin. This may be useful to support your development lifecycle without having to jump into the operator's CLI.

That is all that is required in essence, but as stated, you should read [`sbt-conductr`'s documentation](https://github.com/sbt/sbt-conductr/blob/master/README.md) as there are a few additional requirements, particularly if you are managing a Play 2.3 application.

Now go and develop reactive applications or services for ConductR!
