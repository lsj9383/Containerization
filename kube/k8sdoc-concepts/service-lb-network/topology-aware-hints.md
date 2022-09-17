# 拓扑感知提示

拓扑感知通过一些提示（例如 Clients 怎么使用服务端点的建议），从而实现了拓扑感知的路由功能。

此方法添加元数据以启用 EndpointSlice 和/或 Endpoints 对象的消费者，以便可以将到这些网络端点的流量路由到更接近其起源的位置。

例如，你可以在一个地域内路由流量，以降低通信成本，或提高网络性能。

## 动机

Kubernetes 集群越来越多的部署到多区域环境中。

拓扑感知提示提供了一种把流量限制在它的发起区域之内的机制。这个概念一般被称之为 “拓扑感知路由”。

在获取 `服务（Service）` 的 Endpoint 时， EndpointSlice 控制器会评估每一个 Endpoint 的拓扑（地域和区域），填充提示字段，并将其分配到某个区域。

集群组件，例如kube-proxy 就可以使用这些提示信息，并用他们来影响流量的路由（倾向于拓扑上相邻的端点）。

## 使用拓扑感知提示

你可以通过把注解 service.kubernetes.io/topology-aware-hints 的值设置为 auto，来激活服务的拓扑感知提示功能。 这告诉 EndpointSlice 控制器在它认为安全的时候来设置拓扑提示。

## 工作原理

此特性启用的功能分为两个组件：

- EndpointSlice 控制器
- kube-proxy

### EndpointSlice 控制器

此特性开启后，EndpointSlice Controller 负责在 EndpointSlice 上设置提示信息。

Controller 按比例给每个区域分配一定比例数量的端点。这个比例来源于此区域中运行节点的**可分配 CPU 核心数**。

例如，如果一个区域拥有 2 CPU 核心，而另一个区域只有 1 CPU 核心， 那控制器将给那个有 2 CPU 的区域分配两倍数量的端点。

以下示例展示了提供提示信息后 EndpointSlice 的样子：

```yaml
apiVersion: discovery.k8s.io/v1
kind: EndpointSlice
metadata:
  name: example-hints
  labels:
    kubernetes.io/service-name: example-svc
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
    zone: zone-a
    hints:
      forZones:
        - name: "zone-a"
```

### kube-proxy

kube-proxy 组件依据 EndpointSlice 控制器设置的提示，过滤由它负责路由的端点。

在大多数场合，这意味着 kube-proxy 可以把流量路由到同一个区域的端点。有时，控制器从某个不同的区域分配端点，以确保在多个区域之间更平均的分配端点。 这会导致部分流量被路由到其他区域。

流量路由主要看发起请求的节点区域配置，可以到节点中去查看区域信息，例如：

```sh
$ kubectl get nodes 10.0.1.2 -o yaml|grep zone
    failure-domain.beta.kubernetes.io/zone: "100003"
    topology.com.tencent.cloud.csi.cbs/zone: ap-guangzhou-3
    topology.kubernetes.io/zone: "100003"
```

## 保护措施

Kubernetes 控制平面和每个节点上的 kube-proxy，在使用拓扑感知提示功能前，会应用一些保护措施规则。

如果没有检出，kube-proxy 将无视区域限制，从集群中的任意节点上选择端点。

- **端点数量不足：**如果一个集群中，端点数量少于区域数量，控制器不创建任何提示。
- **不可能实现均衡分配：** 在一些场合中，不可能实现端点在区域中的平衡分配。 例如，假设 zone-a 比 zone-b 大两倍，但只有 2 个端点， 那分配到 zone-a 的端点可能收到比 zone-b 多两倍的流量。 如果控制器不能确定此“期望的过载”值低于每一个区域可接受的阈值，控制器将不指派提示信息。 重要的是，这不是基于实时反馈。所以对于单独的端点仍有可能超载。
- **一个或多个节点信息不足：** 如果任一节点没有设置标签 topology.kubernetes.io/zone， 或没有上报可分配的 CPU 数据，控制平面将不会设置任何拓扑感知提示， 继而 kube-proxy 也就不能通过区域过滤端点。
- **一个或多个端点没有设置区域提示：** 当这类事情发生时， kube-proxy 会假设这是正在执行一个从/到拓扑感知提示的转移。 在这种场合下过滤Service 的端点是有风险的，所以 kube-proxy 回撤为使用所有的端点。
- **不在提示中的区域：**如果 kube-proxy 不能根据一个指示在它所在的区域中发现一个端点， 它回撤为使用所有节点的端点。当你的集群新增一个新的区域时，这种情况发生概率很高。


## 限制

- 当 Service 的 externalTrafficPolicy 或 internalTrafficPolicy 设置值为 **Local** 时，**拓扑感知提示功能不可用**。你可以在一个集群的不同服务中使用这两个特性，但不能在同一个服务中这么做。
- 这种方法不适用于大部分流量来自于一部分区域的服务。 相反的，这里假设入站流量将根据每个区域中节点的服务能力按比例的分配。
- EndpointSlice 控制器在计算每一个区域的容量比例时，会忽略未就绪的节点。 在大量节点未就绪的场景下，这样做会带来非预期的结果。
- EndpointSlice 控制器在计算每一个区域的部署比例时，并不会考虑 容忍度。 如果服务后台的 Pod 被限制只能运行在集群节点的一个子集上，这些信息并不会被使用。
- 这种方法和自动扩展机制之间不能很好的协同工作。例如，如果大量流量来源于一个区域， 那只有分配到该区域的端点才可用来处理流量。这会导致 Pod 自动水平扩展 要么不能拾取此事件，要么新增 Pod 被启动到其他区域。
