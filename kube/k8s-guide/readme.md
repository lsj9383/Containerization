# Kubernetes 权威指南

[TOC]

## 概览

## Kubernetes 基本概念和术语

### Master

Kubernetes 里 Controller Plane 负责集群控制，运行的节点被称为 Master 节点。

Master 运行以下进程：

- Kubernetes API Server（kube-apiserver）：提供了 HTTP Rest 接口的关键服务进程，是 Kubernetes 里所有资源的增、删、改、查等操作的唯一入口，也是集群控制的入口进程。
- Kubernetes Controller Manager（kube-controller-manager）：Kubernetes 里所有资源对象的自动化控制中心，可以将其理解为资源对象的“大总管”。
- Kubernetes Scheduler（kube-scheduler）：负责资源调度（Pod调度）的进程，相当于公交公司的“调度室”。
- etcd：负责进行资源存储。

### Node

Kubernetes 集群中的其他机器被称为 Node，在较早的版本中也被称为 Minion。与 Master 一样，Node可以是一台物理主机，也可以是一台虚拟机。

Node 是 Kubernetes 集群中的工作负载节点，每个 Node 都会被 Master 分配一些工作负载，当某个Node宕机时，其上的工作负载会被 Master 自动转移到其他节点上。

Node 运行以下进程：

- kubelet：负责 Pod 对应的容器的创建、启停等任务，同时与 Master 密切协作，实现集群管理的基本功能。
- kube-proxy：实现 Kubernetes Service 的通信与负载均衡机制的重要组件。
- Docker Engine（docker）：Docker 引擎，负责本机的容器创建和管理工作。

我们可以查询集群中有哪些 Node：

```sh
$ kubectl get nodes
NAME        STATUS   ROLES    AGE   VERSION
10.0.1.12   Ready    <none>   96d   v1.22.5-tke.9
10.0.1.2    Ready    <none>   96d   v1.22.5-tke.9
```

通过 `kubectl describe node <node_name>` 查看某个 Node 的详细信息：

```sh
$ kubectl describe node 10.0.1.12
Name:               10.0.1.12
Roles:              <none>
Labels:             beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/instance-type=S3.MEDIUM2
                    beta.kubernetes.io/os=linux
                    cloud.tencent.com/node-instance-id=ins-jifjbvhg
                    failure-domain.beta.kubernetes.io/region=gz
                    failure-domain.beta.kubernetes.io/zone=100003
                    kubernetes.io/arch=amd64
                    kubernetes.io/hostname=10.0.1.12
                    kubernetes.io/os=linux
                    node.kubernetes.io/instance-type=S3.MEDIUM2
                    topology.com.tencent.cloud.csi.cbs/zone=ap-guangzhou-3
                    topology.kubernetes.io/region=gz
                    topology.kubernetes.io/zone=100003
Annotations:        csi.volume.kubernetes.io/nodeid: {"com.tencent.cloud.csi.cbs":"ins-jifjbvhg"}
                    node.alpha.kubernetes.io/ttl: 0
                    volumes.kubernetes.io/controller-managed-attach-detach: true
CreationTimestamp:  Thu, 02 Feb 2023 11:12:16 +0800
Taints:             <none>
Unschedulable:      false
Lease:
  HolderIdentity:  10.0.1.12
  AcquireTime:     <unset>
  RenewTime:       Tue, 09 May 2023 19:20:08 +0800
Conditions:
  Type                 Status  LastHeartbeatTime                 LastTransitionTime                Reason                       Message
  ----                 ------  -----------------                 ------------------                ------                       -------
  NetworkUnavailable   False   Thu, 02 Feb 2023 11:12:23 +0800   Thu, 02 Feb 2023 11:12:23 +0800   RouteCreated                 RouteController created a route
  MemoryPressure       False   Tue, 09 May 2023 19:19:34 +0800   Thu, 02 Feb 2023 11:12:16 +0800   KubeletHasSufficientMemory   kubelet has sufficient memory available
  DiskPressure         False   Tue, 09 May 2023 19:19:34 +0800   Thu, 02 Feb 2023 11:12:16 +0800   KubeletHasNoDiskPressure     kubelet has no disk pressure
  PIDPressure          False   Tue, 09 May 2023 19:19:34 +0800   Thu, 02 Feb 2023 11:12:16 +0800   KubeletHasSufficientPID      kubelet has sufficient PID available
  Ready                True    Tue, 09 May 2023 19:19:34 +0800   Thu, 02 Feb 2023 11:13:07 +0800   KubeletReady                 kubelet is posting ready status
Addresses:
  InternalIP:  10.0.1.12
  ExternalIP:  43.136.58.179
  Hostname:    10.0.1.12
Capacity:
  cpu:                2
  ephemeral-storage:  51539404Ki
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             1763088Ki
  pods:               61
Allocatable:
  cpu:                1900m
  ephemeral-storage:  47498714648
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             1136400Ki
  pods:               61
System Info:
  Machine ID:                 23bed29b358543de979ba0c585cd79a4
  System UUID:                23bed29b-3585-43de-979b-a0c585cd79a4
  Boot ID:                    77bccdd7-fbe2-4ae9-9a43-0fd608ebfd1c
  Kernel Version:             5.4.119-19-0009.11
  OS Image:                   TencentOS Server 3.1 (Final)
  Operating System:           linux
  Architecture:               amd64
  Container Runtime Version:  docker://19.3.9-tke.1
  Kubelet Version:            v1.22.5-tke.9
  Kube-Proxy Version:         v1.22.5-tke.9
PodCIDR:                      172.16.0.0/26
PodCIDRs:                     172.16.0.0/26
ProviderID:                   qcloud:///100003/ins-jifjbvhg
Non-terminated Pods:          (11 in total)
  Namespace                   Name                                    CPU Requests  CPU Limits  Memory Requests  Memory Limits  Age
  ---------                   ----                                    ------------  ----------  ---------------  -------------  ---
  default                     kubernetes-proxy-8ccd9c785-jw9qh        100m (5%)     0 (0%)      128Mi (11%)      0 (0%)         95d
  default                     nodeport-test-deploy-bc9796544-xbtx2    0 (0%)        0 (0%)      0 (0%)           0 (0%)         84d
  kube-system                 coredns-5dcdbb78f4-5pl2v                100m (5%)     0 (0%)      30M (2%)         170M (14%)     96d
  kube-system                 csi-cbs-controller-7769587bbf-dsbws     600m (31%)    7 (368%)    300Mi (27%)      9Gi (830%)     96d
  kube-system                 csi-cbs-node-hjjgg                      0 (0%)        0 (0%)      0 (0%)           0 (0%)         96d
  kube-system                 ip-masq-agent-ksvrn                     0 (0%)        0 (0%)      0 (0%)           0 (0%)         96d
  kube-system                 kube-proxy-bm6vj                        0 (0%)        0 (0%)      0 (0%)           0 (0%)         96d
  kube-system                 l7-lb-controller-78674b9ff8-hpsc4       0 (0%)        0 (0%)      0 (0%)           0 (0%)         96d
  kube-system                 tke-bridge-agent-pltzs                  0 (0%)        0 (0%)      0 (0%)           0 (0%)         96d
  kube-system                 tke-cni-agent-l9skz                     0 (0%)        0 (0%)      0 (0%)           0 (0%)         96d
  kube-system                 tke-monitor-agent-v5c9t                 10m (0%)      100m (5%)   30Mi (2%)        100Mi (9%)     96d
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  Resource           Requests         Limits
  --------           --------         ------
  cpu                810m (42%)       7100m (373%)
  memory             510247808 (43%)  9938534016 (854%)
  ephemeral-storage  0 (0%)           0 (0%)
  hugepages-1Gi      0 (0%)           0 (0%)
  hugepages-2Mi      0 (0%)           0 (0%)
Events:              <none>
```

可见包含了以下信息：

- Node 的基本信息：名称、标签、创建时间等。
- Node 当前的运行状态：Node 启动后会做一系列的自检工作，比如磁盘空间是否不足（DiskPressure）、内存是否不足（MemoryPressure）、网络是否正常（NetworkUnavailable）、PID 资源是否充足（PIDPressure）。在一切正常时设置 Node 为 Ready 状态（Ready=True），该状态表示 Node 处于健康状态，Master 将可以在其上调度新的任务了（如启动 Pod）。
- Node 的主机地址与主机名。
- Node 上的资源数量：描述 Node 可用的系统资源，包括 CPU、内存数量、最大可调度 Pod 数量等。
- Node 可分配的资源量：描述 Node 当前可用于分配的资源量。
- 主机系统信息：包括主机 ID、系统 UUID、Linux kernel 版本号、操作系统类型与版本、Docker版本号、kubelet与kube-proxy的版本号等。
- 当前运行的 Pod 列表概要信息。
- 已分配的资源使用概要信息，例如资源申请的最低、最大允许使用量占系统总量的百分比。
- Node 相关的 Event 信息。

### Pod

一个 Pod 中包括多个容器，它们共享网络空间，可以共享挂载等。

一个 Pod 由两类容器组成组成：

- 特殊的被称为 “根容器” 的 Pause 容器
- 一个或多个紧密相关的用户业务容器

Kubernetes 为每个 Pod 都分配了唯一的 IP 地址，称之为 Pod IP，一个 Pod 里的多个容器共享 Pod IP 地址（也就是共享网络空间）。

**注意：**

- Kubernetes 假设要求底层网络支持集群内任意两个 Pod 之间的 TCP/IP 直接通信，这通过 Flannel 等机制进行实现。

### Label（标签）

一个 Label 是一个 key=value 的键值对，其中 key 与 value 由用户自己指定。

一个资源对象可以定义任意数量的 Label，同一个 Label 也可以被添加到任意数量的资源对象上。Label 通常在资源对象定义时确定，也可以在对象创建后动态添加或者删除。

一些常用的 Label 示例如下。

- 版本标签："release" : "stable"、"release" : "canary"。
- 环境标签："environment":"dev"、"environment":"qa"、"environment":"production"。
- 架构标签："tier" : "frontend"、"tier" : "backend"、"tier" : "middleware"。
- 分区标签："partition" : "customerA"、"partition" : "customerB"。
- 质量管控标签："track" : "daily"、"track" : "weekly"。

给某个资源对象定义一个 Label，就相当于给它打了一个标签，随后可以通过 Label Selector（标签选择器）查询和筛选拥有某些 Label 的资源对象。

Label Selector 在 Kubernetes 中的重要使用场景如下：

- kube-controller 进程通过在资源对象 RC 上定义的 Label Selector 来筛选要监控的 Pod 副本数量，使 Pod 副本数量始终符合预期设定的全自动控制流程。
- kube-proxy 进程通过 Service 的 Label Selector 来选择对应的 Pod，自动建立每个 Service 到对应 Pod 的请求转发路由表，从而实现 Service 的智能负载均衡机制。
- 通过对某些 Node 定义特定的 Label，并且在 Pod 定义文件中使用 NodeSelector 这种标签调度策略，kube-scheduler 进程可以实现 Pod 定向调度的特性。

### Replication Controller

RC 是 Kubernetes 系统中的核心概念之一，简单来说，它其实定义了一个期望的场景，即声明某种 Pod 的副本数量在任意时刻都符合某个预期值，所以 RC 的定义包括如下几个部分：

- Pod 期待的副本数量。
- 用于筛选目标 Pod 的 Label Selector。
- 当 Pod 的副本数量小于预期数量时，用于创建新 Pod 的 Pod 模板（template）。

一个 RC Demo：

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: frontend
spec:
  replicas: 1
  selector:
    tier: frontend
  template:
    metadata:
      labels:
        app: app-demo
        tier: frontend
    spec:
      containers:
        - name: tomcat-demo
          image: tomcat
          imagePullPolicy: IfNotPresent
          env:
            - name: GET_HOSTS_FROM
              value: dns
          ports:
            - containerPort: 80
```

定义了一个 RC 并将其提交到 Kubernetes 集群中后，Master 上的 Controller Manager 组件就得到通知，定期巡检系统中当前存活的目标 Pod，并确保目标 Pod 实例的数量刚好等于此 RC 的期望值，如果有过多的 Pod 副本在运行，系统就会停掉一些 Pod，否则系统会再自动创建一些 Pod。

此外，在运行时，我们可以通过修改 RC 的副本数量，来实现 Pod 的动态缩放（Scaling），这可以通过执行 kubectl scale 命令来一键完成：

```sh
$ kubectl scale rc redis-slave --replicas=3
scaled
```

Replication Controller 由于与 Kubernetes 代码中的模块 Replication Controller 同名，同时 “Replication Controller” 无法准确表达它的本意，所以在 Kubernetes 1.2 中，升级为另外一个新概念：Replica Set。

官方解释 RS 其为 “下一代的RC” 。RS 与 RC当前的唯一区别是，Replica Sets 支持基于集合的 Label selector（Set-based selector），而 RC 只支持基于等式的Label Selector（equality-based selector），这使得Replica Set的功能更强。下面是等价于之前RC例子的Replica Set的定义（省去了Pod模板部分的内容）：

```yaml
apiVersion: extensions/v1beta1
kind: ReplicaSet
metadata:
  name: frontend
spec:
  replicas: 1
  selector:
    matchLabels:
      tier: frontend
    matchExpressions:
      - {key: tier, operator: In, values: [frontend]}
  template:
    ......
```

### Deployment
