# Endpoint Slices（端点切片）

端点切片（EndpointSlices） 提供了一种简单的方法来跟踪 Kubernetes 集群中的网络端点（network endpoints）。

它们为 Endpoints 提供了一种可扩缩和可拓展的替代方案。

## 动机

Endpoints API 提供了在 Kubernetes 中跟踪网络端点的一种简单而直接的方法。遗憾的是，随着 Kubernetes 集群和服务逐渐开始为更多的后端 Pod 处理和发送请求， 原来的 Endpoints API 的局限性变得越来越明显。最重要的是那些因为要处理大量网络端点而带来的挑战。

由于任一 Service 的所有网络端点都保存在同一个 Endpoints 资源中，这类资源可能变得非常巨大，而这一变化会影响到 Kubernetes 组件（比如主控组件）的性能，并在 Endpoints 变化时产生大量的网络流量和额外的处理。

EndpointSlice 能够帮助你缓解这一问题，还能为一些诸如拓扑路由这类的额外功能提供一个可扩展的平台。

## EndpointSlice 资源

在 Kubernetes 中，EndpointSlice 包含对一组 Endpoints 对象的引用。

Control Panel 会自动为设置了选择算符的 Kubernetes Service 创建 EndpointSlice。

EndpointSlice 通过唯一的协议、端口号和 Service 名称将网络端点组织在一起。 EndpointSlice 的名称必须是合法的 DNS 子域名。

例如，下面是 Kubernetes Service example 的 EndpointSlice 资源示例：

```yaml
apiVersion: discovery.k8s.io/v1
kind: EndpointSlice
metadata:
  name: example-abc
  labels:
    kubernetes.io/service-name: example
addressType: IPv4
ports:
  - name: http
    protocol: TCP
    port: 80
endpoints:
  - addresses:
      - "10.1.2.3"
    conditions:
      ready: true
    hostname: pod-1
    nodeName: node-1
    zone: us-west2-a
```

默认情况下，Control Panel 创建和管理的 EndpointSlice 将包含不超过 100 个端点。

你可以使用 kube-controller-manager 的 `--max-endpoints-per-slice` 标志设置此值，最大值为 1000。

当涉及如何路由内部流量时，EndpointSlice 可以充当 kube-proxy 的决策依据。启用该功能后，在服务的端点数量庞大时会有可观的性能提升。

### 地址类型

EndpointSlice 支持三种地址类型：

- IPv4
- IPv6
- FQDN (完全合格的域名)

### 状况

EndpointSlice API 存储了可能对使用者有用的、有关 Endpoint 的状况。 这三个状况分别是：

- ready
- serving
- terminating

```sh
$ kubectl get endpointslices my-nginx-n65sg -o yaml
...
endpoints:
- addresses:
  - 172.16.0.43
  conditions:
    ready: true
    serving: true
    terminating: false
  nodeName: 10.0.1.2
- addresses:
  - 172.16.0.83
  conditions:
    ready: true
    serving: true
    terminating: false
  nodeName: 10.0.1.10
...
```

#### Ready（就绪）

ready 状况是映射 Pod 的 Ready 状况的。

对于处于运行中的 Pod，它的 Ready 状况被设置为 True，应该将此 EndpointSlice 状况也设置为 true。出于兼容性原因，当 Pod 处于终止过程中，ready 永远不会为 true。

#### Serving（服务中）

serving 状况与 ready 状况相同，不同之处在于它不考虑终止状态。 如果 EndpointSlice API 的使用者关心 Pod 终止时的就绪情况，就应检查此状况。

**注意：**

- 尽管 serving 与 ready 几乎相同，但是它是为防止破坏 ready 的现有含义而增加的。
- 如果对于处于终止中的端点，ready 可能是 true，那么对于现有的客户端来说可能是有些意外的， 因为从始至终，Endpoints 或 EndpointSlice API 从未包含处于终止中的端点。 出于这个原因，ready 对于处于终止中的端点总是 false， 并添加了新的状况 serving，以便客户端可以独立于 ready 的现有语义来跟踪处于终止中的 Pod 的就绪情况。

### Terminating（终止中）

Terminating 是表示端点是否处于终止中的状况。 对于 Pod 来说，这是设置了删除时间戳的 Pod。

### 拓扑信息

EndpointSlice 中的每个端点都可以包含一定的拓扑信息。 拓扑信息包括端点的位置，对应节点、可用区的信息。 这些信息体现为 EndpointSlices 的如下端点字段：

- nodeName - 端点所在的 Node 名称；
- zone - 端点所处的可用区。

### 管理

通常，Control Panel 会创建和管理 EndpointSlice 对象。EndpointSlice 对象还有一些其他使用场景， 例如作为服务网格（Service Mesh）的实现。 这些场景都会导致有**其他实体或者控制器**负责管理额外的 EndpointSlice 集合。

为了确保多个实体可以管理 EndpointSlice 而且不会相互产生干扰， Kubernetes 定义了标签 `endpointslice.kubernetes.io/managed-by`，用来标明哪个实体在管理某个 EndpointSlice。

Endpoint Slices Controller 会在自己所管理的所有 EndpointSlice 上将该标签值设置为 `endpointslice-controller.k8s.io`。 管理 EndpointSlice 的其他实体也应该为此标签设置一个唯一值：

```sh
$ kubectl get endpointslices my-nginx-n65sg -o yaml
...
metadata:
  labels:
    endpointslice.kubernetes.io/managed-by: endpointslice-controller.k8s.io
    kubernetes.io/service-name: my-nginx
    run: my-nginx
...
```

### 属主关系

在大多数场合下，EndpointSlice 都由某个 Service 所有，（因为）该端点切片正是为该服务跟踪记录其端点。

这一属主关系是通过为每个 EndpointSlice 设置一个属主（owner）引用，同时设置 `kubernetes.io/service-name` 标签来标明的，目的是方便查找隶属于某 Service 的所有 EndpointSlice。

```sh
$ kubectl get endpointslices my-nginx-n65sg -o yaml
...
metadata:
  labels:
    endpointslice.kubernetes.io/managed-by: endpointslice-controller.k8s.io
    kubernetes.io/service-name: my-nginx
    run: my-nginx
  name: my-nginx-n65sg
  namespace: default
  ownerReferences:
  - apiVersion: v1
    blockOwnerDeletion: true
    controller: true
    kind: Service
    name: my-nginx
    uid: 2f4db209-108d-4895-a451-08c13c8ed475
  resourceVersion: "6220855599"
  selfLink: /apis/discovery.k8s.io/v1/namespaces/default/endpointslices/my-nginx-n65sg
  uid: 522a11c2-0418-4165-876b-5efcd578cc01
ports:
- name: ""
  port: 80
  protocol: TCP
```

### EndpointSlice 镜像

在某些场合，应用会创建定制的 Endpoints 资源。为了保证这些应用不需要并发的更改 Endpoints 和 EndpointSlice 资源，集群的控制面将大多数 Endpoints 映射到对应的 EndpointSlice 之上。

控制面对 Endpoints 资源进行映射的例外情况有：

- Endpoints 资源上标签 endpointslice.kubernetes.io/skip-mirror 值为 true。
- Endpoints 资源包含标签 control-plane.alpha.kubernetes.io/leader。
- 对应的 Service 资源不存在。
- 对应的 Service 的选择算符不为空。

每个 Endpoints 资源可能会被转译到多个 EndpointSlices 中去。 当 Endpoints 资源中包含多个子网或者包含多个 IP 协议族（IPv4 和 IPv6）的端点时， 就有可能发生这种状况。 每个子网最多有 1000 个地址会被镜像到 EndpointSlice 中。

### EndpointSlices 的分布问题

每个 EndpointSlice 都有一组端口值，适用于资源内的所有端点。当为 Service 使用命名端口时，Pod 可能会就同一命名端口获得不同的端口号， 因而需要不同的 EndpointSlice。这有点像 Endpoints 用来对子网进行分组的逻辑。

控制面尝试尽量将 EndpointSlice 填满，不过不会主动地在若干 EndpointSlice 之间执行再平衡操作。这里的逻辑也是相对直接的：

1. 列举所有现有的 EndpointSlices，移除那些不再需要的端点并更新那些已经变化的端点。
1. 列举所有在第一步中被更改过的 EndpointSlices，用新增加的端点将其填满。
1. 如果还有新的端点未被添加进去，尝试将这些端点添加到之前未更改的切片中， 或者创建新切片。


这里比较重要的是，与在 EndpointSlice 之间完成最佳的分布相比，第三步中更看重限制 EndpointSlice 更新的操作次数。例如，如果有 10 个端点待添加，有两个 EndpointSlice 中各有 5 个空位，上述方法会创建一个新的 EndpointSlice 而不是将现有的两个 EndpointSlice 都填满。换言之，与执行多个 EndpointSlice 更新操作相比较， 方法会优先考虑执行一个 EndpointSlice 创建操作。

由于 kube-proxy 在每个节点上运行并监视 EndpointSlice 状态，EndpointSlice 的每次变更都变得相对代价较高，因为这些状态变化要传递到集群中每个节点上。 这一方法尝试限制要发送到所有节点上的变更消息个数，即使这样做可能会导致有多个 EndpointSlice 没有被填满。

在实践中，上面这种并非最理想的分布是很少出现的。大多数被 EndpointSlice 控制器处理的变更都是足够小的，可以添加到某已有 EndpointSlice 中去的。 并且，假使无法添加到已有的切片中，不管怎样都会快就会需要一个新的 EndpointSlice 对象。Deployment 的滚动更新为重新为 EndpointSlice 打包提供了一个自然的机会，所有 Pod 及其对应的端点在这一期间都会被替换掉。

### 重复的 Endpoints

由于 EndpointSlice 变化的自身特点，某个 Endpoint 可能会同时出现在不止一个 EndpointSlice 中。

鉴于不同的 EndpointSlice 对象在不同时刻到达 Kubernetes 的监视/缓存中，这种情况的出现是很自然的。使用 EndpointSlice 的实现必须能够处理端点出现在多个切片中的状况，此时需要考虑去重。

关于如何执行端点去重（deduplication）的参考实现，你可以在 kube-proxy 的 EndpointSlice 实现中找到。
