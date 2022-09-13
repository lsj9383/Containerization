# 镜像

容器镜像（Image）所承载的是封装了应用程序及其所有软件依赖的二进制数据。

容器镜像是可执行的软件包，可以单独运行，该软件包对所处的运行时环境具有良定（Well Defined）的假定。

你通常会创建应用的容器镜像并将其推送到某仓库（Registry），然后在 Pod 中引用它。

## 镜像名称

容器镜像通常会被赋予 pause、example/mycontainer 或者 kube-apiserver 这类的名称。

镜像名称可以：

- 包含所在仓库的主机名，例如：fictional.registry.example/imagename。
- 包含仓库的端口号，例如：fictional.registry.example:10443/imagename。
- 如果你不指定仓库的主机名，Kubernetes 认为你在使用 **Docker 公共仓库**。

在镜像名称之后，你可以添加一个标签（Tag）（与使用 docker 或 podman 等命令时的方式相同）。使用 tag 能让你辨识同一镜像序列中的**不同版本**，如果你不指定 tag，Kubernetes 认为你想使用标签 **latest**。

## 更新镜像

当你最初创建一个 Deployment、 StatefulSet、Pod 或者其他包含 Pod 模板的对象时，如果没有显式设定镜像拉取策略，Pod 中所有容器的默认镜像拉取策略是 `IfNotPresent`。

这一策略（IfNotPresent）会使得 kubelet 在镜像已经存在的情况下直接略过拉取镜像的操作。

### 镜像拉取策略

容器的 imagePullPolicy 和镜像的标签会影响 kubelet 尝试拉取（下载）指定的镜像。

以下列表包含了 imagePullPolicy 可以设置的值，以及这些值的效果：

策略 | 默认 | 描述
-|-|-
IfNotPresent | √ | 只有当镜像在本地不存在时才会拉取。
Always | × | 每当 kubelet 启动一个容器时，kubelet 会查询容器的镜像仓库，将镜像解析为一个镜像摘要。<br>如果 kubelet 有一个容器镜像，并且对应的摘要已在本地缓存，kubelet 就会使用其缓存的镜像；<br>否则，kubelet 就会使用解析后的摘要拉取镜像，并使用该镜像来启动容器。
Never | × | Kubelet 不会尝试获取镜像。如果镜像已经以某种方式存在本地，kubelet 会尝试启动容器；否则，会启动失败。

只要能够可靠地访问镜像仓库，那么 **imagePullPolicy: Always** 会更加高效，你的容器运行时（Docker）可以注意到节点上已经存在的镜像层，这样就不需要再次下载。

**注意：**

- 也就是说，虽然 Always 每次都会尝试去拉取镜像，但是因为镜像的各个 Layer 都缓存在本地的，实质上都不会真实的去拉取。这种方式同时还避免了镜像被修改，以及使用 latest 镜像等无法更新的情况。
- 在生产环境中部署容器时，你应该避免使用 :latest 标签，因为这使得正在运行的镜像的版本难以追踪，并且难以正确地回滚。相反，应指定一个有意义的标签，如 v1.42.0。

为了确保 Pod 总是使用相同版本的容器镜像，你可以指定镜像的摘要，即将 `<image-name>:<tag>` 替换为 `<image-name>@<digest>`，例如： 

```txt
image@sha256:45b23dee08af5e43a7fea6c4cf9c25ccf269ee113168c19722f87876677c5cb2
```

当使用镜像标签时，如果该镜像标签所对应的镜像内容被替换了，可能会出现新旧代码混杂在 Pod 中运行的情况。

镜像摘要唯一标识了镜像的特定版本，因此 Kubernetes 每次启动具有指定镜像名称和摘要的容器时，都会运行相同的代码。 通过摘要指定镜像可固定你运行的代码，这样镜像仓库的变化就不会导致版本的混杂。

有一些第三方的准入控制器 在创建 Pod（和 Pod 模板）时产生变更，这样运行的工作负载就是根据镜像摘要，而不是标签来定义的。

无论镜像仓库上的标签发生什么变化，你都想确保你所有的工作负载都运行相同的代码，那么**指定镜像摘要**会很有用。

#### 默认镜像拉取策略

当你（或控制器）向 API 服务器提交一个新的 Pod 时，你的集群会在满足特定条件时设置 imagePullPolicy 字段：

- 如果你省略了 imagePullPolicy 字段，并且容器镜像的标签是 **:latest**， imagePullPolicy 会自动设置为 `Always`。
- 如果你省略了 imagePullPolicy 字段，并且没有指定容器镜像的标签， imagePullPolicy 会自动设置为 `Always`。
- 如果你省略了 imagePullPolicy 字段，并且为容器镜像指定了**非 :latest** 的标签， imagePullPolicy 就会自动设置为 `IfNotPresent`。

**注意：**

- 容器的 imagePullPolicy 的值总是在对象**初次创建**时设置的，如果后来镜像使用的 tag 发生变化，则不会更新。

例如：

1. 如果你用一个 **非 :latest** 的镜像标签创建一个 Deployment， 此时 imagePullPolicy 的策略会被设置为 `IfNotPresent`。
1. 随后更新该 Deployment 的镜像标签为 **:latest**，则 imagePullPolicy 字段不会变成 `Always`。

如果需要修改，你必须手动更改已创建的 Kubernetes 对象的拉取策略。

#### 必要的镜像拉取
