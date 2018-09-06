---
title: CRUSH算法
date: 2018-09-06 15:21:23
tags: fs
---

CEPH 的CRUSH算法
CRUSH(Controlled Replication Under Scalable Hashing)是一种基于hash的分布算法，其主要解决两个问题：

1. 如果系统中的存储设备发生变化，如何最小化数据迁移从而使得系统尽快分布均衡
2. 在大型分布式系统中如何分布这些备份从而使得数据具有较高的可靠性

### straw与straw2算法 ###
straw的算法
```
max_x = -1
max_item = -1
for each item:
    x = hash(input,r)
    x *= item_straw
    if x > max_x:
       max_x = x
       max_item = item
return item
```
这个就有问题了scaling factor(比例因子) 是其他iteam的权重所有的，这个就意味着改变A的权重，可能会影响到B和C的权重了

新的straw2的算法是这样的
```
max_x = -1
max_item = -1
for each item:
    x = hash(input,r)
   x = ln(x / 65536) / weight
   if x > max_x:
      max_x = x
      max_item = item
return item
```
可以看到这个是一个weight的简单的函数，这个意味着改变一个item的权重不会影响到其他的项目

### clister map ###
clister map 常用节点（层级）类型

类型ID|类型名称|类型ID|类型名称
:--:|:--:|:--:|:--:
0|osd|6|pod
1|host|7|room
2|chassis|8|datacenter
3|rack|9|regin
4|row|10|root
5|pdu

placement rule（数据分布策略）

1. take
2. select
3. emit
