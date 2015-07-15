# Resolving other services

When you create a bundle (we'll show you how to do that in the next section) you can declare service names to ConductR. You can resolve a URL to the service if the service is running. For example if you have a RESTful `/accounts` service you can resolve it from within your application or service as using Play WS.

Firstly establish the URL for locating the accounts service. Note that your code should look toward factoring out this type of behavior so that you do not hard-code host addresses or ports. For this example though, we'll relax that recommendation for the sake of clarity:

```Scala
val accounts = LocationService.getLookupUrl("/accounts", "http://127.0.0.1:9000/accounts")
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

Where you have non-HTTP services to locate, you can use the ServiceLocator API directly. See the section on [TCP and UDP service lookups](TCP-Lookups.html) for more information on that.