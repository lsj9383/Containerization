# 卷（Volumes）

[TOC]

## 概览

Container 中的文件在磁盘上是临时存放的，这给 Container 中运行的较重要的应用程序带来一些问题：

- 第一个问题：当容器崩溃时文件丢失。kubelet 会重新启动容器，但容器会以干净的状态重启。
- 第二个问题：kubernetes 要求在同一 Pod 中运行多个容器并共享文件时出现。 

Kubernetes 卷（Volume）这一抽象概念能够解决这两个问题。

## 背景

Docker 也有卷（Volume）的概念，但对它只有少量且松散的管理。Docker 卷是磁盘上或者另外一个容器内的一个目录。Docker 提供卷驱动程序，但是其功能非常有限。

Kubernetes 支持很多类型的卷。Pod 可以同时使用任意数目的卷类型，例如：

- 临时卷类型的生命周期与 Pod 相同，当 Pod 不再存在时，Kubernetes 也会销毁临时卷。
- 持久卷可以比 Pod 的存活期长，当 Pod 不再存在是，Kubernetes 不会销毁持久卷。

**注意：**

- 对于任何类型的 Volumes，在容器重启期间数据都不会丢失。

卷的核心是一个目录，其中可能存有数据，Pod 中的容器可以访问该目录中的数据。该目录是如何形成的、支持它的介质以及它的内容，都是由所使用的**特定卷类型决定的**。

Pod 中使用卷非常简单，概括而言就两步：

1. 在 Pod 的 `.spec.volumes` 字段中设置为给 Pod 提供的卷，提供 Volume 的名称类型等。
1. 在 `.spec.containers[*].volumeMounts` 字段中声明卷在容器中的挂载位置。

容器中的进程看到的文件系统视图是由两部分组成：

- 容器镜像的初始内容
- 挂载在容器中的卷（如果定义了的话）

任何在该文件系统下的写入操作，如果被允许的话，都会影响接下来容器中进程访问文件系统时所看到的内容。

卷挂载在镜像中的指定路径下。 Pod 配置中的每个容器必须独立指定各个卷的挂载位置。

Volume 不能挂载到其他 Volume 之上（即某个挂载了卷的子目录，不能挂载另外一个卷。但其实存在一种使用 subPath 的相关机制），也不能与其他卷有硬链接。

## 卷类型

Kubernetes 支持下列类型的卷，有些已经弃用：

卷类型 | 是否可用 | 数据共享范围 | 描述 | 使用前提
-|-|-|-|-
cephfs | √ | Pod 共享 | [cephfs](https://docs.ceph.com/en/latest/cephfs/) 卷允许你将现存的 CephFS 卷挂载到 Pod 中。<br>cephfs 卷的内容在 Pod 被删除时会被保留，只是卷被卸载了。 | 你的 Ceph 服务器必须已经运行并将要使用的 share 导出（exported）
configMap | √ | Pod 共享 | 提供了向 Pod 注入**配置数据**的方法。 | 需要先配置 ConfigMap 对象才能使用。
downwardAPI | √ | Pod 共享 | 通过文本的形式向 Pod 中的容器提供 downwardAPI 信息。 | -
emptyDir | √ | Pod 内容器共享 | 每个 Pod 都会创建一个 Pod 内容器临时共享卷，即 emptyDir。当 Pod 删除时，emptyDir 会释放（容器重启不会释放 emptyDir）。 | -
fc (光纤通道) | √ | Pod 共享 | 允许将现有的**光纤通道块存储卷**挂载到 Pod 中。 | 必须先配置 FC SAN Zoning，Kubernetes 主机才可以使用 fc。
hostPath | √ | Node 上的 Pod 共享 | 将宿主机节点上的文件系统挂载到 Pod 容器中。存在安全风险，不推荐使用。若要使用，推荐只读方式。 | -
iscsi | √ | Pod 共享 | 将 iSCSI (基于 IP 的 SCSI) 卷挂载到你的 Pod 中。<br> iscsi 卷的内容在删除 Pod 时会被保留，卷只是被卸载 | 必须先拥有自己的 iSCSI 服务器，并在上面创建卷。
local | √ | | |
nfs | √ | Pod 共享 | 能将 NFS (网络文件系统) 挂载到你的 Pod 中。<br>nfs 卷的内容在删除 Pod 时会被保存，卷只是被卸载。| 必须运行自己的 NFS 服务器并将目标 share 导出备用。
persistentVolumeClaim | √ | Pod 共享 | 用来将持久卷（PersistentVolume）挂载到 Pod 中。避免开发者关注各类存储系统细节。 | 待补充
projected（投射）| √ | - | 投射卷能将若干现有的卷来源映射到同一目录上。| -
rbd | √ | Pod 共享 | 将 Rados 块设备卷挂载到你的 Pod 中。<br>rbd 卷的内容在删除 Pod 时会被保存，卷只是被卸载。| 必须安装运行 Ceph。
secret | √ | Pod 共享 | 用来给 Pod 传递敏**感信息**，例如密码。 | 需要先创建 Secret 对象。
awsElasticBlockStore  | × | | |
azureDisk | × | | |
azureFile | × | | |
cinder | × | |
gcePersistentDisk | × | | |
gitRepo | × | | |
glusterfs | × | | |
portworxVolume | × | |
vsphereVolume | × | |

### cephfs

cephfs volume 允许你将现存的 CephFS 卷挂载到 Pod 中。

不像 emptyDir 那样会在 Pod 被删除的同时也会被删除，cephfs 卷的内容在 Pod 被删除时会被保留，只是卷被卸载了。

这意味着 cephfs 卷可以被预先填充数据，且这些数据可以在 **Pod 之间共享**。同一 cephfs 卷可同时被多个写者挂载。

**注意：**

- 在使用 Ceph 卷之前，你的 Ceph 服务器必须已经运行并将要使用的 share 导出（exported）。

更多信息请参考 [CephFS 示例](https://github.com/kubernetes/examples/tree/master/volumes/cephfs/)。

### configMap

configMap volume 提供了向 Pod 注入配置数据的方法。

在 Kubernetes 中有一种 ConfigMap 对象，这类对象专门存储配置数据。ConfigMap 对象中存储的数据可以被 configMap 类型的 volume 引用，然后被 Pod 中运行的容器化应用使用，使用起来就像是本地目录中的文件。

引用 configMap 对象时，你可以在 volume 中通过它的名称来引用。你可以自定义 ConfigMap 中特定条目所要使用的路径。

下面的配置显示了如何将名为 log-config 的 ConfigMap 挂载到名为 configmap-pod 的 Pod 中：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: configmap-pod
spec:
  containers:
    - name: test
      image: busybox:1.28
      volumeMounts:
        - name: config-vol
          mountPath: /etc/config
  volumes:
    - name: config-vol
      configMap:
        name: log-config
        items:
          - key: log_level
            path: log_level
```

名为 `log-config` 的 ConfigMap 对象，会以卷的形式挂载，并且该 ConfigMap 对象存储在 log_level 条目中的所有内容都被挂载到 Pod 的 `/etc/config/log_level` 路径下。 

**注意：**

- 这个路径来源于卷的 mountPath 和 log_level 键对应的 path。

### downwardAPI

downwardAPI 卷用于为应用提供 downward API 数据。在这类卷中，所公开的数据以纯文本格式的只读文件形式存在。

**注意：**

- 容器以 subPath 卷挂载方式使用 downward API 时，在字段值更改时将**不能**接收到它的更新。
- 那么非 subPath 形式则可以。

这是一个 downwardAPI 的文件形式注入示例：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kubernetes-downwardapi-volume-example
  labels:
    zone: us-est-coast
    cluster: test-cluster1
    rack: rack-22
  annotations:
    build: two
    builder: john-doe
spec:
  containers:
    - name: client-container
      image: k8s.gcr.io/busybox
      command: ["sh", "-c"]
      args:
      - while true; do
          if [[ -e /etc/podinfo/labels ]]; then
            echo -en '\n\n'; cat /etc/podinfo/labels; fi;
          if [[ -e /etc/podinfo/annotations ]]; then
            echo -en '\n\n'; cat /etc/podinfo/annotations; fi;
          sleep 5;
        done;
      volumeMounts:
        - name: podinfo
          mountPath: /etc/podinfo
  volumes:
    - name: podinfo
      downwardAPI:
        items:
          - path: "labels"
            fieldRef:
              fieldPath: metadata.labels
          - path: "annotations"
            fieldRef:
              fieldPath: metadata.annotations
```

### emptyDir

当 Pod 分派到某个 Node 上时，emptyDir 卷会被创建，并且在 Pod 在该节点上运行期间，卷一直存在。

emptyDir volume 就像其名称表示的那样，卷最初是空的。尽管 Pod 中的容器挂载 emptyDir 卷的路径可能相同也可能不同，这些容器都可以读写 emptyDir 卷中相同的文件（所用容器共享 emptyDir 中的文件）。 

**注意：**

- 当 Pod 因为某些原因被从节点上删除时，emptyDir 卷中的数据也会被永久删除。
- 容器崩溃并不会导致 Pod 被从节点上移除，因此容器崩溃期间 emptyDir 卷中的数据是安全的。

emptyDir 的一些用途：

- 缓存空间，例如基于磁盘的归并排序。
- 为耗时较长的计算任务提供检查点，以便任务能方便地从崩溃前状态恢复执行。
- 在 Web 服务器容器服务数据时，保存内容管理器容器获取的文件。


emptyDir 卷存储在该节点所使用的介质取决于环境，这里的介质可以是：磁盘、SSD、网络存储甚至内存。

你可以将 `emptyDir.medium` 字段设置为 "Memory"，以告诉 Kubernetes 为你挂载 **tmpfs（基于 RAM 的文件系统）**。虽然 tmpfs 速度非常快，但是要注意它与磁盘不同。tmpfs 在节点重启时会被清除，并且你所写入的所有文件都会计入容器的内存消耗，受容器内存限制约束。

**注意：**

- 当启用 SizeMemoryBackedVolumes 特性门控时，你可以为基于内存提供的卷指定大小。
- 如果未指定大小，则基于内存的卷的大小为 Linux 主机上内存的 50%。

emptyDir 配置示例：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pd
spec:
  containers:
  - image: registry.k8s.io/test-webserver
    name: test-container
    volumeMounts:
    - mountPath: /cache
      name: cache-volume
  volumes:
  - name: cache-volume
    emptyDir: {}
```

### fc (光纤通道)

fc 卷类型允许将现有的**光纤通道块存储卷**挂载到 Pod 中。

可以使用卷配置中的参数 targetWWNs 来指定单个或多个目标 WWN（World Wide Names）。如果指定了多个 WWN，targetWWNs 期望这些 WWN 来自多路径连接。

**注意：**

- 你必须配置 FC SAN Zoning，以便预先向目标 WWN 分配和屏蔽这些 LUN（卷），这样 Kubernetes 主机才可访问它们。

更多详情请参考 FC 示例。

### hostPath

**警告：**

- HostPath 卷存在许多安全风险，最佳做法是尽可能避免使用 HostPath。当必须使用 HostPath 卷时，它的范围**应仅限于所需的文件或目录**，并以**只读方式挂载**。
- 如果通过 AdmissionPolicy 限制 HostPath 对特定目录的访问，则必须要求 volumeMounts 使用 readOnly 挂载以使策略生效。

hostPath 卷能将**主机节点**文件系统上的文件或目录挂载到你的 Pod 中。虽然这不是大多数 Pod 需要的，但是它为一些应用程序提供了强大的逃生舱。

例如，hostPath 的一些用法有：

- 运行一个需要访问 Docker 内部机制的容器；可使用 hostPath 挂载 /var/lib/docker 路径。
- 在容器中运行 cAdvisor 时，以 hostPath 方式挂载 /sys。
- 允许 Pod 指定给定的 hostPath 在运行 Pod 之前是否应该存在，是否应该创建以及应该以什么方式存在。


除了必需的 path 属性之外，你可以选择性地为 hostPath 卷指定 type。支持的 type 值如下：

取值 | 行为
-|-
空字符串（默认 | 用于向后兼容，这意味着在安装 hostPath 卷之前不会执行任何检查。
DirectoryOrCreate | 如果给定路径不存在，那么将根据需要创建空目录，权限设置为 0755，具有与 kubelet 相同的组和属主信息。
Directory | 在给定路径上必须存在目录。
FileOrCreate | 如果在给定路径上什么都不存在，那么将在那里根据需要创建空文件，权限设置为 0644，具有与 kubelet 相同的组和所有权。
File | 在给定路径上必须存在的文件。
Socket | 在给定路径上必须存在的 UNIX 套接字。
CharDevice | 在给定路径上必须存在的字符设备。
BlockDevice | 在给定路径上必须存在的块设备。

hostPath 配置示例：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pd
spec:
  containers:
  - image: registry.k8s.io/test-webserver
    name: test-container
    volumeMounts:
    - mountPath: /test-pd
      name: test-volume
  volumes:
  - name: test-volume
    hostPath:
      # 宿主上目录位置
      path: /data
      # 此字段为可选
      type: Directory
```

当使用这种类型的卷时要小心，因为：

- HostPath 卷可能会暴露特权系统凭据（例如 Kubelet）或特权 API（例如容器运行时套接字），可用于容器逃逸或攻击集群的其他部分。
- 具有相同配置（例如基于同一 PodTemplate 创建）的多个 Pod 会由于节点上文件的不同而在不同节点上有不同的行为。
- 在底层主机上创建的文件或目录只能由 root 写入。您要么需要在特权容器中以 root 身份运行进程，要么修改主机上的文件权限以便能够写入 hostPath 卷。

### iscsi

iscsi 卷能将 iSCSI (基于 IP 的 SCSI) 卷挂载到你的 Pod 中。

不像 emptyDir 那样会在删除 Pod 的同时也会被删除，iscsi 卷的内容在删除 Pod 时会被保留，卷只是被卸载。

这意味着 iscsi 卷可以被预先填充数据，并且这些数据可以在 Pod 之间共享。

**注意：**

- 在使用 iSCSI 卷之前，你必须拥有自己的 iSCSI 服务器，并在上面创建卷。

iSCSI 的一个特点是它可以同时被多个用户以只读方式挂载。 这意味着你可以用数据集预先填充卷，然后根据需要在尽可能多的 Pod 上使用它。 不幸的是，iSCSI 卷只能由单个使用者以读写模式挂载。不允许同时写入。

更多详情请参考 [iSCSI 示例](https://github.com/kubernetes/examples/tree/master/volumes/iscsi)。

### local

### nfs

nfs 卷能将 NFS (网络文件系统) 挂载到你的 Pod 中。 不像 emptyDir 那样会在删除 Pod 的同时也会被删除，nfs 卷的内容在删除 Pod 时会被保存，卷只是被卸载。 这意味着 nfs 卷可以被预先填充数据，并且这些数据可以在 Pod 之间共享。

**注意：**

- 在使用 NFS 卷之前，你必须运行自己的 NFS 服务器并将目标 share 导出备用。

要了解更多详情请参考 [NFS 示例](https://github.com/kubernetes/examples/tree/master/staging/volumes/nfs)。

### persistentVolumeClaim

persistentVolumeClaim 卷用来将持久卷（PersistentVolume）挂载到 Pod 中。

持久卷申领（PersistentVolumeClaim）是用户在不知道特定云环境细节的情况下“申领”持久存储（例如 GCE PersistentDisk 或者 iSCSI 卷）的一种方法。

**注意：**

- 如前所述，可以看到由非常多不同类型的持久性存储，例如 nfs、iSCSI 卷等等，它们配置和接口都大不相同，无法统一，需要开发者关注。
- 为了让用户不用去关注持久存储卷细节，而是使用统一接口的 persistentVolumeClaim 卷，减轻开发者的复杂度。

更多详情请参考[持久卷](https://kubernetes.io/zh-cn/docs/concepts/storage/persistent-volumes/)。

### projected （投射）

投射卷能将若干现有的卷来源映射到同一目录上。

更多详情请参考[投射卷](https://kubernetes.io/zh-cn/docs/concepts/storage/projected-volumes/)。

### rbd

rbd 卷允许将 Rados 块设备卷挂载到你的 Pod 中。 不像 emptyDir 那样会在删除 Pod 的同时也会被删除，rbd 卷的内容在删除 Pod 时会被保存，卷只是被卸载。 这意味着 rbd 卷可以被预先填充数据，并且这些数据可以在 Pod 之间共享。

说明： 在使用 RBD 之前，你必须安装运行 Ceph。
RBD 的一个特性是它可以同时被多个用户以只读方式挂载。 这意味着你可以用数据集预先填充卷，然后根据需要在尽可能多的 Pod 中并行地使用卷。 不幸的是，RBD 卷只能由单个使用者以读写模式安装。不允许同时写入。

更多详情请参考 RBD 示例。

### secret

secret 卷用来给 Pod 传递敏**感信息**，例如密码。

你可以将 Secret 存储在 Kubernetes API 服务器上，然后以文件的形式挂载到 Pod 中，无需直接与 Kubernetes 耦合。

secret 卷的戒指由 **tmpfs（基于 RAM 的文件系统）** 提供存储，因此它们永远不会被写入非易失性（持久化的）存储器。

**说明：**

- 使用前你必须在 Kubernetes API 中创建 Secret。
- 容器以 subPath 卷挂载方式挂载 Secret 时，将感知不到 Secret 的更新。

## 使用 subPath

有时，在单个 Pod 中共享卷以供多方使用是很有用的。 `volumeMounts.subPath` 属性可用于指定所引用的卷内的子路径，而不是其根路径。

下面例子展示了如何配置某包含 LAMP 堆栈（Linux Apache MySQL PHP）的 Pod 使用同一共享卷：

- PHP 应用的代码和相关数据映射到卷的 html 文件夹
- MySQL 数据库存储在卷的 mysql 文件夹中：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-lamp-site
spec:
    containers:
    - name: mysql
      image: mysql
      env:
      - name: MYSQL_ROOT_PASSWORD
        value: "rootpasswd"
      volumeMounts:
      - mountPath: /var/lib/mysql
        name: site-data
        subPath: mysql
    - name: php
      image: php:7.0-apache
      volumeMounts:
      - mountPath: /var/www/html
        name: site-data
        subPath: html
    volumes:
    - name: site-data
      persistentVolumeClaim:
        claimName: my-lamp-site-data
```

**注意：**

- 此示例中的 subPath 配置不建议在生产环境中使用。
- 更合理的方式我认为应该是根据其用途区分为不同的存储卷。

#### 带有环境变量的 subPath

使用 subPathExpr 字段可以基于 downward API 环境变量来构造 subPath 目录名。**subPath 和 subPathExpr 属性是互斥的**。

在这个示例中：

- Pod 使用 subPathExpr 来 hostPath 卷 `/var/log/pods` 中创建目录 pod1
- hostPath 卷采用来自 downwardAPI 的 Pod 名称生成目录名
- 宿主目录 `/var/log/pods/pod1` 被挂载到容器的 /logs 中。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod1
spec:
  containers:
  - name: container1
    env:
    - name: POD_NAME
      valueFrom:
        fieldRef:
          apiVersion: v1
          fieldPath: metadata.name
    image: busybox:1.28
    command: [ "sh", "-c", "while [ true ]; do echo 'Hello'; sleep 10; done | tee -a /logs/hello.txt" ]
    volumeMounts:
    - name: workdir1
      mountPath: /logs
      # 包裹变量名的是小括号，而不是大括号
      subPathExpr: $(POD_NAME)
  restartPolicy: Never
  volumes:
  - name: workdir1
    hostPath:
      path: /var/log/pods
```

## 资源

emptyDir 卷的存储介质（例如磁盘、SSD 等）是由保存 kubelet 数据的根目录（通常是 /var/lib/kubelet）的文件系统的介质确定。

Kubernetes 对 emptyDir 卷或者 hostPath 卷可以消耗的空间没有限制，容器之间或 Pod 之间也没有隔离。

要了解如何使用资源规约来请求空间，可参考[如何管理资源](https://kubernetes.io/zh-cn/docs/concepts/configuration/manage-resources-containers/)。

## 树外（Out-of-Tree）卷插件

Out-of-Tree 卷插件包括：

- 容器存储接口（CSI）
- FlexVolume（已弃用）

它们使存储供应商能够创建自定义存储插件，而无需将插件源码添加到 Kubernetes 代码仓库。

以前，所有卷插件（如上面列出的卷类型）都是“树内（In-Tree）”的：

- “树内” 插件是与 Kubernetes 的核心组件一同构建、链接、编译和交付的。 这意味着向 Kubernetes 添加新的存储系统（卷插件）需要将代码合并到 Kubernetes 核心代码库中。
- “树外” 插件，即开发商根据接口来开发自定义存储插件，变更 kubernetes 核心组件。

CSI 和 FlexVolume 都允许独立于 Kubernetes 代码库开发卷插件，并作为扩展部署（安装）在 Kubernetes 集群上。

对于希望创建树外（Out-Of-Tree）卷插件的存储供应商，请参考[卷插件常见问题](https://github.com/kubernetes/community/blob/master/sig-storage/volume-plugin-faq.md)。

### csi

[容器存储接口 (CSI)](https://github.com/container-storage-interface/spec/blob/master/spec.md) 为容器编排系统（如 Kubernetes）定义标准接口，以将任意存储系统暴露给它们的容器工作负载。

更多详情请阅读 [CSI 设计方案](https://github.com/kubernetes/design-proposals-archive/blob/main/storage/container-storage-interface.md)。

**注意：**

- Kubernetes v1.13 废弃了对 CSI 规范版本 0.2 和 0.3 的支持，并将在以后的版本中删除。
- CSI 驱动可能并非兼容所有的 Kubernetes 版本。 请查看特定 CSI 驱动的文档，以了解各个 Kubernetes 版本所支持的部署步骤以及兼容性列表。

一旦在 Kubernetes 集群上部署了 CSI 兼容卷驱动程序，用户就可以使用 csi 卷类型来挂接、挂载 CSI 驱动所提供的卷。

csi 卷可以在 Pod 中以三种方式使用：

- 通过 PersistentVolumeClaim(#persistentvolumeclaim) 对象引用
- 使用一般性的临时卷
- 使用 CSI 临时卷， 前提是驱动支持这种用法

## 挂载卷的传播

挂载卷的传播能力允许将容器安装的卷共享到同一 Pod 中的其他容器，甚至共享到同一节点上的其他 Pod。

换句话说，其实就是：

- 主机在卷中又挂在了新的卷，能否传播给 Pod 中的容器。
- Pod 中的容器的卷中如果人为进行新的挂载，能否将新挂载传播给主机和 Pod 的其他容器。

如果都不支持，即传播类型为 None；如果支持前者，即传播类型为 HostToContainer；如果两者都支持，即传播类型为 Bidirectional。

卷的挂载传播特性由 `Container.volumeMounts[i].mountPropagation` 字段控制。 它的值包括：

mountPropagation | 描述
-|-
None | 此卷挂载将导致 Pod 中的容器不会感知到主机后续在此卷或其任何子目录上执行的挂载变化。类似的，容器所创建的卷挂载在主机上是不可见的。**这是默认模式**。<br>该模式等同于 Linux 内核文档中描述的 private 挂载传播选项。
HostToContainer | 此卷挂载将导致 Pod 中容器会感知到主机后续针对此卷或其任何子目录的挂载操作。<br>换句话说，如果主机在此挂载卷中挂载任何内容，容器将能看到它被挂载在那里。<br>该模式等同于 Linux 内核文档中描述的 rslave 挂载传播选项。
Bidirectional | 这种卷挂载和 HostToContainer 挂载表现相同。另外，容器创建的卷挂载将被传播回至主机和使用同一卷的所有 Pod 的所有容器。<br>该模式等同于 Linux 内核文档中描述的 rshared 挂载传播选项。

**注意：**

- 配置了 Bidirectional 挂载传播选项的 Pod 如果在同一卷上挂载了内容，挂载传播设置为 HostToContainer 的容器都将能看到这一变化。
- Bidirectional 形式的挂载传播可能比较危险。 它可以破坏主机操作系统，因此它只被允许在特权容器中使用。 强烈建议你熟悉 Linux 内核行为。 此外，由 Pod 中的容器创建的任何卷挂载必须在终止时由容器销毁（卸载）。

### 配置

在某些部署环境中，挂载传播正常工作前，必须在 Docker 中正确配置挂载共享（mount share），如下所示。

编辑你的 Docker systemd 服务文件，按下面的方法设置 MountFlags：

```sh
MountFlags=shared
```

或者，如果存在 MountFlags=slave 就删除掉。然后重启 Docker 守护进程：

```sh
sudo systemctl daemon-reload
sudo systemctl restart docker
```

这里有一篇对 [kubernetes 的挂载传播](https://jicki.cn/kubernetes-mount-propagation/) 的测试。
