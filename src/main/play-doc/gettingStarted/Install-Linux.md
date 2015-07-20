# Typesafe ConductR %PLAY_VERSION%

## Installation

This is a tutorial for installing ConductR and shows how this is done for a small cluster of 3 machines.

### Requirements

The requirements of a ConductR host are:

* Debian or Rpm based system (recommended: Ubuntu 14.04 LTS or RHEL/CentOS 6.6)
* Oracle Java Runtime Environment 8 (JRE 8)
* Python 3.4
* Debian or Rpm package of ConductR

#### Installing JRE 8

Install Java 8 as the default JRE. You will need to accept the Oracle license agreement.

```bash
sudo add-apt-repository -y ppa:webupd8team/java
sudo apt-get update
sudo apt-get -y install oracle-java8-installer && sudo apt-get clean
sudo apt-get -y install oracle-java8-set-default
echo "JAVA_HOME=/usr/lib/jvm/java-8-oracle" | sudo tee -a /etc/environment
```

#### Optional dependencies

* Docker - for running bundles inside a Docker container

### Installing ConductR on the first machine

This tutorial uses three systems with the addresses `172.17.0.{1,2,3}`. To simplify the installation and configuration instructions, we are going to use the hostname command. Please ensure the hostname is set correctly or substitute your addresses as appropriate for $(hostname). To set the hostname, pass the ip address to hostname.

```bash
sudo hostname $(hostname -i)
```

The tutorial also assumes that you have obtained the `conductr_%PLAY_VERSION%_all.deb` Debian or `conductr_%PLAY_VERSION%-1.noarch.rpm` Rpm package.

Install ConductR as any other Debian or Rpm package.

``` bash
[172.17.0.1]$ sudo dpkg -i conductr_%PLAY_VERSION%_all.deb
```
or
``` bash
[172.17.0.1]$ sudo yum install conductr-%PLAY_VERSION%-1.noarch.rpm
```

ConductR is automatically registered as a service and started. ConductR provides cluster and application information as well as its control interface via a REST API.

``` bash
[172.17.0.1]$ curl -s 127.0.0.1:9005/members | python3 -m json.tool
```

A typical response contains the current members of the cluster (shown here as just one), the address of the node that the queried control server is running on and a list of unreachable nodes (shown here as empty).

``` json
{
  "members": [
    {
      "node": "akka.tcp://conductr@127.0.0.1:9004",
      "nodeUid": "-1595142725",
      "roles": [
        "all-conductrs"
      ],
      "status": "Up"
    }
  ],
  "selfNode": "akka.tcp://conductr@127.0.0.1:9004",
  "selfNodeUid": "-1595142725",
  "unreachable": []
}
```

The IP addresses in the response indicate that ConductR is listening to the `localhost` address. To be able to form an inter-machine cluster, ConductR must be configured to listen to the machine's host interface. This can be enabled adding a property declaration for `CONDUCTR_IP` to the start command as follows:

``` bash
[172.17.0.1]$ echo -DCONDUCTR_IP=$(hostname) | sudo tee -a /usr/share/conductr/conf/application.ini
[172.17.0.1]$ sudo service conductr restart
```

Check for the cluster information once again, but now use the host address of the machine.

``` bash
[172.17.0.1]$ curl -s $(hostname):9005/members | python3 -m json.tool
```

#### Installation miscellany

The ConductR service runs under the `conductr` user along with the `conductr` group. Its pid file is written to: `/var/run/conductr/running.pid` and its install location is `/usr/share/conductr`.

ConductR logs via the syslog protocol using TCP destined locally on port 514. Debian distributions such as Ubuntu come with the [RSYSLOG](http://www.rsyslog.com/) logging service and so its configuration is shown next. Other distributions may require installing RYSLOG.

``` bash
[172.17.0.1]$ echo '$ModLoad imtcp' | sudo tee -a /etc/rsyslog.d/conductr.conf
[172.17.0.1]$ echo '$InputTCPServerRun 514' | sudo tee -a /etc/rsyslog.d/conductr.conf
[172.17.0.1]$ sudo service rsyslog restart
```

Viewing `/var/log/syslog` (Debian) or `/var/log/messages` (Rpm) will then show ConductR and bundle output.

By default ConductR's logging is quite sparse. Unless an error or warning occurs then there will be no log output. To increase the verbosity of the logging you can use this command:

```bash
[172.17.0.1]$ echo -Dakka.loglevel=debug | sudo tee -a /usr/share/conductr/conf/application.ini
```

Consolidated logging is discussed further down.

#### Optional dependencies

##### Docker

ConductR supports running applications and services within Docker. If you plan on running Docker based bundles, you will need to install [Docker](https://docs.docker.com/) according to [the official documentation](https://docs.docker.com/installation/ubuntulinux/). Once Docker is installed then add ConductR's user/group to the `docker` group so that it has [the correct permissions in order to access Docker](http://docs.docker.com/installation/ubuntulinux/#giving-non-root-access):

``` bash
[172.17.0.1]$ sudo usermod -a -G docker conductr
```

### Installing ConductR on the remaining machines

_Repeat each step in this section also on the `172.17.0.3` machine._

First ensure that the following ports are available between the machines forming the cluster:

* 9004 - Akka remoting for ConductR
* 9006 - Bundle streaming between ConductR nodes
* 10000 to 10999 - the default range of ports allocated to bundle component endpoints

The node running on the `172.17.0.1` machine is called a seed node, which is a node that is going to be used as the initial contact point when joining a cluster. Install ConductR on the new machine and configure the address of a seed node:

``` bash
[172.17.0.2]$ sudo dpkg -i conductr_%PLAY_VERSION%_all.deb
```
or
```bash
[172.17.0.2]$ echo -DCONDUCTR_IP=$(hostname) | sudo tee -a /usr/share/conductr/conf/application.ini
[172.17.0.2]$ echo --seed 172.17.0.1:9004 | sudo tee -a /usr/share/conductr/conf/application.ini
[172.17.0.2]$ sudo service conductr restart
```

sudo yum install conductr_%PLAY_VERSION%-1.noarch.rpm

You should now see a new node in the cluster members list by using the following query:

``` bash
[172.17.0.2]$ curl -s 172.17.0.2:9005/members | python3 -m json.tool
```

Install optional dependencies if required. Each ConductR node requires same optional dependencies to be installed.

### Installing a Proxy

_Perform each step in this section on all nodes: `172.17.0.1`, `172.17.0.2` and `172.17.0.3`. For full resilience a proxy should be installed for each machine that ConductR is installed on._

Proxying application endpoints is required when running more than one instance of ConductR; which should be always for production style scenarios. Proxying endpoints permits connectivity from both external callers and for bundle components to communicate with other bundle components. This also allows an external caller to contact an application that is running on any ConductR node by contacting any proxy instance.

We will be using `HAProxy`. Install HAProxy version 1.5 or newer.

``` bash
[172.17.0.1]$ sudo apt-get -y install haproxy
```
or
``` bash
[172.17.0.1]$ sudo yum install haproxy
```

On Red Hat Enterprise Linux (RHEL) 6, haproxy is in the RHEL Server Load Balancer (v6 for 64-bit x86_64) rhel-lb-for-rhel-6-server-rpms channel. You'll need to add this channel to your server.

On some Debian distributions you may need to add a dedicated Personal Package Archive (PPA) in order to install HAProxy 1.5 via the package manager. For example:
``` bash
[172.17.0.1]$ sudo add-apt-repository -y ppa:vbernat/haproxy-1.5
[172.17.0.1]$ sudo apt-get update
[172.17.0.1]$ sudo apt-get -y install haproxy
```

ConductR provides an application that listens for bundle events from ConductR and updates HAProxy configuration accordingly. Install ConductR-HAProxy debian or RPM package which comes with the ConductR:

``` bash
[172.17.0.1]$ sudo dpkg -i /usr/share/conductr/extra/conductr-haproxy_%PLAY_VERSION%_all.deb
```
or
``` bash
[172.17.0.1]$ sudo yum install  /usr/share/conductr/extra/conductr-haproxy-%PLAY_VERSION%-1.noarch.rpm
```

Grant HAProxy configuration file read and write access to ConductR-HAProxy application:

``` bash
[172.17.0.1]$ sudo chown conductr-haproxy:conductr-haproxy /etc/haproxy/haproxy.cfg
```

After updating the configuration file ConductR-HAProxy is going to signal HAProxy to reload its configuration. Grant permissions for the particular command that ConductR-HAProxy is going to use by modifying the `sudoers` file:

``` bash
[172.17.0.1]$ echo "conductr-haproxy ALL=(root) NOPASSWD: /etc/init.d/haproxy reload" | sudo tee -a /etc/sudoers
```

On RHEL and CentOS it may also be neccessary to [disable default requiretty](https://bugzilla.redhat.com/show_bug.cgi?id=1020147) for the conductr-haproxy user.
``` bash
[172.17.0.1]$ echo 'Defaults: conductr-haproxy  !requiretty' | sudo tee -a /etc/sudoers
```

Set the ConductR IP address which is going to be used by ConductR-HAProxy to listen to bundle events in the cluster:

``` bash
[172.17.0.1]$ echo -Dconductr-haproxy.ip=$(hostname)| sudo tee -a /usr/share/conductr-haproxy/conf/application.ini
[172.17.0.1]$ sudo service conductr-haproxy restart
```

Observe ConductR-HAProxy logs. You should see a successfully opened connection to ConductR.

That's it! You now have a cluster of three ConductR nodes ready to start running applications. ConductR comes with a `visualizer` sample application. Head over to the next [Quickstart](Quickstart.html) section to learn how to deploy visualizer application to your fresh ConductR cluster.

### Consolidated Logging

When multiple machines are involved in a cluster it quickly becomes difficult to view the log files of distributed applications. ConductR allows the logging output of itself and the bundles that it executes to be directed to a "syslog collector". Syslog is a widely used protocol for Unix based machines and is supported by a number of cloud-based log providers, as well as local operating system support.

We expect that you will install a syslog collector such as [rsyslog](http://www.rsyslog.com/) within ConductR's network. You do not want to send lots of log traffic across the internet as it should be at least filtered first. The collector should be configured to accept TCP connections.

However to quickly demonstrate consolidated logging, a cloud service named [Papertrail](https://papertrailapp.com/) can be used. Papertrail is a very simple "tail like" service for viewing distributed logs. We will use it to show how to configure the simplest possible static endpoint. Once you configure an account with Papertrail then you will be provided with a host and a port. 

**Important**: ConductR logs over TCP so make sure that you configure papertrail so that it accepts plain text connections (tip: _Accounts/Log Destinations/Edit Settings/Accept connections via_).

Supposing that the address assigned to your at Papertrail is `logs2.papertrailapp.com` and `38564` then you configure ConductR like so:

``` bash
[172.17.0.1]$ echo \
  -Dcontrail.syslog.server.host=logs2.papertrailapp.com \
  -Dcontrail.syslog.server.port=38564 | \
  sudo tee -a /usr/share/conductr/conf/application.ini
[172.17.0.1]$ sudo service conductr restart
```

You can also apply a similar configuration to `conductr-haproxy` by substituting `conductr`.

#### Other hosted solutions
As previously stated you should use a hosted solution such as rsyslog in general. You may also want to investigate [Logstash](http://logstash.net/) running with [Kibana](https://www.elastic.co/products/kibana) and [Elasticsearch](https://www.elastic.co/products/elasticsearch) as an interesting hosted log analysis infrastructure. Of course they all provide syslog adapters, so can be easily used together with Contrail.

Generally speaking Contrail is compatible with any log aggregator speaking the Syslog protocol.
