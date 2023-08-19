+++
title = "Test Driving Openstack Manila"
date = "2016-05-22"
slug = "2016/05/22/test-driving-openstack-manila"
Categories = []
+++

Do you need to provision an NFS share for your Hadoop cluster? And what about creating a CIFS share to make your files accesible to the Windows clients? Manila is a provisioning and management service for shared file systems within OpenStack. Let's test-drive it in this blogpost.

<!--more-->

In this introductory article, we're going to allocate a volume in Cinder and provide that volume as an NFS share to our Nova instances. For this, I'm using the OpenStack Mitaka installed via TripleO on RHEL7. The Manila version included in the Mitaka release is version 2.0.

After installing Manila, the following Manila services are running on the controller nodes:

* *openstack-manila-api* exposes REST APIs that the Manila client talks to.
* *openstack-manila-scheduler* makes provisioning decisions when creating a new share.
* *openstack-manila-share* comes with a host of drivers to talk to the storage systems.

## Configuring the generic share driver

In order for Manila to allocate shares on Cinder volumes, we'll have to configure Manila to use the *generic* share driver. For that we'll add a new Manila backend `generic_backend` into `/etc/manila/manila.conf`:

{% codeblock lang:ini %}
[DEFAULT]
enabled_share_backends = generic_backend
default_share_type = generic
[generic_backend]
share_driver = manila.share.drivers.generic.GenericShareDriver
share_backend_name = generic_backend
service_instance_name_template = manila_service_instance_%s
service_image_name = manila-service-image-master
driver_handles_share_servers = True
service_instance_flavor_id = 103
connect_share_server_to_tenant_network = True
service_instance_user = manila
path_to_public_key = /etc/manila/id_rsa.pub
path_to_private_key = /etc/manila/id_rsa
manila_service_keypair_name = manila-service
{% endcodeblock %}

Before explaining the configuration settings, I'll briefly describe how the *generic* driver actually works. Behind the scenes, the generic driver creates a so called *service instance*. The service instance is a Nova instance owned by the Manila service. It's not even visible to the tenant users. Manila allocates a Cinder volume and asks Nova to attach that volume to the service instance. Afterwards, Manila connects to the service instance using SSH in order to create the filesytem on the attached Cinder volume and mount it and export that as a NFS/CIFS share to the tenant instances.

The service instance can be created by the OpenStack administrator or we can configure Manila to create the service instance by itself (option `driver_handles_share_servers = True`).

The service instance will be created from the image that we have to upload into Glance beforehand. I downloaded an existing Manila service image from [here](http://tarballs.openstack.org/manila-image-elements/images/manila-service-image-master.qcow2). This image is based on Ubuntu 14.04.4 LTS and includes the `manila` user account and the NFS and Samba server software packages. I uploaded this image into Glance under the name `manila-service-image-master`.

Next I've chosen the size of the machine used for the service instance with `service_instance_flavor_id = 103`.

The service instance is connected to two networks. The first network is called a *service network* and is created by Manila before booting up the service instance. Manila uses this network for the SSH access to the service instance. The second network is a *share network*. The NFS server managed by Manila is accessible on this network. In our case, because we have configured `connect_share_server_to_tenant_network = True`, the share network will directly map to one of our tenant networks.

Finally, we have to generate a public/private key pair and tell Manila about it using the options `path_to_public_key` and `path_to_private_key`. Manila will upload this keypair into Nova under the name `manila-service`. When creating the service instance, Nova injects the public key into the instance and so allows Manila the SSH access.

In order to make our generic backend available to the Manila users, we're going to define a `generic` share type next.

## Defining a share type

The *share type* has a similar purpose as the *volume type* in Cinder. It defines the backend used for the share creation. If there are multiple share backends available, an OpenStack administrator can define a separate share type for each of them. When creating a new share, the user can choose which share type to allocate the storage from.

To create a `generic` share type that maps to our `generic` backend you can run the following commands as an OpenStack administrator:

{% codeblock lang:sh %}
manila type-create generic True
manila type-key generic set share_backend_name=generic_backend
{% endcodeblock %}

## Creating a share and mounting it

Finally, we're done with all the configuration and can start enjoying our share service. All the following commands are run as an ordinary tenant user.

At first, we'd like to create a share network and map it to one of our tenant networks:

{% codeblock lang:sh %}
manila share-network-create --neutron-net-id 4f179a8c-7068-4f0b-9be4-9cb11451b401 --neutron-subnet-id c7d753b0-039b-4f8c-9e0f-012651ff4ada --name management
{% endcodeblock %}

Now we can create our first NFS share called `myshare` with the size 1 GB:

{% codeblock lang:sh %}
manila create --name myshare --share-network management NFS 1
{% endcodeblock %}

Creating the first share on a given tenant network takes longer as Manila has to spin up a new service instance in the background.

Eventually, the status of the share turns into `available` which means that the share is ready. The `manila show myshare` command will display the location from where we can mount the share. In our case, it is `10.13.243.173:/shares/share-b87367aa-3ef3-4282-a6b5-e45cab991b6c`. Before we can mount the share we have to allow access to it by modifying the access list:

{% codeblock lang:sh %}
manila access-allow --access_level rw myshare ip 10.13.244.12
{% endcodeblock %}

The above command provides an instance having the IP address 10.13.244.12 with a read-write access to the share. Note that the IP addresses 10.13.243.173 and 10.13.244.12 belong to the same network. Finally, we can SSH into the instance and mount the share with:

{% codeblock lang:sh %}
sudo mount -t nfs 10.13.243.173:/shares/share-b87367aa-3ef3-4282-a6b5-e45cab991b6c /mnt
{% endcodeblock %}
