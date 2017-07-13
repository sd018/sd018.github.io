---
layout:     post
title:      "Ceph NTP Server Configuration "
subtitle:   " \"如何部署NTP Server\""
date:       2017-6-6 14:43:01  
author:     "sd018"
header-img: "img/post-bg-unix-linux.jpg"
tags:
    - ceph
---


> 搭建多节点Ceph存储集群时，若不配置NTP服务，则会发生时间同步问题。本文简单介绍下Ceph服务器NTP的设置。


发生时钟同步问题时，Ceph集群报错信息如下

```ruby
[root@vm-shalce97 ~]# ceph -s
    cluster 84db831c-1f29-44a7-a601-eddd24a70ae4
     health HEALTH_WARN
     clock skew detected on vm-shalce99
     monmap e1: 3 mons at {vm-shalce97=10.72.243.113:6789/0,vm-shalce98=10.72.243.114:6789/0,vm-shalce99=10.72.243.115:6789/0}
            election epoch 70, quorum 0,1,2 vm-shalce97,vm-shalce98,vm-shalce99
      fsmap e43: 1/1/1 up {0=vm-shalce97=up:active}, 1 up:standby
     osdmap e481: 9 osds: 9 up, 9 in
            flags sortbitwise,require_jewel_osds
      pgmap v6219179: 776 pgs, 14 pools, 134 GB data, 46238 objects
            269 GB used, 584 GB / 854 GB avail
                 776 active+clean
  client io 26400 B/s wr, 0 op/s rd, 7 op/s wr
```
核心信息就是: **clock skew detected on vm-shalce99**

# 配置NTP服务

首先查看NTP是否正常安装
```ruby
systemctl status ntpd
● ntpd.service - Network Time Service
   Loaded: loaded (/usr/lib/systemd/system/ntpd.service; disabled; vendor preset: disabled)
   Active: inactive (dead)
```
启动服务并设置开机自动运行  
```ruby
systemctl start ntpd && systemctl enable ntpd
```  
配置NTP config文件
注释掉以下几行  
server 0.centos.pool.ntp.org iburst  
server 1.centos.pool.ntp.org iburst  
server 2.centos.pool.ntp.org iburst  
server 3.centos.pool.ntp.org iburst  
并增加新的NTP服务器  
server 10.72.1.9  
```ruby
vim /etc/ntp.conf
# For more information about this file, see the man pages
# ntp.conf(5), ntp_acc(5), ntp_auth(5), ntp_clock(5), ntp_misc(5), ntp_mon(5).

driftfile /var/lib/ntp/drift

# Permit time synchronization with our time source, but do not
# permit the source to query or modify the service on this system.
restrict default nomodify notrap nopeer noquery

# Permit all access over the loopback interface.  This could
# be tightened as well, but to do so would effect some of
# the administrative functions.
restrict 127.0.0.1
restrict ::1

# Hosts on local network are less restricted.
#restrict 192.168.1.0 mask 255.255.255.0 nomodify notrap

# Use public servers from the pool.ntp.org project.
# Please consider joining the pool (http://www.pool.ntp.org/join.html).
#server 0.centos.pool.ntp.org iburst
#server 1.centos.pool.ntp.org iburst
#server 2.centos.pool.ntp.org iburst
#server 3.centos.pool.ntp.org iburst

###add following lines:
server 10.72.1.9

#broadcast 192.168.1.255 autokey	# broadcast server
#broadcastclient			# broadcast client
#broadcast 224.0.1.1 autokey		# multicast server
#multicastclient 224.0.1.1		# multicast client
#manycastserver 239.255.254.254		# manycast server
#manycastclient 239.255.254.254 autokey # manycast client

# Enable public key cryptography.
#crypto

includefile /etc/ntp/crypto/pw

# Key file containing the keys and key identifiers used when operating
# with symmetric key cryptography.
keys /etc/ntp/keys

# Specify the key identifiers which are trusted.
#trustedkey 4 8 42

# Specify the key identifier to use with the ntpdc utility.
#requestkey 8

# Specify the key identifier to use with the ntpq utility.
#controlkey 8

# Enable writing of statistics records.
#statistics clockstats cryptostats loopstats peerstats

# Disable the monitoring facility to prevent amplification attacks using ntpdc
# monlist command when default restrict does not include the noquery flag. See
# CVE-2013-5211 for more details.
# Note: Monitoring will not be disabled with the limited restriction flag.
disable monitor
```  
重启服务
```ruby
systemctl daemon-reload && systemctl restart ntpd
```
检查NTP服务正常运行`ntpq  -p`
```ruby
[root@vm-shalce97 ~]# ntpq  -p
     remote           refid      st t when poll reach   delay   offset  jitter
==============================================================================
 shasatdc02.aego .LOCL.           1 u   50   64  377    2.578    3.574   7.217
```
可以观察到时间同步delay时间不断趋近，时间不会超过5分钟，表示NTP配置正常。
在其他Ceph节点依次进行如上操作，使整个群集时钟同步。

手动同步命令
```
ntpdate -u {HostI P}
```
