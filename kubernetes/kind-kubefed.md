---
title: 使用kind测试集群联邦
date: 2021-08-27 14:53:21
tags: k8s
categories:
 - kubernetes
---



Kind 是 Kubernetes In Docker 的缩写，顾名思义是使用 Docker 容器作为 Node 并将 Kubernetes 部署至其中的一个工具。官方文档中也把 Kind 作为一种本地集群搭建的工具进行推荐。

### 安装二进制文件

#### 安装kind

```shell
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.11.1/kind-linux-amd64
chmod +x ./kind
mv ./kind /some-dir-in-your-PATH/kind
```

#### 安装helm

参考 https://helm.sh/docs/intro/install/

```shell
wget https://get.helm.sh/helm-v3.6.3-linux-amd64.tar.gz
tar -xvzf helm-v3.6.3-linux-amd64.tar.gz
mv linux-amd64/helm /some-dir-in-your-PATH/helm
rm -rf linux-amd64
rm helm-v3.6.3-linux-amd64.tar.gz
```

#### 安装kubectl

```shell
curl -LO https://storage.googleapis.com/kubernetes-release/release/v1.21.2/bin/linux/amd64/kubectl
chmod +x ./kubectl
mv ./kubectl /some-dir-in-your-PATH/kubectl
```



#### 安装kubefedctl

```shell
wget https://ghproxy.com/https://github.com/kubernetes-sigs/kubefed/releases/download/v0.8.1/kubefedctl-0.8.1-linux-amd64.tgz
tar -xvzf kubefedctl-0.8.1-linux-amd64.tgz
mv kubefedctl /some-dir-in-your-PATH/kubefedctl
```



### kind配置与介绍

使用 kind 创建 Kubernetes 集群非常的方便，只需要一行命令即可

```shell
kind create cluster
```

删除集群

```shell
kind delete cluster
```

以上操作的集群的默认名称是kind，可以使用`--name`参数指定集群名，更多创建的参数如下：

```shell
zxl@zxl:~$ kind create cluster  --help
Creates a local Kubernetes cluster using Docker container 'nodes'

Usage:
  kind create cluster [flags]

Flags:
      --config string       path to a kind config file
  -h, --help                help for cluster
      --image string        node docker image to use for booting the cluster
      --kubeconfig string   sets kubeconfig path instead of $KUBECONFIG or $HOME/.kube/config
      --name string         cluster name, overrides KIND_CLUSTER_NAME, config (default kind)
      --retain              retain nodes for debugging when cluster creation fails
      --wait duration       wait for control plane node to be ready (default 0s)

Global Flags:
      --loglevel string   DEPRECATED: see -v instead
  -q, --quiet             silence all stderr output
  -v, --verbosity int32   info log verbosity
```

`--kubeconfig`指定生成kubeconfig文件的路径

`--name`  指定集群的名字，当我们有多个k8s集群时，这很有用

`--config`用来指定启动的k8s集群的额外配置文件，必须是一个符合`apiVersion: kind.x-k8s.io/v1alpha4` 且`  kind: Cluster`描述的一个yaml文件,如下：

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: cluster-name  #集群名
nodes: #节点配置，如下，启动一个master节点，一个worker节点
- role: control-plane   #master节点
  image: kindest/node:v1.21.1 #指定镜像，同 kind create cluster参数--image
  extraPortMappings:
  - containerPort: 6443
    hostPort: 45876
    listenAddress: "0.0.0.0"
    protocol: tcp #默认值，可不设置
- role: worker #
```



### kind创建联邦集群

##### 创建主集群

先创建一个k8s集群，作为master用来管理其他集群

```shell
kind create cluster --name fed-master
```

使用命令`docker`及`kubectl`命令查看如下：

```shell
zxl@zxl:~$ docker ps
CONTAINER ID   IMAGE                  COMMAND                  CREATED          STATUS          PORTS                       NAMES
853e3b89eddb   kindest/node:v1.21.1   "/usr/local/bin/entr…"   39 minutes ago   Up 39 minutes   127.0.0.1:45875->6443/tcp   fed-master-control-plane

kind@kind:~$ kubectl get node
NAME                       STATUS   ROLES                  AGE   VERSION
fed-master-control-plane   Ready    control-plane,master   40m   v1.21.1

kind@kind:~$ kubectl get pod -A
NAMESPACE            NAME                                               READY   STATUS    RESTARTS   AGE
kube-system          coredns-558bd4d5db-mqs7s                           1/1     Running   0          39m
kube-system          coredns-558bd4d5db-szvnc                           1/1     Running   0          39m
kube-system          etcd-fed-master-control-plane                      1/1     Running   0          39m
kube-system          kindnet-s4dxh                                      1/1     Running   0          39m
kube-system          kube-apiserver-fed-master-control-plane            1/1     Running   0          39m
kube-system          kube-controller-manager-fed-master-control-plane   1/1     Running   0          39m
kube-system          kube-proxy-hcj6h                                   1/1     Running   0          39m
kube-system          kube-scheduler-fed-master-control-plane            1/1     Running   0          40m
local-path-storage   local-path-provisioner-547f784dff-8qjgv            1/1     Running   0          39m
```



##### 创建子集群

接下来我们需要创建两个子集群，用于创建联邦时使用，且子集群的kube-apiserver端口需要暴露供fed-master访问

```shell
kind create cluster --name fed-worker01
kind create cluster --name fed-worker02
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
chmod +x change_server.sh
./change_server.sh fed-worker01
./change_server.sh fed-worker02
```

为方便访问查看，备份子集群的配置文件

```shell
kubectl config use-context kind-fed-worker01  #将kubectl的context切换到fed-worker01集群
cp ~/.kube/config ~/.kube/fed-worker01-config
kubectl config use-context kind-fed-worker02  #将kubectl的context切换到fed-worker02集群
cp ~/.kube/config ~/.kube/fed-worker02-config
alias fed01kubectl="kubectl --kubeconfig  ~/.kube/fed-worker01-config"
alias fed02kubectl="kubectl --kubeconfig  ~/.kube/fed-worker02-config"
kubectl config use-context kind-fed-master  #将kubectl的context切换到主集群
```

查看集群：

```shell
zxl@zxl:~$ kind get clusters
fed-master
fed-worker01
fed-worker02

zxl@zxl:~$ kubectl config use-context kind-fed-master
zxl@zxl:~$ kubectl get node
NAME                       STATUS   ROLES                  AGE    VERSION
fed-master-control-plane   Ready    control-plane,master   161m   v1.21.1

zxl@zxl:~$ fed01kubectl get node
NAME                         STATUS   ROLES                  AGE   VERSION
fed-worker01-control-plane   Ready    control-plane,master   38m   v1.21.1

zxl@zxl:~$ fed02kubectl get node
NAME                         STATUS   ROLES                  AGE     VERSION
fed-worker02-control-plane   Ready    control-plane,master   5m59s   v1.21.1
```

至次，集群准备完成



### kubefed安装测试

使用helm安装kubefed到主集群

```
helm repo add kubefed-charts https://raw.githubusercontent.com/kubernetes-sigs/kubefed/master/charts
helm --namespace kube-federation-system upgrade -i kubefed kubefed-charts/kubefed --version=v0.8.1 --create-namespace
```

查看安装的资源

```shell
zxl@zxl:~$ kubectl -n kube-federation-system get pod
NAME                                          READY   STATUS    RESTARTS   AGE
kubefed-admission-webhook-5c96c86688-hkgbf    1/1     Running   0          4m1s
kubefed-controller-manager-74b97866c4-6g895   1/1     Running   0          2m49s
kubefed-controller-manager-74b97866c4-v784c   1/1     Running   0          2m46s

zxl@zxl:~$ kubectl get crd
NAME                                                 CREATED AT
clusterpropagatedversions.core.kubefed.io            2021-08-25T10:13:13Z
federatedclusterroles.types.kubefed.io               2021-08-25T10:13:12Z
federatedconfigmaps.types.kubefed.io                 2021-08-25T10:13:12Z
federateddeployments.types.kubefed.io                2021-08-25T10:13:12Z
federatedingresses.types.kubefed.io                  2021-08-25T10:13:12Z
federatedjobs.types.kubefed.io                       2021-08-25T10:13:12Z
federatednamespaces.types.kubefed.io                 2021-08-25T10:13:12Z
federatedreplicasets.types.kubefed.io                2021-08-25T10:13:12Z
federatedsecrets.types.kubefed.io                    2021-08-25T10:13:12Z
federatedserviceaccounts.types.kubefed.io            2021-08-25T10:13:12Z
federatedservices.types.kubefed.io                   2021-08-25T10:13:12Z
federatedservicestatuses.core.kubefed.io             2021-08-25T10:13:13Z
federatedtypeconfigs.core.kubefed.io                 2021-08-25T10:13:13Z
kubefedclusters.core.kubefed.io                      2021-08-25T10:13:13Z
kubefedconfigs.core.kubefed.io                       2021-08-25T10:13:13Z
propagatedversions.core.kubefed.io                   2021-08-25T10:13:13Z
replicaschedulingpreferences.scheduling.kubefed.io   2021-08-25T10:13:13Z
```

添加子集群到主集群kubefed的管控下：

```shell
kubefedctl join fed-worker01 --cluster-context kind-fed-worker01 --host-cluster-context kind-fed-master -v=2
kubefedctl join fed-worker02 --cluster-context kind-fed-worker02 --host-cluster-context kind-fed-master -v=2
```

查看添加的集群

```
zxl@zxl:~$ kubectl get kubefedclusters -A
NAMESPACE                NAME           AGE   READY
kube-federation-system   fed-worker01   97s   True
kube-federation-system   fed-worker02   46s   True
```



##### kubefed架构介绍

如上，我们看到，在主集群部署Kubefed Controller Manger与Kubefed  AdmissionWebhook，并创建了一系列的CRD，这种方式就是我们常说的Operator。

其架构如下：
![](https://img-blog.csdnimg.cn/bb6ead19d6c94ad89e0530bcd46792dd.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBA5LqR5aSp6aOe6ZWc,size_20,color_FFFFFF,t_70,g_se,x_16#pic_center)

- 主集群kube-apiserver监听来自api、kubectl、kubefedctl的请求、创建管理CRD资源
- Kubefed  AdmissionWebhook校验资源的有效性
- Kubefed Controller Manger监听资源、组合数据，分发资源到各个子集群，并监听子集群资源状态，维护其生命周期


下图是kubefed的工作流程图：
![](https://img-blog.csdnimg.cn/img_convert/b09252c79f358f73b86657a48b101509.png)

- Type配置：声明Kubefed需要处理的API类型
- Cluster配置：声明Kubefed需要管理的目标集群
- Propagation: 表示将资源分发到对应集群的版本信息等。
- Templates: 定义资源在所有集群中共同的部分
- Placement: 定义资源应该出现在哪些集群中
- Overrides: 定义集群对Template字段的覆盖
- Status: 收集被分发到所有集群的资源的状态
- Policy: 决定一个资源应当被分发到联邦集群的什么子集中
- Scheduling: 决定工作负载如何跨越不同集群分布

每一种联邦资源，都包含三个部分，分别是Template、Placement，Overrides；Kubefed Controller Manger根据这些信息，组装出分发往各个子集群的资源，并监听其状态，维护其生命周期

##### kubefed 相关CRD

- Core CRD
  - kubefedconfigs ：控制器的配置项，包含作用域、控制器选主配置、子集群健康检查、特性门控开关、SyncController、StatusController等
  - kubefedclusters 声明Kubefed需要管理的目标集群,存储集群信息(Host、CA、Secret)
  - federatedtypeconfigs （关联federatedType及targetType，控制能否传播）
  - clusterpropagatedversions (记录集群资源的版本信息)
  - propagatedversions (记录namespace资源的版本信息)
- Types CRD
  [federatedxxxx.types.kubefed.io](http://federatedxxxx.types.kubefed.io) (声明Kubefed需要处理的API类型)
- Schedulin CRD
  replicaschedulingpreferences (提供了一种机制，来自动化的机制来维护所有集群成员中Deployment/Replicaset类型工作负载的副本总数)

```shell
zxl@zxl:~$ kubectl -n kube-federation-system get federatedtypeconfigs.core.kubefed.io
NAME                                     AGE
clusterroles.rbac.authorization.k8s.io   23h
configmaps                               23h
deployments.apps                         23h
ingresses.extensions                     23h
jobs.batch                               23h
namespaces                               23h
replicasets.apps                         23h
secrets                                  23h
serviceaccounts                          23h
services                                 23h
```



##### 测试集群联邦

创建ns并联邦

```shell
kubefedctl enable statefulsets  #创建默认不启用的federatedtypeconfigs
kubefedctl enable customresourcedefinitions #创建默认不启用的federatedtypeconfigs
#创建ns并联邦
kubectl create ns ddcloud
kubefedctl federate namespace ddcloud #会联邦namespace下的资源 使用--contents --skip-api-resources "configmaps,apps" 忽略指定的资源
#查看联邦的federatednamespace
```

查看新启用的资源及联邦资源

```
zxl@zxl:~$ kubectl -n kube-federation-system get federatedtypeconfigs #可以看到新启用了customresourcedefinitions与statefulsets
NAME                                             AGE
clusterroles.rbac.authorization.k8s.io           39h
configmaps                                       39h
customresourcedefinitions.apiextensions.k8s.io   19s
deployments.apps                                 39h
ingresses.extensions                             39h
jobs.batch                                       39h
namespaces                                       39h
replicasets.apps                                 39h
secrets                                          39h
serviceaccounts                                  39h
services                                         39h
statefulsets.apps                                28s
zxl@zxl:~$ kubectl get federatednamespace -A
NAMESPACE   NAME      AGE
ddcloud     ddcloud   11s

#相应的，我们在子集群看到对应的ns已经创建
zxl@zxl:~$ fed01kubectl get ns ddcloud
NAME      STATUS   AGE
ddcloud   Active   2m35s
zxl@zxl:~$ fed02kubectl get ns ddcloud
NAME      STATUS   AGE
ddcloud   Active   2m40s
```



##### 测试应用的部署与差异化配置

创建一个联邦的Deployment，`fed-nginx.yaml`如下：

```yaml
apiVersion: types.kubefed.io/v1beta1
kind: FederatedDeployment
metadata:
  name: fed-nginx
  namespace: ddcloud
spec:
  template:
    metadata:
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
          containers:
          - image: nginx
            name: nginx
  placement:  #选择应用可以部署的集群
    clusters: # or clusterSelector
    - name: fed-worker01
    - name: fed-worker02
  overrides:  #配置各个集群部署的deployment的差异化配置
  - clusterName: fed-worker01
    clusterOverrides:
    - path: "/spec/replicas"
      value: 2
    - path: "/spec/template/spec/containers/0/image"
      value: "nginx:1.17.0-alpine"
    - path: "/metadata/annotations"
      op: "add"
      value:
        foo: this_is_worker01
  - clusterName: fed-worker02
    clusterOverrides:
    - path: "/metadata/annotations"
      op: "add"
      value:
        foo: this_is_worker02
```

部署

```
kubectl apply -f fed-nginx.yaml #部署FederatedDeployment
```

查看：

```
zxl@zxl:~$ fed01ubectl -n ddcloud get deploy #查看子集群fed-worker01中的deploy
NAME        READY   UP-TO-DATE   AVAILABLE   AGE
fed-nginx   1/1     1            1           2m17s
zxl@zxl:~$ fed01kubectl -n ddcloud get pod #查看子集群fed-worker02的pod
NAME                         READY   STATUS    RESTARTS   AGE
fed-nginx-8656cd9f7f-4t2zt   1/1     Running   0          48s
fed-nginx-8656cd9f7f-dww8t   1/1     Running   0          48s

zxl@zxl:~$ fed02ubectl -n ddcloud get deploy #查看子集群fed-worker02的deploy
NAME        READY   UP-TO-DATE   AVAILABLE   AGE
fed-nginx   2/2     2            2           2m59s
zxl@zxl:~$ fed02kubectl -n ddcloud get pod
NAME                         READY   STATUS    RESTARTS   AGE
fed-nginx-6799fc88d8-st9dq   1/1     Running   0          55s
```

注解及镜像差异请自行查看Pod



##### ReplicaSchedulingPreference

控制Deployment或ReplicaSet副本数在各个集群中的权重、分布等

```yaml
# 平均分配权重
apiVersion: scheduling.kubefed.io/v1alpha1
kind: ReplicaSchedulingPreference
metadata:
  name: fed-nginx
  namespace: ddcloud
spec:
  targetKind: FederatedDeployment
  totalReplicas: 2
  clusters:
    "*":
      weight: 1
---
# 指定各个集群的权重
apiVersion: scheduling.kubefed.io/v1alpha1
kind: ReplicaSchedulingPreference
metadata:
  name: fed-nginx
  namespace: ddcloud
spec:
  targetKind: FederatedDeployment
  totalReplicas: 3
  clusters:
    fed-worker02:
      weight: 1
    fed-worker02:
      weight: 2
---
# 指定各个集群的权重并设置最大、最小副本限制
apiVersion: scheduling.kubefed.io/v1alpha1
kind: ReplicaSchedulingPreference
metadata:
  name: fed-nginx
  namespace: ddcloud
spec:
  targetKind: FederatedDeployment
  totalReplicas: 3
  clusters:
    fed-worker02:
      minReplicas: 1
      maxReplicas: 2
      weight: 1
    fed-worker02:
      minReplicas: 1
      maxReplicas: 2
      weight: 2

```



参考：

[kubefed](https://github.com/kubernetes-sigs/kubefed)
[karmada](https://github.com/karmada-io/karmada)
