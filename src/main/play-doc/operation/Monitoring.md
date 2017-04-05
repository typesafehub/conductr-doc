# Lightbend Monitoring

[Lightbend Monitoring](http://developer.lightbend.com/docs/monitoring/latest/home.html) is a suite of insight tools for monitoring Lightbend Platform applications. These tools, which include [OpsClarity](https://www.opsclarity.com/), gain visibility into ConductR and its bundles — to respond early to changes that could indicate problems, to tune a system, or to track down the cause of unexpected behavior.

This guide discusses ConductR specifics regarding Lightbend Monitoring. For a comprehensive description please visit [Lightbend Monitoring's documentation](http://developer.lightbend.com/docs/monitoring/latest/home.html).

## Monitoring ConductR

ConductR is a Lightbend Reactive Platform based application and has been configured to support Lightbend Monitoring with zero configuration. By default, Akka, clustering, remoting, dispatchers and the Split Brain Resolver are all instrumented. In addition, this telemetry is reported via DogStatsD/UDP to 127.0.0.1 on port 8125. A DogStatsD protocol service such as the [OpsClarity](https://www.opsclarity.com/) agent is expected to reside at that address.

Please follow the instructions for installing OpsClarity agents [here](https://support.opsclarity.com/hc/en-us/articles/213214888-Install-the-OpsClarity-agent). 
By way of example, to start the OpsClarity agent on port 8125 (port 8125 is required in place of the 10101 default port given that the latter will clash with ConductR):

```
sudo bash ./agent-installer.sh -p 8125 -k "<your-ops-clarity-key-goes-here>"
```

## Monitoring Bundles

You should also incorporate Lightbend Monitoring capabilities into your bundles. Please see [the related OpsClarity documentation](https://support.opsclarity.com/hc/en-us/articles/115005141468-Akka-Statsd-Support) which describes how to configure Lightbend Monitoring for the DogStatsD protocol (used by OpsClarity).
