# 临时容器

有一种特殊的容器，该容器在现有 Pod 中临时运行，以便完成用户发起的操作，例如故障排查。

你会使用临时容器来检查服务，而不是用它来构建应用程序。

## 了解临时容器

Pod 是 Kubernetes 应用程序的基本构建块。

由于 Pod 是一次性且可替换的，因此一旦 Pod 创建，就无法将容器加入到 Pod 中。取而代之的是，通常使用 Deployment 以受控的方式来删除并替换 Pod。

有时有必要检查现有 Pod 的状态。例如，对于难以复现的故障进行排查。

在这些场景中，可以在现有 Pod 中**添加运行临时容器**来检查其状态并运行任意命令。

### 什么是临时容器？

临时容器与其他容器的不同之处在于: 

- 它们缺少对资源或执行的保证
- 并且永远不会自动重启

因此临时不适用于构建应用程序。

临时容器使用与常规容器相同的 ContainerSpec 节来描述，但许多字段是不兼容和不允许的:

- 临时容器没有端口配置，因此像 ports，livenessProbe，readinessProbe 这样的字段是不允许的。
- Pod 资源分配是不可变的，因此 resources 配置是不允许的。
- 有关允许字段的完整列表，请参见 EphemeralContainer 参考文档。


临时容器是使用 API 中的一种特殊的 ephemeralcontainers 处理器进行创建的，而不是直接添加到 pod.spec 段，因此无法使用 kubectl edit 来添加一个临时容器。

与常规容器一样: **将临时容器添加到 Pod 后，将不能更改或删除临时容器**。

### 临时容器的用途

当由于容器崩溃或容器镜像不包含调试工具而**导致 kubectl exec 无用**时，临时容器对于交互式故障排查很有用。

使用临时容器时，使用[进程名字空间共享](https://kubernetes.io/zh-cn/docs/tasks/configure-pod-container/share-process-namespace/)很有帮助，可以查看其他容器中的进程。

这是一个使用临时容器的示例:

```sh
# kubectl debug -it <pod-name> --image=<image> --target=<容器的进程命名空间>
$ kubectl debug -it ephemeral-demo --image=busybox:1.28 --target=ephemeral-demo
```

**注意:**

- 并非所有集群都开启了临时容器特性

  ```sh
  ephemeral containers are disabled for this cluster (error from server: "the server could not find the requested resource").
  ```
