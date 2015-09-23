# Logging

When multiple machines are involved in a cluster it quickly becomes difficult to view the log files of distributed applications. ConductR allows the logging output of itself and the bundles that it executes to be directed to a "syslog collector". Syslog is a widely used protocol for Unix based machines and is supported by a number of cloud-based log providers, as well as local operating system support.

The syslog collector can send the log messages to any kind of logging solution. The ConductR distribution includes Elasticsearch and Kibana as an opt-in logging infrastructure. How to configure Elasticsearch and Kibana or other popular logging solutions are described in the next sections.


## Setting up Elasticsearch

Elasticsearch is available as the `conductr-elasticsearch` bundle and can be found in the `extra` folder inside the ConductR installation folder. Also a default configuration for a typical production environment has been provided. To load and run Elasticsearch to ConductR use the control API of ConductR, e.g. by using the CLI:

```bash
conduct load file:/usr/share/conductr/extra/conductr-elasticsearch-{version}-{digest}.zip elasticsearch-prod-digest.zip
conduct run conductr-elasticsearch
```

`conductr-elasticsearch` is using the the role `elasticsearch`.

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