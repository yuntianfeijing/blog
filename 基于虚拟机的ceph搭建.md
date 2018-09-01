---
title: 基于虚拟机的ceph搭建
date: 2018-08-31 15:19:50
tags: fs
---

基于KVM虚拟机,使用Centos7.4镜像3节点ceph集群

## 创建虚拟机及准备 ##
使用KVM新建Centos7.4基础镜像　centos7.qcow2
```
qemu-img create -f qcow2 /workspace/ceph/vm/node1.qcow2 -o　backing_file=/workspace/vmos/centos7.qcow2
qemu-img create -f qcow2 /workspace/ceph/vm/node2.qcow2 -o　backing_file=/workspace/vmos/centos7.qcow2
qemu-img create -f qcow2 /workspace/ceph/vm/node3.qcow2 -o　backing_file=/workspace/vmos/centos7.qcow2
```
由于内存有限，只分了3个虚拟机来搭建集群，其中node1因为既是master节点、osd节点，又是网关节点。node2和node3是osd节点和mds节点(用于测试CephFS，不是必须)
node1需要更大内存，我给了1G,其余的512M,网络使用虚拟网络NET（开始使用桥接的时候，总是只有一台虚拟机可以访问外网）
然后给每个虚拟机添加3块10G硬盘用于OSD存储,如下图：
![avatar](https://yuntianfeijing.oss-cn-beijing.aliyuncs.com/blog%2Fpicture%2F2018%2Fnode1-disk.png)

## 防火墙开放端口 ##
```
sudo iptables -A INPUT -i eth0 -p tcp -s 192.168.0.0/16 --dport 6789 -j ACCEPT
sudo iptables -A INPUT -i eth0 -p tcp -s 192.168.0.0/16 --dport 7480 -j ACCEPT
sudo iptables -A INPUT -i eth0 -p tcp -s 192.168.0.0/16 --dport 6800:7300 -j ACCEPT
service iptables save
```
## 配置无密码登录 ##

然后分别在3台虚拟机上修改hostname,
```
hostnamectl set-hostname node1
hostnamectl set-hostname node2
hostnamectl set-hostname node3
```
以下如无特殊说明，操作均需要在3台都来一遍
分别修改/etc/hosts,添加如下：
```
192.168.122.63 node1
192.168.122.66 node2
192.168.122.156 node3
```
生成无密码登录sshkey：
```
ssh-keygen
```
配置无密码登录：
```
ssh-copy-id node1
ssh-copy-id node2
ssh-copy-id node3
```
`sudo sudoers`设置当前用户可以无密码使用sudo权限(root用户忽略，但官方文档不建议用root搭建集群)
```
yourname    ALL=(ALL)       NOPASSWD: ALL
```
若有`Defaults requiretty`行，需注释掉

## 工具安装 ##
操作均需要在3台都来一遍
安装时钟同步工具
```
sudo yum install ntp ntpdate ntp-doc
```
添加ceph源,此处使用163源，创建`sudo vim /etc/yum.repos.d/ceph.repo`
```
[Ceph]
name=Ceph noarch packages
baseurl=http://mirrors.163.com/ceph/rpm-jewel/el7/x86_64
enabled=1
gpgcheck=1
type=rpm-md
gpgkey=http://mirrors.163.com/ceph/keys/release.asc
priority=1

[Ceph-noarch]
name=Ceph noarch packages
baseurl=http://mirrors.163.com/ceph/rpm-jewel/el7/noarch
enabled=1
gpgcheck=1
type=rpm-md
gpgkey=http://mirrors.163.com/ceph/keys/release.asc
priority=1

[ceph-source]
name=Ceph source packages
baseurl=http://mirrors.163.com/ceph/rpm-jewel/el7/SRPMS
enabled=1
gpgcheck=1
type=rpm-md
gpgkey=http://mirrors.163.com/ceph/keys/release.asc
priority=1
```
node1节点安装`ceph-deploy`
```
sudo  yum clean all && yum makecache
yum install -y yum-utils
yum install --nogpgcheck -y epel-release
yum install yum-plugin-priorities
yum install -y ceph-deploy
```
## 集群安装 ##
以下node1节点上操作
用`ceph-deploy`安装集群
```
cd ~
mkdir cluster
cd cluster
ceph-deploy new node1
```
修改`ceph.conf`，添加
```
osd pool default size = 2
```
初始化OSD节点：
```
ceph-deploy install node1 node2 node3
```
添加初始监控节点并收集密钥
```
ceph-deploy mon create-initial
```
硬盘格式化、准备，激活OSD
```
ceph-deploy disk zap node1:sda node1:sdb node1:sdc
ceph-deploy disk zap node2:sda node2:sdb node2:sdc
ceph-deploy disk zap node3:sda node3:sdb node3:sdc

ceph-deploy osd prepare node1:sda node1:sdb node1:sdc
ceph-deploy osd prepare node2:sda node2:sdb node2:sdc
ceph-deploy osd prepare node3:sda node3:sdb node3:sdc

ceph-deploy osd activate node1:sda node1:sdb node1:sdc
ceph-deploy osd activate node2:sda node2:sdb node2:sdc
ceph-deploy osd activate node3:sda1 node3:sdb1 node3:sdc1
```
集群安装成功检查，检查`ceph health`
```
long@node1:$ ceph health
HEALTH_OK
```
## 添加元数据服务器 ##
```
ceph-deploy mds create node2 node3
```

## 网关安装 ##
```
ceph-deploy install --rgw node1
ceph-deploy admin node1
ceph-deploy rgw create node1
```
访问`http://192.168.122.63:7480/`,有返回说明网关正常

添加自启动
>node1:
>systemctl enable ceph-mon.target
>systemctl enable ceph-osd.target
>systemctl enable ceph.target
>systemctl enable ceph-radosgw.target
>node2:
>systemctl enable ceph-osd.target
>systemctl enable ceph-mds.target
>systemctl enable ceph.target
>node3:
>systemctl enable ceph-osd.target
>systemctl enable ceph-mds.target
>systemctl enable ceph.target


参考：
<http://docs.ceph.org.cn>
