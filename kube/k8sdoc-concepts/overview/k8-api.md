# Kubernetes API

Kubernetes 控制面的核心是 API 服务器。 API 服务器负责提供 HTTP API，以供用户、集群中的不同部分和集群外部组件相互通信。

Kubernetes API 使你可以查询和操纵 Kubernetes API 中对象（例如：Pod、Namespace、ConfigMap 和 Event）的状态。

大部分操作都可以通过 kubectl 命令行接口或类似 kubeadm 这类命令行工具来执行，这些工具在背后也是调用 API。

不过，你也可以使用 REST 调用来访问这些 API。

如果你正在编写程序来访问 Kubernetes API， 可以考虑使用：[客户端库](https://kubernetes.io/zh-cn/docs/reference/using-api/client-libraries/)。

