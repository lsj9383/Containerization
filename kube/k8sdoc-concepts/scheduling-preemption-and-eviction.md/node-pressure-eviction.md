# 节点压力驱逐

节点压力驱逐是 kubelet 主动终止 Pod 以回收节点上资源的过程。

kubelet 监控集群节点的 CPU、内存、磁盘空间和文件系统的 inode 等资源。当这些资源中的一个或者多个达到特定的消耗水平，kubelet 可以主动地使节点上一个或者多个 Pod 失效，以回收资源防止饥饿。

在节点压力驱逐期间，kubelet 将所选 Pod 的 PodPhase 设置为 Failed。这将终止 Pod。

kubelet 并不理会你配置的 PodDisruptionBudget 或者是 Pod 的 terminationGracePeriodSeconds：

- 如果你使用了软驱逐条件，kubelet 会考虑你所配置的 `eviction-max-pod-grace-period`。
- 如果你使用了硬驱逐条件，它使用 0s 宽限期来终止 Pod。

如果 Pod 是由 Deployment 等负载资源管理，则控制平面或 kube-controller-manager 会创建新的 Pod 来代替被驱逐的 Pod。

**注意：**

- kubelet 在终止 Pod 之前会尝试回收节点级资源。例如，它会在磁盘资源不足时删除未使用的容器镜像。

kubelet 使用各种参数来做出驱逐决定，存在以下参数：

- 驱逐信号
- 驱逐条件
- 监控间隔

## 驱逐信号（Eviction signals）

**驱逐信号**，指的是特定资源在特定时间点的状态。

kubelet 使用驱逐信号，通过将信号与驱逐条件进行比较来做出驱逐决定，驱逐条件是节点上应该可用资源的最小量。

kubelet 使用以下驱逐信号：

驱逐信号 | 描述
-|-
memory.available | memory.available := node.status.capacity[memory] - node.stats.memory.workingSet
nodefs.available | nodefs.available := node.stats.fs.available
nodefs.inodesFree | nodefs.inodesFree := node.stats.fs.inodesFree
imagefs.available | imagefs.available := node.stats.runtime.imagefs.available
imagefs.inodesFree | imagefs.inodesFree := node.stats.runtime.imagefs.inodesFree
pid.available | pid.available := node.stats.rlimit.maxpid - node.stats.rlimit.curproc

**注意：**

- 在上表中，描述列显示了 kubelet 如何获取信号的值。
- `memory.available` 的值来自 cgroupfs，而不是像 `free -m` 这样的工具。这很重要，因为 `free -m` 在容器中不起作用。

## 驱逐条件

你可以为 kubelet 指定自定义驱逐条件，以便在作出驱逐决定时使用。

驱逐条件的形式为 `[eviction-signal][operator][quantity]`，其中：

- eviction-signal 是要使用的驱逐信号。
- operator 是你想要的关系运算符，比如 <（小于）。
- quantity 是驱逐条件数量，例如 1Gi。quantity 的值必须与 Kubernetes 使用的数量表示相匹配。你可以使用文字值或百分比（%）。

例如，如果一个节点的总内存为 10Gi 并且你希望在可用内存低于 1Gi 时触发驱逐，则可以将驱逐条件定义为 `memory.available<10%` 或 `memory.available< 1G`（你不能同时使用二者）。

你可以配置软和硬驱逐条件。

### 软驱逐条件

软驱逐条件将驱逐条件与管理员所必须指定的宽限期配对。在超过宽限期之前，kubelet 不会驱逐 Pod。

如果没有指定的软驱逐宽限期，kubelet 会在启动时返回错误。

您可以指定 kubelet 在驱逐期间使用的：

- 软驱逐阈值宽限期
- 最大允许 pod 终止宽限期

如果您指定允许的最大宽限期并且满足软驱逐阈值，则 kubelet 使用两个宽限期中较小的一个。如果您没有指定允许的最大宽限期，kubelet 会立即杀死被驱逐的 pod，而不会正常终止。

你可以使用以下标志来配置软驱逐条件：

- eviction-soft：一组驱逐条件，如 memory.available<1.5Gi， 如果驱逐条件持续时长超过指定的宽限期，可以触发 Pod 驱逐。
- eviction-soft-grace-period：一组驱逐宽限期， 如 memory.available=1m30s，定义软驱逐条件在触发 Pod 驱逐之前必须保持多长时间。
- eviction-max-pod-grace-period：在满足软驱逐条件而终止 Pod 时使用的最大允许宽限期（以秒为单位）。

### 硬驱逐条件

硬驱逐条件**没有宽限期**，意味着当达到硬驱逐条件时，kubelet 会立即杀死 pod，而不会正常终止以回收紧缺的资源。

你可以使用 `eviction-hard` 标志来配置一组硬驱逐条件，例如 memory.available<1Gi。

kubelet 具有以下默认硬驱逐条件：

```txt
memory.available<100Mi
nodefs.available<10%
imagefs.available<15%
nodefs.inodesFree<5%（Linux 节点）
```

查看驱逐条件主要通过 kubelet 的运行参数：

```sh
$ ps aux|grep kubelet|grep eviction-hard
root       29131  2.5  5.0 1954180 89620 ?       Ssl  Sep08 399:14 /usr/bin/kubelet ...--eviction-hard=nodefs.available<10%,nodefs.inodesFree<5%,imagefs.available<15%,memory.available<100Mi...
```

## 驱逐监测间隔

kubelet 根据其配置的 `housekeeping-interval`（默认为 10s）评估驱逐条件。

## 节点条件

因为满足硬或软驱逐条件阈值，kubelet 会报告节点状况以反映节点处于压力之下。

kubelet 根据下表将驱逐信号映射为节点状况：

节点条件 | 驱逐信号 | 描述
-|-|-
MemoryPressure | memory.available | 节点上的可用内存已满足驱逐条件
DiskPressure | nodefs.available、nodefs.inodesFree、imagefs.available 或 imagefs.inodesFree	节点的根文件系统或镜像文件系统上的可用磁盘空间和 inode 已满足驱逐条件
PIDPressure | pid.available | (Linux) 节点上的可用进程标识符已低于驱逐条件

kubelet 根据配置的 `--node-status-update-frequency` 更新节点条件，默认为 10s。

### 节点条件振荡

在某些情况下，节点在软驱逐条件上下振荡，而没有保持定义的宽限期。这会导致报告的节点条件在 true 和 false 之间不断切换，从而导致错误的驱逐决策。

为了防止振荡，你可以使用 `eviction-pressure-transition-period` 标志，该标志控制 kubelet 在将节点条件转换为不同状态之前必须等待的时间。默认值为 5m。

## 回收节点级资源

kubelet 在驱逐最终用户 Pod 之前会先尝试回收节点级资源。

当报告 DiskPressure 节点状况时，kubelet 会根据节点上的文件系统回收节点级资源。

是否有 imagefs | 描述 | 操作
-|-|
有 | 节点有一个专用的 imagefs 文件系统供容器运行时使用 | 如果 nodefs 文件系统满足驱逐条件，kubelet 垃圾收集死亡 Pod 和容器。<br>如果 imagefs 文件系统满足驱逐条件，kubelet 将删除所有未使用的镜像。
无 | 节点只有一个满足驱逐条件的 nodefs 文件系统 | 对死亡的 Pod 和容器进行垃圾收集<br>删除未使用的镜像<br>kubelet 驱逐时 Pod 的选择 

## kubelet 驱逐时 Pod 的选择

如果 kubelet 回收节点级资源的尝试没有使驱逐信号低于条件，则 kubelet 开始驱逐最终用户 Pod。

kubelet 使用以下参数来确定 Pod 驱逐顺序：

- Pod 的资源使用是否超过其请求
- Pod 优先级
- Pod 相对于请求的资源使用情况

因此，kubelet 按以下顺序排列和驱逐 Pod：

1. 首先考虑资源使用量超过其 request 的 BestEffort 或 Burstable Pod。这些 Pod 会根据它们的优先级以及它们的资源使用级别超过其请求的程度被逐出。
1. 资源使用量少于 request 的 Guaranteed Pod 和 Burstable Pod 根据其优先级被最后驱逐。

说明：kubelet 不使用 Pod 的 QoS 类来确定驱逐顺序。在回收内存等资源时，你可以使用 QoS 类来估计最可能的 Pod 驱逐顺序。

仅当 Guaranteed Pod 中所有容器都被指定了请求和限制并且二者相等时，才保证 Pod 不被驱逐。这些 Pod 永远不会因为另一个 Pod 的资源消耗而被驱逐。

当 kubelet 因 inode 或 PID 不足而驱逐 Pod 时， 它使用优先级来确定驱逐顺序，因为 inode 和 PID 没有 request。

## 最小驱逐回收

在某些情况下，驱逐 Pod 只会回收少量的紧俏资源。这可能导致 kubelet 反复达到配置的驱逐条件并触发多次驱逐。

你可以使用 `--eviction-minimum-reclaim `标志或 kubelet 配置文件 为每个资源配置最小回收量：

- 当 kubelet 注意到某个资源耗尽时，它会继续回收该资源，直到回收到你所指定的数量为止。

其实相当于一个窗口：`[available, available + reclaim]`，资源抵到 available 阈值开始回收，要一直到 `available + reclaim` 阈值才停止回收。

例如，以下配置设置最小回收量：

```yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
evictionHard:
  memory.available: "500Mi"
  nodefs.available: "1Gi"
  imagefs.available: "100Gi"
evictionMinimumReclaim:
  memory.available: "0Mi"
  nodefs.available: "500Mi"
  imagefs.available: "2Gi"
```

在这个例子中，如果 nodefs.available 信号满足驱逐条件，kubelet 会回收资源，直到达到 1Gi 的条件，然后继续回收至少 500Mi 直到信号达到 1.5Gi。

这里有个疑问：最小回收过程中，节点会恢复成可调度状态否？（至少因为防震荡的原因，会默认持续5min）

类似地，kubelet 会回收 imagefs 资源，直到 imagefs.available 信号达到 102Gi。

对于所有资源，默认的 eviction-minimum-reclaim 为 0。

## 节点内存不足行为

虽然我们针对节点有驱逐 Pod 释放内存的能力，但是这始终需要时间，如果释放出内存前，节点遇到内存不足（OOM）事件，如何处理呢？

在 Kubernetes 中，如果节点在 kubelet 回收足够内存之前遇到内存不足（OOM）事件，则节点依赖 oom_killer 来响应。

kubelet 根据 Pod 的服务质量（QoS）为每个容器设置一个 oom_score_adj 值，得分越高的容器越容易被杀死。

服务质量 | oom_score_adj
-|-
Guaranteed | -997
BestEffort | 1000
Burstable | min(max(2, 1000 - (1000 * memoryRequestBytes) / machineMemoryCapacityBytes), 999)

oom_killer 根据它在节点上使用的内存百分比计算 oom_score， 然后加上 oom_score_adj 得到每个容器有效的 oom_score。然后它会杀死得分最高的容器。

与 Pod 驱逐不同，如果容器被 OOM 杀死， kubelet 可以根据其 RestartPolicy 重新启动它。

## 最佳实践

以下部分描述了驱逐配置的最佳实践。

### 可调度的资源和驱逐策略 

当你为 kubelet 配置驱逐策略时， 你应该确保调度程序不会在 Pod 触发驱逐时对其进行调度，因为这类 Pod 会立即引起内存压力。

考虑以下场景：

- 节点内存容量：10Gi
- 操作员希望为系统守护进程（内核、kubelet 等）保留 10% 的内存容量
- 操作员希望在节点内存利用率达到 95% 以上时驱逐 Pod，以减少系统 OOM 的概率。

为此，kubelet 启动设置如下：

```sh
--eviction-hard=memory.available<500Mi          # 硬驱逐条件
--system-reserved=memory=1.5Gi                  # 系统预留
```

在此配置中，--system-reserved 标志为系统预留了 1.5Gi 的内存：`1.5Gi = 总内存的 10% + 驱逐条件量`。

如果 Pod 使用的内存超过其 request 或者系统使用的内存超过 1Gi，此时 memory.available 地域 500Mi，就会触发驱逐条件。

## DaemonSet

Pod 优先级是做出驱逐决定的主要因素。

如果你不希望 kubelet 驱逐属于 DaemonSet 的 Pod，请在 Pod 规约中为这些 Pod 提供足够高的 priorityClass。你还可以使用优先级较低的 priorityClass 或默认配置，仅在有足够资源时才运行 DaemonSet Pod。
