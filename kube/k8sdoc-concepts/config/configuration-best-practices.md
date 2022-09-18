# 最佳配置实践

介绍并整合了整个用户指南、入门文档和示例中介绍的配置最佳实践。

该最佳实践是一份不断改进的文件。如果你认为某些内容缺失但可能对其他人有用，请不要犹豫，提交 Issue 或提交 PR。

## 一般配置提示

- 定义配置时，请指定**最新的稳定 API 版本**。
- 在推送到集群之前，**配置文件应存储在版本控制中**。这允许你在必要时快速回滚配置更改。它还有助于集群重新创建和恢复。
- 使用 YAML 而不是 JSON 编写配置文件。虽然这些格式几乎可以在所有场景中互换使用，但 YAML 往往更加用户友好。
- 只要有意义，就将**相关对象分组到一个文件中**。 一个文件通常比几个文件更容易管理。请参阅 [guestbook-all-in-one.yaml](https://github.com/kubernetes/examples/blob/master/guestbook/all-in-one/guestbook-all-in-one.yaml) 文件作为此语法的示例。
- 请注意，可以在目录上调用许多 kubectl 命令。例如，你可以在配置文件的目录中调用 kubectl apply。
- 除非必要，否则**不指定默认值**：简单的最小配置会降低错误的可能性。（我个人认为不指定默认值，会降低可读性，以及难以理解特性会以什么方式执行。同时系统也不确定默认值是否会改变）
- 将对象描述放在注释中，以便有更好的可读性。

## “Naked” Pods 与 ReplicaSet，Deployment 和 Jobs

- 如果可能，不要使用 Naked Pods（裸 Pods，即，未绑定到 ReplicaSet 或 Deployment 的 Pod）。如果节点发生故障，将不会重新调度独立的 Pods。
- Deployment 既可以创建一个 ReplicaSet 来确保预期个数的 Pod 始终可用，也可以指定替换 Pod 的策略（例如 RollingUpdate）。除了一些显式的 restartPolicy: Never 场景外，Deployment 通常比直接创建 Pod 要好得多（容器 Never 重启，同时容器会结束，会导致 Pod 结束，并且 Deployment 会重新拉一个新的 Pod）。
- Job 也可能是合适的选择。

## 服务（Service）

在创建相应的后端工作负载（Deployment 或 ReplicaSet），以及在需要访问它的任何工作负载**之前**创建服务。

当 Kubernetes 启动容器时，它提供指向启动容器时正在运行的所有服务的环境变量。例如，如果存在名为 foo 的服务，则所有容器将在其初始环境中获得以下变量。

```sh
FOO_SERVICE_HOST=<the host the Service is running on>
FOO_SERVICE_PORT=<the port the Service is running on>
```

这确实意味着在顺序上的要求 - 必须在 Pod 本身被创建之前创建 Pod 想要访问的任何 Service，否则将环境变量不会生效（但是 DNS 没有此限制）。

- 一个**可选（尽管强烈推荐）**的集群插件是 DNS 服务器。DNS 服务器为新的 Services 监视 Kubernetes API，并为每个创建一组 DNS 记录。 如果在整个集群中启用了 DNS，则所有 Pods 应该能够自动对 Services 进行名称解析。
- 除非绝对必要，否则不要为 Pod 指定 hostPort。将 Pod 绑定到 hostPort 时，它会限制 Pod 可以调度的位置数，因为每个 `<hostIP, hostPort, protocol>` 组合必须是唯一的。如果你没有明确指定 hostIP 和 protocol，Kubernetes 将使用 0.0.0.0 作为默认 hostIP 和 TCP 作为默认 protocol。

如果你只需要访问端口以进行调试，则可以使用 apiserver proxy或 kubectl port-forward。

如果你明确需要在节点上公开 Pod 的端口，请在使用 hostPort 之前考虑使用 NodePort 服务。

- 避免使用 hostNetwork，原因与 hostPort 相同。
- 当你不需要 kube-proxy 负载均衡时，使用 Headless Service（直连的话，性能更好，不需要 iptables 转一下了，还能 ping 通）。

## 使用标签

- 定义并使用标签来**识别**你的应用程序或 Deployment 的语义属性。
  - 例如：`{ app.kubernetes.io/name: MyApp, tier: frontend, phase: test, deployment: v3 }`。你可以使用这些标签为其他资源选择合适的 Pod。
  - 一个选择所有 tier: frontend Pod 的服务，或者 app.kubernetes.io/name: MyApp 的所有 phase: test 组件。


通过从选择器中省略特定发行版的标签，可以使服务跨越多个 Deployment。当你需要不停机的情况下更新正在运行的服务，可以使用 Deployment。

Deployment 描述了对象的期望状态，并且如果对该规范的更改被成功应用，则 Deployment 控制器以受控速率将实际状态改变为期望状态。

- 对于常见场景，应使用 Kubernetes 通用标签。这些标准化的标签丰富了对象的元数据，使得包括 kubectl 和仪表板（Dashboard） 这些工具能够以可互操作的方式工作。
- 你可以操纵标签进行调试。 由于 Kubernetes 控制器（例如 ReplicaSet）和服务使用选择器标签来匹配 Pod，从 Pod 中删除相关标签将阻止其被控制器考虑或由服务提供服务流量。 如果删除现有 Pod 的标签，其控制器将创建一个新的 Pod 来取代它。这是在"隔离"环境中调试先前"活跃"的 Pod 的有用方法。要以交互方式删除或添加标签，请使用 `kubectl label`。

## 使用 kubectl

- 使用 `kubectl apply -f <directory>`。它在 `<directory>` 中的所有 .yaml、.yml 和 .json 文件中查找 Kubernetes 配置，并将其传递给 apply。
- 使用标签选择器进行 get 和 delete 操作，而不是特定的对象名称。
- 使用 kubectl run 和 kubectl expose 来快速创建单容器部署和服务。
