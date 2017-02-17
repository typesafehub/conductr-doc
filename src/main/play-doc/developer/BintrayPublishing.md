# Bintray Publishing

ConductR provides bundles which are hosted at Bintray. These bundles can be resolved using a [shorthand expression](DeployingBundlesOps#Bundle-shorthand-expression), i.e. `conduct load visualizer` will resolve `visualizer` to the actual Visualizer bundle on Bintray.

It's possible to extend this treatment to your bundles by publishing them to Bintray. Once published to Bintray, the equivalent [shorthand expression](DeployingBundlesOps#Bundle-shorthand-expression) for your bundles can be used with the `conduct load` command to load your bundles into ConductR.

Publishing your bundles to Bintray is the first step to allow you to utilise ConductR's [Continuous Delivery](ContinuousDeliverySetupOps) feature.

## Requirements

Bintray credentials are required to publish bundle and bundle configuration.

## Installation

Add [sbt-bintray-bundle](https://github.com/sbt/sbt-bintray-bundle) to the project.

```
addSbtPlugin("com.typesafe.sbt" % "sbt-bintray-bundle" % "1.1.1")
```

At the time of writing, the latest version of [sbt-bintray-bundle](https://github.com/sbt/sbt-bintray-bundle) is `1.1.1`. Replace this version with the latest version of the plugin.


Configure the bintray credentials required for publishing.

```
$ cat ~/.bintray/.credentials
realm = Bintray API Realm
host = api.bintray.com
user = <obtain the user id from bintray>
password = <obtain the API key from bintray>
```

## Publishing bundles

### Single module project

Once the plugin is enabled, configure the publishing details of the bundle in the project's `build.sbt`.

Ensure SBT native packager or any of its plugin is enabled, e.g.

```
lazy val root = (project in file(".")).enablePlugins(JavaAppPackaging)
```

Add the following settings to enable publishing of the bundle. Replace the `licenses`, `bintrayVcsUrl`, `bintrayOrganization`, and `bintrayRepository` accordingly.

```
// A license is required for bintray packages
licenses := Seq("Apache-2.0" -> url("http://www.apache.org/licenses/LICENSE-2.0"))

inConfig(Bundle)(Seq(
  // A version control system url is required for bintray packages
  bintrayVcsUrl := Some("https://github.com/acme/my-project"),
  // Optionally, if you want to publish to an org repo other than your own
  bintrayOrganization := Some("orgname")
  // Optionally, if you want to change the name of the repo ("bundle" is the default)
  bintrayRepository := "test-bundle-repo",
))
```


Run the following command to publish the bundle.

```
sbt bundle:publish
```

With default settings, the bundle with its metadata and file will be staged to Bintray, but it won't be released until someone manually publishes the bundle.

To immediately release, set the setting `bintrayReleaseOnPublish` to `true`.


### Multi module project

Ensure the bundle's module has SBT native packager or any of its plugin is enabled, e.g. In this example, a bundle called `myModule` is declared from the project root's `build.sbt`.

```
lazy val myModule = project
  .in(file("my-module"))
  .enablePlugins(JavaAppPackaging)
```

Add the following settings to the module's `built.sbt` to enable publishing of the bundle. Replace the `licenses`, `bintrayVcsUrl`, `bintrayOrganization`, and `bintrayRepository` accordingly.

```
// A license is required for bintray packages
licenses := Seq("Apache-2.0" -> url("http://www.apache.org/licenses/LICENSE-2.0"))

inConfig(Bundle)(Seq(
  // A version control system url is required for bintray packages
  bintrayVcsUrl := Some("https://github.com/acme/my-project"),
  // Optionally, if you want to publish to an org repo other than your own
  bintrayOrganization := Some("orgname")
  // Optionally, if you want to change the name of the repo ("bundle" is the default)
  bintrayRepository := "bundle-repo",
))

```

Run the following command to publish the bundle.

```
sbt myModule/bundle:publish
```

## Publishing bundles configuration

Bundle configuration can also be published to bintray.

> Take care when publishing configurations that contain sensitive data e.g. passwords and secrets.

Ensure that the target repository on Bintray is protected by credentials.

### Single module project

Once the plugin is enabled, configure the publishing details of the bundle in the project's `build.sbt`.

Ensure SBT native packager or any of its plugin is enabled, e.g.

```
lazy val root = (project in file(".")).enablePlugins(JavaAppPackaging)
```

Add the following settings to enable publishing of the bundle. Replace the `licenses`, `bintrayVcsUrl`, `bintrayOrganization`, and `bintrayRepository` accordingly.

```
// A license is required for bintray packages
licenses := Seq("Apache-2.0" -> url("http://www.apache.org/licenses/LICENSE-2.0"))

inConfig(BundleConfiguration)(Seq(
  // A version control system url is required for bintray packages
  bintrayVcsUrl := Some("https://github.com/sbt/sbt-bintray-bundle"),
  // Optionally, if you want to publish to an org repo other than your own
  bintrayOrganization := Some("orgname")
  // Optionally, if you want to change the name of the repo ("bundle" is the default)
  bintrayRepository := "bundle-configuration-repo",
))
```

Run the following command to publish the bundle.

```
sbt myModule/configuration:publish
```

### Multi module project

Ensure the bundle's module has SBT native packager or any of its plugin is enabled, e.g. In this example, a bundle called `myModule` is declared from the project root's `build.sbt`.

```
lazy val myModule = project
  .in(file("my-module"))
  .enablePlugins(JavaAppPackaging)
```

Add the following settings to the module's `built.sbt` to enable publishing of the bundle. Replace the `licenses`, `bintrayVcsUrl`, `bintrayOrganization`, and `bintrayRepository` accordingly.

```
// A license is required for bintray packages
licenses := Seq("Apache-2.0" -> url("http://www.apache.org/licenses/LICENSE-2.0"))

inConfig(BundleConfiguration)(Seq(
  // A version control system url is required for bintray packages
  bintrayVcsUrl := Some("https://github.com/acme/my-project"),
  // Optionally, if you want to publish to an org repo other than your own
  bintrayOrganization := Some("orgname")
  // Optionally, if you want to change the name of the repo ("bundle" is the default)
  bintrayRepository := "bundle-configuration-repo",
))

```

Run the following command to publish the bundle.

```
sbt myModule/configuration:publish
```

### Multiple configurations

When a project/module has multiple configuration, the configuration can be published as such.

Consider the following configuration. The project has a module called `my-module` which has two configurations called `mode-open` and `mode-yellow`.

```
my-module/src/bundle-configuration/
                                   |
                                   |- mode-open/
                                                |- runtime-config.sh
                                   |
                                   |- mode-yellow/
                                                  |- runtime-config.sh
```

Assuming `mode-open` is the default bundle configuration having `licenses`, `bintrayVcsUrl`, `bintrayOrganization`, and `bintrayRepository` settings configured as such.

```
BundleKeys.configurationName := "mode-open"

licenses := Seq("Apache-2.0" -> url("http://www.apache.org/licenses/LICENSE-2.0"))

inConfig(BundleConfiguration)(Seq(
  // A version control system url is required for bintray packages
  bintrayVcsUrl := Some("https://github.com/acme/my-project"),
  // Optionally, if you want to publish to an org repo other than your own
  bintrayOrganization := Some("orgname")
  // Optionally, if you want to change the name of the repo ("bundle" is the default)
  bintrayRepository := "bundle-configuration-repo",
))
```

The configuration `mode-yellow` can be configured as such. The `licenses`, `bintrayVcsUrl`, `bintrayOrganization`, and `bintrayRepository` settings for `mode-yellow` will be extended from `mode-open`. Note `BundleKeys.configurationName in modeYellow` which sets the

```
lazy val modeYellow = config("mode-yellow").extend(BundleConfiguration)
BundlePlugin.configurationSettings(modeYellow)
BundleKeys.configurationName in modeYellow := "mode-yellow"
BintrayBundle.settings(modeYellow, isBundleConfiguration = true)
```

To publish the `mode-yellow` configuration, run the following command.

```
sbt myModule/mode-yellow:publish
```