# 干扰（Disruptions）

本指南针对的是希望构建高可用性应用的应用所有者，他们有必要了解可能发生在 Pod 上的干扰类型。

文档同样适用于想要执行自动化集群操作（例如升级和自动扩展集群）的集群管理员

## 自愿干扰和非自愿干扰

Pod 不会消失，除非有人（用户或控制器）将其销毁，或者出现了不可避免的硬件或软件系统错误。

我们把这些**不可避免的情况**称为应用的非自愿干扰（Involuntary Disruptions）。例如：

- 节点下层物理机的硬件故障
- 集群管理员错误地删除虚拟机（节点示例）
- 云提供商或虚拟机管理程序中的故障导致的虚拟机消失
- 内核错误
- 节点由于集群网络隔离从集群中消失
- 由于节点资源不足导致 pod 被驱逐。

除了资源不足的情况，大多数用户应该都熟悉这些情况；它们不是特定于 Kubernetes 的。

我们称**其他情况**为自愿干扰（Voluntary Disruptions），包括由应用所有者发起的操作和由集群管理员发起的操作。

典型的应用所有者的操作包括：

- 删除 Deployment 或其他管理 Pod 的控制器。
- 更新了 Deployment 的 Pod 模板导致 Pod 重启。
- 直接删除 Pod（例如，因为误操作）。

集群管理员操作包括：

- 排空（drain）节点进行修复或升级。
- 从集群中排空节点以缩小集群（了解集群自动扩缩）。
- 从节点中移除一个 Pod，以允许其他 Pod 使用该节点。

这些操作可能由集群管理员直接执行，也可能由集群管理员所使用的自动化工具执行，或者由集群托管提供商自动执行。

咨询集群管理员或联系云提供商，或者查询发布文档，以确定是否为集群启用了任何资源干扰源。如果没有启用，可以不用创建 Pod Disruption Budgets（Pod 干扰预算）

**注意：**

- 并非所有的自愿干扰都会受到 Pod 干扰预算的限制。例如，删除 Deployment 或 Pod 的删除操作就会跳过 Pod 干扰预算检查。

## 处理干扰

以下是减轻非自愿干扰的一些方法：

- 确保 Pod 在 request 中给出所需资源。
- 如果需要更高的可用性，请复制应用。
- 为了在运行复制的应用程序时获得更高的可用性，请将应用程序分布在机架之间（使用反关联性）或跨区域（如果使用多区域集群）。

不同自愿干扰会有不同的频率，在一个基本的 Kubernetes 集群中：

- 有用户触发的干扰
- 集群管理员或托管提供商可能运行一些可能导致自愿干扰的额外服务。例如，节点软更新可能导致自愿干扰。
- 集群（节点）自动缩放的某些实现可能导致碎片整理和紧缩节点的自愿干扰。

集群管理员或托管提供商应该已经记录了各级别的自愿干扰（如果有的话）。有些配置选项，例如在 pod spec 中 使用 PriorityClasses 也会产生自愿（和非自愿）的干扰（也就是抢占产生的干扰）。

Kubernetes 提供特性来满足在出现**频繁自愿干扰**的同时运行高可用的应用。我们称这些特性为干扰预算（Disruption Budget）。

## Pod 干扰预算（Pod Disruption Budget）

即使你会经常引入自愿性干扰，Kubernetes 提供的功能也能够支持你运行高度可用的应用。

作为一个应用的所有者，你可以为每个应用创建一个 PodDisruptionBudget（PDB）。

PDB 将限制在同一时间因自愿干扰导致的多副本应用中发生宕机的 Pod 数量。例如：

- 基于票选机制的应用希望确保运行中的副本数永远不会低于票选所需的数量。
- Web 前端可能希望确保提供负载的副本数量永远不会低于总数的某个百分比。

集群管理员和托管提供商应该使用遵循 PodDisruptionBudgets 的接口（通过调用Eviction API），而不是直接删除 Pod 或 Deployment。

例如：

- kubectl drain 命令可以用来标记某个节点即将停止服务。运行 kubectl drain 命令时，工具会尝试驱逐你所停服的节点上的所有 Pod。
- kubectl drain 所提交的驱逐请求可能会暂时被拒绝，所以该工具会周期性地重试所有失败的请求，直到目标节点上的所有的 Pod 都被终止，或者达到配置的超时时间。

PDB 指定应用可以容忍的副本数量（相当于应该有多少副本）。例如，具有 `.spec.replicas: 5` 的 Deployment 在任何时间都应该有 5 个 Pod。如果 PDB 允许其在某一时刻有 4 个副本，那么驱逐 API 将**允许同一时刻仅有一个（而不是两个）Pod 自愿干扰**（相当预算是一个）。

Pod 的“预期”数量由管理这些 Pod 的工作负载资源的 `.spec.replicas` 参数计算出来的。 控制平面通过检查 Pod 的 `.metadata.ownerReferences` 来发现关联的工作负载资源。

**注意：**

- **PDB 无法防止非自愿干扰**，但它们确实计入预算。

由于应用的滚动升级而被删除或不可用的 Pod 确实会计入干扰预算，但是工作负载资源（如 Deployment 和 StatefulSet） 在进行滚动升级时不受 PDB 的限制。

应用更新期间的故障处理方式是在对应的工作负载资源的 spec 中配置的。

当使用驱逐 API 驱逐 Pod 时，Pod 会被优雅地终止

## PodDisruptionBudget 例子

假设集群有 3 个节点，node-1 到 node-3。

集群上运行了一些应用：

- 应用 A，有 3 个副本。分别是：
  - `pod-a`
  - `pod-b`
  - `pod-c`
- 应用 B（和 A 无关），有一个 pod：`pod-x`。

最初，所有的 Pod 分布如下：

node-1 | node-2 | node-3
-|-|-
pod-a available | pod-b available | pod-c available
pod-x available | - | -

应用 A 的 3 个 Pod 都是 deployment 的一部分，并且共同拥有同一个 PDB，要求 3 个 Pod 中至少有 2 个 Pod 始终处于可用状态。

例如，假设集群管理员想要重启系统，升级内核版本来修复内核中的缺陷。集群管理员首先使用 kubectl drain 命令尝试腾空 node-1 节点。命令尝试驱逐 pod-a 和 pod-x。操作立即就成功了。两个 Pod 同时进入 terminating 状态。这时的集群处于下面的状态：

node-1 draining | node-2 | node-3
-|-|-
pod-a terminating | pod-b available | pod-c available
pod-x terminating | - | -

Deployment 控制器观察到其中一个 Pod 正在终止，因此它创建了一个替代 Pod pod-d。由于 node-1 被封锁（cordon），pod-d 落在另一个节点上。

同样其他控制器也创建了 pod-y 作为 pod-x 的替代品。

当前集群的状态如下：

node-1 draining | node-2 | node-3
-|-|-
pod-a terminating | pod-b available | pod-c available
pod-x terminating | pod-d starting | pod-y

在某一时刻，Pod 被终止，集群如下所示：

node-1 drained | node-2 | node-3
-|-|-
- | pod-b available | pod-c available
- | pod-d starting | pod-y

此时，如果一个急躁的集群管理员试图排空（drain）node-2 或 node-3，**drain 命令将被阻塞**，因为对于 Deployment 来说只有 2 个可用的 Pod，并且它的 PDB 至少需要 2 个（此时 Pod d 还没启动）。

经过一段时间，pod-d 变得可用。集群状态如下所示：

node-1 drained | node-2 | node-3
-|-|-
- | pod-b available | pod-c available
- | pod-d available | pod-y

现在，集群管理员试图排空（drain）node-2。drain 命令将尝试按照某种顺序驱逐两个 Pod：

- 假设先是 pod-b，然后是 pod-d。

命令成功驱逐 pod-b，但是当它尝试驱逐 pod-d 时将被拒绝，因为对于 Deployment 来说只剩一个可用的 Pod 了。

Deployment 创建 pod-b 的替代 Pod pod-e。因为集群中没有足够的资源来调度 pod-e，drain 命令再次阻塞。集群最终将是下面这种状态：

node-1 drained | node-2 | node-3 | no node
-|-|-|-
- | pod-b terminating | pod-c available | pod-e pending
- | pod-d available | pod-y | -

此时，集群管理员需要增加一个节点到集群中以继续升级操作。

可以看到 Kubernetes 如何改变干扰发生的速率，根据：

- 应用需要多少个副本
- 优雅关闭应用实例需要多长时间
- 启动应用新实例需要多长时间
- 控制器的类型
- 集群的资源能力

## Pod 干扰状况

**注意：**

- 要使用此行为，你必须在集群中启用 PodDisruptionsCondition 特性门控。

启用后，会给 Pod 添加一个 DisruptionTarget 状况，用来表明该 Pod 因为发生干扰而被删除。状况中的 reason 字段进一步给出 Pod 终止的原因，如下：

- PreemptionByKubeScheduler，Pod 将被调度器抢占，目的是接受优先级更高的新 Pod。要了解更多的相关信息，请参阅 Pod 优先级和抢占。
- DeletionByTaintManager，由于 Pod 不能容忍 NoExecute 污点，Pod 将被 Taint Manager（kube-controller-manager 中节点生命周期控制器的一部分）删除； 请参阅基于污点的驱逐。
- EvictionByEvictionAPI， Pod 已被标记为通过 Kubernetes API 驱逐。
- DeletionByPodGC， 绑定到一个不再存在的 Node 上的 Pod 将被 Pod 垃圾收集删除。

**注意：**

- Pod 的干扰可能会被中断。控制平面可能会重新尝试继续干扰同一个 Pod，但这没办法保证。 因此，DisruptionTarget 条件可能会添被加到 Pod 上， 但该 Pod 实际上可能不会被删除。 在这种情况下，一段时间后，Pod 干扰状况将被清除。

使用 Job（或 CronJob）时，你可能希望将这些 Pod 干扰状况作为 Job Pod 失效策略的一部分。

## 分离集群所有者和应用所有者角色

通常，将集群管理者和应用所有者视为彼此了解有限的独立角色是很有用的。这种责任分离在下面这些场景下是有意义的：

- 当有许多应用团队共用一个 Kubernetes 集群，并且有自然的专业角色
- 当第三方工具或服务用于集群自动化管理

Pod 干扰预算通过在角色之间提供接口来支持这种分离。

如果你的组织中没有这样的责任分离，则可能不需要使用 Pod 干扰预算。

## 如何在集群上执行干扰性操作

如果你是集群管理员，并且需要对集群中的所有节点执行干扰操作，例如节点或系统软件升级，则可以使用以下选项

- 接受升级期间的停机时间。
- 故障转移到另一个完整的副本集群。
  - 没有停机时间，但是对于重复的节点和人工协调成本可能是昂贵的。
- 编写可容忍干扰的应用和使用 PDB。
  - 不停机。
  - 最小的资源重复。
  - 允许更多的集群管理自动化。
  - 编写可容忍干扰的应用是棘手的，但对于支持容忍自愿干扰所做的工作，和支持自动扩缩和容忍非 自愿干扰所做工作相比，有大量的重叠