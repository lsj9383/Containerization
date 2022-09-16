# 服务（Service）

Service 将运行在一组 Pods 上的应用程序公开为网络服务的抽象 Kubernetes 对象。

使用 Kubernetes，你无需修改应用程序即可使用服务发现机制：

- Kubernetes 为 Pod 提供自己的 IP 地址。
- 并为一组 Pod 提供相同的 DNS 名，并且可以在它们之间进行负载均衡。

## 动机

Pod 是非永久性资源，如果你使用 Deployment 来运行你的应用程序，则它可以动态创建和销毁 Pod（创建和销毁 Kubernetes Pod 以匹配集群的期望状态）。

每个 Pod 都有自己的 IP 地址，但是在 Deployment 中，在同一时刻运行的 Pod 集合可能与稍后运行该应用程序的 Pod 集合不同（即 IP 会变化）。

这导致了一个问题：

- 如果一组 Pod（称为“后端”）为集群内的其他 Pod（称为“前端”）提供功能，那么客户端如何找出并跟踪要连接的 IP 地址，以便前端可以使用提供工作负载的后端部分？

为了解决这个问题，引入了 Services。

## Service 资源

Kubernetes Service 定义了这样一种抽象：

- 逻辑上的一组 Pod，一种可以访问它们的策略（通常称为微服务）。 
- Service 所针对的 Pod 集合通常是通过选择算符来确定的（要了解定义服务端点的其他方法，请参阅不带选择算符的服务）。

举个例子，考虑一个图片处理后端，它运行了 3 个副本。这些副本是可互换的 —— 前端不需要关心它们调用了哪个后端副本。然而组成这一组后端程序的 Pod 实际上可能会发生变化，前端客户端不应该也没必要知道，而且也不需要跟踪这一组后端的状态。

Service 定义的抽象能够解耦这种关联。

### 云原生服务发现

如果你想要在应用程序中使用 Kubernetes API 进行服务发现，则可以查询 API 服务器 的 Endpoints 资源，只要服务中的 Pod 集合发生更改，Endpoints 就会被更新。

**注意：**

- 即客户端去查询 Pod 关联的 Endpoints 资源，当 Pod IP 发生变化，Endpoints 会提现出来。相当于一种比较复杂的 DNS 方案。

## 定义 Service

Service 在 Kubernetes 中是一个 REST 对象，和 Pod 类似。像所有的 REST 对象一样，Service 定义可以基于 POST 方式，请求 API server 创建新的实例。 Service 对象的名称必须是合法的 RFC 1035 标签名称。

例如，假定有一组 Pod，它们对外暴露了 9376 端口，同时还被打上 app=MyApp 标签：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app.kubernetes.io/name: MyApp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
```

上述配置创建一个名称为 "my-service" 的 Service 对象，它会将请求代理到 Pods 上，并且这些 Pods：

- 具有标签 app.kubernetes.io/name=MyApp 的 Pods
- 使用 TCP 端口 9376

Kubernetes 为该服务分配一个 IP 地址（有时称为 “集群 IP”），该 IP 地址由服务代理使用。 (请参见下面的 VIP 和 Service 代理).

服务选择算符的控制器不断扫描与其选择算符匹配的 Pod，然后将所有更新发布到也称为 “my-service” 的 Endpoint 对象。

**注意：**

- Service 能够将一个接收 port 映射到任意的 targetPort。默认情况下，targetPort 将被设置为与 port 字段相同的值。


Pod 中的端口定义是有名字的，你可以在 Service 的 targetPort 属性中引用这些名称。例如，我们可以通过以下方式将 Service 的 targetPort 绑定到 Pod 端口：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    app.kubernetes.io/name: proxy
spec:
  containers:
  - name: nginx
    image: nginx:stable
    ports:
      - containerPort: 80
        name: http-web-svc

---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app.kubernetes.io/name: proxy
  ports:
  - name: name-of-service-port
    protocol: TCP
    port: 80
    targetPort: http-web-svc
```

即使 Service 中使用同一配置名称混合使用多个 Pod，各 Pod 通过不同的端口号支持相同的网络协议， 此功能也可以使用。这为 Service 的部署和演化提供了很大的灵活性。 例如，你可以在新版本中更改 Pod 中后端软件公开的端口号，而不会破坏客户端。

服务的默认协议是 TCP；你还可以使用任何其他受支持的协议。

由于许多服务需要公开多个端口，因此 Kubernetes 在服务对象上支持多个端口定义。 每个端口定义可以具有相同的 protocol，也可以具有不同的协议。

### 没有选择算符的 Service

由于选择算符的存在，服务最常见的用法是为 Kubernetes Pod 的访问提供抽象，但是当与相应的 Endpoints 对象一起使用且没有选择算符时， 服务也可以为其他类型的后端提供抽象，包括在集群外运行的后端。 例如：

- 希望在生产环境中使用外部的数据库集群，但测试环境使用自己的数据库。
- 希望服务指向另一个名字空间（Namespace） 中或其它集群中的服务。
- 你正在将工作负载迁移到 Kubernetes。在评估该方法时，你仅在 Kubernetes 中运行一部分后端。


在任何这些场景中，都能够定义没有选择算符的 Service。 实例:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
```

由于此服务没有选择算符，因此不会自动创建相应的 Endpoints 对象。 你可以通过手动添加 Endpoints 对象，将服务手动映射到运行该服务的网络地址和端口：

```yaml
apiVersion: v1
kind: Endpoints
metadata:
  # 这里的 name 要与 Service 的名字相同
  name: my-service
subsets:
  - addresses:
      - ip: 192.0.2.42
    ports:
      - port: 9376
```

**注意：**

- 当你为某个 Service 创建一个 Endpoints 对象时，你要将**Endpoints 对象的名称设置为与 Service 的名称相同**。
- 端点 IP 地址不能是其他 Kubernetes 服务的集群 IP，因为 kube-proxy 不支持将虚拟 IP 作为目标。

访问没有选择算符的 Service，与有选择算符的 Service 的原理相同。 请求将被路由到用户定义的 Endpoint，YAML 中为：192.0.2.42:9376（TCP）。

### 超出容量的 Endpoints

如果某个 Endpoints 资源中包含的端点个数超过 1000，则 Kubernetes v1.22 版本 （及更新版本）的集群会将为该 Endpoints 添加注解 endpoints.kubernetes.io/over-capacity: truncated。

这一注解表明所影响到的 Endpoints 对象已经超出容量，此外 Endpoints 控制器还会将 Endpoints 对象数量截断到 1000。

Endpoints 里面存的就是 Pod IP - Pod Port 对

### EndpointSlices

EndpointSlices 是一种 API 资源，可以为 Endpoints 提供更可扩展的替代方案。

尽管从概念上讲与 Endpoints 非常相似，但 EndpointSlices 允许跨多个资源分布网络端点。 默认情况下，一旦到达 100 个 Endpoint，该 EndpointSlice 将被视为“已满”，届时将创建其他 EndpointSlices 来存储任何其他 Endpoints。

EndpointSlices 提供了附加的属性和功能，这些属性和功能在 EndpointSlices 中有详细描述。

### 应用协议

appProtocol 字段提供了一种为每个 Service 端口指定应用协议的方式。此字段的取值会被映射到对应的 Endpoints 和 EndpointSlices 对象。

## 虚拟 IP 和 Service 代理

在 Kubernetes 集群中，每个 Node 运行一个 kube-proxy 进程。 kube-proxy 负责为 Service 实现了一种 VIP（虚拟 IP）的形式，而不是 ExternalName 的形式。

这个 VIP 其实就是 Cluster IP。

### 为什么不使用 DNS 轮询？

时不时会有人问到为什么 Kubernetes 依赖代理将入站流量转发到后端。那其他方法呢？例如，是否可以配置具有多个 A 值（或 IPv6 为 AAAA）的 DNS 记录，并依靠轮询名称解析？

使用服务代理有以下几个原因：

- DNS 实现的历史由来已久，它不遵守记录 TTL，并且在名称查找结果到期后对其进行缓存。
- 有些应用程序仅执行一次 DNS 查找，并无限期地缓存结果。
- 即使应用和库进行了适当的重新解析，DNS 记录上的 TTL 值低或为零也可能会给 DNS 带来高负载，从而使管理变得困难。


在本页下文中，你可以了解各种 kube-proxy 实现是如何工作的。

总的来说，你应该注意当运行 kube-proxy 时，内核级别的规则可能会被修改（例如，可能会创建 iptables 规则）。

在某些情况下直到你重新引导才会清理这些内核级别的规则。因此，运行 kube-proxy 只能由了解在计算机上使用低级别、特权网络代理服务后果的管理员来完成。尽管 kube-proxy 可执行文件支持 cleanup 功能，但此功能不是官方特性，因此只能按原样使用。

### 配置

请注意，kube-proxy 可以以不同的模式启动，具体取决于其配置。

- kube-proxy 的配置是通过 ConfigMap 完成的，并且 kube-proxy 的 ConfigMap 有效地弃用了 kube-proxy 几乎所有标志的行为。
- kube-proxy 的 ConfigMap 不支持实时重新加载配置。
- kube-proxy 的 ConfigMap 参数不能在启动时被全部校验和验证。例如，如果你的操作系统不允许你运行 iptables 命令，则标准内核 kube-proxy 实现将无法工作。

### userspace 代理模式

这种模式，kube-proxy 会监视 Kubernetes Control Panel 对 Service 对象和 Endpoints 对象的添加和移除操作。

具体而言：

- kube-proxy 发现了新的 service 后会修改 iptables 规则，让所有请求该 cluster IP 的请求都到 kube-proxy，
- kube-proxy 从 Endpoints 对象中获取该 Cluster IP 可以路由给那些 Pod 地址（pod-ip: pod-port）。对于一个 Service 有多个 Pod 地址的情况，默认使用轮询算法选择 Pod 地址。

![](assets/services-userspace-overview.svg)

### iptables 代理模式

这种模式，kube-proxy 会监视 Kubernetes 控制节点对 Service 对象和 Endpoints 对象的添加和移除。

对每个 Service，它会配置 iptables 规则，从而让 iptables 捕获到达该 Service 的 clusterIP 和端口的请求，进而将请求重定向到 Service 的一组后端中的**某个 Pod** 上面。

默认的策略是，kube-proxy 在 iptables 模式下**随机选择一个后端**。

使用 iptables 处理流量具有较低的系统开销，因为流量由 Linux netfilter 处理， 而无需在用户空间和内核空间之间切换。 这种方法也可能更可靠。

**注意：**

- iptables 的转发目标地址是有多个的，kube-proxy 会把这些目标地址（pod-ip:pod-port）都设置到到 iptables 中去，当有请求发往 cluster-ip:port 时，iptables 随机选择一个。
- 如果所选的第一个 Pod 没有响应，则连接失败。 这与用户空间模式不同：对于 userspace 代理模式，在这种情况下，kube-proxy 将检测到与第一个 Pod 的连接已失败，并会自动使用其他后端 Pod 重试。
- 这是因为 iptables 没有一个 proxy 可以进行重试，只会在流量过来的时候随机选择一个。

你可以使用 Pod 就绪探测器 验证后端 Pod 可以正常工作，以便 iptables 模式下的 kube-proxy 仅看到测试正常的后端。 这样做意味着你避免将流量通过 kube-proxy 发送到已知已失败的 Pod。

![](assets/services-iptables-overview.svg)

### IPVS 代理模式

在 ipvs 模式下，kube-proxy 监视 Kubernetes 服务和端点，调用 netlink 接口相应地创建 IPVS 规则，并定期将 IPVS 规则与 Kubernetes 服务和端点同步。

该控制循环可确保 IPVS 状态与所需状态匹配。访问服务时，IPVS 将流量定向到后端 Pod 之一。

IPVS 代理模式基于类似于 iptables 模式的 netfilter 挂钩函数， 但是使用哈希表作为基础数据结构，并且在内核空间中工作。

这意味着，与 iptables 模式下的 kube-proxy 相比，IPVS 模式下的 kube-proxy 重定向通信的延迟要短，并且在同步代理规则时具有更好的性能。与其他代理模式相比，IPVS 模式还支持更高的网络流量吞吐量。

IPVS 提供了更多选项来平衡后端 Pod 的流量。这些是：

- rr：轮替（Round-Robin）
- lc：最少链接（Least Connection），即打开链接数量最少者优先
- dh：目标地址哈希（Destination Hashing）
- sh：源地址哈希（Source Hashing）
- sed：最短预期延迟（Shortest Expected Delay）
- nq：从不排队（Never Queue）

**注意：**

- 要在 IPVS 模式下运行 kube-proxy，必须在启动 kube-proxy 之前使 IPVS 在节点上可用。
- 当 kube-proxy 以 IPVS 代理模式启动时，它将验证 IPVS 内核模块是否可用。如果未检测到 IPVS 内核模块，则 kube-proxy 将退回到以 iptables 代理模式运行。

![](assets/services-ipvs-overview.svg)

在这些代理模型中，绑定到服务 IP 的流量： 在客户端不了解 Kubernetes 或服务或 Pod 的任何信息的情况下，将 Port 代理到适当的后端。

如果要确保每次都将来自特定客户端的连接传递到同一 Pod，可以：

- 通过将 service.spec.sessionAffinity 设置为 "ClientIP" （默认值是 "None"），来基于客户端的 IP 地址选择会话亲和性。

你还可以通过适当设置 service.spec.sessionAffinityConfig.clientIP.timeoutSeconds 来设置最大会话停留时间。（默认值为 10800 秒，即 3 小时）。

要查看当前 kube-proxy 使用什么代理模式，可以：

```sh
$ curl "http://localhost:10249/proxyMode"
iptables
```

因为一般都用的是 nat 模式，这种比较快。这里举一个分析 iptables 代理模式 nat 的例子：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  selector:
    matchLabels:
      run: my-nginx
  replicas: 2
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - name: my-nginx
        image: nginx
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: my-nginx
  labels:
    run: my-nginx
spec:
  ports:
    - port: 80
      protocol: TCP
  selector:
    run: my-nginx
```

后面启用该 service 和 deployment：

```sh
# 我们有一个 service: my-nginx
$ kubectl get service
my-nginx              ClusterIP      172.16.253.3     <none>        80/TCP          69s

# 这个 service 有需要路由给两个 pod 地址（Endpoints），同时集群 IP（VIP）为 172.16.253.3：
$ kubectl describe service my-nginx
...
IP:                172.16.253.3
Endpoints:         172.16.0.43:80,172.16.0.83:80
...

# 看一下我们的 NAT 输出匹配链
$ iptables -t nat -nvL OUTPUT
Chain OUTPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         
1015K   72M KUBE-SERVICES  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kubernetes service portals */

# 可以看到所有的输出都是交给 KUBE-SERVICES 规则来处理的，再看一下它是怎么处理的：
$ iptables -t nat -nvL KUBE-SERVICES
Chain KUBE-SERVICES (2 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 KUBE-SVC-L65ENXXZWWSAPRCR  tcp  --  *      *       0.0.0.0/0            172.16.253.3         /* default/my-nginx cluster IP */ tcp dpt:80

# 可以看到指向 172.16.253.3 的数据会交给 KUBE-SVC-L65ENXXZWWSAPRCR 规则处理
$ iptables -t nat -nvL KUBE-SVC-L65ENXXZWWSAPRCR
Chain KUBE-SVC-L65ENXXZWWSAPRCR (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 KUBE-SEP-CKTK4HVPCCPKEGFT  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* default/my-nginx */ statistic mode random probability 0.50000000000
    0     0 KUBE-SEP-5DPHGI3VF3PCK466  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* default/my-nginx */

# 可以看到对于指向 172.16.253.3 的请求，会以 50% 的概率选择 KUBE-SEP-CKTK4HVPCCPKEGFT 和 KUBE-SEP-5DPHGI3VF3PCK466 规则
$ iptables -t nat -nvL KUBE-SEP-CKTK4HVPCCPKEGFT
Chain KUBE-SEP-CKTK4HVPCCPKEGFT (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 KUBE-MARK-MASQ  all  --  *      *       172.16.0.43          0.0.0.0/0            /* default/my-nginx */
    0     0 DNAT       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* default/my-nginx */ tcp to:172.16.0.43:80

$ iptables -t nat -nvL KUBE-SEP-5DPHGI3VF3PCK466
Chain KUBE-SEP-5DPHGI3VF3PCK466 (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 KUBE-MARK-MASQ  all  --  *      *       172.16.0.83          0.0.0.0/0            /* default/my-nginx */
    0     0 DNAT       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* default/my-nginx */ tcp to:172.16.0.83:80

# 到此可以看出对于 NAT 的处理，最终以 50% 的概率选择 172.16.0.43 或 172.16.0.83
```

这里有个比较好的 k8s NAT 分析: [K8S – 手把手分析 service 生成的 iptables 规则](https://yuerblog.cc/2019/12/09/k8s-%E6%89%8B%E6%8A%8A%E6%89%8B%E5%88%86%E6%9E%90service%E7%94%9F%E6%88%90%E7%9A%84iptables%E8%A7%84%E5%88%99/)

## 多端口 Service

对于某些服务，你需要公开多个端口。 Kubernetes 允许你在 Service 对象上配置多个端口定义。 为服务使用多个端口时，必须提供所有端口名称，以使它们无歧义。 例如：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app.kubernetes.io/name: MyApp
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 9376
    - name: https
      protocol: TCP
      port: 443
      targetPort: 9377
```

## 自定义 Cluster IP 地址

在 Service 创建的请求中，可以通过设置 `spec.clusterIP` 字段来指定自己的集群 IP 地址。比如，希望替换一个已经已存在的 DNS 条目，或者遗留系统已经配置了一个固定的 IP 且很难重新配置。

用户选择的 IP 地址必须合法，并且这个 IP 地址在 `service-cluster-ip-range CIDR` 范围内，这对 API 服务器来说是通过一个标识来指定的。

如果 IP 地址不合法，API 服务器会返回 HTTP 状态码 422，表示值不合法。

**注意：**

- 没有设置 Cluster IP 会随机选择
- 设置 Cluster IP 为 None，会启动无头服务

## 流量策略

### 外部流量策略

你可以通过设置 `spec.externalTrafficPolicy` 字段来控制来自于外部的流量是如何路由的。 可选值有:

- Cluster，会将外部流量路由到所有就绪的 Endpoint。
- Local，会只路由到当前节点上就绪的 Endpoint。如果流量策略设置为 Local，而且当前节点上没有就绪的端点，kube-proxy 不会转发请求相关服务的任何流量。

如果本地有端点，而且所有端点处于终止中的状态，那么 kube-proxy 会忽略任何设为 Local 的外部流量策略。 在所有本地端点处于终止中的状态的同时，kube-proxy 将请求指定服务的流量转发到位于其它节点的状态健康的端点， 如同外部流量策略设为 Cluster。

针对处于正被终止状态的端点这一转发行为使得外部负载均衡器可以优雅地排出由 NodePort 服务支持的连接，就算是健康检查节点端口开始失败也是如此。 否则，当节点还在负载均衡器的节点池内，在 Pod 终止过程中的流量会被丢掉，这些流量可能会丢失。

### 内部流量策略

你可以设置 `spec.internalTrafficPolicy` 字段来控制内部来源的流量是如何转发的。可设置的值有：

- Cluster，会将内部流量路由到所有就绪端点。
- Local，设置为 Local 只会路由到当前节点上就绪的端点。 如果流量策略是 Local，而且当前节点上没有就绪的端点，那么 kube-proxy 会丢弃流量。

## 服务发现

Service 还需要解决 Client 如何拿到 Cluster IP 的问题，这一问题可以通过服务发现解决。

Kubernetes 支持两种基本的服务发现模式：

- 环境变量
- DNS

### 环境变量

当 Pod 运行在 Node 上，kubelet 会为每个活跃的 Service 添加一组环境变量。

kubelet 为 Pod 添加环境变量：

- `{SVCNAME}_SERVICE_HOST`
- `{SVCNAME}_SERVICE_PORT`

这里 Service 的名称需大写，横线被转换成下划线。

举个例子，一个名称为 redis-primary 的 Service 暴露了 TCP 端口 6379，同时给它分配了 Cluster IP 地址 10.0.0.11，这个 Service 生成了如下环境变量：

```sh
REDIS_PRIMARY_SERVICE_HOST=10.0.0.11
REDIS_PRIMARY_SERVICE_PORT=6379
REDIS_PRIMARY_PORT=tcp://10.0.0.11:6379
REDIS_PRIMARY_PORT_6379_TCP=tcp://10.0.0.11:6379
REDIS_PRIMARY_PORT_6379_TCP_PROTO=tcp
REDIS_PRIMARY_PORT_6379_TCP_PORT=6379
REDIS_PRIMARY_PORT_6379_TCP_ADDR=10.0.0.11
```

**注意：**

- 后续新增的 Service 时，不会往已经存在的 Pod 中写环境变量
- 当你具有需要访问服务的 Pod 时，并且你正在使用环境变量方法将端口和集群 IP 发布到客户端 Pod 时，必须在客户端 Pod 出现之前创建服务。否则，这些客户端 Pod 将不会设定其环境变量。
- 如果仅使用 DNS 查找服务的集群 IP，则无需担心此设定问题。

### DNS

你可以（几乎总是应该）使用附加组件为 Kubernetes 集群设置 DNS 服务。

支持集群的 DNS 服务器（例如 CoreDNS）监视 Kubernetes API 中的新服务，并为每个服务创建一组 DNS 记录。如果在整个集群中都启用了 DNS，则所有 Pod 都应该能够通过其 DNS 名称自动解析服务。

例如：

- 你在 Kubernetes 命名空间 my-ns 中有一个名为 my-service 的服务
- 控制平面和 DNS 服务共同为 `my-service.my-ns` 创建 DNS 记录。
- my-ns 命名空间中的 Pod 应该能够通过按名检索 my-service 来找到服务（my-service.my-ns 也可以工作）。

关于 DNS 更多信息可以参考 [Pod 与 Service 的 DNS](https://kubernetes.io/zh-cn/docs/concepts/services-networking/dns-pod-service/)，只需要指导可以通过 DNS 找到 Cluster IP 即可。

## 无头服务（Headless Services）

有时不需要或不想要负载均衡，遇到这种情况，可以通过指定 Cluster IP（spec.clusterIP）的值为 "None" 来创建 Headless Service。

您可以使用无头服务与其他服务发现机制进行交互，而无需绑定到 Kubernetes 的实现。

对于 `Headless Services` 并不会分配 Cluster IP，kube-proxy 也不会处理它们，而且平台也不会为它们进行负载均衡和路由。DNS 如何实现自动配置，依赖于 Service 是否定义了选择算符。

### 带选择算符的服务

对定义了选择算符的 Headless service，Endpoints 控制器在 API 中创建了 Endpoints 记录，并且修改 DNS 配置返回 A 记录（IP 地址），通过这个地址直接到达 Service 的后端 Pod 上。

### 无选择算符的服务

对没有定义选择算符的无头服务，Endpoints 控制器不会创建 Endpoints 记录。 然而 DNS 系统会查找和配置，无论是：

- 对于 ExternalName 类型的服务，查找其 CNAME 记录
- 对所有其他类型的服务，查找与 Service 名称相同的任何 Endpoints 的记录

## 发布服务（服务类型）

对一些应用的某些部分（如前端），可能希望将其暴露给 Kubernetes 集群外部的 IP 地址。

Kubernetes ServiceTypes 允许指定你所需要的 Service 类型，默认是 ClusterIP。

Type 的取值以及行为如下：

Type | 描述
-|-
ClusterIP | 通过集群的内部 IP 暴露服务，选择该值时服务**只能够在集群内部访问**。这也是默认的 ServiceType。
NodePort | 通过每个节点上的 IP 和静态端口（NodePort）暴露服务。NodePort 服务会路由到自动创建的 ClusterIP 服务。<br>通过请求 `<节点 IP>:<节点端口>`，你可以从集群的外部访问一个 NodePort 服务。
LoadBalancer | 使用云提供商的负载均衡器向外部暴露服务。外部负载均衡器可以将流量路由到自动创建的 NodePort 服务和 ClusterIP 服务上。
ExternalName | 通过返回 CNAME 和对应值，可以将服务映射到 externalName 字段的内容（例如，foo.bar.example.com）。无需创建任何类型代理。

你也可以使用 Ingress 来暴露自己的服务。 Ingress 不是一种服务类型，但它充当集群的入口点。它可以将路由规则整合到一个资源中，因为它可以在同一 IP 地址下公开多个服务。

### NodePort 类型

如果你将 type 字段设置为 NodePort，则 Kubernetes 控制平面将在 `--service-node-port-range` 标志指定的范围内分配端口（默认值：30000-32767）。每个节点将那个端口（每个节点上的相同端口号）代理到你的服务中。你的服务在其 `.spec.ports[*].nodePort` 字段中要求分配的端口。

如果需要特定的端口号，你可以在 nodePort 字段中指定一个值。控制平面将为你分配该端口或报告 API 事务失败。这意味着你需要自己注意可能发生的端口冲突。你还必须使用有效的端口号，该端口号在配置用于 NodePort 的范围内。

使用 NodePort 可以让你自由设置自己的负载均衡解决方案，配置 Kubernetes 不完全支持的环境，甚至直接暴露一个或多个节点的 IP。

例如：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  type: NodePort
  selector:
    app.kubernetes.io/name: MyApp
  ports:
      # 默认情况下，为了方便起见，`targetPort` 被设置为与 `port` 字段相同的值。
    - port: 80
      targetPort: 80
      # 可选字段
      # 默认情况下，为了方便起见，Kubernetes 控制平面会从某个范围内分配一个端口号（默认：30000-32767）
      nodePort: 30007
```

### LoadBalancer 类型

在使用支持外部负载均衡器的云提供商的服务时，设置 type 的值为 "LoadBalancer"，将为 Service 提供负载均衡器。

负载均衡器是异步创建的，关于被提供的负载均衡器的信息将会通过 Service 的 `status.loadBalancer` 字段发布出去。

实例：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app.kubernetes.io/name: MyApp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
  clusterIP: 10.0.171.239
  type: LoadBalancer
status:
  loadBalancer:
    ingress:
    - ip: 192.0.2.127
```

来自外部负载均衡器的流量将直接重定向到后端 Pod 上，不过实际它们是如何工作的，这要依赖于云提供商。

某些云提供商允许设置 loadBalancerIP。 在这些情况下，将根据用户设置的 loadBalancerIP 来创建负载均衡器。如果没有设置 loadBalancerIP 字段，将会给负载均衡器指派一个临时 IP。

如果设置了 loadBalancerIP，但云提供商并不支持这种特性，那么设置的 loadBalancerIP 值将会被忽略掉。

**说明：**

- 在 Azure 上，如果要使用用户指定的公共类型 loadBalancerIP， 则首先需要创建静态类型的公共 IP 地址资源。 此公共 IP 地址资源应与集群中其他自动创建的资源位于同一资源组中。 例如，MC_myResourceGroup_myAKSCluster_eastus。

将分配的 IP 地址设置为 loadBalancerIP。确保你已更新云提供程序配置文件中的 securityGroupName。 有关对 CreatingLoadBalancerFailed 权限问题进行故障排除的信息， 请参阅与 Azure Kubernetes 服务（AKS）负载平衡器一起使用静态 IP 地址 或在 AKS 集群上使用高级联网时出现 CreatingLoadBalancerFailed。

### ExternalName 类型

类型为 ExternalName 的服务将服务映射到 DNS 名称，而不是典型的选择算符，例如 my-service 或者 cassandra。你可以使用 spec.externalName 参数指定这些服务。

例如，以下 Service 定义将 prod 名称空间中的 my-service 服务映射到 my.database.example.com：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
  namespace: prod
spec:
  type: ExternalName
  externalName: my.database.example.com
```

**注意：**

- ExternalName 服务接受 IPv4 地址字符串，但作为包含数字的 DNS 名称，而不是 IP 地址。
- 类似于 IPv4 地址的外部名称不能由 CoreDNS 或 ingress-nginx 解析，因为外部名称旨在指定规范的 DNS 名称。要对 IP 地址进行硬编码，请考虑使用无头 Services。
- 看起来就是个将 service DNS 添加一个 CNAME 的机制。

当查找主机 `my-service.prod.svc.cluster.local` 时，集群 DNS 服务返回 CNAME 记录，其值为 `my.database.example.com`。访问 my-service 的方式与其他服务的方式相同，但主要区别在于重定向发生在 DNS 级别，而不是通过代理或转发。 如果以后你决定将数据库移到集群中，则可以启动其 Pod，添加适当的选择算符或端点以及更改服务的 type。


### 外部 IP

如果外部的 IP 路由到集群中一个或多个 Node 上，Kubernetes Service 会被暴露给这些 externalIPs。

通过外部 IP（作为目的 IP 地址）进入到集群，打到 Service 的端口上的流量，将会被路由到 Service 的 Endpoint 上。

externalIPs 不会被 Kubernetes 管理，它属于集群管理员的职责范畴。

根据 Service 的规定，externalIPs 可以同任意的 ServiceType 来一起指定。在上面的例子中，my-service 可以在 "80.11.12.10:80"(externalIP:port) 上被客户端访问。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app.kubernetes.io/name: MyApp
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 9376
  externalIPs:
    - 80.11.12.10
```

## 不足之处

更多细节可以参考 [不足之处](https://kubernetes.io/zh-cn/docs/concepts/services-networking/service/#%E4%B8%8D%E8%B6%B3%E4%B9%8B%E5%A4%84)。
