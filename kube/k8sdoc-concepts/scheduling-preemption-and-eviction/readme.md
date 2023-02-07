# 调度、抢占和驱逐

## 子章节

- [Kubernetes 调度器](kubernetes-scheduler.md)
- [将 Pod 指派给节点](assigning-pods-to-nodes.md)
- [Pod 开销](pod-overhead.md)
- Pod 调度就绪态
- Pod 拓扑分布约束
- [污点和容忍度](taints-and-tolerations.md)
- 调度框架
- 动态资源分配
- 调度器性能调优
- 资源装箱
- [Pod 优先级和抢占](pod-priority-and-preemption.md)
- [节点压力驱逐](node-pressure-eviction.md)
- API 发起的驱逐

## 概览

在 Kubernetes 中，有三个重要的 Pod 分配概览：

概念 | 描述
-|-
调度 (scheduling) | 指的是确保 Pod 匹配到合适的节点， 以便 kubelet 能够运行它们。
抢占 (Preemption) | 指的是**终止低优先级的 Pod** 以便高优先级的 Pod 可以调度运行的过程。
驱逐 (Eviction) | 是在资源匮乏的节点上，主动让一个或多个 Pod 失效的过程。

## 调度

- [Kubernetes 调度器](https://kubernetes.io/zh-cn/docs/concepts/scheduling-eviction/kube-scheduler/)
- [将 Pod 指派到节点](https://kubernetes.io/zh-cn/docs/concepts/scheduling-eviction/assign-pod-node/)
- [Pod 开销](https://kubernetes.io/zh-cn/docs/concepts/scheduling-eviction/pod-overhead/)
- [Pod 拓扑分布约束](https://kubernetes.io/zh-cn/docs/concepts/scheduling-eviction/topology-spread-constraints/)
- [污点和容忍度](https://kubernetes.io/zh-cn/docs/concepts/scheduling-eviction/taint-and-toleration/)
- [调度框架](https://kubernetes.io/zh-cn/docs/concepts/scheduling-eviction/scheduling-framework/)
- [调度器性能调试](https://kubernetes.io/zh-cn/docs/concepts/scheduling-eviction/scheduler-perf-tuning/)
- [扩展资源的资源装箱](https://kubernetes.io/zh-cn/docs/concepts/scheduling-eviction/resource-bin-packing/)

## Pod 干扰

[Pod 干扰](https://kubernetes.io/zh-cn/docs/concepts/workloads/pods/disruptions/) 是指节点上的 pod 被自愿或非自愿**终止的过程**：

- 自愿干扰是由应用程序所有者或集群管理员有意启动的。
- 非自愿干扰是无意的，可能由不可避免的问题触发，如节点耗尽资源或意外删除。

Pod 干扰涉及的内容有：

- [Pod 优先级和抢占](https://kubernetes.io/zh-cn/docs/concepts/scheduling-eviction/pod-priority-preemption/)
- [节点压力驱逐](https://kubernetes.io/zh-cn/docs/concepts/scheduling-eviction/node-pressure-eviction/)
- [API 发起的驱逐](https://kubernetes.io/zh-cn/docs/concepts/scheduling-eviction/api-eviction/)

## QoS 服务质量

更多 QoS 信息请参考：[配置 Pod 的服务质量](https://kubernetes.io/zh-cn/docs/tasks/configure-pod-container/quality-service-pod/)

QoS 服务质量，是 Kubernetes 根据 Pod 的配置为 Pod 设置的一个属性，该属性为节点内存资源不足时，该如何释放资源提供了指导性参考。

Kubernetes 创建 Pod 时就给 Pod 指定了下列一种 QoS 类（是 k8s 根据 Pod 的配置自动指定的）：

QoS 类 | 含义 | 条件 | 描述
-|-|-|-
Guaranteed | 有保证的 Pod | 同时满足以下条件：<br>1) Pod 中的所有容器都且仅设置了 CPU 和内存的 limits。<br>2) pod中的所有容器都设置了 CPU 和内存的 requests。<br>3) Pod 中同一个容器的 request 和 limit 相等的。 | Kubernetes 认为此类 Pod 是稳定的，可以很好的控制 CPU 和内存，这些 Pod 的资源一定不会超过 request（因为 limit 和 request 相等）。
Burstable | 不稳定的 Pod | Pod 中的容器设置了 request 和 limit，但是至少有一个 requests 和 limits 的设置不相同。<br>**注意：**若容器指定了 requests 而未指定 limits，则 limits 的值等于节点 resource 的最大值；若容器指定了 limits 而未指定 requests，则 requests 的值等于 limits。| Kubernetes 认为 Pod 是不稳定的，因为使用的资源可能会超过 request，导致节点容量不足。
BestEffort | 尽最大努力的 Pod | Pod 中所有容器均为设置 request 和 limit。| Kubernetes 认为这是最不靠谱的 Pod。
