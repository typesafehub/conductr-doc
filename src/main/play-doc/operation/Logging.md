# Logging

When multiple machines are involved in a cluster it quickly becomes difficult to view the log files of distributed applications. ConductR allows the logging output of itself and the bundles that it executes to be directed to a "syslog collector". Syslog is a widely used protocol for Unix based machines and is supported by a number of cloud-based log providers, as well as local operating system support.

We expect that you will install a syslog collector such as [rsyslog](http://www.rsyslog.com/) within ConductR's network. You do not want to send lots of log traffic across the internet as it should be at least filtered first. The collector should be configured to accept TCP connections.

However to quickly demonstrate consolidated logging, a cloud service named [Papertrail](https://papertrailapp.com/) can be used. Papertrail is a very simple "tail like" service for viewing distributed logs. We will use it to show how to configure the simplest possible static endpoint. Once you configure an account with Papertrail then you will be provided with a host and a port. 

**Important**: ConductR logs over TCP so make sure that you configure papertrail so that it accepts plain text connections (tip: _Accounts/Log Destinations/Edit Settings/Accept connections via_).

Supposing that the address assigned to your at Papertrail is `logs2.papertrailapp.com` and `38564` then you configure ConductR like so:

``` bash
echo \
  -Dcontrail.syslog.server.host=logs2.papertrailapp.com \
  -Dcontrail.syslog.server.port=38564 | \
  sudo tee -a /usr/share/conductr/conf/application.ini
sudo /etc/init.d/conductr restart
```

You can also apply a similar configuration to `conductr-haproxy` by substituting `conductr`.

## Other hosted solutions

As previously stated you should use a hosted solution such as rsyslog in general. You may also want to investigate [Logstash](http://logstash.net/) running with [Kibana](https://www.elastic.co/products/kibana) and [Elasticsearch](https://www.elastic.co/products/elasticsearch) as an interesting hosted log analysis infrastructure. Of course they all provide syslog adapters, so can be easily used together with Contrail.

Generally speaking Contrail is compatible with any log aggregator speaking the Syslog protocol.
