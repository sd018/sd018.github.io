---
layout:     post
title:      "Docker LVM Increase "
subtitle:   " \"To Resize a direct-lvm thin pool  \""
date:       2017-6-15 13:33:01  
author:     "sd018"
header-img: "img/post-bg-docker-lvm.jpg"
tags:
    - docker
---
[TOC]

> Dokcer device 模式切换成direct-lvm后，尽管有LVM自动扩展功能。但当VG剩余空间扩充完后，则会导致存储空间异常。我们可以通过`lvs`或`lvs -a`来监控我们的资源池，也可以通过第三方监控软件从OS层面监控，如Nagios。下文介绍当资源池资源不足时，如何扩充存储空间。

# Resize a direct-lvm thin pool
若需要扩展`direct-lvm`资源池，首先在Dokcer主机上添加一块磁盘并分配给dokcer。具体步骤如下：  
查看现有存储使用情况
```bash
[root@template-centos_k8s ~]# lvs -o+seg_monitor
  LV       VG     Attr       LSize  Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert Monitor  
  root     centos -wi-ao---- 17.47g                                                              
  swap     centos -wi-ao----  8.00g                                                            
  thinpool docker twi-a-t--- 38.00g             2.46   0.05                             monitored
```

格式化磁盘操作，以`/dev/sdc`为例

```bash
[root@template-centos_k8s ~]# fdisk /dev/sdc
Welcome to fdisk (util-linux 2.23.2).
Device does not contain a recognized partition table

Building a new DOS disklabel with disk identifier 0x3e27bbd4.

Command (m for help): n
Partition type:
   p   primary (0 primary, 0 extended, 4 free)
   e   extended
Select (default p): p
Partition number (1-4, default 1):
First sector (2048-20971519, default 2048):
Using default value 2048
Last sector, +sectors or +size{K,M,G} (2048-20971519, default 20971519):
Using default value 20971519
Partition 1 of type Linux and of size 10 GiB is set

Command (m for help): t
Selected partition 1
Hex code (type L to list all codes): 8e
Changed type of partition 'Linux' to 'Linux LVM'

Command (m for help): p

Disk /dev/sdc: 10.7 GB, 10737418240 bytes, 20971520 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk label type: dos
Disk identifier: 0x3e27bbd4

   Device Boot      Start         End      Blocks   Id  System
/dev/sdc1            2048    20971519    10484736   8e  Linux LVM

Command (m for help): w
The partition table has been altered!

Calling ioctl() to re-read partition table.
Syncing disks.
```
重新加载磁盘，获取刚刚格式化的磁盘

```bash
[root@template-centos_k8s ~]# partprobe
```
用刚刚格式化的`/dev/sdc1`创建新的PV卷
```bash
[root@template-centos_k8s ~]# pvcreate /dev/sdc1
  Physical volume "/dev/sdc1" successfully created.
```
将新的磁盘扩充到`Volume group`,使用命令`vgextend `
```bash
[root@template-centos_k8s ~]# vgextend docker /dev/sdc1
Volume group "docker" successfully extended  
```
扩充`docker/thinpool`的逻辑卷，使用命令`lvmextend`
```bash
[root@template-centos_k8s ~]#  lvextend -l+100%FREE -n docker/thinpool
  Size of logical volume docker/thinpool_tdata changed from 38.00 GiB (9727 extents) to 49.20 GiB (12594 extents).
  Logical volume docker/thinpool_tdata successfully resized.
```
通过docker info 验证磁盘空间已扩容
```bash
[root@template-centos_k8s ~]# docker info
Containers: 0
 Running: 0
 Paused: 0
 Stopped: 0
Images: 9
Server Version: 1.12.6
Storage Driver: devicemapper
 Pool Name: docker-thinpool
 Pool Blocksize: 524.3 kB
 Base Device Size: 10.74 GB
 Backing Filesystem: xfs
 Data file:
 Metadata file:
 Data Space Used: 1.003 GB
 Data Space Total: 52.82 GB
 Data Space Available: 51.82 GB
 Metadata Space Used: 221.2 kB
 Metadata Space Total: 427.8 MB
 Metadata Space Available: 427.6 MB
 Thin Pool Minimum Free Space: 5.282 GB
 Udev Sync Supported: true
 Deferred Removal Enabled: true
 Deferred Deletion Enabled: false
 Deferred Deleted Device Count: 0
```
>参考：[Use the Device Mapper storage driver](https://docs.docker.com/engine/userguide/storagedriver/device-mapper-driver/)
