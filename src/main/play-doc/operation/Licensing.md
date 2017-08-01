# Licensing

It is required to load a license into your ConductR cluster if you want to be able to use more than one ConductR agent. To license the cluster you must use the ConductR CLI `conduct load-license` command.

The license will only need to be loaded once for as long as the cluster is up, or until the license expires.  Anyone using the CLI to work with an already licensed cluster will be use the same license. 

## Loading the License

### Online Mode

The standard workflow assumes that the workstation you are running the ConductR CLI on has connectivity to the internet.

Steps to load the license.

1. Execute `conduct load-license` at the command line.
2. Retrieve your Enterprise Suite access token from https://www.lightbend.com/platform/enterprise-suite/access-token
3. Enter your access token when prompted

```
$ conduct load-license

An access token is required. Please visit https://www.lightbend.com/platform/enterprise-suite/access-token to obtain one for free or for your commercial licenses.

Please enter your access token: XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
Loading license into ConductR at 192.168.10.1

Licensed To: aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa
Expires In: 365 days (Fri 27 Jul 2018 16:09PM)
Grants: akka-sbr, cinnamon, conductr, fortify

License successfully loaded
```

If successful you will see the following.

- A message indicating the license was added to the ConductR cluster.
- A summary of license information, including your user token.  This will match your bintray credentials in your `~/.lightbend/commercial.properties` file.
- A date when the license expires and you will need to execute `load-license` again.
- Components of Enterprise Suite that your license is authorized for.

### Offline Mode

In privileged environments, loading the license is straightforward.  In environments that cannot access the internet you must install the license in offline mode (i.e. from your CLI without having an HTTP proxy setup, or if you’re installing the CLI on a server with no internet access).

Steps to install the license in offline mode.

1. Login to your lightbend.com account with your licensed email and setup your `~/.lightbend/commercial.credentials` as described on [lightbend.com](https://www.lightbend.com/product/conductr/developer).  While setting up your `commercial.credentials` it’s important to use the email associated with your organization's license, and not a personal account.  In enterprise environments your license would usually be associated with your corporate email.
2. Retrieve your license information from https://www.lightbend.com/product/conductr/license
3. Copy & paste the license into your `~/.lightbend/license` file.
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

When you issue `conduct info` you will see the same license details that were returned when the license was first loaded.

### Updating your License

If you need to update the license on an already licensed cluster you can do so using the `--force` command switch. This is useful when you want to change the license to a different user or when your license is close to expiring.  Enter `conduct load-license --force` and the command line and then follow the prompts to load a new license.

## Automate Licensing at Startup

When a license is loaded it is submitted to the ConductR cluster and distributed to all the ConductR core nodes.  The license resides in memory, so in the event of a complete cluster shutdown the data is lost.  In production this would be an unlikely scenario as your cluster should be able to tolerate the failure of some nodes.  As long as one node in your cluster remains online your license will be intact.

In non-production environments it may be commonplace to shutdown whole clusters for a number of reasons: the cluster is no longer required, to save on public cloud instance charges, etc.

When a cluster starts up in any environment it is important to license it before app bundles are loaded and run again. By retrieving your license manually as described in "Offline Mode", you can automate the licensing of a cluster whenever it's first started up.

Remember that licenses expire, so set a reminder to occassionally retrieve an updated `./.conductr/license` file for your deployment script.  An existing ConductR cluster and its running bundles will continue to run when the license expires.  New agents, or agents that try to rejoin after a failure, will not be able to join the cluster.

In future versions of ConductR the license may be persisted to disk.
