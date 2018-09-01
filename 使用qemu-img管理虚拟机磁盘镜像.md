---
title: 使用qemu-img管理虚拟机磁盘镜像
date: 2018-05-24 16:22:27
tags:
---

基于现有的镜像生成新的镜像：
```
qemu-img create -f qcow2 /workspace/vmos/vm/win7.qcow2 -o backing_file=/workspace/vmos/win7.qcow2
```
