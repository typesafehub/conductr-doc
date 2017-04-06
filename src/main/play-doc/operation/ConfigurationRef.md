# Configuration Reference

The configuration of ConductR Core and Agent depends on the ConductR mode. Choose one of the following sections depending on the mode:

* [Standalone mode](#Standalone-mode)
* [DC/OS mode](#DC/OS-mode)

## Standalone mode

### ConductR Core

Each one of the following properties can be overridden by declaring its path and value in ConductR Core's `conf/conductr.ini`. For example to set ConductR Core's storage directory you'd specify `-Dconductr.storage-dir=/some-other-dir`.

### ConductR Core Configuration

Here is the entire configuration for ConductR Core when running in Standalone mode.

```
akka {
  log-dead-letters                 = off
  log-dead-letters-during-shutdown = off
  loggers                          = [akka.event.slf4j.Slf4jLogger, com.typesafe.contrail.adapter.syslog.akka.SyslogLogger]
  logging-filter                   = "akka.event.slf4j.Slf4jLoggingFilter"
  loglevel                         = info

  actor {
    provider = akka.cluster.ClusterActorRefProvider

    debug {
      lifecycle = off
      unhandled = on
    }

    enable-additional-serialization-bindings = on

    serializers {
      conductr-core-proto  = "com.typesafe.conductr.CoreProtobufSerializer"
      conductr-agent-proto = "com.typesafe.conductr.AgentProtobufSerializer"
    }

    serialization-bindings {
      "java.io.Serializable"                     = none
      "java.lang.String"                         = java
      "akka.dispatch.sysmsg.Watch"               = java

      "com.typesafe.conductr.CoreRemotableMessage"  = conductr-core-proto
      "com.typesafe.conductr.AgentRemotableMessage" = conductr-agent-proto
    }

    // We use a round-robin-pool actor here given that the default
    // consistent hashing one depends on akka remoting - which
    // may not have started up in time for contrail. Alternatively
    // we could supply our own consistent hashing router for this
    // use-case that did not depend on akka remoting. Something to
    // consider if contrail ever saw the light of day in application
    // code outside of ConductR.
    deployment {
      /IO-DNS/inet-address {
        mailbox = "unbounded"
        router = "round-robin-pool"
        nr-of-instances = 4
      }
    }
  }

  cluster {
    # INTERNAL: Do not overwrite this seed nodes parameter.
    # Instead, either use ConductR's --seed option or declare the `conductr.seed-nodes` parameter in your application.conf
    # For more information check out the `conductr.seed-nodes` documentation
    seed-nodes                  = []

    # The `replicator` role is required for bundle replication purposes, hence core nodes which hosts the bundle
    # files must have this role defined.
    # By default all core nodes act as host for bundle files, hence all core nodes will have the `replicator` role.
    roles                       = ["replicator"]

    downing-provider-class = "com.lightbend.akka.sbr.SplitBrainResolverProvider"
    split-brain-resolver {
      # Should cluster split occurs, keep the partition having majority of remaining nodes
      active-strategy = keep-majority
    }

    failure-detector {
      # The recommended value for slower networks including cloud environments: http://doc.akka.io/docs/akka/snapshot/scala/cluster-usage.html
      threshold = 12
    }
  }

  extensions = [
    "akka.cluster.client.ClusterClientReceptionist",
    "akka.cluster.ddata.DistributedData"
  ]

  http {
    parsing.max-content-length   = 200m
    server.remote-address-header = on
  }

  # We fallback on DNS SRV records if we cannot resolve services within ConductR
  io.dns {
    resolver = async-dns
    async-dns {
      resolve-srv = true
      resolv-conf = on
    }
  }

  remote {
    enabled-transports          = [akka.remote.netty.tcp]
    log-remote-lifecycle-events = off

    netty.tcp {
      hostname = ${conductr.ip}
      port     = 9004
    }
  }

  diagnostics.checker {
    confirmed-power-user-settings = [
      "akka.cluster.failure-detector.threshold",
      "akka.io.dns.resolver",
      "akka.io.dns.async-dns.resolv-conf",
      "akka.io.dns.async-dns.resolve-srv"
    ]
  }
}

# Lightbend ConductR configuration.
conductr {
  # The name of the ConductR cluster, i.e. the Akka system name
  system-name = "conductr"

  # The IP address used by default for all services, e.g. control server, bundle stream server, etc.
  # Defaults to the loopback IP address and can be set via the CONDUCTR_IP environment variable.
  ip = "127.0.0.1"
  ip = ${?CONDUCTR_IP}

  # The mode in which ConductR is started.
  # Modes: 'standalone', 'mesos'
  # By default 'standalone' is used.
  # If ConductR should be started on Mesos then it is recommended to specify the '--mesos-master' option when starting
  # the ConductR process. The '--mesos-master' option will set the mode to 'mesos' and
  # the 'conductr.mesos-scheduler-client.master' to the given address
  mode = "standalone"

  # The seed nodes or constructr coordination service addresses to bootstrap the ConductR cluster.
  # The ConductR option --seed is available to add entries to `conductr-seed-nodes`.
  # Depending on the given addresses the ConductR cluster is bootstrapped differently:
  # 1. If empty, the ConductR node joins the cluster with itself (default)
  # 2. If the given addresses does not contain a Zookeeper address, i.e. does not start with zk://, then these addresses
  #    are interpretes as seed nodes. The ConductR node will join another ConductR cluster by given seed nodes.
  # 3. If at least one of the given addresses is a Zookeeper address, i.e. address starts with zk://, then the other addresses
  #    are filtered out and only the Zookeeper addresses are kept. These Zookeeper addresses are then interpreted
  #    as ConstructR coordination service nodes. ConstructR is then started and therefore responsible to bootstrap the ConductR cluster.
  seed-nodes = []

  # The directory where bundles and their configuration are written to.
  storage-dir = ${java.io.tmpdir}/conductr/${conductr.ip}/bundles

  # The amount of time Lightbend ConductR expects to wait on achieving a read quorum. 10 seconds should
  # cover large clusters, but you may need to increase this if read timeouts appear
  # frequently in your log files.
  read-quorum-timeout = 10 seconds

  # As per read-quorum-timeout.
  write-quorum-timeout = 10 seconds

  # The expected maximum amount of time it takes for the actor system to shutdown.
  # If the actor system fails to shutdown within this period, an error message will be printed.
  #
  # ConductR will attempt to restart its actor system when encountering error. If the shutdown timeout occurs in this
  # scenario, System.exit() with non-zero exit code will be called.
  #
  # If the shutdown timeout occurs as part of the JVM shutdown hook, System.exit() will not be called as invoking
  # System.exit() within a shutdown hook thread will cause the call to block indefinitely:
  # https://docs.oracle.com/javase/8/docs/api/java/lang/Runtime.html#exit-int-
  actor-system-shutdown-timeout = 60 seconds

  # Bundle Publisher
  bundle-publisher {
    # The time that a bundle can pending to be served for.
    request-ttl = 2 minutes
  }

  # The agent monitor keeps track of agent membership state.
  agent-monitor {
    # This is a parameter used for consistent hashing when determining what ConductR is responsible for
    # a given agent identifier. The parameter should not need to be changed.
    virtual-nodes-factor = 10

    # The time it takes for the monitor to assume that it is unlikely for an agent to come back
    # and therefore assume that it has no associated executions. Note that an agent may still
    # come back after this period.
    become-down-timeout = 15 seconds

    # The period of time to wait before we assume that an agent will never come back and we should
    # thus clean up any remaining state in relation to it.
    down-idle-timeout = 10 minutes

    # The amount of time we'll tolerate waiting for the agent information event
    agent-info-timeout = 20 seconds

    # The amount of time to wait until the bundle execution state is considered as settled.
    bundle-execution-state-settling-time = 5 seconds

    # The amount of time to wait to allow for the removal of agent state from core before reconciling
    # execution state with an agent again.
    agent-removal-settling-time = 5 seconds

    # The name of the launcher on the remote agent
    launcher-agent-name = "standalone-launcher-agent"
  }

  # Bundling streaming is an out-of-band private channel used between the ConductRs in
  # order to stream bundles and associated configuration.
  bundle-stream-server {
    # The protocol, IP address and port on which to serve bundling streaming server requests.
    protocol = http
    ip       = ${conductr.ip}
    port     = 9006

    # The amount of time to wait on attempting to bind.
    bind-timeout = 5 seconds

    # The amount of time to wait on one ConductR connecting to another's bundle stream
    # server.
    connect-timeout = 2 minutes

    # Bundles and configuration are transiently served i.e. once served then they are no
    # longer available. This parameter dictates the maximum number of bundles that are ready to
    # be served at one time.
    max-nr-of-pending-requests = 500

    # The time it takes to obtain request information within the handling of an http request
    # for streaming data.
    request-fetch-timeout = 5 seconds

    # The time that a bundle can pending to be served for.
    request-ttl = 2 minutes
  }

  # The control server provides http services in relation to controlling the ConductR e.g.
  # loading bundles, starting bundles, obtaining cluster state etc.
  control-server {
    # The protocol, IP address and port on which to serve control protocol requests.
    protocol = http
    ip       = ${conductr.ip}
    port     = 9005

    # The amount of time to wait on attempting to bind.
    bind-timeout = 5 seconds

    # The amount of time to wait on publishing a bundle.
    bundle-publisher-timeout = 20 seconds

    # The amount of time to wait on obtaining agent events publisher.
    agent-events-timeout = 5 seconds

    # The amount of time to wait on obtaining state events publisher.
    state-events-timeout = 5 seconds

    # The amount of time to wait on uploading a bundle into Lightbend ConductR. This timeout will vary depending
    # on how long it takes for clients of Lightbend ConductR to upload bundles i.e. the WAN will dictate this
    # parameter. If you find that clients e.g. the CLI, are reporting timeouts then try increasing
    # this parameter.
    load-scheduler-timeout = 1 minute

    # The amount of time to wait on requesting that an unload occurs. This should generally be fast
    # as it is only a request to be scheduled.
    unload-scheduler-timeout = 5 seconds

    # The amount of time to wait on requesting that a bundle be started.
    start-scheduler-timeout = 5 seconds

    # The amount of time to wait on performing cluster management operations e.g. join a cluster,
    # drop a member from the cluster etc.
    cluster-management-timeout = 5 seconds

    # The amount of time to wait when querying for logs or events for a particular bundle.
    log-repository-lookup-timeout = 5 seconds

    # The default number of events or logs fetched for each query if not specified.
    log-repository-lookup-default-size = 10

    # The maximum number of events or logs that can be fetched for each query.
    log-repository-lookup-max-size = 100

    # The service name of the deployment service for pipeline delivery
    deployment-service-name = "deployments"

    # The amount of time to wait for a service locator lookup in the context of proxying
    # control protocol requests.
    service-locator-timeout = 5 seconds

    # The service name of the web UI to serve from the control protocol root context.
    web-ui-service-name = "visualizer"

    # The maximum size of a bundle configuration given that it is read into memory
    max-bundle-conf-size = 8k

    sse {
      state {
        # The size of state event publisher buffer.
        publisher-buffer-size = 100

        # The interval of the heartbeat to keep the connection alive.
        heartbeat-interval = 1 second
      }

      cluster {
        # The size of cluster event publisher buffer.
        publisher-buffer-size = 100

        # The interval of the heartbeat to keep the connection alive.
        heartbeat-interval = 1 second
      }
    }
  }

  core-services {
    # The waiting time to retrieve the core services information
    retrieve-timeout = 5 seconds
  }

  # Loading is associated with the loading, replication and unloading of a bundle and its associated configuration.
  load {
    # When replicating, this parameter dictates the amount of time in between waiting on the convergence
    # of the data it needs in order to make decisions around replication. These decisions amount to
    # how many replicas of a bundle are to be made, or how many may be removed.
    bundle-replicator-converge-retry-delay = 5 seconds

    # The time that a bundle replicator for a given bundle id will exist before it stops
    # given no activity. Replicators remain active only when there is work to be done.
    bundle-replicator-idle-time = 2 minutes

    # The amount of time that a ConductR is expected to wait on another ConductR in order to replicate
    # a bundle. This parameter will be influenced by the speed on the bundle stream network between ConductRs.
    # If you find that clients e.g. the CLI, are reporting timeouts then try increasing this parameter.
    bundle-retrieve-timeout = 5 minutes

    # The amount of time required to source a node bundle file on disk at the current node.
    load-scheduler-source-timeout = 5 seconds

    # The amount of delay put in place before replying after a successful bundle load.
    # This delay is introduced to allow subscribers to be notified of CRDT changes before sending the reply, thus
    # reducing the chance of client seeing inconsistent information between subscribers (i.e. between LoadScheduler vs
    # ScaleScheduler)
    load-scheduler-reply-delay = ${akka.cluster.distributed-data.notify-subscribers-interval}

    # The amount of time LoadExecutor will wait for bundle info to be updated to all Core nodes.
    load-executor-bundle-info-update-timeout = ${conductr.write-quorum-timeout}

    # The number of bundles and configuration to retain on the file system. There is one bundle and config
    # stored per member up to a nr-of-replicas maximum. When a bundle starts executing, another replication
    # will occur. The goal is to ensure that nr-of-replicas represents the number of non-executing bundles
    # and config in the cluster. Having bundles and config made available in this way ensures that they can
    # be started quickly.
    nr-of-replicas = 3

    # This is a parameter used for consistent hashing when determining what ConductR is responsible for
    # a given bundle identifier. The parameter should not need to be changed.
    virtual-nodes-factor = 10

    # The amount of time to wait for changes in cluster membership before reacting i.e. re-determining
    # bundle replication.
    membership-settling-time = 5 seconds
  }

  # The resource provider receives requests for resources and provides resource offers.
  resource-provider {
    # The maximum number of resource requests that may be outstanding across the cluster at any one time.
    max-nr-of-pending-requests = 50

    # The amount of time to wait for a resource offer to be matched before being discarded.
    resource-offer-timeout = 20 seconds

    # When set, cluster roles are checked in terms of offers being made and matched. This should be off
    # if you are not concerned about what bundles can run where. For production purposes though, we recommend you
    # consider a topology where certain bundles run on certain machines e.g. when considering DMZs, databases
    # nodes etc.
    match-offer-roles = on
  }

  # Running is associated with the starting, scaling and stopping of a bundle within the cluster.
  run {
    # The maximum number of tries for starting a bundle.
    bundle-scaler-max-nr-of-tries = 3

    # Timeout values for the various states of a bundle scaler.
    bundle-scaler-idle-timeout      = 2 minutes
    bundle-scaler-scaling-timeout   = 10 seconds
    bundle-scaler-launching-timeout = 2 minutes
    bundle-scaler-starting-timeout  = 10 minutes

    # Timeout value for an idle system scaler.
    system-scaler-idle-timeout = 2 minutes

    # Timeout value for getting an acknowledge from a bundle scaler.
    system-scaler-acknowledge-timeout = 5 seconds

    # The maximum number of tries for getting an acknowledge from a bundle scaler.
    system-scaler-acknowledge-max-nr-of-tries = 3

    # This is a parameter used for consistent hashing when determining what ConductR is responsible for
    # a given bundle identifier. The parameter should not need to be changed.
    virtual-nodes-factor = 10

    # The amount of time that a ConductR is expected to wait on another ConductR in order to replicate
    # a bundle. This parameter will be influenced by the speed on the bundle stream network between ConductRs.
    # If you find that clients e.g. the CLI, are reporting timeouts then try increasing this parameter.
    bundle-retrieve-timeout = 5 minutes

    # The amount of time to wait for changes in cluster and agent membership before reacting i.e. re-determining
    # bundle scale.
    membership-settling-time = 5 seconds

  }

  service-locator-server {
    # The protocol, IP address and port on which to serve service locator requests.
    protocol = http
    ip       = ${conductr.ip}
    port     = 9008

    # The max time it should take for a service lookup to occur internally.
    locate-timeout = 5 seconds

    # Whether to fallback to DNS SRV lookups if local locator lookups fail
    dns-srv-fallback = off

    cache {
      # The max age a service name lookup should be cached for by a client.
      max-age        = 1 minute

      # The maximum number of service location responses to cache
      max-responses  = 100
    }

    sse {
      service {
        # The size of state event publisher buffer.
        publisher-buffer-size = 100

        # The interval of the heartbeat to keep the connection alive.
        heartbeat-interval = 1 second
      }
    }
  }

  service-proxy {
    # The name given to a service representing the proxy being used.
    name = "conductr-haproxy"
  }

  status-server {
    # The protocol, IP address and port on which to serve start-status server requests.
    protocol = http
    ip       = ${conductr.ip}
    port     = 9007

    # The duration to wait on attempting to bind
    bind-timeout = 5 seconds

    # The max time it should take for a status update to occur internally.
    status-timeout = 5 seconds
  }
}

# Lightbend Contrail configuration
contrail {

  syslog {

    loglevel = ${akka.loglevel}

    # Used as APP-NAME in syslog events.
    app-name = ConductR

    server {
      service-locator {
        enabled        = on
        selection-path = "/user/reaper/service-locator"
        service-name   = "elastic-search"
      }

      elasticsearch {
        enabled = on
        index = "conductr"
      }
    }
  }
}

# Lightbend Monitoring configuration
cinnamon {

  instrumentation = off

  akka.actors {
    "/user/*" {
      report-by = class
    }
  }
}

# ConstructR configuration

constructr {
  # Never give up on ZK disappearing - the consequences of which would be that ConstructR
  # shuts down the actor system if retries are exceeded. We therefore set the # of retries
  # to the max positive value of a 32 bit signed int as that is how it is represented
  # internally.
  nr-of-retries = 2147483647
}
```

### ConductR Agent

Each one of the following properties can be overridden by declaring its path and value in ConductR Agent's `conf/conductr-agent.ini`. For example to set ConductR Agent's storage directory you'd specify `-Dconductr.agent.storage-dir=/some-other-dir`.

### ConductR Agent Configuration

Here is the entire configuration for ConductR Agent when running in Standalone mode.

```
akka {
  log-dead-letters                 = off
  log-dead-letters-during-shutdown = off
  loggers                          = [akka.event.slf4j.Slf4jLogger, com.typesafe.contrail.adapter.syslog.akka.SyslogLogger]
  logging-filter                   = "akka.event.slf4j.Slf4jLoggingFilter"
  loglevel                         = info

  actor {
    provider = akka.actor.LocalActorRefProvider

    enable-additional-serialization-bindings = on

    serializers = {
      conductr-agent-proto = "com.typesafe.conductr.AgentProtobufSerializer"
    }

    serialization-bindings {
      "java.io.Serializable"                     = none

      "com.typesafe.conductr.AgentRemotableMessage" = conductr-agent-proto
    }

    // We use a round-robin-pool actor here given that the default
    // consistent hashing one depends on akka remoting - which
    // may not have started up in time for contrail. Alternatively
    // we could supply our own consistent hashing router for this
    // use-case that did not depend on akka remoting. Something to
    // consider if contrail ever saw the light of day in application
    // code outside of ConductR.
    deployment {
      /IO-DNS/inet-address {
        mailbox = "unbounded"
        router = "round-robin-pool"
        nr-of-instances = 4
      }
    }
  }

  cluster.client {
    # ConductR Agent will try to reconnect to ConductR core upon disconnection for the period specified by this config.
    # Once the period is elapsed, the ConductR Agent will restart its actor system which will cause its bundles
    # to stop, and thus be consistent with ConductR core's view.
    reconnect-timeout = 30s
  }

  http {
    parsing.max-content-length = 200m
    server.remote-address-header = on
  }

  remote {
    enabled-transports = []
    netty.tcp {
      hostname = ${conductr.agent.ip}
    }
  }
}

conductr.agent {
  # The IP address used by default for all services, e.g. control server, bundle stream server, etc.
  # Defaults to the loopback IP address and can be set via the CONDUCTR_AGENT_IP environment variable.
  ip = "127.0.0.1"
  ip = ${?CONDUCTR_AGENT_IP}

  # The mode in which the ConductR agent is started.
  # Modes: 'standalone', 'mesos'
  # By default 'standalone' is used.
  mode = "standalone"

  # The directory where bundles and their configuration are written to.
  storage-dir = ${java.io.tmpdir}/conductr-agent/${conductr.agent.ip}/bundles

  # The directory where bundles record their pids for the purposes of being reaped if they're still
  # hanging around when ConductR restarts (perhaps due to ConductR having been rudely killed via
  # SIGKILL).
  bundle-pidfile-dir = ${conductr.agent.storage-dir}/pids

  # The address and port of the ConductR Core's `remote.netty.tcp.hostname` and `remote.netty.tcp.port` respectively.
  # This is used by the ConductR agent's Akka cluster client to form the initial contact so connection to
  # ConductR core can be established.
  core-nodes = []

  # The name of the ConductR core cluser, i.e. the Akka system name.
  # Corresponds to the ConductR core settings `conductr.system-name`.
  # This is used by the ConductR agent's Akka cluster client to form the initial contact so connection to
  # ConductR core can be established.
  core-system-name = "conductr"

  # The role of the ConductR agent.
  # The roles will be used to decide where bundles can execute, i.e. given a bundle is configured with the role
  # of "front-end", only agents that has "front-end" role can execute this bundle.
  roles = ["web"]

  # The expected maximum amount of time it takes for the actor system to shutdown.
  # If the actor system fails to shutdown within this period, an error message will be printed.
  #
  # ConductR will attempt to restart its actor system when encountering error. If the shutdown timeout occurs in this
  # scenario, System.exit() with non-zero exit code will be called.
  #
  # If the shutdown timeout occurs as part of the JVM shutdown hook, System.exit() will not be called as invoking
  # System.exit() within a shutdown hook thread will cause the call to block indefinitely:
  # https://docs.oracle.com/javase/8/docs/api/java/lang/Runtime.html#exit-int-
  actor-system-shutdown-timeout = 60 seconds

  agent-info {
    # The amount of time we'll tolerate waiting for a reply for bundle information before bailing out.
    launcher-agent-bundle-info-timeout = 5 seconds

    # The interval between each reconcilation event of agent info in order to ensure that the
    # agent state is in sync with core.
    reconcile-agent-info-interval = 15 seconds

    # The actor path of the resource provider in the ConductR core
    agent-monitor-remote-path = "/user/reaper/agent-monitor"
  }

  # Bundling streaming is an out-of-band private channel used between the ConductRs in
  # order to stream bundles and associated configuration.
  bundle-stream-server {
    # The amount of time to wait on one ConductR connecting to another's bundle stream
    # server.
    connect-timeout = 30 seconds
  }

  # Server which proxies ConductR services - control protocol, service locator, and status server.
  # The port of the services mirrors that of the ConductR, and the actual URL of these services will be obtained from
  # ConductR Core.
  core-services-proxy-server {
    protocol = http
    host = ${conductr.agent.ip}

    # The port where control protocol requests will be proxied to ConductR Core.
    control-server-port = 19005

    # The port where status server requests will be proxied to ConductR Core.
    status-server-port = 19007

    # The port where service locator requests will be proxied to ConductR Core.
    service-locator-server-port = 19008
  }

  resources {
    # The interval of polling for nr of cpus, memory, and diskspace
    poll-interval = 2 seconds

    # The actor path of the resource provider in the ConductR core
    resource-provider-remote-path = "/user/reaper/standalone-resource-provider-scale"
  }

  # Running is associated with the starting, scaling and stopping of a bundle within the cluster.
  run {
    # The start and end (inclusive) of the range for dynamically allocated ports.
    # It's a requirement that users of the ConductR must ensure that this port range is free.
    allocated-ports {
      start = 10000
      end   = 10999
    }

    # The command used to start Bash; this default should work on most systems.
    # Another sensible alternative would be ["/bin/bash"]
    bash-command = ["/usr/bin/env", "bash"]

    # When ConductR stops a bundle, it will attempt to shutdown the bundle's running children processes.
    # It does so by issuing a SIGTERM, and after a certain pause, it will issue a SIGKILL for each of the child process.
    # These configuration values control how long the pause between SIGTERM and SIGKILL mentioned above. "term-delay"
    # is the time between issuing a SIGTERM and then waiting before looking to see if it may be necessary to then do a
    # sleep of "term-kill-delay" which will then cause a SIGKILL. "term-delay" can be viewed as the normal amount of time
    # we allow processes to exit gracefully. Put another way, processes will have up to 8 seconds to exit gracefully,
    # but we expect most to exit within 2 seconds. The reason we have the two parameters is so that we can avoid the
    # longer delay in general. Note that we must stay well within 10 seconds as a Unix's system's restart scripts
    # are often about 10 seconds before they SIGKILL ConductR itself.
    term-delay      = 3 seconds
    term-kill-delay = 5 seconds

    # The amount of time to download a bundle from a ConductR core instance.
    # This parameter will be influenced by the speed on the bundle stream network between ConductR agent and core.
    # If you find that the ConductR agent is reporting timeouts then try increasing this parameter.
    bundle-retrieve-timeout = 5 minutes

    core-services-proxy {
      # The actor path of the actor that returns the ip address of the services provided in the ConductR core
      remote-path = "/user/reaper/core-services"

      request {
        # The amount of timeout waiting for reply from the ConductR core node when requesting for ip address of the
        # services
        timeout = 5 seconds

        backoff {
          min-backoff = ${conductr.agent.run.core-services-proxy.request.timeout}
          max-backoff = 30 seconds
          random-factor = 0.2
        }
      }
    }

    bundle-house-keeping {
      # Bundle house keeping will try to remove unused files from the bundles directory specified in
      # `conductr.agent.storage-dir`. Bundle house keeping will be invoked when a new bundle starts.
      #
      # Bundle house keeping will attempt to delete bundle run directories except the latest directories.
      # The number of the latest directories to be kept is controlled by `bundle-dirs-threshold`. This attempt will
      # remove old bundle run directories and its contents (i.e. jars and logs, etc).
      #
      # After cleaning up the bundle run directories, and if the size threshold is breached, bundle house
      # keeping will delete files older than specified `time-threshold`. This attempt will remove old bundle zip files.
      #
      # If the size threshold is still breached after this attempt, log warning message will be issued.

      # Number of bundle run directory to be kept within a particular bundle.
      # Important: never set this number below 1, otherwise the directory of currently running bundle will be deleted.
      bundle-run-dirs-threshold = 2

      size-threshold = 4GB
      time-threshold = 30 days

      # Time delay between bundle housekeeping being scheduled to when it's performed.
      start-delay = 5 seconds
    }
  }

  service-proxy {
    # The name given to a service representing the proxy being used.
    name = "conductr-haproxy"
  }
}

# Lightbend Contrail configuration
contrail {

  syslog {

    loglevel = ${akka.loglevel}

    # Used as APP-NAME in syslog events.
    app-name = "ConductR-Agent"

    server {
      service-locator {
        enabled        = on
        selection-path = "/user/reaper/syslog-service-locator-client"
        service-name   = "elastic-search"
      }

      elasticsearch {
        enabled = on
        index = "conductr"
      }
    }
  }
}

# Lightbend Monitoring configuration
cinnamon {

  instrumentation = off

  akka.actors {
    "/user/*" {
      report-by = class
    }
  }
}
```

## DC/OS mode

On DC/OS, the configuration of ConductR Core and Agent is performed via the DC/OS Services UI page. For more information how to specify custom configuration, check out [Cluster Configuration DC/OS mode](ClusterConfiguration#DC/OS-mode).

### ConductR Core Configuration

Here is the entire configuration for ConductR Core when running in DC/OS mode.

```
akka {
  log-dead-letters                 = off
  log-dead-letters-during-shutdown = off
  loggers                          = [akka.event.slf4j.Slf4jLogger, com.typesafe.contrail.adapter.syslog.akka.SyslogLogger]
  logging-filter                   = "akka.event.slf4j.Slf4jLoggingFilter"
  loglevel                         = info

  actor {
    provider = akka.cluster.ClusterActorRefProvider

    debug {
      lifecycle = off
      unhandled = on
    }

    enable-additional-serialization-bindings = on

    serializers {
      conductr-core-proto  = "com.typesafe.conductr.CoreProtobufSerializer"
      conductr-agent-proto = "com.typesafe.conductr.AgentProtobufSerializer"
    }

    serialization-bindings {
      "java.io.Serializable"                     = none
      "java.lang.String"                         = java
      "akka.dispatch.sysmsg.Watch"               = java

      "com.typesafe.conductr.CoreRemotableMessage"  = conductr-core-proto
      "com.typesafe.conductr.AgentRemotableMessage" = conductr-agent-proto
    }

    // We use a round-robin-pool actor here given that the default
    // consistent hashing one depends on akka remoting - which
    // may not have started up in time for contrail. Alternatively
    // we could supply our own consistent hashing router for this
    // use-case that did not depend on akka remoting. Something to
    // consider if contrail ever saw the light of day in application
    // code outside of ConductR.
    deployment {
      /IO-DNS/inet-address {
        mailbox = "unbounded"
        router = "round-robin-pool"
        nr-of-instances = 4
      }
    }
  }

  cluster {
    # INTERNAL: Do not overwrite this seed nodes parameter.
    # Instead, either use ConductR's --seed option or declare the `conductr.seed-nodes` parameter in your application.conf
    # For more information check out the `conductr.seed-nodes` documentation
    seed-nodes                  = []

    # The `replicator` role is required for bundle replication purposes, hence core nodes which hosts the bundle
    # files must have this role defined.
    # By default all core nodes act as host for bundle files, hence all core nodes will have the `replicator` role.
    roles                       = ["replicator"]

    downing-provider-class = "com.lightbend.akka.sbr.SplitBrainResolverProvider"
    split-brain-resolver {
      # Should cluster split occurs, keep the partition having majority of remaining nodes
      active-strategy = keep-majority
    }

    failure-detector {
      # The recommended value for slower networks including cloud environments: http://doc.akka.io/docs/akka/snapshot/scala/cluster-usage.html
      threshold = 12
    }
  }

  extensions = [
    "akka.cluster.client.ClusterClientReceptionist",
    "akka.cluster.ddata.DistributedData"
  ]

  http {
    parsing.max-content-length   = 200m
    server.remote-address-header = on
  }

  # We fallback on DNS SRV records if we cannot resolve services within ConductR
  io.dns {
    resolver = async-dns
    async-dns {
      resolve-srv = true
      resolv-conf = on
    }
  }

  remote {
    enabled-transports          = [akka.remote.netty.tcp]
    log-remote-lifecycle-events = off

    netty.tcp {
      hostname = ${conductr.ip}
      port     = 9004
    }
  }

  diagnostics.checker {
    confirmed-power-user-settings = [
      "akka.cluster.failure-detector.threshold",
      "akka.io.dns.resolver",
      "akka.io.dns.async-dns.resolv-conf",
      "akka.io.dns.async-dns.resolve-srv"
    ]
  }
}

# Lightbend ConductR configuration.
conductr {
  # Overwrites the system name with the `MARATHON_APP_ID` environment variable which is set on DCOS.
  # On DCOS, ConductR is started via Marathon so it uses the unique Marathon app id instead.
  # The marathon app id is converted into a legal actor system name.
  # In particular, the first '/' character is dropped and all subsequent '/' characters are replaced with '_'.
  system-name = ${?MARATHON_APP_ID}

  # Overwrites the ConductR IP address with 'LIBPROCESS_IP' environment variable.
  # This variable is usually set in a Mesos environment, e.g. on DCOS.
  ip = ${?LIBPROCESS_IP}

  # The ConductR mode
  mode = "mesos"

  # The seed nodes or constructr coordination service addresses to bootstrap the ConductR cluster.
  # The ConductR option --seed is available to add entries to `conductr-seed-nodes`.
  # Depending on the given addresses the ConductR cluster is bootstrapped differently:
  # 1. If empty, the ConductR node joins the cluster with itself (default)
  # 2. If the given addresses does not contain a Zookeeper address, i.e. does not start with zk://, then these addresses
  #    are interpretes as seed nodes. The ConductR node will join another ConductR cluster by given seed nodes.
  # 3. If at least one of the given addresses is a Zookeeper address, i.e. address starts with zk://, then the other addresses
  #    are filtered out and only the Zookeeper addresses are kept. These Zookeeper addresses are then interpreted
  #    as ConstructR coordination service nodes. ConstructR is then started and therefore responsible to bootstrap the ConductR cluster.
  seed-nodes = []

  # The directory where bundles and their configuration are written to.
  storage-dir = ${java.io.tmpdir}/conductr/${conductr.ip}/bundles

  # The amount of time Lightbend ConductR expects to wait on achieving a read quorum. 10 seconds should
  # cover large clusters, but you may need to increase this if read timeouts appear
  # frequently in your log files.
  read-quorum-timeout = 10 seconds

  # As per read-quorum-timeout.
  write-quorum-timeout = 10 seconds

  # The expected maximum amount of time it takes for the actor system to shutdown.
  # If the actor system fails to shutdown within this period, an error message will be printed.
  #
  # ConductR will attempt to restart its actor system when encountering error. If the shutdown timeout occurs in this
  # scenario, System.exit() with non-zero exit code will be called.
  #
  # If the shutdown timeout occurs as part of the JVM shutdown hook, System.exit() will not be called as invoking
  # System.exit() within a shutdown hook thread will cause the call to block indefinitely:
  # https://docs.oracle.com/javase/8/docs/api/java/lang/Runtime.html#exit-int-
  actor-system-shutdown-timeout = 60 seconds

  # Bundle Publisher
  bundle-publisher {
    # The time that a bundle can pending to be served for.
    request-ttl = 2 minutes
  }

  # The agent monitor keeps track of agent membership state.
  agent-monitor {
    # This is a parameter used for consistent hashing when determining what ConductR is responsible for
    # a given agent identifier. The parameter should not need to be changed.
    virtual-nodes-factor = 10

    # The time it takes for the monitor to assume that it is unlikely for an agent to come back
    # and therefore assume that it has no associated executions. Note that an agent may still
    # come back after this period.
    become-down-timeout = 15 seconds

    # The period of time to wait before we assume that an agent will never come back and we should
    # thus clean up any remaining state in relation to it.
    down-idle-timeout = 10 minutes

    # The amount of time we'll tolerate waiting for the agent information event
    agent-info-timeout = 20 seconds

    # The amount of time to wait until the bundle execution state is considered as settled.
    bundle-execution-state-settling-time = 5 seconds

    # The amount of time to wait to allow for the removal of agent state from core before reconciling
    # execution state with an agent again.
    agent-removal-settling-time = 5 seconds

    # The name of the launcher on the remote agent
    launcher-agent-name = "mesos-launcher-agent"
  }

  # Bundling streaming is an out-of-band private channel used between the ConductRs in
  # order to stream bundles and associated configuration.
  bundle-stream-server {
    # The protocol, IP address and port on which to serve bundling streaming server requests.
    protocol = http
    ip       = ${conductr.ip}
    port     = 9006

    # The amount of time to wait on attempting to bind.
    bind-timeout = 5 seconds

    # The amount of time to wait on one ConductR connecting to another's bundle stream
    # server.
    connect-timeout = 2 minutes

    # Bundles and configuration are transiently served i.e. once served then they are no
    # longer available. This parameter dictates the maximum number of bundles that are ready to
    # be served at one time.
    max-nr-of-pending-requests = 500

    # The time it takes to obtain request information within the handling of an http request
    # for streaming data.
    request-fetch-timeout = 5 seconds

    # The time that a bundle can pending to be served for.
    request-ttl = 2 minutes
  }

  # The control server provides http services in relation to controlling the ConductR e.g.
  # loading bundles, starting bundles, obtaining cluster state etc.
  control-server {
    # The protocol, IP address and port on which to serve control protocol requests.
    protocol = http
    ip       = ${conductr.ip}
    port     = 9005

    # The amount of time to wait on attempting to bind.
    bind-timeout = 5 seconds

    # The amount of time to wait on publishing a bundle.
    bundle-publisher-timeout = 20 seconds

    # The amount of time to wait on obtaining agent events publisher.
    agent-events-timeout = 5 seconds

    # The amount of time to wait on obtaining state events publisher.
    state-events-timeout = 5 seconds

    # The amount of time to wait on uploading a bundle into Lightbend ConductR. This timeout will vary depending
    # on how long it takes for clients of Lightbend ConductR to upload bundles i.e. the WAN will dictate this
    # parameter. If you find that clients e.g. the CLI, are reporting timeouts then try increasing
    # this parameter.
    load-scheduler-timeout = 1 minute

    # The amount of time to wait on requesting that an unload occurs. This should generally be fast
    # as it is only a request to be scheduled.
    unload-scheduler-timeout = 5 seconds

    # The amount of time to wait on requesting that a bundle be started.
    start-scheduler-timeout = 5 seconds

    # The amount of time to wait on performing cluster management operations e.g. join a cluster,
    # drop a member from the cluster etc.
    cluster-management-timeout = 5 seconds

    # The amount of time to wait when querying for logs or events for a particular bundle.
    log-repository-lookup-timeout = 5 seconds

    # The default number of events or logs fetched for each query if not specified.
    log-repository-lookup-default-size = 10

    # The maximum number of events or logs that can be fetched for each query.
    log-repository-lookup-max-size = 100

    # The service name of the deployment service for pipeline delivery
    deployment-service-name = "deployments"

    # The amount of time to wait for a service locator lookup in the context of proxying
    # control protocol requests.
    service-locator-timeout = 5 seconds

    # The service name of the web UI to serve from the control protocol root context.
    web-ui-service-name = "visualizer"

    # The maximum size of a bundle configuration given that it is read into memory
    max-bundle-conf-size = 8k

    sse {
      state {
        # The size of state event publisher buffer.
        publisher-buffer-size = 100

        # The interval of the heartbeat to keep the connection alive.
        heartbeat-interval = 1 second
      }

      cluster {
        # The size of cluster event publisher buffer.
        publisher-buffer-size = 100

        # The interval of the heartbeat to keep the connection alive.
        heartbeat-interval = 1 second
      }
    }
  }

  core-services {
    # The waiting time to retrieve the core services information
    retrieve-timeout = 5 seconds
  }

  # Loading is associated with the loading, replication and unloading of a bundle and its associated configuration.
  load {
    # When replicating, this parameter dictates the amount of time in between waiting on the convergence
    # of the data it needs in order to make decisions around replication. These decisions amount to
    # how many replicas of a bundle are to be made, or how many may be removed.
    bundle-replicator-converge-retry-delay = 5 seconds

    # The time that a bundle replicator for a given bundle id will exist before it stops
    # given no activity. Replicators remain active only when there is work to be done.
    bundle-replicator-idle-time = 2 minutes

    # The amount of time that a ConductR is expected to wait on another ConductR in order to replicate
    # a bundle. This parameter will be influenced by the speed on the bundle stream network between ConductRs.
    # If you find that clients e.g. the CLI, are reporting timeouts then try increasing this parameter.
    bundle-retrieve-timeout = 5 minutes

    # The amount of time required to source a node bundle file on disk at the current node.
    load-scheduler-source-timeout = 5 seconds

    # The amount of delay put in place before replying after a successful bundle load.
    # This delay is introduced to allow subscribers to be notified of CRDT changes before sending the reply, thus
    # reducing the chance of client seeing inconsistent information between subscribers (i.e. between LoadScheduler vs
    # ScaleScheduler)
    load-scheduler-reply-delay = ${akka.cluster.distributed-data.notify-subscribers-interval}

    # The amount of time LoadExecutor will wait for bundle info to be updated to all Core nodes.
    load-executor-bundle-info-update-timeout = ${conductr.write-quorum-timeout}

    # The number of bundles and configuration to retain on the file system. There is one bundle and config
    # stored per member up to a nr-of-replicas maximum. When a bundle starts executing, another replication
    # will occur. The goal is to ensure that nr-of-replicas represents the number of non-executing bundles
    # and config in the cluster. Having bundles and config made available in this way ensures that they can
    # be started quickly.
    nr-of-replicas = 3

    # This is a parameter used for consistent hashing when determining what ConductR is responsible for
    # a given bundle identifier. The parameter should not need to be changed.
    virtual-nodes-factor = 10

    # The amount of time to wait for changes in cluster membership before reacting i.e. re-determining
    # bundle replication.
    membership-settling-time = 5 seconds
  }

  mesos-scheduler-client {
    # The address of the Mesos master instance.
    # The address can be either a Zookeeper or Mesos master address.
    # A Zookeeper address is typically used when running multiple Mesos master instances (High-availabilty mode)
    # An address directly to the Mesos master is used in standalone mode.
    # Zookeeper formats:
    #   zk://host1:port1,host2:port2,.../path
    #   zk://username:password@host1:port1,host2:port2,.../path
    #   file:///path/to/file (where file contains one of the above)
    # Mesos standalone format:
    #   host:port
    master = "127.0.0.1:5050"

    # The time to wait before retrying the querying of a launcher from within the client during the client's
    # initialization phase
    launcher-identity-retry-delay = 1 second

    # Initial duration until the mesos scheduler client is restarted after a termination
    # The duration is increased for any subsequent restart up to a maximum duration specified in `max-backoff`
    min-backoff = 5 seconds

    # Maximum duration until the mesos scheduler client is restarted after a termination
    max-backoff = 30 seconds

    # After calculation of the exponential back-off an additional
    # random delay based on this factor is added, e.g. `0.2` adds up to `20%` delay.
    backoff-random-factor = 0.2

    # The time to wait before sending reconciliation requests to Mesos after a
    # Mesos re-registration has occurred.
    reregistration-reconciliation-delay = 15 seconds

    # ConductR framework info
    framework {
      # Name of the Mesos framework.
      name = "conductr"

      # If set, framework pid, executor pids and status updates are
      # checkpointed to disk by the agents. Checkpointing allows a
      # restarted agent to reconnect with old executors and recover
      # status updates, at the cost of disk I/O.
      checkpoint = true

      # Time to wait before the Mesos master will kill all the tasks and executors associated
      # with the downed ConductR core node that established a connection with Mesos
      # If another ConductR core node is available in the cluster then this node will automatically
      # re-establish a connection to Mesos with the same framework id once the previous node is down.
      # The failover-timeout is then canceled.
      failover-timeout = 7 days

      # Determine the Unix user that an executor should be launched as.
      # The default "" will automatically set it to the current user.
      user = ""

      # The framework role assigned to ConductR.
      role = "slave_public"
    }

    mesos-roles {
      # ConductR will use this mappings to translate bundle roles (specified through build.sbt or the bundle.conf) to
      # Mesos role.
      # When mapping is not found, `bundle-role-default` will be used instead.
      #
      # Based on the value defined below, this setup will allow ConductR HAProxy bundle to run on Mesos' public slaves,
      # while other bundles can run on the private slaves (which is by default assigned to '*' role).
      bundle-role-mappings = {
        cpus = {
          haproxy = "slave_public"
        }
        mem = {
          haproxy = "slave_public"
        }
        disk = {
          haproxy = "slave_public"
        }
        ports = {
          haproxy = "slave_public"
        }
      }

      # The default Mesos role assigned to a bundle when when no bundle role can be otherwise determined.
      bundle-role-default = "*"
    }

    mesos-executor {
      name = "conductr-agent"

      # The start command of a Mesos executor (the conduct agent) that gets invoked on executor startup.
      start-command = "$(pwd)/conductr-agent-*/bin/conductr-agent"

      # Log level of the ConductR agent. Defaults to the loglevel of the ConductR core node.
      loglevel = ${akka.loglevel}

      # When supplied this represents the URIs required by a ConductR agent, including the agent itself.
      # When using Mesos, this is used to supply the Mesos Fetcher with the agent's packages so that it may
      # be dynamically downloaded.
      agent-uris = []

      # Controls whether downloads of the agent are cached. The default is to cache downloads for
      # improved performance.
      agent-download-cache = on

      # Resources needed to start a Mesos executor that runs a ConductR agent
      # The CPU param represents Mesos's interpretation, which is expressed as
      # a fraction of the total # of CPUs. Here, we use the Mesos convention of
      # expressing 1.9/nrOfCpus.
      resources {
        cpu        = 1.9
        memory     = 536870912
        disk-space = 1000000000

        # The port resources represents the ports required by the ConductR agent.
        # The resources will be declared when launching the ConductR agent's ghost task, effectively reserving these
        # ports for the ConductR agent's usage.
        ports = {
          # Port representing Akka remoting.
          # ConductR agent's remoting will be configured with this port.
          # TODO: remove this once the replacement for Akka Cluster Client is in place
          akka-remoting = 2552,

          # Port range reserved for exposing endpoints for running bundles within ConductR.
          # ConductrR agent's port allocator settings will be configured based on this port range.
          allocated-ports = {
            start = 10000
            end = 10999
          }

          # Ports opened by the Agents to proxy requests from its Bundles to ConductR Core services, i.e.
          # Control Protocol, Status Server, and Service Locator.
          # ConductR agent's proxy port will be configured with these ports.
          proxy {
            control-server = 19005
            status-server = 19007
            service-locator-server = 19008
          }
        }
      }
    }
  }  

  # The resource provider receives requests for resources and provides resource offers.
  resource-provider {
    # The maximum number of resource requests that may be outstanding across the cluster at any one time.
    max-nr-of-pending-requests = 50

    # The amount of time to wait for a resource offer to be matched before being discarded.
    resource-offer-timeout = 20 seconds

    # When set, cluster roles are checked in terms of offers being made and matched. This should be off
    # if you are not concerned about what bundles can run where. For production purposes though, we recommend you
    # consider a topology where certain bundles run on certain machines e.g. when considering DMZs, databases
    # nodes etc.
    match-offer-roles = off
  }

  # Running is associated with the starting, scaling and stopping of a bundle within the cluster.
  run {
    # The maximum number of tries for starting a bundle.
    bundle-scaler-max-nr-of-tries = 3

    # Timeout values for the various states of a bundle scaler.
    bundle-scaler-idle-timeout      = 2 minutes
    bundle-scaler-scaling-timeout   = 10 seconds
    bundle-scaler-launching-timeout = 2 minutes
    bundle-scaler-starting-timeout  = 10 minutes

    # Timeout value for an idle system scaler.
    system-scaler-idle-timeout = 2 minutes

    # Timeout value for getting an acknowledge from a bundle scaler.
    system-scaler-acknowledge-timeout = 5 seconds

    # The maximum number of tries for getting an acknowledge from a bundle scaler.
    system-scaler-acknowledge-max-nr-of-tries = 3

    # This is a parameter used for consistent hashing when determining what ConductR is responsible for
    # a given bundle identifier. The parameter should not need to be changed.
    virtual-nodes-factor = 10

    # The amount of time that a ConductR is expected to wait on another ConductR in order to replicate
    # a bundle. This parameter will be influenced by the speed on the bundle stream network between ConductRs.
    # If you find that clients e.g. the CLI, are reporting timeouts then try increasing this parameter.
    bundle-retrieve-timeout = 5 minutes

    # The amount of time to wait for changes in cluster and agent membership before reacting i.e. re-determining
    # bundle scale.
    membership-settling-time = 5 seconds

  }

  service-locator-server {
    # The protocol, IP address and port on which to serve service locator requests.
    protocol = http
    ip       = ${conductr.ip}
    port     = 9008

    # The max time it should take for a service lookup to occur internally.
    locate-timeout = 5 seconds

    # In a Mesos environment we should indeed fallback to DNS SRV as it is provided by Mesos.
    dns-srv-fallback = on

    cache {
      # The max age a service name lookup should be cached for by a client.
      max-age        = 1 minute

      # The maximum number of service location responses to cache
      max-responses  = 100
    }

    sse {
      service {
        # The size of state event publisher buffer.
        publisher-buffer-size = 100

        # The interval of the heartbeat to keep the connection alive.
        heartbeat-interval = 1 second
      }
    }
  }

  service-proxy {
    # The name given to a service representing the proxy being used.
    name = "conductr-haproxy"
  }

  status-server {
    # The protocol, IP address and port on which to serve start-status server requests.
    protocol = http
    ip       = ${conductr.ip}
    port     = 9007

    # The duration to wait on attempting to bind
    bind-timeout = 5 seconds

    # The max time it should take for a status update to occur internally.
    status-timeout = 5 seconds
  }
}

# Lightbend Contrail configuration
contrail {

  syslog {

    loglevel = ${akka.loglevel}

    # Used as APP-NAME in syslog events.
    app-name = ConductR

    server {
      # We lean on the Mesos Elasticsearch instead of one managed by ConductR
      service-locator {
        enabled        = on
        selection-path = "/user/reaper/service-locator"
        service-name = "_client-port._elasticsearch-executor._tcp.elasticsearch.mesos"
      }

      elasticsearch {
        enabled = on
        index = "conductr"
      }
    }
  }
}

# Lightbend Monitoring configuration
cinnamon {

  instrumentation = off

  akka.actors {
    "/user/*" {
      report-by = class
    }
  }
}

# ConstructR configuration

constructr {
  # Never give up on ZK disappearing - the consequences of which would be that ConstructR
  # shuts down the actor system if retries are exceeded. We therefore set the # of retries
  # to the max positive value of a 32 bit signed int as that is how it is represented
  # internally.
  nr-of-retries = 2147483647
}

# A connection context for running the Mesos scheduler driver
mesos-scheduler-driver-run-executor {
  executor = "thread-pool-executor"

  # Controls the number of mesos scheduler drivers per node
  # The `MesosSchedulerDriver.run()` blocks the entire thread until the driver has been stopped or aborted.
  # Therefore we decicate a separate dispatcher to the run() method.
  # Only one driver per node is allowed, meaning we can we only dedicate 1 thread to this dispatcher
  thread-pool-executor {
    # Maximum number of threads within the pool.
    # Supports running only 1 driver at a time
    fixed-pool-size = 1
  }
}
```

### ConductR Agent Configuration

Here is the entire configuration for ConductR Agent when running in DC/OS mode.

```
akka {
  log-dead-letters                 = off
  log-dead-letters-during-shutdown = off
  loggers                          = [akka.event.slf4j.Slf4jLogger, com.typesafe.contrail.adapter.syslog.akka.SyslogLogger]
  logging-filter                   = "akka.event.slf4j.Slf4jLoggingFilter"
  loglevel                         = info

  actor {
    provider = akka.actor.LocalActorRefProvider

    enable-additional-serialization-bindings = on

    serializers = {
      conductr-agent-proto = "com.typesafe.conductr.AgentProtobufSerializer"
    }

    serialization-bindings {
      "java.io.Serializable"                     = none

      "com.typesafe.conductr.AgentRemotableMessage" = conductr-agent-proto
    }

    // We use a round-robin-pool actor here given that the default
    // consistent hashing one depends on akka remoting - which
    // may not have started up in time for contrail. Alternatively
    // we could supply our own consistent hashing router for this
    // use-case that did not depend on akka remoting. Something to
    // consider if contrail ever saw the light of day in application
    // code outside of ConductR.
    deployment {
      /IO-DNS/inet-address {
        mailbox = "unbounded"
        router = "round-robin-pool"
        nr-of-instances = 4
      }
    }
  }

  cluster.client {
    # ConductR Agent will try to reconnect to ConductR core upon disconnection for the period specified by this config.
    # Once the period is elapsed, the ConductR Agent will restart its actor system which will cause its bundles
    # to stop, and thus be consistent with ConductR core's view.
    reconnect-timeout = 30s
  }

  http {
    parsing.max-content-length = 200m
    server.remote-address-header = on
  }

  remote {
    enabled-transports = []
    netty.tcp {
      hostname = ${conductr.agent.ip}
    }
  }
}

conductr.agent {
  # Overwrites the ConductR IP address with 'LIBPROCESS_IP' environment variable.
  # This variable is usually set in a Mesos environment, e.g. on DCOS.
  ip = ${?LIBPROCESS_IP}

  # The ConductR mode
  mode = "mesos"

  # 'MESOS_SANDBOX' environment variable points to the current directory of the running ConductR Agent.
  # This means bundle and config zip files as well as bundle execution directories will be placed underneath inside
  # 'MESOS_SANDBOX' directory.
  #
  # IMPORTANT: do not point the storage dir to 'MESOS_SANDBOX'
  # The housekeeping will attempt to delete outdated bundle execution dirs as well as all outdated files within this
  # 'storage-dir'. When 'storage-dir' is pointed to 'MESOS_SANDBOX', this will risk all other agent files to be deleted
  # if the agent has been running for an extended period of time.
  storage-dir=${?MESOS_SANDBOX}/bundles

  # The directory where bundles record their pids for the purposes of being reaped if they're still
  # hanging around when ConductR restarts (perhaps due to ConductR having been rudely killed via
  # SIGKILL).
  bundle-pidfile-dir = ${conductr.agent.storage-dir}/pids

  # The address and port of the ConductR Core's `remote.netty.tcp.hostname` and `remote.netty.tcp.port` respectively.
  # This is used by the ConductR agent's Akka cluster client to form the initial contact so connection to
  # ConductR core can be established.
  core-nodes = []

  # The name of the ConductR core cluser, i.e. the Akka system name.
  # Corresponds to the ConductR core settings `conductr.system-name`.
  # This is used by the ConductR agent's Akka cluster client to form the initial contact so connection to
  # ConductR core can be established.
  core-system-name = "conductr"

  # The role of the ConductR agent.
  # The roles will be used to decide where bundles can execute, i.e. given a bundle is configured with the role
  # of "front-end", only agents that has "front-end" role can execute this bundle.
  roles = ["web"]

  # The expected maximum amount of time it takes for the actor system to shutdown.
  # If the actor system fails to shutdown within this period, an error message will be printed.
  #
  # ConductR will attempt to restart its actor system when encountering error. If the shutdown timeout occurs in this
  # scenario, System.exit() with non-zero exit code will be called.
  #
  # If the shutdown timeout occurs as part of the JVM shutdown hook, System.exit() will not be called as invoking
  # System.exit() within a shutdown hook thread will cause the call to block indefinitely:
  # https://docs.oracle.com/javase/8/docs/api/java/lang/Runtime.html#exit-int-
  actor-system-shutdown-timeout = 60 seconds

  agent-info {
    # The amount of time we'll tolerate waiting for a reply for bundle information before bailing out.
    launcher-agent-bundle-info-timeout = 5 seconds

    # The interval between each reconcilation event of agent info in order to ensure that the
    # agent state is in sync with core.
    reconcile-agent-info-interval = 15 seconds

    # The actor path of the resource provider in the ConductR core
    agent-monitor-remote-path = "/user/reaper/agent-monitor"
  }

  # Bundling streaming is an out-of-band private channel used between the ConductRs in
  # order to stream bundles and associated configuration.
  bundle-stream-server {
    # The amount of time to wait on one ConductR connecting to another's bundle stream
    # server.
    connect-timeout = 30 seconds
  }

  # Server which proxies ConductR services - control protocol, service locator, and status server.
  # The port of the services mirrors that of the ConductR, and the actual URL of these services will be obtained from
  # ConductR Core.
  core-services-proxy-server {
    protocol = http
    host = ${conductr.agent.ip}

    # The port where control protocol requests will be proxied to ConductR Core.
    control-server-port = 19005

    # The port where status server requests will be proxied to ConductR Core.
    status-server-port = 19007

    # The port where service locator requests will be proxied to ConductR Core.
    service-locator-server-port = 19008
  }

  resources {
    # The interval of polling for nr of cpus, memory, and diskspace
    poll-interval = 2 seconds

    # The actor path of the resource provider in the ConductR core
    resource-provider-remote-path = "/user/reaper/standalone-resource-provider-scale"
  }

  # Running is associated with the starting, scaling and stopping of a bundle within the cluster.
  run {
    # The start and end (inclusive) of the range for dynamically allocated ports.
    # It's a requirement that users of the ConductR must ensure that this port range is free.
    allocated-ports {
      start = 10000
      end   = 10999
    }

    # The command used to start Bash; this default should work on most systems.
    # Another sensible alternative would be ["/bin/bash"]
    bash-command = ["/usr/bin/env", "bash"]

    # When ConductR stops a bundle, it will attempt to shutdown the bundle's running children processes.
    # It does so by issuing a SIGTERM, and after a certain pause, it will issue a SIGKILL for each of the child process.
    # These configuration values control how long the pause between SIGTERM and SIGKILL mentioned above. "term-delay"
    # is the time between issuing a SIGTERM and then waiting before looking to see if it may be necessary to then do a
    # sleep of "term-kill-delay" which will then cause a SIGKILL. "term-delay" can be viewed as the normal amount of time
    # we allow processes to exit gracefully. Put another way, processes will have up to 8 seconds to exit gracefully,
    # but we expect most to exit within 2 seconds. The reason we have the two parameters is so that we can avoid the
    # longer delay in general. Note that we must stay well within 10 seconds as a Unix's system's restart scripts
    # are often about 10 seconds before they SIGKILL ConductR itself.
    term-delay      = 3 seconds
    term-kill-delay = 5 seconds

    # The amount of time to download a bundle from a ConductR core instance.
    # This parameter will be influenced by the speed on the bundle stream network between ConductR agent and core.
    # If you find that the ConductR agent is reporting timeouts then try increasing this parameter.
    bundle-retrieve-timeout = 5 minutes

    core-services-proxy {
      # The actor path of the actor that returns the ip address of the services provided in the ConductR core
      remote-path = "/user/reaper/core-services"

      request {
        # The amount of timeout waiting for reply from the ConductR core node when requesting for ip address of the
        # services
        timeout = 5 seconds

        backoff {
          min-backoff = ${conductr.agent.run.core-services-proxy.request.timeout}
          max-backoff = 30 seconds
          random-factor = 0.2
        }
      }
    }

    bundle-house-keeping {
      # Bundle house keeping will try to remove unused files from the bundles directory specified in
      # `conductr.agent.storage-dir`. Bundle house keeping will be invoked when a new bundle starts.
      #
      # Bundle house keeping will attempt to delete bundle run directories except the latest directories.
      # The number of the latest directories to be kept is controlled by `bundle-dirs-threshold`. This attempt will
      # remove old bundle run directories and its contents (i.e. jars and logs, etc).
      #
      # After cleaning up the bundle run directories, and if the size threshold is breached, bundle house
      # keeping will delete files older than specified `time-threshold`. This attempt will remove old bundle zip files.
      #
      # If the size threshold is still breached after this attempt, log warning message will be issued.

      # Number of bundle run directory to be kept within a particular bundle.
      # Important: never set this number below 1, otherwise the directory of currently running bundle will be deleted.
      bundle-run-dirs-threshold = 2

      size-threshold = 4GB
      time-threshold = 30 days

      # Time delay between bundle housekeeping being scheduled to when it's performed.
      start-delay = 5 seconds
    }
  }

  service-proxy {
    # The name given to a service representing the proxy being used.
    name = "conductr-haproxy"
  }
}

# Lightbend Contrail configuration
contrail {

  syslog {

    loglevel = ${akka.loglevel}

    # Used as APP-NAME in syslog events.
    app-name = "ConductR-Agent"

    server {
      # We lean on the Mesos Elasticsearch instead of one managed by ConductR
      service-locator {
        enabled        = on
        selection-path = "/user/reaper/syslog-service-locator-client"
        service-name   = "_client-port._elasticsearch-executor._tcp.elasticsearch.mesos"
      }

      elasticsearch {
        enabled = on
        index = "conductr"
      }
    }
  }
}

# Lightbend Monitoring configuration
cinnamon {

  instrumentation = off

  akka.actors {
    "/user/*" {
      report-by = class
    }
  }
}

# A connection context for running the Mesos executor driver
mesos-executor-driver-run-executor {
  executor = "thread-pool-executor"

  # Controls the number of mesos executor drivers per node
  # The `MesosExecutorDriver.run()` blocks the entire thread until the driver has been stopped or aborted.
  # Therefore we decicate a separate dispatcher to the run() method.
  # Only one driver per node is allowed, meaning we can we only dedicate 1 thread to this dispatcher
  thread-pool-executor {
    # Maximum number of threads within the pool.
    # Supports running only 1 driver at a time
    fixed-pool-size = 1
  }
}
```
