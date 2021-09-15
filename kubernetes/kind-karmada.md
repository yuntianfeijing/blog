---
title: 使用kind测试karmada
date: 2021-08-30 14:59:46
tags: k8s
categories:
  - kubernetes
---

Karmada（Kubernetes Armada）是一个华为开源的 Kubernetes 管理系统，基于 Kubernetes Federation v1 和 v2 开发，它可以跨多个 Kubernetes 集群和云运行云原生应用程序，而无需对应用程序进行更改。通过直接使用 Kubernetes 原生 API 并提供高级调度功能。Karmada通过PropagationPolicy，Binding与Work的抽象，可以将任意资源同步到匹配的cluster，并通过Policy进行差异化的配置，并进行重载，通过ReplicaSchedulingPolicy配置集群权重，配合hpa，使应用可以在多个集群间伸缩。

### Karmada是怎么工作的

#### Karmada架构图

![image-20210830150754901](https://i.loli.net/2021/08/30/p9W58DVXv2lHcqs.png)



| 组件                       | 原生    | 说明                                                         |
| :------------------------- | :------ | :----------------------------------------------------------- |
| etcd                       | 是      | ETCD存储karmada API对象                                      |
| kube-apiserver             | 是      | API Server是所有其他组件都可以与之通信的REST端点             |
| kube-controller-manager    | 是      | Controller Manager将根据您通过API服务器创建的API对象执行操作 |
| karmada-controller-manager | karmada | karmada controller-manager, 监听相关资源，创建ResourceBinding及ClusterResourceBinding，更新work，同步集群间资源 |
| karmada-scheduler          | karmada | 加载Policy，分析资源，更新work                               |
| karmada-webhook            | karmada | karmada资源校验                                              |
| karmada-agent              | karmada | 取决于Cluster的同步模式，若是push模式，则不需要部署，否则需要部署到管理的集群中 |

Karmada控制器管理器运行各种控制器，这些控制器监视karmada对象，然后与基础集群的API服务器对话以创建常规的Kubernetes资源。

（1）Cluster Controller：将Kubernetes集群附加到Karmada，以通过创建集群对象来管理集群的生命周期。

（2）Policy Controller：控制器监视PropagationPolicy对象。添加PropagationPolicy对象时，它将选择与resourceSelector匹配的一组资源，并为每个资源对象创建ResourceBinding。

（3）Binding Controller：控制器监视ResourceBinding对象，并使用单个资源清单创建与每个集群相对应的Work对象。

（4）Execution Controller：控制器监视工作对象，当创建工作对象时，它将资源分配给成员集群。

#### Karmada工作流

![image-20210830150944623](https://i.loli.net/2021/08/30/JT8gZEawFW2Pf4V.png)

**Resource template**：Karmada使用Kubernetes自己的API定义的Resource template，以使其易于与Kubernetes上已采用的现有工具集成

**Propagation Policy**:：Karmada提供了Propagation(placement) Policy API，以定义多集群调度和传播需求。

- 支持策略1：n映射：工作负载，用户无需在每次创建联合应用程序时都指出调度约束
- 使用默认策略，用户可以仅与K8s API进行交互

**Override Policy**：Karmada提供独立的Override Policy API，用于专门化与群集相关的配置自动化。例如：

- 根据成员群集区域覆盖镜像
- 根据云提供商覆盖StorageClass

##### 启动流程

创建一个Deployment (kubectl)→ PropagationPolicy(kubectl) → ResourceBinding(controller) → Work(按照集群区分) → 同步资源
创建一个ClusterRole(kubectl) → ClusterPropagationPolicy(kubectl) → ClusterResourceBinding(controller) → Work(按照集群区分) → 同步资源

1、用户通过kukectl或者api 提交一个资源(cluster resource or namespace resource)，此时，资源只存在于karmada的etcd中

2、用户通过kukectl或者api 创建PropagationPolicy（或ClusterPropagationPolicy），karmada-controller-manager会监听PropagationPolicy的创建事件，创建ResourceBinding(ClusterResourceBinding)，然后创建work，

karmada-controller-manager会监听各个集群的资源，更新work及work status

3、karmada-scheduler监听资源、Policy、Binding、Work等信息，更新Work

### karmada部署

##### 安装karmadactl

##### 部署控制端

```shell
git clone https://github.com/karmada-io/karmada.git
cd karmada
hack/local-up-karmada.s
cp karmadactl /some-dir-in-your-PATH/karmadactl
```

该脚本`hack/local-up-karmada.sh`将为您执行以下任务：

- 启动Kubernetes集群`karmada-host`以运行karmada控制平面
- 根据当前代码库构建karmada控制平面组件
- 在集群`karmada-host`上部署karmada控制平面组件,包含etcd、kube-apiserver、kube-controller-manager等k8s组件及自身组件karmada-controller-manager、karmada-scheduler、karmada-webhook

此脚本会在kind启动的Kubernetes集群中部署一整套karmada的依赖，包括etcd，apiserver，

这样，karmada就部署完成了

```shell
kubectl --kubeconfig  ~/.kube/karmada.config config use-context karmada-host
cp ~/.kube/karmada.config ~/.kube/karmada-host.config
kubectl --kubeconfig  ~/.kube/karmada.config config use-context karmada-apiserver
alias kmasterkubectl="kubectl --kubeconfig  ~/.kube/karmada.config"
alias khostkubectl="kubectl --kubeconfig  ~/.kube/karmada-host.config"
```

分别查看如下：

```shell
zxl@zxl:~$ khostkubectl get pod -A  #kind 启动的k8s集群
NAMESPACE            NAME                                                 READY   STATUS    RESTARTS   AGE
karmada-system       etcd-0                                               1/1     Running   0          18h
karmada-system       karmada-apiserver-64f969fd6f-mddpp                   1/1     Running   0          18h
karmada-system       karmada-controller-manager-7d66968445-ghfht          1/1     Running   0          17h
karmada-system       karmada-kube-controller-manager-847d88c674-q4lgf     1/1     Running   0          18h
karmada-system       karmada-scheduler-7c8d678979-9kgfv                   1/1     Running   0          17h
karmada-system       karmada-webhook-5bfd9fb89d-4mc9s                     1/1     Running   0          17h
kube-system          coredns-f9fd979d6-ftv7c                              1/1     Running   0          18h
kube-system          coredns-f9fd979d6-rp4lh                              1/1     Running   0          18h
kube-system          etcd-karmada-host-control-plane                      1/1     Running   0          18h
kube-system          kindnet-ztpcg                                        1/1     Running   0          18h
kube-system          kube-apiserver-karmada-host-control-plane            1/1     Running   0          18h
kube-system          kube-controller-manager-karmada-host-control-plane   1/1     Running   0          18h
kube-system          kube-proxy-6rnbs                                     1/1     Running   0          18h
kube-system          kube-scheduler-karmada-host-control-plane            1/1     Running   0          18h
local-path-storage   local-path-provisioner-78776bfc44-mzrk4              1/1     Running   0          18h

zxl@zxl:~$ kmasterkubectl get crd  #k8s集群中启动的一套类似k8s的karmada集群,kube-on-kube
NAME                                           CREATED AT
clusteroverridepolicies.policy.karmada.io      2021-08-30T07:59:48Z
clusterpropagationpolicies.policy.karmada.io   2021-08-30T07:59:47Z
clusterresourcebindings.work.karmada.io        2021-08-30T07:59:50Z
clusters.cluster.karmada.io                    2021-08-30T07:59:45Z
overridepolicies.policy.karmada.io             2021-08-30T07:59:47Z
propagationpolicies.policy.karmada.io          2021-08-30T07:59:46Z
replicaschedulingpolicies.policy.karmada.io    2021-08-30T07:59:49Z
resourcebindings.work.karmada.io               2021-08-30T07:59:49Z
serviceexports.multicluster.x-k8s.io           2021-08-30T07:59:51Z
serviceimports.multicluster.x-k8s.io           2021-08-30T07:59:51Z
works.work.karmada.io                          2021-08-30T07:59:49Z
```

##### CRD介绍

| CRD                      | scope | describe                                                     |
| :----------------------- | :---- | :----------------------------------------------------------- |
| Cluster                  | false | 录入的集群信息 <br />包含：集群APIEndpoint <br />SecretRef（for kubeconfig） <br />ClusterSyncMode: push,pull <br />集群位置信息(Provider,Region,Zone) 污点 Taints <br />集群状态:APIEnablements(GVR),NodeSummary等 |
| ClusterOverridePolicy    | false | 资源匹配：ResourceSelectors <br />集群匹配： TargetCluster<br />重载部分：<br />PlaintextOverrider：Path，Operator，Value<br />ImageOverrider：Predicate，Component(registry,repository,tag)，Operator,Value |
| ClusterPropagationPolicy | false | 资源选择：ResourceSelectors<br />相关资源是否被选中：Association<br />集群匹配： Placement（ClusterAffinity，ClusterTolerations，SpreadConstraints）<br />重载策略列表： DependentOverrides`调度器选择： SchedulerName |
| ClusterResourceBinding   | false | 资源：Resource 集群绑定：Clusters                            |
| OverridePolicy           | true  | 同ClusterOverridePolicy                                      |
| PropagationPolicy        | true  | 同ClusterPropagationPolicy                                   |
| ResourceBinding          | true  | 同ClusterResourceBinding                                     |
| ReplicaSchedulingPolicy  | true  | PropagationPolicy的总副本数 <br />资源选择:ResourceSelectors<br />总副本数：TotalReplicas<br />集群偏好：ClusterPreferences（ClusterAffinity，Weight） |
| Work                     | true  | WorkSpec：Workload资源<br />WorkStatus：Conditions，ManifestStatuses |

##### 创建子集群

```shell
kind create cluster --name karmada-worker01
kind create cluster --name karmada-worker02
```

因为kind未放开apiserver的外部访问，所以按照如下步骤执行，替换kubeconfig中的, 新建脚本`change_server.sh`如下:

```sh
#!/usr/bin/env bash

function usage() {
  echo "This script change kind k8s server by container ip and port."
  echo "Example: change_server.sh fed-worker01"
}
FED_CLUSTER_NAME=$1
if [[ -z "${FED_CLUSTER_NAME}" ]]; then
  usage
  exit 1
fi

container_ip=$(docker inspect --format='{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' "${FED_CLUSTER_NAME}-control-plane")
kubectl config set-cluster "kind-${FED_CLUSTER_NAME}" --server="https://${container_ip}:6443"
```

执行脚本，替换kubeconfig中的server

```shell
./change_server.sh karmada-worker01
./change_server.sh karmada-worker02
```

为方便访问查看，备份子集群的配置文件

```shell
kubectl config use-context kind-karmada-worker01  #将kubectl的context切换到karmada-worker01集群
cp ~/.kube/config ~/.kube/karmada-worker01-config
kubectl config use-context kind-karmada-worker02  #将kubectl的context切换到karmada-worker02集群
cp ~/.kube/config ~/.kube/karmada-worker02-config
alias k01kubectl="kubectl --kubeconfig  ~/.kube/karmada-worker01-config"
alias k02kubectl="kubectl --kubeconfig  ~/.kube/karmada-worker02-config"
```

##### 添加子集群

添加子集群的方法有两种，这取决于karmada主从之间的同步方式是pull还是push，

- push模式： 由karamda控制端监听各个子集群状态，并将组合的资源分发到各个子集群
- pull模式： 在子集群部署karmada-agent,用于监听与本集群相关的资源，并维护资源的生命周期

Push模式部署：

```
karmadactl join karmada-worker01  --kubeconfig ~/.kube/karmada.config  --cluster-kubeconfig ~/.kube/karmada-worker01-config --cluster-context kind-karmada-worker01

karmadactl join karmada-worker02  --kubeconfig ~/.kube/karmada.config  --cluster-kubeconfig ~/.kube/karmada-worker02-config --cluster-context kind-karmada-worker02
```

Pull模式部署：

```
hack/deploy-karmada-agent.sh ~/.kube/karmada.config karmada-apiserver ~/.kube/karmada-worker01-config kind-karmada-worker01

hack/deploy-karmada-agent.sh ~/.kube/karmada.config karmada-apiserver ~/.kube/karmada-worker02-config kind-karmada-worker02
```

此处我们采用的是Push模式，完成后查看如下

```
zxl@zxl:~$ kmasterkubectl get cluster
NAME               VERSION   MODE   READY   AGE
karmada-worker01   v1.21.1   Push   True    14m
karmada-worker02   v1.21.1   Push   True    14m
```

### 测试 

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: karmada-nginx
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx
        name: nginx
---
apiVersion: policy.karmada.io/v1alpha1
kind: OverridePolicy
metadata:
  name: nginx-propagation-worker01
spec:
  resourceSelectors:
    - apiVersion: apps/v1
      kind: Deployment
      name: karmada-nginx
      #labelSelector: *metav1.LabelSelector
  targetCluster:  #*ClusterAffinity 支持 labelSelector，fieldSelector，clusterNames，exclude等方式
    clusterNames:
      - karmada-worker01
  overriders:
    plaintext: #[]PlaintextOverrider
      - path: "/metadata/annotations"
        operator: "add"
        value: {"test":"this_is_worker01"}
    #imageOverrider:
    #commandOverrider:
    #argsOverrider:
---
apiVersion: policy.karmada.io/v1alpha1
kind: OverridePolicy
metadata:
  name: nginx-propagation-worker02
spec:
  resourceSelectors:
    - apiVersion: apps/v1
      kind: Deployment
      name: karmada-nginx
      #labelSelector: *metav1.LabelSelector
  targetCluster:  #*ClusterAffinity 支持 labelSelector，fieldSelector，clusterNames，exclude等方式
    clusterNames:
      - karmada-worker02
  overriders:
    plaintext: #[]PlaintextOverrider
      - path: "/metadata/annotations"
        operator: "add"
        value: {"test":"this_is_worker02"}
    imageOverrider:
     - predicate:
         path: "/spec/template/spec/containers/0/image"
       component: "Tag" # Registry;Repository or Tag
       operator: "add"
       value: "1.17.0-alpine"
---
apiVersion: policy.karmada.io/v1alpha1
kind: PropagationPolicy
metadata:
  name: nginx-propagation
spec:
  resourceSelectors:
    - apiVersion: apps/v1
      kind: Deployment
      name: karmada-nginx
      #labelSelector: *metav1.LabelSelector
  #association: bool #相关资源也需要同步，如挂载的configmap
  placement:
    clusterAffinity: #支持 labelSelector，fieldSelector，clusterNames，exclude等方式
      clusterNames:
      - karmada-worker01
      - karmada-worker02
    replicaScheduling: #带有副本数的资源的分发策略
        replicaSchedulingType: Divided # or Duplicated,重复或者分发
        replicaDivisionPreference: Weighted #or Aggregated， 按照权重或者聚合(尽量平铺)
        weightPreference:
          staticWeightList:
          - targetCluster: # clusterAffinity
              clusterNames:
                - karmada-worker01
            weight: 1
          - targetCluster: # clusterAffinity
              clusterNames:
                - karmada-worker02
            weight: 2
  dependentOverrides: #[]string 相关的重载策略
    - nginx-propagation-worker01
    - nginx-propagation-worker02
    #spreadConstraints: []SpreadConstraint 广播策略，根据Field(cluster;region;zone;provider) 或Label，区分选中的子集群，并限制可选中子集群的最大最小数目
    #clusterTolerations: []corev1.Toleration 容忍有污点的集群
  #schedulerName：string #使用的调度器名字
```

查看部署情况：

```
zxl@zxl:~$ kmasterkubectl get deploy
NAME            READY   UP-TO-DATE   AVAILABLE   AGE
karmada-nginx   3/3     3            3           3m9s

zxl@zxl:~$ kmasterkubectl get resourcebinding
NAME                       AGE
karmada-nginx-deployment   3m13s

zxl@zxl:~$ kmasterkubectl get work -A
NAMESPACE                     NAME                      AGE
karmada-es-karmada-worker01   karmada-nginx-d5fbb6dc5   3m15s
karmada-es-karmada-worker02   karmada-nginx-d5fbb6dc5   3m15s

zxl@zxl:~$ k01kubectl get deploy
NAME            READY   UP-TO-DATE   AVAILABLE   AGE
karmada-nginx   1/1     1            1           3m27s

zxl@zxl:~$ k02kubectl get deploy
NAME            READY   UP-TO-DATE   AVAILABLE   AGE
karmada-nginx   2/2     2            2           3m31s
```

Done!
