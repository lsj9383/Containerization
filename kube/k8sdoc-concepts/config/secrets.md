# Secrets

Secret 是一种包含**少量敏感信息**，例如：

- 密码
- 令牌
- 密钥的对象

使用 Secret 意味着你不需要在应用程序代码中包含这类敏感机密数据。

由于创建 Secret 可以独立于使用它们的 Pod，因此在创建、查看和编辑 Pod 的工作流程中暴露 Secret（及其数据）的风险较小。

Kubernetes 和在集群中运行的应用程序也可以对 Secret 采取额外的预防措施，例如避免将机密数据写入非易失性存储。

Secret **类似于 ConfigMap 但专门用于保存机密数据**。

**注意：**

- 默认情况下，Kubernetes Secret **未加密**地存储在 API 服务器的底层数据存储（etcd）中。这会导致：
  - 任何拥有 API 访问权限的人都可以检索或修改 Secret，任何有权访问 etcd 的人也可以。
  - 任何有权限在命名空间中创建 Pod 的人都可以使用该访问权限读取该命名空间中的任何 Secret。

为了安全地使用 Secret，请至少执行以下步骤：

- 为 Secret 启用静态加密；
- 启用或配置 RBAC 规则来限制读取和写入 Secret 的数据（包括通过间接方式）。需要注意的是，被准许创建 Pod 的人也隐式地被授权获取 Secret 内容。
- 在适当的情况下，还可以使用 RBAC 等机制来限制允许哪些主体创建新 Secret 或替换现有 Secret。

## Secret 的使用

Pod 可以用三种方式之一来使用 Secret：

- 作为挂载到一个或多个容器上的 Volume 中的文件。
- 作为容器的环境变量。
- 由 kubelet 在为 Pod 拉取镜像时使用。

Kubernetes 控制面也使用 Secret，例如：

- 引导令牌 Secret 是一种帮助自动化节点注册的机制。

## Secret 的替代方案

除了使用 Secret 来保护机密数据，你也可以选择一些替代方案。

下面是一些选项：

- 如果你的云原生组件需要执行身份认证来访问你所知道的、在同一 Kubernetes 集群中运行的另一个应用， 你可以使用 ServiceAccount 及其令牌来标识你的客户端身份。
- 你可以运行的第三方工具也有很多，这些工具可以运行在集群内或集群外，提供机密数据管理。例如，这一工具可能是 Pod 通过 HTTPS 访问的一个服务，该服务在客户端能够正确地通过身份认证 （例如，通过 ServiceAccount 令牌）时，提供机密数据内容。
- 就身份认证而言，你可以为 X.509 证书实现一个定制的签名者，并使用 CertificateSigningRequest 来让该签名者为需要证书的 Pod 发放证书。
- 你可以使用一个设备插件来将节点本地的加密硬件暴露给特定的 Pod。例如，你可以将可信任的 Pod 调度到提供可信平台模块（Trusted Platform Module，TPM）的节点上。 这类节点是另行配置的。
- 你还可以将如上选项的两种或多种进行组合，包括直接使用 Secret 对象本身也是一种选项。

## 使用 Secret

### 创建 Secret

- [使用 kubectl 命令来创建 Secret](https://kubernetes.io/zh-cn/docs/tasks/configmap-secret/managing-secret-using-kubectl/)
- [基于配置文件来创建 Secret](https://kubernetes.io/zh-cn/docs/tasks/configmap-secret/managing-secret-using-config-file/)
- [使用 kustomize 来创建 Secret](https://kubernetes.io/zh-cn/docs/tasks/configmap-secret/managing-secret-using-kustomize/)

**注意：**

- 使用 yaml 创建 secret 时，需要使用 base64 encode 后的值。在挂载的时候，会将值 base64 decode 后放到文件系统中。

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
data:
  USER_NAME: YWRtaW4=           # admin
  PASSWORD: MWYyZDFlMmU2N2Rm    # 1f2d1e2e67df
```

也有另外一种使用非 base64 编码创建 secret 的方法，本质上是工具帮忙做了 Base64：

```yaml
# 先创建文件
$ echo -n 'admin' > ./username.txt
$ echo -n '1f2d1e2e67df' > ./password.txt

# 创建 secret（不含后缀的文件名是 secret key 的名称）
$ kubectl create secret generic mysecret \
  --from-file=./username.txt \
  --from-file=./password.txt

# 也可以人为指定 secret key
$ kubectl create secret generic mysecret \
  --from-file=username=./username.txt \
  --from-file=password=./password.txt

# 验证 secret：
$ kubectl get secret mysecret -o jsonpath='{.data}'
{"password":"MWYyZDFlMmU2N2Rm","username":"YWRtaW4="}
```

#### 对 Secret 名称与数据的约束

Secret 对象的名称必须是合法的 DNS 子域名。

在为创建 Secret 编写配置文件时，你可以设置 data 与/或 stringData 字段。 data 和 stringData 字段都是可选的。data 字段中所有键值都必须是 base64 编码的字符串。如果不希望执行这种 base64 字符串的转换操作，你可以选择设置 stringData 字段，其中可以使用任何字符串作为其取值。

data 和 stringData 中的键名只能包含字母、数字、-、_ 或 . 字符。 stringData 字段中的所有键值对都会在内部被合并到 data 字段中。 如果某个主键同时出现在 data 和 stringData 字段中，stringData 所指定的键值具有高优先级。

#### 尺寸限制

每个 Secret 的尺寸**最多为 1MiB**。

施加这一限制是为了避免用户创建非常大的 Secret， 进而导致 API 服务器和 kubelet 内存耗尽。

代替方法是：创建很多 Secret，不过创建很多小的 Secret 也可能耗尽内存。你可以使用资源配额来约束每个名字空间中 Secret（或其他资源）的个数。

#### 编辑 Secret

你可以使用 kubectl 来编辑一个已有的 Secret：

```yaml
kubectl edit secrets mysecret
```

这一命令会启动你的默认编辑器，允许你更新 data 字段中存放的 base64 编码的 Secret 值； 例如：

```yaml
# 请编辑以下对象。以 `#` 开头的几行将被忽略，
# 且空文件将放弃编辑。如果保存此文件时出错，
# 则重新打开此文件时也会有相关故障。
apiVersion: v1
data:
  username: YWRtaW4=
  password: MWYyZDFlMmU2N2Rm
kind: Secret
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: { ... }
  creationTimestamp: 2020-01-22T18:41:56Z
  name: mysecret
  namespace: default
  resourceVersion: "164619"
  uid: cfee02d6-c137-11e5-8d73-42010af00002
type: Opaque
```

这一示例清单定义了一个 Secret，其 data 字段中包含两个主键：username 和 password。

清单中的字段值是 Base64 字符串，不过，当你在 Pod 中使用 Secret 时，kubelet 为 Pod 及其中的容器提供的是解码后的数据。

你可以在一个 Secret 中打包多个主键和数值，也可以选择使用多个 Secret， 完全取决于哪种方式最方便。

### 使用 Secret

Secret 可以以数据卷的形式挂载，也可以作为环境变量暴露给 Pod 中的容器使用。Secret 也可用于系统中的其他部分，而不是一定要直接暴露给 Pod。例如:

- Secret 也可以包含系统中其他部分在替你与外部系统交互时要使用的凭证数据。

Kubernetes 会检查 Secret 的卷数据源，确保所指定的对象引用确实指向类型为 Secret 的对象。因此，**如果 Pod 依赖于某 Secret，该 Secret 必须先于 Pod 被创建**，但实操的时候，也可以指定 Secret 可选。

如果 Secret 内容无法取回（可能因为 Secret 尚不存在或者临时性地出现 API 服务器网络连接问题）：

- kubelet 会周期性地重试 Pod 运行操作。
- kubelet 也会为该 Pod 报告 Event 事件，给出读取 Secret 时遇到的问题细节。

#### 可选的 Secret

当你定义一个基于 Secret 的**环境变量**时，你可以将其标记为可选（我自测将 secret 用于 volume 也可以设置为可选）。默认情况下，所引用的 Secret 都是必需的。

**注意：**

- 只有所有必选的 Secret 都可用时，Pod 中的容器才能启动运行。
- 如果 Pod 引用了 Secret 中的特定主键，而虽然 Secret 本身存在，对应的主键不存在， Pod 启动也会失败。

### 在 Pod 中以文件形式使用 Secret

如果你希望在 Pod 中访问 Secret 内的数据，一种方式是让 Kubernetes 将 Secret 以 Pod 中一个或多个容器的文件系统中的文件的形式呈现出来。

要配置这种行为，你需要：

1. 创建一个 Secret 或者使用已有的 Secret。多个 Pod 可以引用同一个 Secret。
1. 更改 Pod 定义，在 `.spec.volumes[]` 下添加一个卷。根据需要为卷设置其名称， 并将 `.spec.volumes[].secret.secretName` 字段设置为 Secret 对象的名称。
1. 为每个需要该 Secret 的容器添加 `.spec.containers[].volumeMounts[]`。 并将 `.spec.containers[].volumeMounts[].readOnly` 设置为 true， 将 `.spec.containers[].volumeMounts[].mountPath` 设置为希望 Secret 被放置的、目前尚未被使用的路径名。
1. 运行你的命令行，以便程序读取所设置的目录下的文件。Secret 的 data 映射中的**每个主键都成为 mountPath 下面的文件名**。

下面是一个通过卷来挂载名为 mysecret 的 Secret 的 Pod 示例：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
  - name: mypod
    image: redis
    volumeMounts:
    - name: foo
      mountPath: "/etc/foo"
      readOnly: true
  volumes:
  - name: foo
    secret:
      secretName: mysecret
      optional: false   # 默认设置，意味着 "mysecret" 必须已经存在
                        # 如果设置为 true，意味着可以 mysecret 可以不存在
```

你要访问的每个 Secret 都需要通过 `.spec.volumes` 来引用。

**注意：**

- Kubernetes v1.22 版本之前都会自动创建用来访问 Kubernetes API 的凭证。 这一老的机制是基于创建可被挂载到 Pod 中的令牌 Secret 来实现的。
- 在最近的版本中（包括 Kubernetes v1.25），API 凭据是直接通过 TokenRequest API 来获得的，这一凭据会使用投射卷挂载到 Pod 中。使用这种方式获得的令牌有确定的生命期，并且在挂载它们的 Pod 被删除时自动作废。
- 你仍然可以手动创建服务账号令牌。例如，当你需要一个永远都不过期的令牌时。不过，仍然建议使用 TokenRequest 子资源来获得访问 API 服务器的令牌。你可以使用 kubectl create token 命令调用 TokenRequest API 获得令牌。

#### 将 Secret 键投射到特定目录

你也可以控制 Secret 键所投射到的卷中的路径。你可以使用 `.spec.volumes[].secret.items` 字段来更改每个主键的目标路径：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
  - name: mypod
    image: redis
    volumeMounts:
    - name: foo
      mountPath: "/etc/foo"
      readOnly: true
  volumes:
  - name: foo
    secret:
      secretName: mysecret
      items:
      - key: username
        path: my-group/my-username
```

将发生的事情如下：

1. mysecret 中的键 username 会出现在容器中的路径为 `/etc/foo/my-group/my-username`， 而不是 `/etc/foo/username`。
1. Secret 对象的 password 键不会被投射。
1. 如果使用了 `.spec.volumes[].secret.items`，则只有 items 中指定了的主键会被投射。如果要通过这种方式使用 Secret 中的所有主键，则需要将它们全部枚举到 items 字段中。
1. 如果忽略 `.spec.volumes[].secret.items`，那么会所有键都进行投射。

#### Secret 文件的访问权限

你可以为某个 Secret 主键设置 POSIX 文件访问权限位。如果你不指定访问权限，默认会使用 0644。 你也可以为整个 Secret 卷设置默认的访问模式，然后再根据需要在主键层面重载。

例如，你可以像下面这样设置默认的模式：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
  - name: mypod
    image: redis
    volumeMounts:
    - name: foo
      mountPath: "/etc/foo"
  volumes:
  - name: foo
    secret:
      secretName: mysecret
      defaultMode: 0400
```

该 Secret 被挂载在 `/etc/foo` 下，Secret 卷挂载所创建的所有文件的访问模式都是 `0400`。

#### 使用来自卷中的 Secret 值

在挂载了 Secret 卷的容器内，Secret 的主键都呈现为文件。

下面是在上例中的容器内执行命令的结果：

```sh
$ ls /etc/foo/
username
password

$ cat /etc/foo/username
admin

$ cat /etc/foo/password
1f2d1e2e67df
```

#### 挂载的 Secret 是被自动更新的

当卷中包含来自 Secret 的数据，而对应的 Secret 被更新，Kubernetes 会跟踪到这一操作并更新卷中的数据。更新的方式是异步的，会保证最终一致性。

**注意：**

- 对于以 `subPath` 形式挂载 Secret 卷的容器而言，它们无法收到自动的 Secret 更新。

具体而言：

- Kubelet 组件会维护一个缓存，在其中保存节点上 Pod 卷中使用的 Secret 的当前主键和取值。
- 你可以配置 kubelet 如何检测所缓存数值的变化。
- kubelet 配置中的 configMapAndSecretChangeDetectionStrategy 字段控制 kubelet 所采用的策略。默认的策略是 Watch。
- 对 Secret 的更新操作既可以通过 API 的 watch 机制（默认）来传播， 基于设置了生命期的缓存获取，也可以通过 kubelet 的同步回路来从集群的 API 服务器上轮询获取。

因此，从 Secret 被更新到新的主键被投射到 Pod 中，中间存在一个延迟。这一延迟的上限是 kubelet 的同步周期加上缓存的传播延迟，其中缓存的传播延迟取决于所选择的缓存类型。对应上一段中提到的几种传播机制，延迟时长为 watch 的传播延迟、所配置的缓存 TTL 或者对于直接轮询而言是零。

### 以环境变量的方式使用 Secret

如果需要在 Pod 中以环境变量的形式使用 Secret：

1. 创建 Secret（或者使用现有 Secret）。多个 Pod 可以引用同一个 Secret。
1. 更改 Pod 定义，在要使用 Secret 键值的每个容器中添加与所使用的主键对应的环境变量。 读取 Secret 主键的环境变量应该在 `env[].valueFrom.secretKeyRef` 中填写 Secret 的名称和主键名称。
1. 更改你的镜像或命令行，以便程序读取环境变量中保存的值。


下面是一个通过环境变量来使用 Secret 的示例 Pod：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-env-pod
spec:
  containers:
  - name: mycontainer
    image: redis
    env:
      - name: SECRET_USERNAME
        valueFrom:
          secretKeyRef:
            name: mysecret
            key: username
            optional: false # 此值为默认值；意味着 "mysecret"
                            # 必须存在且包含名为 "username" 的主键
      - name: SECRET_PASSWORD
        valueFrom:
          secretKeyRef:
            name: mysecret
            key: password
            optional: false # 此值为默认值；意味着 "mysecret"
                            # 必须存在且包含名为 "password" 的主键
  restartPolicy: Never
```

#### 非法环境变量

对于通过 envFrom 字段来填充环境变量的 Secret 而言， 如果其中包含的主键不能被当做合法的环境变量名，这些主键会被忽略掉。**Pod 仍然可以启动**。

如果你定义的 Pod 中包含非法的变量名称，则 Pod 可能启动失败， 会形成 reason 为 `InvalidVariableNames` 的事件，以及列举被略过的非法主键的消息。

下面的例子中展示了一个 Pod，引用的是名为 mysecret 的 Secret， 其中包含两个非法的主键：1badkey 和 2alsobad。

```sh
$ kubectl get events
LASTSEEN   FIRSTSEEN   COUNT     NAME            KIND      SUBOBJECT                         TYPE      REASON
0s         0s          1         dapi-test-pod   Pod                                         Warning   InvalidEnvironmentVariableNames   kubelet, 127.0.0.1      Keys [1badkey, 2alsobad] from the EnvFrom secret default/mysecret were skipped since they are considered invalid environment variable names.
```

#### 通过环境变量使用 Secret 值

在通过环境变量来使用 Secret 的容器中，Secret 主键展现为普通的环境变量。这些变量的取值是 Secret 数据的 Base64 解码值。

下面是在前文示例中的容器内执行命令的结果：

```sh
$ echo "$SECRET_USERNAME"
admin

$ echo "$SECRET_PASSWORD"
1f2d1e2e67df
```

**注意：**

- 如果容器通过环境变量来使用 Secret，那么 Secret 更新在容器内是看不到的（不会自动更新环境变量）。只有容器被重启。有一些第三方的解决方案，能够在 Secret 发生变化时触发容器重启。


### 容器镜像拉取使用 Secret

如果你尝试从私有仓库拉取容器镜像，你需要一种方式让每个节点上的 kubelet 能够完成与镜像库的身份认证。你可以通过配置拉取的 Secret 来实现这点。Secret 是在 Pod 层面来配置的。

Pod 的 imagePullSecrets 字段是一个对 Pod 所在的名字空间中的 Secret 的引用列表。你可以使用 imagePullSecrets 来将镜像仓库访问凭据传递给 kubelet。kubelet 使用这个信息来替你的 Pod 拉取私有镜像。

#### 使用 imagePullSecrets

imagePullSecrets 字段是一个列表，包含对同一名字空间中 Secret 的引用。 你可以使用 imagePullSecrets 将包含 Docker（或其他）镜像仓库密码的 Secret 传递给 kubelet。kubelet 使用此信息来替 Pod 拉取私有镜像。 参阅 PodSpec API 进一步了解 imagePullSecrets 字段。

### 在静态 Pod 中使用 Secret

你不可以在静态 Pod 中使用 ConfigMap 或 Secret。

## Secret 的类型

创建 Secret 时，你可以使用 Secret 资源的 type 字段，或者与其等价的 kubectl 命令行参数（如果有的话）为其设置类型。 Secret 类型有助于对 Secret 数据进行编程处理。

Kubernetes 提供若干种内置的类型，用于一些常见的使用场景。针对这些类型，Kubernetes 所执行的合法性检查操作以及对其所实施的限制各不相同。

内置类型 | 用法
-|-
Opaque（默认） | 用户定义的任意数据
kubernetes.io/service-account-token | 服务账号令牌
kubernetes.io/dockercfg	~/.dockercfg | 文件的序列化形式
kubernetes.io/dockerconfigjson | ~/.docker/config.json 文件的序列化形式
kubernetes.io/basic-auth | 用于基本身份认证的凭据
kubernetes.io/ssh-auth | 用于 SSH 身份认证的凭据
kubernetes.io/tls | 用于 TLS 客户端或者服务器端的数据
bootstrap.kubernetes.io/token | 启动引导令牌数据

其实 type 可以设置任何值（如果 type 值为空字符串，则被视为 Opaque 类型）。

Kubernetes 并不对类型的名称作任何限制。不过，如果你要使用内置类型之一，则你必须满足为该类型所定义的所有要求。

如果你要定义一种公开使用的 Secret 类型，请遵守 Secret 类型的约定和结构，例如：cloud-hosting.example.net/cloud-api-credentials。

### Opaque Secret

当 Secret 配置文件中未作显式设定时，默认的 Secret 类型是 Opaque。

当你使用 kubectl 来创建一个 Secret 时，你会使用 generic 子命令来标明要创建的是一个 Opaque 类型 Secret。 例如，下面的命令会创建一个空的 Opaque 类型 Secret 对象：

```yaml
$ kubectl create secret generic empty-secret

$ kubectl get secret empty-secret
NAME           TYPE     DATA   AGE
empty-secret   Opaque   0      2m6s
```

DATA 列显示 Secret 中保存的数据条目个数。 在这个例子种，0 意味着你刚刚创建了一个空的 Secret。

### 服务账号令牌 Secret

类型为 `kubernetes.io/service-account-token` 的 Secret 用来存放标识`服务账号的令牌凭据`。

**注意：**

- 从 v1.22 开始，这种类型的 Secret 不再被用来向 Pod 中加载凭据数据，建议通过 TokenRequest API 来获得令牌，而不是使用服务账号令牌 Secret 对象。
- 通过 TokenRequest API 获得的令牌比保存在 Secret 对象中的令牌更加安全，因为这些令牌有着被限定的生命期，并且不会被其他 API 客户端读取。你可以使用 kubectl create token 命令调用 TokenRequest API 获得令牌。

只有在你无法使用 TokenRequest API 来获取令牌，并且你能够接受因为将永不过期的令牌凭据写入到可读取的 API 对象而带来的安全风险时，才应该创建服务账号令牌 Secret 对象。

使用这种 Secret 类型时，你需要确保对象的注解 `kubernetes.io/service-account-name` 被设置为某个已有的服务账号名称。如果你同时负责 ServiceAccount 和 Secret 对象的创建，应该先创建 ServiceAccount 对象。

当 Secret 对象被创建之后，某个 Kubernetes控制器会填写 Secret 的其它字段，例如 kubernetes.io/service-account.uid 注解以及 data 字段中的 token 键值，使之包含实际的令牌内容。

下面的配置实例声明了一个服务账号令牌 Secret：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: secret-sa-sample
  annotations:
    kubernetes.io/service-account.name: "sa-name"
type: kubernetes.io/service-account-token
data:
  # 你可以像 Opaque Secret 一样在这里添加额外的键/值偶对
  extra: YmFyCg==
```

创建了 Secret 之后，等待 Kubernetes 在 data 字段中填充 token 主键。

参考 [ServiceAccount 文档](https://kubernetes.io/zh-cn/docs/tasks/configure-pod-container/configure-service-account/)了解服务账号的工作原理。

你也可以查看 Pod 资源中的 automountServiceAccountToken 和 serviceAccountName 字段文档，进一步了解从 Pod 中引用服务账号凭据。

#### Docker 配置 Secret

你可以使用下面两种 type 值之一来创建 Secret，用以存放用于访问容器镜像仓库的凭据：

- `kubernetes.io/dockercfg`
- `kubernetes.io/dockerconfigjson`

`kubernetes.io/dockercfg` 是一种保留类型，用来存放 `~/.dockercfg` 文件的序列化形式。 该文件是配置 Docker 命令行的一种老旧形式。使用此 Secret 类型时，你需要确保 Secret 的 data 字段中包含名为 .dockercfg 的主键，其对应键值是用 base64 编码的某 `~/.dockercfg` 文件的内容。

下面是一个 kubernetes.io/dockercfg 类型 Secret 的示例：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: secret-dockercfg
type: kubernetes.io/dockercfg
data:
  .dockercfg: |
        "<base64 encoded ~/.dockercfg file>"
```

**注意：**

- 如果你不希望执行 base64 编码转换，可以使用 stringData 字段代替。

当你使用清单文件来创建这两类 Secret 时，API 服务器会检查 data 字段中是否存在所期望的主键， 并且验证其中所提供的键值是否是合法的 JSON 数据。 不过，API 服务器不会检查 JSON 数据本身是否是一个合法的 Docker 配置文件内容。

当你没有 Docker 配置文件，或者你想使用 kubectl 创建一个 Secret 来访问容器仓库时，你可以这样做：

```sh
kubectl create secret docker-registry secret-tiger-docker \
  --docker-email=tiger@acme.example \
  --docker-username=tiger \
  --docker-password=pass1234 \
  --docker-server=my-registry.example:5000
```

上面的命令创建一个类型为 kubernetes.io/dockerconfigjson 的 Secret。 如果你对 .data.dockerconfigjson 内容进行转储并执行 base64 解码：

```sh
$ kubectl get secret secret-tiger-docker -o jsonpath='{.data.*}' | base64 -d

# 那么输出等价于这个 JSON 文档（这也是一个有效的 Docker 配置文件）：
{
  "auths": {
    "my-registry.example:5000": {
      "username": "tiger",
      "password": "pass1234",
      "email": "tiger@acme.example",
      "auth": "dGlnZXI6cGFzczEyMzQ="
    }
  }
}
```

**注意：**

- auths 值是 base64 编码的，其内容被屏蔽但未被加密。 任何能够读取该 Secret 的人都可以了解镜像库的访问令牌。

### 基本身份认证 Secret

`kubernetes.io/basic-auth` 类型用来存放用于基本身份认证所需的凭据信息。

使用这种 Secret 类型时，Secret 的 data 字段必须包含以下两个键之一：

- username: 用于身份认证的用户名；
- password: 用于身份认证的密码或令牌。

以上两个键的键值都是 base64 编码的字符串。当然你也可以在创建 Secret 时使用 stringData 字段来提供明文形式的内容。

以下清单是基本身份验证 Secret 的示例：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: secret-basic-auth
type: kubernetes.io/basic-auth
stringData:
  username: admin      # kubernetes.io/basic-auth 类型的必需字段
  password: t0p-Secret # kubernetes.io/basic-auth 类型的必需字段
```

提供基本身份认证类型的 Secret 仅仅是出于方便性考虑。 你也可以使用 Opaque 类型来保存用于基本身份认证的凭据。 不过，使用预定义的、公开的 Secret 类型（kubernetes.io/basic-auth） 有助于**帮助其他用户理解 Secret 的目的**，并且对其中存在的主键形成一种约定。 API 服务器会检查 Secret 配置中是否提供了所需要的主键。

### SSH 身份认证 Secret

Kubernetes 所提供的内置类型 `kubernetes.io/ssh-auth` 用来存放 SSH 身份认证中所需要的凭据。

使用这种 Secret 类型时，你就必须在其 data （或 stringData） 字段中提供一个 `ssh-privatekey 键值对`，作为要使用的 SSH 凭据。

下面的清单是一个 SSH 公钥/私钥身份认证的 Secret 示例：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: secret-ssh-auth
type: kubernetes.io/ssh-auth
data:
  # 此例中的实际数据被截断
  ssh-privatekey: |
          MIIEpQIBAAKCAQEAulqb/Y ...
```

提供 SSH 身份认证类型的 Secret 仅仅是出于用户方便性考虑。 你也可以使用 Opaque 类型来保存用于 SSH 身份认证的凭据。 不过，使用预定义的、公开的 Secret 类型（kubernetes.io/ssh-auth） 有助于其他人理解你的 Secret 的用途，也可以就其中包含的主键名形成约定。 API 服务器确实会检查 Secret 配置中是否提供了所需要的主键。

**注意：**

- SSH 私钥自身无法建立 SSH 客户端与服务器端之间的可信连接。 需要其它方式来建立这种信任关系，以缓解“中间人（Man In The Middle）” 攻击，例如向 ConfigMap 中添加一个 known_hosts 文件。

### TLS Secret

Kubernetes 提供一种内置的 kubernetes.io/tls Secret 类型，用来存放 TLS 场合通常要使用的证书及其相关密钥。 TLS Secret 的一种典型用法是为 Ingress 资源配置传输过程中的数据加密，不过也可以用于其他资源或者直接在负载中使用。 当使用此类型的 Secret 时，Secret 配置中的 data （或 stringData）字段必须包含 tls.key 和 tls.crt 主键，尽管 API 服务器实际上并不会对每个键的取值作进一步的合法性检查。

下面的 YAML 包含一个 TLS Secret 的配置示例：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: secret-tls
type: kubernetes.io/tls
data:
  # 此例中的数据被截断
  tls.crt: |
        MIIC2DCCAcCgAwIBAgIBATANBgkqh ...
  tls.key: |
        MIIEpgIBAAKCAQEA7yn3bRHQ5FHMQ ...
```

提供 TLS 类型的 Secret 仅仅是出于用户方便性考虑。 你也可以使用 Opaque 类型来保存用于 TLS 服务器与/或客户端的凭据。 不过，使用内置的 Secret 类型的有助于对凭据格式进行归一化处理，并且 API 服务器确实会检查 Secret 配置中是否提供了所需要的主键。

当使用 kubectl 来创建 TLS Secret 时，你可以像下面的例子一样使用 tls 子命令：

```sh
kubectl create secret tls my-tls-secret \
  --cert=path/to/cert/file \
  --key=path/to/key/file
```

这里的公钥/私钥对都必须事先已存在。用于 --cert 的公钥证书必须是 RFC 7468 中 5.1 节 中所规定的 DER 格式，且与 --key 所给定的私钥匹配。 私钥必须是 DER 格式的 PKCS #8。

**注意：**

- 类型为 kubernetes.io/tls 的 Secret 中包含密钥和证书的 DER 数据，以 Base64 格式编码。 如果你熟悉私钥和证书的 PEM 格式，base64 与该格式相同，只是你需要略过 PEM 数据中所包含的第一行和最后一行。

例如，对于证书而言，你不要包含 `--------BEGIN CERTIFICATE-----` 和 `-------END CERTIFICATE----` 这两行。

### 启动引导令牌 Secret

通过将 Secret 的 type 设置为 `bootstrap.kubernetes.io/token` 可以创建启动引导令牌类型的 Secret。这种类型的 Secret 被设计用来支持**节点的启动引导过程**。其中包含用来为周知的 ConfigMap 签名的令牌。

启动引导令牌 Secret 通常创建于 kube-system 名字空间内，并以 `bootstrap-token-<令牌 ID>` 的形式命名； 其中 `<令牌 ID>` 是一个由 6 个字符组成的字符串，用作令牌的标识。

以 Kubernetes 清单文件的形式，某启动引导令牌 Secret 可能看起来像下面这样：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: bootstrap-token-5emitj
  namespace: kube-system
type: bootstrap.kubernetes.io/token
data:
  auth-extra-groups: c3lzdGVtOmJvb3RzdHJhcHBlcnM6a3ViZWFkbTpkZWZhdWx0LW5vZGUtdG9rZW4=
  expiration: MjAyMC0wOS0xM1QwNDozOToxMFo=
  token-id: NWVtaXRq
  token-secret: a3E0Z2lodnN6emduMXAwcg==
  usage-bootstrap-authentication: dHJ1ZQ==
  usage-bootstrap-signing: dHJ1ZQ==
```

启动引导令牌类型的 Secret 会在 data 字段中包含如下主键：

- `token-id`：由 6 个随机字符组成的字符串，作为令牌的标识符。必需。
- `token-secret`：由 16 个随机字符组成的字符串，包含实际的令牌机密。必需。
- `description`：供用户阅读的字符串，描述令牌的用途。可选。
- `expiration`：一个使用 RFC3339 来编码的 UTC 绝对时间，给出令牌要过期的时间。可选。
- `usage-bootstrap-<usage>`：布尔类型的标志，用来标明启动引导令牌的其他用途。
- `auth-extra-groups`：用逗号分隔的组名列表，身份认证时除被认证为 `system:bootstrappers` 组之外，还会被添加到所列的用户组中。

上面的 YAML 文件可能看起来令人费解，因为其中的数值均为 base64 编码的字符串。实际上，你完全可以使用下面的 YAML 来创建一个一模一样的 Secret：

```yaml
apiVersion: v1
kind: Secret
metadata:
  # 注意 Secret 的命名方式
  name: bootstrap-token-5emitj
  # 启动引导令牌 Secret 通常位于 kube-system 名字空间
  namespace: kube-system
type: bootstrap.kubernetes.io/token
stringData:
  auth-extra-groups: "system:bootstrappers:kubeadm:default-node-token"
  expiration: "2020-09-13T04:39:10Z"
  # 此令牌 ID 被用于生成 Secret 名称
  token-id: "5emitj"
  token-secret: "kq4gihvszzgn1p0r"
  # 此令牌还可用于 authentication （身份认证）
  usage-bootstrap-authentication: "true"
  # 且可用于 signing （证书签名）
  usage-bootstrap-signing: "true"
```

## Secret 的信息安全问题

尽管 ConfigMap 和 Secret 的工作方式类似，但 Kubernetes 对 Secret 有一些额外的保护。

Secret 通常保存重要性各异的数值，其中很多都可能会导致 Kubernetes 中 （例如，服务账号令牌）或对外部系统的特权提升。 即使某些个别应用能够推导它期望使用的 Secret 的能力， 同一名字空间中的其他应用可能会让这种假定不成立。

- 只有当某个节点上的 Pod 需要某 Secret 时，对应的 Secret 才会被发送到该节点上。
- 如果将 Secret 挂载到 Pod 中，kubelet 会将数据的副本保存在在 tmpfs 中，这样机密的数据不会被写入到持久性存储中。一旦依赖于该 Secret 的 Pod 被删除，kubelet 会删除来自于该 Secret 的机密数据的本地副本。
- 同一个 Pod 中可能包含多个容器。默认情况下，你所定义的容器只能访问默认 ServiceAccount 及其相关 Secret。你必须显式地定义环境变量或者将卷映射到容器中，才能为容器提供对其他 Secret 的访问。

针对同一节点上的多个 Pod 可能有多个 Secret。不过，只有某个 Pod 所请求的 Secret 才有可能对 Pod 中的容器可见。因此，一个 Pod 不会获得访问其他 Pod 的 Secret 的权限。

**注意：**

- 节点上的所有特权容器都可能访问到该节点上使用的所有 Secret。

### 针对开发人员的安全性建议

应用在从环境变量或卷中读取了机密信息内容之后仍要对其进行保护。例如， 你的应用应该避免用明文的方式将 Secret 数据写入日志，或者将其传递给不可信的第三方。
如果你在一个 Pod 中定义了多个容器，而只有一个容器需要访问某 Secret， 定义卷挂载或环境变量配置时，应确保其他容器无法访问该 Secret。
如果你通过清单来配置某 Secret， Secret 数据以 Base64 的形式编码，将此文件共享，或者将其检入到某源码仓库， 都意味着 Secret 对于任何可以读取清单的人都是可见的。 Base64 编码 不是 一种加密方法，与明文相比没有任何安全性提升。
部署与 Secret API 交互的应用时，你应该使用 RBAC 这类鉴权策略来限制访问。
在 Kubernetes API 中，名字空间内对 Secret 对象的 watch 和 list 请求是非常强大的能力。 在可能的时候应该避免授予这类访问权限，因为通过列举 Secret， 客户端能够查看对应名字空间内所有 Secret 的取值。

### 针对集群管理员的安全性建议

**注意：**

- 能够创建使用 Secret 的 Pod 的用户也可以查看该 Secret 的取值。
- 即使集群策略不允许某用户直接读取 Secret 对象，这一用户仍然可以**通过运行一个 Pod 来访问 Secret 的内容**。

对管理员建议：

- 保留（使用 Kubernetes API）对集群中所有 Secret 对象执行 watch 或 list 操作的能力，这样只有特权级最高、系统级别的组件能够执行这类操作。
- 在部署需要通过 Secret API 交互的应用时，你应该通过使用 RBAC 这类鉴权策略来限制访问。
- 在 API 服务器上，对象（包括 Secret）会被持久化到 etcd 中； 因此：
  - 只应准许集群管理员访问 etcd（包括只读访问）；
  - 为 Secret 对象启用[静态加密](https://kubernetes.io/zh-cn/docs/tasks/administer-cluster/encrypt-data/)， 这样这些 Secret 的数据就不会以明文的形式保存到 etcd 中；
  - 当 etcd 的持久化存储不再被使用时，请考虑彻底擦除存储介质；
  - 如果存在多个 etcd 实例，**请确保 etcd 使用 SSL/TLS 来完成其对等通信**。
