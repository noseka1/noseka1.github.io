---
layout: post
title: "Deploying Kubernetes on OpenStack using Heat"
date: 2016-06-26 08:28:11 -0700
comments: true
categories: cloud devops
---

Want to install Kubernetes on top of OpenStack? There are [many ways](http://kubernetes.io/docs/getting-started-guides/) how to install a Kubernetes cluster. The upcoming Kubernetes 1.3 release comes with yet another method called [OpenStack Heat](http://kubernetes.io/docs/getting-started-guides/openstack-heat/). In this article, we're going to explore this deployment method when creating a minimum Kubernetes cluster on top of OpenStack.

<!-- more -->

In this tutorial, there are three OpenStack virtual machines involved. The first machine, called the *Kubernetes installer* machine, is created manually and is used for compiling Kubernetes from source and running the Kubernetes installer. The other two OpenStack machines, *Kubernetes master* and *Kubernetes node*, are created during the installation process.

The Kubernetes installer machine and both of the Kubernetes machines run on the CentOS-7-x86_64-GenericCloud-1605 image. You can download this image from the [CentOS image repository](http://cloud.centos.org/centos/7/images/). After I uploaded the CentOS 7 image into OpenStack, it has been assigned ID `17e4e783-321c-48c1-9308-6f99d67c5fa6` for me.

## Building Kubernetes from source

First off, let's spin up a Kubernetes installer machine in OpenStack. I recommend using the `m1.large` flavor that comes with 8 GB of RAM. The compilation of Kubernetes is rather memory intensive.

To ensure consistent and reproducible builds, a Docker container is created at the beginning of the build process and the build proceeds within the container. So, let's quickly setup Docker on our build machine:

{% codeblock lang:sh %}
sudo yum install docker
{% endcodeblock %}

Configure the Docker service to start on boot and then start it:

{% codeblock lang:sh %}
sudo systemctl enable docker
sudo systemctl start docker
{% endcodeblock %}

The Kubernetes build scripts expect that the `docker` command can successfully contact the Docker daemon. In the default CentOS configuration, the `sudo docker` is required in order to connect to the `/var/run/docker.sock` socket which is owned by the user root. To overcome the permission problem, let's create a wrapper script that will invoke the `docker` command using `sudo`:

{% codeblock lang:sh %}
mkdir bin
echo -e '#!/bin/bash\nexec sudo /usr/bin/docker "$@"' > bin/docker
chmod 755 bin/docker
export PATH=~/bin:$PATH
{% endcodeblock %}

You can test your changes with the `docker info` command which should work now.

Kubernetes is written in the Go language and its source code is stored in a Git repository. So, let's install the Go language environment and Git:

{% codeblock lang:sh %}
sudo yum install golang git
{% endcodeblock %}

Next we'll clone the Kubernetes Git repository and start the build. The `quick-release` make target creates a build for the amd64 architecture only and doesn't run any tests.

{% codeblock lang:sh %}
git clone https://github.com/kubernetes/kubernetes.git
cd kubernetes
make quick-release
{% endcodeblock %}

After about 15 minutes when the build was successful, you'll find the distribution tarballs `kubernetes.tar.gz` and `kubernetes-salt.tar.gz` in the `_output/release-tars` directory.

## Setting up the OpenStack CLI tools

The Kubernetes installer uses the OpenStack CLI tools to talk to OpenStack in order to create a Kubernetes cluster. Before you can install the OpenStack CLI tools on CentOS 7, you have to enable the OpenStack Mitaka RPM repository:

{% codeblock lang:sh %}
sudo yum install centos-release-openstack-mitaka
{% endcodeblock %}

Install the OpenStack CLI tools that are used by the Kubernetes installer when creating a cluster with:

{% codeblock lang:sh %}
sudo yum install python-openstackclient python-swiftclient python-glanceclient python-novaclient python-heatclient
{% endcodeblock %}

Next, you have to obtain your OpenStack `openrc.sh` file and source it into your environment:

{% codeblock lang:sh %}
. openrc.sh
{% endcodeblock %}

You should be able to talk to OpenStack now. For example, check if you can list the available OpenStack networks with:
{% codeblock lang:sh %}
openstack network list
{% endcodeblock %}

## Configuring the Kubernetes installer

In this section, we're going to more or less follow the instructions found in the chapter [OpenStack Heat](http://kubernetes.io/docs/getting-started-guides/openstack-heat/) of the Kubernetes documentation.

When deploying the Kubernetes cluster, the installer executes the following steps that you can find in `cluster/openstack-heat/util.sh`:

* Upload the distribution tarballs `kubernetes.tar.gz` and `kubernetes-salt.tar.gz` into the `kubernetes` container in Swift
* Upload the virtual machine image for the Kubernetes VMs into Glance
* Add the user's keypair into Nova
* Run a Heat script in order to create the Kubernetes VMs and put them on a newly created private network. Create a router connecting the private network with an external network.
* At the first boot, the Kubernetes VMs download the distribution tarballs from Swift and install the Kubernetes software using Salt

Let's create an `openstack-heat.sh` file with the configuration values for the Kubernetes installer:
{% codeblock lang:sh %}
export KUBERNETES_PROVIDER=openstack-heat
export STACK_NAME=kubernetes
export KUBERNETES_KEYPAIR_NAME=mykeypair
export NUMBER_OF_MINIONS=1
export MAX_NUMBER_OF_MINIONS=1
export EXTERNAL_NETWORK=gateway
export CREATE_IMAGE=false
export DOWNLOAD_IMAGE=false
export IMAGE_ID=17e4e783-321c-48c1-9308-6f99d67c5fa6
export DNS_SERVER=10.0.0.10
export SWIFT_SERVER_URL=https://openstack.localdomain:13808/swift/v1
{% endcodeblock %}

The above configuration will create exactly one Kubernetes master and one Kubernetes node. It will inject the keypair called `mykeypair` into both of them. Note that you have to ensure that the keypair `mykeypair` exists in Nova before proceeding. You probably want to change the name of the external network to a network available in your OpenStack. We're going to use the same CentOS 7 image for both of our Kubernetes VMs. This CentOS image has already been uploaded into OpenStack and in my case it was assigned ID `17e4e783-321c-48c1-9308-6f99d67c5fa6`. You also want to change the IP address of the DNS server to something that suits your environment. The Swift server URL is the public endpoint of your Swift server that you can obtain from the output of the command `openstack catalog show object-store`.

When your configuration is ready, you can source it into your environment:

{% codeblock lang:sh %}
. openstack-heat.sh
{% endcodeblock %}

Next, in my environment I had a problem where the IP range of the private network created by Kubernetes collided with the existing corporate network in my company. I had to directly edit the file `cluster/openstack-heat/kubernetes-heat/kubecluster.yaml` to change the `10.0.0.0/24` CIDR to something like `10.123.0.0/24`. If you don't have this problem you can safely use the default settings.

The Kubernetes cluster can leverage the underlying OpenStack cloud to attach existing Cinder volumes to the Kubernetes pods and to create external loadbalancers. For this to work, Kubernetes has to know how to connect to OpenStack APIs. With regard to the external loadbalancers, we also need to tell Kubernetes what Neutron subnet the loadbalancer's VIP should be placed on.

The OpenStack configuration can be found in the *cloud-config* script `cluster/openstack-heat/kubernetes-heat/fragments/configure-salt.yaml`. You can see that this script will create a configuration file `/srv/kubernetes/openstack.conf` on the Kubernetes machine which contains the OpenStack settings. In my case, I changed the original block:

{% codeblock %}
[Global]
auth-url=$OS_AUTH_URL
username=$OS_USERNAME
password=$OS_PASSWORD
region=$OS_REGION_NAME
tenant-id=$OS_TENANT_ID
{% endcodeblock %}

to read:

{% codeblock %}
[Global]
auth-url=$OS_AUTH_URL
username=$OS_USERNAME
password=$OS_PASSWORD
region=$OS_REGION_NAME
tenant-id=$OS_TENANT_ID
domain-name=MyDomain # Keystone V3 domain
[LoadBalancer]
lb-version=v1
subnet-id=73f8eb91-90cf-42f4-85d0-dcff44077313
{% endcodeblock %}

Besides adding the `LoadBalancer` section, I also appended the `domain-name` option to the end of the `Global` section, as in my OpenStack environment I want to authenticate against a non-default Keystone V3 domain.

## Installing the Kubernetes cluster

After you've sourced both the `openrc.sh` and `openstack-heat.sh` environment settings, you can kick off the installation of the Kubernetes cluster with:

{% codeblock lang:sh %}
./cluster/kube-up.sh
{% endcodeblock %}

After about 25 minutes, you should have a Kubernetes cluster up and running. You can check the status of the Kubernetes pods with the command:

{% codeblock lang:sh %}
./cluster/kubectl.sh get pods --namespace kube-system
{% endcodeblock %}

All pods should be running. The network topology of the Kubernetes cluster as displayed by Horizon:

{% img center /images/posts/kube.png %}
