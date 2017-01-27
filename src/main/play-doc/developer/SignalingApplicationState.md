# Signaling application state

> For Play and Lagom applications, `sbt-conductr` automatically signals that the application has been started or exited. For these kind of application the following instructions can be skipped.

Your application or service should tell ConductR when it has completed its initialization and is ready for work. We have a fall-back strategy for where this is not possible or practical (more on that later), but for now, let's assume that you can modify your source code.

For a [Play](https://www.playframework.com/) application or service, signalling successful startup may be when its http endpoint is online having declared a connection to a database. For an [Akka](http://akka.io/) based application then it may be when your actor system has been initialized and you have connected to some other service.

ConductR's only requirement of you is to use a library and call a single function as `StatusService.signalStartedOrExit()`. Note that if you call this function outside of running within ConductR then it does nothing so you can continue to develop and debug as you have always done.

The library comes in multiple flavours: 
- Java 8 : `java-conductr-bundle-lib`
- Scala 2.11 / JDK: `scala-conductr-bundle-lib`
- Akka 2.3.x/2.4.x: `akka23-conductr-bundle-lib`
- Play 2.3.x: `play23-conductr-bundle-lib`
- Play 2.4.x: `play24-conductr-bundle-lib`
- Lagom 1.x.x: `lagom1-conductr-bundle-lib`

To use it add one of the libraries as a dependency to your `build.sbt`:

```scala
libraryDependencies += "com.typesafe.conductr" %% "scala-conductr-bundle-lib" % "1.4.14"
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

## Application Exit Codes

ConductR uses your application's exit code to determine whether it should be restarted in order to reform the cluster. Following POSIX conventions, applications with non-zero exit codes are considered failed and will be restarted, while applications exiting with exit code 0 will be viewed as having exited normally, and thus ConductR will take no action.

This is particularly important for Akka applications using [Split Brain Resolver](http://doc.akka.io/docs/akka/rp-current/scala/split-brain-resolver.html) with the `registerOnMemberRemoved` callback to shut down the ActorSystem after a split brain downing decision has been made. A non-zero return code must be issued to ensure that ConductR is instructed to recover the cluster to its previous state.

This is how to shut down the ActorSystem and exit the JVM such that the application will be restarted by ConductR after being downed by Split Brain Resolver:
```scala
Cluster(system).registerOnMemberRemoved {
  // exit JVM with a non-zero exit code when ActorSystem has been terminated 
  system.registerOnTermination(System.exit(-1))
  // shut down ActorSystem
  system.terminate()

  // In case ActorSystem shutdown takes longer than 10 seconds,
  // exit the JVM forcefully.
  // We must spawn a separate thread to not block current thread,
  // since that would have blocked the shutdown of the ActorSystem.
  new Thread {
    override def run(): Unit = {
      if (Try(Await.ready(system.whenTerminated, 10.seconds)).isFailure)
        System.exit(-1)
    }
  }.start()
}
```

