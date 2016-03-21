---
layout: post
title: "Assigning Roles to Nodes Directly in RDO"
date: 2015-11-09 20:49:40 -0800
comments: true
categories: cloud devops
---

RDO Manager defines multiple roles that nodes can play in OpenStack deployment. For large-sized installations, RDO features automatic assignment of roles to nodes. This assignment is based on the facts that RDO obtained about each node during the introspection. However, for smaller deployments, you might prefer to assign the roles to the available nodes by hand. It was not straight forward for me to find out about this manual option even when it is described in the [TripleO documentation](http://docs.openstack.org/developer/tripleo-docs/advanced_deployment/profile_matching.html#optional-manually-add-the-profiles-to-the-nodes "TripleO documentation"). Let's review the required configuration steps in this blogpost.

<!-- more -->

The relationship between roles and nodes is organized via flavors. A flavor is a set of properties that the target node must match in order to be eligible for deployment of a specific role. The manual assignment of a role to a node is a three-step process:

1. Define a flavor with a property `capabilities:profile` set to the role name
2. Add the same profile to the capabilities list of the target node
3. Tell RDO what flavor to use for a specific role when beginning the deployment

The creation of flavors with the associated `capabilities:profile` property looks like this:

{% codeblock lang:sh %}
openstack flavor create --id auto --ram 4096 --disk 40 --vcpus 1 ceph
openstack flavor create --id auto --ram 4096 --disk 40 --vcpus 1 cinder
openstack flavor create --id auto --ram 4096 --disk 40 --vcpus 1 compute
openstack flavor create --id auto --ram 4096 --disk 40 --vcpus 1 controller
openstack flavor create --id auto --ram 4096 --disk 40 --vcpus 1 swift

openstack flavor set --property "cpu_arch"="x86_64" --property "capabilities:boot_option"="local" --property "capabilities:profile"="ceph" ceph
openstack flavor set --property "cpu_arch"="x86_64" --property "capabilities:boot_option"="local" --property "capabilities:profile"="cinder" cinder
openstack flavor set --property "cpu_arch"="x86_64" --property "capabilities:boot_option"="local" --property "capabilities:profile"="compute" compute
openstack flavor set --property "cpu_arch"="x86_64" --property "capabilities:boot_option"="local" --property "capabilities:profile"="controller" controller
openstack flavor set --property "cpu_arch"="x86_64" --property "capabilities:boot_option"="local" --property "capabilities:profile"="swift" swift
{% endcodeblock %}

Now we need to add the profiles to the capabilities list of the respective nodes:

{% codeblock lang:sh %}
ironic node-update <node1 UUID here> replace properties/capabilities='profile:ceph,boot_option:local'
ironic node-update <node2 UUID here> replace properties/capabilities='profile:cinder,boot_option:local'
ironic node-update <node3 UUID here> replace properties/capabilities='profile:compute,boot_option:local'
ironic node-update <node4 UUID here> replace properties/capabilities='profile:controller,boot_option:local'
ironic node-update <node5 UUID here> replace properties/capabilities='profile:swift,boot_option:local'
{% endcodeblock %}

When deploying the OpenStack cloud, we need to tell the RDO manager what flavor to use for each specific role:

{% codeblock lang:sh %}
openstack overcloud deploy \
--templates /usr/share/openstack-tripleo-heat-templates \
--ceph-storage-scale 1 \
--block-storage-scale 1 \
--compute-scale 1 \
--control-scale 1 \
--swift-storage-scale 1 \
--ceph-storage-flavor ceph \
--block-storage-flavor cinder \
--compute-flavor compute \
--control-flavor controller \
--swift-storage-flavor swift
{% endcodeblock %}

And that's all for today. Hope you're enjoying the full control over your OpenStack cloud deployment.
