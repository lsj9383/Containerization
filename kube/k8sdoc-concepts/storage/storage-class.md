# 存储类（StorageClass）

[TOC]

## 概览

StorageClass 为管理员提供了描述存储 "类" 的方法。

不同的 Class 可能会映射到不同的服务质量等级或备份策略，或是由集群管理员制定的任意策略。

Kubernetes 本身并不清楚各种类代表的什么，这个类的概念在其他存储系统中有时被称为 "配置文件"。

## StorageClass 资源

每个 StorageClass 都包含 provisioner、parameters 和 reclaimPolicy 字段，这些字段会在 StorageClass 需要动态分配 PV 时会使用到。

StorageClass 对象的命名很重要，用户使用这个命名来请求生成一个特定的类。当创建 StorageClass 对象时，管理员设置 StorageClass 对象的命名和其他参数，同时**一旦创建了对象就不能再对其更新**。

管理员也可以为没有申请绑定到特定 StorageClass 的 PVC 指定一个默认的存储类。

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp2
reclaimPolicy: Retain
allowVolumeExpansion: true
mountOptions:
  - debug
volumeBindingMode: Immediate
```

这是我本机的一个 storage class 资源列表：

```yaml
$ kubectl get sc
NAME            PROVISIONER                 RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
cbs (default)   com.tencent.cloud.csi.cbs   Delete          Immediate           false                  9d
cfs             com.tencent.cloud.csi.cfs   Delete          Immediate           false                  83m

$ kubectl describe sc cbs
Name:                  cbs
IsDefaultClass:        Yes
Annotations:           storageclass.beta.kubernetes.io/is-default-class=true
Provisioner:           com.tencent.cloud.csi.cbs
Parameters:            type=cbs
AllowVolumeExpansion:  <unset>
MountOptions:          <none>
ReclaimPolicy:         Delete
VolumeBindingMode:     Immediate
Events:                <none>
```

### 存储制备器

每个 StorageClass 都有一个制备器（Provisioner），用来决定使用哪个卷插件（Volume Plugin）来制备 PV。该字段必须指定。

卷插件 | 内置制备器 | 配置例子
-|-|-
AWSElasticBlockStore | ✓ | AWS EBS
AzureFile | ✓ | Azure File
AzureDisk | ✓ | Azure Disk
CephFS | - | -
Cinder | ✓ | OpenStack Cinder
FC | - | -
FlexVolume | - | -
GCEPersistentDisk | ✓ | GCE PD
Glusterfs | ✓ | Glusterfs
iSCSI | - | -
NFS | - | NFS
RBD | ✓ | Ceph RBD
VsphereVolume | ✓ | vSphere
PortworxVolume | ✓ | Portworx Volume
Local | - | Local

腾讯云这里用的是自己创建的卷插件：`com.tencent.cloud.csi.cfs` 和 `com.tencent.cloud.csi.cbs`

Volume Plugin 是用来创建 PV 的，也因此为什么腾讯云在创建这些 PV 之前需要先安装插件，还要创建一个 storageclass。

### 回收策略（Reclaim Policy）

由 StorageClass 动态创建的 PersistentVolume 会在类的 reclaimPolicy 字段中指定回收策：可以是 Delete 或者 Retain。

如果 StorageClass 对象被创建时没有指定 reclaimPolicy，它将**默认为 Delete**。

通过 StorageClass 手动创建并管理的 PersistentVolume 会使用它们被创建时指定的回收策略。

### 允许卷扩展（Allow Volume Expansion）

PersistentVolume 可以配置为可扩展。将此功能设置为 true 时，允许用户通过编辑相应的 PVC 对象来调整卷大小。

当下层 StorageClass 的 allowVolumeExpansion 字段设置为 true 时，以下类型的卷支持卷扩展。

卷类型 | Kubernetes 版本要求
-|-
gcePersistentDisk | 1.11
awsElasticBlockStore | 1.11
Cinder | 1.11
glusterfs | 1.11
rbd | 1.11
Azure File | 1.11
Azure Disk | 1.11
Portworx | 1.11
FlexVolume | 1.13
CSI | 1.14 (alpha), 1.16 (beta)

**注意：**

- 此功能仅可用于扩容卷，不能用于缩小卷。

### 挂载选项（Mount Options）

由 StorageClass 动态创建的 PersistentVolume 将使用类中 mountOptions 字段指定的挂载选项。

如果卷插件不支持挂载选项，却指定了挂载选项，则制备操作会失败。挂载选项在 StorageClass 和 PV 上都不会做验证，如果其中一个挂载选项无效，那么这个 PV 挂载操作就会失败。

### 卷绑定模式（Volume Binding Mode）

volumeBindingMode 字段控制了**卷绑定和动态制备应该发生在什么时候**。有两种取值：

- Immediate，默认值。表示一旦创建了 PVC 也就完成了卷绑定和动态制备。
- WaitForFirstConsumer。表示创建 PVC 后并不会理解创建 PV，而是等待使用该 PVC 的 Pod 被创建后才会创建 PV。

为什么要等待抵押给 Pod 创建呢？因为对于由于拓扑限制而非集群所有节点可达的存储后端，PersistentVolume 会在不知道 Pod 调度要求的情况下绑定或者制备。集群管理员可以通过指定 WaitForFirstConsumer 模式来解决此问题。此时 PV 会根据 Pod 调度约束指定的拓扑来选择或制备。这些包括但不限于：

- 资源需求
- 节点筛选器
- Pod 亲和性和互斥性
- 污点和容忍度

以下插件支持动态制备的 WaitForFirstConsumer 模式:

- AWSElasticBlockStore
- GCEPersistentDisk
- AzureDisk

**注意：**

- 如果你选择使用 WaitForFirstConsumer，请不要在 Pod 规约中使用 nodeName 来指定节点亲和性。如果在这种情况下使用 nodeName，Pod 将会绕过调度程序，PVC 将停留在 pending 状态。
- 相反，在这种情况下，你可以使用节点选择器作为主机名，如下所示：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: task-pv-pod
spec:
  nodeSelector:
    kubernetes.io/hostname: kube-01
  volumes:
    - name: task-pv-storage
      persistentVolumeClaim:
        claimName: task-pv-claim
  containers:
    - name: task-pv-container
      image: nginx
      ports:
        - containerPort: 80
          name: "http-server"
      volumeMounts:
        - mountPath: "/usr/share/nginx/html"
          name: task-pv-storage
```

### 允许的拓扑结构

当集群操作人员使用了 WaitForFirstConsumer 的卷绑定模式， 在大部分情况下就没有必要将制备限制为特定的拓扑结构。然而，如果还有需要的话，可以使用 allowedTopologies。

这个例子描述了如何将制备卷的拓扑限制在特定的区域， 在使用时应该根据插件支持情况替换 zone 和 zones 参数。

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-standard
volumeBindingMode: WaitForFirstConsumer
allowedTopologies:
- matchLabelExpressions:
  - key: failure-domain.beta.kubernetes.io/zone
    values:
    - us-central-1a
    - us-central-1b
```

## 参数（Parameters）

Storage Classes 的 Parameters 描述了存储类的卷。

Parameters 的结构取决于制备器，可以接受不同的参数。 例如，参数 type 的值 io1 和参数 iopsPerGB 特定于 EBS PV。当参数被省略时，会使用默认值。

一个 StorageClass 最多可以定义 512 个参数。这些参数对象的总长度不能超过 256 KiB, 包括参数的键和值。

本文省略各个制备器的具体参数，感兴趣可以参考 [参数](https://kubernetes.io/zh-cn/docs/concepts/storage/storage-classes/#parameters)。

## 使用腾讯云 cfs StorageClass 的一个示例

前提：

- 启动了腾讯云 cfs
- tke 上安装了 cfs 插件（提供制备器）
- tke 上创建 cfs 的 storageclass

```sh
# 查看集群中的 cfs storageclass
$ kubectl get sc
NAME            PROVISIONER                 RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
cbs (default)   com.tencent.cloud.csi.cbs   Delete          Immediate           false                  9d
cfs             com.tencent.cloud.csi.cfs   Delete          Immediate           false                  116m

$ kubectl describe sc cfs
Name:                  cfs
IsDefaultClass:        No
Annotations:           <none>
Provisioner:           com.tencent.cloud.csi.cfs
Parameters:            pgroupid=pgroupbasic,storagetype=SD,subnetid=subnet-n6wa7k6q,vers=3,vpcid=vpc-364sd70l,zone=ap-guangzhou-3
AllowVolumeExpansion:  <unset>
MountOptions:          <none>
ReclaimPolicy:         Delete
VolumeBindingMode:     Immediate
Events:                <none>

# 创建一个使用该 StorageClass 的 PVC，因为该 StorageClass 存在，并且具备相应的制备器，所以可以自动生成 PV
$ cat <<EOF > ./dynamic-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dynamic-cfs-pv-claim
spec:
  storageClassName: cfs
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 3Gi
EOF

$ kubectl apply -f dynamic-pvc.yaml


# 查看 PVC 是否绑定动态创建的 PV，可见已经绑定了一个 pvc-fa6ee85b-303b-4a07-9dbe-e8e0b7344d39
$ kubectl get pvc
NAME                   STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
cfs-pv-claim           Bound    cfs-test-pv                                10Gi       RWX            cfs            124m
dynamic-cfs-pv-claim   Bound    pvc-fa6ee85b-303b-4a07-9dbe-e8e0b7344d39   3Gi        RWX            cfs            7m15s

# 查看创建细节，从 Event 看，第一次创建竟然还失败了
$ kubectl describe pvc dynamic-cfs-pv-claim
kubectl describe pvc dynamic-cfs-pv-claim
Name:          dynamic-cfs-pv-claim
Namespace:     default
StorageClass:  cfs
Status:        Bound
Volume:        pvc-fa6ee85b-303b-4a07-9dbe-e8e0b7344d39
Labels:        <none>
Annotations:   pv.kubernetes.io/bind-completed: yes
               pv.kubernetes.io/bound-by-controller: yes
               volume.beta.kubernetes.io/storage-provisioner: com.tencent.cloud.csi.cfs
Finalizers:    [kubernetes.io/pvc-protection]
Capacity:      3Gi
Access Modes:  RWX
VolumeMode:    Filesystem
Used By:       <none>
Events:
  Type     Reason                 Age                    From                                                                                        Message
  ----     ------                 ----                   ----                                                                                        -------
  Normal   ExternalProvisioning   8m22s (x3 over 8m28s)  persistentvolume-controller                                                                 waiting for a volume to be created, either by external provisioner "com.tencent.cloud.csi.cfs" or manually created by system administrator
  Warning  ProvisioningFailed     8m18s                  com.tencent.cloud.csi.cfs_csi-provisioner-cfsplugin-0_784ff493-164c-4a30-b31e-efd1d615bc4a  failed to provision volume with StorageClass "cfs": rpc error: code = DeadlineExceeded desc = context deadline exceeded
  Normal   Provisioning           8m17s (x2 over 8m28s)  com.tencent.cloud.csi.cfs_csi-provisioner-cfsplugin-0_784ff493-164c-4a30-b31e-efd1d615bc4a  External provisioner is provisioning volume for claim "default/dynamic-cfs-pv-claim"
  Normal   ProvisioningSucceeded  8m12s                  com.tencent.cloud.csi.cfs_csi-provisioner-cfsplugin-0_784ff493-164c-4a30-b31e-efd1d615bc4a  Successfully provisioned volume pvc-fa6ee85b-303b-4a07-9dbe-e8e0b7344d39

# 查看动态创建的 PV（cfs-test-pv 是我静态创建的 cfs PV）
$ kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                          STORAGECLASS   REASON   AGE
cfs-test-pv                                10Gi       RWX            Retain           Bound    default/cfs-pv-claim           cfs                     133m
pvc-fa6ee85b-303b-4a07-9dbe-e8e0b7344d39   3Gi        RWX            Delete           Bound    default/dynamic-cfs-pv-claim   cfs                     9m23s
# 可见动态创建的 PV，其 CAPACITY 和 PVC 申领是相同大小的

# 查看动态 PV 细节
$ describe pv pvc-fa6ee85b-303b-4a07-9dbe-e8e0b7344d39
Name:            pvc-fa6ee85b-303b-4a07-9dbe-e8e0b7344d39
Labels:          <none>
Annotations:     pv.kubernetes.io/provisioned-by: com.tencent.cloud.csi.cfs
Finalizers:      [kubernetes.io/pv-protection]
StorageClass:    cfs
Status:          Bound
Claim:           default/dynamic-cfs-pv-claim
Reclaim Policy:  Delete
Access Modes:    RWX
VolumeMode:      Filesystem
Capacity:        3Gi
Node Affinity:   <none>
Message:         
Source:
    Type:              CSI (a Container Storage Interface (CSI) volume source)
    Driver:            com.tencent.cloud.csi.cfs
    FSType:            
    VolumeHandle:      cfs-a8ypofxz
    ReadOnly:          false
    VolumeAttributes:      fsid=hbx6fn7h
                           host=10.0.1.14
                           pgroupid=pgroupbasic
                           storage.kubernetes.io/csiProvisionerIdentity=1663485418567-8081-com.tencent.cloud.csi.cfs
                           storagetype=SD
                           subnetid=subnet-n6wa7k6q
                           vers=3
                           vpcid=vpc-364sd70l
                           zone=ap-guangzhou-3
Events:                <none>
```

**注意：**
- 从腾讯云 cfs 中查看，可以看到启动了一个新的 cfs。
- 动态 PV 的挂载点就在根目录。
- 删除 PVC 时，会删掉 PV（因为是 Delete 回收策略），此时腾讯云 cfs 中可以看到文件系统数量少了一个。
