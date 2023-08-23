+++
title = "5 Linux Commands You Didn't Know You Needed"
date = "2016-11-01"
slug = "2016/11/01/5-linux-commands-you-didnt-know-you-needed"
Categories = [ "devops" ]
+++

When preparing for the RHCSA and RHCE exams, I found several useful commands I was not really aware of. In this blog post I'll share them with you.

<!--more-->

# findmnt

The `findmnt` command is part of the essential package *util-linux* and hence is available on pretty much all Linux systems. It can print all mounted filesystems in the tree-like format. I found the output of `findmnt` command more readable than the output provided by the more popular `mount` command. This is an example of how the filesystem mounts on a Ceph node look like:

{{< highlight shell "linenos=table" >}}
$ findmnt
TARGET                           SOURCE     FSTYPE     OPTIONS
/                                /dev/sda2  xfs        rw,relatime,seclabel,attr2,inode64,noquota
├─/sys                           sysfs      sysfs      rw,nosuid,nodev,noexec,relatime,seclabel
│ ├─/sys/kernel/security         securityfs securityfs rw,nosuid,nodev,noexec,relatime
│ ├─/sys/fs/cgroup               tmpfs      tmpfs      ro,nosuid,nodev,noexec,seclabel,mode=755
│ │ ├─/sys/fs/cgroup/systemd     cgroup     cgroup     rw,nosuid,nodev,noexec,relatime,xattr,release_agent=/usr/lib/systemd/systemd-cgroups-agent,name=systemd
│ │ ├─/sys/fs/cgroup/cpu,cpuacct cgroup     cgroup     rw,nosuid,nodev,noexec,relatime,cpuacct,cpu
│ │ ├─/sys/fs/cgroup/perf_event  cgroup     cgroup     rw,nosuid,nodev,noexec,relatime,perf_event
│ │ ├─/sys/fs/cgroup/devices     cgroup     cgroup     rw,nosuid,nodev,noexec,relatime,devices
│ │ ├─/sys/fs/cgroup/blkio       cgroup     cgroup     rw,nosuid,nodev,noexec,relatime,blkio
│ │ ├─/sys/fs/cgroup/cpuset      cgroup     cgroup     rw,nosuid,nodev,noexec,relatime,cpuset
│ │ ├─/sys/fs/cgroup/hugetlb     cgroup     cgroup     rw,nosuid,nodev,noexec,relatime,hugetlb
│ │ ├─/sys/fs/cgroup/net_cls     cgroup     cgroup     rw,nosuid,nodev,noexec,relatime,net_cls
│ │ ├─/sys/fs/cgroup/memory      cgroup     cgroup     rw,nosuid,nodev,noexec,relatime,memory
│ │ └─/sys/fs/cgroup/freezer     cgroup     cgroup     rw,nosuid,nodev,noexec,relatime,freezer
│ ├─/sys/fs/pstore               pstore     pstore     rw,nosuid,nodev,noexec,relatime
│ ├─/sys/fs/selinux              selinuxfs  selinuxfs  rw,relatime
│ ├─/sys/kernel/debug            debugfs    debugfs    rw,relatime
│ └─/sys/kernel/config           configfs   configfs   rw,relatime
├─/proc                          proc       proc       rw,nosuid,nodev,noexec,relatime
│ ├─/proc/sys/fs/binfmt_misc     systemd-1  autofs     rw,relatime,fd=26,pgrp=1,timeout=300,minproto=5,maxproto=5,direct
│ └─/proc/fs/nfsd                nfsd       nfsd       rw,relatime
├─/dev                           devtmpfs   devtmpfs   rw,nosuid,seclabel,size=16307108k,nr_inodes=4076777,mode=755
│ ├─/dev/shm                     tmpfs      tmpfs      rw,nosuid,nodev,seclabel
│ ├─/dev/pts                     devpts     devpts     rw,nosuid,noexec,relatime,seclabel,gid=5,mode=620,ptmxmode=000
│ ├─/dev/mqueue                  mqueue     mqueue     rw,relatime,seclabel
│ └─/dev/hugepages               hugetlbfs  hugetlbfs  rw,relatime,seclabel
├─/run                           tmpfs      tmpfs      rw,nosuid,nodev,seclabel,mode=755
│ └─/run/user/1002               tmpfs      tmpfs      rw,nosuid,nodev,relatime,seclabel,size=3265340k,mode=700,uid=1002,gid=1002
├─/var/lib/nfs/rpc_pipefs        rpc_pipefs rpc_pipefs rw,relatime
├─/var/lib/ceph/osd/ceph-1       /dev/sdb1  xfs        rw,noatime,seclabel,attr2,inode64,logbsize=256k,sunit=512,swidth=512,noquota
├─/var/lib/ceph/osd/ceph-3       /dev/sdc1  xfs        rw,noatime,seclabel,attr2,inode64,logbsize=256k,sunit=512,swidth=512,noquota
├─/var/lib/ceph/osd/ceph-10      /dev/sdg1  xfs        rw,noatime,seclabel,attr2,inode64,logbsize=256k,sunit=512,swidth=512,noquota
├─/var/lib/ceph/osd/ceph-9       /dev/sdf1  xfs        rw,noatime,seclabel,attr2,inode64,logbsize=256k,sunit=512,swidth=512,noquota
├─/var/lib/ceph/osd/ceph-4       /dev/sdd1  xfs        rw,noatime,seclabel,attr2,inode64,logbsize=256k,sunit=512,swidth=512,noquota
└─/var/lib/ceph/osd/ceph-7       /dev/sde1  xfs        rw,noatime,seclabel,attr2,inode64,logbsize=256k,sunit=512,swidth=512,noquota
{{< / highlight >}}

# ss

The `ss` (soscket statistics) command is a replacement for the good old `netstat` command. It comes in the *iproute* package which is an essential part of all modern Linux distributions. I found `ss` command available on systems where the `netstat` command was missing. Here is a sample output of the `ss` command running on my Linux desktop:

{{< highlight shell "linenos=table" >}}
$ sudo ss -tlnp | cat
State      Recv-Q Send-Q Local Address:Port               Peer Address:Port
LISTEN     0      100          *:15929                    *:*                   users:(("skype",pid=25943,fd=49))
LISTEN     0      50           *:39964                    *:*                   users:(("java",pid=6015,fd=146))
LISTEN     0      5      192.168.122.1:53                       *:*                   users:(("dnsmasq",pid=1625,fd=6))
LISTEN     0      128          *:22                       *:*                   users:(("sshd",pid=1442,fd=3))
LISTEN     0      128         :::22                      :::*                   users:(("sshd",pid=1442,fd=4))
{{< / highlight >}}

I'm switching from using the `netstat` command to `ss`. What about you?

# ip

After years of using the `ifconfig` utility, it took me some effort to move to its modern replacement - the `ip` command. Recently, I discovered two useful features of the `ip` utility.

To obtain a detailed information about the packets transferred by individual network interfaces, use the `-s` (statistics) parameter. For example:

{{< highlight shell "linenos=table" >}}
$ ip -s link
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    RX: bytes  packets  errors  dropped overrun mcast
    1416492    20158    0       0       0       0
    TX: bytes  packets  errors  dropped carrier collsns
    1416492    20158    0       0       0       0
2: enp0s31f6: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP mode DEFAULT group default qlen 1000
    link/ether 18:66:da:21:33:87 brd ff:ff:ff:ff:ff:ff
    RX: bytes  packets  errors  dropped overrun mcast
    13776256661 26486602 0       0       0       1661059
    TX: bytes  packets  errors  dropped carrier collsns
    2313484427 9811792  0       0       0       0
{{< / highlight >}}

To figure out which network interface would be used to send a packet to the specified IP address:

{{< highlight shell "linenos=table" >}}
$ ip route get 192.168.0.1
192.168.0.1 via 10.5.0.1 dev enp0s31f6  src 10.5.0.225
    cache
{{< / highlight >}}

When sending a packet to the target destination `192.168.0.1`, the kernel will route the packet via the `enp0s31f6` interface. The IP `10.5.0.1` is my default route.

# lscpu

On modern machines the output of `cat /proc/cpuinfo` can be really long. To find out what CPU configuration a machine comes with I prefer to use the `lscpu` command. This is an example output of the `lscpu` command running on an OpenStack compute node:

{{< highlight shell "linenos=table" >}}
$ lscpu
Architecture:          x86_64
CPU op-mode(s):        32-bit, 64-bit
Byte Order:            Little Endian
CPU(s):                32
On-line CPU(s) list:   0-31
Thread(s) per core:    2
Core(s) per socket:    8
Socket(s):             2
NUMA node(s):          2
Vendor ID:             GenuineIntel
CPU family:            6
Model:                 63
Model name:            Intel(R) Xeon(R) CPU E5-2640 v3 @ 2.60GHz
Stepping:              2
CPU MHz:               1200.062
BogoMIPS:              5198.45
Virtualization:        VT-x
L1d cache:             32K
L1i cache:             32K
L2 cache:              256K
L3 cache:              20480K
NUMA node0 CPU(s):     0,2,4,6,8,10,12,14,16,18,20,22,24,26,28,30
NUMA node1 CPU(s):     1,3,5,7,9,11,13,15,17,19,21,23,25,27,29,31
{{< / highlight >}}

In the above output, the interesting lines are the `Socket(s)`, `Core(s) per socket`, `Thread(s) per core` and `CPU(s)`. In our case, we're looking at a machine with 2 physical CPUs (Sockets), each of them having 8 physical cores (Cores per socket). Each of the physical cores has 2 processing threads (Threads per core) aka logical CPUs due to the Hyper-Threading technology. In total, there are 32 logical CPUs available to the Linux scheduler to schedule a task on.

# lspci

The last command in our overview is the `lspci` command. If you ever wondered which kernel driver is controlling your hardware device, you can find out with:

{{< highlight shell "linenos=table" >}}
$ lspci -k

...

01:05.0 VGA compatible controller: Advanced Micro Devices, Inc. [AMD/ATI] RS880 [Radeon HD 4250]
        Subsystem: ASUSTeK Computer Inc. M5A88-V EVO
        Kernel driver in use: radeon
01:05.1 Audio device: Advanced Micro Devices, Inc. [AMD/ATI] RS880 HDMI Audio [Radeon HD 4200 Series]
        Subsystem: ASUSTeK Computer Inc. M5A88-V EVO
        Kernel driver in use: snd_hda_intel
02:00.0 FireWire (IEEE 1394): VIA Technologies, Inc. VT6315 Series Firewire Controller
        Subsystem: ASUSTeK Computer Inc. M5A88-V EVO
        Kernel driver in use: firewire_ohci
02:00.1 IDE interface: VIA Technologies, Inc. VT6415 PATA IDE Host Controller (rev a0)
        Subsystem: ASUSTeK Computer Inc. Motherboard
        Kernel driver in use: pata_via
03:00.0 USB controller: ASMedia Technology Inc. ASM1042 SuperSpeed USB Host Controller
        Subsystem: ASUSTeK Computer Inc. P8B WS Motherboard
        Kernel driver in use: xhci_hcd
05:00.0 Network controller: Realtek Semiconductor Co., Ltd. RTL8192CE PCIe Wireless Network Adapter (rev 01)
        Subsystem: Realtek Semiconductor Co., Ltd. RTL8192CE PCIe Wireless Network Adapter
        Kernel driver in use: rtl8192ce
06:00.0 Ethernet controller: Realtek Semiconductor Co., Ltd. RTL8111/8168/8411 PCI Express Gigabit Ethernet Controller (rev 06)
        Subsystem: ASUSTeK Computer Inc. P8P67 and other motherboards
        Kernel driver in use: r8169
{{< / highlight >}}
