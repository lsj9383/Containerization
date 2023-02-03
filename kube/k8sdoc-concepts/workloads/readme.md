# 工作负载

## 子章节

- [Pod](pods/readme.md)
- [工作负载资源](workloadresource/readme.md)

## 概览

什么是工作负载？

> 工作负载是在 Kubernetes 上运行的应用程序。

在 Kubernetes 中，最小的工作单位是 Pod，工作负载就是对 Pod 进行管理的。

对于 Pod：

- 在 Kubernetes 中，无论你的负载是由单个组件还是由多个一同工作的组件构成，你都可以在一组 Pod 中运行它。
- 在 Kubernetes 中，Pod 代表的是集群上处于运行状态的一组`容器`的集合。
- Kubernetes 的 Pod 遵循其生命周期规律。例如，当在你的集群中运行了某个 Pod，但是 Pod 所在的节点出现致命错误时，所有该节点上的 Pod 的状态都会变成失败。Kubernetes 将这类失败视为最终状态： 即使该节点后来恢复正常运行，你也需要创建新的 Pod 以恢复应用。

不过，为了减轻用户的使用负担，通常不需要用户直接管理每个 Pod。 而是使用负载资源来替用户管理一组 Pod。这些负载资源通过配置 Controller 来确保正确类型的、处于运行状态的 Pod 个数是正确的，与用户所指定的状态相一致。

Kubernetes 提供若干种内置的工作负载资源：

- Deployment/ReplicaSet
- StatefulSet
- DaemonSet
- Job/CronJob
