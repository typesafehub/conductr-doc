# Typesafe ConductR %PLAY_VERSION%

## EC2 Installation

This is a tutorial for setting up ConductR on Amazon Web Services. This tutorial will provide you with all the key configuration details needed to run ConductR on EC2. It presumes a working knowledge of EC2 and does not provide click-by-click instructions for using EC2. Detailed instructions for all AWS steps discussed can be found in the [AWS documentation](https://aws.amazon.com/documentation/).

For general installation instructions, please see [Installation][Install.html)

### Requirements

You will need the following for this installation:

* Amazon Web Services account with EC2 admin rights
* Debian package of ConductR

### Preparing EC2

Begin by preparing the EC2 network and security environment. This tutorial uses a Virtual Private Cloud (VPC) with a Classless Inter-Domain Routing (CIDR) of `10.0.0.0/16` and example addresses will be based accordingly.

#### Subnets and Security Groups
For better resilience, deploy nodes across multiple availability zones (AZ). Create three subnets in the VPC. Place each subnet in a different availability zone by specifying an AZ during subnet creation. Our example subnets names indicate their AZ. They are SN-A with a CIDR of `10.0.1.0/24`, SN-B with a CIDR of `10.0.2.0/24` and SN-C with a CIDR of `10.0.3.0/24`. Each subnet will need an internet gateway added to their route table for the destination `0.0.0.0/0`. This required so that our nodes can access the internet. All subnets can use the same route table.

Create two security groups in the VPC named SG-Nodes and SG-ELB. SG-ELB will be for our load balancer. We'll only expose port 80 and 443 to the world (`0.0.0.0/0`) here. SG-Nodes will be for the nodes. We'll need to open one or more service ports to the load balancer. When adding the inbound rule, enter the identifier for SG-ELB in the source, such as sg-a803cb4a, to allow traffic from the load balancer security group. Our service will be on port 9999 and we'll use 9005 for monitoring. We'll need to allow TCP Port 9999 and 9005 from our load balancer security group SG-ELB in to SG-Nodes, our nodes security group. Nodes will also need to communicate with each other. Add an inbound rule to allow port 9004 and ports 10000-10999 from the SG-Nodes. Finally, SG-Nodes should also allow ssh on port 22 from Anywhere (`0.0.0.0/0`) so we can access our nodes from the internet.

The resultant security groups should now have the following inbound rules:

SG-ELB Inbound Rules

| Type    | Proto   | Port        | Source     | 
| :------ | :-----  | :---------- | :--------- |
| HTTP    | TCP     | 80          | 0.0.0.0/0  |
| HTTPS   | TCP     | 443         | 0.0.0.0/0  |

SG-Nodes Inbound Rules

| Type	  | Proto  | Port        | Source     |
| :------ | :----- | :---------- | :--------- |
| Custom  |TCP     | 9005        | SG-ELB     |
| Custom  |TCP     | 9999        | SG-ELB     |
| Custom  |TCP     | 9004        | SG-Nodes   |
| Custom  |TCP     | 10000-10999 | SG-Nodes   |
| SSH     |TCP     | 22          | 0.0.0.0/0  |		

#### Load Balancer

Create a load balancer from the EC2 control panel. You will need to create an internet gateway and attach it to your VPC in order to have a public load balancer. We'll add an optional HTTPS protocol listener on port 443 to the default port 80 HTTP listener. For this tutorial we will map both of our listeners to instance port 9999. Add all three subnets to the load balancer and assign the load balancer to the SG-ELB security group. Optionally you can upload an SSL Certificate to use the ELB as your TLS endpoint if you added the HTTPS listener. For health monitoring we'll use ConductR's bundles call, HTTP:9005/bundles.

### Preparing the AMI

Launch a single instance of the desired base AMI to use as our image master. We'll use the Ubuntu 14.10 HVM EBS-SSD boot image in US-East-1, ami-ee793a86. If you choose another base image, use an EBS boot image as they are much easy to image. Be certain to assign a public ip address in instance details. 

Access the console of the image instance with root access. For Ubuntu AMIs this is done as the user ubuntu using the PEM file specified at launch. The user ubuntu has sudo access. Other images will use different users. Check with the image provider for the correct user name to use.

#### Installing JRE 8

Install Java 8 as the default JRE. You will need to accept the Oracle license agreement.

```bash
sudo add-apt-repository -y ppa:webupd8team/java
sudo apt-get update
sudo apt-get -y install oracle-java8-installer && sudo apt-get clean
sudo apt-get -y install oracle-java8-set-default
echo "JAVA_HOME=/usr/lib/jvm/java-8-oracle" | sudo tee -a /etc/environment
```
#### Installing ConductR 

The tutorial assumes that you have obtained the `conductr_%PLAY_VERSION%_all.deb` Debian package. 

Secure copy (scp) the ConductR installation package to the image host and install ConductR as any other Debian package.

``` bash
sudo dpkg -i conductr_%PLAY_VERSION%_all.deb
```

ConductR is automatically registered as a service and started.

#### Installation miscellany

The ConductR service runs under the `conductr` user along with the `conductr` group. Its pid file is written to: `/var/run/conductr/running.pid` and its install location is `/usr/share/conductr`.

ConductR logs via the syslog protocol using TCP destined locally on port 514. Debian distributions such as Ubuntu come with the [RSYSLOG](http://www.rsyslog.com/) logging service and so its configuration is shown next:

``` bash
echo '$ModLoad imtcp' | sudo tee -a /etc/rsyslog.d/conductr.conf
echo '$InputTCPServerRun 514' | sudo tee -a /etc/rsyslog.d/conductr.conf
```

### Installing a Proxy

Proxying application endpoints is required when running more than one instance of ConductR; which should be always for production style scenarios. Proxying endpoints permits connectivity from both external callers and for bundle components to communicate with other bundle components. This also allows an external caller to contact an application that is running on any ConductR node by contacting any proxy instance.

We will be using `HAProxy`. Add a dedicated Personal Package Archive (PPA) and install HAProxy.

``` bash
sudo add-apt-repository -y ppa:vbernat/haproxy-1.5
sudo apt-get update
sudo apt-get -y install haproxy
```

ConductR provides an application that listens for bundle events from ConductR and updates HAProxy configuration accordingly. Install ConductR-HAProxy debian package which comes with the ConductR:

``` bash
sudo dpkg -i /usr/share/conductr/extra/conductr-haproxy_%PLAY_VERSION%_all.deb
```

Grant HAProxy configuration file read and write access to ConductR-HAProxy application:

``` bash
sudo chown conductr-haproxy:conductr-haproxy /etc/haproxy/haproxy.cfg
```

After updating the configuration file ConductR-HAProxy is going to signal HAProxy to reload its configuration. Grant permissions for the particular command that ConductR-HAProxy is going to use by modifying the `sudoers` file:

``` bash
echo "conductr-haproxy ALL=(root) NOPASSWD: /etc/init.d/haproxy reload" | sudo tee -a /etc/sudoers
```

#### Optional dependencies

Consolidated logging is discussed further down.

##### Docker

ConductR supports running applications and services within Docker. If you plan on running Docker based bundles, you will need to install [Docker](https://docs.docker.com/) according to [the official documentation](https://docs.docker.com/installation/ubuntulinux/). Once Docker is installed then add ConductR's user/group to the `docker` group so that it has [the correct permissions in order to access Docker](http://docs.docker.com/installation/ubuntulinux/#giving-non-root-access):

``` bash
sudo usermod -a -G docker conductr
```

### Create the AMI

With our packages installed we can create the ConductR machine image. Image the host by selecting the running instance in the EC2 dashboard and using the Create Image option from the Actions menus. We are now done with the image host and it can be terminated.

### Bring up the cluster

Once your ConductR AMI is available, launch three instances. In this tutorial we'll launch one instance into SN-A, SN-B and SN-C each so that our cluster spans three availability zones. All instances will be launched into our SG-Nodes security group. Be certain to assign public IP addresses to we can ssh into our nodes. 

We will now configure ConductR on the instances and form a cluster. Repeat these steps on each of the three instances ConductR AMI. 

To simplify the configuration instructions, we are going to use the hostname command. You must set the hostname on EC2. Simply pass the instance's private ip address to hostname.

```bash
sudo hostname 10.0.1.10
```

To be able to form an inter-machine cluster, ConductR must be configured to listen to the machine's private host interface. This can be enabled adding a property declaration for `CONDUCTR_IP` to the start command as follows:

``` bash
echo -DCONDUCTR_IP=$(hostname) | sudo tee -a /usr/share/conductr/conf/application.ini
sudo /etc/init.d/conductr restart
```

Set the ConductR IP address which is going to be used by ConductR-HAProxy to listen to bundle events in the cluster:

``` bash
echo -Dconductr-haproxy.ip=$(hostname)| sudo tee -a /usr/share/conductr-haproxy/conf/application.ini
sudo /etc/init.d/conductr-haproxy restart
```

#### Specifying the seed node

Pick one node as the seed node and instruct the other two instances to use the other as the seed node. Here we have chosen `10.0.2.20` as the seed node and will perform this additional step on all other nodes *except* the seed node `10.0.2.20`.

``` bash
echo --seed 10.0.2.20:9004 | sudo tee -a /usr/share/conductr/conf/application.ini
sudo /etc/init.d/conductr restart
```

### Check the cluster
ConductR provides cluster and application information as well as its control interface via a REST API.
 
 ``` bash
curl -s $(hostname):9005/members | python3 -m json.tool
```

A typical response contains the current members of the cluster (shown here is a three node cluster), the address of the node that the queried control server is running on and a list of unreachable nodes (shown here as empty).
 
``` json
{
    "members": [
        {
            "node": "akka.tcp://conductr@10.0.1.10:9004",
            "nodeUid": "-810451778",
            "roles": [
                "all-conductrs"
            ],
            "status": "Up"
        },
        {
            "node": "akka.tcp://conductr@10.0.2.20:9004",
            "nodeUid": "280222358",
            "roles": [
                "all-conductrs"
            ],
            "status": "Up"
        },
        {
            "node": "akka.tcp://conductr@10.0.3.30:9004",
            "nodeUid": "1503330106",
            "roles": [
                "all-conductrs"
            ],
            "status": "Up"
        }

    ],
    "selfNode": "akka.tcp://conductr@10.0.2.20:9004",
    "selfNodeUid": "280222358",
    "unreachable": []
}

```

That's it! You now have a cluster of three ConductR nodes ready to start running applications. Add all cluster instances to the load balancer. Your cluster will be reachable by the DNS name specified in the load balancer description. You can add this as a CNAME to your DNS zone file to make the cluster reachable using a hostname from your domain.

ConductR comes with a `visualizer` sample application. Head over to the next [Quickstart](Quickstart.html) section to learn how to deploy visualizer application to your fresh ConductR cluster.

### Consolidated Logging

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

#### Other hosted solutions

As previously stated you should use a hosted solution such as rsyslog in general. You may also want to investigate [Logstash](http://logstash.net/) running with [Kibana](https://www.elastic.co/products/kibana) and [Elasticsearch](https://www.elastic.co/products/elasticsearch) as an interesting hosted log analysis infrastructure. Of course they all provide syslog adapters, so can be easily used together with Contrail.

Generally speaking Contrail is compatible with any log aggregator speaking the Syslog protocol.
