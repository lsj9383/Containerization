# 调度、抢占和驱逐

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
