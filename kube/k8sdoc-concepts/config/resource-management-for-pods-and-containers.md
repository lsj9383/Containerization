# 为 Pod 和容器管理资源

当你定义 Pod 时可以**选择性**地为**每个容器**设定所需要的资源数量。

最常见的可设定资源是：

- CPU
- 内存（RAM）大小
- 此外还有其他类型的资源（例如 PV）

具体而言：

- 当你为 Pod 中的 Container 指定了资源 *请求（request）* 时， **kube-scheduler 就利用该信息决定将 Pod 调度到哪个节点上**。
- 当你为 Container 指定了资源 *限制（limit）* 时，**kubelet 就可以确保运行的容器不会使用超出所设限制的资源**。
- kubelet 还会为容器预留 *请求（request）* 数量的系统资源，供其使用。

## 请求（requests）和限制（limits）

如果 Pod 运行所在的节点具有足够的可用资源：

- 容器可能（且可以）使用超出对应资源 request 属性所设置的资源量。
- 不过，容器不可以使用超出其资源 limit 属性所设置的资源量。

例如：

- 如果你将容器的 memory 的 request 设置为 256 MiB，而该容器所处的 Pod 被调度到一个具有 8 GiB 内存的节点上，并且该节点上没有其他 Pod 运行，那么该容器就可以尝试使用更多的内存。
- 如果你将容器的 memory 的 limit设置为 4 GiB，kubelet 和 Container Runtime 就会确保该 limit 生效。
  - Container Runtime 会禁止容器使用超出所设置资源 limit 的资源。
  - 例如：当容器中进程尝试使用超出所允许内存量的资源时，系统内核会将尝试申请内存的进程终止，并引发内存不足（OOM）错误。

limit 可以以两种形式来试试：

- 被动方式来实现（系统会在发现违例时进行干预）
- 通过强制生效的方式实现（系统会避免容器用量超出限制）

具体 limit 是什么行为，不同的 Container Runtime 采用不同方式来实现相同的限制。

**注意：**

- 如果你为某个资源指定了 limit，但不指定 request，并且没有应用准入时机制为该资源设置默认请求，那么 Kubernetes 复制你所指定的 limit，将其用作资源的 request。

## 资源类型（Resource types）

CPU 和内存都是资源类型（Resource types），每种资源类型具有其基本单位：

- CPU 表达的是计算处理能力，其单位是 [Kubernetes CPU](https://kubernetes.io/zh-cn/docs/concepts/configuration/manage-resources-containers/#meaning-of-cpu)。
- 内存的单位是字节。对于 Linux 负载，则可以指定巨页（Huge Page）资源。 巨页是 Linux 特有的功能，节点内核在其中分配的内存块比默认页大小大得多。

CPU 和内存统称为**计算资源**，或简称为**资源**，这些计算资源的数量是**可测量的**，也是可以被请求、被分配、被消耗。

计算资源与 API 资源不同，API 资源（如 Pod 和 Service）是可通过 Kubernetes API 服务器读取和修改的对象。

## Pod 和 容器的资源请求和限制

针对每个容器，你都可以指定其资源限制和请求，包括如下选项：

```txt
spec.containers[].resources.limits.cpu
spec.containers[].resources.limits.memory
spec.containers[].resources.limits.hugepages-<size>
spec.containers[].resources.requests.cpu
spec.containers[].resources.requests.memory
spec.containers[].resources.requests.hugepages-<size>
```

尽管你只能逐个容器地指定 requests 和 limits，z这对集群计算 Pod 的总体资源请求和限制也是有用的。

对某个 Pod 资源而言，Pod 的资源 request/limits 是 Pod 中各容器的 requests/limits 的总和。

## Kubernetes 中的资源单位

计算资源的单位：

- CPU 资源单位
- 内存资源单位

### CPU 资源单位

CPU 资源的限制和请求以 “cpu” 为单位。在 Kubernetes 中，一个 CPU 等于 1 个物理 CPU 核，或者 1 个虚拟核，这取决于节点是一台物理主机还是运行在某物理主机上的虚拟机。

你也可以表达带小数 CPU 的请求，例如：

- 当你定义一个容器，将其 spec.containers[].resources.requests.cpu 设置为 0.5
- 你所请求的 CPU 是你请求 1.0 CPU 时的一半

对于 CPU 资源单位，数量表达式 0.1 等价于表达式 100m，可以看作 “100 millicpu”。有些人说成是“一百毫核”，其实说的是同样的事情。

**注意：**

- 1.0 CPU = 1000m CPU（这里 m 是毫的意思，类似 1s = 1000ms）。
- CPU 资源总是设置为资源的**绝对数量**而非相对数量值。例如，无论容器运行在单核、双核或者 48-核的机器上，500m CPU 表示的是大约相同的计算能力。
- Kubernetes 不允许设置精度小于 1m 的 CPU 资源。 因此，当 CPU 单位小于 1 或 1000m 时，使用毫核的形式是有用的； 例如 5m 而不是 0.005。

### 内存资源单位

memory 的限制和请求以**字节**为单位。

你可以使用普通的整数，或者带有以下 **数量后缀** 的定点数字来表示内存：E、P、T、G、M、k。 你也可以使用对应的 2 的幂数：Ei、Pi、Ti、Gi、Mi、Ki。 例如，以下表达式所代表的是大致相同的值：

128974848、129e6、129M、128974848000m、123Mi
请注意后缀的大小写。如果你请求 400m 临时存储，实际上所请求的是 0.4 字节。 如果有人这样设定资源请求或限制，可能他的实际想法是申请 400Mi 字节（400Mi） 或者 400M 字节。

## 容器资源示例

以下 Pod 有两个容器：

- 每个容器的 requests 为 0.25 CPU 和 64MiB（226 字节）内存
- 每个容器的 limit 为 0.5 CPU 和 128MiB 内存。

换句话说，你可以认为该 Pod 的资源请求为 0.5 CPU 和 128 MiB 内存，资源限制为 1 CPU 和 256MiB 内存。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: frontend
spec:
  containers:
  - name: app
    image: images.my-company.example/app:v4
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
  - name: log-aggregator
    image: images.my-company.example/log-aggregator:v6
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
```

## 带资源请求的 Pod 如何调度

当你创建一个 Pod 时，Kubernetes 调度程序将为 Pod 选择一个节点，选择节点的根本逻辑是：

- 每个节点对每种资源类型都会计算一个容量上限：可以为 Pod 提供的 CPU 总量和 Memory 总量。
- Kubernetes 调度程序（kube-scheduler）确保对于每种资源类型，所调度的容器的资源 request 的总和小于节点的容量。

**注意：**

- 尽管节点上的实际内存或 CPU 资源使用量非常低，如果容量检查失败，调度程序仍会拒绝在该节点上放置 Pod。

当稍后节点上资源用量增加，例如到达请求率的每日峰值区间时（在 request 内），节点上也不会出现资源不足的问题。

**注意：**

- 当超过 request，就可能让节点出现资源不足的问题。
- 超过 request 但小于 limit 的情况
  - 对于 CPU 而言，没有什么影响，对于内存而言。
  - 如果节点内存不足，则 Pod 会被驱逐。
- 超过 limit 的情况
  - 对于 CPU 而言，无法占用更多的 CPU 时间了，但是不会杀死容器也不会驱逐 Pod。
  - 对于内存而言，会将容器 OOM。如果节点内存不足，则 Pod 会驱逐。

## Kubernetes 应用资源 request 与 limit 的方式

当 kubelet 将容器作为 Pod 的一部分启动时，它会将容器的 CPU 和内存 requests 与 limits 传递给 Container Runtime。在 Linux 系统上，Container Runtime 通常会配置内核 CGroups，负责应用并实施所定义的约束。

对于 CPU 的 request 和 limit 的实施（这里是调度完成后的实施）：

- CPU requests 定义的是一个权重值。如果若干不同的容器（CGroups）需要在一个共享的系统上竞争运行，CPU request 大的负载会获得比 request 小的负载更多的 CPU 时间。
- CPU limits 定义的是容器可使用的 CPU 的硬性上限。在每个调度周期（时间片）期间，Linux 内核检查是否已经超出该限制；内核会在允许该 cgroup 恢复执行之前会等待。

对于 Memory 的 request 和 limit 的实施：

- 内存 request 主要用于 Pod 调度期间。在调度完成后，对于一个启用了 CGroup v2 的节点上，Container Runtime 可能会使用内存 request 作为设置 memory.min 和 memory.low 的**提示值**（request 只是提示值，主要用处就是调度和预留空间）。
- 内存 limit 定义的是 cgroup 的内存限制。如果容器尝试分配的内存量超出 limit，则 Linux 内核的内存不足处理子系统会被激活，并停止尝试分配内存的容器中的某个进程。 如果该进程在容器中 PID 为 1，而容器被标记为可重新启动，则 Kubernetes 会重新启动该容器。
- Pod 或容器的内存限制也适用于通过内存提供的 Volume，例如 emptyDir 卷。kubelet 会跟踪 tmpfs 形式的 emptyDir 卷用量，将其作为容器的内存用量， 而不是临时存储用量。

**注意：**

- 如果某容器内存用量超过其内存 request 并且所在节点内存不足时（即便没有达到 limit），容器所处的 Pod 可能被**驱逐**。
- 每个容器可能被允许也可能不被允许使用超过其 CPU 限制的处理时间。但是，容器运行时不会由于 CPU 使用率过高而杀死 Pod 或容器。
- 要确定某容器是否会由于资源限制而无法调度或被杀死，请参[阅疑难解答](https://kubernetes.io/zh-cn/docs/concepts/configuration/manage-resources-containers/#troubleshooting)。
  
另外，驱逐条件主要看 kubelet 的启动参数，这是一个实例：

```sh
ps aux|grep kubelet|grep evi
root       29131  2.4  5.1 1954180 91232 ?       Ssl  Sep08 370:40 /usr/bin/kubelet --anonymous-auth=false --authentication-token-webhook=true --authorization-mode=Webhook --client-ca-file=/etc/kubernetes/cluster-ca.crt --cloud-config=/etc/kubernetes/qcloud.conf --cloud-provider=external --cluster-dns=172.16.254.137 --cluster-domain=cluster.local --eviction-hard=nodefs.available<10%,nodefs.inodesFree<5%,imagefs.available<15%,memory.available<100Mi --fail-swap-on=false --feature-gates=CSIMigration=true,CSIMigrationQcloudCbs=true,CSIMigrationQcloudCbsComplete=true --hostname-override=10.0.1.2 --image-pull-progress-deadline=10m0s --kube-reserved=cpu=100m,memory=512Mi --kubeconfig=/etc/kubernetes/kubelet-kubeconfig --max-pods=61 --network-plugin=cni --non-masquerade-cidr=0.0.0.0/0 --pod-infra-container-image=ccr.ccs.tencentyun.com/library/pause:latest --provider-id=qcloud:///100003/ins-la4y1b7u --register-schedulable=true --register-with-taints=tke.cloud.tencent.com/uninitialized=true:NoSchedule --serialize-image-pulls=false --v=2
```

## 监控计算和内存资源用量

kubelet 会将 Pod 的资源使用情况作为 Pod status 的一部分来报告的。

如果为集群配置了可选的监控工具，则可以直接从指标 API 或者监控工具获得 Pod 的资源使用情况。
