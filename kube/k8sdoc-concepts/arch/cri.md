# 容器运行时接口（Container Runtime Interface，CRI）

CRI 是一个插件接口，它使 kubelet 能够使用各种容器运行时，无需重新编译集群组件。

你需要在集群中的每个节点上都有一个可以正常工作的 Container Runtime（容器运行时），这样 kubelet 能启动 Pod 及其容器。

**重要的是：**

- 容器运行时接口（CRI）是 kubelet 和容器运行时之间通信的主要协议。
- Kubernetes 容器运行时接口 (CRI) 定义了用于集群组件 kubelet 和**容器运行时**之间通信的主要 gRPC 协议。

## API

特性状态： Kubernetes v1.23 [stable]

当通过 gRPC 连接到 Container Runtime 时，kubelet 充当客户端。

Container Runtime 和镜像服务端点必须可用，可以使用命令行标志的 --image-service-endpoint 和 --container-runtime-endpoint 在 kubelet 中单独配置。

Kubelet 对使用的 CRI 会进行版本协商：

- 对 Kubernetes v1.25，kubelet 偏向于使用 CRI v1 版本。
- 如果容器运行时不支持 CRI 的 v1 版本，那么 kubelet 会尝试协商任何旧的其他支持版本。
- 如果 kubelet 无法协商支持的 CRI 版本，则 kubelet 放弃并且不会注册为节点。

## 升级

升级 Kubernetes 时，kubelet 会尝试在组件重启时自动选择最新的 CRI 版本。

如果失败，则将如上所述进行回退。如果由于容器运行时已升级而需要 gRPC 重拨， 则容器运行时还必须支持最初选择的版本，否则重拨预计会失败。这需要重新启动 kubelet。
