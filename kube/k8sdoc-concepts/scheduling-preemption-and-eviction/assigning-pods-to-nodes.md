# 将 Pod 指派给节点

你可以约束一个 Pod 以便限制其能够：

- **只能**在特定的节点上运行
- **优先**在特定的节点上运行

有几种方法可以实现这点：

- 推荐的方法都是用 Label Selector 来进行选择。

通常这样的约束不是必须的，因为调度器将自动进行合理的放置（比如，将 Pod 分散到节点上，而不是将 Pod 放置在可用资源不足的节点上等等）。但在某些情况下，你可能需要进一步控制 Pod 被部署到哪个节点。例如：

- 确保 Pod 最终落在连接了 SSD 的机器上。
- 或者将来自两个不同的服务且有大量通信的 Pods 被放置在同一个可用区节点上。

你可以使用下列方法中的任何一种来选择 Kubernetes 对特定 Pod 的调度：

- 与节点标签匹配的 [nodeSelector](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#built-in-node-labels)
- [亲和性与反亲和性](https://kubernetes.io/zh-cn/docs/concepts/scheduling-eviction/assign-pod-node/#affinity-and-anti-affinity)
- nodeName 字段
- [Pod 拓扑分布约束](https://kubernetes.io/zh-cn/docs/concepts/scheduling-eviction/assign-pod-node/#pod-topology-spread-constraints)

## 节点标签

节点与很多其他 Kubernetes 对象类似，节点也有 Label：

- 你可以手动地添加标签
- Kubernetes 也会为集群中所有节点添加一些标准的标签

参考[常用的标签、注解和污点](https://kubernetes.io/zh-cn/docs/reference/labels-annotations-taints/)以了解常见的节点标签。

**注意：**

- 这些标签的取值是取决于云提供商的，并且是无法在可靠性上给出承诺的。 例如，kubernetes.io/hostname 的取值在某些环境中可能与节点名称相同， 而在其他环境中会取不同的值。

## 节点隔离/限制

通过节点隔离限制功能，你可以确保特定的 Pod 只能运行在具有一定隔离性，安全性或监管属性的节点上。

如果使用 Label 来实现节点的隔离，请选择 kubelet 无法修改的 Label。这可以防止受损节点在其自身上设置这些标签，以便调度程序将工作负载调度到受损节点上。

那怎么让 kubelet 修改 Label 呢？

- [NodeRestriction 准入插件](https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/admission-controllers/#noderestriction) 可以防止 kubelet 设置或修改带有 `node-restriction.kubernetes.io/` 前缀的标签。

要使用该标签前缀进行节点隔离：

- 确保你在使用节点鉴权机制并且已经启用了 NodeRestriction 准入插件。
- 将带有 `node-restriction.kubernetes.io/` 前缀的标签添加到 Node 对象，然后在 Pod 的 nodeSelector 中使用这些标签。

## nodeSelector

nodeSelector 是节点选择约束的最简单且推荐的形式。

你可以将 nodeSelector 字段添加到 Pod Spec 中设置你希望目标节点所具有的**节点标签**。Kubernetes 只会将 Pod 调度到拥有你所指定的每个标签的节点上。

例如：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    env: test
spec:
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
  nodeSelector:
    disktype: ssd
```

**注意：**

- 这种做法并不一定安全，建议标签使用 `node-restriction.kubernetes.io/`，这样可以防止标签被修改。

## 亲和性与反亲和性

nodeSelector 提供了一种最简单的方法来将 Pod 约束到具有特定标签的节点上。

亲和性和反亲和性扩展了你可以定义的约束类型。使用亲和性与反亲和性的一些好处有：

- 亲和性、反亲和性语言的表达能力更强。nodeSelector 只能选择拥有所有指定标签的节点。 亲和性、反亲和性为你提供对选择逻辑的更强控制能力。
- 你可以标明某规则是 “软需求” 或者 “偏好”，这样**调度器在无法找到匹配节点时仍然调度该 Pod**。
- 你可以使用节点上（或其他拓扑域中）运行的其他 Pod 的标签来实施调度约束，而不是只能使用节点本身的标签。这个能力让你能够定义规则允许哪些 Pod 可以被放置在一起。

亲和性功能由两种类型的亲和性组成：

- **节点亲和性功能**类似于 nodeSelector 字段，但它的表达能力更强，并且允许你指定软规则。
- **Pod 间亲和性/反亲和性**允许你根据其他 Pod 的标签来约束 Pod。

### 节点亲和性

节点亲和性概念上类似于 nodeSelector，它使你可以根据节点上的标签来约束 Pod 可以调度到哪些节点上。

节点亲和性有两种：

- **requiredDuringSchedulingIgnoredDuringExecution**，调度器只有在规则被满足的时候才能执行调度。此功能类似于 nodeSelector，但其语法表达能力更强。
- **preferredDuringSchedulingIgnoredDuringExecution**，调度器会尝试寻找满足对应规则的节点。如果找不到匹配的节点，调度器仍然会调度该 Pod。

**注意：**

- 在上述类型中，IgnoredDuringExecution 意味着如果节点标签在 Kubernetes 调度 Pod 时发生了变更，Pod 仍将继续运行（也就是即便节点标签不满足 Pod 条件了，也不会驱逐 Pod）。

你可以使用 Pod 规约中的 `.spec.affinity.nodeAffinity` 字段来设置节点亲和性。

例如，考虑下面的 Pod 规约：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: with-node-affinity
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: topology.kubernetes.io/zone
            operator: In
            values:
            - antarctica-east1
            - antarctica-west1
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: another-node-label-key
            operator: In
            values:
            - another-node-label-value
  containers:
  - name: with-node-affinity
    image: k8s.gcr.io/pause:2.0
```

在这一示例中，所应用的规则如下：

- 节点**必须**包含一个键名为 `topology.kubernetes.io/zone` 的标签，并且该标签的取值必须为 antarctica-east1 或 antarctica-west1。
- 节点**最好**具有一个键名为 `another-node-label-key` 且取值为 another-node-label-value 的标签。

NotIn 和 DoesNotExist 可用来实现节点反亲和性行为。

**注意：**

- 如果你同时指定了 nodeSelector 和 nodeAffinity，两者必须都要满足，才能将 Pod 调度到候选节点上。

#### 节点亲和性权重

你可以为 `preferredDuringSchedulingIgnoredDuringExecution` 亲和性类型的每个实例设置 `weight` 字段，其取值范围是 1 到 100。

具体而言：

- 当调度器找到能够满足 Pod 的其他调度请求的节点时，调度器会遍历节点满足的所有的偏好性规则，并将对应表达式的 weight 值求和。
- 最终的求和值会添加到该节点的其他优先级函数的评分之上。
- 在调度器为 Pod 作出调度决定时，总分最高的节点的优先级也最高。

例如，考虑下面的 Pod 规约：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: with-affinity-anti-affinity
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: kubernetes.io/os
            operator: In
            values:
            - linux
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: label-1
            operator: In
            values:
            - key-1
      - weight: 50
        preference:
          matchExpressions:
          - key: label-2
            operator: In
            values:
            - key-2
  containers:
  - name: with-node-affinity
    image: k8s.gcr.io/pause:2.0
```

如果存在两个候选节点，都满足 preferredDuringSchedulingIgnoredDuringExecution 规则：

- 其中一个节点具有标签 label-1:key-1
- 另一个节点具有标签 label-2:key-2
- 调度器会考察各个节点的 weight 取值，并将该权重值添加到节点的其他得分值之上。

**注意：**

- 如果你希望 Kubernetes 能够成功地调度此例中的 Pod，你必须拥有打了 `kubernetes.io/os=linux` 标签的节点。

#### 逐个调度方案中设置节点亲和性

在配置[多个调度方案](https://kubernetes.io/zh-cn/docs/reference/scheduling/config/#multiple-profiles)时，你可以将某个方案与节点亲和性关联起来，如果某个调度方案仅适用于某组特殊的节点时，这样做是很有用的。

要实现这点，可以在调度器配置中为 NodeAffinity 插件的 args 字段添加 addedAffinity。例如：

```yaml
apiVersion: kubescheduler.config.k8s.io/v1beta3
kind: KubeSchedulerConfiguration

profiles:
  - schedulerName: default-scheduler
  - schedulerName: foo-scheduler
    pluginConfig:
      - name: NodeAffinity
        args:
          addedAffinity:
            requiredDuringSchedulingIgnoredDuringExecution:
              nodeSelectorTerms:
              - matchExpressions:
                - key: scheduler-profile
                  operator: In
                  values:
                  - foo
```

这里的 addedAffinity 除遵从 Pod 规约中设置的节点亲和性之外，还适用于将 `.spec.schedulerName` 设置为 foo-scheduler。

换言之，为了匹配 Pod，节点需要满足：

- 调度方案中的 addedAffinity
- 以及 Pod 的 `.spec.NodeAffinity`

由于 addedAffinity 对最终用户不可见，其行为可能对用户而言是出乎意料的。 应该使用与调度方案名称有明确关联的节点标签。

**注意：**

- DaemonSet Controller 为 DaemonSet 创建 Pods，但该控制器不理会调度方案。
- DaemonSet Controller 创建 Pod 时，默认的 Kubernetes 调度器负责放置 Pod，并遵从 DaemonSet 控制器中的 nodeAffinity 规则。

### Pod 间亲和性与反亲和性的类型

与节点亲和性类似，Pod 的亲和性与反亲和性也有两种类型：

- **requiredDuringSchedulingIgnoredDuringExecution**，必须的。
- **preferredDuringSchedulingIgnoredDuringExecution**，优先的。

例如：

- 你可以使用 requiredDuringSchedulingIgnoredDuringExecution 亲和性来告诉调度器，将两个服务的 Pod 放到同一个云提供商可用区内，因为它们彼此之间通信非常频繁。
- 类似地，你可以使用 preferredDuringSchedulingIgnoredDuringExecution 反亲和性来将同一服务的多个 Pod 分布到多个云提供商可用区中。

使用方式：

- 对于 Pod 间亲和性，可以使用 Pod 规约中的 `.affinity.podAffinity` 字段。
- 对于 Pod 间反亲和性，可以使用 Pod 规约中的 `.affinity.podAntiAffinity` 字段。

#### Pod 亲和性示例

考虑下面的 Pod 规约：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: with-pod-affinity
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: security
            operator: In
            values:
            - S1
        topologyKey: topology.kubernetes.io/zone
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
            - key: security
              operator: In
              values:
              - S2
          topologyKey: topology.kubernetes.io/zone
  containers:
  - name: with-pod-affinity
    image: k8s.gcr.io/pause:2.0
```

本示例定义了一条 Pod 亲和性规则和一条 Pod 反亲和性规则：

- Pod 亲和性规则配置为 requiredDuringSchedulingIgnoredDuringExecution
- Pod 反亲和性配置为 preferredDuringSchedulingIgnoredDuringExecution

亲和性规则表示：

- 仅当节点和至少一个已运行且有 `security=S1` 的标签的 Pod 处于同一区域时，才可以将该 Pod 调度到节点上。
- 更确切的说，调度器必须将 Pod 调度到具有 topology.kubernetes.io/zone=V 标签的节点上，并且集群中至少有一个位于该可用区的节点上运行着带有 security=S1 标签的 Pod。

反亲和性规则表示：

- 如果节点处于 Pod 所在的同一可用区且至少一个 Pod 具有 security=S2 标签，则该 Pod 不应被调度到该节点上。
- 更确切地说，如果同一可用区中存在其他运行着带有 security=S2 标签的 Pod 节点， 并且节点具有标签 topology.kubernetes.io/zone=R，Pod 不能被调度到该节点上。

原则上，topologyKey 可以是任何合法的标签键。出于性能和安全原因，topologyKey 有一些限制。

#### 名字空间选择算符

用户也可以使用 namespaceSelector 选择匹配的名字空间，namespaceSelector 是对名字空间集合进行标签查询的机制。 亲和性条件会应用到 namespaceSelector 所选择的名字空间和 namespaces 字段中所列举的名字空间之上。 注意，空的 namespaceSelector（{}）会匹配所有名字空间，而 null 或者空的 namespaces 列表以及 null 值 namespaceSelector 意味着“当前 Pod 的名字空间”。

#### 更实际的用例

Pod 间亲和性与反亲和性在与更高级别的集合（例如 ReplicaSet、StatefulSet、 Deployment 等）一起使用时，它们可能更加有用。 这些规则使得你可以配置一组工作负载，使其位于所定义的同一拓扑中； 例如优先将两个相关的 Pod 置于相同的节点上。

以一个三节点的集群为例。你使用该集群运行一个带有内存缓存（例如 Redis）的 Web 应用程序。 在此例中，还假设 Web 应用程序和内存缓存之间的延迟应尽可能低。 你可以使用 Pod 间的亲和性和反亲和性来尽可能地将该 Web 服务器与缓存并置。

在下面的 Redis 缓存 Deployment 示例中：

- 副本上设置了标签 app=store
- podAntiAffinity 规则告诉调度器避免将多个带有 app=store 标签的副本部署到同一节点上。因此，每个独立节点上会创建一个缓存实例。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-cache
spec:
  selector:
    matchLabels:
      app: store
  replicas: 3
  template:
    metadata:
      labels:
        app: store
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - store
            topologyKey: "kubernetes.io/hostname"
      containers:
      - name: redis-server
        image: redis:3.2-alpine
```

下例的 Deployment ：

- 为 Web 服务器创建带有标签 app=web-store 的副本
- Pod 亲和性规则告诉调度器将每个副本放到存在标签为 app=store 的 Pod 的节点上。
- Pod 反亲和性规则告诉调度器决不要在单个节点上放置多个 app=web-store 服务器。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-server
spec:
  selector:
    matchLabels:
      app: web-store
  replicas: 3
  template:
    metadata:
      labels:
        app: web-store
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - web-store
            topologyKey: "kubernetes.io/hostname"
        podAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - store
            topologyKey: "kubernetes.io/hostname"
      containers:
      - name: web-app
        image: nginx:1.16-alpine
```

创建前面两个 Deployment 会产生如下的集群布局，每个 Web 服务器与一个缓存实例并置， 并分别运行在三个独立的节点上。

node-1 | node-2 | node-3
-|-|-
webserver-1 | webserver-2 | webserver-3
cache-1 | cache-2 | cache-3

总体效果是每个缓存实例都非常可能被在同一个节点上运行的某个客户端访问。这种方法旨在最大限度地减少偏差（负载不平衡）和延迟。

你可能还有使用 Pod 反亲和性的一些其他原因。 参阅 [ZooKeeper 教程](https://kubernetes.io/zh-cn/docs/tutorials/stateful-application/zookeeper/#tolerating-node-failure) 了解一个 StatefulSet 的示例，该 StatefulSet 配置了反亲和性以实现高可用， 所使用的是与此例相同的技术。

## nodeName

nodeName 是比亲和性或者 nodeSelector 更为直接的形式：

- nodeName 是 Pod Spec 的一个字段。
- 如果 nodeName 字段不为空，调度器会忽略该 Pod，而指定节点上的 kubelet 会尝试将 Pod 放到该节点上。

**注意：**

- 使用 nodeName 规则的优先级会高于使用 nodeSelector 或亲和性与非亲和性的规则。

使用 nodeName 来选择节点的方式有一些局限性：

- 如果所指代的节点不存在，则 Pod 无法运行，而且在某些情况下可能会被自动删除。
- 如果所指代的节点无法提供用来运行 Pod 所需的资源，Pod 会失败，而其失败原因中会给出是否因为内存或 CPU 不足而造成无法运行。
- 在云环境中的节点名称并不总是可预测的，也不总是稳定的。
- 下面是一个使用 nodeName 字段的 Pod 规约示例：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx
  nodeName: kube-01
```

上面的 Pod 只能运行在节点 kube-01 之上。

## Pod 拓扑分布约束

你可以使用**拓扑分布约束（Topology Spread Constraints）**来控制 Pod 在集群内故障域之间的分布，故障域的示例有：

- 区域（Region）
- 可用区（Zone）
- 节点
- 其他用户自定义的拓扑域

这样做有助于提升性能、实现高可用或提升资源利用率。

阅读 [Pod 拓扑分布约束](https://kubernetes.io/zh-cn/docs/concepts/scheduling-eviction/topology-spread-constraints/)以进一步了解这些约束的工作方式。

