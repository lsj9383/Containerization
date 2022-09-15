# Statefulset

StatefulSet 是用来管理**有状态应用**的工作负载 API 对象。

StatefulSet 用来管理某 Pod 集合的部署和扩缩，并为这些 Pod **提供持久存储和持久标识符**。

和 Deployment 类似的点：

- StatefulSet 管理基于相同容器规约的一组 Pod。

和 Deployment 不同的点：

- StatefulSet 为它们的每个 Pod **维护了一个有粘性的 ID**。这些 Pod 是基于相同的规约来创建的，但是不能相互替换：无论怎么调度，每个 Pod 都有一个永久不变的 ID。

如果希望使用存储卷为工作负载提供持久存储，可以使用 StatefulSet 作为解决方案的一部分。尽管 StatefulSet 中的单个 Pod 仍可能出现故障， 但持久的 Pod 标识符使得将现有卷与替换已失败 Pod 的新 Pod 相匹配变得更加容易。

## 使用 StatefulSet

StatefulSet 对于需要满足以下一个或多个需求的应用程序很有价值：

- 稳定的、唯一的网络标识符。
- 稳定的、持久的存储。
- 有序的、优雅的部署和扩缩。
- 有序的、自动的滚动更新。

在上面描述中，“稳定的” 意味着 Pod 调度或重调度的整个过程是有持久性的。

如果应用程序不需要任何稳定的标识符或有序的部署、删除或扩缩，则应该使用由一组无状态的副本控制器提供的工作负载来部署应用程序，比如 Deployment 或者 ReplicaSet 可能更适用于你的无状态应用部署需要。

## 限制

- 给定 Pod 的存储必须由 PersistentVolume Provisioner 基于所请求的 storage class 来制备，或者由管理员预先制备。
- 删除或者扩缩 StatefulSet 并不会删除它关联的存储卷。这样做是为了保证数据安全，它通常比自动清除 StatefulSet 所有相关的资源更有价值。
- StatefulSet 当前需要[无头服务（Headless Service）](https://kubernetes.io/zh-cn/docs/concepts/services-networking/service/#headless-services)来负责 Pod 的网络标识。你需要负责创建此服务。
- 当删除一个 StatefulSet 时，该 StatefulSet 不提供任何终止 Pod 的保证。为了实现 StatefulSet 中的 Pod 可以有序且优雅地终止，可以在删除之前将 StatefulSet 缩容到 0。
- 在默认 Pod 管理策略(OrderedReady) 时使用滚动更新，可能进入需要人工干预才能修复的损坏状态。

## 组件

下面的示例演示了 StatefulSet 的组件：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None           # headless service configuration
  selector:
    app: nginx
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  selector:
    matchLabels:
      app: nginx # 必须匹配 .spec.template.metadata.labels
  serviceName: "nginx"
  replicas: 3 # 默认值是 1
  minReadySeconds: 10 # 默认值是 0
  template:
    metadata:
      labels:
        app: nginx # 必须匹配 .spec.selector.matchLabels
    spec:
      terminationGracePeriodSeconds: 10
      containers:
      - name: nginx
        image: registry.k8s.io/nginx-slim:0.8
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "my-storage-class"
      resources:
        requests:
          storage: 1Gi
```

上述例子中：

- 名为 nginx 的 Headless Service 用来控制网络域名。
- 名为 web 的 StatefulSet 有一个 Spec，它表明将在独立的 3 个 Pod 副本中启动 nginx 容器。
- volumeClaimTemplates 将通过 PersistentVolume 制备程序所准备的 PersistentVolumes 来提供稳定的存储。

StatefulSet 的命名需要遵循 DNS 子域名规范。

### Pod 选择算符

你必须设置 StatefulSet 的 `.spec.selector` 字段，使之匹配其在 `.spec.template.metadata.labels` 中设置的标签。

未指定匹配的 Pod 选择算符将在创建 StatefulSet 期间导致验证错误。

### 卷申领模板

你可以设置 `.spec.volumeClaimTemplates`，它可以使用 PersistentVolume 制备程序所准备的 PersistentVolumes 来提供稳定的存储。

### 最短就绪秒数

`.spec.minReadySeconds` 是一个可选字段。它指定新创建的 Pod 应该在没有任何容器崩溃的情况下运行并准备就绪，才能被认为是可用的。该字段默认为 0（Pod 准备就绪后将被视为可用）。 

这用于在使用滚动更新策略时检查滚动的进度。

## Pod 标识

StatefulSet Pod 具有唯一的标识，该标识包括：

- 顺序标识
- 稳定的网络标识
- 稳定的存储

该标识和 Pod 是绑定的，与该 Pod 调度到哪个节点上无关。

### 有序索引

对于具有 N 个副本的 StatefulSet，该 StatefulSet 中的每个 Pod 将被分配一个从 0 到 N-1 的整数序号，该序号在此 StatefulSet 上是唯一的。

Pod 的创建顺序也按这个索引顺序创建。

### 稳定的网络 ID

StatefulSet 中的每个 Pod 根据 StatefulSet 的名称和 Pod 的序号派生出它的主机名。

Pod 主机名的格式为：

```txt
$(StatefulSet 名称)-$(序号)
```

上例将会创建三个名称分别为 web-0、web-1、web-2 的 Pod（Pod 也是这个名字）。

StatefulSet 可以使用无头服务控制它的 Pod 的网络域。

管理域的这个服务的格式为：

```txt
$(服务名称).$(名字空间).svc.cluster.local
```

其中 `cluster.local` 是集群域。 

同时每个 Pod 创建成功，该 Pod 就会得到一个匹配的 DNS 子域，格式为：

```txt
$(pod 名称).$(所属服务的 DNS 域名)
```

其中所属服务由 StatefulSet 的 serviceName 域来设定（本质上就是在服务的域名前面加一个 pod 名称）。

取决于集群域内部 DNS 的配置，有可能无法查询一个刚刚启动的 Pod 的 DNS 命名。 当集群内其他客户端在 Pod 创建完成前发出 Pod 主机名查询时，就会发生这种情况。

如果需要在 Pod 被创建之后及时发现它们，可使用以下选项：

- 直接查询 Kubernetes API（比如，利用 watch 机制）而不是依赖于 DNS 查询
- 缩短 Kubernetes DNS 驱动的缓存时长（通常这意味着修改 CoreDNS 的 ConfigMap，目前缓存时长为 30 秒）

你需要负责创建无头服务以便为 Pod 提供网络标识。

下面给出一些选择集群域、服务名、StatefulSet 名、及其怎样影响 StatefulSet 的 Pod 上的 DNS 名称的示例：

集群域名 | 服务（名字空间/名字） | StatefulSet（名字空间/名字） | StatefulSet 域名 | Pod DNS | Pod 主机名
-|-|-|-|-|-
cluster.local | default/nginx | default/web | nginx.default.svc.cluster.local | web-{0..N-1}.nginx.default.svc.cluster.local | web-{0..N-1}
cluster.local | foo/nginx | foo/web	nginx.foo.svc.cluster.local | web-{0..N-1}.nginx.foo.svc.cluster.local | web-{0..N-1}
kube.local | foo/nginx | foo/web | nginx.foo.svc.kube.local | web-{0..N-1}.nginx.foo.svc.kube.local | web-{0..N-1}

### 稳定的存储

对于 StatefulSet 中定义的每个 VolumeClaimTemplate，每个 Pod 接收到一个 PersistentVolumeClaim。

在上面的 nginx 示例中，每个 Pod 将会得到基于 StorageClass my-storage-class 制备的 1 Gib 的 PersistentVolume。

如果没有声明 StorageClass，就会使用默认的 StorageClass。

当一个 Pod 被调度（重新调度）到节点上时，它的 volumeMounts 会挂载与其 PersistentVolumeClaims 相关联的 PersistentVolume。

**注意：**

- 当 Pod 或者 StatefulSet 被删除时，与 PersistentVolumeClaims 相关联的 PersistentVolume 并不会被删除。要删除它必须通过手动方式来完成。

### Pod 名称标签

当 StatefulSet 控制器（Controller） 创建 Pod 时，它会添加一个标签 statefulset.kubernetes.io/pod-name，该标签值设置为 Pod 名称。

这个标签允许你给 StatefulSet 中的特定 Pod 绑定一个 Service。

## 部署和扩缩保证

- 对于包含 N 个 副本的 StatefulSet，当部署 Pod 时，它们是依次创建的，顺序为 0..N-1。
- 当删除 Pod 时，它们是逆序终止的，顺序为 N-1..0。
- 在将扩缩操作应用到 Pod 之前，它前面的所有 Pod 必须是 Running 和 Ready 状态。
- 在一个 Pod 终止之前，所有的继任者必须完全关闭。

StatefulSet 不应将 pod.Spec.TerminationGracePeriodSeconds 设置为 0。这种做法是不安全的，要强烈阻止，更多这方面的讨论请参考 [强制删除 StatefulSet 中的 Pods](https://kubernetes.io/zh-cn/docs/tasks/run-application/force-delete-stateful-set-pod/)。

在上面的 nginx 示例被创建后，会按照 web-0、web-1、web-2 的顺序部署三个 Pod：

1. 在 web-0 进入 Running 和 Ready 状态前不会部署 web-1。
1. 在 web-1 进入 Running 和 Ready 状态前不会部署 web-2。 

**注意：**

- 如果 web-1 已经处于 Running 和 Ready 状态，而 web-2 尚未部署，在此期间发生了 web-0 运行失败，那么 web-2 将不会被部署，要等到 web-0 部署完成并进入 Running 和 Ready 状态后，才会部署 web-2。

如果用户想将示例中的 StatefulSet 扩缩为 replicas=1：首先被终止的是 web-2。 在 web-2 没有被完全停止和删除前，web-1 不会被终止。

**注意：**

- 当 web-2 已被终止和删除、web-1 尚未被终止，如果在此期间发生 web-0 运行失败， 那么就不会终止 web-1，必须等到 web-0 进入 Running 和 Ready 状态后才会终止 web-1。

### Pod 管理策略

StatefulSet 允许你放宽其排序保证， 同时通过它的 `.spec.podManagementPolicy` 域保持其唯一性和身份保证。

- OrderedReady Pod 管理，是 StatefulSet 的默认设置。 它实现了上面描述的功能。
- Parallel Pod 管理,让 StatefulSet 控制器并行的启动或终止所有的 Pod，启动或者终止其他 Pod 前，无需等待 Pod 进入 Running 和 ready 或者完全停止状态。
  - 这个选项只会影响扩缩操作的行为，更新则不会被影响。

## 更新策略

StatefulSet 的 `.spec.updateStrategy` 字段让你可以配置和禁用掉自动滚动更新 Pod 的容器、标签、资源请求或限制、以及注解。

有两个允许的值：

- OnDelete， 控制器将不会自动更新 StatefulSet 中的 Pod。用户必须手动删除 Pod 以便让控制器创建新的 Pod，以此来对 StatefulSet 的 .spec.template 的变动作出反应。
- RollingUpdate，对 StatefulSet 中的 Pod 执行自动的滚动更新。这是**默认的更新策略**。

### 滚动更新（RollingUpdate）

当 StatefulSet 的 .spec.updateStrategy.type 被设置为 RollingUpdate 时， StatefulSet 控制器会删除和重建 StatefulSet 中的每个 Pod：

- 它将按照与 Pod 终止相同的顺序（从最大序号到最小序号）进行，每次更新一个 Pod。

Kubernetes 控制平面会等到被更新的 Pod 进入 Running 和 Ready 状态，然后再更新其前身。 如果你设置了 .spec.minReadySeconds（查看最短就绪秒数）， 控制平面在 Pod 就绪后会额外等待一定的时间再执行下一步。