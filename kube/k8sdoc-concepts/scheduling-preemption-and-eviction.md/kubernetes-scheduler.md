# Kubernetes 调度器

在 Kubernetes 中，`调度` 是指将 Pod 放置到合适的节点上，以便对应节点上的 Kubelet 能够运行这些 Pod。

## 调度概览

调度器通过 Kubernetes 的监测（Watch）机制来发现集群中**新创建且尚未被调度到节点上的 Pod**，这些被称为**未调度的 Pod**。

调度器会将所发现的每一个未调度的 Pod 调度到一个合适的节点上来运行。调度器会依据下文的调度原则来做出调度选择。

如果你想要理解 Pod 为什么会被调度到特定的节点上，或者你想要尝试实现一个自定义的调度器，这篇文章将帮助你了解调度。

## kube-scheduler

[kube-scheduler](https://kubernetes.io/zh-cn/docs/reference/command-line-tools-reference/kube-scheduler/) 是 Kubernetes 集群的默认调度器，并且是集群 Control Panel 的一部分。如果你真得希望或者有这方面的需求，kube-scheduler 在设计上允许你自己编写一个调度组件并替换原有的 kube-scheduler。

对每一个新创建的 Pod 或者是未被调度的 Pod，kube-scheduler **会选择一个最优**的节点去运行这个 Pod。然而，Pod 内的每一个容器对资源都有不同的需求，而且 Pod 本身也有不同的需求。因此，Pod 在被调度到节点上之前，根据这些特定的调度需求，需要对集群中的节点进行一次过滤。

在一个集群中，满足一个 Pod 调度请求的所有节点称之为 **可调度节点**。 如果没有任何一个节点能满足 Pod 的资源请求， 那么这个 Pod 将一直停留在未调度状态直到调度器能够找到合适的 Node。

调度器先在集群中找到一个 Pod 的所有可调度节点，然后根据一系列函数对这些可调度节点打分，选出其中得分最高的节点来运行 Pod。之后，调度器将这个调度决定通知给 kube-apiserver，这个过程叫做**绑定**。

在做调度决定时需要考虑的因素包括：

- 单独和整体的资源需求
- 硬件/软件/策略限制
- 亲和以及反亲和要求
- 数据局部性
- 负载间的干扰。

## kube-scheduler 调度流程

kube-scheduler 给一个 Pod 做调度选择时包含两个步骤：

步骤 | 步骤 | 描述
-|-|-
1 | 过滤 | 过滤阶段会将所有满足 Pod 调度需求的节点选出来。<br>例如，PodFitsResources 过滤函数会检查候选节点的可用资源能否满足 Pod 的资源 request。<br>通常情况下，这个节点列表包含不止一个节点。如果这个列表是空的，代表这个 Pod 不可调度。
2 | 打分 | 在打分阶段，调度器会为 Pod 从所有可调度节点中选取一个最合适的节点。根据当前启用的打分规则，调度器会给每一个可调度节点进行打分。
3 | 绑定 | kube-scheduler 会将 Pod 调度到得分最高的节点上。<br>如果存在多个得分最高的节点，kube-scheduler 会从中随机选取一个。

**注意：**

- PodFitsResources 只能使用节点当前运行的容器的 request 来计算，不会用实际的 node 可用资源计算。
- CheckNodeMemoryPressure 过滤函数可以将内存压力过大的筛选掉。

支持以下两种方式配置调度器的过滤和打分行为：

- [调度策略](https://kubernetes.io/zh-cn/docs/reference/scheduling/policies/)，允许你配置**过滤**所用的断言（Predicates） 和打分所用的优先级（Priorities）。
- [调度配置](https://kubernetes.io/zh-cn/docs/reference/scheduling/config/#profiles)，允许你配置实现不同调度阶段的插件，包括：QueueSort、Filter、Score、Bind、Reserve、Permit 等等。 你也可以配置 kube-scheduler 运行不同的配置文件。

这里还有篇调度的博客文章：[Kubernetes 调度初探](https://ustack.io/2021-01-08-Kubernetes%20%E8%B0%83%E5%BA%A6%E7%B3%BB%E7%BB%9F%E5%88%9D%E6%8E%A2.html)
