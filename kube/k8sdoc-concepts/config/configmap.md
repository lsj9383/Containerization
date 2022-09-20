# ConfigMap

ConfigMap 是一种 API 对象，用来将**非机密性**的数据保存到**键值对**中。

使用时，Pods 可以将 ConfigMap 用作：

- 环境变量
- 命令行参数
- 存储卷中的配置文件。

ConfigMap 将你的环境配置信息和容器镜像解耦，便于应用配置的修改。

## 动机

使用 ConfigMap 来将你的**配置数据**和**应用程序代码**分开。

比如，假设你正在开发一个应用，它可以在你自己的电脑上（用于开发）和在云上 （用于实际流量）运行。你的代码里有一段是用于查看环境变量 DATABASE_HOST：

- 在本地运行时，你将这个变量设置为 localhost
- 在云上，你将其设置为引用 Kubernetes 集群中的公开数据库组件的服务。

这让你可以获取在云中运行的容器镜像，并且如果有需要的话，在本地调试完全相同的代码。

ConfigMap 在设计上**不是用来保存大量数据**的，具体而言：

- 在 ConfigMap 中保存的数据**不可超过 1 MiB**。
- 如果你需要保存超出此尺寸限制的数据，你可能希望考虑挂载**存储卷**或者使用**独立的数据库或者文件服务**。

## ConfigMap 对象

ConfigMap 是一个 API 对象， 让你可以存储其他对象所需要使用的配置。

和其他 Kubernetes 对象都有一个 spec 不同的是，ConfigMap 的使用 data 和 binaryData 字段。这些字段能够接收键-值对作为其取值。data 和 binaryData 字段都是可选的。data 字段设计用来保存 UTF-8 字符串，而 binaryData 则被设计用来保存二进制数据作为 base64 编码的字串。

从 v1.19 开始，你可以添加一个 immutable 字段到 ConfigMap 定义中， 创建`不可变更的 ConfigMap`。

## ConfigMaps 和 Pods

你可以写一个引用 ConfigMap 的 Pod 的 spec，并根据 ConfigMap 中的数据在该 Pod 中配置容器。

这个 Pod 和 ConfigMap 必须要在同一个 namespace 中。

**注意：**

- 静态 Pod 中的 spec 字段不能引用 ConfigMap 或任何其他 API 对象。

这是一个 ConfigMap 的示例，它的一些键只有一个值：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: game-demo
data:
  # 类属性键；每一个键都映射到一个简单的值
  player_initial_lives: "3"
  ui_properties_file_name: "user-interface.properties"

  # 类文件键
  game.properties: |
    enemy.types=aliens,monsters
    player.maximum-lives=5    
  user-interface.properties: |
    color.good=purple
    color.bad=yellow
    allow.textmode=true
```

你可以使用四种方式来使用 ConfigMap 配置 Pod 中的容器：

- 在容器命令和参数内
- 容器的环境变量
- 在只读卷里面添加一个文件，让应用来读取
- 编写代码在 Pod 中运行，使用 Kubernetes API 来读取 ConfigMap

这些不同的方法适用于不同的数据使用方式：

- 对前三个方法，kubelet 使用 ConfigMap 中的数据在 Pod 中启动容器。
- 第四种方法意味着你必须编写代码才能读取 ConfigMap 和它的数据。然而，由于你是直接使用 Kubernetes API，因此只要 ConfigMap 发生更改，你的应用就能够通过订阅来获取更新，并且在这样的情况发生的时候做出反应。

通过直接进入 Kubernetes API，这个技术也可以让你能够获取到不同的名字空间里的 ConfigMap。

下面是一个 Pod 的示例，它通过使用 game-demo 中的值来配置一个 Pod：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: configmap-demo-pod
spec:
  containers:
    - name: demo
      image: alpine
      command: ["sleep", "3600"]
      env:
        # 定义环境变量
        - name: PLAYER_INITIAL_LIVES # 请注意这里和 ConfigMap 中的键名是不一样的
          valueFrom:
            configMapKeyRef:
              name: game-demo           # 这个值来自 ConfigMap
              key: player_initial_lives # 需要取值的键
        - name: UI_PROPERTIES_FILE_NAME
          valueFrom:
            configMapKeyRef:
              name: game-demo
              key: ui_properties_file_name
      volumeMounts:
      - name: config
        mountPath: "/config"
        readOnly: true
  volumes:
    # 你可以在 Pod 级别设置卷，然后将其挂载到 Pod 内的容器中
    - name: config
      configMap:
        # 提供你想要挂载的 ConfigMap 的名字
        name: game-demo
        # 来自 ConfigMap 的一组键，将被创建为文件
        items:
        - key: "game.properties"
          path: "game.properties"
        - key: "user-interface.properties"
          path: "user-interface.properties"
```

上面的例子：

- 定义了一个卷并将它作为 `/config` 文件夹挂载到 demo 容器内。创建两个文件（虽然 ConfigMap 中包含了四个键，但是这里只指定了两个）：
  - /config/game.properties
  - /config/user-interface.properties

可以通过完全忽略配置 items 数组，让 ConfigMap 中的每个键都会变成一个与该键同名的文件，这样你会得到四个文件。

**注意：**

- 上面这种对 configmap 的使用方法，在 ConfigMap 修改时 volume 可以动态感知到，但可能不会太及时，因为这是一个异步的过程。
- configmap 更新时，环境变量的值不会动态更新。

## 使用 ConfigMap

ConfigMap 可以作为**数据卷挂载**。此外，ConfigMap 也可被系统的其他组件使用，而不一定直接暴露给 Pod。例如：

- ConfigMap 可以保存系统中其他组件要使用的配置数据。

ConfigMap 最常见的用法是为同一 namespace 里某 Pod 中运行的容器执行配置。 你也可以单独使用 ConfigMap。

### 在 Pod 中将 ConfigMap 当做文件使用

要在一个 Pod 的存储卷中使用 ConfigMap:

1. 创建一个 ConfigMap 对象或者使用现有的 ConfigMap 对象。多个 Pod 可以引用同一个 ConfigMap。
1. 修改 Pod 定义，在 **spec.volumes[]** 下添加一个卷。 为该卷设置任意名称，之后将 `spec.volumes[].configMap.name` 字段设置为对你的 ConfigMap 对象的引用。
1. 为每个需要该 ConfigMap 的容器添加一个 `.spec.containers[].volumeMounts[]`：
   - 设置 `.spec.containers[].volumeMounts[].readOnly=true`
   - 将 `.spec.containers[].volumeMounts[].mountPath` 设置为一个未使用的目录名，ConfigMap 的内容将出现在该目录中。
1. 运行你的命令行，以便程序能够从该目录中查找文件。ConfigMap 中的每个 data 键会变成 mountPath 下面的一个文件名。

下面是一个将 ConfigMap 以卷的形式进行挂载的 Pod 示例：

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
    configMap:
      name: configmaps
```

你希望使用的每个 ConfigMap 都需要在 spec.volumes 中被引用到。

如果 Pod 中有多个容器，则每个容器都需要自己的 volumeMounts 块，但针对每个 ConfigMap，你只需要设置一个 spec.volumes 块。

### 被挂载的 ConfigMap 内容会被自动更新

当容器的 volume 中使用的 ConfigMap 被更新时，在容器 volume 中映射的键最终也会被更新：

- kubelet 组件会在每次周期性同步时检查所挂载的 ConfigMap 是否为最新。
- 需要注意的是，对于环境变量引入的配置信息，不会更新。

不过，kubelet 使用的是其本地的高速缓存来获得 ConfigMap 的当前值。

ConfigMap 的自动更新有三种方式：

- 可以通过 watch 操作实现内容传播（默认形式）
- 可实现基于 TTL 的缓存
- 可以直接经过所有请求重定向到 API 服务器

因此，从 ConfigMap 被更新的那一刻算起，到新的主键被投射到 Pod 中去，这一时间跨度可能与 kubelet 的同步周期加上高速缓存的传播延迟相等。这里的传播延迟取决于所选的高速缓存类型 （分别对应 watch 操作的传播延迟、高速缓存的 TTL 时长或者 0）。

**注意：**

- 以环境变量方式使用的 ConfigMap 数据不会被自动更新。 更新这些数据需要重新启动 Pod。
- 使用 ConfigMap 作为 subPath 卷挂载的容器将不会收到 ConfigMap 的更新。

## 不可变更的 ConfigMap

Kubernetes 提供了 **Immutable Secret and ConfigMaps** 特性。

具体而言，Kubernetes 会提供了一种将各个 Secret 和 ConfigMap 设置为**不可变更**的 options。对于大量使用 ConfigMap 的集群 （至少有数万个各不相同的 ConfigMap 给 Pod 挂载）而言，禁止更改 ConfigMap 的数据有以下好处：

- 保护应用，使之免受意外（不想要的）更新所带来的负面影响。
- 通过大幅降低对 kube-apiserver 的压力提升集群性能， 是因为系统会关闭对已标记为不可变更的 ConfigMap 的监视操作。


此功能特性由 ImmutableEphemeralVolumes 特性门控来控制。 你可以通过将 immutable 字段设置为 true 创建不可变更的 ConfigMap。 例如：


```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  ...
data:
  ...
immutable: true
```

一旦某 ConfigMap 被标记为不可变更，则**无法逆转这一变化**，也**无法更改 data 或 binaryData 字段的内容**。

你若要修改数据：

- 只能删除并重建 ConfigMap。因为现有的 Pod 会维护一个已被删除的 ConfigMap 的挂载点，建议重新创建这些 Pods。
