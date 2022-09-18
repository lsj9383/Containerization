# 临时卷

有些应用程序需要额外的存储，但并不关心数据在重启后是否仍然可用。例如：

- 缓存服务经常受限于内存大小，而且可以将不常用的数据转移到比内存慢的存储中，对总体性能的影响并不大。
- 另有些应用程序需要以文件形式注入的只读数据，比如配置数据或密钥。

临时卷 就是为此类用例设计的。因为卷会遵从 Pod 的生命周期，与 Pod 一起创建和删除，所以停止和重新启动 Pod 时，不会受持久卷在何处可用的限制。

临时卷在 Pod 规约中以**内联**方式定义，这简化了应用程序的部署和管理（即和 Pod 定义在一起）。

## 临时卷的类型

Kubernetes 为了不同的用途，支持几种不同类型的临时卷：

- emptyDir：Pod 启动时为空，**存储空间来自本地的 kubelet 根目录（通常是根磁盘）或内存**。
- configMap、 downwardAPI、 secret： 将不同类型的 Kubernetes 数据注入到 Pod 中。
- CSI 临时卷： 类似于前面的卷类型，但由专门支持此特性的指定 CSI 驱动程序提供。
- 通用临时卷： 它可以由所有支持 PV 的存储驱动程序提供。

**注意：**

- emptyDir、configMap、downwardAPI、secret 是作为 本地临时存储 提供的。它们由各个节点上的 kubelet 管理。
- **CSI 临时卷** 必须由第三方 CSI 存储驱动程序提供。
- **通用临时卷** 可以由第三方 CSI 存储驱动程序提供，也可以由支持动态制备的任何其他存储驱动程序提供。一些专门为 CSI 临时卷编写的 CSI 驱动程序，不支持动态制备：因此这些驱动程序不能用于通用临时卷。

## 通用临时卷

通用临时卷类似于 emptyDir，因为它为每个 Pod 提供临时数据存放目录，在最初制备完毕时一般为空。不过通用临时卷也有一些额外的功能特性：

- 存储可以是本地的，也可以是网络连接的。
- 卷可以有固定的大小，Pod 不能超量使用。
- 卷可能有一些初始数据，这取决于驱动程序和参数。
- 支持典型的卷操作，前提是相关的驱动程序也支持该操作，包括：快照、 克隆、 调整大小和[存储容量跟踪](https://kubernetes.io/zh-cn/docs/concepts/storage/storage-capacity/)。


示例：

```yaml
kind: Pod
apiVersion: v1
metadata:
  name: my-app
spec:
  containers:
    - name: my-frontend
      image: busybox:1.28
      volumeMounts:
      - mountPath: "/scratch"
        name: scratch-volume
      command: [ "sleep", "1000000" ]
  volumes:
    - name: scratch-volume
      ephemeral:
        volumeClaimTemplate:
          metadata:
            labels:
              type: my-frontend-volume
          spec:
            accessModes: [ "ReadWriteOnce" ]
            storageClassName: "scratch-storage-class"
            resources:
              requests:
                storage: 1Gi
```

可见，通用临时卷本质上就是编写 PVC 的模板，由集群自动生成 PVC，并将其挂载到 Pod 中。因为是临时卷，所以其中的数据生命周期跟随 Pod。

## 生命周期和 PVC

关键的设计思想是在 Pod 的卷来源中允许使用 **卷申领的参数**，PVC 的标签、注解和整套字段集均被支持。创建这样一个 Pod 后，临时卷控制器在 Pod 所属的 namespace 中创建一个实际的 PVC 对象， 并确保删除 Pod 时，同步删除 PVC。

如上设置将触发卷的绑定与/或制备，相应动作或者在 StorageClass 使用即时卷绑定时立即执行， 或者当 Pod 被暂时性调度到某节点时执行 (WaitForFirstConsumer 卷绑定模式)。 对于通用的临时卷，建议采用后者，这样调度器就可以自由地为 Pod 选择合适的节点。 对于即时绑定，调度器则必须选出一个节点，使得在卷可用时，能立即访问该卷。

就资源所有权而言，拥有通用临时存储的 Pod 是提供临时存储 (ephemeral storage) 的 PVC 的所有者，因此：

- 当 Pod 被删除时，Kubernetes 垃圾收集器会删除 PVC
- 如果 PVC 采用了 StorageClass 动态制备，那么 PVC 删除时需要根据其回收策略来判断是否需要回收 PV
- 通常而言，StorageClass 的默认回收策略是 Delete，一般会触发清理动态 PV。同时，也建议对于通用临时存储的动态 PV，回收策略最好就选 Delete。

**注意：**

- 你可以使用带有 retain 回收策略的 StorageClass 提供通用准临时存储。因为 PVC 删除时，动态创建的 PV 无法自动删除，所以该存储比 Pod 寿命长，在这种情况下，你需要确保单独进行卷清理。
- 当这些 PVC 存在时，它们可以像其他 PVC 一样使用。特别是，它们可以被引用作为批量克隆或快照的数据源。 PVC 对象还保持着卷的当前状态。

## PVC 的命名

通用临时卷自动创建的 PVC 采取确定性的命名机制：

- 名称是 Pod 名称和卷名称的组合，中间由连字符(-)连接。

在上面的示例中，PVC 将命名为 `my-app-scratch-volume` 。这种确定性的命名机制使得与 PVC 交互变得更容易：因为一旦知道 Pod 名称和卷名，就不必搜索它。

这种命名机制也引入了潜在的冲突， 不同的 Pod 之间（名为 “Pod-a” 的 Pod 挂载名为 "scratch" 的卷， 和名为 "pod" 的 Pod 挂载名为 “a-scratch” 的卷，这两者均会生成名为 "pod-a-scratch" 的 PVC），或者在 Pod 和手工创建的 PVC 之间可能出现冲突。

**以下冲突会被检测到：**

- 如果 PVC 是为 Pod 创建的，那么它只用于临时卷。 此检测基于所有权关系。现有的 PVC 不会被覆盖或修改。 但这并不能解决冲突，因为如果没有正确的 PVC，Pod 就无法启动。

**注意：**

- 当同一个 namespace 中命名 Pod 和通用临时卷时，要小心，以防止发生此类冲突。

## 安全

启用 GenericEphemeralVolume（通用临时卷） 特性会有一些副作用：

- 用户能创建 Pod 就能间接地创建 PVC， 即使他们没有权限直接创建 PVC。

集群管理员必须意识到这一点。 如果这不符合他们的安全模型，他们应该使用**准入 Webhook 拒绝包含通用临时卷的对象**。

PVC 的正常命名空间配额仍然适用，因此即使允许用户使用这种新机制，他们也不能使用它来规避其他策略（这里指的就是即便可以通过通用临时卷来制作 PVC，但也不能超出配额）。
