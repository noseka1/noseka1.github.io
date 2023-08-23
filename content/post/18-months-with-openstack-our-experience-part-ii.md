+++
title = "18 Months with Openstack, Our Experience, Part II"
date = "2018-03-08"
slug = "2018/03/08/18-months-with-openstack-our-experience-part-ii"
Categories = [ "cloud", "devops" ]
+++

In the [previous post](/blog/2018/02/19/18-months-with-openstack-our-experience-part-i/), we discussed our experience with the deployment of OpenStack. In this article, we're going to share the lessons learned when operating it. It took effort to tame the OpenStack beast and make it work reliably. If you want to know how we accomplished that, read on.

<!--more-->

# Monitoring OpenStack using Icinga

As the old sysadmin saying goes:
> If you don't monitor it, it's not in production.

When looking for a tool to monitor OpenStack, we came across the [Monasca](https://wiki.openstack.org/wiki/Monasca) project. Monasca is a monitoring-as-a-service solution built exclusively for OpenStack. The idea of deploying a system which was from the ground up designed for OpenStack was very appealing. However, after taking a closer look at Monasca we steered away from it. Firstly, Monasca was built around Big Data technologies like [Apache Kafka](https://kafka.apache.org) and [Apache Storm](http://storm.apache.org/) which are a great fit for large-scale deployments. For our rather low-scale use case they seemed to be a bit too heavy. Secondly, around Kilo release it was difficult to predict how much adoption Monasca would find in the community. Instead of Monasca, we eventually decided to go with [Icinga](https://www.icinga.com/) which is a derivate of Nagios, a de facto industry standard between monitoring solutions. I wrote about the monitoring of OpenStack using Icinga in one of my previous [posts](/blog/2015/11/30/monitoring-openstack-cluster-with-icinga/). Setting up Icinga to monitor OpenStack meant to search for Nagios plugins to check various parts of the OpenStack cluster. In addition to the standard set of plugins that comes with the Nagios distribution, we ended up using many plugins that we found on the Internet:

*  [check_mem](https://github.com/justintime/nagios-plugins) Monitor Linux system memory usage.
* [check_diskstat](https://github.com/mclarkson/check_diskstat) Linux disk I/O checks, tps, read, write, avg. request size, avg. queue size and avg. wait time.
* [check_process](https://github.com/nguttman/Nagios-Checks/tree/master/Unix/Check_Process) UNIX process monitoring.
* [check_service](https://github.com/jonschipp/nagios-plugins/blob/master/check_service.sh) Monitors services managed by systemd.
* [check-ceph-dash](https://github.com/Crapworks/check_ceph_dash) Monitors overall Ceph cluster status. Requires [ceph-dash](https://github.com/Crapworks/ceph-dash) to be installed.
* [check_haproxy](https://github.com/noseka1/check_haproxy) Monitors the health of HAProxy backends.
* [haproxy_http_stats](https://github.com/polymorf/check_haproxy) Turns the HAProxy statistics into Nagios performance data.
* [nagios-plugin-check_galera_cluster](https://github.com/noseka1/nagios-plugin-check_galera_cluster) Checks the status of a Galera cluster.
* [check_keepalived_vrrp](https://github.com/alaskacommunications/nagios_check_keepalived) Monitors Keepalived VRRP subsystem.
* [check_memcached](https://github.com/willixix/WL-NagiosPlugins/blob/master/check_memcached.pl) Checks Memcached statistics.
* [check_redis](https://github.com/willixix/WL-NagiosPlugins/blob/master/check_redis.pl) Checks Redis status variables.
* [check_uptime](https://github.com/willixix/WL-NagiosPlugins/blob/master/check_uptime.pl) Tracks system uptime. Great to detect power outages.
* [check_mongodb](https://github.com/mzupan/nagios-plugin-mongodb) Monitor MongoDB servers.
* [monitoring-for-openstack](https://github.com/noseka1/monitoring-for-openstack) Monitor various OpenStack services (Nova, Cinder, Glance, Neutron ...)
* [openstack-nagios-plugins](https://github.com/noseka1/openstack-nagios-plugins) Yet another set of Nagios plugins to monitor OpenStack services.
* [check_mysql_health](https://labs.consol.de/nagios/check_mysql_health/index.html) Monitor health and performance of a MySQL database.
* [nagios-plugins-rabbitmq](https://github.com/nagios-plugins-rabbitmq/nagios-plugins-rabbitmq) Set of nagios checks useful for monitoring a RabbitMQ cluster.

Besides monitoring the availability of OpenStack services, monitoring the performance of hypervisor hosts was another important point to ensure smooth operations and user happiness. You've heard about the "noisy neighbor" problem before, haven't you? Time to time it happened to us that users unknowingly started a workload that would hog the CPU, disk or network I/O of the hypervisor to the extent that other virtual machines running on the same hypervisor were slowed down. In such a situation it was important that Icinga would alert the OpenStack operator that would resolve the problem before the affected users would notice.

# Ceilometer metrics and events

[Ceilometer](https://docs.openstack.org/ceilometer) is an OpenStack data collection service that collects telemetry data across all OpenStack components. This telemetry data provides useful insights into the OpenStack operation and I would strongly recommend to you to deploy Ceilometer and configure it to store the telemetry data in the backend of your choice. The data provided by the Ceilometer service can be divided into two categories: measurements and events.

## Ceilometer measurements

[Ceilometer measurements](https://docs.openstack.org/ceilometer/pike/admin/telemetry-measurements.html) are performance data. Ceilometer collects performance samples by polling the OpenStack infrastructure elements in regular intervals. For instance, Ceilometer measures CPU, memory, disk and network usage of individual virtual machines hosted on OpenStack, it can measure the performance of hypervisor hosts and much more. It's up to you to choose which data interests you. We ended up collecting merely the performance data of individual virtual machines. Monitoring of hypervisor hosts was better left to Icinga.

There are many options of how to process the performance data generated by Ceilometer. We configured Ceilometer to send the performance samples in the [MessagePack](https://msgpack.org) format over UDP protocol to [Logstash](https://www.elastic.co/products/logstash). Logstash in turn forwards the data to the [InfluxDB](https://www.influxdata.com/) storage. [Grafana](https://grafana.com/) is used to view and graph the performance data stored in InfluxDB. We spent quite a bit of time configuring Logstash to enrich the data coming from Ceilometer to be able to create a beautiful Grafana dashboard that would display the performance graphs of individual virtual machines hosted on OpenStack. Our OpenStack users would be able to look up their virtual machine in the Grafana dashboard based on the OpenStack project and the display name of the instance. After investing all the effort to create the dashboard the practice showed that nobody really cared about the performance monitoring of most of the virtual machines. And if we deployed a virtual machine we wanted to monitor, we preferred to just install the Icinga monitoring agent on it.

## Ceilometer events

[Ceilometer events](https://docs.openstack.org/ceilometer/pike/admin/telemetry-events.html) represent any action made in the OpenStack system, for example: successful user authentication, creating a virtual machine, terminating a virtual machine, creating a volume, attaching a volume to a virtual machine and many others. Ceilometer generates events based on the notifications that are published by the OpenStack services on the message bus. For instance, the list of notifications published by the Nova components can be found [here](https://docs.openstack.org/nova/latest/reference/notifications.html). In the past, I wrote an [article](/blog/2015/05/25/openstack-nova-notifications-subscriber/) describing how to subscribe to the Nova notifications on the RabbitMQ message bus.

In our OpenStack deployment, we configured Ceilometer to send events to [Elasticsearch](https://www.elastic.co/). [Kibana](https://www.elastic.co/products/kibana) is used to view and search for the collected events. Having all the OpenStack events collected and archived at one place turned out to be really helpful. One day, a co-worker of mine brought up a complaint that somebody deleted his virtual machine. Deleting virtual machines of other people, who would dare that? Instead of asking around and disturbing people on the team, we were able to look up all the events pertaining to the lost virtual machine. We found out that the termination event ran on behalf of the Jenkins user. It didn't take much longer to identify the Jenkins job which deleted the virtual machine. Finally, it turned out that the co-worker that complained about the loss of "his" virtual machine was actually handed over the virtual machine only temporarily and that the machine was deleted and recreated every night by Jenkins. And I told to myself, what an *automated* world!

# Log collection using ELK

In addition to storing Ceilometer events in Elasticsearch, we also configured [Filebeat](https://www.elastic.co/products/beats/filebeat) to collect OpenStack logs and Linux system logs from all the OpenStack nodes and store them in Elasticsearch. They will come handy in the future when explaining other "mysteries" happening in our OpenStack cluster.

# Tempest and Rally

When you deploy an OpenStack cluster, how do you verify that your cluster functions correctly? Icinga checks cover a very small subset of the OpenStack functionality. To accomplish a thorough verification of the OpenStack cluster, we started using the [Tempest](https://docs.openstack.org/tempest/latest/) project. Tempest is a battery of integration tests that are used to verify OpenStack's functionality and it is a part of the continuous integration pipeline of the OpenStack project. Tempest tests send requests to the OpenStack APIs and verify the responses. As the goal of the Tempest project is to verify the integration of OpenStack components during development,  the included integration tests were a bit too low-level for our use case of merely verifying that the OpenStack cluster functioned properly. However, there were no better tools available at the time and it did the trick for us.

After a while of using Tempest, we discovered yet another project called [Rally](https://docs.openstack.org/developer/rally/). Rally is a benchmarking tool that is used to measure OpenStack's performance and to identify performance bottlenecks. Rally builds on top of Tempest and it comes with a set of predefined scenarios that are executed against the OpenStack cluster. Example scenarios are: boot and delete server, boot server from volume, create a subnet, create and attach volume, create and delete a Heat stack, and many more. The available scenarios were just right to verify our cloud! On top of that, Rally generates beautiful reports with the overview of executed tasks and their duration. We ended up creating a cron job that schedules the Rally tests to run every two hours. The test results are monitored using Icinga which in the case of test failure sends an alert to the operator.

Because our OpenStack cluster was constantly exercised by the Rally tests, we were able to quickly spot resources that OpenStack didn't clean up properly and that were piling up. We have seen diverse OpenStack database tables growing infinitely.  We have experienced Neutron leaving processes running on the controller nodes, leaving empty network namespaces behind or filling up the `/var/log/neutron` directory with files. Remember that we experienced these issues while using the Mitaka release of OpenStack. I'm sure that things improved since then. To address the resource leaks, we wrote custom clean-up scripts. I'm publishing them for you to use at your own risk. You can find them on [GitHub](https://github.com/noseka1/openstack-periodic-cleanup).

# Tracking cloud resource usage

In OpenStack, we were missing some kind of reporting on resource usage. In our organization, each development team has its dedicated project in OpenStack. It is important to us to understand, how much of the cloud resources each team is consuming, i.e. how many CPU cores, memory, and volumes. In addition to per project usage, we also monitor the total resource usage across the entire cluster. In the case, that the total usage is reaching the total capacity available we can organize additional hardware ahead of time.

Around OpenStack Mitaka release, we didn't find any tool that would generate the usage reports. However, OpenStack's MariaDB database contains all the input data required to create such reports. It was rather straightforward to create a set of SQL scripts to generate the reports directly out of the OpenStack's database. We run these scripts periodically using Icinga, so that we can see the report output on our monitoring dashboard. If you are interested, you can find our OpenStack usage report scripts on [GitHub](https://github.com/noseka1/openstack-cloud-report).

# Further notes

I'd like to describe several further observations that we made while operating OpenStack. Once again, our experience pertains to the Mitaka release of OpenStack only. Many of the issues we stumbled upon might have been resolved in the newer releases of OpenStack.

*  In order to get OpenStack working smoothly, you should expect to use some amount of duct tape and bubble gum. As OpenStack was implemented in Python, patching OpenStack is relatively easy. Many times I was able to find fixes for our issues on the project development branches and needed just to port them to our OpenStack version.
* OpenStack is deployed on many nodes. It was useful for us to write Ansible scripts to automate the restart of the RabbitMQ cluster and to automate the restart of all OpenStack services on all nodes (aka restart the world). Due to the issues with the OpenStack TripleO installer in the Mitaka release, we are still forced to restart the world after adding a compute node.
* Switching to Keystone Fernet tokens considerably reduced the load on the MariaDB database. We enabled Fernet tokens even in the Mitaka release of OpenStack.
* RabbitMQ, Cinder Backup and several other services require higher amount of open file descriptors. For instance, RabbitMQ [recommends](https://www.rabbitmq.com/production-checklist.html) to allow at least 50K of open file descriptors. Insufficient amount of file descriptors caused our RabbitMQ to crash. As RabbitMQ is the communication backbone of OpenStack, you can imagine how much fun it caused.
* Our OpenStack networking is set up to use Neutron's OpenVSwitch driver and VLANs. In the default configuration, it happened to us that the multicast traffic sent by a single virtual machine flooded the entire network and caused OpenVSwitch to begin dropping packets. We didn't do any further research on the multicast on OpenStack topic so far, just avoided sending multicast altogether.
* Kudos goes to the [Ceph](https://ceph.com/) storage. We are running the old Ceph v0.94 Hammer which was able to survive emergency situations like lost storage node and running out of space condition without any problems.

# Conclusion

In this blog post, we shared some of our experiences with operating OpenStack. We described the monitoring using Icinga, collecting Ceilometer metrics and events, collecting system logs using the ELK stack, verifying the OpenStack functionality with Rally and generating cloud resource usage reports.

OpenStack is not the easiest software to run, however, if you do your homework you will succeed. At the present time, OpenStack just works for us and brings a lot of value to the teams in our company.

If you'd like to share your experience with operating OpenStack, I would love to hear from you. Please, feel free to use the comment section below.
