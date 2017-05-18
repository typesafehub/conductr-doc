# Installation

Choose on of the following installation guides to get started:

* [Linux Installation](#Linux-Installation)
* [EC2 Installation](#EC2-Installation)
* [DC/OS Installation](#DC/OS-Installation)

# Linux Installation

> In order to obtain the Debian or RPM installations of ConductR then please [contact our sales department](https://www.lightbend.com/company/contact). To evaluate ConductR in general then [please visit our product page](http://www.lightbend.com/products/conductr) which provides instructions on getting started. Otherwise if you are looking to use ConductR for free from a development perspective then please [head over to our developer section](DevQuickStart).

> This is a tutorial for installing ConductR on Linux in production mode. It shows how this is done for a small cluster of 3 machines. If you are looking for a non-production Linux installation (for example, a QA environment that is close to production), be sure to read about [how to setup for non-production](ClusterSetupConsiderations#Setting-up-for-non-production) after reading the remainder of this page.

## Prerequisites

* x86/64 bit Debian or Rpm based Linux system (recommended: Ubuntu 16.04 LTS or RHEL/CentOS 7)
* Oracle or OpenJDK Java Runtime Environment 8 (JRE 8)
* Debian or Rpm package of ConductR Core
* Debian or Rpm package of ConductR Agent

### Installing JRE 8

Install Java 8 as the default JRE on the system.

On Ubuntu 14.10 and newer, you can use the built in package manager.

```bash
sudo apt-get update && sudo apt-get install openjdk-8-jdk
```

On RHEL/CentOS 7, you can use the built in package manager.

```bash
sudo yum install java-1.8.0-openjdk.x86_64
```

On Ubuntu 14.04 and older, you can use the webupd8team repository provided that you accept the Oracle license agreement.

```bash
sudo add-apt-repository -y ppa:webupd8team/java
sudo apt-get update
sudo apt-get -y install oracle-java8-installer && sudo apt-get clean
sudo apt-get -y install oracle-java8-set-default
echo "JAVA_HOME=/usr/lib/jvm/java-8-oracle" | sudo tee -a /etc/environment
```

NOTE: You can also configure ConductR to use a local JRE installation. To do this, you must follow these steps after installing ConductR Core and Agent in the subsequent sections.

```bash
# This example assumes you've installed the JRE to /opt/my-jdk and have already installed ConductR Core and Agent

echo 'JAVA_HOME=/opt/my-jdk' | sudo tee -a /etc/default/conductr
echo 'JAVA_HOME=/opt/my-jdk' | sudo tee -a /etc/default/conductr-agent

echo 'PATH="/opt/my-jdk/bin:/usr/local/bin:/usr/bin:/usr/local/sbin:/usr/sbin"' | sudo tee -a /etc/default/conductr
echo 'PATH="/opt/my-jdk/bin:/usr/local/bin:/usr/bin:/usr/local/sbin:/usr/sbin"' | sudo tee -a /etc/default/conductr-agent
```

### Optional dependencies

* Docker - for running bundles inside a Docker container

## Installing ConductR on the first machine

ConductR comprises of the ConductR Core and ConductR Agent. ConductR Core is responsible for cluster-wide scaling
 and replication decision making, as well as hosting the application files or the bundles. ConductR Agent is responsible
 for executing the application processes. The ConductR Core and the ConductR Agent are run as separate processes,
 and hence separate services within the operating system.

This tutorial uses three systems with the addresses `172.17.0.{1,2,3}`. To simplify the installation and configuration
 instructions, we are going to use the hostname command to export the host's IP address as `HOSTIP`.
 Please ensure the hostname is set correctly or substitute your addresses as appropriate for `$(hostname -i)`, below.

> It is *very* important that you use an IP address when configuring or reaching ConductR. DNS provides a layer of indirection that you will not want. You should always be mindful of the specific network interface that you bind a service to, including ConductR - if only for reasons of security. Along those lines, binding to 0.0.0.0 (any network interface) should always be avoided.

```bash
export HOSTIP=$(hostname -i)
echo $(HOSTIP)
```

> If the result of `echo $(HOSTIP)` above is not an IP address then you will probably need to use `ifconfig` in order to list the network interfaces that you have and then choose one.

### Installing ConductR Core on the first machine
The tutorial also assumes that you have obtained the `conductr_%PLAY_VERSION%_all.deb` Debian or `conductr_%PLAY_VERSION%-1.noarch.rpm` Rpm package.

Install ConductR Core as any other Debian or Rpm package.

```bash
[172.17.0.1]$ sudo dpkg -i conductr_%PLAY_VERSION%_all.deb
```

or


```bash
[172.17.0.1]$ sudo yum install conductr-%PLAY_VERSION%-1.noarch.rpm
```

ConductR Core is automatically registered as a service and started. ConductR provides cluster and application information as well as its control interface via a REST API exposed as part of the ConductR Core.

```bash
export CONDUCTR_IP=$(hostname -i)
[172.17.0.1]$ conduct members
```

A typical response contains the current members of the cluster (shown here as just one),
 the address of the node that the queried control server is running on and a list
 of unreachable nodes (shown here as empty).

``` bash
UID         ADDRESS                             ROLES       STATUS  REACHABLE
-890829264  akka.tcp://conductr@127.0.0.1:9004  replicator  Up            Yes
```

The IP addresses in the response indicate that ConductR Core is listening to the `localhost` address. To be able to form an inter-machine cluster, ConductR Core must be configured to listen to the machine's host interface. This can be enabled adding a property declaration for `CONDUCTR_IP` to the start command as follows:

```bash
[172.17.0.1]$ echo -DCONDUCTR_IP=$(HOSTIP) | sudo tee -a /usr/share/conductr/conf/conductr.ini
[172.17.0.1]$ sudo service conductr restart
```

Check for the cluster information once again, but now use the host address of the machine.

```bash
[172.17.0.1]$ conduct members
```

The ConductR Core service runs under the `conductr` user along with the `conductr` group. Its install location is `/usr/share/conductr`.

### Installing ConductR Agent on the first machine

The tutorial also assumes that you have obtained the `conductr-agent_%PLAY_VERSION%_all.deb` Debian or `conductr-agent_%PLAY_VERSION%-1.noarch.rpm` Rpm package.

Install ConductR Agent as any other Debian or Rpm package.

```bash
[172.17.0.1]$ sudo dpkg -i conductr-agent_%PLAY_VERSION%_all.deb
```
or

```bash
[172.17.0.1]$ sudo yum install conductr-agent-%PLAY_VERSION%-1.noarch.rpm
```

ConductR Agent is automatically registered as a service and started.

ConductR Agent needs to be connected to a ConductR core node in order for ConductR to run any application process. To establish this connection, configure ConductR Agent as such:

```bash
[172.17.0.1]$ echo -Dconductr.agent.ip=$(HOSTIP) | sudo tee -a /usr/share/conductr-agent/conf/conductr-agent.ini
[172.17.0.1]$ echo --core-node $(HOSTIP):9004 | sudo tee -a /usr/share/conductr-agent/conf/conductr-agent.ini
```

Once configured, restart the ConductR Agent service.

```bash
[172.17.0.1]$ sudo service conductr-agent restart
```

The ConductR Agent service runs under the `conductr-agent` user along with the `conductr-agent` group. Its pid file is written to: `/var/run/conductr-agent/running.pid` and its install location is `/usr/share/conductr-agent`.


### Consolidated Logging

Lightbend's consolidated logging feature is based upon Elasticsearch and requires access to an Elasticsearch engine
 to work using the default configuration.
 Please note that Elasticsearch is not required however `conduct logs` and `conduct events` does require
 Elasticsearch as described in [[Consolidated logging|ConsolidatedLogging]].
 The [[Consolidated logging|ConsolidatedLogging]] section describes the steps required which allow you to select the
 appropriate logging method for you.

By default ConductR's logging is quite sparse. Unless an error or warning occurs then there will be no log output.

To increase the verbosity of the ConductR Core logging you can use this command:

```bash
[172.17.0.1]$ echo -Dakka.loglevel=debug | sudo tee -a /usr/share/conductr/conf/conductr.ini
```

As with all `conductr.ini` file changes, you must restart the ConductR Core service for this change to take effect.

```bash
[172.17.0.1]$sudo service conductr restart

```

Similarly for ConductR Agent:

```bash
[172.17.0.1]$ echo -Dakka.loglevel=debug | sudo tee -a /usr/share/conductr-agent/conf/conductr-agent.ini
```

```bash
[172.17.0.1]$sudo service conductr-agent restart

```

### Optional dependencies

#### Docker

ConductR supports running applications and services within Docker. If you plan on running Docker based bundles, you will need to install [Docker](https://docs.docker.com/) according to [the official documentation](https://docs.docker.com/installation/ubuntulinux/). Once Docker is installed then add the ConductR-Agent daemon user/group to the `docker` group so that it has [the correct permissions in order to access Docker](http://docs.docker.com/installation/ubuntulinux/#giving-non-root-access):

```bash
[172.17.0.1]$ sudo groupadd -f docker
[172.17.0.1]$ sudo usermod -a -G docker conductr-agent
[172.17.0.1]$ sudo service docker restart
[172.17.0.1]$ sudo conductr-agent restart
```

## Installing ConductR on the remaining machines

_Repeat each step in this section also on the `172.17.0.2` and `172.17.0.3` machine._

For a three node cluster, we recommend that you install both the Core and Agent on all three nodes.
For larger clusters, 3 core-only nodes can manage many agent-only nodes. An Agent must be installed for a node to be able execute bundles.

First ensure that the following ports are available between the machines forming the cluster:

* 9004 - Akka remoting for ConductR
* 9006 - Bundle streaming between ConductR nodes
* 10000 to 10999 - the default range of ports allocated to bundle component endpoints

On most systems `netstat` can be used to check port status.
 For example use: `netstat -an | grep 9004` to check that status of port `9004`.
 The port is in-use if `netstat` shows the _local address_ as being in the `LISTEN` or `ESTABLISHED` state.

### Installing ConductR Core on the remaining machines

The node running on the `172.17.0.1` machine is called a seed node, which is a node that is going to be used as the initial contact point when joining a cluster. Install ConductR Core on the new machine and configure the address of a seed node:

```bash
[172.17.0.2]$ sudo dpkg -i conductr_%PLAY_VERSION%_all.deb
```
or

```bash
sudo yum install conductr_%PLAY_VERSION%-1.noarch.rpm
```
then

```bash
[172.17.0.2]$ echo -DCONDUCTR_IP=$(HOSTIP) | sudo tee -a /usr/share/conductr/conf/conductr.ini
[172.17.0.2]$ echo --seed 172.17.0.1:9004 | sudo tee -a /usr/share/conductr/conf/conductr.ini
[172.17.0.2]$ sudo service conductr restart
```

You should now see a new node in the cluster members list by using the following query:

```bash
[172.17.0.2]$ conduct members
```

### Installing ConductR Agent on the remaining machines

Install ConductR Agent:

```bash
[172.17.0.1]$ sudo dpkg -i conductr-agent_%PLAY_VERSION%_all.deb
```
or

```bash
[172.17.0.1]$ sudo yum install conductr-agent-%PLAY_VERSION%-1.noarch.rpm
```

Next, you'll need to configure a couple of settings in `conductr-agent.ini` and grant some privileges. ConductR uses [runc](https://runc.io/) to spawn and run OCI-based bundles. Since `runc` makes use of [cgroups](https://en.wikipedia.org/wiki/Cgroups), the OCI component of ConductR must be granted root access by making an entry in `sudoers`. Great care has been taken to ensure only this component requires root privileges. 

```bash
[172.17.0.1]$ echo -Dconductr.agent.ip=$(HOSTIP) | sudo tee -a /usr/share/conductr-agent/conf/conductr-agent.ini
[172.17.0.1]$ echo --core-node $(HOSTIP):9004 | sudo tee -a /usr/share/conductr-agent/conf/conductr-agent.ini
[172.17.0.1]$ echo "conductr-agent ALL=(root) NOPASSWD:SETENV: /usr/share/conductr-agent/bin/conductr-oci" | sudo tee -a /etc/sudoers
[172.17.0.1]$ sudo service conductr-agent restart
```

Some bundles may require specific node configuration before the bundle can be executed on that node.
 Configuring a node for dynamic proxying with HAProxy for example, requires root access configuration that cannot be
 performed by a runtime bundle initialization script. A node is assigned the `haproxy` role to indicate that the prerequisite
 steps have been performed.

If your bundles requires specific dependencies in order to execute, install those dependencies now.
 All others nodes expected to be run such bundles should have the same optional dependencies installed.
 Use a custom [role](ClusterConfiguration#Roles) such as `my-role` if only a subset of nodes will be able to execute the bundle.

## Installing a Proxy

_Perform each step in this section on all nodes: `172.17.0.1`, `172.17.0.2` and `172.17.0.3`. For full resilience a proxy should be installed for each machine that ConductR Agent is installed on._

Proxying application endpoints is required when running more than one instance of ConductR; which should be always for production style scenarios. Proxying endpoints permits connectivity from both external callers and for bundle components to communicate with other bundle components. This also allows an external caller to contact an application that is running on any ConductR node by contacting any proxy instance.

We will be using `HAProxy` version 1.5 or newer.

Install HAProxy using the following commands.

```bash
[172.17.0.1]$ sudo apt-get -y install haproxy
```
or
```bash
[172.17.0.1]$ sudo yum install haproxy
```

On Red Hat Enterprise Linux (RHEL) 6, haproxy is in the RHEL Server Load Balancer (v6 for 64-bit x86_64) rhel-lb-for-rhel-6-server-rpms channel. You'll need to add this channel to your server.

On some Debian distributions you may need to add a dedicated Personal Package Archive (PPA) in order to install HAProxy 1.5 via the package manager. For example:

```bash
[172.17.0.1]$ sudo add-apt-repository -y ppa:vbernat/haproxy-1.5
[172.17.0.1]$ sudo apt-get update
[172.17.0.1]$ sudo apt-get -y install haproxy
```

ConductR provides a ConductR-HAProxy bundle that listens for bundle events from ConductR and updates the local HAProxy configuration file accordingly. We must specifically allow the bundle to use `sudo` to reload HAProxy.

We have the user `conductr-agent` own the HAProxy config file.

```bash
[172.17.0.1]$ sudo chown conductr-agent:conductr-agent /etc/haproxy/haproxy.cfg
```

After updating the HAProxy configuration file, ConductR-HAProxy will signal HAProxy to reload for the updated configuration.

On RHEL and CentOS it may also be neccessary to [disable default requiretty](https://bugzilla.redhat.com/show_bug.cgi?id=1020147) for the `conductr-agent` user in `sudoers`.

```bash
[172.17.0.1]$ echo 'Defaults: conductr-agent  !requiretty' | sudo tee -a /etc/sudoers
```

HAProxy reload script is located in `/usr/bin/reloadHAProxy.sh` by default. ConductR-HAProxy will install its reload script in this location upon startup.

We will limit the bundle's sudo privileges to running `/usr/bin/reloadHAProxy.sh`. Grant permissions to the `conductr-agent` user to run the `reloadHAProxy.sh` command. An addition to `/etc/sudoers` allows for using `sudo` without password for the `reloadHAProxy.sh` script.

```bash
[172.17.0.1]$ sudo touch /usr/bin/reloadHAProxy.sh
[172.17.0.1]$ sudo chmod 0770 /usr/bin/reloadHAProxy.sh
[172.17.0.1]$ sudo chown conductr-agent:conductr-agent /usr/bin/reloadHAProxy.sh
[172.17.0.1]$ echo "conductr-agent ALL=(root) NOPASSWD: /usr/bin/reloadHAProxy.sh" | sudo tee -a /etc/sudoers
```

Note: If it is not possible or preferrable to edit the /etc/sudoers file, HAProxy may still be used with ConductR by using an [[alternative HAProxy configuration|DynamicProxyConfiguration#enabling-conductr-to-use-haproxy-without-altering-the-sudoers-file]].

## Loading and Running ConductR-HAProxy Bundle

ConductR-HAProxy bundle listens for bundle changes within ConductR and updates the HAProxy config to expose the bundle endpoints accordingly.

### Prepare ConductR-HAProxy nodes

_Perform each step in this section on all nodes: `172.17.0.1`, `172.17.0.2` and `172.17.0.3`_.

ConductR-HAProxy bundle must be installed on all nodes where HAProxy is installed, and these nodes can be
 distinguished by the `haproxy` role. Assign the `haproxy` [role](ClusterConfiguration#Roles) to the nodes where the proxy will be hosted.

Append the `haproxy` role to the default `web` role as follows:

```bash
[172.17.0.1]$ echo -Dconductr.agent.roles.0=web | sudo tee -a /usr/share/conductr-agent/conf/conductr-agent.ini
[172.17.0.1]$ echo -Dconductr.agent.roles.1=haproxy | sudo tee -a /usr/share/conductr-agent/conf/conductr-agent.ini
[172.17.0.1]$ sudo service conductr-agent restart
```


### Use CLI to load and run ConductR-HAProxy bundle

_Execute the step in this section only on the `172.17.0.1` machine. We could also use a host that is not a cluster member as our control node._

These instructions for loading and running the ConductR-HAProxy bundle require the [[CLI|CLI]] to be installed. Continue with the next step once [[ConductR CLI|CLI]] is installed.

Load the ConductR-HAProxy bundle from the [bundles repo](https://bintray.com/typesafe/bundle).

```bash
[172.17.0.1]$ conduct load conductr-haproxy
```

Scale ConductR-HAProxy so that ConductR-HAProxy is running on every proxy node in the cluster. In our case we have 3 nodes where the proxy is expected to be running, so we scale up the ConductR-HAProxy to 3 instances.

```bash
[172.17.0.1]$ conduct run conductr-haproxy --scale 3
```

That's it! You now have a cluster of three ConductR nodes ready to start running applications. ConductR comes with a `visualizer` sample application. Head over to the next section [[CLI|CLI]] to learn how to deploy visualizer application to your fresh ConductR cluster.

## Starting and Stopping ConductR

ConductR has been designed to be "always on". If you wish to use ConductR as a transient test environment that can be stopped and started at will (for example) then you must become familiar with [how to setup for non-production](ClusterSetupConsiderations#Setting-up-for-non-production). Otherwise individual ConductR services may be stopped and started at will. Existing executions will be re-scheduled for execution elsewhere and once the new service is started, it can also receive new executions.

# EC2 Installation

> This is a tutorial for setting up a ConductR cluster on [Amazon Web Services EC2](http://aws.amazon.com/ec2/) provided in two forms. The first uses [Ansible](http://www.ansible.com) to automate the installation. The second achieves the same result using the EC2 Management Console and ssh'ing into the instances. For general installation instructions, please see [Linux Installation][#Linux-Installation).

## Prerequisites

* Amazon Web Services(AWS) account with EC2 admin rights
* Debian package of ConductR
* Debian package of ConductR Agent

Prior to using the Ansible playbooks to create your cluster, you will needs the following:

* Lightbend credentials. Obtain for free from [lightbend.com](https://www.lightbend.com/product/conductr/developer). Apply your credentials to `conductr/files/commercial.credentials.template` and save as `conductr/files/commercial.credentials`.
* Access Key and Secret values for your AWS account.
* Ansible installed on a controller host. This host can also be used as a [bastion host](https://en.wikipedia.org/wiki/Bastion_host).
* An AWS Key Pair (PEM file) on the Ansible controller host.
* A copy of the ConductR and ConductR Agent deb installation packages on the Ansible controller host.
* A copy of the [ConductR-Ansible](https://github.com/typesafehub/conductr-ansible) repository on the Ansible controller host.

## Ansible Instructions

The [ConductR-Ansible](https://github.com/typesafehub/conductr-ansible) plays and playbooks provision [Lightbend ConductR](https://conductr.lightbend.com) cluster nodes in AWS EC2 using [Ansible](http://www.ansible.com). The branches of the repository track that of ConductR. As such the 2.0.x branch of ConductR-Ansible would be used to build ConductR 2.0.x clusters.

Optionally, use create-network-ec2.yml to setup a new Virtual Private Cloud (VPC) and create your cluster in the new VPC. You only need to provide your access keys and what region to execute in. The playbook outputs a vars file for use with the build-cluster-ec.yml. If you choose to use an existing VPC, be certain to note the required security group rules.

The playbook build-cluster-ec2.yml launches three instances across three availability zones and one instance for imaging. ConductR Core, ConductR Agent and HAProxy are installed on all instances and configured to form a cluster. Conductr-HAProxy is loaded and the nodes are registered with a load balancer. This playbook can be used with the newly created VPC from create-network-ec2.yml or your existing VPC and security groups.

### Prepare controller host

The controller host is the host from which we will run the playbooks. The controller host should have a good, fast connection to the EC2 region we are to execute in. An Ubuntu instance running in the target EC2 region is an easy way to achieve this. See [Ansible Installation](http://docs.ansible.com/intro_installation.html) for Ansible's requirements.

From a shell on the controller host, clone the Ansible and ConductR-Ansible repositories.

```bash
sudo apt-get -y install build-essential libssl-dev libffi-dev python-dev
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

```bash
export AWS_ACCESS_KEY_ID='ABC123'
export AWS_SECRET_ACCESS_KEY='abc123'
export ANSIBLE_HOST_KEY_CHECKING=False
```

Upload the files required for ConductR installation and your EC2 key pair file to the controller host. These files comprises of:

 * ConductR Core deb file
 * ConductR Agent deb file
 * HAProxy reload script

The ConductR installation files should be put in `conductr-ansible/conductr/files`. The file name must match the value of `CONDUCTR_PKG`, `CONDUCTR_AGENT_PKG`, and `HAPROXY_RELOAD_SCRIPT` in the vars file used.

The content of the HAProxy reload script can be found in [Dynamic Proxy Configuration](#DynamicProxyConfiguration).

Your controller host is now ready to run plays.

### Create network

ConductR-Ansible can create and prepare a new VPC for use with ConductR. Running ConductR in its own VPC isolates the cluster from the rest of your EC2 network. If you have existing services in EC2 that ConductR needs to be able to access on the local network using an EC2 private ip address, you need to use your existing VPC. In all other cases, creating a ConductR VPC is recommended, but is not required if you are comfortabling setting up the network yourself.

```bash
ansible-playbook create-network-ec2.yml
```

Running the playbook creates a new VPC named "ConductR VPC" in the us-east-1 region. To specify a different [EC2 region](http://docs.aws.amazon.com/general/latest/gr/rande.html#ec2_region) in which to execute, pass `EC2_REGION` using a -e key value pair. For example to execute in eu-west-1 we would use:

```bash
ansible-playbook create-network-ec2.yml -e "EC2_REGION=eu-west-1"
```

The create network playbook produces a vars file in the `vars` folder named `{{EC2_REGION}}_vars.yml` where {{EC2_REGION}} is the region used. You **must** add the name of your key pair to `{{EC2_REGION}}_vars.yml` in order to use it with the build cluster script. Change the "Key Pair Name" of `KEYPAIR: "Key Pair Name"` to that of the key pair name, which may be different than the file name and generally does not end in the .pem file extension.

If you chose to execute in a region other than us-east-1, you will also need to change the AMI value for `IMAGE` in your vars file to that of an Ubuntu image in that region. The AMI listed is the Ubuntu 16.04 LTS HVM EBS boot image published by Canonical for us-east-1. Other recent versions and types of Ubuntu instances are expected to work. The [Ubuntu AMI Locator](http://cloud-images.ubuntu.com/locator/ec2/) can help you find AMIs for alternative regions and instance types.

Re-running the create network playbook in the same EC2 region refreshes the network to the playbook configuration and does not create multiple VPCs.

### Build cluster

The second playbook launches three instances into the specified VPC and configures them into a cluster with an Elastic Load Balancer (ELB). The use of an ELB provides us with a single DNS name to access our cluster by, but its use is not required.

We pass both our vars file and EC2 PEM key file to our playbook as command line arguments. The VARS_FILE template can be the one created from the create network script. If you want to use an existing network instead, there is a `vars.yml` template you can use as a template without running the create network script.

```bash
ansible-playbook build-cluster-ec2.yml -e "VARS_FILE=vars/{{EC2_REGION}}_vars.yml" --private-key /path/to/{{keypair}}
```

The private-key value must be the local path and filename of the keypair that has the key pair name `KEYPAIR` specified in the vars file. For example our key pair may be named `ConductR_Key` in AWS and reside locally as `~/secrets/ConductR.pem`. In which case we would set `KEYPAIR` to `ConductR_Key` and pass `~/secrets/ConductR.pem` as our private-key argument. The private key file must be only accessible to owner and will require using `chmod 600 /path/to/{{keypair}}` if accessible to others.

```bash
ansible-playbook build-cluster-ec2.yml -e "VARS_FILE=vars/us-east-1_vars.yml" --private-key ~/secrets/ConductR.pem
```

If the playbook completes successfully, you will have a three node cluster that can be accessed using the ELB DNS name. ConductR comes with a `visualizer` sample application. The playbook created ELB includes a listener mapping port 80 to Visualizer's port 9999 port mapping.

You can ssh into one of the cluster nodes using its public ip address to use the [CLI](CLI). Use the username from the `REMOTE_USER` (currently "ubuntu") and the PEM file as for the identify file (-i). The ConductR CLI has been installed to all nodes for you. Once deployed, you can view the Visualizer via port 80 using the ELB DNS name in your browser.

Re-running this playbook launches a new set of instances. This means it can be re-run to create additional ConductR clusters. For example we might re-run the playbook to create a new cluster using a new version of ConductR to test new features. If we change only the values of `CONDUCTR_PKG`, `CONDUCTR_AGENT_PKG`, and `ELB` in the vars file to a new ConductR version package and new ELB, running the playbook again will create a new cluster using the new version in the same subnets as the previous version.

For further information about using ConductR-Ansible, please see the project [Readme](https://github.com/typesafehub/conductr-ansible/blob/master/README.md).

## Manual Instructions

This tutorial will provide you with all the key configuration details needed to run ConductR on EC2. It presumes a working knowledge of EC2 and does not provide click-by-click instructions for using EC2. Detailed instructions for all AWS steps discussed can be found in the [AWS documentation](https://aws.amazon.com/documentation/).

This section provides information for running in the advanced private agent mode but presumes use of the simple setup where all nodes run all three services. See (ConductR Architecture)[Overview#ConductRArchitecture] for more information.

### Preparing EC2

Begin by preparing the EC2 network and security environment. This tutorial uses a Virtual Private Cloud (VPC) with a Classless Inter-Domain Routing (CIDR) of `10.0.0.0/16` and example addresses will be based accordingly.

For better resilience, deploy nodes across multiple availability zones (AZ). In our example we will be deploying across three availability zones.

#### ELB Security Group

Create a security group called SG-ELB to provide access control from the ELB to the ConductR public agent nodes.

SG-ELB Inbound Rules

| Type    | Proto   | Port        | Source     | Description         |
| :------ | :-----  | :---------- | :--------- | :------------------ |
| HTTP    | TCP     | 80          | 0.0.0.0/0  | HTTP Public Access  |
| HTTPS   | TCP     | 443         | 0.0.0.0/0  | HTTPS Public Access |


#### Bastion Host Subnet and Security Group

> This is required for private agents but _optional_ for 'flat' clusters where all nodes have public ip addreses.

Create a subnet in any AZ. The bastion host will be hosted in this subnet, and thus only allowing SSH access from the outside world.

The bastion host will require SSH access to all ConductR nodes in the cluster. This subnet will need an internet gateway added to their route table for the destination `0.0.0.0/0`. This required so that our bastion host can access the internet.

In this case the subnet will be called SN-BASTION with a CIDR of `10.0.1.250/255`.

Create a security group called SG-BASTION to provide access control to the bastion host.

SG-BASTION Inbound Rules

| Type    | Proto   | Port        | Source     | Description       |
| :------ | :-----  | :---------- | :--------- | :---------------- |
| SSH     | TCP     | 22          | 0.0.0.0/0  | Remote SSH Access |



#### Public Subnets and Security Group

Create a subnet in each AZ. This subnet will be a public subnet, and thus accessible from outside world via ELB. The ConductR public agents will be hosted in this subnet.

Our example subnets names indicate their AZ. In this case the subnet will be called SN-PUBLIC-A with a CIDR of `10.0.1.100/124`, SN-PRIVATE-B with a CIDR of `10.0.2.100/124`, and SN-PRIVATE-C with a CIDR of `10.0.3.100/124`. Each subnet will need an internet gateway added to their route table for the destination `0.0.0.0/0`. This required so that our nodes can access the internet.

Create security group called SG-AGENT-PUBLIC which will provide access control to ConductR public agents.

SG-AGENT-PUBLIC Inbound Rules

| Type    | Proto   | Port        | Source     | Description                      |
| :------ | :-----  | :---------- | :--------- | :------------------------------- |
| SSH     | TCP     | 22          | SG-BASTION | Remote SSH Access                |
| Custom  | TCP     | 2552        | SG-CORE    | Akka Remoting from Core to Agent |
| HTTP    | TCP     | 9000        | SG-ELB     | HTTP application endpoints       |
| HTTP    | TCP     | 9009        | SG-ELB     | ELB health check                 |
| HTTP    | TCP     | 9999        | SG-ELB     | Visualizer                       |


#### Private Subnets

> Only create this subnet for use with private agent topology.

Create a subnet in each AZ. This subnet will be private, and thus not directly accessible from outside world. The ConductR core and the ConductR private agents will be hosted in this subnet.

Our example subnets names indicate their AZ. In this case the subnet will be called SN-PRIVATE-A with a CIDR of `10.0.1.0/49`, SN-PRIVATE-B with a CIDR of `10.0.2.0/49`, and SN-PRIVATE-C with a CIDR of `10.0.3.0/49`. Each subnet will need an internet gateway added to their route table for the destination `0.0.0.0/0`. This required so that our nodes can access the internet.

Create two security groups in the VPC named SG-AGENT-PRIVATE and SG-CORE.  SG-AGENT-PRIVATE will provide access control to the ConductR private agents, while SG-CORE to the ConductR core nodes.

SG-AGENT-PRIVATE Inbound Rules

| Type    | Proto   | Port        | Source          | Description                                   |
| :------ | :-----  | :---------- | :-------------- | :-------------------------------------------- |
| SSH     | TCP     | 22          | SG-BASTION      | Remote SSH Access                             |
| Custom  | TCP     | 2552        | SG-CORE         | Akka Remoting from Core to Agent              |
| HTTP    | TCP     | 10000-10999 | SG-AGENT-PUBLIC | Proxy access from ConductR public agent nodes |


SG-CORE Inbound Rules

| Type    | Proto   | Port        | Source           | Description                      |
| :------ | :-----  | :---------- | :--------------- | :------------------------------- |
| SSH     | TCP     | 22          | SG-BASTION       | Remote SSH Access                |
| Custom  | TCP     | 9004        | SG-AGENT-PRIVATE | Akka Remoting from Agent to Core |
| Custom  | TCP     | 9004        | SG-AGENT-PUBLIC  | Akka Remoting from Agent to Core |
| Custom  | TCP     | 9005        | SG-AGENT-PRIVATE | ConductR Control Protocol        |
| Custom  | TCP     | 9005        | SG-AGENT-PUBLIC  | ConductR Control Protocol        |
| Custom  | TCP     | 9006        | SG-AGENT-PRIVATE | ConductR Bundle Stream Server    |
| Custom  | TCP     | 9006        | SG-AGENT-PUBLIC  | ConductR Bundle Stream Server    |
| Custom  | TCP     | 9007        | SG-AGENT-PRIVATE | ConductR Status Server           |
| Custom  | TCP     | 9007        | SG-AGENT-PUBLIC  | ConductR Status Server           |
| Custom  | TCP     | 9008        | SG-AGENT-PRIVATE | ConductR Service Locator         |
| Custom  | TCP     | 9008        | SG-AGENT-PUBLIC  | ConductR Service Locator         |

#### Load Balancer

Create an external facing load balancer from the EC2 control panel. You will need to create an internet gateway and attach it to your VPC in order to have a public load balancer. We'll add an optional HTTPS protocol listener on port 443 to the default port 80 HTTP listener. For this tutorial we will map both of our listeners to instance port 9000 and port 9999. Add all public agent subnets to the load balancer and assign the load balancer to the SG-ELB security group. Optionally you can upload an SSL Certificate to use the ELB as your TLS endpoint if you added the HTTPS listener. For health monitoring we'll use ConductR's proxy status endpoint, HTTP:9009/status. This endpoint will return an OK when ConductR's proxy has been configured.

### Preparing the AMI

Launch a single instance of the desired base AMI to use as our image master. We'll use the Ubuntu 16.04 LTS HVM EBS-SSD boot image in US-East-1, e.g. ami-fd6e3bea. If you choose another base image, use an EBS boot image as they are much easy to image unless you know what your doing there. Be certain to assign a public ip address in instance details to make it easy to ssh into. For running core, agents and proxies separately, launch an instance per role for imaging.

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

The tutorial assumes that you have obtained the `conductr_%PLAY_VERSION%_all.deb` and `conductr-agent_%PLAY_VERSION%_all.deb` Debian package for ConductR Core and ConductR Agent respectively.

Secure copy (scp) the ConductR installation package to the image host and install ConductR Core and ConductR Agent as any other Debian package.

```bash
sudo dpkg -i conductr_%PLAY_VERSION%_all.deb
sudo dpkg -i conductr-agent_%PLAY_VERSION%_all.deb
```

ConductR Core and ConductR Agent are automatically registered as a service and started.

#### Installation miscellany

The ConductR Core service runs under the `conductr` user along with the `conductr` group. Its pid file is written to: `/var/run/conductr/running.pid` and its install location is `/usr/share/conductr`.

The ConductR Agent service runs under the `conductr-agent` user along with the `conductr-agent` group. Its pid file is written to: `/var/run/conductr-agent/running.pid` and its install location is `/usr/share/conductr-agent`.

#### Consolidated Logging

[[Consolidated logging|ConsolidatedLogging]] section describes the steps required which allow you to select the appropriate logging method for you. ConductR is bundled with an Elasticsearch based solution and is configured for that by default.

### Installing a Proxy

Proxying application endpoints is required when external communication to a service is required. We will be using `HAProxy`. Add a dedicated Personal Package Archive (PPA) and install HAProxy.

```bash
sudo add-apt-repository -y ppa:vbernat/haproxy-1.5
sudo apt-get update
sudo apt-get -y install haproxy
```

ConductR provides a ConductR-HAProxy bundle that listens for bundle events from ConductR and updates the local HAProxy configuration file accordingly. We must specifically allow the bundle to use `sudo` to reload HAProxy.

First, we have the user `conductr-agent` own the HAProxy config file.

```bash
[172.17.0.1]$ sudo chown conductr-agent:conductr-agent /etc/haproxy/haproxy.cfg
```

#### Preparing HAProxy reload script

After updating the HAProxy configuration file, ConductR-HAProxy will signal HAProxy to reload for the updated configuration.

Prepare the reload script in `/usr/bin/reloadHAProxy.sh`. ConductR-HAProxy will install its reload script in this location upon startup.

We will limit the bundle's sudo privileges to running a single script in `/usr/bin` for that purpose. Grant permissions to the `conductr-agent` user to write and run the `reloadHAProxy.sh` command. An addition to `/etc/sudoers` allows for using `sudo` without password for the `reloadHAProxy.sh` script. If a more specific reload sequence is required, a custom reload script can be specified using the CONDUCTR_RELOADHAPROXY_SCRIPT environment variable in a configuration bundle.

```bash
[172.17.0.1]$ sudo touch /usr/bin/reloadHAProxy.sh
[172.17.0.1]$ sudo chmod 0770 /usr/bin/reloadHAProxy.sh
[172.17.0.1]$ sudo chown conductr-agent:conductr-agent /usr/bin/reloadHAProxy.sh
[172.17.0.1]$ echo "conductr-agent ALL=(root) NOPASSWD: /usr/bin/reloadHAProxy.sh" | sudo tee -a /etc/sudoers
```

On RHEL and CentOS it may also be neccessary to [disable default requiretty](https://bugzilla.redhat.com/show_bug.cgi?id=1020147) for the `conductr-agent` user in `sudoers`.

```bash
[172.17.0.1]$ echo 'Defaults: conductr-agent  !requiretty' | sudo tee -a /etc/sudoers
```

#### Optional dependencies

##### Docker

ConductR supports running applications and services within Docker. If you plan on running Docker based bundles, you will need to install [Docker](https://docs.docker.com/) according to [the official documentation](https://docs.docker.com/installation/ubuntulinux/). Once Docker is installed then add the ConductR-Agent daemon user/group to the `docker` group so that it has [the correct permissions in order to access Docker](http://docs.docker.com/installation/ubuntulinux/#giving-non-root-access):

```bash
sudo usermod -a -G docker conductr-agent
```

### Create the AMI

With our packages installed we can create the ConductR machine image. Image the host by selecting the running instance in the EC2 dashboard and using the Create Image option from the Actions menus. If you created a host for each role, image each role. We are now done with the image host(s) and it/they can be terminated.

### Bring up the cluster

Once your ConductR AMI is available, launch three instances. In this tutorial we'll launch one instance into SN-A, SN-B and SN-C each so that our cluster spans three availability zones. All instances will be launched into our SG-Nodes security group. Be certain to assign public IP addresses to we can ssh into our nodes or use an admin bastion with private network access.

We will now configure ConductR on the instances and form a cluster. Repeat these steps on each of the ConductR core instances.

To be able to form an inter-machine cluster, ConductR Core must be configured to listen to the machine's private host interface. This can be enabled adding a property declaration for `CONDUCTR_IP` to the start command as follows:

```bash
echo -DCONDUCTR_IP=$(hostname -i) | sudo tee -a /usr/share/conductr/conf/conductr.ini
sudo service conductr restart
```

#### Specifying the seed node

Pick one node as the seed node and instruct the other two instances to use the other as the seed node. Here we have chosen `10.0.2.20` as the seed node and will perform this additional step on all other nodes *except* the seed node `10.0.2.20`.

```bash
echo --seed 10.0.2.20:9004 | sudo tee -a /usr/share/conductr/conf/conductr.ini
sudo service conductr restart
```

### Check the cluster
ConductR provides cluster and application information as well as its control interface via a REST API.
Use the CLI to view cluster membership

 ```bash
conduct members
```

A typical response contains the current members of the cluster (shown here is a three node cluster), the address of the node that the queried control server is running on and a list of unreachable nodes (shown here as empty).

``` bash
UID         ADDRESS                             ROLES       STATUS  REACHABLE
1503330106  akka.tcp://conductr@10.0.1.10:9004  replicator  Up            Yes
280222358   akka.tcp://conductr@10.0.2.20:9004  replicator  Up            Yes
-690829234  akka.tcp://conductr@10.0.3.30:9004  replicator  Up            Yes
```

### Configuring ConductR Agent

_Repeat each step in this section on each node running the agent service._

ConductR Agent needs to be connected to a ConductR core node in order for ConductR to run any application process. To establish this connection, configure ConductR Agent as such:

```bash
[172.17.0.1]$ echo -Dconductr.agent.ip=$(HOSTIP) | sudo tee -a /usr/share/conductr-agent/conf/conductr-agent.ini
[172.17.0.1]$ echo --core-node $(HOSTIP):9004 | sudo tee -a /usr/share/conductr-agent/conf/conductr-agent.ini
```

Once configured, restart the ConductR Agent service.

```bash
[172.17.0.1]$ sudo service conductr-agent restart
```


### Loading and Running ConductR-HAProxy Bundle

ConductR-HAProxy bundle listens for bundle changes within ConductR and updates the HAProxy config to expose the bundle endpoints accordingly.

#### Prepare ConductR-HAProxy nodes

_Repeat each step in this section on each node providing proxying._

ConductR-HAProxy bundle must be installed on all nodes where HAProxy is installed, and these nodes can be distinguished by the `haproxy` role. Assign the `haproxy` role to the nodes where the proxy will be hosted.

Append the `haproxy` role to the default `web` role as follows:

```bash
echo -Dconductr.agent.roles.0=web | sudo tee -a /usr/share/conductr-agent/conf/conductr-agent.ini
echo -Dconductr.agent.roles.1=haproxy | sudo tee -a /usr/share/conductr-agent/conf/conductr-agent.ini
sudo service conductr-agent restart
```
Note that role matching is not enforced unless [role matching is enabled](ClusterConfiguration#Roles) on the core nodes.

### Use CLI to load and run ConductR-HAProxy bundle

_Execute this step once for the entire cluster. This step would need to be repeated if the entire cluster is stopped and restarted, such as a development test lab._

These instructions for loading and running the ConductR-HAProxy bundle require the [[CLI|CLI]] to be installed. Continue with the next step once [[ConductR CLI|CLI]] is installed.

Load and run the ConductR-HAProxy bundle from the [bundles repo](https://bintray.com/typesafe/bundle) as follows.
 Scale ConductR-HAProxy so that ConductR-HAProxy is running on every proxy node in the cluster.
 In our case we have 3 nodes where the proxy is expected to be running, so we scale up the ConductR-HAProxy to 3 instances.
 We'll export the env var `CONDUCTR_IP` to target the local node. You can also use the `--host` option to specify the target.

```bash
export CONDUCTR_IP=$(hostname -i)
conduct load conductr-haproxy
conduct run conductr-haproxy --scale 3
```

That's it! You now have a cluster of three ConductR nodes ready to start running applications.

Add all cluster instances to the load balancer. Your cluster will be reachable by the DNS name specified in the load balancer description. You can add this as a CNAME to your DNS zone file to make the cluster reachable using a hostname in your domain.

ConductR comes with a `visualizer` sample application. Head over to the next section [[CLI|CLI]] to learn how to deploy visualizer application to your fresh ConductR cluster.

# DC/OS Installation

The following guide will outline the steps to deploy and run ConductR as a framework within [DC/OS](https://dcos.io/).

## Prerequisite

* An existing, working DC/OS 1.8 cluster. The [installation and setup of DC/OS](https://dcos.io/install/) cluster is outside of the scope of this guide.
* A working installation of DC/OS CLI tools successfully authenticated against the DC/OS cluster.
* For consolidated logging, ElasticSearch must be running in the cluster. Installing from Universe will satisfy this requirement.

## Deploy ConductR into DC/OS cluster

Obtain the ConductR's application definition JSON.

> In order to obtain the JSON required for installations of ConductR on DC/OS then please [contact our sales department](https://www.lightbend.com/company/contact). To evaluate ConductR in general then [please visit our product page](http://www.lightbend.com/products/conductr) which provides instructions on getting started. Otherwise if you are looking to use ConductR for free from a development perspective then please [head over to our developer section](DevQuickStart).

In `Services`, post the JSON file to deploy the ConductR Service. This can be done using json mode of the 'Deploy New Service' dialog. Refer to DC/OS documentation for deployment steps given the application definition JSON.

By default ConductR will be deployed with a single core scheduler instance. Upon startup ConductR will launch its agent executor process on each of the available Mesos agent nodes.

Wait until the ConductR instance's health is marked as healthy before proceeding.

If more instances are required, scale ConductR to the desired total schedulers using the `Instances` within `Edit Service` not the `Scale` dialog. Ensure that all of the instances are healthy before proceeding. Multiple core schedulers are recommended for resilience however schedulers are not required on all nodes.

_To stop a deployed and running ConductR in DC/OS cluster, be sure to follow the instructions on the [Cluster Stop](ClusterStop#DC/OS-mode) page._

## Integrate with the DC/OS CLI

These instructions for managing ConductR require the [[CLI|CLI]] to be installed.

First, prepare the DC/OS CLI:

```
$ conduct setup-dcos
```

You can now use `dcos conduct <conduct-subcommand>` to connect with your DC/OS cluster. For example, you should be able to query ConductR:

```
$ dcos conduct info
ID  NAME  TAG  #REP  #STR  #RUN  ROLES
```

## Installing a Proxy on Ubuntu

> Skip the next section and scroll down to ["Installing a Proxy on CoreOS"](#Installing-a-Proxy-on-CoreOS) when using CloudFormation.

_Perform each step in this section on all public nodes. For full resilience a proxy should be installed for each public node machine. The public node machines are machines assigned with `slave_public` role._

Proxying application endpoints is required when running more than one instance of ConductR; which should be always for production style scenarios. Proxying endpoints permits connectivity from both external callers and for bundle components to communicate with other bundle components. This also allows an external caller to contact an application that is running on any ConductR node by contacting any proxy instance.

ConductR-HAProxy requires `HAProxy` version 1.5 or newer.

Install HAProxy using the following commands.

```bash
$ sudo apt-get -y install haproxy
```

or

```bash
$ sudo yum install haproxy
```

On Red Hat Enterprise Linux (RHEL) 6, haproxy is in the RHEL Server Load Balancer (v6 for 64-bit x86_64) rhel-lb-for-rhel-6-server-rpms channel. You'll need to add this channel to your server.

On some Debian distributions you may need to add a dedicated Personal Package Archive (PPA) in order to install HAProxy 1.5 via the package manager. For example:

```bash
$ sudo add-apt-repository -y ppa:vbernat/haproxy-1.5
$ sudo apt-get update
$ sudo apt-get -y install haproxy
```

ConductR provides a ConductR-HAProxy bundle that listens for bundle events from ConductR and updates the local HAProxy configuration file accordingly. We must specifically allow the bundle to use `sudo` to reload HAProxy.

Under DC/OS installation, the ConductR-HAProxy bundle will execute as a task owned by a custom executor started by ConductR on the public node machine. For the steps below, replace `{executor-user}` with the actual name of the user which will execute the custom executor and ConductR-HAProxy bundle.

We have the executor user own the HAProxy config file.

```bash
$ sudo chown {executor-user}:{executor-user} /etc/haproxy/haproxy.cfg
```

Prepare the reload script in `/usr/bin/reloadHAProxy.sh`. ConductR-HAProxy will install its reload script in this location upon startup.

We will limit the bundle's sudo privileges to running `/usr/bin/reloadHAProxy.sh`. Grant permissions to the executor user to run the `reloadHAProxy.sh` command. An addition to `/etc/sudoers` allows for using `sudo` without password for the `reloadHAProxy.sh` script.

```bash
[172.17.0.1]$ sudo touch /usr/bin/reloadHAProxy.sh
[172.17.0.1]$ sudo chmod 0770 /usr/bin/reloadHAProxy.sh
[172.17.0.1]$ sudo chown {executor-user}:{executor-user} /usr/bin/reloadHAProxy.sh
[172.17.0.1]$ echo "{executor-user} ALL=(root) NOPASSWD: /usr/bin/reloadHAProxy.sh" | sudo tee -a /etc/sudoers
```

On RHEL and CentOS it may also be neccessary to [disable default requiretty](https://bugzilla.redhat.com/show_bug.cgi?id=1020147) for the named execution user in `sudoers`.

```bash
$ echo 'Defaults: {executor-user}  !requiretty' | sudo tee -a /etc/sudoers
```

Skip the next section and scroll down to ["Loading and Running ConductR-HAProxy Bundle"](#Loading_and_Running_ConductR-HAProxy_Bundle).

## Installing a Proxy on CoreOS

_Perform each step in this section on all public nodes. For full resilience a proxy should be installed for each public node machine. The public node machines are machines assigned with `slave_public` role._

Proxying application endpoints is required when running more than one instance of ConductR; which should be always for production style scenarios. Proxying endpoints permits connectivity from both external callers and for bundle components to communicate with other bundle components. This also allows an external caller to contact an application that is running on any ConductR node by contacting any proxy instance.

ConductR-HAProxy requires `HAProxy` version 1.5 or newer.

The CloudFormation template will build the public node machines running on CoreOS, and HAProxy will only be available through Docker. We will use HAProxy version 1.5 official Docker image published to [Docker Hub](https://hub.docker.com/_/haproxy/).

The CloudFormation template will assign `root` user to execute custom executors and its task, and as such ConductR-HAProxy process will be run under the `root` user.

The HAProxy Docker image doesn't ship with any configuration files. As such, we will need to provide a basic HAProxy configuration.

```bash
$ sudo mkdir /etc/haproxy
$ sudo touch /etc/haproxy/haproxy.cfg
```

Populate the `/etc/haproxy/haproxy.cfg` file with the following configuration.

```
defaults
    log global
    mode    http
    option  httplog
    option  dontlognull
    timeout connect 5000
    timeout client  50000
    timeout server  50000

# This is a test endpoint to validate that HAProxy has been started successfully
frontend monitor
  bind :65535
  mode http
  monitor-uri /test
```

### Running the HAProxy Docker container

Please ensure the hostname is set correctly or substitute your addresses as appropriate for $(HOSTIP). To set the hostname, pass the ip address to hostname.

```bash
sudo hostname $(hostname -i)
```

Run the HAProxy Docker container.

```
$ docker run -d --name haproxy  -p $(HOSTIP):80:80 -p $(HOSTIP):443:443 -p $(HOSTIP):8999:8999 -p $(HOSTIP):9000:9000 -p $(HOSTIP):9999:9999 -p $(HOSTIP):65535:65535 -v /etc/haproxy:/usr/local/etc/haproxy:ro haproxy:1.5
```

The above container has `haproxy` as its name and uses configuration file is located at `/etc/haproxy/haproxy.cfg`. The directory `/etc/haproxy` is mounted within the container on `/usr/local/etc/haproxy`. This will allow updates to `/etc/haproxy/haproxy.cfg` to be visible within the container.

Any application bundle service ports must be exposed for proxying. The container started above exposes the ports `80`, `443`, `8999`, `9000`, `9999`, and `65535.`
 This is to expose ports for HTTP/HTTPS and statistics as well as for applications on `9000`, ConductR Visualizer on `9999` and for the HAProxy test endpoint.
 You can include additional ports or port ranges for your endpoints using the `-p` option as required.

NOTE: The ports exposed by the container must be bound to the cluster address, generally the internal IP address, to be used as the service delivery interface for the node using the `hostname` command.
  You must verify or set your system hostname or otherwise substitute the $(HOSTIP) in the above example with the correct IP address for  your environment.

Run `docker ps -a` to ensure the container has been started successfully.

```
$ docker ps -a
CONTAINER ID        IMAGE               COMMAND                CREATED             STATUS              PORTS                                                                               NAMES
534236e89eac        haproxy:1.5         "/docker-entrypoint.   15 seconds ago      Up 15 seconds       10.0.7.118:80->80/tcp, 10.0.7.118:443->443/tcp, 10.0.7.118:8999->8999/tcp, 10.0.7.118:9000->9000/tcp, 10.0.7.118:9999->9999/tcp, 10.0.7.118:65535->65535/tcp   haproxy
```

An additional check can be performed by using `curl` command against the HAProxy test endpoint.

```
$ curl -v http://$(HOSTIP):65535/test
*   Trying 10.0.7.118...
* Connected to 10.0.7.118 (10.0.7.118) port 65535 (#0)
> GET /test HTTP/1.1
> Host: 10.0.7.118:65535
> User-Agent: curl/7.48.0
> Accept: */*
>
* HTTP 1.0, assume close after body
< HTTP/1.0 200 OK
< Cache-Control: no-cache
< Connection: close
< Content-Type: text/html
<
<html><body><h1>200 OK</h1>
Service ready.
</body></html>
* Closing connection 0
```

### Preparing the HAProxy reload script

After updating the HAProxy configuration file, ConductR-HAProxy will signal HAProxy to reload for the updated configuration.

For ConductR-HAProxy, the HAProxy reload script is located at `/usr/bin/reloadHAProxy.sh` by default. However on CoreOS the `/usr` is a read-only volume, and thus the HAProxy reload script must be made available elsewhere.

We will use `/etc/haproxy/reloadHAProxy.sh.` ConductR-HAProxy will install the reload script to this location upon startup.

```bash
$ sudo touch /etc/haproxy/reloadHAProxy.sh
$ sudo chmod 0770 /etc/haproxy/reloadHAProxy.sh
```

### Customizing ConductR-HAProxy Bundle

_The use of a configuration bundle is required as that we're using Docker-based HAProxy with a non-default HAProxy script reload location._

A configuration bundle has been prepared using the above configuration. If you are not using the `/etc/haproxy` configuration for CoreOS public nodes described above, you will need to update the configuration bundle accordingly.

## Loading and Running ConductR-HAProxy Bundle

The ConductR-HAProxy bundle will automatically update HAProxy to expose bundle services for access from outside cluster via the public service interface. ConductR-HAProxy will ensure the HAProxy configuration is kept up to date based on the bundles which are running in the cluster.

### Use CLI to load and run ConductR-HAProxy bundle

Upload the ConductR-HAProxy bundle to the bastion host and use the [[CLI|CLI]] to load the ConductR-HAProxy bundle from the [bundles repo](https://bintray.com/typesafe/bundle).

For Linux (not bundle configuration):

```bash
$ dcos conduct load conductr-haproxy
```

For CoreOS:

Load `conductr-haproxy` with the `conductr-haproxy-coreos` configuration bundles:

```
$ dcos conduct load conductr-haproxy conductr-haproxy-coreos
```

Scale ConductR-HAProxy so that ConductR-HAProxy is running on every proxy node in the cluster. In our case we have 3 nodes where the proxy is expected to be running, so we scale up the ConductR-HAProxy to 3 instances.

```bash
$ dcos conduct run conductr-haproxy --scale 3
```

## Verifying bundle startup

Use the command `conduct info` from the bastion host to verify successful startup of the bundle.

```bash
$ dcos conduct info
ID       NAME          TAG  #REP  #STR  #RUN  ROLES
6e68f05  visualizer  2.0.0     3     0     1  web
```

In the example above, a bundle called `visualizer` has been started successfully.

This can also be confirmed by executing the `dcos task` command which will display the bundle running as a task from the context of DC/OS. The `visualizer:2.0.0` task belongs to the currently running `visualizer` bundle with the tag `2.0.0`.

```bash
$ dcos task
NAME              HOST       USER  STATE  ID
visualizer:2.0.0  10.0.3.75  root    R    6e68f055d1f5715ad3ff19172fa5efaf_0f6e119c-9288-425a-89e3-36379dcaccda
```

The `compatibilityVersion` is explained in the to [bundle configuration](BundleConfiguration).

The bundle can now be scaled to run multiple instances.

```bash
$ dcos conduct run --scale=3 visual
```

Test service endpoints from the public nodes to ensure connectivity. The `/etc/haproxy/haproxy.cfg` file on the public nodes contains the frontend and backend configurations for services exposed by bundles. Test the front end bind port and path using `curl` and the public node ip address to ensure proxying is working correctly.

## Configuring the service gateway

Now that we have configured ConductR-HAProxy to run on the public nodes, we'll want to configure them as a public service. If the public nodes have public IP addresses, the addresses can be listed in DNS directly.

If we do not want to have multiple "A" records, we can use a load balancer, such as EC2's ELB, for the service CNAME address. In doing such, we will create a new a load balancer instance for the ConductR services as this allows the load balancer's health check to use ConductR-HAProxy's health endpoint, HTTP:9009/status. This endpoint will return a 200 when ConductR's proxy is operating correctly. In the example of AWS ELB, we create a new ELB using the same (or similar custom) subnet and security group as the public nodes with internet access. Add listeners to map public ports such as 80 and 443 on the public CNAME to serviced ports on the public proxy nodes and ensure security group rules allow the ELB to access the public nodes on the service ports. By default conductr-haproxy will serve http on port 9000 and so you will typically want to serve port 9000 from either 80 or 443 (the latter being preferred given security).
