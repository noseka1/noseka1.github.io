+++
title = "Bootstrapping a Galera Cluster on RHEL7"
date = "2016-01-31"
slug = "2016/01/31/bootstrapping-a-galera-cluster-on-rhel7"
Categories = [ "cloud", "devops" ]
+++

The MariaDB Galera packages provided by the RDO project in their OpenStack repositories don't seem to include a command or script to bootstrap the cluster. Let's look at an alternative way to bring the cluster up.

<!--more-->

RHEL7 comes with the init system `systemd`. Unfortunately, systemd doesn't provide a way to pass command-line arguments to the unit files. Hence, doing something like this won't work:
{{< highlight shell "linenos=table" >}}
[root@rhel1 ~]$ systemctl start mariadb --wsrep_new_cluster
systemctl: unrecognized option '--wsrep_new_cluster'
{{< / highlight >}}

Instead of passing command-line arguments, systemd allows for creating [multiple instances](http://0pointer.de/blog/projects/instances.html) of the same service where each instance can obtain it's own set of environment variables. The Percona XtraDB Cluster includes the standard and the bootstrap service instance definitions in the RPM package `Percona-XtraDB-Cluster-server`. To boostrap the Percona cluster, the first node can be started with the following command:

{{< highlight shell "linenos=table" >}}
[root@percona1 ~]$ systemctl start mysql@bootstrap.service
{{< / highlight >}}

At the moment, this boostrap service definition is missing in the RDO OpenStack packages. Before a similar `mysql@.service` script is available in RDO you can start the MariaDB Galera cluster as follows:

* On the first node, start the MariaDB with the `--wsrep-new-cluster` to create a new cluster:
{{< highlight shell "linenos=table" >}}
[root@rhel1 ~]$ /usr/bin/mysqld_safe --wsrep-new-cluster
{{< / highlight >}}
Let the command run in the foreground.

*  On the remaining cluster nodes start the mariadb service as usual:
{{< highlight shell "linenos=table" >}}
[root@rhel2 ~]$ systemctl start mariadb
{{< / highlight >}}

*  After the cluster has been fully formed, stop the mariadb on the first node by sending it a SIGQUIT (press CTRL + \\ on the console).

*  On the first node, start the mariadb service via systemd:
{{< highlight shell "linenos=table" >}}
[root@rhel1 ~]$ systemctl start mariadb
{{< / highlight >}}

That's it. You can check the status of each of the cluster nodes by running the following command:

{{< highlight shell "linenos=table" >}}
[root@rhel1 ~]$ mysql -e "SHOW GLOBAL STATUS LIKE 'wsrep%';"
{{< / highlight >}}
