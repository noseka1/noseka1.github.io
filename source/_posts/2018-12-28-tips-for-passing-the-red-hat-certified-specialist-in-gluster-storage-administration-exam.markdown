---
layout: post
title: "Tips for Passing the Red Hat Certified Specialist in Gluster Storage Administration Exam"
date: 2018-12-28 11:24:40 -0800
comments: true
categories: devops
---

Recently I passed the [Red Hat Certified Specialist in Gluster Storage Administration](https://www.redhat.com/en/services/training/ex236-red-hat-certified-specialist-in-gluster-storage-administration-exam) exam. In this blog post, I would like to share some of my experience and exam tips with you.

<!-- more -->

There are two major open-source storage technologies available today: [Ceph](https://ceph.com/) and [Gluster](https://www.gluster.org/). For a couple of years, I used Ceph as a provider of a block and object storage for an OpenStack cluster with great success. I was curious to learn about what Gluster has to offer. To explore Gluster, I leveraged the training materials and labs included in my [Red Hat Learning Subscription](https://www.redhat.com/en/services/training/learning-subscription). Also, to give myself a particular goal to achieve, I signed up for the Red Hat Certified Specialist in Gluster Storage Administration exam. The main topics included on the exam are creating different types of Gluster volumes, volume snapshotting, exporting Gluster volumes via highly-available NFS and Samba, asynchronous replication (geo-replication), tiered volumes and transport security.

As I was working through the learning materials for the first time, I used the provided virtual machines to solve the lab exercises. I liked that after I completed the exercise I could run a grading script that would check my configuration and to see if anything was still missing. However, after I realized that setting up a Gluster cluster from scratch doesn't take much effort, I decided to spin up my own practice environment. I created three virtual machines (2 GB RAM, 4 vCPUs) on my laptop using libvirt/QEMU/KVM and installed CentOS 7 and Gluster 4.1. The software on the real exam is RHEL 7 and Gluster 3, however, it didn't make much difference to me. Switching to my custom practice environment removed the perceived latency while typing and allowed me to easily copy and paste between console windows. In addition to that, I could keep my virtual machines running all the time. Virtual machines in the lab environment provided by Red Hat shutdown automatically after two hours unless you bump up the timer.

If you are preparing for the Gluster certification exam, here are several things that are good to know:

* The complete [Red Hat Gluster Storage documentation](https://access.redhat.com/documentation/en-us/red_hat_gluster_storage) is available to you during the exam in a searchable PDF format. The Red Hat Gluster Storage Administration Guide turned out to be particularly useful. I recommend reading through the guide as a part of the preparation for the exam. You will learn where the important configuration parts are located in the guide and will be able to quickly refer to them during the exam.
* Preparing underlying LVM volumes for Gluster bricks takes quite a bit of time which you won't have plenty of in the exam. Here is my advice: at the beginning of the exam, walk through the exam tasks and create all the LVM volumes that you will need up front.
* Command `gluster volume set help` lists all the Gluster volume parameters along with their short descriptions. If you cannot remember the exact parameter name just grep through the output of this command.
* If you cannot remember the Gluster mount options, you can find them by typing `man mount.glusterfs`
* Provisioning of LVM thin volumes is thoroughly documented in `man lvmthin`
* To list all services that you can enable on the firewall, type `firewall-cmd --get-services`
* To find out whether SELinux denied access, issue the command  `grep denied /var/log/audit/audit.log`. SELinux can provide a hint on how to possibly solve the access issue, use the command `sealert -a /var/log/audit/audit.log`
* To list SELinux types related to Gluster, you can type `strings /sys/fs/selinux/policy | grep gluster`
* And last but not least, you can list volumes exported by the NFS server with `showmount -e <HOST>`. To show volumes available on a Samba server, issue the command `smbclient -L <HOST> -U %`

And how did I do on the exam? Well, I would not say that the exam was difficult, however, I also didn't practice much before the exam. Unfortunately, I ran out of time before I could complete the last exam task. Overall, I achieved 236 points out of 300. As the passing score was 210, I did pass!

Are you preparing for the Red Hat Gluster Storage exam? I would love to hear from you! Please, leave your comments or questions in the comment section below.

{% img /images/posts/rhcs_gluster_storage_administration_badge.png %}
