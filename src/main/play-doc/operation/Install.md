# Installation

Choose on of the following installation guides to get started:

* [Linux Installation](#Linux-Installation)
* [EC2 Installation](#EC2-Installation)

# Development Installation

> Follow this guide to install and run ConductR inside Docker on your development environment in the context of an sbt application.

## Prerequisites

* [Docker](https://www.docker.com)
* [SBT](http://www.scala-sbt.org) or [Activator](https://www.typesafe.com/get-started)

For development purposes we provide a Docker image which has all of the neccessary software pre-installed to use ConductR. 

Now head over to the [[Developers guide|DevQuickStart]] to get started.

# Linux Installation

> This is a tutorial for installing ConductR on linux in production mode. It shows how this is done for a small cluster of 3 machines.

## Prerequisites

* Debian or Rpm based system (recommended: Ubuntu 14.04 LTS or RHEL/CentOS 6.6)
* Oracle Java Runtime Environment 8 (JRE 8)
* Python 3.4
* Debian or Rpm package of ConductR

RHEL/CentOS 7 support is waiting for stable [sbt-native-packer systemd support](https://github.com/sbt/sbt-native-packager#experimental-systemd-bootsystem-support).

### Installing JRE 8

Install Java 8 as the default JRE on the system.

On Ubuntu, you can use the webupd8team repository provided that you accept the Oracle license agreement.

``` bash
sudo add-apt-repository -y ppa:webupd8team/java
sudo apt-get update
sudo apt-get -y install oracle-java8-installer && sudo apt-get clean
sudo apt-get -y install oracle-java8-set-default
echo "JAVA_HOME=/usr/lib/jvm/java-8-oracle" | sudo tee -a /etc/environment
```

### Optional dependencies

* Docker - for running bundles inside a Docker container

## Installing ConductR on the first machine

This tutorial uses three systems with the addresses `172.17.0.{1,2,3}`. To simplify the installation and configuration instructions, we are going to use the hostname command. Please ensure the hostname is set correctly or substitute your addresses as appropriate for $(hostname). To set the hostname, pass the ip address to hostname.

``` bash
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

### Installation miscellany

The ConductR service runs under the `conductr` user along with the `conductr` group. Its pid file is written to: `/var/run/conductr/running.pid` and its install location is `/usr/share/conductr`.

ConductR logs via the syslog protocol using TCP destined locally on port 514. Debian distributions such as Ubuntu come with the [RSYSLOG](http://www.rsyslog.com/) logging service and so its configuration is shown next. Other distributions may require installing RYSLOG.

``` bash
[172.17.0.1]$ echo '$ModLoad imtcp' | sudo tee -a /etc/rsyslog.d/conductr.conf
[172.17.0.1]$ echo '$InputTCPServerRun 514' | sudo tee -a /etc/rsyslog.d/conductr.conf
[172.17.0.1]$ sudo service rsyslog restart
```

Viewing `/var/log/syslog` (Ubuntu) or `/var/log/messages` (RHEL) will then show ConductR and bundle output.

By default ConductR's logging is quite sparse. Unless an error or warning occurs then there will be no log output. To increase the verbosity of the logging you can use this command:

``` bash
[172.17.0.1]$ echo -Dakka.loglevel=debug | sudo tee -a /usr/share/conductr/conf/application.ini
```

[[Other logging configurations | Logging]] are possible, including an experimental solution based on Elasticsearch.

### Optional dependencies

#### Docker

ConductR supports running applications and services within Docker. If you plan on running Docker based bundles, you will need to install [Docker](https://docs.docker.com/) according to [the official documentation](https://docs.docker.com/installation/ubuntulinux/). Once Docker is installed then add ConductR's user/group to the `docker` group so that it has [the correct permissions in order to access Docker](http://docs.docker.com/installation/ubuntulinux/#giving-non-root-access):

``` bash
[172.17.0.1]$ sudo usermod -a -G docker conductr
```

## Installing ConductR on the remaining machines

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

``` bash
sudo yum install conductr_%PLAY_VERSION%-1.noarch.rpm
```
then

``` bash
[172.17.0.2]$ echo -DCONDUCTR_IP=$(hostname) | sudo tee -a /usr/share/conductr/conf/application.ini
[172.17.0.2]$ echo --seed 172.17.0.1:9004 | sudo tee -a /usr/share/conductr/conf/application.ini
[172.17.0.2]$ sudo service conductr restart
```

You should now see a new node in the cluster members list by using the following query:

``` bash
[172.17.0.2]$ curl -s 172.17.0.2:9005/members | python3 -m json.tool
```

Install optional dependencies if required. Each ConductR node requires same optional dependencies to be installed.

## Installing a Proxy

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

ConductR provides an application that listens for bundle events from ConductR and updates HAProxy configuration accordingly. The ConductR-HAProxy installation package can be found in the ConductR installation in the `extra` folder. Install ConductR-HAProxy debian or RPM package:

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

That's it! You now have a cluster of three ConductR nodes ready to start running applications. ConductR comes with a `visualizer` sample application. Head over to the next section [[CLI|CLI]] to learn how to deploy visualizer application to your fresh ConductR cluster.



# EC2 Installation

> This is a tutorial for setting up a ConductR cluster on [Amazon Web Services EC2](http://aws.amazon.com/ec2/) provided in two forms. The first uses [Ansible](http://www.ansible.com) to automate the installation. The second achieves the same result using the EC2 Management Console and ssh'ing into the instances. For general installation instructions, please see [Linux Installation][#Linux-Installation).

## Prerequisites

* Amazon Web Services(AWS) account with EC2 admin rights
* Debian package of ConductR

Prior to using the Ansible playbooks to create your cluster, you will needs the following:

* Access Key and Secret values for your AWS account.
* Ansible installed on a controller host.
* An AWS Key Pair (PEM file) on the Ansible controller host.
* A copy of the ConductR deb installation package on the Ansible controller host.
* A copy of the [ConductR-Ansible](https://github.com/typesafehub/conductr-ansible) repository on the Ansible controller host.
* Ability to accept Oracle's Java License.

## Ansible Instructions

The [ConductR-Ansible](https://github.com/typesafehub/conductr-ansible) plays and playbooks provision [Typesafe ConductR](https://conductr.typesafe.com) cluster nodes in AWS EC2 using [Ansible](http://www.ansible.com).

Use create-network-ec2.yml to setup a new Virtual Private Cloud (VPC) and create your cluster in the new VPC. You only need to provide your access keys and what region to execute in. The playbook outputs a vars file for use with the build-cluster-ec.yml.

The playbook build-cluster-ec2.yml launches three instances across two availability zones. ConductR is installed on all instances and configured to form a cluster. The nodes are registered with a load balancer. This playbook can be used with the newly created VPC from create-network-ec2.yml or your existing VPC and security groups.

### Prepare controller host

The controller host is the host from which we will run the playbooks. The controller host should have a good, fast connection to the EC2 region we are to execute in. An Ubuntu instance running in the target EC2 region is an easy way to achieve this. See [Ansible Installation](http://docs.ansible.com/intro_installation.html) for Ansible's requirements.

From a shell on the controller host, clone the Ansible and ConductR-Ansible repositories.

``` bash
sudo apt-get -y install python-setuptools autoconf g++ python2.7-dev
sudo easy_install pip
sudo pip install paramiko PyYAML Jinja2 httplib2 boto
sudo mkdir /etc/ansible
echo -e "[local]\n127.0.0.1" | sudo tee -a /etc/ansible/hosts
sudo apt-get -y install git
git clone https://github.com/ansible/ansible.git --recursive
cd ansible
source ./hacking/env-setup -q
cd ..
git clone https://github.com/typesafehub/conductr-ansible.git
cd conductr-ansible
```

Export your AWS access key id and secret.

``` bash
export AWS_ACCESS_KEY_ID='ABC123'
export AWS_SECRET_ACCESS_KEY='abc123'
export ANSIBLE_HOST_KEY_CHECKING=False
```

Upload the ConductR installation package and your EC2 key pair file to the controller host. The ConductR installation package should be put in `conductr-asnsible/conductr/files`. The file name must match the value of `CONDUCTR_PKG` in the vars file used.

Your controller host is now ready to run plays.

### Create network

ConductR-Ansible can create and prepare a new VPC for use with ConductR. Running ConductR in it's own VPC isolates the cluster from the rest of your EC2 network. If you have existing services in EC2 that ConductR needs to be able to access on the local network using an EC2 private ip address, you need to use your existing VPC. In all other cases, creating a ConductR VPC is recommended, but is not required if you are comfortabling setting up the network yourself.

``` bash
ansible-playbook create-network-ec2.yml
```

Runing the playbook creates a new VPC named "ConductR VPC" in the us-east-1 region. To specify a different [EC2 region](http://docs.aws.amazon.com/general/latest/gr/rande.html#ec2_region) in which to execute, pass `EC2_REGION` using a -e key value pair. For example to execute in eu-west-1 we would use: 

``` bash
ansible-playbook create-network-ec2.yml -e "EC2_REGION=eu-west-1"
```

The create network playbook produces a vars file in the `vars` folder named `{{EC2_REGION}}_vars.yml` where {{EC2_REGION}} is the region used. You **must** add the name of your key pair to `{{EC2_REGION}}_vars.yml` in order to use it with the build cluster script. Change the "Key Pair Name" of `KEYPAIR: "Key Pair Name"` to that of the key pair name, which may be different than the file name and generally does not end in the .pem file extension.

If you chose to execute in a region other than us-east-1, you will also need to change the AMI value for `IMAGE` in your vars file to that of an Ubuntu image in that region. The AMI listed is the Ubuntu 14.04 LTS HVM EBS boot image published by Canonical for us-east-1. Other recent versions and types of Ubuntu instances are expected to work. The [Ubuntu AMI Locator](http://cloud-images.ubuntu.com/locator/ec2/) can help you find AMIs for alternative regions and instance types.
 
Re-running the create network playbook in the same EC2 region refreshes the network to the playbook configuration and does not create multiple VPCs.
 
### Build cluster

The second playbook launches three instances into the specified VPC and configures them into a cluster with an Elastic Load Balancer (ELB). The use of an ELB provides us with a single DNS name to access our cluster by, but its use is not required. 

We pass both our vars file and EC2 PEM key file to our playbook as command line arguments. The VARS_FILE template can be the one created from the create network script. If you want to use an existing network instead, there is a `vars.yml` template you can use as a template without running the create network script. 

The private-key value must be the local path and filename of the keypair that has the key pair name `KEYPAIR` specified in the vars file. For example our key pair may be named `ConductR_Key` in AWS and reside locally as `~/secrets/ConductR.pem`. In which case we would set `KEYPAIR` to `ConductR_Key` and pass `~/secrets/ConductR.pem` as our private-key argument. The private key file must be only accessible to owner and will require using `chmod 600 /path/to/{{keypair}}` if accessible to others.

``` bash
ansible-playbook build-cluster-ec2.yml -e "VARS_FILE=vars/{{EC2_REGION}}_vars.yml" --private-key /path/to/{{keypair}}
```

If the playbook completes successfully, you will have a three node cluster that can be accessed using the ELB DNS name. ConductR comes with a `visualizer` sample application. The playbook created ELB includes a listener mapping port 80 to Visualizer's port 9999 port mapping. 

Head over to the next section [[Managing application|ManagingApplication]] to learn how to deploy visualizer application to your fresh ConductR cluster. You can ssh into one of the cluster nodes using it's public ip address to deploy Visualizer. Use the username from the `REMOTE_USER` (currently "ubuntu") and the PEM file as for the identify file (-i). The ConductR CLI has been installed to all nodes for you. Once deployed, you can view the Visualizer via port 80 using the ELB DNS name in your browser.

Re-running this playbook launches a new set of instances. This means it can be re-run to create additional ConductR clusters. For example we might re-run the playbook to create a new cluster using a new version of ConductR to test new features. If we change only the values of `CONDUCTR_PKG` and `ELB` in the vars file to a new ConductR version package and new ELB, running the playbook again will create a new cluster using the new version in the same subnets as the previous version.

For further information about using ConductR-Ansible, please see the project [Readme](https://github.com/typesafehub/conductr-ansible/blob/master/README.md).

## Manual Instructions

This tutorial will provide you with all the key configuration details needed to run ConductR on EC2. It presumes a working knowledge of EC2 and does not provide click-by-click instructions for using EC2. Detailed instructions for all AWS steps discussed can be found in the [AWS documentation](https://aws.amazon.com/documentation/).

### Preparing EC2

Begin by preparing the EC2 network and security environment. This tutorial uses a Virtual Private Cloud (VPC) with a Classless Inter-Domain Routing (CIDR) of `10.0.0.0/16` and example addresses will be based accordingly.

#### Subnets and Security Groups
For better resilience, deploy nodes across multiple availability zones (AZ). Create three subnets in the VPC. Place each subnet in a different availability zone by specifying an AZ during subnet creation. Our example subnets names indicate their AZ. They are SN-A with a CIDR of `10.0.1.0/24`, SN-B with a CIDR of `10.0.2.0/24` and SN-C with a CIDR of `10.0.3.0/24`. Each subnet will need an internet gateway added to their route table for the destination `0.0.0.0/0`. This required so that our nodes can access the internet. All subnets can use the same route table.

Create two security groups in the VPC named SG-Nodes and SG-ELB. SG-ELB will be for our load balancer. We'll only expose port 80 and 443 to the world (`0.0.0.0/0`) here. SG-Nodes will be for the nodes. We'll need to open one or more service ports to the load balancer. When adding the inbound rule, enter the identifier for SG-ELB in the source, such as sg-a803cb4a, to allow traffic from the load balancer security group. Our service will be on port 9999 and we'll use 9005 for monitoring. We'll need to allow TCP Port 9999 and 9005 from our load balancer security group SG-ELB in to SG-Nodes, our nodes security group. Nodes will also need to communicate with each other. Add an inbound rule to allow port 9004, 9006 and port range 10000-10999 from the SG-Nodes. Finally, SG-Nodes should also allow ssh on port 22 from Anywhere (`0.0.0.0/0`) so we can also directly access our nodes from the internet.

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
| Custom  |TCP     | 9006        | SG-Nodes   |
| Custom  |TCP     | 10000-10999 | SG-Nodes   |
| SSH     |TCP     | 22          | 0.0.0.0/0  |		

#### Load Balancer

Create an external facing load balancer from the EC2 control panel. You will need to create an internet gateway and attach it to your VPC in order to have a public load balancer. We'll add an optional HTTPS protocol listener on port 443 to the default port 80 HTTP listener. For this tutorial we will map both of our listeners to instance port 9999. Add all three subnets to the load balancer and assign the load balancer to the SG-ELB security group. Optionally you can upload an SSL Certificate to use the ELB as your TLS endpoint if you added the HTTPS listener. For health monitoring we'll use ConductR's bundles endpoint, HTTP:9005/bundles.

### Preparing the AMI

Launch a single instance of the desired base AMI to use as our image master. We'll use the Ubuntu 14.04 LTS HVM EBS-SSD boot image in US-East-1, ami-76b2a71e. If you choose another base image, use an EBS boot image as they are much easy to image unless you know what your doing there. Be certain to assign a public ip address in instance details to make it easy to ssh into.

Access the console of the image instance with root access. For Ubuntu AMIs this is done as the user ubuntu using the PEM file specified at launch. The user ubuntu has sudo access. Other images will use different users. Check with the image provider for the correct user name to use.

#### Installing JRE 8

Install Java 8 as the default JRE. You will need to accept the Oracle license agreement.

``` bash
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

To be able to form an inter-machine cluster, ConductR must be configured to listen to the machine's private host interface. This can be enabled adding a property declaration for `CONDUCTR_IP` to the start command as follows:

``` bash
echo -DCONDUCTR_IP=$(hostname -i) | sudo tee -a /usr/share/conductr/conf/application.ini
sudo service conductr restart
```

Set the ConductR IP address which is going to be used by ConductR-HAProxy to listen to bundle events in the cluster:

``` bash
echo -Dconductr-haproxy.ip=$(hostname -i)| sudo tee -a /usr/share/conductr-haproxy/conf/application.ini
sudo service conductr-haproxy restart
```

#### Specifying the seed node

Pick one node as the seed node and instruct the other two instances to use the other as the seed node. Here we have chosen `10.0.2.20` as the seed node and will perform this additional step on all other nodes *except* the seed node `10.0.2.20`.

``` bash
echo --seed 10.0.2.20:9004 | sudo tee -a /usr/share/conductr/conf/application.ini
sudo service conductr restart
```

### Check the cluster
ConductR provides cluster and application information as well as its control interface via a REST API.
 
 ``` bash
curl -s $(hostname -i):9005/members | python3 -m json.tool
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

ConductR comes with a `visualizer` sample application. Head over to the next section [[CLI|CLI]] to learn how to deploy visualizer application to your fresh ConductR cluster.
