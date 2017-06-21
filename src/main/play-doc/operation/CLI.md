# Command line interface (CLI)

ConductR provides a REST API which allows you as the operator to:

* query the information on loaded bundles and running services
* manage the lifecycle of bundles (load, run, stop, unload)

The API can be used by any HTTP client, but ConductR comes with CLI tool implemented in Python. To get the latest version of the CLI install it locally as a pip package.

## Installation

The CLI is distributed as a "native" distribution for Windows, OS X and Linux. Lightbend hosts these native images at Bintray: https://bintray.com/lightbend/generic/conductr-cli. Download an archive that is suitable for your environment and then place the resultant package in a place accessible from your `PATH`. For example, on Unix, please copy the contents of archive to your `/usr/local/bin` folder.

To verify the installation type:

```bash
conduct -h
```

...you should then see output similar to the following:

```bash
conduct -h
usage: conduct [-h]
               {version,info,service-names,acls,load,run,start,stop,unload,events,logs,setup-dcos,deploy,members,agents,load-license}
               ...

optional arguments:
  -h, --help            show this help message and exit

commands:
...
```

## DC/OS

The CLI is able to integrate with the DC/OS CLI e.g. `dcos conduct info` will render the current ConductR state on DC/OS. To setup CLI integration type `conduct setup-dcos`.

## Packaging configuration

In addition to consuming services provided by ConductR, the CLI also provides a quick way of packaging custom configuration to a bundle. We will go through most of the CLI features by deploying the Visualizer bundle to ConductR that comes together with the ConductR installation. The Visualizer can be resolved and loaded from the [bundles repo](https://bintray.com/typesafe/bundle) using the `load` command.

```bash
conduct load visualizer
```

Visualizer is a sample Play Framework application that queries ConductR API endpoints and visualizes the ConductR cluster together with deployed and running bundles.

[[images/visualizer.png]]

Some applications require additional configuration when deployed to different environments. Visualizer allows setting `POLL_INTERVAL` environment variable which controls how quickly ConductR is polled after receiving events that state has changed.

A strong feature of ConductR is that configuration may be coupled with a bundle at the time of loading. Thus a bundle can be associated with different configurations; perhaps one configuration for a test environment, and another for production. Furthermore, different versions of a bundle may be associated with the same configuration. Whatever the combination is, the bundle and its configuration that is loaded may always be distinguished from others given that unique identifiers are returned for them.

Let's load the visualizer but specify the `POLL_INTERVAL` environment variable.

```bash
conduct load visualizer --env POLL_INTERVAL=500ms
```

This created a *configuration bundle* for you that contains a `runtime-config.sh` file and loaded it alongside the visualizer. This file will be executed by ConductR immediately before it starts your bundle.

> You could also create a `runtime-config.sh` yourself if you needed to implement more advanced logic. For more information on this and other configuration options, see "[Bundle Configuration](../developer/BundleConfiguration)".

## Accessing the Visualizer

Access to services in ConductR is proxied for high availability and load balancing. To access Visualizer, and supposing that ConductR is running at `172.17.0.1`, use the URL, `http://172.17.0.1:9999`. Alternatively, if you can only access ConductR nodes using SSH, create a SSH tunnel that tunnels local port from your machine to the Visualizer service `ssh -L 9999:172.17.0.1:9999 172.17.0.1` and then access Visualizer by pointing your browser to `http://localhost:9999`.

Visualizer shows ConductR nodes as small blue circles with IP addresses next to them. Green circles denote bundles, and spinning circle means that a bundle is running. You should see one instance of bundle running. Try starting Visualizer on more nodes by executing:

```bash
conduct run --host 172.17.0.1 --scale 2 visualizer
```

You should see another green circle start spinning, which means that another instance of Visualizer was started. Play around with more `conduct` commands and see how it affects ConductR cluster visualization.

Our aim is to make using Lightbend ConductR by operators akin to using Play by developers; a joyful and productive experience! ConductR starts to shine when used in the context of managing more than 2 nodes; a common scenario for reactive applications. Go and spin those nodes up!

## HTTP Basic Authentication

> Note: From a cluster setup perspective, HTTP Basic Authentication for the ConductR CLI can be set up and enforced as described in the section [Securing the ConductR CLI with Basic Authentication](DynamicProxyConfiguration#Securing-the-ConductR-CLI-with-Basic-Authentication).

To enable HTTP Basic Authentication on the ConductR CLI, provide the following settings file in the `~/.conductr/settings.conf`.

```
conductr {
  auth {
    enabled  = true
    username = "steve"
    password = "stevespassword"
  }
  server_ssl_verification_file = "/home/user/validate-server.pem"
}
```

HTTP Basic Authentication is enabled if the flag `enabled` is set to `true`. Setting it to `false` is disabling basic auth.
Set the `username` and `password` accordingly. The `server_ssl_verification_file` points to an absolute path of the file used to validate the SSL cert of the server.

If HTTP Basic Authentication is enabled then the CLI will send HTTP requests using HTTPS instead of HTTP.

The `~/.conductr/settings.conf` file may also specify configurations for multiple clusters as shown below:

```
conductr {
  auth {
    "192.168.99.100" {
      enabled  = true
      username = "steve"
      password = "stevespassword"
      server_ssl_verification_file = "/home/user/validate-server.pem"
    }
    "192.168.99.200" {
      enabled  = false
    }
  }
}
```

In the above case, authentication credentials will be used when the ConductR CLI interacts with the cluster located at `192.168.99.100`, while the ConductR Core located at `192.168.99.200` will not require authentication.

With the above setting in place, the ConductR CLI can be used as normally:

```
conduct info --host 192.168.99.100
```

If valid credentials are not found in ~/.conductr/settings.conf, the ConductR CLI will return a 401 Unauthorized error.

## ConductR CLI on alternative ports

If the ConductR Control Protocol is configurted to listen for requests on a port other than `9005`, the ConductR CLI can still be used by specifying the `--port` (-p) argument. For example, if we wanted to use the ConductR CLI within the cluster setup in the example described in [Securing the ConductR CLI with Basic Authentication](DynamicProxyConfiguration#Securing-the-ConductR-CLI-with-Basic-Authentication), we could access the Control Protocol directly using port `9055`:

```
conduct info --host 172.17.0.1 -p 9055
```
