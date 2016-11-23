# Consolidated Logging

When multiple machines are involved in a cluster it quickly becomes difficult to view the log files of distributed applications. ConductR allows the logging output of itself and the bundles that it executes to be directed to a "syslog collector". Syslog is a widely used protocol for Unix based machines and is supported by a number of cloud-based log providers, as well as local operating system support.

The syslog collector can send the log messages to any kind of logging solution. The ConductR distribution includes Elasticsearch and Kibana as an opt-in logging infrastructure. How to configure Elasticsearch and Kibana or other popular logging solutions are described in the next sections.

## Logging Structure

Before discussing the types of logging available to you, it will be useful to understand the nature of ConductR logs. There are 3 types of logs:

1. bundle events;
2. bundle logs; and
3. ConductR's own logs.

All of these types of logs will be sent to the one log collector. They are distiguished given [Syslog's definition of structured data](https://tools.ietf.org/html/rfc5424) and ConductR's usage of it. 

The following sub-sections describe each log type and how they are distinguished.

### Bundle Events

Bundle events describe what has happened to a bundle in terms of whether it has loaded, been replicated to a node, scaled up or down, whether resources cannot be found to scale and so forth. The following structured data items determine that a log message represents a bundle event:

* data.mdc.bundleId:`$bundleId`* OR data.mdc.bundleId:`$bundleName`
* data.mdc.tag:conductr*

Note that `$bundleName` is used for some ConductR events where there is no bundle identiier available e.g. when loading a bundle.

### Bundle Logs

Bundle logs provide the stdout and stderr output of your bundle and are identified in a similar manner to events, only there will be no "conductr" tag:

* data.mdc.bundleId:`$bundleId`*
* missing data.mdc.tag:conductr*

Note also that the severity level of this log message will indicate `INFO` for stdout and `ERROR` for stderr.

### ConductR's Logs

ConductR's logs are always identified given the absence of a bundle identifier tag.

* missing (data.mdc.bundleId:`$bundleId`* AND data.mdc.bundleId:`$bundleName`*)

## Setting up Elasticsearch

Elasticsearch is available either as the `conductr-elasticsearch` bundle or you can [use your own](#Customized-Elasticsearch). The provided bundle can be found in the `extra` folder inside the ConductR installation folder. Also a default configuration for a typical production environment has been provided.

`conductr-elasticsearch` is using the the role `elasticsearch`. Make sure that the ConductR nodes which should run Elasticsearch have this role assigned in `application.ini`. This role will determine which nodes will be eligable to run `conductr-elasticsearch` and are to be configured accordingly.

Firstly for each node that will run Elasticsearch you must enable access to `/var/log` and `/var/lib`:

```bash
sudo mkdir -p /var/lib/elasticsearch /var/log/elasticsearch
sudo chown conductr:conductr /var/lib/elasticsearch
sudo chown conductr:conductr /var/log/elasticsearch
```

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

### Elasticsearch bundle rolling upgrade

Elasticsearch stores its data within the filesystem. As such, to preserve existing data when managing the deployment, it's important to ensure the data is replicated to the new instance of Elasticsearch.

This can be achieved by following these steps.

* Commission a new ConductR node where the new Elasticsearch instance is going to execute.
* Start this new node and ensure this node is able to successfully join the existing ConductR cluster.
* Turn off one of the old instance where Elasticsearch is running.
* Wait until the new Elasticsearch instance joins the cluster successfully and the data replicated.
  * Access the Elasticsearch cluster health endpoint: `http://<ip address of new node>:9200/elastic-search/_cluster/health?level=shards&pretty=true`
  * Note the state of the cluster health (i.e. the `status` field).
  * It is expected for the cluster health to move from `green` -> `yellow` -> `green` while the new node is joining the cluster. It's important to wait and give sufficient time for the cluster health state to turn green. When the old instance is turned off, there will be a slight delay before the cluster state turns to `yellow`. Similarly, when a new instance joins the cluster there will be a slight delay before the cluster state turns from `yellow` to `green`.
  * Once the cluster join process is completed, the cluster health will stay at `green`.
  * The number of node (i.e. `number_of_nodes` field) shows the correct total number of nodes including the newly joined instance.
  * All shards are assigned (i.e. the value of `unassigned_shards` is `0`) - all shards being assigned indicates the data is now replicated across the Elasticsearch nodes.
* Repeat with the remaining nodes until every old node has been decommissioned.

Generally, Elasticsearch will be resilient enough in the face of failure as long as there is `(n + 1) / 2` remaining nodes available, where `n` is the total number of nodes. So as an example, in the cluster of 5 nodes, ES should be able to cope with losing two nodes but not three.

It's important to note the rolling upgrade of the Elasticsearch bundle is only possible between ConductR versions which are binary compatible.

For migrating between binary incompatible ConductR versions, the Elasticsearch data needs to be moved to the new cluster to preserve existing data. The simplest way to do this is to copy the contents of the `/var/lib/elasticsearch` from the old nodes to the new nodes. Alternatively, Elasticsearch provides [backup and restore](https://www.elastic.co/guide/en/elasticsearch/reference/1.5/modules-snapshots.html) facility to move data between 2 different clusters.

### Customized Elasticsearch

You can configure ConductR to use an alternate Elasticsearch cluster for events and logging. Here are some considerations for you if you should choose this path.

Firstly you must tell ConductR where your customized Elasticsearch instance is within its `application.ini`, and also turn off ConductR service locating it given the fixed location:

```
-Dcontrail.syslog.server.host=<some-ip>
-Dcontrail.syslog.server.port=<some-port>
-Dcontrail.syslog.server.service-locator.enabled=off
```

The same also goes for conductr-haproxy's `application.ini`.

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

## Setting up Kibana

Kibana is a popular UI to display data stored in Elasticsearch. In the context of ConductR, Kibana can be configured to display, filter and search log messages. It is available as the `conductr-kibana` bundle and can be found in the `extra` folder inside the ConductR installation folder. The version 4.1.2 of Kibana is used. This bundle only works in conjunction with the `conductr-elasticsearch` bundle. To load and run Kibana to ConductR use the control API of ConductR, e.g. by using the CLI:

```bash
conduct load file:/usr/share/conductr/extra/conductr-kibana-{version}-{digest}.zip
conduct run conductr-kibana
```

This bundle doesn't require any additional bundle configuration file. It is using the role `kibana`. Make sure that the ConductR nodes which should run Kibana have this role assigned.

Now the Kibana UI can be accessed on the port `5601`, e.g.: http://192.168.59.103:5601.

[[images/kibana_index_initial.png]]

### Connecting Kibana with Elasticsearch

In order to display the log messages from Elasticsearch in Kibana an `index pattern` need to be created initially. If no index pattern has been created yet, the Kibana UI is redirecting you to the page to configure it. All log messages in Elasticsearch are stored in the index `conductr`. Therefor, create in Kibana the index pattern `conductr`. As the `Time-field name` select `header.timestamp`.

[[images/kibana_index_configuration.png]]

The newly created index pattern is automatically set to the default one. Now, head over to the `Discover` tab to view the log messages.

[[images/kibana_discover.png]]

By default, the log messages of the last 15 minutes are displayed. If ConductR or the application bundles haven't produced any log messages during this timeframe, no messages will be displayed. In this case you can adjust the timeframe on the top right.

### Customizing Discover tab

By default, Kibana displays in the `Discover` tab all fields which Elasticsearch has been stored in the index. Some of these fields doesn't contain helpful information to debug log messages. Therefor, we recommend to select only the fields you are intersted in. As a start you can download and import this custom search.

<a href="resources/operation/files/conductr_default_search.json" download="conductr_default_search.json">Download ConductR Default Search</a>

In Kibana, select the `Settings` tab and go to `Objects`. Click on `Import` and select the downloaded `conductr_default_search.json` file. This imports the custom search. Now click on the `view` button to see the selected fields in action.

[[images/kibana_custom_search.png]]

The `Discover` tab has now selected fields in the left field pane. Also these fields are selected as columns in the log message pane.

[[images/kibana_discover_custom.png]]

The `conductr_default_search` custom search selects these fields:

Name                | Description
--------------------|---------------------
Time                | Timestamp
header.pri.severity | Log level
data.mdc.bundleName | Name of the bundle. The log messages of ConductR itself do not contain any bundle name.
message             | Log message
data.mdc.class      | Application class which produced the log message
header.hostname     | ConductR node
data.mdc.requestId  | Unqiue log message request id

### Filtering log messages

Every field can be used to filter log messages. You can also apply multiple filters. To filter the log messages by a bundle add the filter `data.mdc.bundleName` to the search by clicking on `data.mdc.bundleName` in the left field pane. This displays the available bundles you can filter on. Select a bundle by clicking on the respective magnifier icon.

[[images/kibana_filter_by_bundle.png]]

To select only the log messages from ConductR itself, filter again by bundle name. This time, create a query where the bundle name is `null` by entering `data.mdc.bundleName is null` into the search field:

[[images/kibana_filter_by_conductr.png]]

## Setting up RSYSLOG

ConductR logs via the syslog protocol using TCP destined conventionally on port 514. Debian distributions such as Ubuntu come with the [RSYSLOG](http://www.rsyslog.com/) logging service and so its configuration is shown next. Other distributions may require installing RSYSLOG.

To configure ConductR for RSYSLOG:

``` bash
echo \
  -Dcontrail.syslog.server.host=127.0.0.1 \
  -Dcontrail.syslog.server.port=514 \
  -Dcontrail.syslog.server.elasticsearch.enabled=off | \
  sudo tee -a /usr/share/conductr/conf/application.ini
sudo /etc/init.d/conductr restart
```

The same also goes for conductr-haproxy's `application.ini` and service.

...and to configure RSYSLOG:

``` bash
[172.17.0.1]$ echo '$ModLoad imtcp' | sudo tee -a /etc/rsyslog.d/conductr.conf
[172.17.0.1]$ echo '$InputTCPServerRun 514' | sudo tee -a /etc/rsyslog.d/conductr.conf
[172.17.0.1]$ sudo service rsyslog restart
```

Viewing `/var/log/syslog` (Ubuntu) or `/var/log/messages` (RHEL) will then show ConductR and bundle output.

## Setting up Papertrail

A popular cloud service is [Papertrail](https://papertrailapp.com/). Papertrail is a very simple "tail like" service for viewing distributed logs. Once you configure an account with Papertrail, you will be provided with a host and a port. With this information you can configure a static endpoint.

**Important**: ConductR logs over TCP so make sure that you configure papertrail so that it accepts plain text connections: _Accounts/Log Destinations/Edit Settings/Accept connections via_

Supposing that the address assigned to your at Papertrail is `logs2.papertrailapp.com` and `38564`  you configure ConductR as:

``` bash
echo \
  -Dcontrail.syslog.server.host=logs2.papertrailapp.com \
  -Dcontrail.syslog.server.port=38564 \
  -Dcontrail.syslog.server.elasticsearch.enabled=off | \
  sudo tee -a /usr/share/conductr/conf/application.ini
sudo /etc/init.d/conductr restart
```

You can also apply a similar configuration to `conductr-haproxy` by substituting `conductr`.

## Other solutions

ConductR is compatible with any log aggregator speaking the syslog protocol. The log messages of a bundle are written to `stdout` and `stderr`. When using another logging infrastructure we recommend to deploy this infrastructure inside the ConductR cluster. You do not want to send lots of log traffic across the internet. Another approach is to use a syslog collector such as [rsyslog](http://www.rsyslog.com/) to filter the log messages before sending them to the logging cloud service.

**Important**: ConductR logs over TCP so make sure that configure your logging infrastructure accordingly.

## Controlling ConductR log level

By default ConductR will log at `info` level.

To view ConductR logs at `debug` level, configure ConductR as:

``` bash
echo \
  -Dakka.loglevel=debug | \
  sudo tee -a /usr/share/conductr/conf/application.ini
sudo /etc/init.d/conductr restart
```

The same also goes for conductr-haproxy's `application.ini` and service.

With this setting only ConductR `debug` level logs will be visible. In other words, `debug` level messages from frameworks and libraries utilized by ConductR will not be visible.

To view all `debug` level log messages, configure ConductR as:

``` bash
echo \
  -Droot.loglevel=debug \
  -Dakka.loglevel=debug | \
  sudo tee -a /usr/share/conductr/conf/application.ini
sudo /etc/init.d/conductr restart
```

The same also goes for conductr-haproxy's `application.ini` and service.

With this setting debug messages from various frameworks and libraries utilized by ConductR will be visible, e.g. debug messages from Akka.

**Important**: with this setting enabled, the number of log messages generated will be increase drammatically.
