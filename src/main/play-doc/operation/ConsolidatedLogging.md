# Consolidated Logging

When multiple machines are involved in a cluster, it quickly becomes difficult to view the log files of distributed applications.
 ConductR allows the logging output of itself and the bundles that it executes to be directed to a "syslog collector".
 Syslog is a widely used protocol for UNIX-based machines and is supported by a number of cloud-based log providers, as well as local operating system support.

The syslog collector can send the log messages to any kind of logging solution. ConductR provides bundles for Elasticsearch
 and Kibana as an opt-in logging infrastructure. How to configure Elasticsearch and Kibana or other popular logging solutions
 are described in the next sections.

## Logging Structure

Before discussing the types of logging available to you, it will be useful to understand the nature of ConductR logs.
 There are 3 types of logs:

1. Bundle events
2. Bundle logs
3. ConductR logs

All of these types of logs will be sent to the one log collector. They are distinguished given
 [Syslog's definition of structured data](https://tools.ietf.org/html/rfc5424) and ConductR's usage of it.

The following sub-sections describe each log type and how they are distinguished.

### Bundle Events

Bundle events describe what has happened to a bundle in terms of whether it has loaded, been replicated to a node,
 scaled up or down, whether resources cannot be found to scale and so forth. The following structured data items
 determine that a log message represents a bundle event:

* `data.mdc@49285.bundleId:$bundleId` OR `data.mdc@49285.bundleId:$bundleName`
* `data.mdc@49285.tag:conductr`

Note that `$bundleName` is used for some ConductR events where there is no bundle identifier available e.g. when loading a bundle.

### Bundle Logs

Bundle logs provide the stdout and stderr output of your bundle and are identified in a similar manner to events,
 only there will be no "conductr" tag:

* `data.mdc.bundleId:$bundleId`
* Missing `data.mdc.tag:conductr`

Note also that the severity level of this log message will indicate `INFO` for stdout and `ERROR` for stderr.

### ConductR Logs

ConductR's logs are always identified given the absence of a bundle identifier tag.

* Missing (`data.mdc.bundleId:$bundleId` AND `data.mdc.bundleName:$bundleName`)

## Setting up Elasticsearch

The setup of Elasticsearch depends on the ConductR mode or if you want to use an alternate Elasticsearch cluster outside of ConductR. Please choose one of the possible setup options:

* [ConductR Standalone mode](#Elasticsearch-on-Standalone-ConductR-cluster)
* [ConductR DC/OS mode](#Elasticsearch-on-DC/OS-cluster)
* [External Elasticsearch cluster](#External-Elasticsearch-cluster)

### Elasticsearch on Standalone ConductR cluster

To deploy Elasticsearch to your standalone ConductR cluster, use the `conductr-elasticsearch` bundle.
 This bundle is hosted in the [Typesafe bundles](https://bintray.com/typesafe/bundle/conductr-elasticsearch) repository on Bintray.
 The CLI will resolve the bundle from Bintray when specifying the bundle name `conductr-elasticsearch` during `conduct load`.
 The `conductr-elasticsearch` bundle will run as a single node without any configuration.
 A clustering configuration more typical of a production environment has been provided in the [bundle-configuration](https://bintray.com/typesafe/bundle-configuration/elasticsearch-prod) repository.
 One should run multiple instances when using production mode to avoid data loss.

`conductr-elasticsearch` is using the the role `elasticsearch`. Make sure that the ConductR Agent nodes which should run Elasticsearch have this role assigned in `conductr-agent.ini`. This role will determine which nodes will be eligible to run `conductr-elasticsearch` and are to be configured accordingly.

Firstly, for each node that will run Elasticsearch you must enable access to `/var/log` and `/var/lib`:

```bash
sudo mkdir -p /var/lib/elasticsearch /var/log/elasticsearch
sudo chown conductr-agent:conductr-agent /var/lib/elasticsearch
sudo chown conductr-agent:conductr-agent /var/log/elasticsearch
```

To load and run Elasticsearch use the control API of ConductR, e.g. by using the CLI:

```bash
conduct load conductr-elasticsearch elasticsearch-prod
conduct run conductr-elasticsearch
```

With that, the syslog collector streams the log messages to Elasticsearch. Use the CLI to access log messages by bundle id or name, e.g.:

```bash
conduct logs my-bundle
```

#### Configuration

The provided Elasticsearch bundle configuration is using these settings:
- Memory: 4 GB
- Number of CPUs: 2
- Disk Space: 10 GB

To change the settings create a new bundle configuration by modifying the `bundle.conf` file inside of the bundle configuration zip file. Afterward, reload the bundle with the new configuration.

#### Elasticsearch bundle rolling upgrade

Elasticsearch stores its data within the filesystem. As such, to preserve existing data when managing the deployment, it's important to ensure the data is replicated to the new instance of Elasticsearch.

This can be achieved by following these steps.

* Commission a new ConductR node where the new Elasticsearch instance is going to execute.
* It is assumed the new ConductR node will run both ConductR Core and ConductR Agent process.
* Start this new node and ensure the ConductR Agent on this node is able to successfully join the existing ConductR cluster.
* Turn off one of the old instances where Elasticsearch is running.
* Wait until the new Elasticsearch instance joins the cluster successfully and the data replicated.
  * Access the Elasticsearch cluster health endpoint: `http://<ip address of new node>:9200/elastic-search/_cluster/health?level=shards&pretty=true`
  * Note the state of the cluster health (i.e. the `status` field).
  * It is expected for the cluster health to move from `green` -> `yellow` -> `green` while the new node is joining the cluster. It's important to wait and give sufficient time for the cluster health state to turn green. When the old instance is turned off, there will be a slight delay before the cluster state turns to `yellow`. Similarly, when a new instance joins the cluster there will be a slight delay before the cluster state turns from `yellow` to `green`.
  * Once the cluster join process is completed, the cluster health will stay at `green`.
  * The number of nodes (i.e. `number_of_nodes` field) shows the correct total number of nodes including the newly joined instance.
  * All shards are assigned (i.e. the value of `unassigned_shards` is `0`) - all shards being assigned indicates the data is now replicated across the Elasticsearch nodes.
* Repeat with the remaining nodes until every old node has been decommissioned.

Generally, Elasticsearch will be resilient enough in the face of failure as long as there is `(n + 1) / 2` remaining nodes available, where `n` is the total number of nodes. So as an example, in the cluster of 5 nodes, ES should be able to cope with losing two nodes but not three.

It's important to note the rolling upgrade of the Elasticsearch bundle is only possible between ConductR versions which are binary compatible.

For migrating between binary incompatible ConductR versions, the Elasticsearch data needs to be moved to the new cluster to preserve existing data. The simplest way to do this is to copy the contents of the `/var/lib/elasticsearch` from the old nodes to the new nodes. Alternatively, Elasticsearch provides [backup and restore](https://www.elastic.co/guide/en/elasticsearch/reference/1.5/modules-snapshots.html) facility to move data between 2 different clusters.

### Elasticsearch on DC/OS cluster

On DC/OS, it is recommended to use the corresponding DC/OS service instead of running Elasticsearch inside ConductR. Install the Elasticsearch service from the DC/OS universe with the DC/OS CLI:

```bash
dcos package install elasticsearch
```

Elasticsearch is started via Marathon. The status of the Elasticsearch cluster can be checked with:

```bash
dcos marathon task list | awk '{print $5}' | grep elasticsearch | head -n1 | xargs dcos marathon task show
{
  "appId": "/elasticsearch",
  "healthCheckResults": [
    {
      "alive": true,
      "consecutiveFailures": 0,
      "firstSuccess": "2016-12-14T13:12:03.634Z",
      "lastFailure": null,
      "lastFailureCause": null,
      "lastSuccess": "2016-12-14T14:01:05.342Z",
      "taskId": "elasticsearch.d2c66494-c1fe-11e6-98b2-4ee48a083375"
    }
  ],
  "host": "10.0.1.77",
  "id": "elasticsearch.d2c66494-c1fe-11e6-98b2-4ee48a083375",
  "ipAddresses": [
    {
      "ipAddress": "10.0.1.77",
      "protocol": "IPv4"
    }
  ],
  "ports": [
    31105
  ],
  "servicePorts": [
    31105
  ],
  "slaveId": "9df6d049-6b71-4e7e-bc9b-0b5b5b2ec489-S0",
  "stagedAt": "2016-12-14T13:11:28.346Z",
  "startedAt": "2016-12-14T13:11:29.230Z",
  "state": "TASK_RUNNING",
  "version": "2016-12-14T13:11:28.319Z"
}
```

The Elasticsearch cluster has successfully started if the state is equal to `TASK_RUNNING`.

By default, ConductR automatically writes the log messages and events to the Elasticsearch service. It uses the DNS SRV record of the service to resolve it.

```
-Dconductr.service-locator-server.external-service-addresses.elastic-search.0="http+srv://_client-port._elasticsearch-executor._tcp.elasticsearch.mesos"
```

In case you modify the name of the Elasticsearch service, please override the above configuration key in the ConductR service configuration accordingly.

If you're using the `elastic` package instead of `elasticsearch`, note that the service name has changed. You'll need to specify the address of Elasticsearch in your Marathon configuration (under the `cmd` key):

```
-Dconductr.service-locator-server.external-service-addresses.elastic-search.0=http+srv://_coordinator-0-node._tcp.elastic.mesos
```

### External Elasticsearch cluster

You can configure ConductR to use another Elasticsearch cluster for events and logging.
 Here are some considerations for you if you should choose this path.

Specify the ingest node bulk endpoint of your customized Elasticsearch instance in the ConductR Core configuration file:

```bash
echo \
  -Dconductr.service-locator-server.external-service-addresses.elastic-search.0=https://<user>:<password>@<elasticsearch-host>:443/_bulk/ \
  sudo tee -a /usr/share/conductr/conf/conductr.ini
sudo service conductr restart
```

The Elasticsearch bundle that we provide in Standalone mode has been configured to support back-pressure when receiving
 event and logging data from ConductR. By default, Elasticsearch will accept bulk index requests regardless of whether
 it will process them. This means that under certain load conditions, Elasticsearch could lose data being sent to it.
 To counter this, here is the configuration we use for Elasticsearch (we have chosen a sharding factor of 5, substitute yours accordingly):

```
threadpool.bulk.type: fixed
threadpool.bulk.queue_size: 5
```

The goal of the above settings is for Elasticsearch to reject bulk index requests if it does not have the resources to
 process them immediately. In the case of ConductR as a provider of bulk index messages, ConductR will buffer its
 messages until Elasticsearch is ready to process them. ConductR will also roll up messages within its buffer and
 prioritize them by severity (lowest priority messages are rolled up first).

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

## Setting up Kibana

Kibana is a popular UI to display data stored in Elasticsearch. In the context of ConductR, Kibana can be configured to display, filter and search log messages. The setup of Kibana depends on the ConductR mode. Please choose one of the possible setup options:

* [ConductR Standalone mode](#Kibana-on-Standalone-ConductR-cluster)
* [ConductR DC/OS mode](#Kibana-on-DC/OS-cluster)

### Kibana on Standalone ConductR cluster

To deploy Kibana onto a ConductR standalone cluster, use the `conductr-kibana` bundle. This bundle is hosted on Bintray in the [Typesafe bundles](https://bintray.com/typesafe/bundle/conductr-kibana) repository and is automatically resolved when using the CLI. The bundle uses the version 4.1.2 of Kibana. It only works in conjunction with the `conductr-elasticsearch` bundle. To load and run Kibana on ConductR use the control API of ConductR, e.g. by using the CLI:

```bash
conduct load conductr-kibana
conduct run conductr-kibana
```

This bundle doesn't require any additional bundle configuration file. It is using the role `kibana`. Make sure that the ConductR nodes which should run Kibana have this role assigned.

Now the Kibana UI can be accessed on the port `5601`, e.g.: http://192.168.10.1:5601 (this is the default address on the developer sandbox - substitute the address of your proxy appropriately).

[[images/kibana_index_initial.png]]

### Kibana on DC/OS cluster

On DC/OS, it is recommended to use the corresponding DC/OS service instead of running Kibana inside ConductR. Install the Kibana service from the DC/OS universe with the DC/OS CLI:

```bash
dcos package install kibana
```

Kibana is started via Marathon. The status of the Kibana cluster can be checked with:

```bash
dcos marathon task list | awk '{print $5}' | grep kibana | head -n1 | xargs dcos marathon task show
{
  "appId": "/kibana",
  "healthCheckResults": [
    {
      "alive": true,
      "consecutiveFailures": 0,
      "firstSuccess": "2016-12-14T13:10:56.142Z",
      "lastFailure": null,
      "lastFailureCause": null,
      "lastSuccess": "2016-12-14T14:48:59.980Z",
      "taskId": "kibana.bc8ab643-c1fe-11e6-98b2-4ee48a083375"
    }
  ],
  "host": "10.0.1.80",
  "id": "kibana.bc8ab643-c1fe-11e6-98b2-4ee48a083375",
  "ipAddresses": [
    {
      "ipAddress": "10.0.1.80",
      "protocol": "IPv4"
    }
  ],
  "ports": [
    5601
  ],
  "servicePorts": [
    5601
  ],
  "slaveId": "cb77bbc5-7a93-4eb0-a8d6-150729a210d8-S1",
  "stagedAt": "2016-12-14T13:10:51.045Z",
  "startedAt": "2016-12-14T13:10:51.963Z",
  "state": "TASK_RUNNING",
  "version": "2016-12-14T13:10:51.016Z"
}
```

The Kibana cluster has successfully started if the state is equal to `TASK_RUNNING`. Once running, the Kibana UI is accessible at `http://dcos-host/app/kibana`.

### Connecting Kibana with Elasticsearch

In order to display the log messages from Elasticsearch in Kibana an `index pattern` need to be created initially. If no index pattern has been created yet, the Kibana UI is redirecting you to the page to configure it. All log messages in Elasticsearch are stored in the index `conductr`. Therefore, create in Kibana the index pattern `conductr`. As the `Time-field name` select `header.timestamp`.

[[images/kibana_index_configuration.png]]

The newly created index pattern is automatically set to the default one. Now, head over to the `Discover` tab to view the log messages.

[[images/kibana_discover.png]]

By default, the log messages of the last 15 minutes are displayed. If ConductR or the application bundles haven't produced any log messages during this timeframe, no messages will be displayed. In this case, you can adjust the timeframe on the top right.

### Customizing Discover tab

By default, Kibana displays in the `Discover` tab all fields which Elasticsearch has been stored in the index. Some of these fields don't contain helpful information to debug log messages. Therefore, we recommend selecting only the fields you are interested in. As a start, you can download and import this custom search.

<a href="resources/operation/files/conductr_default_search.json" download="conductr_default_search.json">Download ConductR Default Search</a>

In Kibana, select the `Settings` tab and go to `Objects`. Click on `Import` and select the downloaded `conductr_default_search.json` file. This imports the custom search. Now click on the `view` button to see the selected fields in action.

[[images/kibana_custom_search.png]]

The `Discover` tab has now selected fields in the left field pane. Also, these fields are selected as columns in the log message pane.

[[images/kibana_discover_custom.png]]

The `conductr_default_search` custom search selects these fields:

Name                     | Description
-------------------------|---------------------
Time                     | Timestamp
header.pri.severity      | Log level
data.mdc@49285.bundleId  | Id of the bundle
message                  | Log message
data.mdc@49285.class     | Application class which produced the log message
header.hostname          | ConductR node
data.mdc@49285.requestId | Unique log message request id

### Filtering log messages

Every field can be used to filter log messages. You can also apply multiple filters.

#### Filter by bundle

To filter the log messages by a bundle, enter the following search string into the search field. Replace `$bundleId` with the respective bundle id:

```
data.mdc@49285.bundleId:$bundleId AND _missing_:data.mdc@49285.tag
```

[[images/kibana_filter_by_bundle.png]]

#### Filter by ConductR log messages

To select the log messages from the ConductR core and agents nodes itself, filter by a missing bundleId and bundleId:

```
_missing_:data.mdc@49285.bundleId AND _missing_:data.mdc@49285.bundleName
```

[[images/kibana_filter_by_conductr.png]]

## Setting up RSYSLOG

ConductR logs via the syslog protocol using TCP destined conventionally on port 514.
 Debian distributions such as Ubuntu come with the [RSYSLOG](http://www.rsyslog.com/) logging service and so its configuration is shown next. Other distributions may require installing RSYSLOG.

To configure ConductR Core for RSYSLOG:

```bash
echo \
  -Dconductr.service-locator-server.external-service-addresses.elastic-search.0=http://127.0.0.1:514 \
  sudo tee -a /usr/share/conductr/conf/conductr.ini
sudo service conductr restart
```

...and to configure RSYSLOG:

```bash
echo '$ModLoad imtcp' | sudo tee -a /etc/rsyslog.d/conductr.conf
echo '$InputTCPServerRun 514' | sudo tee -a /etc/rsyslog.d/conductr.conf
sudo service rsyslog restart
```

Viewing `/var/log/syslog` (Ubuntu) or `/var/log/messages` (RHEL) will then show ConductR and bundle output.

## Setting up Humio

A popular cloud service is [Humio](https://humio.com/). Humio is a log management service for developers
 that is "like tail and grep with aggregations and graphs built-in". Once you create an account with Humio, you will be provided with a host and ingest token.
 With this information, you can configure a static endpoint.

Supposing that the host assigned to your at Humio is `go.humio.com` you configure ConductR Core as:


```bash
echo \
  -Dconductr.service-locator-server.external-service-addresses.elastic-search.0=https://<ingest token>@go.humio.com:443/api/v1/dataspaces/<dataspace>/ingest/elasticsearch/ \
  sudo tee -a /usr/share/conductr/conf/conductr.ini
sudo service conductr restart
```
Where `<ingest token>` is your Humio `ingest token` and `<dataspace>` the name of your `dataspace`.
 See the [Humio Doc](https://go.humio.com/docs/integrations/log-shippers/logstash/index.html) for further details.

## Other solutions

ConductR is compatible with any log aggregator speaking the syslog protocol.
 The log messages of a bundle are written to `stdout` and `stderr`. When using another logging infrastructure we recommend
 to deploy this infrastructure inside the ConductR cluster. You do not want to send lots of log traffic across the internet.
 Another approach is to use a syslog collector such as [rsyslog](http://www.rsyslog.com/) to filter the log messages before sending them to the logging cloud service.

## Controlling ConductR log level

By default, ConductR will log at `info` level.

To view ConductR Core logs at `debug` level, configure ConductR as:

```bash
echo \
  -Dakka.loglevel=debug | \
  sudo tee -a /usr/share/conductr/conf/conductr.ini
sudo service conductr restart
```

similarly for the ConductR Agent:

```bash
echo \
  -Dakka.loglevel=debug | \
  sudo tee -a /usr/share/conductr-agent/conf/conductr-agent.ini
sudo service conductr-agent restart
```

With this setting only ConductR `debug` level logs will be visible. In other words, `debug` level messages from frameworks and libraries utilized by ConductR will not be visible.

To view all `debug` level log messages, configure ConductR Core as:

```bash
echo \
  -Droot.loglevel=debug \
  -Dakka.loglevel=debug | \
  sudo tee -a /usr/share/conductr/conf/conductr.ini
sudo service conductr restart
```

similarly for the ConductR Agent:

```bash
echo \
  -Droot.loglevel=debug \
  -Dakka.loglevel=debug | \
  sudo tee -a /usr/share/conductr-agent/conf/conductr-agent.ini
sudo service conductr-agent restart
```

With this setting debug messages from various frameworks and libraries utilized by ConductR will be visible, e.g. debug messages from Akka.

**Important**: with this setting enabled, the number of log messages generated will increase dramatically.
