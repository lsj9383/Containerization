# 节点

Kubernetes 通过将容器放入在节点（Node）上运行的 Pod 中来执行你的工作负载。

节点可以是一个**虚拟机**或者**物理机器**，取决于所在的集群配置。

每个节点包含运行 Pod 所需的服务，而对这些节点的管理，由控制面板（Control Panel）负责。

通常集群中会有若干个节点；而在一个学习所用或者资源受限的环境中，你的集群中也可能只有一个节点。

节点上的组件包括：

- kubelet
- kube-proxy
- 容器运行时（Container Runtinme）

## 管理

向 API 服务器添加节点的方式主要有两种：

- 节点上的 kubelet 向控制面执行自注册
- 你（或者别的什么人）手动添加一个 **Node 对象**（没错，Node 在 k8s 中也抽象成了对象）。


当用上面任意方法向 API 添加节点后，Control Panel 会检查新的 Node 对象是否合法。

例如，如果你尝试使用下面的 JSON 对象来创建 Node 对象：

```json
{
  "kind": "Node",
  "apiVersion": "v1",
  "metadata": {
    "name": "10.240.79.157",
    "labels": {
      "name": "my-first-k8s-node"
    }
  }
}
```

Kubernetes 会在内部创建一个 Node 对象作为节点的表示，同时进行检查：

- Kubernetes 检查 kubelet 向 API 服务器注册节点时使用的 metadata.name 字段是否匹配。
- 如果节点是健康的（即所有必要的服务都在运行中），则该节点可以用来运行 Pod。
- 否则，直到该节点变为健康之前，所有的集群活动都会忽略该节点。

**注意：**

- Kubernetes 会一直保存着非法节点对应的对象，并持续检查该节点是否已经变得健康。
- 你，或者某个控制器必须**显式地**删除该 Node 对象以停止健康检查操作。

Node 对象的名称必须是合法的 **DNS 子域名**。

### 节点名称唯一性

节点的 name 用来标识 Node 对象。

没有两个 Node 可以同时使用相同的名称（node 对象没有命名空间）。

### 节点自注册

当 kubelet 标记 `--register-node` 为 true（默认）时，它会尝试向 API 服务注册自己。这是首选模式，被绝大多数发行版选用。

对于自注册模式，kubelet 使用下列参数启动：

- --kubeconfig - 用于向 API 服务器执行身份认证所用的凭据的路径。
- --cloud-provider - 与某云驱动 进行通信以读取与自身相关的元数据的方式。
- --register-node - 自动向 API 服务注册。
- --register-with-taints - 使用所给的污点列表 （逗号分隔的 <key>=<value>:<effect>）注册节点。当 register-node 为 false 时无效。
- --node-ip - 节点 IP 地址。
- --node-labels - 在集群中注册节点时要添加的标签。 （参见 NodeRestriction 准入控制插件所实施的标签限制）。
- --node-status-update-frequency - 指定 kubelet 向控制面发送状态的频率。

启用 Node 鉴权模式和 NodeRestriction 准入插件时，仅授权 kubelet 创建或修改其自己的节点资源。

### 手动节点管理

你可以使用 kubectl 来创建和修改 Node 对象。

如果你希望手动创建节点对象时，请设置 kubelet 标志 --register-node=false。

你可以修改 Node 对象（忽略 --register-node 设置）。例如，你可以修改节点上的标签或并标记其为不可调度。

你可以结合使用 Node 上的标签和 Pod 上的选择算符来控制调度。例如，你可以限制某 Pod 只能在符合要求的节点子集上运行。

如果标记节点为不可调度（unschedulable），将阻止新 Pod 调度到该 Node 之上， 但不会影响任何已经在其上的 Pod。 这是重启节点或者执行其他维护操作之前的一个有用的准备步骤。

要标记一个 Node 为不可调度，执行以下命令：

```sh
kubectl cordon $NODENAME
```

**注意：**

- 被 DaemonSet 控制器创建的 Pod 能够在不可调度的节点上运行。

## 节点状态

一个节点的状态包含以下信息:

- 地址（Addresses）
- 状况（Condition）
- 容量与可分配（Capacity）
- 信息（Info）
- 
你可以使用 kubectl 来查看节点状态和其他细节信息：

```sh
kubectl describe node <节点名称>
```

### 地址

**Addresses** 中包含了多个 IP 字段，这些字段的用法取决于你的云服务商或者物理机配置。

- HostName：由节点的内核报告。可以通过 kubelet 的 --hostname-override 参数覆盖。
- ExternalIP：通常是节点的可外部路由（从集群外可访问）的 IP 地址。
- InternalIP：通常是节点的仅可在集群内部路由的 IP 地址。

```sh
# 例如某集群内部 IP 为 10.0.1.10，外部 IP 为 119.29.68.7 的 node：

[root@VM-1-2-centos ~]# kubectl describe node 10.0.1.10|grep Hostname
  Hostname:    10.0.1.10
[root@VM-1-2-centos ~]# kubectl describe node 10.0.1.10|grep ExternalIP
  ExternalIP:  119.29.68.7
[root@VM-1-2-centos ~]# kubectl describe node 10.0.1.10|grep InternalIP
  InternalIP:  10.0.1.10
```

### 状况

**Conditions** 字段描述了所有 Running 节点的状况。状况的示例包括：

节点状况 | 描述
Ready | 如节点是健康的并已经准备好接收 Pod 则为 True；<br>False 表示节点不健康而且不能接收 Pod；<br>Unknown 表示节点控制器在最近 node-monitor-grace-period 期间（默认 40 秒）没有收到节点的消息（没有心跳）
DiskPressure | True 表示节点存在磁盘空间压力，即磁盘可用量低, 否则为 False
MemoryPressure | True 表示节点存在内存压力，即节点内存可用量低，否则为 False
PIDPressure | True 表示节点存在进程压力，即节点上进程过多；否则为 False。（其实就是 pid 句柄是否充足）
NetworkUnavailable | True 表示节点网络配置不正确；否则为 False

```sh
$ kubectl describe node 10.0.1.10
...
Conditions:
  Type                 Status  LastHeartbeatTime                 LastTransitionTime                Reason                       Message
  ----                 ------  -----------------                 ------------------                ------                       -------
  NetworkUnavailable   False   Thu, 08 Sep 2022 17:56:28 +0800   Thu, 08 Sep 2022 17:56:28 +0800   RouteCreated                 RouteController created a route
  MemoryPressure       False   Tue, 13 Sep 2022 01:36:15 +0800   Thu, 08 Sep 2022 17:56:27 +0800   KubeletHasSufficientMemory   kubelet has sufficient memory available
  DiskPressure         False   Tue, 13 Sep 2022 01:36:15 +0800   Thu, 08 Sep 2022 17:56:27 +0800   KubeletHasNoDiskPressure     kubelet has no disk pressure
  PIDPressure          False   Tue, 13 Sep 2022 01:36:15 +0800   Thu, 08 Sep 2022 17:56:27 +0800   KubeletHasSufficientPID      kubelet has sufficient PID available
  Ready                True    Tue, 13 Sep 2022 01:36:15 +0800   Thu, 08 Sep 2022 17:57:17 +0800   KubeletReady                 kubelet is posting ready status
...
```

如果 Ready 状况的 status 处于 Unknown 或者 False 状态的时间超过了 pod-eviction-timeout 值（一个传递给 kube-controller-manager 的参数），节点控制器会对节点上的所有 Pod 触发 API 发起的驱逐。默认的逐出超时时长为 5 分钟。

某些情况下，当节点不可达时，API 服务器不能和其上的 kubelet 通信。 删除 Pod 的决定不能传达给 kubelet，直到它重新建立和 API 服务器的连接为止。 与此同时，被计划删除的 Pod 可能会继续在游离的节点上运行。

节点控制器在确认 Pod 在集群中已经停止运行前，不会强制删除它们。 你可以看到可能在这些无法访问的节点上运行的 Pod 处于 Terminating 或者 Unknown 状态。 如果 kubernetes 不能基于下层基础设施推断出某节点是否已经永久离开了集群， 集群管理员可能需要手动删除该节点对象。

从 Kubernetes 删除节点对象将导致 API 服务器删除节点上所有运行的 Pod 对象并释放它们的名字。

### 容量（Capacity）与可分配（Allocatable）

这两个值描述节点上的可用资源：CPU、内存和可以调度到节点上的 Pod 的个数上限。

- capacity 块中的字段标示节点拥有的资源总量。
- allocatable 块指示节点上可供**普通 Pod 消耗的资源量**。

```sh
...
$ kubectl describe node 10.0.1.10
Capacity:
  cpu:                2
  ephemeral-storage:  51539404Ki
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             1763188Ki
  pods:               61
Allocatable:
  cpu:                1900m
  ephemeral-storage:  47498714648
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             1136500Ki
  pods:               61
...
```

### 信息（Info）
Info 指的是节点的一般信息，如内核版本、Kubernetes 版本（kubelet 和 kube-proxy 版本）、 容器运行时详细信息，以及节点使用的操作系统。 kubelet 从节点收集这些信息并将其发布到 Kubernetes API。

```sh
...
$ kubectl describe node 10.0.1.10
...
System Info:
  Machine ID:                 94d22c567e65467da68d2aa8bc10941e
  System UUID:                94d22c56-7e65-467d-a68d-2aa8bc10941e
  Boot ID:                    eae9d273-3cb5-48a2-ba7b-0b2c725e7cb1
  Kernel Version:             5.4.119-19-0009.3
  OS Image:                   TencentOS Server 3.1 (Final)
...
```

## 心跳

Kubernetes 节点发送的心跳帮助你的集群确定每个节点的可用性，并在检测到故障时采取行动。

对于节点，有两种形式的心跳:

- 更新节点的 `.status`（就是 Conditions 里面的那个 status）。
- kube-node-lease 名字空间中的 Lease（租约）对象。 每个节点都有一个关联的 Lease 对象。
- 
与 Node 的 **.status** 更新相比，**Lease** 是一种轻量级资源。使用 Lease 来表达心跳在大型集群中可以减少这些更新对性能的影响。

kubelet 负责创建和更新节点的 **.status**，以及更新它们对应的 Lease：

- 当节点状态发生变化时，或者在配置的时间间隔内没有更新事件时，kubelet 会更新 .status。 .status 更新的默认间隔为 5 分钟（比节点不可达事件的 40 秒默认超时时间长很多）。
- kubelet 会创建并每 10 秒（默认更新间隔时间）更新 Lease 对象。 Lease 的更新独立于 Node 的 .status 更新而发生。 如果 Lease 的更新操作失败，kubelet 会采用指数回退机制，从 200 毫秒开始重试， 最长重试间隔为 7 秒钟。

```sh
$ kubectl describe node 10.0.1.10
...
Lease:
  HolderIdentity:  10.0.1.10
  AcquireTime:     <unset>
  RenewTime:       Tue, 13 Sep 2022 01:51:22 +0800
Conditions:
  Type                 Status  LastHeartbeatTime                 LastTransitionTime                Reason                       Message
  ----                 ------  -----------------                 ------------------                ------                       -------
  NetworkUnavailable   False   Thu, 08 Sep 2022 17:56:28 +0800   Thu, 08 Sep 2022 17:56:28 +0800   RouteCreated                 RouteController created a route
  MemoryPressure       False   Tue, 13 Sep 2022 01:51:17 +0800   Thu, 08 Sep 2022 17:56:27 +0800   KubeletHasSufficientMemory   kubelet has sufficient memory available
  DiskPressure         False   Tue, 13 Sep 2022 01:51:17 +0800   Thu, 08 Sep 2022 17:56:27 +0800   KubeletHasNoDiskPressure     kubelet has no disk pressure
  PIDPressure          False   Tue, 13 Sep 2022 01:51:17 +0800   Thu, 08 Sep 2022 17:56:27 +0800   KubeletHasSufficientPID      kubelet has sufficient PID available
  Ready                True    Tue, 13 Sep 2022 01:51:17 +0800   Thu, 08 Sep 2022 17:57:17 +0800   KubeletReady                 kubelet is posting ready status
...
```


## 节点控制器

节点控制器是 Kubernetes 控制面组件，管理节点的方方面面。

节点控制器在节点的生命周期中扮演多个角色：

- 第一个是当节点注册时为它分配一个 CIDR 区段（如果启用了 CIDR 分配）。
- 第二个是保持节点控制器内的节点列表与云服务商所提供的可用机器列表同步。如果在云环境下运行，只要某节点不健康，节点控制器就会询问云服务是否节点的虚拟机仍可用。 如果不可用，节点控制器会将该节点从它的节点列表删除。
- 第三个是监控节点的健康状况。节点控制器负责：
  - 在节点不可达的情况下，在 Node 的 `.status` 中更新 `Ready` 状况。
  - 在这种情况下，节点控制器将 Node Ready 状况更新为 Unknown。
  - 如果节点仍然无法访问：对于不可达节点上的所有 Pod 触发 API 发起的逐出操作。 默认情况下，节点控制器在将节点标记为 Unknown 后等待 5 分钟提交第一个驱逐请求。

默认情况下，节点控制器每 5 秒检查一次节点状态，可以使用 kube-controller-manager 组件上的 **--node-monitor-period** 参数来配置周期。

### 逐出速率限制

大部分情况下，节点控制器把逐出速率限制在每秒 --node-eviction-rate 个（默认为 0.1）。 这表示它每 10 秒钟内至多从一个节点驱逐 Pod。

当一个节点变为不健康时，节点的驱逐行为将发生改变。节点控制器会同时检查可用区域中不健康（Ready 状况为 Unknown 或 False） 的节点的百分比：

- 如果不健康节点的比例超过 --unhealthy-zone-threshold （默认为 0.55）， 驱逐速率将会降低。
- 如果集群较小（意即小于等于 --large-cluster-size-threshold 个节点，默认为 50）， 驱逐操作将会停止。
- 否则驱逐速率将降为每秒 --secondary-node-eviction-rate 个（默认为 0.01）。

### 资源容量跟踪

Node 对象会跟踪节点上资源的容量（例如可用内存和 CPU 数量）。 通过自注册机制生成的 Node 对象会在注册期间报告自身容量。 如果你手动添加了 Node， 你就需要在添加节点时手动设置节点容量。

Kubernetes 调度器保证节点上有足够的资源供其上的所有 Pod 使用。它会检查节点上所有容器的请求的总和不会超过节点的容量。总的请求包括由 kubelet 启动的所有容器，但**不包括**：

- 由容器运行时直接启动的容器
- 不受 kubelet 控制的其他进程。

**注意：**

- 如果要为非 Pod 进程显式保留资源。请参考为[系统守护进程预留资源](https://kubernetes.io/zh-cn/docs/tasks/administer-cluster/reserve-compute-resources/#system-reserved)。

## 节点拓扑

特性状态： Kubernetes v1.18 [beta]

如果启用了 **TopologyManager** 特性[门控](https://kubernetes.io/zh-cn/docs/reference/command-line-tools-reference/feature-gates/)， kubelet 可以在作出资源分配决策时使用拓扑提示。参考[控制节点上拓扑管理策略](https://kubernetes.io/zh-cn/docs/tasks/administer-cluster/topology-manager/)了解详细信息。

## 节点优雅关闭

特性状态： Kubernetes v1.21 [beta]

kubelet 会尝试检测节点系统关闭事件并终止在节点上运行的所有 Pod。

在节点终止期间，**kubelet 保证 Pod 遵从常规的 Pod 终止流程**。

节点优雅关闭特性**依赖于 systemd**，因为它要利用 systemd 抑制器锁机制，在给定的期限内延迟节点关闭。

节点体面关闭特性受 GracefulNodeShutdown 特性门控控制。

**注意：**

- 默认情况下，下面描述的两个配置选项，shutdownGracePeriod 和 shutdownGracePeriodCriticalPods 都是被设置为 0 的，因此不会激活节点优雅关闭功能。
- 要激活此功能特性，这两个 kubelet 配置选项要适当配置，并设置为非零值。

在优雅关闭节点过程中，kubelet 分两个阶段来终止 Pod：

- 终止在节点上运行的常规 Pod。
- 终止在节点上运行的关键 Pod。


节点优雅关闭的特性对应两个 KubeletConfiguration 选项：

- shutdownGracePeriod：指定节点应延迟关闭的总持续时间。此时间是 Pod 体面终止的时间总和，不区分常规 Pod 还是关键 Pod。
- shutdownGracePeriodCriticalPods：在节点关闭期间指定用于终止关键 Pod 的持续时间。该值应小于 shutdownGracePeriod。


例如，如果设置了 shutdownGracePeriod=30s 和 shutdownGracePeriodCriticalPods=10s， 则：

- kubelet 将延迟 30 秒关闭节点。
- 在关闭期间，将保留前 20（30 - 10）秒用于优雅终止常规 Pod， 而保留最后 10 秒用于终止关键 Pod。

**注意：**

- 当 Pod 在正常节点关闭期间被驱逐时，它们会被标记为已经失败（Failed）。
- 运行 kubectl get pods 时，被驱逐的 Pod 的状态显示为 Shutdown。 并且 kubectl describe pod 表示 Pod 因节点关闭而被驱逐：

```txt
Reason:         Terminated
Message:        Pod was terminated in response to imminent node shutdown.
```

## 节点非优雅关闭
特性状态： Kubernetes v1.24 [alpha]

节点关闭的操作可能无法被 kubelet 的节点关闭管理器检测到，是因为：

- 该命令不会触发 kubelet 所使用的抑制锁定机制
- 或者是因为用户错误的原因， 即 ShutdownGracePeriod 和 ShutdownGracePeriodCriticalPod 配置不正确。 请参考以上节点优雅关闭部分了解更多详细信息。

当某节点关闭但 kubelet 的节点关闭管理器未检测到这一事件时：

- 在那个已关闭节点上、属于 StatefulSet 的 Pod 将停滞于终止状态，并且不能移动到新的运行节点上。这是因为已关闭节点上的 kubelet 已不存在，亦无法删除 Pod，因此 StatefulSet 无法创建同名的新 Pod。
  - 如果 Pod 使用了卷，则 VolumeAttachments 不会从原来的已关闭节点上删除，因此这些 Pod 所使用的卷也无法挂接到新的运行节点上。
  - 所以，那些以 StatefulSet 形式运行的应用无法正常工作。
- 如果原来的已关闭节点被恢复，kubelet 将删除 Pod，新的 Pod 将被在不同的运行节点上创建。
- 如果原来的已关闭节点没有被恢复，那些在已关闭节点上的 Pod 将永远滞留在终止状态。

为了缓解上述情况，用户可以手动将具有 NoExecute 或 NoSchedule 效果的 node.kubernetes.io/out-of-service 污点添加到节点上，标记其无法提供服务。

在非优雅关闭期间，Pod 分两个阶段终止：

- 强制删除没有匹配的 out-of-service 容忍度的 Pod。
- 立即对此类 Pod 执行分离卷操作。


**注意：**

- 在添加 node.kubernetes.io/out-of-service 污点之前，应该验证节点已经处于关闭或断电状态（而不是在重新启动中）。
- =将 Pod 移动到新节点后，用户需要手动移除停止服务的污点，并且用户要检查关闭节点是否已恢复，因为该用户是最初添加污点的用户。


## 基于 Pod 优先级的节点优雅关闭

特性状态： Kubernetes v1.23 [alpha]

为了在节点体面关闭期间提供更多的灵活性，尤其是处理关闭期间的 Pod 排序问题，节点体面关闭机制能够关注 Pod 的 **PriorityClass** 设置，前提是你已经在集群中启用了此功能特性。 此功能特性允许集群管理员基于 Pod 的优先级类（Priority Class） 显式地定义节点优雅关闭期间 Pod 的处理顺序。

前文所述的节点体面关闭特性能够分两个阶段关闭 Pod：

- 首先关闭的是非关键的 Pod
- 之后再处理关键 Pod。

如果需要显式地以更细粒度定义关闭期间 Pod 的处理顺序，需要一定的灵活度，这时可以使用基于 Pod 优先级的优雅关闭机制。

当节点体面关闭能够处理 Pod 优先级时，节点体面关闭的处理可以分为多个阶段， 每个阶段关闭特定优先级类的 Pod。

kubelet 可以被配置为按确切的阶段处理 Pod， 且每个阶段可以独立设置关闭时间。

假设集群中存在以下自定义的 Pod 优先级类。

Pod 优先级类名称 | Pod 优先级类数值
-|-
custom-class-a | 100000
custom-class-b | 10000
custom-class-c | 1000
regular/unset | 0

在 kubelet 配置中， shutdownGracePeriodByPodPriority 可能看起来是这样：

Pod 优先级类数值 | 关闭期限
-|-
100000 | 10 秒
10000 | 180 秒
1000 | 120 秒
0 | 60 秒

对应的 kubelet 配置 YAML 将会是：

```yaml
shutdownGracePeriodByPodPriority:
  - priority: 100000
    shutdownGracePeriodSeconds: 10
  - priority: 10000
    shutdownGracePeriodSeconds: 180
  - priority: 1000
    shutdownGracePeriodSeconds: 120
  - priority: 0
    shutdownGracePeriodSeconds: 60
```

上面的表格表明，所有 priority 值大于等于 100000 的 Pod 会得到 10 秒钟期限停止， 所有 priority 值介于 10000 和 100000 之间的 Pod 会得到 180 秒钟期限停止， 所有 priority 值介于 1000 和 10000 之间的 Pod 会得到 120 秒钟期限停止， 所有其他 Pod 将获得 60 秒的时间停止。

用户不需要为所有的优先级类都设置数值。例如，你也可以使用下面这种配置：

Pod 优先级类数值 | 关闭期限
-|-
100000 | 300 秒
1000 | 120 秒
0 | 60 秒


在上面这个场景中，优先级类为 custom-class-b 的 Pod 会与优先级类为 custom-class-c 的 Pod 在关闭时按相同期限处理。

如果在特定的范围内不存在 Pod，则 kubelet 不会等待对应优先级范围的 Pod。 kubelet 会直接跳到下一个优先级数值范围进行处理。

如果此功能特性被启用，但没有提供配置数据，则不会出现排序操作。

使用此功能特性需要启用 GracefulNodeShutdownBasedOnPodPriority 特性门控， 并将 kubelet 配置 中的 shutdownGracePeriodByPodPriority 设置为期望的配置， 其中包含 Pod 的优先级类数值以及对应的关闭期限。

## 交换内存管理

特性状态： Kubernetes v1.22 [alpha]

在 Kubernetes 1.22 之前，节点不支持使用交换内存，并且默认情况下， 如果在节点上检测到交换内存配置，kubelet 将无法启动。在 1.22 以后，可以逐个节点地启用交换内存支持。

要在节点上启用交换内存，必须启用 kubelet 的 NodeSwap 特性门控， 同时使用 **--fail-swap-on** 命令行参数或者将 failSwapOn 配置设置为 false。

用户还可以选择配置 memorySwap.swapBehavior 以指定节点使用交换内存的方式。例如:

```yaml
memorySwap:
  swapBehavior: LimitedSwap
```

可用的 swapBehavior 的配置选项有：

- LimitedSwap：Kubernetes 工作负载的交换内存会受限制。 不受 Kubernetes 管理的节点上的工作负载仍然可以交换。
- UnlimitedSwap：Kubernetes 工作负载可以使用尽可能多的交换内存请求， 一直到达到系统限制为止。

如果启用了特性门控但是未指定 memorySwap 的配置，默认情况下 kubelet 将使用 LimitedSwap 设置。

LimitedSwap 这种设置的行为取决于节点运行的是 v1 还是 v2 的控制组（也就是 cgroups）：

- cgroupsv1: Kubernetes 工作负载可以使用内存和交换，上限为 Pod 的内存限制值（如果设置了的话）。
- cgroupsv2: Kubernetes 工作负载不能使用交换内存。


如需更多信息以及协助测试和提供反馈，请参见 KEP-2400 及其设计提案。
