+++
title = "Tripleo Installer — The Good, the Bad and the Ugly"
date = "2016-03-27"
slug = "2016/03/27/tripleo-installer-the-good"
Categories = [ "cloud" ]
+++

[TripleO](https://wiki.openstack.org/wiki/TripleO) is an OpenStack deployment and management tool I've been using since the Kilo release of OpenStack. It does its job pretty well, however not everything is perfect. My experience presented in this article applies more or less to the Red Hat's OpenStack director too, as the Red Hat OpenStack director is a downstream version of TripleO.

<!--more-->

## The good things about TripleO

### TripleO is a great idea

TripleO, aka OpenStack-on-OpenStack, installs OpenStack cluster using OpenStack. At first, a minimum one-node OpenStack installation is created which is in turn used to provision a much bigger workload OpenStack cluster. I find this TripleO idea amazing. If OpenStack is the best way to manage your infrastructure, then why use something else to install it? As an administrator I would prefer to provision my OpenStack nodes with Ironic before introducing yet another tool like [Cobbler](http://cobbler.github.io/) to do the same job. Needless to say that as the Ironic and Heat components improve, so improves the OpenStack installation experience.

One could argue that using the OpenStack to form an installer comes with a ton of complexity when installing the installer itself. In my experience, however, the installation of the undercloud OpenStack using the provided Puppet scripts doesn't impose any problem.

### TripleO has a vibrant community

TripleO is used to continuously deploy and test the OpenStack cloud during its development. The RDO project adopted TripleO as their OpenStack installation tool. Red Hat derives their OpenStack director installer from the RDO project. A large community of TripleO users is a great plus.

## The bad things about TripleO

### Configuration flexibility

TripleO installer consists of a bunch of Heat templates to orchestrate the overcloud image provisioning and a number of Puppet and shell scripts for the following configuration of the overcloud nodes. These templates and scripts are heavily developed from release to release as the new TripleO features come in. To avoid the upgrade headaches, you should not modify the TripleO templates and scripts directly. Instead, TripleO provides extension points (via extra config) where you can put your customizations. This didn't work for me. My goal was to deploy an Ironic service in the overcloud OpenStack. For that to work, I needed to provision an additional undercloud network including a VIP for the load balancer. This was not possible without patching the Heat templates and Puppet scripts. I dread the day when I'll have to port these patches to the next TripleO release.

Furthermore, the current way to modify OpenStack configuration properties is less straight forward. To configure a property, I have to first grep through the Puppet scripts to find out whether the desired property is managed by Puppet or not. Afterwards, I grep through the TripleO Heat templates to find out whether TripleO provides a direct template parameter to set the Puppet variable or not. Afterwards, I can either pass the parameter to the TripleO template or I set the Puppet variable in the extra config section or I'm on my own.

{% blockquote %}
I'd like to be able to easily modify any property in any configuration file on any OpenStack node.
{% endblockquote %}

OpenStack comes with tons of configuration properties and I think it would be great to have a more straight forward way to configure them.

### Deployment control

TripleO uses Heat to deploy and configure the overcloud OpenStack. Heat orchestrates the infrastracture based on the description provided by the user in the Heat templates. In the Heat templates, we tell Heat what our deployment should look like, but we have no control over the steps Heat will take to get to the desired state. I find this lack of control rather problematic.

{% blockquote %}
A fine-grained deployment control would be desirable.
{% endblockquote %}

Let's say I have an overcloud consisting of 100 nodes. After changing the configuration in my Heat templates, I can only re-run the entire Heat configuration process and hope that I won't end up with a broken cloud. Instead, I'd like to apply the configuration changes to a couple of nodes to make sure that everything works before I continue with the rest of the cloud. The ability to apply only part of the configuration would be useful as well.

## The ugly experience with TripleO

I'd like to share one scary experience I had with the TripleO installer. While using TripleO for a couple of months, I have to say that this was the only serious problem I've encountered.

One day I uploaded an updated node image into the undercloud OpenStack. I was about to create new nodes in the overcloud cluster and wanted to have them provisioned with this new image. After starting the Heat stack update, it occurred to me that the processing took longer than usual. Well, after I SSHed into the overcloud nodes I realized why. Heat simply wiped out the entire disk content of the existing nodes and replaced it with the fresh disk image. Wow, my entire workload cloud was gone!

I learned that when you update the disk image in the undercloud, Heat will find out what nodes have to be updated and will simply replace their disk content with the new image. If you are orchestrating cloud deployments where your machines are cattle, this is what you want, however:

{% blockquote %}
The overcloud baremetal nodes are pets and should not be handled as cattle.
{% endblockquote %}

To protect the overcloud nodes from deletion, I run the following command for each node against the undercloud Nova database:
{% codeblock lang:sql %}
UPDATE instances SET disable_terminate = 1 WHERE uuid = '<uuid of the overcloud instance>';
{% endcodeblock %}

So far, I haven't found a better way how to do it. This effectively prevents deleting the node whether by issuing a `nova delete` command or by Heat when updating the stack.

## Conclusion and suggestions

TripleO installer is a great tool to deploy an OpenStack cloud. It's backed by a large user community and doesn't invent any new tools to install OpenStack.

On the other hand, I'm somewhat sceptical about Heat being the right tool to do software configuration. Funneling the configuration options through the Heat templates down to the Puppet scripts seems cumbersome to me.

I'd like to suggest the following approach: let Heat do the node provisioning, network configuration and perhaps a minimum node setup using cloud-init. At the end of the deployment, Heat would provide the information about the deployment in the format understandable to the configuration management tools like Puppet, Chef or Ansible. The configuration management tool then merges the facts provided by Heat with the tons of OpenStack configuration settings provided by the user. The following OpenStack installation, configuration, and later orchestration would solely be done by the configuration management tool more suitable for this job. Heat would not be involved at all in this stage.
