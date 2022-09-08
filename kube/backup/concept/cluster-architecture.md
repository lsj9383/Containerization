# 集群架构

## Node

kube 通过将 pods 放入 nodes 中运行，来执行期望的工作。节点可以是虚拟机，也可以是物理机。每个节点都包含运行 Pod 所需要的服务，这些服务由 `Control Panel` 提供。

节点上的组建包括：kubelet，container runtime，以及 kube-proxy。

### 节点状态

节点状态可以通过命令进行查询：

```sh
kubectl describe node <节点名称>
```

包含以下几个部分：

- Address，相关的 IP 信息：

  ```sh
  Addresses:
    InternalIP:  w.x.y.z
    Hostname:    vm-w-x-centos
  ```

- `Conditions`，节点的相关状况信息，包括以下几种状况：
  - Ready，为 True 时，节点运行状况良好并准备好接受 Pod；为 False 时，如果节点运行状况不佳并且不接受 Pod；为 Unknown 时，节点控制器最近一次未从节点收到消息 node-monitor-grace-period 信息。
  - MemoryPressure，为 True 时，内存不足。
  - PIDPressure，为 True 时，进程存在压力，即节点上的进程太多。
  - DiskPressure，为 True 时，磁盘大小有压力，也就是说，磁盘容量过低。
  - NetworkUnavailable， True 如果节点的网络配置不正确，否则 False。

  ```sh
  Conditions:
  Type             Status  LastHeartbeatTime                 LastTransitionTime                Reason                       Message
  ----             ------  -----------------                 ------------------                ------                       -------
  MemoryPressure   False   Sun, 25 Oct 2020 16:45:05 +0800   Sat, 24 Oct 2020 22:57:50 +0800   KubeletHasSufficientMemory   kubelet has sufficient memory available
  DiskPressure     False   Sun, 25 Oct 2020 16:45:05 +0800   Sat, 24 Oct 2020 22:57:50 +0800   KubeletHasNoDiskPressure     kubelet has no disk pressure
  PIDPressure      False   Sun, 25 Oct 2020 16:45:05 +0800   Sat, 24 Oct 2020 22:57:50 +0800   KubeletHasSufficientPID      kubelet has sufficient PID available
  Ready            True    Sun, 25 Oct 2020 16:45:05 +0800   Sat, 24 Oct 2020 22:57:52 +0800   KubeletReady                 kubelet is posting ready status
  ```

  - 如果 Ready 的 Status 为 Unknow 和 False 的时间超过了 `pod-eviction-timeout`（传递给 kube-controller-manager 的参数）的时间，则节点上的所有 Pod 都计划由节点控制器删除。默认时间为五分钟。节点控制器在确认 Pod 在集群中已经停止运行前，不会强制删除它们。你可以看到这些可能在无法访问的节点上运行的 Pod 处于 Terminating 或者 Unknown 状态。如果 kube 无法推断节点是否永久离开集群，则需要管理员手动操作删除该节点。
  - 节点生命周期控制器会自动创建代表状况的 污点。 当调度器将 Pod 指派给某节点时，会考虑节点上的污点。 Pod 则可以通过容忍度（Toleration）表达所能容忍的污点。

- `Capacity and Allocatable`，描述节点上的可用资源，例如 CPU、内存以及可以调度到该节点上的最大 pods 数量。
  - Capacity 是节点最大的资源。
  - Allocatable 是节点可进行分配的最大资源。
  - Allocated resources 是已分配使用的资源。

  ```sh
  Capacity:
    cpu:                8
    ephemeral-storage:  103079868Ki
    hugepages-2Mi:      0
    memory:             16165932Ki
    pods:               110
  Allocatable:
    cpu:                8
    ephemeral-storage:  94998406192
    hugepages-2Mi:      0
    memory:             16063532Ki
    pods:               110
  Allocated resources:
    (Total limits may be over 100 percent, i.e., overcommitted.)
    Resource           Requests    Limits
    --------           --------    ------
    cpu                850m (10%)  100m (1%)
    memory             190Mi (1%)  390Mi (2%)
    ephemeral-storage  0 (0%)      0 (0%)
    hugepages-2Mi      0 (0%)      0 (0%)
  ```

- Info，描述节点的常规信息，例如内核版本、kube 版本、Docker 版本、OS 名称等等。

  ```sh
  System Info:
    Machine ID:                 4211ff3f594041f3966d836585a11a05
    System UUID:                CCB9FABA-F3C2-334D-8D4D-26BF23787716
    Boot ID:                    f5e43d7e-429e-425b-97c3-0c004f6b52c1
    Kernel Version:             3.10.107-1-tlinux2_kvm_guest-0049
    OS Image:                   Tencent tlinux 2.2 (Final)
    Operating System:           linux
    Architecture:               amd64
    Container Runtime Version:  docker://19.3.4
    Kubelet Version:            v1.19.3
    Kube-Proxy Version:         v1.19.3
  ```

### Node Controller

Node Controller 是 kube Control Plane 组件的一部分，用于管理节点。

Node Controller 有多个作用：

- 在注册节点时将CIDR块分配给该节点（如果已打开CIDR分配）。

- 使内部节点列表与云提供商的可用计算机列表保持最新。

- 监视节点的运行状况，在节点不可访问时，会将节点 Ready 的 Status 更改为 Unknown（例如节点控制器因为节点 down 而不再收到心跳），若节点始终无法访问，这会让节点上的 Pod 受到驱逐（优雅关闭 Pod）。心跳超过 40s 则记录为 Unknown，持续 5m 则开始驱逐 Pods。节点控制器每过一段时间（-node-monitor-period seconds）检查节点状态。

### 心跳

心跳由 kube nodes 发出，帮助判断节点的可用性。目前有两种形式的心跳：

- NodeStatus，当状态发送变化，或者默认的时间内没有更新时（默认间隔为 5 分钟， 比无法访问的节点的 40 秒默认超时长得多），kubelet 将会更新 NodeStatus。
- Lease object，每个节点都关联了一个 Lease 对象（kube-node-lease 命名空间。
  - kube 每十秒（默认时间）创建和更新 Lease object，且 Lease 的更新是独立于 NodeStatus 的。如果 Lease 更新失败，则 kubelet 将会重试更新，且以 200ms 开始的指数回退重试，最多 7 次
  。

## Control Plane-Node Communication

## Controllers
