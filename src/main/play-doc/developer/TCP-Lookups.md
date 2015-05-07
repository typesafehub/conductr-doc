# Typesafe ConductR %PLAY_VERSION%


## TCP and UDP service lookup

ConductR offers an [`/etc/services`](http://www.lehman.cuny.edu/cgi-bin/man-cgi?services+4) style of experience for situations where you need to resolve a TCP or UDP port (note that if you're using [HAProxy](http://www.haproxy.org/) for proxying then UDP is not supported at this time). Resolving ports at runtime is a good practice given that your application or service becomes de-coupled from the actual port in use.

The general idea is that you call a lookup function each time that you need to make a call to a service. Unfortunately many libraries that you use will not provide you with the opportunity to make that call, and so you may have to resort to an initial static lookup. The following example attempts to locate a fictitious JMS broker service:

```scala
// This will require an implicit ConnectionContext to
// hold a Scala ExecutionContext. There are different
// ConnectionContexts depending on which flavor of the
// library is being used. For the Scala flavor, a Scala
// ExecutionContext is composed. The ExecutionContext
// is needed as "service" is returned as a Future.
// For convenience, we provide a global ConnectionContext
// that may be imported.
import com.typesafe.conductr.bundlelib.scala.ConnectionContext.Implicits.global

// We also provide a cache designed specifically to
// hold location values. Caches are optional to the
// lookup, but they are encouraged.
val locationCache = LocationCache()

// ...and the lookup itself...
val jmsBroker = LocationService.lookup("/jms", locationCache)
```

`jmsBroker` is typed `Future[Option[String]]` meaning that an optional response with the resolved URI will be returned at some time in the future. Supposing that this lookup is made during the initialisation of your program, the service you're looking for may not exist. However calling the same function later may yield the service. This is because services can come and go.

Ideally, we would push the resolution of a service back on ConductR in a similar manner to how HTTP paths are resolved and use DNS for looking up services. Unfortunately [DNS A-records](http://support.simpledns.com/kb/a35/can-i-specify-a-tcp-ip-port-number-for-my-web-server-in-dns-other-than-the-standard-port-80.aspx) do not yield a port number, and there is little library usage of [DNS SVC-record](http://en.wikipedia.org/wiki/SRV_record) types and, by extension, [zeroconf](http://en.wikipedia.org/wiki/Zero-configuration_networking#Link-local_IPv4_addresses).

## TCP and UDP static service lookup

Some bundle components cannot proceed with their initialization unless the service can be located. We encourage you to re-factor these components so that they look up services at the time when they are required, given that services can come and go. However if you are somehow stuck with this style of code then you may consider the following blocking code as a work-around measure:

```scala
val resultUri = Await.result(
  LocationService.lookup("/someservice"),
  sometimeout)
val serviceUri = resultUri.getOrElse {
  if (Env.isRunByConductR) System.exit(70)
  "http://127.0.0.1:9000"
}
```

In the above, the program will exit if a service cannot be located at the time the program initializes; unless the program has not been started by ConductR in which case an alternate URI is provided. Instead of blocking you may also consider using an Akka actor:

```scala
// bundlelib types are imported from com.typesafe.conductr.bundlelib.akka

// ImplicitConnectionContext is a convenience that we provide for obtaining
// a connection context within an actor.

class MyService extends Actor with ImplicitConnectionContext {

  import context.dispatcher

  override def preStart(): Unit =
    LocationService.lookup("/someservice").pipeTo(self)

  override def receive: Receive =
    initial

  private def initial: Receive = {
    case Some(someService: String) =>
      // We now have the service

      context.become(service(someService))

    case None =>
      self ! (if (Env.isRunByConductR) PoisonPill else Some("http://127.0.0.1:9000"))
  }

  private def service(someService: String): Receive = {
    // Regular actor receive handling goes here given that we have a service URI now.
    ...
  }
}
```

This type of actor is used to handle service processing and should only receive service oriented messages once its dependent service URI is known. This is an improvement on the blocking example provided before, as it will not block. However it still has the requirement that `someservice` must be running at the point of initialisation, and that it continues to run. Neither of these requirements may always be satisfied with a distributed system.