# Licensing

It is required to load a license into your ConductR cluster if you want to be able to use more than one ConductR agent. 
To license the cluster you must use the ConductR CLI `conduct load-license` command.

The license will only needed to be loaded once for as long as the cluster is up, or until the license expires.  

## Loading the License

### Online Mode

The standard workflow assumes that the workstation you are running the CLI on has connectivity to the internet.  To load
the license 

1. Execute `conduct load-license` at the command line.
2. Retrieve your Enterprise Suite access token from https://www.lightbend.com/platform/enterprise-suite/access-token
3. Enter your access token when prompted

```
$ conduct load-license

An access token is required. Please visit https://www.lightbend.com/platform/enterprise-suite/access-token to 
obtain one for free or for your commercial licenses.

Please enter your access token: XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
Loading license into ConductR at 192.168.10.1

Licensed To: aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa
Expires In: 365 days (Fri 27 Jul 2018 16:09PM)
Grants: akka-sbr, cinnamon, conductr, fortify

License successfully loaded
```

If successful you will see the following
- A message indicating the license was added to the ConductR cluster.
- A summary of license information, including your user token.  This will match your bintray credentials in your 
`~/.lightbend/commercial.properties` file.
- A date when the license expires and you will need to execute `load-license` again.
- Components of Enterprise Suite that your license is authorized for.

### Offline Mode

In privileged environments, loading the license is straightforward.  In environments that cannot access the internet you 
must install the license in offline mode (i.e. from your CLI without having the HTTP proxy setup, or if you’re 
installing the CLI on a server with no internet access).

Steps to install the license in offline mode.

1. Login to your lightbend.com account with your licensed email and setup your `~/.lightbend/commercial.credentials` 
as described on [lightbend.com](https://www.lightbend.com/product/conductr/developer).  While setting up your 
`commercial.credentials` it’s important to use the email associated with your organization's license, and not a personal
account.  If enterprise environments this is usually your corporate email.
2. Download your `license` file manually.  You can find your license by visiting 
https://www.lightbend.com/product/conductr/license
3. Copy & paste the contents of this page into your `~/.lightbend/license` file.
4. Execute `conduct load-license --offline` at the command line.

```
$ conduct load-license --offline
Skipping downloading license from Lightbend.com
Loading license into ConductR at 192.168.10.1

Licensed To: aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa
Expires In: 365 days (Fri 27 Jul 2018 16:09PM)
Grants: akka-sbr, cinnamon, conductr, fortify

License successfully loaded
```

### Viewing the current license ConductR cluster

When you issue `conduct info` you will see the same license details that were returned when the license was first
loaded.

### Updating your License

If you need to update the license on an already licensed cluster you can do so using the `--force` command switch. This
is useful when you want to change the license to a different user or when your license is close to expiring.  Enter
`conduct load-license --offline` and the command line and then follow the prompts to load a new license.

## Automate Licensing at Startup

When a license is loaded it is submitted to the ConductR cluster and distributed to all the ConductR core nodes.  The
license resides in memory, so in the event of a complete cluster shutdown the data is lost.  In production this would
be an unlikely scenario as your cluster should be able to tolerate the failure of some nodes.

In non-production environments it may be commonplace to shutdown whole clusters for a number of reasons: cluster is no
longer required, to save on public cloud instance charges, etc.

In any environment, whenever a cluster starts up it is important to license before bundles are loaded and run again.
By retrieving your license manually as described in "Offline Mode", you can automate the licensing of a cluster whenever
it's first started up.

Remember that licenses expire, so set a reminder to occassionally retrieve an updated `./.conductr/license` file for 
your deployment script.

## Other ConductR Licensing Information and Caveats

- Anyone using the CLI to work with a licensed cluster will be using the same license.
- Bundles, like licenses, are lost in the event of a total cluster shutdown.  You should plan for this contingency 
when starting your cluster in the same manner you plan to license it.  You should load all required support bundles
(`visualizer`, `haproxy`, `continuous-delivery`, etc.) as well as your latest running app bundles.
- While there is at least one node of the cluster up, the license will be intact.
- Future versions of ConductR will persist the license to disk in the cluster.
