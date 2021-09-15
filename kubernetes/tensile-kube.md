---

title: virtual kubelet 之 tensile-kube
date: 2021-09-01 13:59:54
tags: k8s
categories:
  - kubernetes
---

### virtual kubelet介绍

##### 啥是virtual kubelet？

- 是 Kubernetes kubelet 的一种实现
- 是微软开源的一个基础库，作为一种虚拟的kubelet用来连接k8s集群和其他平台的API

##### 它有啥用？

允许k8s的节点由其他提供者支持实现。也就是说，我们通过实现节点与k8s的交互，让其他平台或者服务成为k8s的虚拟的节点。这样像kata、OpenStack、serverless平台等服务实现跟k8s的对接，能够被k8s调度。

##### 工作原理

从Kubernetes API服务器的角度来看，Virtual Kubelet看起来像普通的kubelet，但其关键区别在于它们在其他地方调度容器，例如在serverless API中，而不是在节点上。其实现被称为 Provider。开源的Provider有 [Alibaba Cloud ECI Provider](https://github.com/virtual-kubelet/alibabacloud-eci)、 [Azure Container Instances Provider](https://github.com/virtual-kubelet/azure-aci)、 [AWS Fargate Provider](https://github.com/virtual-kubelet/aws-fargate)、[OpenStack Zun](https://github.com/virtual-kubelet/virtual-kubelet#openstack-zun-provider)、 [Tensile Kube](https://github.com/virtual-kubelet/tensile-kube) 等

下图展示了一个Kubernetes集群，其中包含一系列标准kubelet和一个Virtual Kubelet：

![img](https://i.loli.net/2021/09/01/QjADHW1dlRSGqMJ.png)

Virtual Kubelet 的 Provider需要实现上面描述的接口，已实现本地平台或者服务对node及pod的描述转换。



### tensile kube介绍

下面我们以tensile kube为例子，熟悉一下Virtual Kubelet，tensile kube是腾讯游戏Tenc容器团队对外开源的K8s多集群调度方案，其将一个k8s集群作为一个Virtual Node，添加到主k8s集群中，这样当主集群的调度器就可以绑定Pod到Virtual Node（也就是子k8s集群）上。

对公司来说，tensile kube能够提供如下收益：

- 集群碎片资源整合，对于离线集群来说，或多或少存在资源碎片。通过tensile-kube，可以将这写碎片合成一个大的资源池，将闲置的资源充分利用起来。

- 服务多集群调度，结合Affinity等，可以实现Pod的多集群调度。对于1.16及以上集群还可以基于TopologySpreadConstraint实现更细粒度的跨级群调度。
- 服务跨集群通信，tensile-kube原生提供通过Service进行跨集群通信的能力。但是前提是：需要不同集群之间的网络通过Pod IP可以互通(开启ServiceControllers)，如：私有集群共用一个Flannel等。
- 便捷化运维集群，当我们进行单个集群升级、变更时，往往需要通知业务迁移、集群屏蔽调度等，在tensile-kube的支持，我们只需要将virtual-node设置为不可能调度。
- 原生kubectl能力，tensile kubectl支持原生kubectl所有操作，包括：kuebctl logs和kubectl exec。这对于运维人员去管理对于下层集群上的Pod会十分方便，没有额外的学习成本。

其架构图如下：

![image-20210901142819476](https://i.loli.net/2021/09/01/sZV3qzrwT2amNpI.png)

tensile kube组件介绍：

- virtual-node： 基于virtual-kubelet实现的Kubernetes provider功能，同时增加了多个controller，用于同步ConfigMap、Sercret、Service等资源
- webhook： 对Pod中可能对上层集群调度产生干扰的字段进行转换，将其写入annotation中，virtual-node在创建Pod时再将其恢复，避免影响用户期望的调度结果（测试时，可以不部署）
- descheduler：用于避免资源碎片带来的影响，基于社区的descheduler进行的二次开发，使之更适用于这个场景（不考虑重调度或测试时，可以不部署）
- multi-scheduler： 主要用于连接下层集群的API Server，避免资源碎片的影响，在调度时判断virtual-node的资源是否足够。（图中未画出来，multi-scheduler会比较重，不是特别推荐使用；如果要使用，需将上层集群kube-scheduler替换成multi-scheduler）



### 测试 tensile kube

测试时，我们只需要部署一个virtual-node，用于连接上下游k8s集群

集群准备：

```shell
kind create cluster --name tensile-master  --kubeconfig ~/.kube/tensile-master.config
kind create cluster --name tensile-worker  --kubeconfig ~/.kube/tensile-worker.config
alias tenmasterkubectl="kubectl --kubeconfig  ~/.kube/tensile-master.config"
alias tenworkerkubectl="kubectl --kubeconfig  ~/.kube/tensile-worker.config"
```

##### virtual node

```shell
git clone http://ghproxy.com/https://github.com/virtual-kubelet/tensile-kube.git
cd tensile-kube
make provider  #会在bin目录下生成二进制文件virtual-node
```
参数：详见`virtual-node --help`,这里列举几个常用的

```
--kubeconfig #指定主集群的kubeconfig
--client-kubeconfig #指定子集群的kubeconfig
```

主集群中kube-system命名空间下的daemonset `kindnet`与`kube-proxy`添加节点亲和性调度，不要调度到子集群中，子集群中已经启动了相关的组件：

```
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: kubernetes.io/role
                operator: NotIn
                values:
                - agent
```

启动virtual-node

```
./bin/virtual-node --kubeconfig ~/.kube/tensile-master.config  --client-kubeconfig  ~/.kube/tensile-worker.config 
```

产看子集群的节点可也看到新增了名为virtual-kubelet的agent节点：

```
zxl@zxl:~$ tenmasterkubectl get node
NAME                           STATUS   ROLES                  AGE   VERSION
tensile-master-control-plane   Ready    control-plane,master   43m   v1.21.1
virtual-kubelet                Ready    agent                  31m   v1.21.1
```

在主集群中部署一个服务，创建`tensile-nginx.yaml`,并将pod调度到virtual-kubelet节点：

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: karmada-nginx
  labels:
    app: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: kubernetes.io/role
                operator: In
                values:
                - agent
      containers:
      - image: nginx
        name: nginx
      tolerations:
      - key: "virtual-kubelet.io/provider"
        operator: "Exists"
        effect: "NoSchedule"
```

执行`tenmasterkubectl apply -f tensile-nginx.yaml`然后查看：

```
zxl@zxl:~$ tenmasterkubectl get pod -o wide #主集群中看到pod被调度到节点virtual-kubelet
NAME                             READY   STATUS    RESTARTS   AGE   IP           NODE
karmada-nginx-85956db6db-7pmwd   1/1     Running   0          58s   10.244.0.5   virtual-kubelet

zxl@zxl:~$ tenworkerkubectl get pod -o wide #子集群中看到的pod与主集群一致
NAME                             READY   STATUS    RESTARTS   AGE   IP           NODE
karmada-nginx-85956db6db-7pmwd   1/1     Running   0          63s   10.244.0.5   tensile-worker-control-plane
```

Done！
