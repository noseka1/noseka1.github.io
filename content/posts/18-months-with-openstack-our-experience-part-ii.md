+++
title = "18 Months With Openstack Our Experience Part Ii"
date = "2018-03-08"
slug = "2018/03/08/18-months-with-openstack-our-experience-part-ii"
Categories = []
+++

In the [previous post](/blog/2018/02/19/18-months-with-openstack-our-experience-part-i/), we discussed our experience with the deployment of OpenStack. In this article, we're going to share the lessons learned when operating it. It took effort to tame the OpenStack beast and make it work reliably. If you want to know how we accomplished that, read on.

<!-- more -->

## Monitoring OpenStack using Icinga

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

## Ceilometer metrics and events

Categories = [