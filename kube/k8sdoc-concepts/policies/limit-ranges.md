# 限制范围

**默认情况下**，Kubernetes 集群上的容器运行使用的[计算资源](https://kubernetes.io/zh-cn/docs/concepts/configuration/manage-resources-containers/)**没有限制**。

使用资源配额，集群管理员可以以 namespace 为单元，限制其中资源的使用与创建。

在 namespace 中，一个 Pod 或 Container 最多能够使用 namespace 的资源配额所定义的 CPU 和内存用量。LimitRange 是在 namespace 内限制资源分配（给多个 Pod 或 Container）的策略对象。

一个 LimitRange（限制范围） 对象提供的限制能够做到：

- 在一个 namespace 中实施对每个 Pod 或 Container 最小和最大的资源使用量的限制。
- 在一个 namespace 中实施对每个 PVC 能申请的最小和最大的存储空间大小的限制。
- 在一个 namespace 中实施对一种资源（CPU 或 Memory）的 request 和 limit 的比值的控制。
- 设置一个 namespace 中对计算资源（其实就是 Pod 中的容器）的默认 request/limit，并且自动的在运行时注入到多个 Container 中。

## 启用 LimitRange

如何启用 LimitRange 限制？

> 当某命名空间中有一个 LimitRange 对象时，将在该命名空间中实施 LimitRange 限制。

一个 LimitRange 示例：

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: cpu-min-max-demo-lr
spec:
  limits:
  - max:
      cpu: "800m"
    min:
      cpu: "200m"
    type: Container
```

**注意：**

- 这是对该命名空间中每个容器的 request 和 limit 要求，并不是对命名空间总和的要求。

## 限制范围总览

管理员在一个命名空间内创建一个 LimitRange 对象。

- 用户在 namespace 内创建 Pod ，Container 和 PersistentVolumeClaim 等资源。
- LimitRanger 准入控制器对所有没有设置计算资源需求的 Pod 和 Container 设置默认的 request 与 limit，并跟踪其使用量以保证没有超出 namespace 中存在的任意 LimitRange 对象中的最小、最大资源使用量以及使用量比值。
- 若创建或更新资源（Pod、 Container、PersistentVolumeClaim）违反了 LimitRange 的约束，向 API 服务器的请求会失败，并返回 HTTP 状态码 403 FORBIDDEN 与描述哪一项约束被违反的消息。
- 若命名空间中的 LimitRange 启用了对 cpu 和 memory 的限制， 用户必须指定这些值的需求 request 与 limit 。否则，系统将会拒绝创建 Pod。
- LimitRange 的验证仅在 **Pod 准入阶段**进行，不对正在运行的 Pod 进行验证。
  
能够使用限制范围创建的策略示例有：

- 在一个有两个节点，8 GiB 内存与 16 个核的集群中，限制一个命名空间的 Pod 申请 100m 单位，最大 500m 单位的 CPU，以及申请 200Mi，最大 600Mi 的内存。
- 为 spec 中没有 cpu 和内存需求值的 Container 定义默认 CPU 限制值与需求值 150m，内存默认需求值 300Mi。

在命名空间的总限制值小于 Pod 或 Container 的限制值的总和的情况下，可能会产生资源竞争。在这种情况下，将不会创建 Container 或 Pod。

**注意：**

- 对 LimitRange 的改变都不会影响任何已经创建了的资源。
