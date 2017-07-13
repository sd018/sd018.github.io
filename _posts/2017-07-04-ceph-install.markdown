---
layout:     post
title:      "ceph install  "
subtitle:   " \"ceph jewel版完整部署  \""
date:       2017-7-11 14:33:01  
author:     "sd018"
header-img: "img/post-bg-ceph.jpg"
tags:
    - ceph
---

> 本文介绍在Centos 7 环境下，通过ceph-deploy工具部署包含1个admin节点，三个node节点的ceph 存储集群。通过搭建了解ceph 集群的基础架构及相关命令。

# 环境
![](https://github.com/sd018/sd018.github.io/blob/master/img/ceph_inf.jpg?raw=true)
三台Cenos7主机，每台主机5块磁盘。一块模拟ssd作为journal盘。其余三块当做OSD盘。

# yum源配置

备份原始Yum源
```bash
mkdir /etc/yum.repos.d/bk
cp /etc/yum.repos.d/*.repo /etc/yum.repos.d/bk
wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
wget -O /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo
sed -i '/aliyuncs/d' /etc/yum.repos.d/CentOS-Base.repo
sed -i '/aliyuncs/d' /etc/yum.repos.d/epel.repo
sed -i 's/$releasever/7/g' /etc/yum.repos.d/CentOS-Base.repo
```
新增ceph源
```bash
cat <<EOF> /etc/yum.repos.d/ceph.repo
[ceph]
name=ceph
baseurl=http://mirrors.163.com/ceph/rpm-jewel/el7/x86_64/
gpgcheck=0
[ceph-noarch]
name=cephnoarch
baseurl=http://mirrors.163.com/ceph/rpm-jewel/el7/noarch/
gpgcheck=0
EOF
```
# Ceph集群初始化配置
*注：以下步骤各个Ceph节点都要执行*

关闭防火墙
```bash
sed -i 's/SELINUX=.*/SELINUX=disabled/' /etc/selinux/config
setenforce 0
systemctl stop firewalld
systemctl disable firewalld
```
安装ceph

```bash
yum install ceph ceph-radosgw
```

* OS参数配优化置  

所有/proc/sys/  * 参数的配置方法，比如上面的pid_max，这一类的参数配置方法都类似，就是把值写入到/etc/sysctl.conf文件中即可：

`/proc/sys/kernel/pid_max` 参数配置方法如下:
```bash
echo 'kernel.pid_max = 4194303' >> /etc/sysctl.conf
```
`/proc/sys/fs/file-max` 参数配置方法如下:
```bash
echo 'fs.file-max = 26234859' >> /etc/sysctl.conf
```
* 调整文件句柄最大数，默认是1024，这里我们需要把这个值放大到
```bash
echo '*  soft  nofile  65536' >> /etc/security/limits.conf
echo '*  hard  nofile  65536' >> /etc/security/limits.conf
```
* 修改磁盘的I/O Scheduler
```bash
echo 'SUBSYSTEM=="block", ATTR{device/model}=="VBOX HARDDISK", ACTION=="add|change", KERNEL=="sd[a-h]", ATTR{queue/scheduler}="noop",ATTR{queue/read_ahead_kb}="8192"' > /etc/udev/rules.d/99-disk.rules
```

格式化磁盘,作为journal盘

```bash
[root@vm-shaltt90 ~]# fdisk /dev/sde
Welcome to fdisk (util-linux 2.23.2).

Changes will remain in memory only, /until you decide to write them.
Be careful before using the write command.

Device does not contain a recognized partition table
Building a new DOS disklabel with disk identifier 0xb1043aa8.

Command (m for help): g
Building a new GPT disklabel (GUID: B80F59B8-E365-4023-B4F7-8A6325E7A1EF)


Command (m for help): n
Partition number (1-128, default 1):
First sector (2048-20971486, default 2048):
Last sector, +sectors or +size{K,M,G,T,P} (2048-20971486, default 20971486):
Created partition 1


Command (m for help): w
The partition table has been altered!

Calling ioctl() to re-read partition table.
Syncing disks.
[root@vm-shaltt90 ~]# partprobe
```
划分好磁盘后，分区情况如下：
```bash
[root@vm-shaltt91 ~]# lsblk
NAME            MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
fd0               2:0    1    4K  0 disk
sda               8:0    0   20G  0 disk
├─sda1            8:1    0  500M  0 part /boot
└─sda2            8:2    0 19.5G  0 part
  ├─centos-root 253:0    0 17.5G  0 lvm  /
  └─centos-swap 253:1    0    2G  0 lvm  [SWAP]
sdb               8:16   0   10G  0 disk
sdc               8:32   0   10G  0 disk
sdd               8:48   0   10G  0 disk
sde               8:64   0    6G  0 disk
└─sde1            8:65   0    6G  0 part
sr0              11:0    1 1024M  0 rom
```
# NTP配置
如何配置NTP服务请参考另一篇博文。  
[Ceph-NTP-Deployment](https://sd018.github.io/2017/06/06/CephNTP-deployment/)



#  部署ceph客户端
*注：以下步骤部署在主节点*

安装Ceph-deploy
```
[root@vm-shalce99 ~]#  yum install ceph-deploy
[root@vm-shalce99 ~]#  ceph-deploy --version
1.5.37
[root@vm-shalce99 ~]# ceph -v
ceph version 10.2.5 (c461ee19ecbc0c5c330aca20f7392c9a00730367)
```
生成ssh密钥。
```
[root@vm-shaltt90 cluster]# ssh-keygen
Generating public/private rsa key pair.
Enter file in which to save the key (/root/.ssh/id_rsa):
Created directory '/root/.ssh'.
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /root/.ssh/id_rsa.
Your public key has been saved in /root/.ssh/id_rsa.pub.
The key fingerprint is:
2e:52:b7:27:28:dd:3b:09:33:24:9f:91:6e:1a:9d:b9 root@vm-shaltt90.aegonthtf.com
The key's randomart image is:
+--[ RSA 2048]----+
|                 |
|                 |
|       .         |
|    . +          |
|     *.=S        |
|    .o%= .       |
|    o++==..      |
|    .oE.o+       |
|        ..       |
+-----------------+
```

将ssh密钥复制到各节点
```
[root@vm-shaltt90 cluster]# ssh-copy-id vm-shaltt91
The authenticity of host 'vm-shaltt91 (10.72.240.111)' can't be established.
ECDSA key fingerprint is c9:0d:07:4e:92:70:3d:d1:28:63:3c:ba:34:ec:69:c0.
Are you sure you want to continue connecting (yes/no)? yes
/usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed
/usr/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed -- if you are prompted now it is to install the new keys
root@vm-shaltt91's password:

Number of key(s) added: 1

Now try logging into the machine, with:   "ssh 'vm-shaltt91'"
and check to make sure that only the key(s) you wanted were added.

[root@vm-shaltt90 cluster]# ssh-copy-id vm-shaltt96
The authenticity of host 'vm-shaltt96 (10.72.240.205)' can't be established.
ECDSA key fingerprint is c9:0d:07:4e:92:70:3d:d1:28:63:3c:ba:34:ec:69:c0.
Are you sure you want to continue connecting (yes/no)? yes
/usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed
/usr/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed -- if you are prompted now it is to install the new keys
root@vm-shaltt96's password:

Number of key(s) added: 1

Now try logging into the machine, with:   "ssh 'vm-shaltt96'"
and check to make sure that only the key(s) you wanted were added.

```
在部署节点创建部署目录并开始部署：
```
[root@vm-shaltt90 ~]#  cd
[root@vm-shaltt90 ~]# mkdir cluster
[root@vm-shaltt90 ~]# cd cluster/
```
执行部署命令
```
[root@vm-shaltt90 cluster]# ceph-deploy new vm-shaltt90 vm-shaltt91 vm-shaltt96
[ceph_deploy.conf][DEBUG ] found configuration file at: /root/.cephdeploy.conf

...若干log

[ceph_deploy.new][DEBUG ] Creating a random mon key...
[ceph_deploy.new][DEBUG ] Writing monitor keyring to ceph.mon.keyring...
[ceph_deploy.new][DEBUG ] Writing initial config to ceph.conf...
```

此时cluster文件夹内容如下
```bash
[root@vm-shaltt90 cluster]# ls
ceph.conf  ceph-deploy-ceph.log  ceph.mon.keyring
```
修改ceph.conf文件
* 下面这个配置项用于OSD启动时不自动更新crush，因为我们要自定义CRUSH。
  osd_crush_update_on_start = false
* 日志盘默认为5G，下面设置为2G
  osd_journal_size = 2048
```
[root@vm-shaltt90 cluster]# echo public_network=10.72.240.0/24 >> ceph.conf
[root@vm-shaltt90 cluster]# echo 'osd_crush_update_on_start = false' >> ceph.conf
[root@vm-shaltt90 cluster]# echo 'osd_journal_size = 2048' >> ceph.conf
```

部署monitor
```
[root@vm-shaltt90 cluster]#  ceph-deploy mon create-initial
[ceph_deploy.conf][DEBUG ] found configuration file at: /root/.cephdeploy.conf
[ceph_deploy.cli][INFO  ] Invoked (1.5.37): /usr/bin/ceph-deploy mon create-initial
[ceph_deploy.cli][INFO  ] ceph-deploy options:
...若干log
[ceph_deploy.gatherkeys][INFO  ] Storing ceph.client.admin.keyring
[ceph_deploy.gatherkeys][INFO  ] Storing ceph.bootstrap-mds.keyring
[ceph_deploy.gatherkeys][INFO  ] keyring 'ceph.mon.keyring' already exists
[ceph_deploy.gatherkeys][INFO  ] Storing ceph.bootstrap-osd.keyring
[ceph_deploy.gatherkeys][INFO  ] Storing ceph.bootstrap-rgw.keyring
[ceph_deploy.gatherkeys][INFO  ] Destroy temp directory /tmp/tmphJvbET
```
此时cluster文件夹内容如下
```bash
[root@ceph-1 cluster]# ls
ceph.bootstrap-mds.keyring  ceph.bootstrap-rgw.keyring  ceph.conf             ceph.mon.keyring
ceph.bootstrap-osd.keyring  ceph.client.admin.keyring   ceph-deploy-ceph.log
```
查看集群状态：
```bash
[root@vm-shaltt90 cluster]# ceph -s
    cluster 8ebfa0a5-ff4a-4348-8375-debebaf8acb6
     health HEALTH_WARN
            clock skew detected on mon.vm-shaltt91, mon.vm-shaltt96
            Monitor clock skew detected
     monmap e1: 3 mons at {vm-shaltt90=10.72.240.110:6789/0,vm-shaltt91=10.72.240.111:6789/0,vm-shaltt96=10.72.240.205:6789/0}
            election epoch 64, quorum 0,1,2 vm-shaltt90,vm-shaltt91,vm-shaltt96
      fsmap e23: 1/1/1 up {0=vm-shaltt90=up:active}, 1 up:standby
     osdmap e154: 9 osds: 9 up, 9 in
            flags sortbitwise,require_jewel_osds
      pgmap v41093: 456 pgs, 12 pools, 31849 bytes data, 198 objects
            394 MB used, 91665 MB / 92060 MB avail
                 456 active+clean
```
开始部署OSD， journal盘对应关系如下:

| OSD盘    | journal盘  |备注     |
| :--------| :--------- | :----- |
| /dev/sdb | /dev/sde1  |10G     |
| /dev/sdc | /dev/sde2  |10G     |
| /dev/sdd | /dev/sde1  |10G     |



执行OSD Prepare
```bash
[root@vm-shaltt90 cluster]# ceph-deploy --overwrite-conf osd prepare vm-shaltt90:/dev/sdb:/dev/sde1 vm-shaltt90:/dev/sdc:/dev/sde2 vm-shaltt90:/dev/sdd:/dev/sde3 vm-shaltt91:/dev/sdb:/dev/sde1 vm-shaltt91:/dev/sdc:/dev/sde2 vm-shaltt91:/dev/sdd:/dev/sde3 vm-shaltt96:/dev/sdb:/dev/sde1 vm-shaltt96:/dev/sdc:/dev/sde2 vm-shaltt96:/dev/sdd:/dev/sde3
[ceph_deploy.conf][DEBUG ] found configuration file at: /root/.cephdeploy.conf
[ceph_deploy.cli][INFO  ] Invoked (1.5.37): /usr/bin/ceph-deploy --overwrite-conf osd prepare vm-shaltt90:/dev/sdb:/dev/sde1 vm-shaltt90:/dev/sdc:/dev/sde2 vm-shaltt90:/dev/sdd:/dev/sde3 vm-shaltt91:/dev/sdb:/dev/sde1 vm-shaltt91:/dev/sdc:/dev/sde2 vm-shaltt91:/dev/sdd:/dev/sde3 vm-shaltt96:/dev/sdb:/dev/sde1 vm-shaltt96:/dev/sdc:/dev/sde2 vm-shaltt96:/dev/sdd:/dev/sde3
...若干log
[vm-shaltt96][INFO  ] checking OSD status...
[vm-shaltt96][DEBUG ] find the location of an executable
[vm-shaltt96][INFO  ] Running command: /bin/ceph --cluster=ceph osd stat --format=json
[vm-shaltt96][WARNIN] there are 9 OSDs down
[vm-shaltt96][WARNIN] there are 9 OSDs out
[ceph_deploy.osd][DEBUG ] Host vm-shaltt96 is now ready for osd use.
```
为journal 添加权限
```
[root@vm-shaltt90 cluster]# chown ceph:ceph /dev/sde[1-3]
```

激活OSD

```
[root@vm-shaltt90 cluster]# ceph-deploy --overwrite-conf osd activate vm-shaltt90:/dev/sdb1 vm-shaltt90:/dev/sdc1 vm-shaltt90:/dev/sdd1 vm-shaltt91:/dev/sdb1 vm-shaltt91:/dev/sdc1 vm-shaltt91:/dev/sdd1 vm-shaltt96:/dev/sdb1 vm-shaltt96:/dev/sdc1 vm-shaltt96:/dev/sdd1
[ceph_deploy.conf][DEBUG ] found configuration file at: /root/.cephdeploy.conf
[ceph_deploy.cli][INFO  ] Invoked (1.5.37): /usr/bin/ceph-deploy --overwrite-conf osd activate vm-shaltt90:/dev/sdb1 vm-shaltt90:/dev/sdc1 vm-shaltt90:/dev/sdd1 vm-shaltt91:/dev/sdb1 vm-shaltt91:/dev/sdc1 vm-shaltt91:/dev/sdd1 vm-shaltt96:/dev/sdb1 vm-shaltt96:/dev/sdc1 vm-shaltt96:/dev/sdd1
[ceph_deploy.cli][INFO  ] ceph-deploy options:
若干log...
[vm-shaltt96][INFO  ] checking OSD status...
[vm-shaltt96][DEBUG ] find the location of an executable
[vm-shaltt96][INFO  ] Running command: /bin/ceph --cluster=ceph osd stat --format=json
[vm-shaltt96][INFO  ] Running command: systemctl enable ceph.target
```

查看osd up

```
[root@vm-shaltt90 cluster]# ceph -s
    cluster 8ebfa0a5-ff4a-4348-8375-debebaf8acb6
     health HEALTH_ERR
            clock skew detected on mon.vm-shaltt91, mon.vm-shaltt96
            64 pgs are stuck inactive for more than 300 seconds
            64 pgs stuck inactive
            64 pgs stuck unclean
            too few PGs per OSD (7 < min 30)
            Monitor clock skew detected
     monmap e1: 3 mons at {vm-shaltt90=10.72.240.110:6789/0,vm-shaltt91=10.72.240.111:6789/0,vm-shaltt96=10.72.240.205:6789/0}
            election epoch 18, quorum 0,1,2 vm-shaltt90,vm-shaltt91,vm-shaltt96
     osdmap e19: 9 osds: 9 up, 9 in
            flags sortbitwise,require_jewel_osds
      pgmap v79: 64 pgs, 1 pools, 0 bytes data, 0 objects
            294 MB used, 91766 MB / 92060 MB avail
                  64 creating
```


# 如何自定义CRUSH
自定义CRUSH的能够构建OSD的组织架构，方便管理及运维。下面介绍如何自定义CRUSH的方法。
* 先查看现有的osd tree
```
[root@vm-shaltt96 ~]# ceph osd tree
ID WEIGHT TYPE NAME    UP/DOWN REWEIGHT PRIMARY-AFFINITY
-1      0 root default                                   
 0      0 osd.0             up  1.00000          1.00000
 1      0 osd.1             up  1.00000          1.00000
 2      0 osd.2             up  1.00000          1.00000
 3      0 osd.3             up  1.00000          1.00000
 4      0 osd.4             up  1.00000          1.00000
 5      0 osd.5             up  1.00000          1.00000
 6      0 osd.6             up  1.00000          1.00000
 7      0 osd.7             up  1.00000          1.00000
 8      0 osd.8             up  1.00000          1.00000
```
创建对应的Bucket

```
[root@vm-shaltt96 ~]# ceph osd crush add-bucket root-sata root
added bucket root-sata type root to crush map
[root@vm-shaltt96 ~]# ceph osd crush add-bucket root-ssd root
added bucket root-ssd type root to crush map
[root@vm-shaltt96 ~]# ceph osd tree
ID WEIGHT TYPE NAME      UP/DOWN REWEIGHT PRIMARY-AFFINITY
-3      0 root root-ssd                                    
-2      0 root root-sata                                   
-1      0 root default                                     
 0      0 osd.0               up  1.00000          1.00000
 1      0 osd.1               up  1.00000          1.00000
 2      0 osd.2               up  1.00000          1.00000
 3      0 osd.3               up  1.00000          1.00000
 4      0 osd.4               up  1.00000          1.00000
 5      0 osd.5               up  1.00000          1.00000
 6      0 osd.6               up  1.00000          1.00000
 7      0 osd.7               up  1.00000          1.00000
 8      0 osd.8               up  1.00000          1.00000
```
将对应的osd 移到对应的目录下
```
[root@vm-shaltt90 cluster]# ceph osd crush add-bucket vm-shaltt90-sata host
added bucket vm-shaltt90-sata type host to crush map
[root@vm-shaltt90 cluster]# ceph osd crush add-bucket vm-shaltt91-sata host
added bucket vm-shaltt91-sata type host to crush map
[root@vm-shaltt90 cluster]# ceph osd crush add-bucket vm-shaltt96-sata host
added bucket vm-shaltt96-sata type host to crush map
[root@vm-shaltt90 cluster]# ceph osd crush move vm-shaltt90-sata root=root-sata
moved item id -4 name 'vm-shaltt90-sata' to location {root=root-sata} in crush map
[root@vm-shaltt90 cluster]# ceph osd crush move vm-shaltt91-sata root=root-sata
moved item id -5 name 'vm-shaltt91-sata' to location {root=root-sata} in crush map
[root@vm-shaltt90 cluster]# ceph osd crush move vm-shaltt96-sata root=root-sata
moved item id -6 name 'vm-shaltt96-sata' to location {root=root-sata} in crush map
[root@vm-shaltt90 cluster]# ceph osd crush add osd.0 0.01 host=vm-shaltt90-sata
add item id 0 name 'osd.0' weight 0.01 at location {host=vm-shaltt90-sata} to crush map
[root@vm-shaltt90 cluster]# ceph osd crush add osd.1 0.01 host=vm-shaltt90-sata
add item id 1 name 'osd.1' weight 0.01 at location {host=vm-shaltt90-sata} to crush map
[root@vm-shaltt90 cluster]# ceph osd crush add osd.2 0.01 host=vm-shaltt90-sata
add item id 2 name 'osd.2' weight 0.01 at location {host=vm-shaltt90-sata} to crush map
[root@vm-shaltt90 cluster]# ceph osd crush add osd.3 0.01 host=vm-shaltt91-sata
add item id 3 name 'osd.3' weight 0.01 at location {host=vm-shaltt91-sata} to crush map
[root@vm-shaltt90 cluster]# ceph osd crush add osd.4 0.01 host=vm-shaltt91-sata
add item id 4 name 'osd.4' weight 0.01 at location {host=vm-shaltt91-sata} to crush map
[root@vm-shaltt90 cluster]# ceph osd crush add osd.5 0.01 host=vm-shaltt91-sata
add item id 5 name 'osd.5' weight 0.01 at location {host=vm-shaltt91-sata} to crush map
[root@vm-shaltt90 cluster]# ceph osd crush add osd.6 0.01 host=vm-shaltt96-sata
add item id 6 name 'osd.6' weight 0.01 at location {host=vm-shaltt96-sata} to crush map
[root@vm-shaltt90 cluster]# ceph osd crush add osd.7 0.01 host=vm-shaltt96-sata
add item id 7 name 'osd.7' weight 0.01 at location {host=vm-shaltt96-sata} to crush map
[root@vm-shaltt90 cluster]# ceph osd crush add osd.8 0.01 host=vm-shaltt96-sata
add item id 8 name 'osd.8' weight 0.01 at location {host=vm-shaltt96-sata} to crush map
```
若不小心加错了，可用下列命令删除
```bash
ceph osd crush rm osd.0 #删了再加就好了，当然万能的还是 -h
ceph osd crush -h
```

查看当前的osd tree配置
```
[root@vm-shaltt91 yum.repos.d]# ceph osd tree
ID WEIGHT  TYPE NAME                 UP/DOWN REWEIGHT PRIMARY-AFFINITY
-3       0 root root-ssd                                               
-2 0.09000 root root-sata                                              
-4 0.03000     host vm-shaltt90-sata                                   
 0 0.00999         osd.0                  up  1.00000          1.00000
 1 0.00999         osd.1                  up  1.00000          1.00000
 2 0.00999         osd.2                  up  1.00000          1.00000
-5 0.03000     host vm-shaltt91-sata                                   
 3 0.00999         osd.3                  up  1.00000          1.00000
 4 0.00999         osd.4                  up  1.00000          1.00000
 5 0.00999         osd.5                  up  1.00000          1.00000
-6 0.03000     host vm-shaltt96-sata                                   
 6 0.00999         osd.6                  up  1.00000          1.00000
 7 0.00999         osd.7                  up  1.00000          1.00000
 8 0.00999         osd.8                  up  1.00000          1.00000
-1       0 root default
```
最后保存修改的CrushMap，命令如下：
```
[root@vm-shaltt90 cluster]# ceph osd getcrushmap -o /tmp/crush
got crush map from osdmap epoch 42
[root@vm-shaltt90 cluster]# crushtool -d /tmp/crush -o /tmp/crush.txt
# -d 的意思是decompile，导出的crush是二进制格式的。
[root@vm-shaltt90 cluster]# vim /tmp/crush.txt
```
在最下面添加如下规则
```
# rules
rule rule-sata { #改个名字，好认
        ruleset 0
        type replicated
        min_size 1
        max_size 10
        step take root-sata #改成图中的SATA的root名
        step chooseleaf firstn 0 type host
        step emit
  }       
```
保存退出，编译刚刚的crush.txt，并注入集群中：
```
[root@vm-shaltt90 cluster]# vim /tmp/crush.txt
[root@vm-shaltt90 cluster]#  crushtool -c /tmp/crush.txt -o /tmp/crush.bin
[root@vm-shaltt90 cluster]# ceph osd setcrushmap -i /tmp/crush.bin
set crush map
```
最后增大rbd的pg数
```
[root@vm-shaltt90 cluster]# ceph osd pool set rbd pg_num 256
set pool 0 pg_num to 256
[root@vm-shaltt90 cluster]# ceph osd pool set rbd pgp_num 256
set pool 0 pgp_num to 256
```
# 总结
以上主要涉及Ceph的基础架构部署，其他的三种存储类型的调用，详见另一篇博客。
