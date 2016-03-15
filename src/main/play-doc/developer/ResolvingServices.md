# Resolving other services

When you create a bundle you can declare service names to ConductR. You can resolve a URL to the service if the service is running. For example if you have a RESTful `/api/accounts` service named `accountservice` you can resolve it from within your application or service as using Play WS.

Firstly establish the URL for locating the accounts service. Note that your code should look toward factoring out this type of behavior so that you do not hard-code host addresses or ports. For this example though, we'll relax that recommendation for the sake of clarity:

```Scala
import com.typesafe.conductr.bundlelib.scala.URL

val accounts = LocationService.getLookupUrl("/accountservice/api/accounts", URL("http://127.0.0.1:9000/api/accounts")
```

If the above code is run with your component has been started by ConductR then the `accounts` value will contain a URL to ConductR's service locator. Otherwise, when not running via ConductR then a service running on the loopback interface will be used; this latter one being something generally useful to you when in development mode, or an address for when running your component outside of ConductR.

You can then make a request:

```scala
WS
  .url(accounts)
  .withFollowRedirects(follow = true)
  .get()
  .map { response =>
    ...
  }
```

How easy is that!

What happens here is that by convention, ConductR will always interpret the first path component as the name of the service (`accountservice`) and remove that from the response. In the above case then, your service will receive a request at `/api/accounts`.

Where you have non-HTTP services to locate, or you want lower level access, you can use the ServiceLocator API directly. See the section on [TCP and UDP service lookups](TCPLookups) for more information on that. Even lower level [transport based APIs are also available](BundleAPI).
