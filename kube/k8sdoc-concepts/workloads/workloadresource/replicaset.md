# Replicaset

ReplicaSet 的目的是维护一组在任何时候都处于运行状态的 Pod 副本的稳定集合。因此，它通常用来保证给定数量的、完全相同的 Pod 的可用性。

**注意：**

- 和 Deployment 不同，无法做到给 Pod 更新版本，只能确保 Pod 数量保持一致。

## ReplicaSet 的工作原理

ReplicaSet 是通过一组字段来定义的，包括：

- 一个用来识别可获得的 Pod 的集合的选择算符 Selector
- 一个用来标明应该维护的副本个数的数值
- 一个指定创建时要使用的 Pod 模板

每个 ReplicaSet 都通过根据需要创建和删除 Pod 以使得副本个数达到期望值，进而实现其存在价值。

当 ReplicaSet 需要创建新的 Pod 时，会使用所提供的 Pod 模板。

ReplicaSet 通过 Pod 上的 `metadata.ownerReferences` 字段连接到附属 Pod，该字段给出当前对象的属主资源。ReplicaSet 所获得的 Pod 都在其 ownerReferences 字段中包含了属主 ReplicaSet 的标识信息。正是通过这一连接，ReplicaSet 知道它所维护的 Pod 集合的状态，并据此计划其操作行为。

ReplicaSet 使用其选择算符来辨识要获得的 Pod 集合。如果某个 Pod 没有 OwnerReference 或者其 OwnerReference 不是一个控制器，且其匹配到某 ReplicaSet 的选择算符，则该 Pod 立即被此 ReplicaSet 设置。

## 何时使用 ReplicaSet

ReplicaSet 确保任何时间都有指定数量的 Pod 副本在运行。 然而，Deployment 是一个更高级的概念，它管理 ReplicaSet，并向 Pod 提供声明式的更新以及许多其他有用的功能。

因此：我们建议使用 Deployment 而不是直接使用 ReplicaSet，除非你需要自定义更新业务流程或根本不需要更新。

这实际上意味着，你可能永远不需要操作 ReplicaSet 对象：而是使用 Deployment，并在 spec 部分定义你的应用。

一个 rs 的示例：

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: frontend
  labels:
    app: guestbook
    tier: frontend
spec:
  # 按你的实际情况修改副本数
  replicas: 3
  selector:
    matchLabels:
      tier: frontend
  template:
    metadata:
      labels:
        tier: frontend
    spec:
      containers:
      - name: php-redis
        image: gcr.io/google_samples/gb-frontend:v3
```

**注意：**

- 这个 `gcr.io/google_samples/gb-frontend:v3` 镜像可能拉不下来，可以换一个。

测试：

```sh
$ kubectl apply -f https://kubernetes.io/examples/controllers/frontend.yaml
replicaset.apps/frontend created

$ kubectl get rs
NAME       DESIRED   CURRENT   READY   AGE
frontend   3         3         3       6s

$ kubectl get pods
NAME             READY   STATUS    RESTARTS   AGE
frontend-b2zdv   1/1     Running   0          6m36s
frontend-vcmts   1/1     Running   0          6m36s
frontend-wtsmm   1/1     Running   0          6m36s

# frontend ReplicaSet 的信息被设置在 metadata 的 ownerReferences 字段中
$ kubectl get pods frontend-b2zdv -o yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: "2020-02-12T07:06:16Z"
  generateName: frontend-
  labels:
    tier: frontend
  name: frontend-b2zdv
  namespace: default
  ownerReferences:
  - apiVersion: apps/v1
    blockOwnerDeletion: true
    controller: true
    kind: ReplicaSet
    name: frontend
    uid: f391f6db-bb9b-4c09-ae74-6a1f77f3d5cf
...
```

# 非模板 Pod 的获得

尽管你完全可以直接创建裸的 Pod，强烈建议你确保这些裸的 Pod 并不包含可能与你的某个 ReplicaSet 的选择算符相匹配的标签。

原因在于 ReplicaSet 并不仅限于拥有在其模板中设置的 Pod，它还可以像前面小节中所描述的那样获得其他 Pod。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod1
  labels:
    tier: frontend
spec:
  containers:
  - name: hello1
    image: gcr.io/google-samples/hello-app:2.0

---

apiVersion: v1
kind: Pod
metadata:
  name: pod2
  labels:
    tier: frontend
spec:
  containers:
  - name: hello2
    image: gcr.io/google-samples/hello-app:1.0
```

由于这些 Pod 没有控制器（Controller，或其他对象）作为其属主引用， 同时其标签与 frontend ReplicaSet 的 Label Selector 匹配，它们会立即被该 ReplicaSet 设置（rs 会主动告诉这些裸 pods 说：“我是你爸爸”）。

新的 Pod 会被该 ReplicaSet 获取，并立即被 ReplicaSet 终止，因为它们的存在会使得 ReplicaSet 中 Pod 个数超出其期望值。

```sh
# 先创建 RS，然后创建两个裸 Pods，且标签匹配
$ kubectl get pods
NAME             READY   STATUS        RESTARTS   AGE
frontend-b2zdv   1/1     Running       0          10m
frontend-vcmts   1/1     Running       0          10m
frontend-wtsmm   1/1     Running       0          10m
pod1             0/1     Terminating   0          1s
pod2             0/1     Terminating   0          1s
```

如果是先创建两个裸 Pods，后创建 RS：

```sh
$ kubectl get pods
NAME             READY   STATUS    RESTARTS   AGE
frontend-hmmj2   1/1     Running   0          9s
pod1             1/1     Running   0          36s
pod2             1/1     Running   0          36s
```

ReplicaSet 已经获得了该 Pod，并仅根据其规约创建新的 Pod， 直到新的 Pod 和原来的 Pod 的总数达到其预期个数。


## 编写 ReplicaSet 的清单

与所有其他 Kubernetes API 对象一样，ReplicaSet 也需要 apiVersion、kind、和 metadata 字段。 对于 ReplicaSet 而言，其 kind 始终是 ReplicaSet。

ReplicaSet 对象的名称必须是合法的 DNS 子域名。

ReplicaSet 也需要 .spec 部分。

### Pod 模板

`.spec.template` 是一个 Pod 模板， 要求设置标签

在 frontend.yaml 示例中，我们指定了标签 tier: frontend。 注意不要将标签与其他控制器的选择算符重叠，否则那些控制器会尝试收养此 Pod。

对于模板的重启策略 字段，`.spec.template.spec.restartPolicy`，**唯一允许的取值是 Always，这也是默认值**.

### Pod 选择算符

`.spec.selector` 字段是一个标签选择算符。 如前文中所讨论的，这些是用来标识要被获取的 Pod 的标签。在签名的 frontend.yaml 示例中，选择算符为：

```yaml
matchLabels:
  tier: frontend
```

在 ReplicaSet 中，`.spec.template.metadata.labels` 的值必须与 `spec.selector` 值相匹配，否则该配置会被 API 拒绝。

### Replicas

你可以通过设置 .spec.replicas 来指定要同时运行的 Pod 个数。 ReplicaSet 创建、删除 Pod 以与此值匹配。

如果你没有指定 .spec.replicas，那么默认值为 1。

## 使用 ReplicaSet

### 删除 ReplicaSet 和它的 Pod

要删除 ReplicaSet 和它的所有 Pod，使用 kubectl delete 命令。

默认情况下，垃圾收集器自动删除所有依赖的 Pod。

### 只删除 ReplicaSet

你可以只删除 ReplicaSet 而不影响它的各个 Pod，方法是使用 kubectl delete 命令并设置 `--cascade=orphan` 选项。

一旦删除了原来的 ReplicaSet，就可以创建一个新的来替换它。

由于新旧 ReplicaSet 的 .spec.selector 是相同的，新的 ReplicaSet 将接管老的 Pod。


### 将 Pod 从 ReplicaSet 中隔离

可以通过改变标签来从 ReplicaSet 中移除 Pod。 这种技术可以用来从服务中去除 Pod，以便进行排错、数据恢复等。

以这种方式移除的 Pod 将被自动替换（假设期望的副本数量没有改变）。

### 扩缩 ReplicaSet

通过更新 .spec.replicas 字段，ReplicaSet 可以被轻松地进行扩缩。

ReplicaSet 控制器能确保匹配标签选择器的数量的 Pod 是可用的和可操作的。

在降低集合规模时，ReplicaSet 控制器通过对可用的所有 Pod 进行排序来优先选择要被删除的那些 Pod。 其一般性算法如下：

1. 首先选择剔除悬决（Pending，且不可调度）的各个 Pod。
1. 如果设置了 `controller.kubernetes.io/pod-deletion-cost` 注解，则注解值较小的优先被裁减掉。
1. 所处节点上副本个数较多的 Pod 优先于所处节点上副本较少者。
1. 如果 Pod 的创建时间不同，最近创建的 Pod 优先于早前创建的 Pod 被裁减（尽可能保留老的，估计是为了保障稳定性）。

如果以上比较结果都相同，则随机选择。

### Pod 删除开销

通过使用 `controller.kubernetes.io/pod-deletion-cost` 注解，用户可以对 ReplicaSet 缩容时要先删除哪些 Pod 设置偏好。

此注解要设置到 Pod 上，取值范围为 `[-2147483647, 2147483647]`。所代表的是删除同一 ReplicaSet 中其他 Pod 相比较而言的开销。删除开销较小的 Pod 比删除开销较高的 Pod 更容易被删除。

Pod 如果未设置此注解，则隐含的设置值为 0。负值也是可接受的。如果注解值非法，API 服务器会拒绝对应的 Pod。

此功能特性处于 Beta 阶段，默认被启用。你可以通过为 kube-apiserver 和 kube-controller-manager 设置特性门控 `PodDeletionCost` 来禁用此功能。

**注意：**

- 此机制实施时仅是尽力而为，并不能对 Pod 的删除顺序作出任何保证；
- 用户应避免频繁更新注解值，例如根据某观测度量值来更新此注解值是应该避免的。**这样做会在 API 服务器上产生大量的 Pod 更新操作。**

使用场景示例：

- 同一应用的不同 Pod 可能其利用率是不同的。在对应用执行缩容操作时，可能希望移除利用率较低的 Pod。
- 为了避免频繁更新 Pod，应用应该在执行缩容操作之前更新一次 `controller.kubernetes.io/pod-deletion-cost` 注解值（将注解值设置为一个与其 Pod 利用率对应的值），尽量不要利用率变了该值就变，而是仅在需要修改 cost 时才修改。
- 如果应用自身控制器缩容操作时（例如 Spark 部署的驱动 Pod），这种机制是可以起作用的。

### ReplicaSet 作为水平的 Pod 自动扩缩器目标

ReplicaSet 也可以作为水平的 Pod 扩缩器 (HPA) 的目标。

也就是说，ReplicaSet 可以被 HPA 自动扩缩。 以下是 HPA 以我们在前一个示例中创建的副本集为目标的示例。

```sh
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: frontend-scaler
spec:
  scaleTargetRef:
    kind: ReplicaSet
    name: frontend
  minReplicas: 3
  maxReplicas: 10
  targetCPUUtilizationPercentage: 50
```

将这个列表保存到 hpa-rs.yaml 并提交到 Kubernetes 集群，就能创建它所定义的 HPA，进而就能根据复制的 Pod 的 CPU 利用率对目标 ReplicaSet 进行自动扩缩。

```sh
$ kubectl apply -f https://k8s.io/examples/controllers/hpa-rs.yaml
```

也有更简单的命令：

```sh
kubectl autoscale rs frontend --max=10 --min=3 --cpu-percent=50
```

## ReplicaSet 的替代方案

替代方案 | 描述
-|-
Deployment（推荐）| Deployment 是一个可以拥有 ReplicaSet 并使用声明式方式在服务器端完成对 Pod 滚动更新的对象。<br>尽管 ReplicaSet 可以独立使用，目前它们的主要用途是提供给 Deployment 作为编排 Pod 创建、删除和更新的一种机制。<br>当使用 Deployment 时，你不必关心如何管理它所创建的 ReplicaSet，Deployment 拥有并管理其 ReplicaSet。
裸 Pod(Bare Pods) | 与用户直接创建 Pod 的情况不同，ReplicaSet 会替换那些由于某些原因被删除或被终止的 Pod，例如在节点故障或破坏性的节点维护（如内核升级）的情况下。<br>因为这个原因，我们建议你使用 ReplicaSet，即使应用程序只需要一个 Pod。
Job | 使用Job 代替 ReplicaSet， 可以用于那些期望自行终止的 Pod。
DaemonSet | 对于管理那些提供主机级别功能（如主机监控和主机日志）的容器， 就要用 DaemonSet 而不用 ReplicaSet。<br>这些 Pod 的寿命与主机寿命有关：这些 Pod 需要先于主机上的其他 Pod 运行， 并且在机器准备重新启动/关闭时安全地终止。
ReplicationController | ReplicaSet 是 ReplicationController 的后继者。二者目的相同且行为类似，只是 ReplicationController 不支持[标签用户指南](https://kubernetes.io/zh-cn/docs/concepts/workloads/controllers/replicationcontroller/)中讨论的基于集合的选择算符需求。
