# 动态卷制备

动态卷制备允许按需创建 PV。 如果没有动态制备，集群管理员必须手动地联系他们的云或存储提供商来创建新的存储卷，然后在 Kubernetes 集群创建 PersistentVolume 对象来表示这些卷。

动态制备功能消除了集群管理员预先配置存储的需要。相反，它在用户请求时自动制备存储。

## 背景

动态卷制备的实现基于 storage.k8s.io API 组中的 StorageClass API 对象。

集群管理员可以根据需要定义多个 StorageClass 对象，每个对象指定一个卷插件（又名 provisioner），卷插件向卷制备商提供在创建卷时需要的数据卷信息及相关参数。

集群管理员可以在集群中定义和公开多种存储（来自相同或不同的存储系统），每种都具有自定义参数集。该设计也确保终端用户不必担心存储制备的复杂性和细微差别，但仍然能够从多个存储选项中进行选择。

## 启用动态卷制备

要启用动态制备功能，集群管理员需要为用户预先创建一个或多个 StorageClass 对象。

StorageClass 对象定义当动态制备被调用时，哪一个驱动将被使用和哪些参数将被传递给驱动。StorageClass 对象的名字必须是一个合法的 DNS 子域名。

以下清单创建了一个 StorageClass 存储类 "slow"，它提供类似标准磁盘的永久磁盘：

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: slow
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-standard
```

以下清单创建了一个 "fast" 存储类，它提供类似 SSD 的永久磁盘。

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-ssd
```

**注意：**

- 这里是通过 parameters 传递给 provisioner 信息，让 provisioner 来创建合适的 PV。

## 使用动态卷制备

用户通过在 PersistentVolumeClaim 中指定 StorageClass 来请求相应 storageclass 的动态制备。

用户现在能够而且应该使用 PersistentVolumeClaim 对象的 storageClassName 字段。 这个字段的值必须能够匹配到集群管理员配置的 StorageClass 名称。

例如，要选择 “fast” 存储类，用户将创建如下的 PersistentVolumeClaim：

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: claim1
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: fast
  resources:
    requests:
      storage: 30Gi
```

该声明会自动制备一块类似 SSD 的永久磁盘。 在删除该 PVC 后，这个 PV 也会被销毁（因为默认动态制备的 PV 其回收策略是 Delete）。

## 设置默认值的行为

可以在集群上启用默认的动态卷 PV 备，以便在未指定存储类的情况下动态设置所有声明。

集群管理员可以通过以下方式启用默认行为：

- 标记一个 StorageClass 为 默认（通过 `storageclass.kubernetes.io/is-default-class` 注解）
- 确保 DefaultStorageClass 准入控制器在 API 服务端被启用。

具体而言：

- 管理员可以通过向其添加 `storageclass.kubernetes.io/is-default-class` annotation 来将特定的 StorageClass 标记为默认。
- 当集群中存在默认的 StorageClass 并且用户创建了一个未指定 storageClassName 的 PersistentVolumeClaim 时，DefaultStorageClass 准入控制器会自动向其中添加指向默认存储类的 storageClassName 字段。

**注意：**

- 集群上最多只能有一个 **默认** 存储类，否则无法创建没有明确指定 storageClassName 的 PVC。

## 拓扑感知

在多可用区集群中，Pod 可以被分散到某个区域的多个可用区。 单可用区存储后端应该被制备到 Pod 被调度到的可用区。 这可以通过设置卷绑定模式来实现。
