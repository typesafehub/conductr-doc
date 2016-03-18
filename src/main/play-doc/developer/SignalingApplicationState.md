# Signaling application state

Your application or service should tell ConductR when it has completed its initialization and is ready for work. We have a fall-back strategy for where this is not possible or practical (more on that later), but for now, let's assume that you can modify your source code.

For a [Play](https://www.playframework.com/) application or service, signalling successful startup may be when its http endpoint is online having declared a connection to a database. For an [Akka](http://akka.io/) based application then it may be when your actor system has been initialized and you have connected to some other service.

ConductR's only requirement of you is to use a library and call a single function as `StatusService.signalStartedOrExit()`. Note that if you call this function outside of running within ConductR then it does nothing so you can continue to develop and debug as you have always done.

The library comes in multiple flavours: 
- Java 8 : `java-conductr-bundle-lib`
- Scala 2.11 / JDK: `scala-conductr-bundle-lib`
- Akka 2.3.x/2.4.x: `akka23-conductr-bundle-lib`
- Play 2.3.x: `play23-conductr-bundle-lib`
- Play 2.4.x: `play24-conductr-bundle-lib`

To use it add one of the libraries as a dependency to your `build.sbt`:

```scala
resolvers += "typesafe-releases" at "http://repo.lightbend.com/typesafe/maven-releases"

libraryDependencies += "com.typesafe.conductr" %% "scala-conductr-bundle-lib" % "1.4.1"
```

The Java and Scala / JDK library have no dependencies other than the JDK and as such, a blocking implementation is used for its HTTP calls (the JDK offers no non-blocking APIs for this). Using the Akka or Play library will ensure that the library is consistent with the respective Akka or Play application and that non-blocking implementations are used:
- Akka: akka-http
- Play: Play.WS

When you are reasonably sure that your code is ready to start processing (it doesn't have to be exactly at that time - ConductR tolerates services being unavailable), call the `signalStartedOrExit` function. 

**Scala Example**
Library: `scala-conductr-bundle-lib`

```scala
import com.typesafe.conductr.lib.scala.ConnectionContext.Implicits._

...

StatusService.signalStartedOrExit()
```

After reading the following sections, you may also wish to refer to [the reference documentation for conductr-bundle-lib](https://github.com/typesafehub/conductr-bundle-lib#typesafe-conductr-bundle-library).
