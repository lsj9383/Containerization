# Pod 开销（Pod Overhead）

在节点上运行 Pod 时，Pod 本身占用大量系统资源，这些是运行 Pod 内**容器所需资源之外**的资源。

在 Kubernetes 中，**Pod Overhead**，是一种用于计算 **Pod 基础设施**在容器 request 和 limit 之上消耗资源的方法。

在 Kubernetes 中，Pod 的开销是根据与 Pod 的 RuntimeClass 相关联的开销在准入时设置的。

如果启用了 Pod Overhead，在调度 Pod 时：

- 除了考虑容器资源 request 的总和外，还要考虑 Pod Overhead。
- 类似地，kubelet 将在确定 Pod cgroups 的大小和执行 Pod 驱逐排序时也会考虑 Pod 开销。

## 配置 Pod 开销

你需要确保使用一个定义了 overhead 字段的 RuntimeClass。

## 使用示例

要使用 Pod 开销，你需要一个定义了 overhead 字段的 RuntimeClass。

作为例子，下面的 RuntimeClass 定义中包含一个虚拟化所用的容器运行时， RuntimeClass 如下，其中每个 Pod 大约使用 120MiB 用来运行虚拟机和寄宿操作系统：

```yaml
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: kata-fc
handler: kata-fc
overhead:
  podFixed:
    memory: "120Mi"
    cpu: "250m"
```

通过指定 kata-fc RuntimeClass 处理程序创建的工作负载会将**内存和 CPU 开销**计入资源配额计算、节点调度以及 Pod cgroup size 确定。

假设我们运行下面给出的工作负载示例 test-pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
spec:
  runtimeClassName: kata-fc
  containers:
  - name: busybox-ctr
    image: busybox:1.28
    stdin: true
    tty: true
    resources:
      limits:
        cpu: 500m
        memory: 100Mi
  - name: nginx-ctr
    image: nginx
    resources:
      limits:
        cpu: 1500m
        memory: 100Mi
```

在准入阶段 RuntimeClass 准入控制器更新工作负载的 PodSpec 以包含 RuntimeClass 中定义的 overhead。

如果 PodSpec 中已定义该字段，该 Pod 将会被拒绝。在这个例子中，由于只指定了 RuntimeClass 名称，所以准入控制器更新了 Pod，使之包含 overhead。

在 RuntimeClass 准入控制器进行修改后，你可以查看更新后的 PodSpec：

```sh
$ kubectl get pod test-pod -o jsonpath='{.spec.overhead}'
map[cpu:250m memory:120Mi]
```

如果定义了 [ResourceQuata](https://kubernetes.io/zh-cn/docs/concepts/policy/resource-quotas/)，则容器 request 的总量以及 overhead 字段都将计算在内。

也就是说，当 kube-scheduler 决定在哪一个节点调度运行新的 Pod 时：

- 调度器会兼顾该 Pod 的 overhead 以及该 Pod 的容器请求总量。

在这个示例中，调度器将资源请求和开销相加，然后寻找具备 2.25 CPU 和 320 MiB 内存可用的节点。

一旦 Pod 被调度到了某个节点，该节点上的 kubelet 将为该 Pod 新建一个 cgroup，底层 Container Runtime 将在这个 Pod 中创建容器。

如果该资源对每一个容器都定义了一个限制（定义了限制值的 Guaranteed QoS 或者 Burstable QoS），kubelet 会为与该资源（CPU 的 cpu.cfs_quota_us 以及内存的 memory.limit_in_bytes） 相关的 Pod cgroup 设定一个上限。该上限基于 PodSpec 中定义的容器限制总量与 overhead 之和。

对于 CPU，如果 Pod 的 QoS 是 Guaranteed 或者 Burstable，kubelet 会基于容器请求总量与 PodSpec 中定义的 overhead 之和设置 cpu.shares。

请看这个例子，验证工作负载的容器请求：

```yaml
# 容器 request 总计 2000m CPU 和 200MiB 内存
$ kubectl get pod test-pod -o jsonpath='{.spec.containers[*].resources.limits}'
map[cpu: 500m memory:100Mi] map[cpu:1500m memory:100Mi]

# 对照从节点观察到的情况来检查一下：该输出显示请求了 2250m CPU 以及 320MiB 内存。request 包含了 Pod 开销在内。
$ kubectl describe node | grep test-pod -B2
Namespace    Name       CPU Requests  CPU Limits   Memory Requests  Memory Limits  AGE
  ---------    ----       ------------  ----------   ---------------  -------------  ---
  default      test-pod   2250m (56%)   2250m (56%)  320Mi (1%)       320Mi (1%)     36m
```

## 验证 Pod cgroup 限制

在工作负载所运行的节点上检查 Pod 的内存 cgroups。在接下来的例子中，将在该节点上使用具备 CRI 兼容的容器运行时命令行工具 crictl。这是一个显示 Pod 开销行为的高级示例，预计用户不需要直接在节点上检查 cgroups。

### 可观察性

在 kube-state-metrics 中可以通过 kube_pod_overhead_* 指标来协助确定何时使用 Pod 开销， 以及协助观察以一个既定开销运行的工作负载的稳定性。 该特性在 kube-state-metrics 的 1.9 发行版本中不可用，不过预计将在后续版本中发布。 在此之前，用户需要从源代码构建 kube-state-metrics。
