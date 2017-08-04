---
layout:     post
title:      "Ceph Storage Deploy "
subtitle:   " \"部署Ceph三种访问类型  \""
date:       2017-7-27 11:08:01  
author:     "sd018"
header-img: "img/post-bg-ceph.jpg"
tags:
    - ceph
---
> 本章主要介绍Ceph的三种存储访问类型，以及它们的简单介绍、部署、使用与使用场景。可以选择适合的Ceph存储的设计方案。    
 
# Ceph Object Gateway Quick Start
RGW是Ceph对象存储网关服务的RADOS Gateway的简称，是一套基于LIBRADOS接口封装而实现的FasrCGI服务，对外提供RESTful风格的对象存储访问和管理接口。RGW基于HTTP协议标准，因此非常适合用于Web类的互联网应用场景。


部署RGW实例
```
[root@vm-shaltt90 cluster]#  ceph-deploy rgw create vm-shaltt90
[ceph_deploy.conf][DEBUG ] found configuration file at: /root/.cephdeploy.conf
[ceph_deploy.cli][INFO  ] Invoked (1.5.37): /usr/bin/ceph-deploy rgw create vm-shaltt90
...若干log
[vm-shaltt90][INFO  ] Running command: systemctl start ceph-radosgw@rgw.vm-shaltt90
[vm-shaltt90][INFO  ] Running command: systemctl enable ceph.target
[ceph_deploy.rgw][INFO  ] The Ceph Object Gateway (RGW) is now running on host vm-shaltt90 and default port 7480
```
修改端口为80
```
[root@vm-shaltt90 cluster]# more ceph.conf
[global]
fsid = 8ebfa0a5-ff4a-4348-8375-debebaf8acb6
mon_initial_members = vm-shaltt90, vm-shaltt91, vm-shaltt96
mon_host = 10.72.240.110,10.72.240.111,10.72.240.205
auth_cluster_required = cephx
auth_service_required = cephx
auth_client_required = cephx

public_network=10.72.240.0/24
osd_crush_update_on_start = false
osd_journal_size = 2048
#增加下面这段
[client]
rgw frontends = civetweb port=80
```
推送配置文件至各个节点
```
[root@vm-shaltt90 cluster]# ceph-deploy --overwrite-conf config push vm-shaltt90 vm-shaltt91 vm-shaltt96
[ceph_deploy.conf][DEBUG ] found configuration file at: /root/.cephdeploy.conf
[ceph_deploy.cli][INFO  ] Invoked (1.5.37): /usr/bin/ceph-deploy --overwrite-conf config push vm-shaltt90 vm-shaltt91 vm-shaltt96
[ceph_deploy.cli][INFO  ] ceph-deploy options:
[ceph_deploy.cli][INFO  ]  username                      : None
[ceph_deploy.cli][INFO  ]  verbose                       : False
..省略若干
[vm-shaltt96][DEBUG ] detect machine type
[vm-shaltt96][DEBUG ] write cluster configuration to /etc/ceph/{cluster}.conf
```
创建admin账号
```
[root@vm-shaltt90 cluster]# radosgw-admin user create --uid="admin" --display-name="admin"
2017-07-03 15:42:39.017779 7f176f9a69c0  0 WARNING: detected a version of libcurl which contains a bug in curl_multi_wait(). enabling a workaround that may degrade performance slightly.
{
    "user_id": "admin",
    "display_name": "admin",
    "email": "",
    "suspended": 0,
    "max_buckets": 1000,
    "auid": 0,
    "subusers": [],
    "keys": [
        {
            "user": "admin",
            "access_key": "NCEO2K2KNHVFTUN71U94",
            "secret_key": "bnLBTblzLf3GD6IFFCKGSCYAyQ7ZS1YT459jk6oc"
        }
..省略若干
    "temp_url_keys": []
}
```

查询刚刚新建的账号
```
[root@vm-shaltt90 ~]# radosgw-admin user info --uid="admin"
2017-08-01 14:59:41.473532 7fcd24cb39c0  0 WARNING: detected a version of libcurl which contains a bug in curl_multi_wait(). enabling a workaround that may degrade performance slightly.
{
    "user_id": "admin",
    "display_name": "admin",
    "email": "",
    "suspended": 0,
    "max_buckets": 1000,
    "auid": 0,
    "subusers": [],
    "keys": [
        {
            "user": "admin",
            "access_key": "NCEO2K2KNHVFTUN71U94",
            "secret_key": "bnLBTblzLf3GD6IFFCKGSCYAyQ7ZS1YT459jk6oc"
        }
    ],
    "swift_keys": [],
    "caps": [],
    "op_mask": "read, write, delete",
    "default_placement": "",
    "placement_tags": [],
    "bucket_quota": {
        "enabled": false,
        "max_size_kb": -1,
        "max_objects": -1
    },
    "user_quota": {
        "enabled": false,
        "max_size_kb": -1,
        "max_objects": -1
    },
    "temp_url_keys": []
}
```
使用Amazon S3工具，选择S3 Compatible
![](https://github.com/sd018/sd018.github.io/blob/master/img/select_S3.jpg?raw=true)

添加rgw账号信息验证,相关信息可以通过上面的命令查询
![](https://github.com/sd018/sd018.github.io/blob/master/img/add_account.jpg?raw=true)
登陆后的界面
![](https://github.com/sd018/sd018.github.io/blob/master/img/S3_UI.jpg?raw=true)  

# Ceph FS Quick Start
Ceph Filesystem简称Ceph FS，是一个支持POSIX接口的文件系统存储类型。Ceph FS需要使用Metadata Server(简称MDS)来管理文件系统的命名空间以及客户端如何访问到后端OSD数据存储中。MDS以一个Daemon进程运行一个服务，即元数据服务器，主要负责Ceph FS集群中文件和目录的管理，确保他们的一致性。

  部署MDS实例
```
[root@vm-shaltt90 cluster]# ceph-deploy --overwrite-conf mds create vm-shaltt90 vm-shaltt91 vm-shaltt96
[ceph_deploy.conf][DEBUG ] found configuration file at: /root/.cephdeploy.conf
[ceph_deploy.cli][INFO  ] Invoked (1.5.37): /usr/bin/ceph-deploy --overwrite-conf mds create vm-shaltt90
[ceph_deploy.cli][INFO  ] ceph-deploy options:
...省略若干
[vm-shaltt90][INFO  ] Running command: systemctl start ceph-mds@vm-shaltt90
[vm-shaltt90][INFO  ] Running command: systemctl enable ceph.target
```
创建数据Pool和元数据Pool
```
[root@vm-shaltt90 cluster]# ceph osd pool create metadata 64
pool 'metadata' created
[root@vm-shaltt90 cluster]# ceph osd pool create data 64
pool 'data' created
[root@vm-shaltt90 cluster]# ceph fs new  cephfs  metadata data
new fs with metadata pool 10 and data pool 11
```
查询MDS状态
```
[root@vm-shaltt90 ~]# ceph mds stat
e45: 1/1/1 up {0=vm-shaltt91=up:active}, 1 up:standby
```
查看Ceph密钥
```
[root@vm-shaltt90 ~]# ceph auth list
installed auth entries:
client.admin
	key: AQC/slVZq293EhAAqXRnrvH5wxw4Jg3HNwYCVw==
	caps: [mds] allow *
	caps: [mon] allow *
	caps: [osd] allow *
```

挂载Ceph FS
```
mount.ceph  10.72.240.110:6789:/ /test  -o name=admin,secret=AQC/slVZq293EhAAqXRnrvH5wxw4Jg3HNwYCVw==
```
*Note:若Windows服务器访问Ceph FS可用samba做相关映射。*

# Block Device Quick Start
RBD块存储是Ceph提供的3种存储类型中使用最广泛、最稳定的存储类型。RBD块设备类似于磁盘，它可以挂载到物理机或虚拟机中。

部署RBD实例
```
[root@vm-shaltt90 ~]# rbd create test_image --size 1024
```
查看刚创建的块设备
```
[root@vm-shaltt90 ~]# rbd ls
test_image
[root@vm-shaltt90 ~]# rbd info test_image
rbd image 'test_image':
	size 1024 MB in 256 objects
	order 22 (4096 kB objects)
	block_name_prefix: rbd_data.1e4e2238e1f29
	format: 2
	features: layering, exclusive-lock, object-map, fast-diff, deep-flatten
	flags:
```
映射块设备到系统
```
[root@vm-shaltt90 ~]# rbd map --read-only test1
/dev/rbd0
```
创建RBD Pool
```
[root@vm-shaltt90 ~]# rados mkpool pool
successfully created pool pool
[root@vm-shaltt90 ~]# rados lspools
rbd
.rgw.root
default.rgw.control
default.rgw.data.root
default.rgw.gc
default.rgw.log
default.rgw.users.uid
default.rgw.users.keys
default.rgw.buckets.index
default.rgw.buckets.data
metadata
data
pool
```
创建RBD image
```
[root@vm-shaltt90 ~]# rbd create rbd/test3 --size 10G --image-format 2 --image-feature  layering
```
*Note:rbd某些新特性需要高版本内核支持，故这里创建RBD启用一个feature*

# k8s调用Ceph
k8s各个节点安装ceph-common工具
```
yum -y install ceph-common
```
获取ceph sercet
```
[root@vm-shaltt90 ceph]# grep key /etc/ceph/ceph.client.admin.keyring |awk '{printf "%s", $NF}'|base64
QVFDL3NsVlpxMjkzRWhBQXFYUm5ydkg1d3h3NEpnM0hOd1lDVnc9PQ==
```
部署ceph-sercet
```
kubectl create -f ceph-secret.yaml
```
Files：[ceph-secret.yaml](https://github.com/kubernetes/examples/blob/master/staging/volumes/rbd/secret/ceph-secret.yaml)  


部署ceph PV&PVC
```
kubectl create -f ceph-volume.yaml
```
Files：[ceph-volume.yaml](https://github.com/kubernetes/examples/blob/master/staging/volumes/rbd/secret/ceph-secret.yaml)  


参考：https://github.com/kubernetes/examples/blob/master/staging/volumes/rbd/README.md
