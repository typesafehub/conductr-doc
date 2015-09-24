# Logging

When multiple machines are involved in a cluster it quickly becomes difficult to view the log files of distributed applications. ConductR allows the logging output of itself and the bundles that it executes to be directed to a "syslog collector". Syslog is a widely used protocol for Unix based machines and is supported by a number of cloud-based log providers, as well as local operating system support.

The syslog collector can send the log messages to any kind of logging solution. The ConductR distribution includes Elasticsearch and Kibana as an opt-in logging infrastructure. How to configure Elasticsearch and Kibana or other popular logging solutions are described in the next sections.


## Setting up Elasticsearch

Elasticsearch is available either as the `conductr-elasticsearch` bundle or you can [use your own](#Customized-Elasticsearch). The provided bundle can be found in the `extra` folder inside the ConductR installation folder. Also a default configuration for a typical production environment has been provided. 

Firstly for each node that will run Elasticsearch you must enable access to `/var/log` and `/var/lib`:

```bash
sudo mkdir -p /var/lib/elasticsearch /var/log/elasticsearch
sudo chown conductr:conductr /var/lib/elasticsearch
sudo chown conductr:conductr /var/log/elasticsearch
```

`conductr-elasticsearch` is using the the role `elasticsearch`.

To load and run Elasticsearch use the control API of ConductR, e.g. by using the CLI:

```bash
conduct load file:/usr/share/conductr/extra/conductr-elasticsearch-{version}-{digest}.zip elasticsearch-prod-{digest}.zip
conduct run conductr-elasticsearch
```

With that, the syslog collector streams the log messages to Elasticsearch. Use the CLI to access log messages by bundle id or name, e.g.:

```bash
conduct logs my-bundle
```

### Configuration

The provided Elasticsearch bundle configuration is using these settings:
- Memory: 4 GB
- Number of CPUs: 2
- Disk Space: 10 GB

To change the settings create a new bundle configuration by modiying the `bundle.conf` file inside of the bundle configuration zip file. The configuration file is named `elasticsearch-prod-{digest}.zip` and is located in the `extra` folder as well. Afterwards reload the bundle with the new configuration.

### Customized Elasticsearch

You can configure ConductR to use an alternate Elasticsearch cluster for events and logging. Here are some considerations for you if you should choose this path.

Firstly you must tell ConductR where your customized Elasticsearch instance is within its `application.ini`:

```
-Dcontrail.syslog.server.host=<some-ip>
-Dcontrail.syslog.server.port=<some-port>
```

The Elasticsearch bundle that we provide has been configured to support back-pressure when receiving event and logging data from ConductR. By default Elasticsearch will accept bulk index requests regardless of whether it will process them. This means that under certain load conditions, Elasticsearch could lose data being sent to it. To counter this, here is the configuration we use for Elasticsearch (we have chosen a sharding factor of 5, substitute yours accordingly):

```
threadpool.bulk.type: fixed
threadpool.bulk.queue_size: 5
```

The goal of the above settings is for Elasticsearch to reject bulk index requests if it does not have the resources to process them immediately. In the case of ConductR as a provider of bulk index messages, ConductR will buffer its messages until Elasticsearch is ready to process them. ConductR will also roll up messages within its buffer and prioritize them by severity (lowest priority messages are rolled up first).

Here are some cluster settings to consider for Elasticsearch:

```
discovery.zen.minimum_master_nodes: <size-of-es-cluster / 2 + 1>
index.number_of_shards: 5
index.number_of_replicas: <size-of-cluster - 1>
```

Finally, we recommend that you consider data retention of events and logging data. Here is an Elasticsearch template we have used:

```json
{
  "conductr" : {
    "template" : "conductr",
    "mappings" : {
      "_default_" : {
        "_ttl" : {
          "enabled" : true,
          "default": "14d"
        }
      }
    }
  }
}
```

More information on configuring Elasticsearch for production can be found in [the Elasticsearch documentation](https://www.elastic.co/guide/en/elasticsearch/guide/current/deploy.html).

## Setting up Papertail

A popular cloud service is [Papertrail](https://papertrailapp.com/). Papertrail is a very simple "tail like" service for viewing distributed logs. Once you configure an account with Papertrail, you will be provided with a host and a port. With this information you can configure a static endpoint. 

**Important**: ConductR logs over TCP so make sure that you configure papertrail so that it accepts plain text connections: _Accounts/Log Destinations/Edit Settings/Accept connections via_

Supposing that the address assigned to your at Papertrail is `logs2.papertrailapp.com` and `38564`  you configure ConductR as:

``` bash
echo \
  -Dcontrail.syslog.server.host=logs2.papertrailapp.com \
  -Dcontrail.syslog.server.port=38564 | \
  sudo tee -a /usr/share/conductr/conf/application.ini
sudo /etc/init.d/conductr restart
```

You can also apply a similar configuration to `conductr-haproxy` by substituting `conductr`.

## Other solutions

ConductR is compatible with any log aggregator speaking the syslog protocol. The log messages of a bundle are written to `stdout` and `stderr`. When using another logging infrastructure we recommend to deploy this infrastructure inside the ConductR cluster. You do not want to send lots of log traffic across the internet. Another approach is to use a syslog collector such as [rsyslog](http://www.rsyslog.com/) to filter the log messages before sending them to the logging cloud service.

**Important**: ConductR logs over TCP so make sure that configure your logging infrastructure accordingly.