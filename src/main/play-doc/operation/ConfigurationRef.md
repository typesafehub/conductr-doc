# Configuration Reference

Each one of the following properties can be overridden by declaring its path and value in ConductR's `conf/application.ini`. For example to set ConductR's storage directory you'd specify `-Dconductr.storage-dir=/some-other-dir`.

## Akka Configuration

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
      "conductr-proto" = "com.typesafe.conductr.ProtobufSerializer"
    }

    serialization-bindings {
      "java.io.Serializable"                   = none
      "com.typesafe.conductr.RemotableMessage" = "conductr-proto"
    }
  }

  cluster {
    seed-nodes                  = ["akka.tcp://conductr@"${conductr.ip}":9004"]
    roles                       = ["web"]

    split-brain-resolver {
      # Should cluster split occurs, keep the partition having majority of remaining nodes
      active-strategy = keep-majority
    }

    failure-detector {
      # The recommended value for slower networks including cloud environments: http://doc.akka.io/docs/akka/snapshot/scala/cluster-usage.html
      threshold = 12
    }
  }

  extensions = ["akka.cluster.ddata.DistributedData"]

  http {
    parsing.max-content-length = 200m
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
      "akka.logging-filter"
    ]

    confirmed-power-user-settings = [
      "akka.cluster.failure-detector.threshold"
    ]
  }
}

# A dispatcher for executing bundle processes
bundle-execution-dispatcher {
  executor = "thread-pool-executor"

  # Controls the number of runnable bundle in a node
  # Each bundle requires 2 threads to monitor stdout and stderr.
  # One additional thread is required by ConductR to run instances of BlockingProcess actor which manage bundle process
  thread-pool-executor {
    # Minimum number of threads within the pool.
    # Supports running 10 bundles at a minimum (i.e. 10 * 2 threads per bundle + 1 Blocking process running thread)
    core-pool-size-min    = 21
    core-pool-size-factor = 1.0
    core-pool-size-max    = 21

    # Maximum number of threads within the pool.
    # Supports running 500 bundles at a minimum (i.e. 500 * 2 threads per bundle + 1 Blocking process running thread)
    max-pool-size-min    = 1001
    max-pool-size-factor = 1.0
    max-pool-size-max    = 1001
  }
}
```

## ConductR Configuration
Here is the entire configuration for ConductR. 

```
conductr {
  # The IP address used by default for all services, e.g. control server, bundle stream server, etc.
  # Defaults to the loopback IP address and can be set via the CONDUCTR_IP environment variable.
  ip = "127.0.0.1"
  ip = ${?CONDUCTR_IP}

  # The directory where bundles and their configuration are written to.
  storage-dir = ${java.io.tmpdir}

  # The directory where bundles record their pids for the purposes of being reaped if they're still
  # hanging around when ConductR restarts (perhaps due to ConductR having been rudely killed via
  # SIGKILL).
  bundle-pidfile-dir = ${java.io.tmpdir}/bundle-pids

  # The amount of time Typesafe ConductR expects to wait on achieving a read quorum. 10 seconds should
  # cover large clusters, but you may need to increase this if read timeouts appear
  # frequently in your log files.
  read-quorum-timeout = 10 seconds

  # As per read-quorum-timeout.
  write-quorum-timeout = 10 seconds

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
    request-fetch-timeout = 2 seconds

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

    # The amount of time to wait on obtaining state events publisher.
    state-events-timeout = 2 seconds

    # The amount of time to wait on uploading a bundle into Typesafe ConductR. This timeout will vary depending
    # on how long it takes for clients of Typesafe ConductR to upload bundles i.e. the WAN will dictate this
    # parameter. If you find that clients e.g. the CLI, are reporting timeouts then try increasing
    # this parameter.
    load-scheduler-timeout = 1 minute

    # The amount of time to wait on requesting that an unload occurs. This should generally be fast
    # as it is only a request to be scheduled.
    unload-scheduler-timeout = 2 seconds

    # The amount of time to wait on requesting that a bundle be started.
    start-scheduler-timeout = 2 seconds

    # The amount of time to wait on performing cluster management operations e.g. join a cluster,
    # drop a member from the cluster etc.
    cluster-management-timeout = 5 seconds

    # The amount of time to wait when querying for logs or events for a particular bundle.
    log-repository-lookup-timeout = 2 seconds

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

  # Loading is associated with the loading, replication and unloading of a bundle and its associated configuration.
  load {
    # When replicating, this parameter dictates the amount of time in between waiting on the convergence
    # of the data it needs in order to make decisions around replication. These decisions amount to
    # how many replicas of a bundle are to be made, or how many may be removed.
    bundle-replicator-converge-retry-delay = 2 seconds

    # The time that a bundle replicator for a given bundle id will exist before it stops
    # given no activity. Replicators remain active only when there is work to be done.
    bundle-replicator-idle-time = 30 seconds

    # The amount of time that a ConductR is expected to wait on another ConductR in order to replicate
    # a bundle. This parameter will be influenced by the speed on the bundle stream network between ConductRs.
    # If you find that clients e.g. the CLI, are reporting timeouts then try increasing this parameter.
    bundle-retrieve-timeout = 5 minutes

    # The amount of time required to source a node bundle file on disk at the current node.
    load-scheduler-source-timeout = 2 seconds

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

  # The resource provider receives requests for resources and provides resource offers.
  resource-provider {
    # The maximum number of resource requests that may be outstanding across the cluster at any one time.
    max-nr-of-pending-requests = 50

    # The amount of time to wait for a resource offer to be matched before being discarded.
    resource-offer-timeout = 20 seconds

    # When set, cluster roles are checked in terms of offers being made and matched. Turn this off only
    # if you are not concerned about what bundles can run where. For production purposes though, you should
    # consider a topology where certain bundles run on certain machines e.g. when considering DMZs, databases
    # nodes etc.
    match-offer-roles = on
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

    # When scheduling the starting of a bundle, should the bundles only be run where they exist on the file
    # system? By default this is enabled, but there are other configurations where bundles are able to run
    # on a host that does not have a copy of the bundle and its configuration on its file system e.g. Mesos
    # slaves.
    should-start-where-loaded = on

    # This is a parameter used for consistent hashing when determining what ConductR is responsible for
    # a given bundle identifier. The parameter should not need to be changed.
    virtual-nodes-factor = 10

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
  }

  service-locator-server {
    # The protocol, IP address and port on which to serve service locator requests.
    protocol = http
    ip       = ${conductr.ip}
    port     = 9008

    # The max time it should take for a service lookup to occur internally.
    locate-timeout = 200 ms

    cache {
      # The max age a service name lookup should be cached for by a client.
      max-age        = 1m

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
    # The IP address for the service proxy.
    ip = ${conductr.ip}
  }

  status-server {
    # The protocol, IP address and port on which to serve start-status server requests.
    protocol = http
    ip       = ${conductr.ip}
    port     = 9007

    # The duration to wait on attempting to bind
    bind-timeout = 5 seconds
  }
}
```

## Contrail Configuration

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

ConductR overrides contrail settings as follows:

```
contrail {

  syslog {

    loglevel = ${akka.loglevel}

    # Used as APP-NAME in syslog events.
    app-name = ConductR

    server {
      host = 127.0.0.1
      port = 9200

      elasticsearch {
        enabled = on
        index = "conductr"
      }
    }
  }
}
```

## Monitoring configuration

The following monitoring properties are declared:

```
cinnamon {

  instrumentation = off

  takipi.actors {
    "/user/*" {
      report-by = class
    }
  }
}
```