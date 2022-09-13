# 垃圾收集

垃圾收集（Garbage Collection）是 Kubernetes 用于清理集群资源的各种机制的统称。

垃圾收集允许系统清理如下资源：

- 终止的 Pod
- 已完成的 Job
- 不再存在属主引用的对象
- 未使用的容器和容器镜像
- 动态制备的、StorageClass 回收策略为 Delete 的 PV 卷
- 阻滞或者过期的 CertificateSigningRequest (CSRs)
- 在以下情形中删除了的节点对象：
  - 当集群使用云控制器管理器运行于云端时；
  - 当集群使用类似于云控制器管理器的插件运行在本地环境中时。
- 节点租约对象

## 属主与从属

Kubernetes 中很多对象通过属主引用互相链接。

属主引用（Owner Reference）可以告诉控制面哪些对象从属于其他对象。

Kubernetes 使用属主引用来为控制面以及其他 API 客户端在删除某对象时提供一个清理关联资源的机会。在大多数场合，Kubernetes 都是自动管理属主引用的。

属主关系与某些资源所使用的标签和选择算符不同。例如，考虑一个创建 EndpointSlice 对象的 Service对象。 Service 对象使用标签来允许控制面确定哪些 EndpointSlice 对象被该Service 使用。 除了标签，每个被 Service 托管的 EndpointSlice 对象还有一个属主引用属性。 属主引用可以帮助 Kubernetes 中的不同组件避免干预并非由它们控制的对象。

## 级联删除

Kubernetes 会检查并删除那些不再拥有属主引用的对象，例如在你删除了 ReplicaSet 之后留下来的 Pod，这些 Pod 的属主已经不存在了。

当你删除某个对象时，你可以控制 Kubernetes 是否去自动删除该对象的 dependents（从属），这个过程称为 级联删除（Cascading Deletion）。

级联删除有两种类型，分别如下：

- 前台级联删除
- 后台级联删除

你也可以使用 Kubernetes Finalizers 来控制垃圾收集机制如何以及何时删除包含属主引用的资源。

### 前台级联删除

在前台级联删除中，正在被你删除的属主对象首先进入 `deletion in progress` 状态。在这种状态下，针对属主对象会发生以下事情：

- Kubernetes API 服务器将对象的 **metadata.deletionTimestamp** 字段设置为对象被标记为要删除的时间点。
- Kubernetes API 服务器也会将 **metadata.finalizers** 字段设置为 foregroundDeletion。
- 在删除过程完成之前，通过 Kubernetes API 仍然可以看到该对象。

当属主对象进入 `deletion in progress` 状态后，控制器删除其 dependents（从属）对象。控制器在删除完所有从属对象之后，删除属主对象。这时，通过 Kubernetes API 就无法再看到该对象。

在前台级联删除过程中，当从属对象带有 `ownerReference.blockOwnerDeletion=true` 时，会阻止属主对象被删除。

### 后台级联删除

在后台级联删除过程中，Kubernetes 服务器立即删除属主对象，控制器在后台清理所有从属对象。

默认情况下，Kubernetes 使用后台级联删除方案，除非你手动设置了要：1）使用前台删除；2）或者选择遗弃从属对象。

### 被遗弃的从属对象

当 Kubernetes 删除某个属主对象时，被留下来的从属对象被称作被遗弃的（Orphaned）对象。

默认情况下，Kubernetes 会删除从属对象。要了解如何重载这种默认行为，可参阅[删除属主对象和遗弃从属对象](https://kubernetes.io/zh-cn/docs/tasks/administer-cluster/use-cascading-deletion/#set-orphan-deletion-policy)。

## 未使用容器和镜像的垃圾收集

kubelet 会每五分钟对未使用的镜像执行一次垃圾收集，每分钟对未使用的容器执行一次垃圾收集。

你应该避免使用外部的垃圾收集工具，因为外部工具可能会破坏 kubelet 的行为，导致外部工具不小心移除应该保留的容器。

### 容器镜像生命周期

Kubernetes 通过其镜像管理器（Image Manager）来管理所有镜像的生命周期，该管理器是 kubelet 的一部分，工作时与 cadvisor （帮助理解容器的资源用量和性能特征的工具）协同。

kubelet 在作出垃圾收集决定时会考虑如下磁盘用量约束：

- HighThresholdPercent
- LowThresholdPercent

磁盘用量超出所配置的 HighThresholdPercent 值时会触发垃圾收集，垃圾收集器会基于镜像上次被使用的时间来按顺序删除它们，首先删除的是最近未使用的镜像。

kubelet 会持续删除镜像，直到磁盘用量到达 LowThresholdPercent 值为止。

**注意：**

- 这里并非删除到 HighThresholdPercent 为止，而是删除到 LowThresholdPercent。类似一个窗口值，如果只删除到 HighThresholdPercent，则很可能会又超过 HighThresholdPercent，导致又触发删除，进而引起删除的频繁执行（抖动）。

### 容器垃圾收集

kubelet 会基于如下变量对所有未使用的容器执行垃圾收集操作，这些变量都是你可以定义的：

- MinAge：kubelet 可以垃圾回收某个容器时该容器的最小年龄。设置为 0 表示禁止使用此规则。
- MaxPerPodContainer：每个 Pod 可以包含的已死亡的容器个数上限。设置为小于 0 的值表示禁止使用此规则。
- MaxContainers：集群中可以存在的已死亡的容器个数上限。设置为小于 0 的值意味着禁止应用此规则。

除以上变量之外，kubelet 还会垃圾收集除无标识的以及已删除的容器，通常从最近未使用的容器开始。

当保持每个 Pod 的最大数量的容器（MaxPerPodContainer）会使得全局的已死亡容器个数超出上限 （MaxContainers）时，MaxPerPodContainers 和 MaxContainers 之间可能会出现冲突。 在这种情况下，kubelet 会调整 MaxPerPodContainer 来解决这一冲突。 最坏的情形是将 MaxPerPodContainer 降格为 1，并驱逐最近未使用的容器。

**注意：**

- 当隶属于某已被删除的 Pod 的容器的年龄超过 MinAge 时，它们也会被删除。
- kubelet 仅会回收由它所管理的容器。

### 配置垃圾收集

你可以通过配置特定于管理资源的控制器来调整资源的垃圾收集行为。
