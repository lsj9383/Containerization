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

#### 强制镜像拉取

如果你想总是强制执行拉取，你可以使用下述的一中方式：

- 设置容器的 imagePullPolicy 为 `Always`。
- 省略 imagePullPolicy，并使用 **:latest** 作为镜像标签。当你提交 Pod 时，Kubernetes 会将策略设置为 `Always`。
- 省略 imagePullPolicy 和镜像的标签； 当你提交 Pod 时，Kubernetes 会将策略设置为 `Always`。
- 启用准入控制器 AlwaysPullImages。

### ImagePullBackOff

当 kubelet 使用容器运行时创建 Pod 时，容器可能因为 `ImagePullBackOff` 导致状态为 Waiting。

ImagePullBackOff 状态意味着，因为 Kubernetes 无法拉取容器镜像（原因包括无效的镜像名称，或从私有仓库拉取而没有 imagePullSecret），导致容器无法启动。

- ImagePull，表示镜像拉取
- BackOff，表示 Kubernetes 将继续尝试拉取镜像，并增加回退延迟。

Kubernetes 会增加每次尝试之间的延迟，直到达到编译限制，即 300 秒（5 分钟）。

## 带镜像索引的多架构镜像

除了提供二进制的镜像之外， 容器仓库也可以提供容器镜像索引。

## 使用私有仓库

从私有仓库读取镜像时可能需要密钥。

凭证可以用以下方式提供:

- 配置节点向私有仓库进行身份验证
  - 所有 Pod 均可读取任何已配置的私有仓库
  - 需要集群管理员配置节点
- 预拉镜像
  - 所有 Pod 都可以使用节点上缓存的所有镜像
  - 需要所有节点的 root 访问权限才能进行设置
- 在 Pod 中设置 ImagePullSecrets
  - 只有提供自己密钥的 Pod 才能访问私有仓库
- 特定于厂商的扩展或者本地扩展
  - 如果你在使用定制的节点配置，你（或者云平台提供商）可以实现让节点向容器仓库认证的机制

### 配置节点以向私有仓库进行身份验证

设置凭据的方式取决于您选择使用的 Container Runtime 和私有仓库。

有关配置私有的容器镜像仓库的示例，请参阅 [从私有镜像库中拉取镜像](https://kubernetes.io/zh-cn/docs/tasks/configure-pod-container/pull-image-private-registry/)，该示例使用 Docker Hub 中的私有仓库。

#### config.json 说明

对于 `.docker/config.json` 的解释在原始 Docker 实现和 Kubernetes 的解释之间有所不同。

在 Docker 中，auths 键只能指定 root URL，而 Kubernetes 允许 glob URLs 以及前缀匹配的路径。

这意味着，像这样的 `.docker/config.json` 是有效的：

```json
{
    "auths": {
        "*my-registry.io/images": {
            "auth": "…"
        }
    }
}
```

使用以下语法匹配 root URL（*my-registry.io）：

```txt
pattern:
    { term }

term:
    '*'         匹配任何无分隔符字符序列
    '?'         匹配任意单个非分隔符
    '[' [ '^' ] 字符范围
                  字符集（必须非空）
    c           匹配字符 c （c 不为 '*', '?', '\\', '['）
    '\\' c      匹配字符 c

字符范围: 
    c           匹配字符 c （c 不为 '\\', '?', '-', ']'）
    '\\' c      匹配字符 c
    lo '-' hi   匹配字符范围在 lo 到 hi 之间字符
```

现在镜像拉取操作会将每种有效模式的凭据都传递给 CRI 容器运行时。

例如下面的容器镜像名称会匹配成功：

- my-registry.io/images
- my-registry.io/images/my-image
- my-registry.io/images/another-image
- sub.my-registry.io/images/my-image
- a.sub.my-registry.io/images/my-image

kubelet 为每个找到的凭证的镜像按顺序拉取。这意味着在 config.json 中可能有多项：

```json
{
    "auths": {
        "my-registry.io/images": {
            "auth": "…"
        },
        "my-registry.io/images/subpath": {
            "auth": "…"
        }
    }
}
```

如果一个容器指定了要拉取的镜像 `my-registry.io/images/subpath/my-image`， 并且其中一个失败，kubelet 将尝试从另一个身份验证源下载镜像。

### 提前拉取镜像

默认情况下，kubelet 会尝试从指定的仓库拉取每个镜像。但是，如果容器属性 imagePullPolicy 设置为 IfNotPresent 或者 Never，则会优先使用（对应 IfNotPresent）或者一定使用（对应 Never）本地镜像。

如果你希望使用提前拉取镜像的方法代替仓库认证，就必须保证**集群中所有节点提前拉取的镜像是相同的**。

这一方案可以用来提前载入指定的镜像以提高速度，或者作为向私有仓库执行身份认证的一种替代方案。

所有的 Pod 都可以使用节点上提前拉取的镜像。

**注意：**

- 该方法适用于你能够控制节点配置的场合。

### 在 Pod 上指定 ImagePullSecrets

Kubernetes 支持在 Pod 中设置容器镜像仓库的密钥。imagePullSecrets 必须与 Pod 位于同一个名字空间中。引用的 Secret 必须是 `kubernetes.io/dockercfg` 或 `kubernetes.io/dockerconfigjson` 类型。

使用 Docker Config 创建 Secret，你需要知道用于向仓库进行身份验证的用户名、密码和客户端电子邮件地址，以及它的主机名。

运行以下命令，注意替换适当的大写值：

```sh
$ kubectl create secret docker-registry <name> \
--docker-server=DOCKER_REGISTRY_SERVER \
--docker-username=DOCKER_USER \
--docker-password=DOCKER_PASSWORD \
--docker-email=DOCKER_EMAIL
```

如果你已经有 Docker 凭据文件，则可以将凭据文件导入为 Kubernetes Secret， 而不是执行上面的命令。 [基于已有的 Docker 凭据创建 Secret](https://kubernetes.io/zh-cn/docs/tasks/configure-pod-container/pull-image-private-registry/#registry-secret-existing-credentials) 解释了如何完成这一操作。

**注意：**

- 这是通过私有仓库镜像来运行容器的推荐方法。
- Pod 只能引用位于自身所在名字空间中的 Secret，因此需要针对每个 namespace 重复执行上述过程（否则其他 namespace 的 pod 无法拉镜像）。

#### 在 Pod 中引用 ImagePullSecrets

现在，在创建 Pod 时，可以在 Pod 定义中增加 imagePullSecrets 部分来引用该 Secret。

imagePullSecrets 数组中的每一项只能引用同一 namespace 中的 Secret。

例如：

```sh
cat <<EOF > pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: foo
  namespace: awesomeapps
spec:
  containers:
    - name: foo
      image: janedoe/awesomeapp:v1
  imagePullSecrets:
    - name: myregistrykey
EOF

cat <<EOF >> ./kustomization.yaml
resources:
- pod.yaml
EOF
```

你也可以将此方法与节点级别的 `.docker/config.json` 配置结合使用。 来自不同来源的凭据会被合并。

imagePullSecrets 的本质是：kubelet 将所有 imagePullSecrets 合并为一个虚拟的 `.docker/config.json` 文件。

