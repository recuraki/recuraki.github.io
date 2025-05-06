---
layout: post
title:  "20240723: おうちのConnect-X4 LxにDOCA-OFEDを入れる"
date:   2024-07-23T00:00:00+09:00
author: Akira Kanai
categories: HostNetwork
tags:	connectx
cover:  "/assets/instacode.png"
description: 自宅のCX-4 Lx環境にDOCA-OFEDを入れました
---


- 前提: 
[NVIDIA MLNX_OFED Documentation v24.04-0.7.0.0](https://docs.nvidia.com/networking/display/mlnxofedv24040700/introduction)

>MLNX_OFED has transitioned into DOCA-Host, and now available as DOCA-OFED (learn about DOCA-Host profiles here).
>
>MLNX_OFED last standalone release is October 2024 Long Term Support (3 years). Starting January 2025 all new features will be included in DOCA-OFED only.

なのでDOCA-OFEDにできるならしたい。


- 事前確認: 

{% highlight shell %}
  Device Type:      ConnectX4LX
  Part Number:      MCX4121A-XCA_Ax
  Description:      ConnectX-4 Lx EN network interface card; 10GbE dual-port SFP28; PCIe3.0 x8; ROHS R6
  PSID:             MT_2420110004
  PCI Device Name:  0000:01:00.0
  Base MAC:         b83fd20fd7ee
  Versions:         Current        Available
     FW             14.32.1010     N/A
     PXE            3.6.0502       N/A
     UEFI           14.25.0017     N/A
{% endhighlight %}

# DOCA
[Manual](https://docs.nvidia.com/doca/sdk/nvidia+doca+installation+guide+for+linux/index.html)はこれ。

doca-hostというのが、

```
These files contain the following components suitable for their respective OS version.
DOCA Devel v2.7.0
DOCA Runtime v2.7.0
DOCA Extra v2.7.0
DOCA OFED v2.7.0
```

なのでこれを入れれば良さそう。


```
sudo su
export DOCA_URL="https://linux.mellanox.com/public/repo/doca/2.7.0/ubuntu22.04/x86_64/"
curl https://linux.mellanox.com/public/repo/doca/GPG-KEY-Mellanox.pub | gpg --dearmor > /etc/apt/trusted.gpg.d/GPG-KEY-Mellanox.pub
echo "deb [signed-by=/etc/apt/trusted.gpg.d/GPG-KEY-Mellanox.pub] $DOCA_URL ./" > /etc/apt/sources.list.d/doca.list
exit
sudo apt-get update
sudo apt-get -y install doca-all
```

```console
...
knem.ko:
 - Uninstallation
   - Deleting from: /lib/modules/5.15.0-100-generic/updates/dkms/
 - Original module
   - No original module was found for this module on this kernel.
   - Use the dkms install command to reinstall any previous module version.

depmod...
Module knem-1.1.4.90mlnx2 for kernel 5.15.0-97-generic (x86_64).
Before uninstall, this module version was ACTIVE on this kernel.

knem.ko:
 - Uninstallation
   - Deleting from: /lib/modules/5.15.0-97-generic/updates/dkms/
 - Original module
   - No original module was found for this module on this kernel.
   - Use the dkms install command to reinstall any previous module version.
...
Setting up mlnx-ofed-kernel-dkms (24.04.OFED.24.04.0.7.0.1-1) ...Loading new mlnx-ofed-kernel-24.04.OFED.24.04.0.7.0.1 DKMS files...
First Installation: checking all kernels...
Building for 5.15.0-97-generic and 5.15.0-100-generic
Building for architecture x86_64
Building initial module for 5.15.0-97-generic
```

でmakeされたので反映していく。

```shell
sudo /etc/init.d/openibd restart
sudo mst restart
sudo mst start
sudo systemctl status rshim
sudo /etc/init.d/openibd restart
```

こででいいはずだが...

```console
ubuntu@optiplex:~$
sudo /etc/init.d/openibd restart
Unloading mlx_compat                                       [FAILED]
rmmod: ERROR: Module mlx_compat is in use by: nvme_core nvme_fabrics
ubuntu@optiplex:~$ ls -alt /usr/bin/mlxfwmanager
-rwxr-xr-x 1 root root 10887728 Apr 25 15:27 /usr/bin/mlxfwmanager

ubuntu@optiplex:~$ sudo mlxconfig -d /dev/mst/mt4117_pciconf0 q

Device #1:
----------

Device type:        ConnectX4LX
Name:               MCX4121A-XCA_Ax
Description:        ConnectX-4 Lx EN network interface card; 10GbE dual-port SFP28; PCIe3.0 x8; ROHS R6
Device:             /dev/mst/mt4117_pciconf0
```

あまりmlx resetしても意味がなさそうなので

```shell
$ sudo shutdown -r now
```

したら上手くいった。原因不明。