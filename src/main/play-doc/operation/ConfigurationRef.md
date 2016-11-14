# Configuration Reference

## ConductR Core

Each one of the following properties can be overridden by declaring its path and value in ConductR Core's `conf/conductr.ini`. For example to set ConductR Core's storage directory you'd specify `-Dconductr.storage-dir=/some-other-dir`.

### Akka Configuration

The following Akka properties are overridden:

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

    serializers {
      "akka-misc"            = "akka.remote.serialization.MiscMessageSerializer"
      "conductr-core-proto"  = "com.typesafe.conductr.CoreProtobufSerializer"
      "conductr-agent-proto" = "com.typesafe.conductr.AgentProtobufSerializer"
    }

    serialization-bindings {
      "java.io.Serializable"                     = none
      "akka.actor.ActorIdentity"                 = "akka-misc"
      "akka.actor.Identify"                      = "akka-misc"
      "com.typesafe.conductr.CoreRemotableMessage"  = "conductr-core-proto"
      "com.typesafe.conductr.AgentRemotableMessage" = "conductr-agent-proto"
    }

    serialization-identifiers {
      "akka.remote.serialization.MiscMessageSerializer" = 16
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
    seed-nodes                  = ["akka.tcp://conductr@"${conductr.ip}":9004"]

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

    client.receptionist {
      name = "receptionist"
      role = ""
      number-of-contacts = 3
      response-tunnel-receive-timeout = 30s
      use-dispatcher = ""

      # ConductR receives continuous heartbeat from the ConductR agent using Akka Cluster Client.
      # If there is no heartbeat received in the time specified, the agent is presumed to be down.
      # Once the ConductR agent is down, it would mean resources from the ConductR agent (memory, disk space, and
      # nr of cpus) is considered unavailable.
      heartbeat-interval = 2s

      # Number of potentially lost/delayed heartbeats that will be
      # accepted before considering it to be an anomaly.
      # The ClusterReceptionist is using the akka.remote.DeadlineFailureDetector, which
      # will trigger if there are no heartbeats within the duration
      # heartbeat-interval + acceptable-heartbeat-pause, i.e. 15 seconds with
      # the default settings.
      acceptable-heartbeat-pause = 13s

      # Failure detection checking interval for checking all ClusterClients
      failure-detection-interval = 2s
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

  remote {
    enabled-transports          = [akka.remote.netty.tcp]
    log-remote-lifecycle-events = off

    netty.tcp {
      hostname = ${conductr.ip}
      port     = 9004
    }
  }

  diagnostics.checker {
    confirmed-typos = [
      "akka.cluster.client.receptionist.heartbeat-interval",
      "akka.cluster.client.receptionist.failure-detection-interval",
      "akka.cluster.client.receptionist.acceptable-heartbeat-pause",
      "akka.actor.serialization-identifiers.\"akka.remote.serialization.MiscMessageSerializer\""
    ]

    confirmed-power-user-settings = [
      "akka.cluster.failure-detector.threshold"
    ]
  }
}
```

## ConductR Core Configuration

Here is the entire configuration for ConductR Core.

```
conductr {
  # The IP address used by default for all services, e.g. control server, bundle stream server, etc.
  # Defaults to the loopback IP address and can be set via the CONDUCTR_IP environment variable.
  ip = "127.0.0.1"
  ip = ${?CONDUCTR_IP}

  # The mode in which ConductR is started.
  # Modes: 'standalone', 'mesos'
  # By default 'standalone' is used.
  # If ConductR should be started on Mesos then it is recommended to specify the 'mesos-master' option when starting
  # the ConductR process. The 'mesos-master' option will set the mode to 'mesos' and
  # the 'resource-provider.mesos.master' to the given address
  mode = "standalone"

  # The directory where bundles and their configuration are written to.
  storage-dir = ${java.io.tmpdir}/conductr/bundles

  # The amount of time Lightbend ConductR expects to wait on achieving a read quorum. 10 seconds should
  # cover large clusters, but you may need to increase this if read timeouts appear
  # frequently in your log files.
  read-quorum-timeout = 10 seconds

  # As per read-quorum-timeout.
  write-quorum-timeout = 10 seconds

  # Bundle Publisher
  bundle-publisher {
    # The time that a bundle can pending to be served for.
    request-ttl = 30 seconds
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
    connect-timeout = 30 seconds

    # Bundles and configuration are transiently served i.e. once served then they are no
    # longer available. This parameter dictates the maximum number of bundles that are ready to
    # be served at one time.
    max-nr-of-pending-requests = 500

    # The time it takes to obtain request information within the handling of an http request
    # for streaming data.
    request-fetch-timeout = 5 seconds

    # The time that a bundle can pending to be served for.
    request-ttl = 30 seconds
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
    bundle-replicator-idle-time = 30 seconds

    # The amount of time that a ConductR is expected to wait on another ConductR in order to replicate
    # a bundle. This parameter will be influenced by the speed on the bundle stream network between ConductRs.
    # If you find that clients e.g. the CLI, are reporting timeouts then try increasing this parameter.
    bundle-retrieve-timeout = 5 minutes

    # The amount of time required to source a node bundle file on disk at the current node.
    load-scheduler-source-timeout = 5 seconds

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
    # bundle replication and scale.
    membership-settling-time = 5 seconds
  }

  # Mesos related settings.
  # There are only used if the 'conductr.mode' is set to 'mesos'
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

    # ConductR framework info
    framework {
      name = "conductr"
      checkpoint = false
      user = "conductr"
    }

    mesos-executor {
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
      # expressing just 0.1/nrOfCpus.
      resources {
        cpu        = 0.1
        memory     = 268400000
        disk-space = 1000000000
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
    # The start and end (inclusive) of the range for dynamically allocated ports.
    # It's a requirement that users of the ConductR must ensure that this port range is free.
    allocated-ports {
      start = 10000
      end   = 10999
    }

    # The command used to start Bash; this default should work on most systems.
    # Another sensible alternative would be ["/bin/bash"]
    bash-command = ["/usr/bin/env", "bash"]

    # The maximum number of tries for starting a bundle.
    bundle-scaler-max-nr-of-tries = 3

    # Timeout values for the various states of a bundle scaler.
    bundle-scaler-idle-timeout      = 30 seconds
    bundle-scaler-scaling-timeout   = 10 seconds
    bundle-scaler-launching-timeout = 30 seconds
    bundle-scaler-starting-timeout  = 10 minutes

    # Timeout value for an idle system scaler.
    system-scaler-idle-timeout = 30 seconds

    # Timeout value for getting an acknowledge from a bundle scaler.
    system-scaler-acknowledge-timeout = 5 seconds

    # The maximum number of tries for getting an acknowledge from a bundle scaler.
    system-scaler-acknowledge-max-nr-of-tries = 3

    # This is a parameter used for consistent hashing when determining what ConductR is responsible for
    # a given bundle identifier. The parameter should not need to be changed.
    virtual-nodes-factor = 10

    # When ConductR stops a bundle, it will attempt to shutdown the bundle's running children processes.
    # It does so by issuing a SIGTERM, and after a certain pause, it will issue a SIGKILL for each of the child process.
    # These configuration values control how long the pause between SIGTERM and SIGKILL mentioned above. "term-delay"
    # is the time between issuing a SIGTERM and then waiting before looking to see if it may be necessary to then do a
    # sleep of "term-kill-delay" which will then cause a SIGKILL. "term-delay" can be viewed as the normal amount of time
    # we allow processes to exit gracefully. Put another way, processes will have up to 10 seconds to exit gracefully,
    # but we expect most to exit within 5 seconds. The reason we have the two parameters is so that we can avoid the
    # longer delay in general.
    term-delay      = 5 seconds
    term-kill-delay = 8 seconds

    # The amount of time that a ConductR is expected to wait on another ConductR in order to replicate
    # a bundle. This parameter will be influenced by the speed on the bundle stream network between ConductRs.
    # If you find that clients e.g. the CLI, are reporting timeouts then try increasing this parameter.
    bundle-retrieve-timeout = 5 minutes

    standalone-mode {
      agent-remote-path = "/user/reaper/standalone-launcher-agent"
    }
  }

  service-locator-server {
    # The protocol, IP address and port on which to serve service locator requests.
    protocol = http
    ip       = ${conductr.ip}
    port     = 9008

    # The max time it should take for a service lookup to occur internally.
    locate-timeout = 5 seconds

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

  agent {
    # The actor path on the ConductR agent which can be used by the Core node to interrogate information on
    # the agent, i.e. roles.
    agent-info-remote-path = "/user/reaper/agent-info"
  }
}
```

### Contrail Configuration

In addition, a module named "contrail" is used for logging. Here is its entire declaration:

```
contrail {

  syslog {

    # Log level used by the configured syslog loggers.
    # See http://tools.ietf.org/html/rfc3164 for detailed explanation of levels (referred to as "Severity")
    # Options: OFF, EMERGENCY, ALERT, CRITICAL, ERROR, WARNING, NOTICE, INFO, DEBUG
    loglevel = INFO

    # Currently only TCP is supported.
    transport = tcp

    # How to determine this host's name, available:
    # hostname - use the unix `hostname` command
    # static   - configured in `contrail.syslog.hostname`
    hostname-mode = hostname

    # Static hostname, used when `contrail.syslog.hostname-mode` is set to `static`
    hostname = localhost

    # used for APP-NAME field in Syslog events
    app-name = contrail-app

    # Timeout for the initial connection
    connect-timeout = 10s

    # The amount of time after which a broken/closed connection to the syslog server should be re-established.
    reconnect-interval = 3s

    # Backpressure mode to be applied to log events if unable to send all over to the syslog collector,
    # Available:
    #   drop-latest
    #   drop-oldest
    #   drop-by-priority
    #   drop-by-severity
    #   drop-by-facility
    collapsing-strategy = drop-by-severity

    # default facility to be used by all syslog-loggers
    # see http://tools.ietf.org/html/rfc3164 for details, allowed values are:
    #   0             kernel messages
    #   1             user-level messages (default)
    #   2             mail system
    #   3             system daemons
    #   4             security/authorization messages (note 1)
    #   5             messages generated internally by syslogd
    #   6             line printer subsystem
    #   7             network news subsystem
    #   8             UUCP subsystem
    #   9             clock daemon (note 2)
    #  10             security/authorization messages (note 1)
    #  11             FTP daemon
    #  12             NTP subsystem
    #  13             log audit (note 1)
    #  14             log alert (note 1)
    #  15             clock daemon (note 2)
    #  16             local use 0  (local0)
    #  17             local use 1  (local1)
    #  18             local use 2  (local2)
    #  19             local use 3  (local3)
    #  20             local use 4  (local4)
    #  21             local use 5  (local5)
    #  22             local use 6  (local6)
    #  23             local use 7  (local7)
    facility = 1

    buffer {
      # number of log messages to be buffered
      size = 100
    }

    # may be used for statically configured Syslog instances.
    # this configuration is only used when Syslog is initialized in static mode (programatically)
    server {
      host = "127.0.0.1"
      port = 514

      # An alternative to providing the above host and port is to use a service locator provided
      # as an actor outside of contrail.
      service-locator {
        enabled = off

        # The actor selection path to resolve in order to communicate with a locator.
        # When service location is enabled then this selection is expected to resolve to
        # a com.typesafe.contrail.adapter.syslog.ServiceLocator actor.
        selection-path = "/user/syslog-service-locator"

        # The service to lookup when locating services
        service-name = "syslog"

        # The maximum amount of time to wait in order to locate a service
        lookup-timeout = 2 seconds
      }

      elasticsearch {
        enabled: off            # Whether or not to regard the endpoint as being that of elasticsearch
        index: "contrail"       # The index to write to - applications will typically override this
        max-post-size: 5 MiB    # The max size in bytes of a message that can be sent to ES (must not exceed 4 Gib)
        post-interval: 1 second # The amount of time to wait for at least one message before posting to ES (unless post-size occurs first)
      }
    }

    structured-data {
      expose = [
        # loggly, # example usage for including API keys etc

        stacktrace, # includes stacktraces of exceptions
        akka, # includes actor system name - those are "autofilled"
        mdc   # includes logger's MDC - those are "autofilled"
      ]

      # loggly {
      #   id = "my-loggly-key"
      #   values = [
      #     # key value pairs, represented as 2 element lists here, because duplicates *are allowed*
      #     # in structured data, they are not however in json so we can't use the {key: value} syntax.
      #     [tag, "my-cluster-007"],
      #     [tag, "backend"]
      #   ]
      # }

      # depth of stacktrace to be included in syslog event with exception
      # full         - entire stacktrace
      # NOT IMPLEMENTED YET: max-cause-## - only the ## first causes
      # NOT IMPLEMENTED YET: max-lines-## - only the ## first lines
      stacktrace-mode = full
    }
  }
}
```

ConductR Core overrides contrail settings as follows:

```
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
```

### Monitoring configuration

The following monitoring properties are declared:

```
cinnamon {

  instrumentation = off

  akka.actors {
    "/user/*" {
      report-by = class
    }
  }
}
```

## ConductR Agent

Each one of the following properties can be overridden by declaring its path and value in ConductR Agent's `conf/conductr-agent.ini`. For example to set ConductR Agent's storage directory you'd specify `-Dconductr.agent.storage-dir=/some-other-dir`.

### Akka Configuration

The following Akka properties are overridden:

```
akka {
  log-dead-letters                 = off
  log-dead-letters-during-shutdown = off
  loggers                          = [akka.event.slf4j.Slf4jLogger, com.typesafe.contrail.adapter.syslog.akka.SyslogLogger]
  logging-filter                   = "akka.event.slf4j.Slf4jLoggingFilter"
  loglevel                         = info

  actor {
    provider = akka.cluster.ClusterActorRefProvider

    serializers = {
      "akka-misc"            = "akka.remote.serialization.MiscMessageSerializer"
      "conductr-agent-proto" = "com.typesafe.conductr.AgentProtobufSerializer"
    }

    serialization-bindings {
      "java.io.Serializable"                     = none
      "akka.actor.ActorIdentity"                 = "akka-misc"
      "akka.actor.Identify"                      = "akka-misc"
      "com.typesafe.conductr.AgentRemotableMessage" = "conductr-agent-proto"
    }

    serialization-identifiers {
      "akka.remote.serialization.MiscMessageSerializer" = 16
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

  http {
    parsing.max-content-length = 200m
  }

  remote {
    netty.tcp {
      hostname = ${conductr.agent.ip}
    }
  }

  diagnostics.checker {
    confirmed-typos = [
      "akka.actor.serialization-identifiers.\"akka.remote.serialization.MiscMessageSerializer\""
    ]
  }

  cluster.client {
    # ConductR Agent will try to reconnect to ConductR core upon disconnection for the period specified by this config.
    # Once the period is elapsed, the ConductR Agent will restart its actor system.
    # DO NOT DISABLE THIS SETTINGS !!!
    # Disabling this setting will mean ConductR Agent will not restart when disconnected from the ConductR core.
    #
    # Once ConductR Agent is responsible for monitoring bundle execution, this setting will ensure runaway process
    # that is no longer visible to ConductR is terminated when ConductR Agent restarts. If this setting is disabled,
    # the runaway process will not be terminated.
    #
    reconnect-timeout = 30s
  }
}
```

## ConductR Agent Configuration

Here is the entire configuration for ConductR Agent.

```
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
  storage-dir = ${java.io.tmpdir}/conductr-agent/bundles

  # The directory where bundles record their pids for the purposes of being reaped if they're still
  # hanging around when ConductR restarts (perhaps due to ConductR having been rudely killed via
  # SIGKILL).
  bundle-pidfile-dir = ${java.io.tmpdir}/conductr-agent/bundles/pids

  # The address and port of the ConductR Core's `remote.netty.tcp.hostname` and `remote.netty.tcp.port` respectively.
  # This is used by the ConductR agent's Akka cluster client to form the initial contact so connection to
  # ConductR core can be established.
  core-nodes = [
    "127.0.0.1:9004"
  ]

  # The role of the ConductR agent.
  # The roles will be used to decide where bundles can execute, i.e. given a bundle is configured with the role
  # of "front-end", only agents that has "front-end" role can execute this bundle.
  roles = ["web"]

  # Bundling streaming is an out-of-band private channel used between the ConductRs in
  # order to stream bundles and associated configuration.
  bundle-stream-server {
    # The amount of time to wait on one ConductR connecting to another's bundle stream
    # server.
    connect-timeout = 30 seconds
  }

  # The control server provides http services in relation to controlling the ConductR e.g.
  # loading bundles, starting bundles, obtaining cluster state etc.
  control-server {
    # The protocol, IP address and port on which to serve control protocol requests.
    protocol = http
    port = 9005
  }

  resources {
    # The interval of polling for nr of cpus, memory, and diskspace
    poll-interval = 2 seconds

    # The actor path of the resource provider in the ConductR core
    resource-provider-remote-path = "/user/reaper/standalone-resource-provider"
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
    # we allow processes to exit gracefully. Put another way, processes will have up to 10 seconds to exit gracefully,
    # but we expect most to exit within 2 seconds. The reason we have the two parameters is so that we can avoid the
    # longer delay in general.
    term-delay      = 2 seconds
    term-kill-delay = 8 seconds

    # The amount of time to download a bundle from a ConductR core instance.
    # This parameter will be influenced by the speed on the bundle stream network between ConductR agent and core.
    # If you find that the ConductR agent is reporting timeouts then try increasing this parameter.
    bundle-retrieve-timeout = 5 minutes

    core-services {
      # The actor path of the actor that returns the ip address of the services provided in the ConductR core
      remote-path = "/user/reaper/core-services"

      request {
        # The amount of timeout waiting for reply from the ConductR core node when requesting for ip address of the
        # services
        timeout = 5 seconds

        backoff {
          min-backoff = ${conductr.agent.run.core-services.request.timeout}
          max-backoff = 30 seconds
          random-factor = 0.2
        }
      }
    }
  }

  service-locator-server {
    # The protocol, IP address and port on which to serve service locator requests.
    protocol = http
    port = 9008
  }

  service-proxy {
    # The name given to a service representing the proxy being used.
    name = "conductr-haproxy"
  }

  status-server {
    # The protocol, IP address and port on which to serve start-status server requests.
    protocol = http
    port     = 9007
  }
}
```


### Contrail Configuration

ConductR Agent overrides contrail settings as follows:

```
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
```

### Monitoring configuration

The following monitoring properties are declared:

```
cinnamon {

  instrumentation = off

  akka.actors {
    "/user/*" {
      report-by = class
    }
  }
}
```