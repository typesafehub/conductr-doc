# Deploying application

ConductR provides REST API which allows you as the operator to:

* query the information on loaded bundles and running services
* manage the lifecycle of bundles (load, run, stop, unload)

The API can be used by any HTTP client, but ConductR comes with CLI tool implemented in Python. To get the latest version of the CLI install it locally as a pip package.

## CLI installation

Firstly `pip3` is required:

```bash
sudo apt-get install python3-setuptools 
sudo easy_install3 -U pip
```

...then:

```bash
pip3 install --user conductr-cli
```

`pip3` will install the CLI in your home folder under the .local folder (at least on Ubuntu). Ensure that your path captures this:

```bash
# Local bin files
PATH=$HOME/.local/bin:$PATH
```

To verify the installation type:

```bash
conduct -h
```

...you should then see output similar to the following:

```bash
usage: conduct [-h] {version,info,services,load,run,stop,unload} ...

optional arguments:
  -h, --help            show this help message and exit

subcommands:
  valid subcommands                                                                                       depl

  {version,info,services,load,run,stop,unload}
                        help for subcommands
    version             print version
    info                print bundle information
    services            print service information
    load                load a bundle
    run                 run a bundle
    stop                stop a abundle
    unload              unload a bundle
```

## Upgrading the CLI

```bash
pip3 install --user --upgrade conductr-cli
```

## Packaging configuration

In addition to consuming services provided by ConductR, the CLI also provides a quick way of packaging custom configuration to a bundle. We will go through most of the CLI features by deploying the Visualizer bundle to ConductR that comes together with the ConductR installation. The Visualizer can be found in the `/usr/share/conductr/samples` directory.

Visualizer is a sample Play Framework application that queries ConductR API endpoints and visualizes the ConductR cluster together with deployed and running bundles.

[[images/visualizer.png]]

Some applications require additional configuration when deployed to different environments. Visualizer allows setting `POLL_INTERVAL` environment variable which controls how quickly ConductR is polled after receiving events that state has changed.

A strong feature of ConductR is that configuration may be coupled with a bundle at the time of loading. Thus a bundle can be associated with different configurations; perhaps one configuration for a test environment, and another for production. Furthermore, different versions of a bundle may be associated with the same configuration. Whatever the combination is, the bundle and its configuration that is loaded may always be distinguished from others given that unique identifiers are returned for them.

Configuration files are executed just before a bundle starts. Let's create a configuration file that exports a `POLL_INTERVAL` environment variable containing a shorter than default poll interval (which is 100ms):

```bash
cd #back to your home folder so we know where we're playing
echo "export POLL_INTERVAL=500ms" >> ./visualizer-poll-interval.sh
```

Once the configuration file is ready, we need to package it up to a bundle. We can use the `shazar` command provided by the CLI which zips configuration files and appends secure digest of the archive to the created bundle files.

```bash
shazar ./visualizer-poll-interval.sh
```

The configuration is now ready to be loaded along with a bundle. Configuration is always provided as the second param to the `conduct load` command.

## Deploying bundles

The standard bundle lifecycle in the ConductR is:

1. Load. Bundle is loaded into a ConductR cluster and replicated among cluster nodes.
2. Run. Bundle is started by running one or more instances of all defined bundle components.
3. Stop. Bundle is stopped by stopping all instances of all defined bundle components.
4. Unload. Bundle is unloaded from a ConductR cluster and all bundle replicas are removed from cluster nodes.

We use `conduct` command provided by the CLI to load a Visualizer bundle together with configuration to the ConductR and then run it. Every `conduct` command that is communicating with a ConductR needs to be given `--ip` and `--port` parameters (these can be omitted if ConductR is running on the same machine and on the default port). Alternatively CLI can use `CONDUCTR_IP` and `CONDUCTR_PORT` environment variables for corresponding parameters.

This data is used by the ConductR when making bundle scheduling decisions. Load the Visualizer bundle by executing:

```bash
conduct load --ip 172.17.0.1 \
             /usr/share/conductr/samples/visualizer-...zip \
             ./visualizer-poll-interval.sh-...zip
```

Note that by default, bundles have a maximum size of 100MB. This can be altered via the `akka.http.server.parsing.max-content-length` setting.

Use `conduct info` command to list all loaded bundles. You should see Visualizer replicated but not running (note that the example below shows 3 replications - you'll only get that if you have 3 or more nodes as the bundle cannot replicate beyond the cluster size).

```bash
conduct info --ip 172.17.0.1
```
...will yield something like:

```bash
ID               NAME              #REP  #STR  #RUN
23391d4-3cc322b  visualizer-0.1.0  3     0     0
```

Run Visualizer by executing:

```bash
conduct run --ip 172.17.0.1 visualizer
```

Whenever you need to refer to a bundle you can use a prefix of or a full bundle id/name. Run `conduct info` once again, to see that the bundle has been successfully started.

``` bash
ID               NAME              #REP  #STR  #RUN
23391d4-3cc322b  visualizer-0.1.0  3     0     1
```

## Accessing services provided by bundles

Access to services in ConductR is proxied for high availability and load balancing. To list all the currently running services in the ConductR execute the `conduct services` command.

```bash
conduct services --ip 172.17.0.1
```

You should see that there is one service called `9000/visualizer` provided by the Visualizer bundle.

```bash
SERVICE       BUNDLE ID       BUNDLE NAME       STATUS
http://:9999  23391d4-3cc322b visualizer-0.1.0  Running
```

To access Visualizer point your browser to any ConductR node and add the name of the service to the URL, e.g. `http://172.17.0.1:9999`. Alternatively, if you can only access ConductR nodes using SSH, create a SSH tunnel that tunnels local port from your machine to the Visualizer service `ssh -L 8080:172.17.0.1:9999 172.17.0.1` (don't forget to substitute the `172.17.0.1`) and then access Visualizer by pointing your browser to `http://localhost:9999`.

Visualizer shows ConductR nodes as small blue circles with IP addresses next to them. Green circles denote bundles, and spinning circle means that a bundle is running. You should see one instance of bundle running. Try starting Visualizer on more nodes by executing:

```bash
conduct run --ip 172.17.0.1 --scale 2 visualizer
```

You should see another green circle start spinning, which means that another instance of Visualizer was started. Play around with more `conduct` commands and see how it affects ConductR cluster visualization.

Our aim is to make using Typesafe ConductR by operators akin to using Play by developers; a joyful and productive experience! ConductR starts to shine when used in the context of managing more than 2 nodes; a common scenario for reactive applications. Go and spin those nodes up!

