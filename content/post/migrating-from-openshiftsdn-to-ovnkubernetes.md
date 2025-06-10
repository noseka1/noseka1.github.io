+++
title = "Migrating from OpenShiftSDN to OVNKubernetes"
date = "2025-06-08"
Categories = [ "devops", "cloud" ]
+++

The OpenShiftSDN CNI (Container Network Interface) plug-in has been deprecated since OpenShift Container Platform 4.14 and was removed in version 4.17. If your current clusters utilize OpenShiftSDN, you won’t be able to upgrade them to version 4.17 without first migrating to OVNKubernetes. If you are considering migrating from OpenShiftSDN to OVNKubernetes, this blog post is intended for you.

<!--more-->

This blog entry is based on the OpenShift documentation section [Migrating from the OpenShift SDN network plugin](https://docs.redhat.com/en/documentation/openshift_container_platform/4.16/html/networking/ovn-kubernetes-network-plugin#migrate-from-openshift-sdn). This documentation section describes two migration methods for transitioning from OpenShiftSDN to OVNKubernetes: the offline migration method and the limited live migration method.

The offline migration method leads to interruptions in pod network connectivity and should only be employed when the limited live migration method is not applicable. Certain less frequent situations in which the limited live migration method cannot be utilized are mentioned in the aforementioned documentation.

The limited live migration method is the favored method for migration since it doesn’t disrupt the network connectivity of the pods. Throughout the live migration process, the connection of the pods to external networks, as well as the connectivity between the pods within the cluster, will function normally. This is accomplished by migrating the CNI of each node individually and utilizing the hybrid overlay feature of OVNKubernetes to maintain a continuous connection between the networks of the two CNI network plugins, ensuring that pods in each network can still communicate with pods in the other network.

This is particularly important if you wish to avoid the risk of workloads misbehaving after a loss of network connectivity or failing to recover once the network connectivity is restored. During the limited live migration, the pods will maintain their access to the network and connectivity with other pods within the cluster. They will merely be restarted on different nodes during the node drain process.

Note that during the network live migration, the cluster nodes are drained and rebooted two times by the machine config operator. As a result, the migration process takes approximately twice as long as a standard cluster upgrade. The initial reboot is required to reduce the MTU of network interfaces. The MTU is decreased by 50 bytes to accommodate the increased overhead associated with the OVNKubernetes overlay when compared to the OpenShiftSDN. The subsequent reboot is required to apply the modifications made to the node start-up scripts, thereby transforming it into an OVNKubernetes node.

The limited live migration has been available since OpenShift version 4.16. If you are currently operating on an earlier version of OpenShift, it is advisable to upgrade to OCP 4.16 first in order to enable the use of the limited live migration method. Ultimately, an upgrade to OCP 4.16 will be necessary regardless.

The rest of this blog post will concentrate on the process of migrating the cluster network through the use of limited live migration.

# Pre-flight checks

The OpenShift documentation includes a [set of considerations](https://docs.redhat.com/en/documentation/openshift_container_platform/4.16/html/networking/ovn-kubernetes-network-plugin#considerations-live-migrating-ovn-kubernetes-network-provider_migrate-from-openshift-sdn) that you ought to examine prior to utilizing the limited live migration. In view of these considerations, this section will perform cluster checks that focus on the common scenarios. The knowledgebase article [7057169 Limited Live Migration from OpenShift SDN to OVN-Kubernetes](https://access.redhat.com/solutions/7057169), contains a more comprehensive list of checks.

## Checking network isolation mode of OpenShiftSDN

OpenShiftSDN supports three isolation modes: network policy, multitenant, and subnet mode. The default network isolation mode in OpenShift 4 is network policy. To check the configured isolation mode, type the command:


{{< highlight shell "linenos=table" >}}
$ oc get network.operator.openshift.io cluster -o yaml
{{< / highlight >}}

{{< highlight yaml "linenos=table" >}}
apiVersion: operator.openshift.io/v1
kind: Network
metadata:
...
  name: cluster
...
spec:
  clusterNetwork:
  - cidr: 10.128.0.0/14
    hostPrefix: 23
  defaultNetwork:
    openshiftSDNConfig:
      enableUnidling: true
      mode: NetworkPolicy
      mtu: 8951
      vxlanPort: 4789
    type: OpenShiftSDN
...
{{< / highlight >}}

The example above indicates that the isolation mode is set to `NetworkPolicy`. If the mode setting is not present in the output, the default network policy mode is applied. The limited live migration doesn’t support clusters with OpenShiftSDN multitenant mode enabled. If your output indicates that the mode is set to `Multitenant`, then live migration cannot be performed, and you will need to resort to offline migration instead.


## Checking IP subnet ranges

By default, OVN-Kubernetes allocates the following ranges of IP addresses:



* `100.64.0.0/16`. This IP address range is used for the `internalJoinSubnet` parameter of OVN-Kubernetes.
* `100.88.0.0/16`. This IP address range is used for the `internalTransSwitchSubnet` parameter of OVN-Kubernetes.

If these IP addresses have been used by OpenShiftSDN or any external network requiring communication with this cluster, you must patch them to use a different IP address range prior to commencing the limited live migration. Refer to the OpenShift documentation section [Patching OVN-Kubernetes address ranges](https://docs.openshift.com/container-platform/4.16/networking/ovn_kubernetes_network_provider/migrate-from-openshift-sdn.html#patching-ovnk-address-ranges_migrate-from-openshift-sdn).


## Checking cluster MTU value

Issue the following command to check the MTU setting:


{{< highlight shell "linenos=table" >}}
$ oc get network.operator.openshift.io cluster -o yaml
{{< / highlight >}}

{{< highlight yaml "linenos=table" >}}
apiVersion: operator.openshift.io/v1
kind: Network
metadata:
  creationTimestamp: "2023-03-24T14:34:45Z"
  generation: 9271
  name: cluster
  resourceVersion: "2150111163"
  uid: 00c83690-d7f6-460e-b27c-c178d8980d5d
spec:
  clusterNetwork:
  - cidr: 10.84.0.0/14
    hostPrefix: 22
  defaultNetwork:
    type: OpenShiftSDN
  disableNetworkDiagnostics: false
  logLevel: Normal
  managementState: Managed
  observedConfig: null
  operatorLogLevel: Normal
  serviceNetwork:
  - 10.88.0.0/16
  unsupportedConfigOverrides: null
{{< / highlight >}}

The MTU can be customized by setting the `spec.defaultNetwork.openshiftSDNConfig.mtu` field. In the above configuration example, this field is not included which means that we are using the default MTU and not a custom MTU. For the default MTU, no further action is required. To configure a custom MTU for OVNKubernetes, refer to the OpenShift [documentation](https://docs.openshift.com/container-platform/4.16/networking/ovn_kubernetes_network_provider/migrate-from-openshift-sdn.html#nw-ovn-kubernetes-migration_migrate-from-openshift-sdn).

Issue the following command to verify the effective MTU value:

{{< highlight shell "linenos=table" >}}
$ oc get network.config.openshift.io cluster -o yaml
{{< / highlight >}}

{{< highlight yaml "linenos=table" >}}
apiVersion: config.openshift.io/v1
kind: Network
metadata:
  creationTimestamp: "2023-03-24T14:22:15Z"
  generation: 2
  name: cluster
  resourceVersion: "4395"
  uid: af0a54a3-a941-4193-88c7-37f575fa6b26
spec:
  clusterNetwork:
  - cidr: 10.84.0.0/14
    hostPrefix: 22
  externalIP:
    policy: {}
  networkType: OpenShiftSDN
  serviceNetwork:
  - 10.88.0.0/16
status:
  clusterNetwork:
  - cidr: 10.84.0.0/14
    hostPrefix: 22
  clusterNetworkMTU: 8950
  networkType: OpenShiftSDN
  serviceNetwork:
  - 10.88.0.0/16
{{< / highlight >}}

In the output presented above, the MTU value for the cluster is `8950` (this can be found in the `status.clusterNetworkMTU` field).

## Checking egress IP addresses

During the migration, existing egress IP addresses will be temporarily disabled and the [OpenShiftSDN egress IP](https://docs.redhat.com/en/documentation/openshift_container_platform/4.16/html/networking/openshift-sdn-network-plugin#assigning-egress-ips) configuration will be automatically converted to the respective [OVNKubernetes egress IP](https://docs.redhat.com/en/documentation/openshift_container_platform/4.16/html/networking/ovn-kubernetes-network-plugin#configuring-egress-ips-ovn) configuration. You can verify the resulting configuration after the migration is complete.

Check if there are any egress IPs configured on the cluster:

{{< highlight shell "linenos=table" >}}
$ oc get netnamespace -A | awk '$3 != ""'
{{< / highlight >}}

If the output is empty like this then there are no egress IPs to convert:

{{< highlight shell "linenos=table" >}}
NAME                                               NETID      EGRESS IPS
{{< / highlight >}}

## Checking egress firewall

The migration process automatically converts the [OpenShiftSDN egress firewall](https://docs.redhat.com/en/documentation/openshift_container_platform/4.16/html/networking/openshift-sdn-network-plugin#configuring-egress-firewall) configuration to the [OVNKubernetes egress firewall](https://docs.redhat.com/en/documentation/openshift_container_platform/4.16/html/networking/network-security#egress-firewall) configuration, i.e. a new `EgressFirewall` resource will be created for each `EgressNetworkPolicy` resource in the cluster. The firewall remains functional throughout the whole migration. You can review the resulting OVNKubernetes egress firewall configuration after the migration is complete.

Check if any `EgressNetworkPolicy` resources exist in the cluster:

{{< highlight shell "linenos=table" >}}
$ oc get egressnetworkpolicies.network.openshift.io -A
{{< / highlight >}}

If the output of the above command is empty like this then there are no egress firewall rules to convert:

{{< highlight shell "linenos=table" >}}
No resources found
{{< / highlight >}}

## Checking multicast namespaces

During the migration, multicast will be temporarily disabled. The migration process will automatically convert the [OpenShiftSDN multicast](https://docs.redhat.com/en/documentation/openshift_container_platform/4.16/html/networking/openshift-sdn-network-plugin#enabling-multicast) configuration to the [OVNKubernetes multicast](https://docs.redhat.com/en/documentation/openshift_container_platform/4.16/html/networking/ovn-kubernetes-network-plugin#nw-about-multicast_ovn-kubernetes-enabling-multicast) configuration by annotating any multicast-enabled namespace with `k8s.ovn.org/multicast-enabled=true`.

To locate namespaces with OpenShiftSDN multicast enabled, enter the following command:

{{< highlight shell "linenos=table" >}}
$ oc get netnamespace -o json | jq -r '.items[] | select(.metadata.annotations."netnamespace.network.openshift.io/multicast-enabled" == "true") | .metadata.name'
{{< / highlight >}}

If the output of the above command is empty, then there are no multicast-enabled namespaces in the cluster and nothing to worry about for the migration.

## Checking egress router pods

Check if there are any egress router pods on the cluster:

{{< highlight shell "linenos=table" >}}
$ oc get pods --all-namespaces -o json | jq '.items[] | select(.metadata.annotations."pod.network.openshift.io/assign-macvlan" == "true") | {name: .metadata.name, namespace: .metadata.namespace}'
{{< / highlight >}}

If there are any egress router pods on the cluster, they must be removed prior to initiating the limited live migration process; otherwise, the process will be blocked. During the migration, the egress router pods won’t be available. You will have to re-create the [OpenShiftSDN egress router](https://docs.redhat.com/en/documentation/openshift_container_platform/4.16/html/networking/openshift-sdn-network-plugin#using-an-egress-router) pods in OVNKubernetes after the migration is complete. Note that the [egress router pods in OVNKubernetes](https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/networking/ovn-kubernetes-network-plugin#using-an-egress-router-ovn) support redirect mode only. The HTTP proxy mode and DNS proxy mode, which are available in OpenShiftSDN, and not supported in OVNKubernetes.

# Limited live migration procedure

Having completed the pre-flight checks, we are now prepared to migrate a cluster to OVNKubernetes. This section serves as a detailed guide through the migration process. We will begin by creating a backup of the Etcd database.

## Creating backup of etcd database

The following steps are derived from the documentation section [Backing up etcd data](https://docs.openshift.com/container-platform/4.14/backup_and_restore/control_plane_backup_and_restore/backing-up-etcd.html#backing-up-etcd-data_backup-etcd).

Start a debug session as root for a control plane node, for example:

{{< highlight shell "linenos=table" >}}
$ oc debug -n default node/master1.example.corp
{{< / highlight >}}

Change your root directory to `/host` in the debug shell:

{{< highlight shell "linenos=table" >}}
$ chroot /host
{{< / highlight >}}

Run the `cluster-backup.sh` script in the debug shell and pass in the location to save the backup to:

{{< highlight shell "linenos=table" >}}
$ /usr/local/bin/cluster-backup.sh /home/core/assets/backup
{{< / highlight >}}

Verify that the backup was created successfully. You should see that two files were created in the target directory:

{{< highlight shell "linenos=table" >}}
$ ls -al /home/core/assets/backup
total 99136
drwxr-xr-x. 2 root root        96 Oct 16 21:55 .
drwxr-xr-x. 3 root root        20 Oct 16 21:55 ..
-rw-------. 1 root root 101437472 Oct 16 21:55 snapshot_2024-10-16_215538.db
-rw-------. 1 root root     70434 Oct 16 21:55 static_kuberesources_2024-10-16_215538.tar.gz
{{< / highlight >}}

## Creating backup of cluster network configuration

Back up the configuration of the cluster network, enter the following command:

{{< highlight shell "linenos=table" >}}
$ oc get network.config.openshift.io cluster -o yaml > cluster-openshift-sdn.yaml
{{< / highlight >}}

## Enabling routing via host (optional)

By default, OVNKubernetes does not pass the network traffic through the host network stack when sending out traffic. Instead, egress traffic is always sent directly through the host interface connected to the `br-ex` bridge. If your cluster node comes with secondary network interfaces that are necessary for accessing specific network destinations, you will need to enable the `routingViaHost` setting and set `ipForwarding` to `Global` within the OVNKubernetes configuration. Additional details can be found in the knowledgebase article [6962558 OVN-Kubernetes ignores host routes for outgoing traffic](https://access.redhat.com/solutions/6962558).

To configure routing via host, issue the following command:

{{< highlight shell "linenos=table" >}}
$ oc patch network.operator/cluster --type merge -p '{"spec":{"defaultNetwork":{"ovnKubernetesConfig":{"gatewayConfig":{"ipForwarding":"Global","routingViaHost":true}}}}}'
{{< / highlight >}}

{{< highlight shell "linenos=table" >}}
network.operator.openshift.io/cluster patched
{{< / highlight >}}

## Executing migration from OpenShiftSDN to OVNKubernetes

Verify that all cluster operators are available, NOT progressing, and NOT degraded:

{{< highlight shell "linenos=table" >}}
$ oc get co
{{< / highlight >}}

In a separate console, you can follow the network operator logs throughout the update:

{{< highlight shell "linenos=table" >}}
$ oc logs --tail 0 -f -n openshift-network-operator deploy/network-operator
{{< / highlight >}}

To initiate the migration from OpenShift SDN to OVN-Kubernetes, enter the following command:

{{< highlight shell "linenos=table" >}}
$ oc patch network.config.openshift.io cluster --type merge --patch '{"metadata":{"annotations":{"network.openshift.io/network-type-migration":""}},"spec":{"networkType":"OVNKubernetes"}}'
{{< / highlight >}}

After issuing the above command, the migration process begins. During this process, the machine config operator reboots the nodes in your cluster two times. The above patch command triggers a roll-out of the OVNKubernetes control plane in the `openshift-ovn-kubernetes` namespace:

{{< highlight shell "linenos=table" >}}
$ oc get po -n openshift-ovn-kubernetes
{{< / highlight >}}

{{< highlight shell "linenos=table" >}}
NAME                                     READY   STATUS    RESTARTS   AGE
ovnkube-control-plane-74cf778b44-h89bl   2/2     Running   0          2m59s
ovnkube-node-4hshg                       8/8     Running   0          56s
ovnkube-node-526jq                       8/8     Running   0          61s
ovnkube-node-jmvvm                       8/8     Running   0          77s
{{< / highlight >}}

The machine config operator will apply new machine configs to all the nodes in the cluster in preparation for the OVN-Kubernetes deployment on the nodes.

Confirm the status of the new machine configuration on the hosts. To list the machine configuration state and the name of the applied machine configuration, enter the following command:

{{< highlight shell "linenos=table" >}}
$ oc describe node | egrep "hostname|machineconfig"
{{< / highlight >}}

Example output:

{{< highlight shell "linenos=table" >}}
kubernetes.io/hostname=master-0
machineconfiguration.openshift.io/currentConfig: rendered-master-c53e221d9d24e1c8bb6ee89dd3d8ad7b
machineconfiguration.openshift.io/desiredConfig: rendered-master-c53e221d9d24e1c8bb6ee89dd3d8ad7b
machineconfiguration.openshift.io/reason:
machineconfiguration.openshift.io/state: Done
{{< / highlight >}}

Verify that the following statements are true:

* The value of `machineconfiguration.openshift.io/state` field is `Done`.
* The value of the `machineconfiguration.openshift.io/currentConfig` field is equal to the value of the `machineconfiguration.openshift.io/desiredConfig` field.

To confirm that the machine config is correct, enter the following command, where `<config_name>` is the name of the machine config from the `machineconfiguration.openshift.io/currentConfig` field:

{{< highlight shell "linenos=table" >}}
$ oc get machineconfig <config_name> -o yaml | grep ExecStart
{{< / highlight >}}

The machine config should include the following update to the systemd configuration:

{{< highlight shell "linenos=table" >}}
ExecStart=/usr/local/bin/configure-ovs.sh OVNKubernetes
{{< / highlight >}}

Check that the reboot is finished by running the following command:

{{< highlight shell "linenos=table" >}}
$ oc get mcp
{{< / highlight >}}

Example output:

{{< highlight shell "linenos=table" >}}
NAME     CONFIG                                             UPDATED   UPDATING   DEGRADED   MACHINECOUNT   READYMACHINECOUNT   UPDATEDMACHINECOUNT   DEGRADEDMACHINECOUNT   AGE
master   rendered-master-c49da2b396ccce1e0208970073433bc7   True      False      False      1              1                   1                     0                      6d20h
worker   rendered-worker-aab994c808afb2014ce1b4c71b232d2f   True      False      False      2              2                   2                     0                      6d20h
{{< / highlight >}}

In the above output, check that the column values are UPDATED=True, UPDATING=False, DEGRADED=False.

Verify that the Multus daemon set rollout is complete before continuing with subsequent steps:

{{< highlight shell "linenos=table" >}}
$ oc -n openshift-multus rollout status daemonset/multus
{{< / highlight >}}

Expected output:

{{< highlight shell "linenos=table" >}}
daemon set "multus" successfully rolled out
{{< / highlight >}}

To confirm that the network plugin is OVN-Kubernetes, enter the following command:

{{< highlight shell "linenos=table" >}}
$ oc get network.config.openshift.io cluster -o jsonpath='{.status.networkType}'
{{< / highlight >}}

Expected output:

{{< highlight shell "linenos=table" >}}
OVNKubernetes
{{< / highlight >}}

Check the network configuration status conditions:

{{< highlight shell "linenos=table" >}}
$ oc get network.config.openshift.io cluster -o yaml
{{< / highlight >}}

The conditions should indicate that the migration is complete like this:

{{< highlight yaml "linenos=table" >}}
...
status:
  clusterNetwork:
  - cidr: 10.84.0.0/14
    hostPrefix: 22
  clusterNetworkMTU: 8900
  conditions:
  - lastTransitionTime: "2024-10-17T19:03:37Z"
    message: ""
    reason: AsExpected
    status: "True"
    type: NetworkDiagnosticsAvailable
  - lastTransitionTime: "2024-10-17T19:50:28Z"
    message: Network type migration is not in progress
    reason: NetworkTypeMigrationNotInProgress
    status: Unknown
    type: NetworkTypeMigrationMTUReady
  - lastTransitionTime: "2024-10-17T19:50:28Z"
    message: Network type migration is not in progress
    reason: NetworkTypeMigrationNotInProgress
    status: Unknown
    type: NetworkTypeMigrationTargetCNIAvailable
  - lastTransitionTime: "2024-10-17T19:50:28Z"
    message: Network type migration is not in progress
    reason: NetworkTypeMigrationNotInProgress
    status: Unknown
    type: NetworkTypeMigrationTargetCNIInUse
  - lastTransitionTime: "2024-10-17T19:50:28Z"
    message: Network type migration is not in progress
    reason: NetworkTypeMigrationNotInProgress
    status: Unknown
    type: NetworkTypeMigrationOriginalCNIPurged
  - lastTransitionTime: "2024-10-17T19:50:28Z"
    message: Network type migration is completed
    reason: NetworkTypeMigrationCompleted
    status: "False"
    type: NetworkTypeMigrationInProgress
  networkType: OVNKubernetes
  serviceNetwork:
  - 10.88.0.0/16
{{< / highlight >}}

In the above output, verify the cluster network MTU. The MTU value should be 8900 (50 bytes less than in the OpenShiftSDN status listed previously):

{{< highlight yaml "linenos=table" >}}
...
status:
...
  clusterNetworkMTU: 8900
...
{{< / highlight >}}

Confirm that the cluster nodes are in the Ready state, enter the following command:

{{< highlight shell "linenos=table" >}}
$ oc get nodes
{{< / highlight >}}

{{< highlight shell "linenos=table" >}}
NAME      STATUS   ROLES                         AGE   VERSION
master1   Ready    control-plane,master,worker   16h   v1.27.12+7bee54d
master2   Ready    control-plane,master,worker   16h   v1.27.12+7bee54d
master3   Ready    control-plane,master,worker   16h   v1.27.12+7bee54d
{{< / highlight >}}

Confirm that all pods are either running or complete:

{{< highlight shell "linenos=table" >}}
$ oc get pods --all-namespaces -o wide --sort-by='{.spec.nodeName}' | grep -v Running | grep -v Complete
{{< / highlight >}}

Verify that all cluster operators are available, NOT progressing, and NOT degraded:

{{< highlight shell "linenos=table" >}}
$ oc get co
{{< / highlight >}}

## Cleaning up after migration completed

After a successful migration, you can remove the migration configuration from the CNO object using the following command:

{{< highlight shell "linenos=table" >}}
$ oc patch network.operator.openshift.io cluster --type merge --patch '{"spec":{"migration":null}}'
{{< / highlight >}}

{{< highlight shell "linenos=table" >}}
network.operator.openshift.io/cluster patched (no change)
{{< / highlight >}}

Remove custom configuration of the OpenShift SDN network provider using the following command:

{{< highlight shell "linenos=table" >}}
$ oc patch network.operator.openshift.io cluster --type merge --patch '{"spec":{"defaultNetwork":{"openshiftSDNConfig":null}}}'
{{< / highlight >}}

{{< highlight shell "linenos=table" >}}
network.operator.openshift.io/cluster patched (no change)
{{< / highlight >}}

Remove the OpenShift SDN control plane namespace. Enter the following command:

{{< highlight shell "linenos=table" >}}
$ oc delete namespace openshift-sdn
{{< / highlight >}}

{{< highlight shell "linenos=table" >}}
namespace "openshift-sdn" deleted
{{< / highlight >}}

## Removing backup of the etcd database

Start a debug session as root for a control plane node, for example:

{{< highlight shell "linenos=table" >}}
$ oc debug -n default node/master1.example.corp
{{< / highlight >}}

Change your root directory to `/host` in the debug shell:

{{< highlight shell "linenos=table" >}}
$ chroot /host
{{< / highlight >}}

Remove the etcd backup:

{{< highlight shell "linenos=table" >}}
$ rm -rf /home/core/assets/backup
{{< / highlight >}}

# Modifications to node network during migration process

During the migration process, the OVS configuration within the cluster undergoes multiple modifications. This section emphasizes some of the alterations you may notice on the nodes.

During the migration, the OpenShiftSDN bridge `br0` is removed and the bridges `br-int`, `br-ex`, and `br-ext` appear on the node. You can examine the bridges by issuing the following command on the cluster node:


{{< highlight shell "linenos=table" >}}
$ sudo ovs-vsctl show
{{< / highlight >}}

{{< highlight shell "linenos=table" >}}
f1aafa3b-f62d-4a7f-beb1-5f930c0e9cd6
    Bridge br-int
        fail_mode: secure
        datapath_type: system
        Port ovn-k8s-mp0
            Interface ovn-k8s-mp0
                type: internal
        Port d959dd57b4835f9
            Interface d959dd57b4835f9
        Port int
            Interface int
                type: patch
                options: {peer=ext}
        Port br-int
            Interface br-int
                type: internal
        Port ovn-9929ca-0
            Interface ovn-9929ca-0
                type: geneve
                options: {csum="true", key=flow, local_ip="192.168.20.23", remote_ip="192.168.20.21"}
        Port patch-br-int-to-br-ex_worker2
            Interface patch-br-int-to-br-ex_worker2
                type: patch
                options: {peer=patch-br-ex_worker2-to-br-int}
        Port "2fa94e4a6dca1ea"
            Interface "2fa94e4a6dca1ea"
        Port af6f10ed5b47464
            Interface af6f10ed5b47464
        Port "1ce93656430cb65"
            Interface "1ce93656430cb65"
    Bridge br-ex
        Port ens3
            Interface ens3
                type: system
        Port patch-br-ex_worker2-to-br-int
            Interface patch-br-ex_worker2-to-br-int
                type: patch
                options: {peer=patch-br-int-to-br-ex_worker2}
        Port br-ex
            Interface br-ex
                type: internal
    Bridge br-ext
        fail_mode: secure
        Port br-ext
            Interface br-ext
                type: internal
        Port ext
            Interface ext
                type: patch
                options: {peer=int}
        Port ext-vxlan
            Interface ext-vxlan
                type: vxlan
                options: {dst_port="4789", key=flow, remote_ip=flow}
    ovs_version: "3.3.2-40.el9fdp"
{{< / highlight >}}

The `br-ext` bridge is a hybrid overlay bridge that ensures connectivity between the OpenShiftSDN and OVNKubernetes networks. Subsequently, during the migration process, this `br-ext` bridge is removed:

{{< highlight shell "linenos=table" >}}
$ sudo ovs-vsctl show
{{< / highlight >}}

{{< highlight shell "linenos=table" >}}
f1aafa3b-f62d-4a7f-beb1-5f930c0e9cd6
    Bridge br-int
        fail_mode: secure
        datapath_type: system
        Port ovn-k8s-mp0
            Interface ovn-k8s-mp0
                type: internal
        Port d959dd57b4835f9
            Interface d959dd57b4835f9
        Port int
            Interface int
                type: patch
                options: {peer=ext}
        Port br-int
            Interface br-int
                type: internal
        Port ovn-9929ca-0
            Interface ovn-9929ca-0
                type: geneve
                options: {csum="true", key=flow, local_ip="192.168.20.23", remote_ip="192.168.20.21"}
        Port patch-br-int-to-br-ex_worker2
            Interface patch-br-int-to-br-ex_worker2
                type: patch
                options: {peer=patch-br-ex_worker2-to-br-int}
        Port "2fa94e4a6dca1ea"
            Interface "2fa94e4a6dca1ea"
        Port af6f10ed5b47464
            Interface af6f10ed5b47464
        Port "1ce93656430cb65"
            Interface "1ce93656430cb65"
    Bridge br-ex
        Port ens3
            Interface ens3
                type: system
        Port patch-br-ex_worker2-to-br-int
            Interface patch-br-ex_worker2-to-br-int
                type: patch
                options: {peer=patch-br-int-to-br-ex_worker2}
        Port br-ex
            Interface br-ex
                type: internal
{{< / highlight >}}

# Conclusion

In this blog we discussed the migration of the cluster network from OpenShiftSDN to OVNKubernetes. We started by outlining the limited live migration method from a high-level perspective. This method allows for the conversion of the existing cluster to OVNKubernetes without interrupting the workloads that are currently running on the cluster. However, there are some important considerations to keep in mind. Specifically, egress IP addresses and multicast communication are temporarily disabled during the migration. Additionally, the migration process does not accommodate the migration of egress router pods. These pods must be removed by the administrator and recreated once the migration is finalized. Subsequently, we detailed the limited live migration process itself, including example commands to monitor the progress of the migration. In conclusion, we examined some of the modifications made to the cluster node throughout the migration process.

I hope you found the OpenShift network migration process outlined in this blog to be informative. Should you have any questions or comments, please feel free to leave them in the comment section below. I look forward to receiving your feedback!
