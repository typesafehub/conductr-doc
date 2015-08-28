# Development Lifecycle

ConductR simplifies the deployment of applications with resilience and elasticity without disrupting the application development lifecycle. Developers continue to develop and test their applications as they normally would prior to deployment. Users of [sbt](http://www.scala-sbt.org/) for example will continue to use `sbt run` and `sbt test` to run and test their applications locally.

Once the application is ready for deployment, developers can view ConductR bundles as just yet another deployment target. Just as developers might use `play dist` or `sbt debian:packageBin` to produce production binaries of an application, the sbt `bundle:dist` task from [sbt-bundle](https://github.com/sbt/sbt-bundle#typesafe-conductr-bundle-plugin) is used to produce ConductR application bundles. Even applications leveraging ConductR specific features such as [service discovery](ResolvingServices.html) can be tested locally without deployment to a running cluster by using the fallback address feature.

ConductR application bundles should not contain deployment specific configuration information such keys, passwords or secrets. Deployment target specific configuration and secrets should instead be set in a [configuration bundle](BundleConfiguration.html# Configuration-Bundles). The configuration bundle is deployed together with the application bundle. This enables a single application bundle to be deployed to multiple environments such test, staging and production by changing only the configuration bundle it is paired with at deployment instead of rebuilding the application bundle.

# Development Sandbox

A Developer Sandbox Docker image is available so that developers can validate and debug ConductR applications without needing to run a full multi-node cluster. The [ConductR Sandbox](https://github.com/typesafehub/sbt-conductr-sandbox) can be utilized locally by developers as well as by Continous Integration (CI) and other automation to validate bundles within a cluster context.

The ConductR Developer Sandbox is available to freely all developers. To request access to the sandbox, login to Typesafe.com to visit the [ConductR Developer page](https://www.typesafe.com/product/conductr/developer).

