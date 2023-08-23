+++
title = "Installing Openstack Liberty on RHEL7"
date = "2015-10-19"
slug = "2015/10/19/installing-openstack-liberty-on-rhel7"
Categories = [ "devops", "cloud" ]
+++

The OpenStack Liberty was released last week. In this article I'll briefly describe how to deploy the OpenStack Liberty on RHEL7 using RDO Manager.

<!--more-->

The [RDO project](https://www.rdoproject.org/ "RDO project") packages the OpenStack software for the Red Hat based platforms. Currently, there are Liberty packages in status testing/release candidate available from the project. Apart from a couple of configuration issues the installation went pretty well and I obtained a basic 2-node cluster.

If you intalled OpenStack Kilo using RDO Manager before I have a good news for you. The installation procedure remains pretty much the same. You can follow the [great guide](http://docs.openstack.org/developer/tripleo-docs/ "TripleO Doc") provided by the TripleO project to get the installation going. First, add the following two repositories:

{{< highlight plaintext "linenos=table" >}}
http://trunk.rdoproject.org/centos7/current-tripleo/delorean.repo
http://trunk.rdoproject.org/centos7/delorean-deps.repo
{{< / highlight >}}

Now you can install the RDO Manager with:

{{< highlight shell "linenos=table" >}}
$ sudo yum install python-tripleoclient
{{< / highlight >}}

In the Kilo release, the `python-tripleoclient` package was called `python-rdomanager-oscplugin`. You can continue with the guide. To build the overcloud images I issue the commands:

{{< highlight shell "linenos=table" >}}
$ export NODE_DIST=rhel7
$ export DIB_LOCAL_IMAGE=rhel-guest-image-7.1-20150224.0.x86_64.qcow2
$ export REG_METHOD=disable
$ export DIB_DEBUG_TRACE=1
$ export DIB_YUM_REPO_CONF=/etc/yum.repos.d/rhel7_mirror.repo
$ export USE_DELOREAN_TRUNK=1
$ export DELOREAN_TRUNK_REPO=http://trunk.rdoproject.org/centos7/current-tripleo
$ export DELOREAN_REPO_FILE=delorean.repo
$ openstack overcloud image build --all
{{< / highlight >}}

As you can see, I'm not really registering the OpenStack nodes with the Red Hat portal. Instead, I'm pulling the RHEL7 packages from the local mirror.

After the installation of the overcloud has completed I realized that some of the OpenStack processes on the overcloud nodes were segfaulting. After switching SELinux from enforcing to permissive mode everything started working as expected.

# A final note

The deployment of OpenStack is rather an involved process even when leveraging the tools like RDO Manager. To truly automate the installation in my specific environment I use a set of Ansible scripts to drive the RDO manager installation. Now that Red Hat acquired Ansible I'm wondering if we're going to get an even better OpenStack installation experience on Red Hat based distributions.
