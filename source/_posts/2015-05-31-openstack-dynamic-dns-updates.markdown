---
layout: post
title: "OpenStack Dynamic DNS Updates"
date: 2015-05-31 22:49:39 -0700
comments: true
categories: devops cloud
---

Today's user story is: As a private cloud user I'd like to have virtual machines registered with internal DNS. Let's look at how a software practitioner solves this problem in a truly agile way.

<!-- more -->

The OpenStack [Designate](https://wiki.openstack.org/wiki/Designate "Designate") project implements DNSaaS. After trying out Designate, I realized that for simple DNS updates DNSaaS is a little too involved.

In my previous [article](/blog/2015/05/25/openstack-nova-notifications-subscriber "OpenStack Nova Notifications Subscriber") I described how to monitor Nova events on RabbitMQ message bus. Two events of interest are `compute.instance.create.end` and `compute.instance.delete.start` that are sent by Nova on instance creation and instance deletion. Both events carry enough information for us to create a simple script that extracts the hostname and IP addresses of the instance from the events and updates the internal DNS using the `nsupdate` command.

You can find the DNS updates implementantion including the systemd startup script at GitHub: [openstack-dns-updater](https://github.com/noseka1/openstack-dns-updater "openstack-dns-updater").
