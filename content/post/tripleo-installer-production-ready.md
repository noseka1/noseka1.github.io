+++
title = "Tripleo Installer, Production Ready?"
date = "2017-01-15"
slug = "2017/01/15/tripleo-installer-production-ready"
Categories = [ "cloud", "devops" ]
+++

[TripleO](https://wiki.openstack.org/wiki/TripleO) is an OpenStack deployment and management tool we've been using on the production systems for a while now. As TripleO is an upstream project for the Red Hat OpenStack Platform Director one would expect a decently working tool able to manage large-scale OpenStack deployments. What is our experience with TripleO?

<!--more-->

## Introduction

Six months have passed since we deployed a private cloud in our company. Our cloud is based on the RDO distribution of OpenStack Mitaka running on top of RHEL 7. I have to say that we're very happy with our cloud-based environment. OpenStack simplified the management of virtual machines and boosted the productivity of our engineering team which enjoys the self-service provided by the OpenStack APIs. Our test automation creates and destroys many virtual machines a day making sure that our software product is tested in a clean and well-defined environment. OpenStack quickly became a critical part of our infrastructure.

Hence we were less pleased when the last week a routine maintenance of the OpenStack cluster turned into an unplanned downtime of two compute nodes. But before we get to the problem itself let me introduce you to the specifics of how we manage the OpenStack cluster.

## Overcloud maintenance is a challenge

A cloud life-cycle management tool of choice in the RDO distribution is TripleO. I published an article about my initial experience with TripleO a while ago: [TripleO Installer - the Good, the Bad and the Ugly](/blog/2016/03/27/tripleo-installer-the-good/). Overall, the way how TripleO configures the OpenStack cluster is rather less flexible. After spending time on customizing and patching TripleO we decided that there must be an easier way. Eventually, we implemented our own set of Ansible scripts that allow an additional fine-grained configuration of OpenStack nodes. After the `openstack overcloud deploy` command is complete we run our Ansible scripts to apply an additional configuration to the overcloud. There are two benefits to this approach. First, we don't have to patch TripleO scripts which will be upgraded in the next release of OpenStack. And second, we can keep using Ansible which is our favorite configuration tool.

Having updated the overcloud using TripleO several times we realized that the update procedure is rather unreliable. Some time the TripleO update would fail with an error. Other time the overcloud update would just hang forever. Probably due to the undeterministic behaviour of the Puppet scripts that constitute a substantial part of TripleO we experienced random errors that would not occur again after restarting the update procedure. Situation got worse after we configured the overcloud Keystone to authenticate OpenStack users against Active Directory. The overcloud update would not run into completion anymore due to a defect in the Puppet scripts.

Because fixing the TripleO scripts would require additional effort and the overcloud update would remain a risky operation either way we concluded that we will require a downtime when updating the overcloud. During the downtime period the existing virtual machines are fully operational only the OpenStack services that allow users to create or delete virtual machines or other cloud resources are not available. In the case of our private cloud this was an acceptable albeit not ideal solution.

In summary, we can depict our OpenStack maintenance process like this:

{{< figure src="/images/posts/openstack_maintenance_process.svg" height="500" width="700" class="center" alt="OpenStack Maintenance Process" >}}

## TripleO installer and the resulting downtime

On all our OpenStack nodes we use bonded network interfaces to protect the nodes against network failures. In network interface bonding a pair of physical network interfaces is combined into a single logical interface. This provides redundancy by allowing failover from one physical interface to another in the case of failure.

It happened to us that on two of our compute nodes one physical network interface per bond was not working. In this situation the network connection is still functional but not redundant anymore. Unfortunately, for the TripleO installer this was not good enough. Normally, during the overcloud update the `os-net-config` utility configures the node networking. Due to the single network interface down `os-net-config` failed to create a correct network configuration. A "safe" default configuration was generated instead which configured all available network interfaces to use DHCP. Unfortunately, we prefer a static network configuration of overcloud nodes and so no DHCP server was available. Hence this "safe" default configuration rendered the two compute nodes unreachable including all the virtual machines that were running on top of them!

We were able to fix the networking issue on the first compute node quickly. However, the physical network interfaces on the second compute node were seriously falling apart. Unfortunately:

>The TripleO installer requires that all the overcloud nodes are reachable during the overcloud update.

In the opposite case the update just stays hanging. It turned out that it was not possible to bring all the network interfaces on the second compute node up but eventually we were able to get at least the management interface working. This allowed us to re-run the overcloud update during which we fooled the TripleO installer to believe that the configuration of the problematic compute node was applied sucessfully. After exceeding the two-hour maintanance window by several hours we were finally done.

## Conclusion

Here I'd like to summarize our six-months long experience with the TripleO installer:

1. In our experience, the configuration of the OpenStack cluster using only the TripleO installer is not flexible enough. As a workaround, we ended up writing a bunch of Ansible scripts.
2. The overcloud update can take a very long time to complete and it can fail because of random errors. Also during the update operation all the overcloud nodes have to be reachable by the TripleO installer. For this reason, I personally cannot imagine using TripleO to manage a cluster with more than one hundred nodes.
3. As we experienced, the TripleO installer can easily break a working OpenStack cluster. This is a big no-no for a production system.

Overall, I think that the TripleO installer in the Mitaka version of OpenStack would need more work to become production ready. In the meantime, we're continuing with patching of what we have.

In the future, there are other projects that could replace the TripleO installer. I found the [kolla-ansible](https://github.com/openstack/kolla-ansible) and [kolla-kubernetes](https://github.com/openstack/kolla-kubernetes) rather promising.
