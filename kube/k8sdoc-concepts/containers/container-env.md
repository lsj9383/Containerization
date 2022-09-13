# 容器环境

Kubernetes 的容器环境给 Container 提供了几个重要的资源：

- 文件系统，它是一个镜像和多个 volumes 的组合。
- 关于容器自身的信息。
- 集群中 Kubernetes 对象的信息

## 容器信息

一个容器的 `hostname` 是该容器运行所在的 Pod 的名称。

容器通过 hostname 命令或者调用 libc 中的 [gethostname()](https://man7.org/linux/man-pages/man2/gethostname.2.html) 可以获取该名称。

Pod 名称和 namespace 可以通过 [下行 API(Downward API)](https://kubernetes.io/zh-cn/docs/tasks/inject-data-application/downward-api-volume-expose-pod-information/) 转换为环境变量。

用户为 Pod 定义的环境变量也可在容器中使用。

**注意：**

- 进入容器的方法：`docker exec -i 69d1 bash`。

## 集群信息

创建容器时**正在运行**的所有服务都可用作该容器的环境变量（后面才运行的服务不会出现在环境变量中）。

这里的服务仅限于:

- 新容器的 Pod 所在的名字空间中的服务
- Kubernetes 控制面的服务

对于名为 foo 的服务，当映射到名为 bar 的容器时，定义了以下变量：

```sh
FOO_SERVICE_HOST=<其上服务正运行的主机>
FOO_SERVICE_PORT=<其上服务正运行的端口>
```

服务具有专用的 IP 地址。如果启用了 DNS 插件， 也可以在容器中通过 DNS 来访问服务，而不用通过环境变量。
