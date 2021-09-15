---
title: Ceph集群的基本使用
date: 2018-08-31 16:23:29
tags: fs
---

上一节介绍了Ceph集群搭建，现在介绍一下集群的基本使用

## CephFS 挂载使用 ##
元数据服务器安装见上一节`sudo ceph mds stat`查看元数据服务状态
创建两个池放CephFS的元数据和数据及CephFS
```
sudo ceph osd pool create cephfs_data 128
sudo ceph osd pool create cephfs_metadata 128
sudo ceph fs new cephfs cephfs_metadata cephfs_data
```
`sudo ceph fs ls`查看fs

**外部机器挂载cephfs**
外部机器挂载cephfs，用户环境是deepin15.6
```
sudo mount -t ceph 192.168.122.63:6789:/ /mnt/kernel_cephfs -o name=admin,secret=AQDW4XNbgYVrMBAAtw/lqdGbNEQgpcVdTtsEqw==
```
`secret`是node1下的`/etc/ceph/ceph.client.admin.keyring`
卸载`sudo umount /mnt/kernel_cephfs`
**用户空间挂载 CEPH 文件系统 **
用户空间挂载 CEPH 文件系统用户环境是deepin15.6
```
sudo apt-get install ceph-fuse
sudo mkdir -p /etc/ceph
sudo scp {user}@{node1}:/etc/ceph/ceph.conf /etc/ceph/ceph.conf
sudo scp {user}@{node1}:/etc/ceph/ceph.client.admin.keyring /etc/ceph/ceph.client.admin.keyring
sudo ceph-fuse -m 192.168.122.63:6789 /mnt/cephfs
```
卸载`sudo umount /mnt/cephfs`

`ceph fs dump` 查看文件系统的编号
若是`/etc/ceph/ceph.conf`存在，有配置可直接 `sudo ceph-fuse /mnt/cephfs`

## 块设备 ##

创建存储池(若不创建会默认存储池foo):`sudo ceph osd pool create block_data 128`
在block_data池内创建bar块设备: （大小1024M）`sudo rbd create --size 1024 block_data/bar`
查看block_data池内的块设备: `sudo rbd ls block_data`
查看默认池rbd池内的块设备: `sudo rbd ls`
查看块设备信息：`sudo rbd info block_data/bar`
调整块设备大小:
```
sudo rbd resize --size 2048 block_data/bar (to increase)
sudo rbd resize --size 1024 block_data/bar --allow-shrink (to decrease)
```
删除块设备:`rbd rm block_data/bar`
获取block_data池内映像（块设备）列表: `sudo rbd list block_data`
映射块设备：`sudo rbd map {pool-name}/{image-name} --id {user-name}`
如果你启用了 cephx 认证，还必须提供密钥，可以用密钥环或密钥文件指定密钥。
```
sudo rbd {pool-name}/{image-name} --id {user-name} --keyring /path/to/keyring
sudo rbd {pool-name}/{image-name} --id {user-name} --keyfile /path/to/file
```
如：
```
[long@node1 ceph]$ sudo rbd map block_data/bar --id admin --keyring /etc/ceph/ceph.client.admin.keyring
rbd: sysfs write failed
RBD image feature set mismatch. You can disable features unsupported by the kernel with "rbd feature disable".
In some cases useful info is found in syslog - try "dmesg | tail" or so.
rbd: map failed: (6) No such device or address
```
查看dmesg
```
[78542.446213] libceph: client4228 fsid d595ca23-037f-41f4-b1d6-409958e0b690
[78542.448928] rbd: image rbd/docker_test does not exist
[78572.308410] libceph: no secret set (for auth_x protocol)
[78572.308413] libceph: error -22 on auth protocol 2 init
[78592.923819] libceph: mon0 192.168.122.63:6789 session established
[78592.924456] libceph: client4231 fsid d595ca23-037f-41f4-b1d6-409958e0b690
[78592.941463] rbd: image bar: image uses unsupported features: 0x38
[78597.310424] libceph: mon0 192.168.122.63:6789 session established
[78597.310532] libceph: client4233 fsid d595ca23-037f-41f4-b1d6-409958e0b690
[78597.318362] rbd: image bar: image uses unsupported features: 0x38
```
查看该镜像支持的特性:
```
[long@node1 ceph]$ sudo rbd info block_data/bar
rbd image 'bar':
	size 1024 MB in 256 objects
	order 22 (4096 kB objects)
	block_name_prefix: rbd_data.10736b8b4567
	format: 2
	features: layering, exclusive-lock, object-map, fast-diff, deep-flatten
	flags:
```
特性说明：
layering: 支持分层
striping: 支持条带化 v2
exclusive-lock: 支持独占锁
object-map: 支持对象映射（依赖 exclusive-lock ）
fast-diff: 快速计算差异（依赖 object-map ）
deep-flatten: 支持快照扁平化操作
journaling: 支持记录 IO 操作（依赖独占锁）

查看本机内核版本是3.10.0-862.9.1.el7.x86_64
经测试，内核版本 3.10，仅支持此特性（layering），其它特性需要使用更高版本内核，或者从新编译内核加载特性模块才行。

有几种解决方法：
1、disable不支持的feature　`sudo rbd  feature disable block_data/bar  exclusive-lock object-map fast-diff deep-flatten`
2、创建 --image-feature选项指定使用特性，不用全部开启。我们的需求仅需要使用快照等特性，开启layering即可
```
sudo rbd create block_data/bar --size 1024 --image-format 2 --image-feature  layering
```
3、创建rbd镜像命令的主机中，修改Ceph配置文件/etc/ceph/ceph.conf，在global section下，增加：`rbd_default_features = 1`
成功：
```
[long@node1 ceph]$ sudo rbd map block_data/bar --id admin --keyring /etc/ceph/ceph.client.admin.keyring
/dev/rbd0
[long@node1 ceph]$ ls /dev/rbd0
/dev/rbd0
```
然后你可以像操作普通硬盘 一样操作它，fdisk 格式化，挂载...
取消映射 `sudo rbd unmap /dev/rbd/{poolname}/{imagename}`

参看：<https://www.cnblogs.com/sisimi/p/7761179.html>

## 网关（对象存储） ##
网关安装见上一节，对于 Ceph 对象网关，在生产环境下你需要开起 Civetweb 所使用的端口。Civetweb默认运行在 7480 端口上。

`ceph auth get client.rgw.node1 >> /etc/ceph/ceph.client.radosgw.keyring`
`sudo vi /etc/ceph/ceph.conf` 添加如下
```
[client.rgw.node1]
rgw_frontends = "civetweb port=7480"
host = node1
keyring = /etc/ceph/ceph.client.radosgw.keyring
rgw socket path = /var/run/ceph/ceph-client.node1.asok
rgw content length compat = true
```

创建radosgw用户进行访问
```
[long@node1 ceph]$ radosgw-admin user create --uid="admin" --display-name="admin"
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
            "access_key": "W3W7DQADM05MG6BSJ9GB",
            "secret_key": "rfOOJQbwLEKYBTBRO3yWmXkMAGBgKSqkDVX8l4ka"
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
