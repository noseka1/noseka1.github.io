+++
title = "Booting Amazon Linux 2 on Openstack"
date = "2018-04-21"
slug = "2018/04/21/booting-amazon-linux-2-on-openstack"
Categories = [ "cloud" ]
+++

Amazon Linux 2 runs on OpenStack perfectly fine. There is only one glitch that you should be aware of. Amazon Linux 2 won't accept metadata and user data provided by OpenStack on boot. That means that you won't be able to SSH into the instance after it comes up. In this brief tutorial, we are going to modify the Amazon Linux 2 image to fix this problem.

<!--more-->

You can download Amazon Linux 2 images from https://cdn.amazonlinux.com/os-images/latest/. An image suitable for OpenStack is located in the `kvm` subdirectory. I downloaded the `amzn2-kvm-2017.12.0.20180330-x86_64.xfs.gpt.qcow2` version of the image. By the time you are reading this tutorial, a newer version of the image may be available.

In the rest of this article, I'm going to use my machine that is running RHEL7 to modify the Amazon Linux 2 image. First, let' download the image:

{{< highlight shell "linenos=table" >}}
$ wget https://cdn.amazonlinux.com/os-images/2017.12.0.20180330/kvm/amzn2-kvm-2017.12.0.20180330-x86_64.xfs.gpt.qcow2
{{< / highlight >}}

Next, let's install the `qemu-img` utility useful for manipulating qcow2 images:

{{< highlight shell "linenos=table" >}}
$ sudo yum install qemu-img
{{< / highlight >}}

Now we can convert the Amazon Linux 2 image from the qcow2 format to the raw format:

{{< highlight shell "linenos=table" >}}
$ qemu-img convert -f qcow2 -O raw amzn2-kvm-2017.12.0.20180330-x86_64.xfs.gpt.qcow2 amzn2-kvm.raw
{{< / highlight >}}

The previous command creates a file `amzn2-kvm.raw` in the current working directory. This file is a binary image of the virtual machine disk. We can explore it using the `fdisk` command:

{{< highlight shell "linenos=table" >}}
$ fdisk -l amzn2-kvm.raw
Disk amzn2-kvm.raw: 26.8 GB, 26843545600 bytes, 52428800 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk label type: gpt
Disk identifier: 88B4CB3B-A2F1-4C9C-82DC-F18B0F440F56


#         Start          End    Size  Type            Name
 1         4096     52428766     25G  Linux filesyste Linux
128         2048         4095      1M  BIOS boot       BIOS Boot Partition
{{< / highlight >}}

The output of the `fdisk` command shows that the disk contains two partitions. The size of the first partition is 25 GB and it holds a Linux filesystem. On the disk, the Linux filesystem starts at the sector number 4096. Given that the size of the sector is 512 bytes, we can tell that the Linux filesystem starts at offset 2097152 (4096 * 512) bytes from the start of the disk image. Knowing the offset of the Linux filesystem, let's loop mount the Linux filesystem under `/mnt`:

{{< highlight shell "linenos=table" >}}
$ sudo mount -o loop,offset=2097152 amzn2-kvm.raw /mnt
{{< / highlight >}}

If everything went well, we can now take a look at the cloud-init configuration of the Amazon Linux 2 image:

{{< highlight shell "linenos=table" >}}
$ sudo vi /mnt/etc/cloud/cloud.cfg
{{< / highlight >}}

In the cloud-init configuration file, you can find the data source list set as follows:

{{< highlight ini "linenos=table" >}}
datasource_list: [ NoCloud, AltCloud, ConfigDrive, OVF, None ]
{{< / highlight >}}

Sadly, none of the listed data sources is available on OpenStack. OpenStack supports its own data source called `OpenStack`. Alternatively, OpenStack is compatible with the AWS data source called `Ec2`. This compatibility ensures that virtual machine images designed for EC2 will work properly on OpenStack. I would expect that the `Ec2` data source would be included in the data source list of the Amazon Linux 2 image but it is not. Let's add the `OpenStack` data source to the list:

{{< highlight ini "linenos=table" >}}
datasource_list: [ OpenStack, NoCloud, AltCloud, ConfigDrive, OVF, None ]
{{< / highlight >}}

I put the `OpenStack` data source at the beginning of the list. You can choose to add it anywhere else. Just make sure that the `None` data source remains as the last one on the list. `None` is a fallback datasource used when no other datasources can be selected and it provides empty metadata and empty user data.

After you saved your changes, you can unmount the Linux filesystem:

{{< highlight shell "linenos=table" >}}
$ sudo umount /mnt
{{< / highlight >}}

And convert the modified image back to the qcow2 format:

{{< highlight shell "linenos=table" >}}
$ qemu-img convert -f raw -O qcow2 amzn2-kvm.raw amzn2-kvm.qcow2
{{< / highlight >}}

Now you can upload the modified image into the OpenStack image repository:

{{< highlight shell "linenos=table" >}}
$ openstack image create --min-disk 25 --min-ram 512 --container-format bare --disk-format qcow2 --file amzn2-kvm.qcow2 amzn2-kvm
{{< / highlight >}}

After the image upload into OpenStack has completed, you can create a test virtual machine off of this image:

{{< highlight shell "linenos=table" >}}
$ openstack server create --image amzn2-kvm --flavor m1.medium --key-name <key-name> --nic net-id=<net-id> amzn-test
{{< / highlight >}}

Note that in the above command, you'll have to replace the `<key-name>` and `<net-id>` placeholders with the name of your key pair and the name of the network you want your instance to be attached to. After the virtual machine has booted up, you should be able to connect to it using SSH. Note that the default user enabled on the Amazon Linux 2 image is `ec2-user`:

{{< highlight shell "linenos=table" >}}
$ ssh ec2-user@amzn-test
Last login: Sun Apr 22 05:08:45 2018 from ales.dev.ussd.verimatrix.com

       __|  __|_  )
       _|  (     /   Amazon Linux 2 AMI
      ___|\___|___|

https://aws.amazon.com/amazon-linux-2/
No packages needed for security; 8 packages available
Run "sudo yum update" to apply all updates.
{{< / highlight >}}

This is the end of the tutorial. You have a working Amazon Linux 2 image on OpenStack, congratulations! If you have any comments or questions, let me know in the comment section below.
