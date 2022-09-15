# Deployments

一个 Deployment 为 Pod 和 ReplicaSet 提供声明式的更新能力。

你负责描述 Deployment 中的**目标状态（Desired State）**，而 Deployment 控制器以受控速率更改**实际状态（actual state）**， 使其变为**期望状态（Desired State）**。

你可以定义 Deployment 以创建新的 ReplicaSet，或删除现有 Deployment， 并通过新的 Deployment 收养其资源。

**注意：**

- 不要管理 Deployment 所拥有的 ReplicaSet。
- 如果存在下面未覆盖的使用场景，请考虑在 Kubernetes 仓库中提出 Issue。

## 用例

以下是 Deployments 的典型用例：

- 创建 Deployment 以将 ReplicaSet 上线。ReplicaSet 在后台创建 Pod。检查 ReplicaSet 的上线状态，查看其是否成功。
- 通过更新 Deployment 的 PodTemplateSpec，声明 Pod 的新状态 。 新的 ReplicaSet 会被创建，Deployment 以受控速率将 Pod 从旧 ReplicaSet 迁移到新 ReplicaSet。 每个新的 ReplicaSet 都会- 更新 Deployment 的版本。
- 如果 Deployment 的当前状态不稳定，回滚到较早的 Deployment 版本。 每次回滚都会更新 Deployment 的版本。
- 扩大 Deployment 规模以承担更多负载。
- 暂停 Deployment 以应用对 PodTemplateSpec 所作的多项修改，然后恢复其执行以启动新的上线版本。
- 使用 Deployment 状态来判定上线过程是否出现停滞。
- 清理较旧的不再需要的 ReplicaSet 。


## 创建 Deployment

下面是一个 Deployment 示例。其中创建了一个 ReplicaSet，负责启动三个 nginx Pod：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```

在该例中：

- 创建名为 `nginx-deployment`（由 .metadata.name 字段标明）的 Deployment。
- 该 Deployment 创建三个（由 .spec.replicas 字段标明）Pod 副本。
- `.spec.selector` 字段定义了 Deployment 如何查找要管理的 Pod。
- `selector` 字段定义 Deployment 如何查找要管理的 Pod。在这里，你选择在 Pod 模板中定义的标签（app: nginx）。不过，更复杂的选择规则是也可能的，只要 Pod 模板本身满足所给规则即可。

template 字段包含以下子字段：

- Pod 被使用 .metadata.labels 字段打上 app: nginx 标签。
- Pod 模板规约（即 .template.spec 字段）指示 Pod 运行一个 nginx 容器， 该容器运行版本为 1.14.2 的 nginx Docker Hub 镜像。
- 创建一个容器并使用 .spec.template.spec.containers[0].name 字段将其命名为 nginx。

pod 的名称是 deployment 自动生成的，使用 deployment 加一个随机后缀。

开始之前，请确保的 Kubernetes 集群已启动并运行。 按照以下步骤创建上述 Deployment：

```sh
$ kubectl apply -f https://k8s.io/examples/controllers/nginx-deployment.yaml

$ kubectl get deployments
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   0/3     0            0           1s
```

在检查集群中的 Deployment 时，所显示的字段有：

- NAME 列出了名字空间中 Deployment 的名称。
- READY 显示应用程序的可用的“副本”数。显示的模式是“就绪个数/期望个数”。其实就是 Pod 的实际个数和期望个数。
- UP-TO-DATE 显示为了达到期望状态已经更新的副本数。
- AVAILABLE 显示应用可供用户使用的副本数。
- AGE 显示应用程序运行的时间。

要查看 Deployment 上线状态：

```sh
$ kubectl rollout status deployment/nginx-deployment

Waiting for rollout to finish: 2 out of 3 new replicas have been updated...
deployment "nginx-deployment" successfully rolled out
```

几秒钟后再次运行 kubectl get deployments。输出类似于：

```sh
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3/3     3            3           18s
```

注意 Deployment 已创建全部三个副本，并且所有副本都是最新的（它们包含最新的 Pod 模板） 并且可用。

要查看 Deployment 创建的 ReplicaSet：

```sh
$ kubectl get rs
NAME                          DESIRED   CURRENT   READY   AGE
nginx-deployment-75675f5897   3         3         3       18s
```

- NAME 列出名字空间中 ReplicaSet 的名称。
- DESIRED 显示应用的期望副本个数，即在创建 Deployment 时所定义的值。此为期望状态。
- CURRENT 显示当前运行状态中的副本个数。
- READY 显示应用中有多少副本可以为用户提供服务。
- AGE 显示应用已经运行的时间长度。

注意 ReplicaSet 的名称始终被格式化为 `[Deployment名称]-[随机字符串]`。其中的随机字符串是使用 **pod-template-hash** 作为种子随机生成的。

要查看每个 Pod 自动生成的标签：

```sh
$ kubectl get pods --show-labels
NAME                                READY     STATUS    RESTARTS   AGE       LABELS
nginx-deployment-75675f5897-7ci7o   1/1       Running   0          18s       app=nginx,pod-template-hash=3123191453
nginx-deployment-75675f5897-kzszj   1/1       Running   0          18s       app=nginx,pod-template-hash=3123191453
nginx-deployment-75675f5897-qqcnn   1/1       Running   0          18s       app=nginx,pod-template-hash=3123191453
```

所创建的 ReplicaSet 确保总是存在三个 nginx Pod。

**注意：**

- 你必须在 Deployment 中指定适当的选择算符和 Pod 模板标签（在本例中为 app: nginx）。
- 标签或者选择算符不要与其他控制器（包括其他 Deployment 和 StatefulSet）重叠。 Kubernetes 不会阻止你这样做，但是如果多个控制器具有重叠的选择算符，它们可能会发生冲突执行难以预料的操作。

### Pod-template-hash 标签

Deployment Controller 将 `pod-template-hash` 标签添加到 Deployment 所创建或收留的每个 ReplicaSet（Pod 中也会加上该标签）。

此标签可确保 Deployment 的子 ReplicaSets 不重叠。标签是通过对 ReplicaSet 的 PodTemplate 进行哈希处理。 所生成的哈希值被添加到 ReplicaSet 选择算符、Pod 模板标签，并存在于在 ReplicaSet 可能拥有的任何现有 Pod 中。

## 更新 Deployment

Deployment 的更新触发时机：

- 当 Deployment Pod 模板（即 .spec.template）发生改变时（例如模板的标签或容器镜像被更新），才会触发 Deployment 更新。
- 其他更新（如对 Deployment 执行扩缩容的操作）不会触发上线动作。

按照以下步骤更新 Deployment：

1. 先来更新 nginx Pod 以使用 nginx:1.16.1 镜像，而不是 nginx:1.14.2 镜像。

```sh
$ kubectl set image deployment.v1.apps/nginx-deployment nginx=nginx:1.16.1
deployment.apps/nginx-deployment image updated
```
2. 查看上线状态：

```sh
$ kubectl rollout status deployment/nginx-deployment
Waiting for rollout to finish: 2 out of 3 new replicas have been updated...

# 当更新完成，输出:
# deployment "nginx-deployment" successfully rolled out
```

3. 获取关于已更新的 Deployment 的更多信息：

```sh
$ kubectl get deployments
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3/3     3            3           36s

# Deployment 通过新建一个新 rs 扩容，再将旧 rs 缩容来实现更新，因此会有两个 rs，他们的 pod-template-hash 不同
$ kubectl get rs
NAME                          DESIRED   CURRENT   READY   AGE
nginx-deployment-1564180365   3         3         3       6s
nginx-deployment-2035384211   0         0         0       36s

# 显示新 pod
$ kubectl get pods
NAME                                READY     STATUS    RESTARTS   AGE
nginx-deployment-1564180365-khku8   1/1       Running   0          14s
nginx-deployment-1564180365-nacti   1/1       Running   0          14s
nginx-deployment-1564180365-z9gth   1/1       Running   0          14s
```

Deployment 可确保在更新时仅关闭一定数量的 Pod。**默认情况下**，它确保至少所需 Pod 的 75% 处于运行状态（最大不可用比例为 25%）。

Deployment 还确保仅所创建 Pod 数量只可能比期望 Pod 数高一点点。 默认情况下，它可确保启动的 Pod 个数比期望个数最多多出 125%（最大峰值 25%）。

例如，如果仔细查看上述 Deployment ，将看到:

1. 它首先创建了一个新的 Pod
1. 然后删除旧的 Pod，并创建了新的 Pod。
1. 它不会杀死旧 Pod，直到有足够数量的新 Pod 已经出现。
1. 在足够数量的旧 Pod 被杀死前并没有创建新 Pod。它确保至少 3 个 Pod 可用，同时最多总共 4 个 Pod 可用。
1. 当 Deployment 设置为 4 个副本时，Pod 的个数会介于 3 和 5 之间。

更多 Deployment 信息：

```sh
$ kubectl describe deployments
Name:                   nginx-deployment
Namespace:              default
CreationTimestamp:      Thu, 30 Nov 2017 10:56:25 +0000
Labels:                 app=nginx
Annotations:            deployment.kubernetes.io/revision=2
Selector:               app=nginx
Replicas:               3 desired | 3 updated | 3 total | 3 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  app=nginx
   Containers:
    nginx:
      Image:        nginx:1.16.1
      Port:         80/TCP
      Environment:  <none>
      Mounts:       <none>
    Volumes:        <none>
  Conditions:
    Type           Status  Reason
    ----           ------  ------
    Available      True    MinimumReplicasAvailable
    Progressing    True    NewReplicaSetAvailable
  OldReplicaSets:  <none>
  NewReplicaSet:   nginx-deployment-1564180365 (3/3 replicas created)
  Events:
    Type    Reason             Age   From                   Message
    ----    ------             ----  ----                   -------
    Normal  ScalingReplicaSet  2m    deployment-controller  Scaled up replica set nginx-deployment-2035384211 to 3
    Normal  ScalingReplicaSet  24s   deployment-controller  Scaled up replica set nginx-deployment-1564180365 to 1
    Normal  ScalingReplicaSet  22s   deployment-controller  Scaled down replica set nginx-deployment-2035384211 to 2
    Normal  ScalingReplicaSet  22s   deployment-controller  Scaled up replica set nginx-deployment-1564180365 to 2
    Normal  ScalingReplicaSet  19s   deployment-controller  Scaled down replica set nginx-deployment-2035384211 to 1
    Normal  ScalingReplicaSet  19s   deployment-controller  Scaled up replica set nginx-deployment-1564180365 to 3
    Normal  ScalingReplicaSet  14s   deployment-controller  Scaled down replica set nginx-deployment-2035384211 to 0
```

可以看：

1. 当第一次创建 Deployment 时，它创建了一个 ReplicaSet（nginx-deployment-2035384211） 并将其直接扩容至 3 个副本。
1. 更新 Deployment 时，它创建了一个新的 ReplicaSet （nginx-deployment-1564180365），并将其扩容为 1，等待其就绪；
1. 然后将旧 ReplicaSet 缩容到 2， 将新的 ReplicaSet 扩容到 2 以便至少有 3 个 Pod 可用且最多创建 4 个 Pod。
1. 然后，它使用相同的滚动更新策略继续对新的 ReplicaSet 扩容并对旧的 ReplicaSet 缩容。 最后，你将有 3 个可用的副本在新的 ReplicaSet 中，旧 ReplicaSet 将缩容到 0。


**注意：**

- Kubernetes 在计算 availableReplicas 数值时不考虑终止过程中的 Pod，availableReplicas 的值一定介于 replicas - maxUnavailable 和 replicas + maxSurge 之间。
- 因此，你可能在上线期间看到 Pod 个数比预期的多，Deployment 所消耗的总的资源也大于 replicas + maxSurge 个 Pod 所用的资源，直到被终止的 Pod 所设置的 terminationGracePeriodSeconds 到期为止。

### 翻转（多 Deployment 动态更新）

## 回滚 Deployment

有时，你可能想要回滚 Deployment；例如，当 Deployment 不稳定时（例如进入反复崩溃状态）。

默认情况下，Deployment 的所有上线记录都保留在系统中，以便可以随时回滚 （你可以通过修改修订历史记录限制来更改这一约束）。

- 假设你在更新 Deployment 时犯了一个拼写错误，将镜像名称命名设置为 nginx:1.161 而不是 nginx:1.16.1：

```sh
$ kubectl set image deployment/nginx-deployment nginx=nginx:1.161 
deployment.apps/nginx-deployment image updated
```

- 此上线进程会出现停滞。你可以通过检查上线状态来验证：

```sh
$ kubectl rollout status deployment/nginx-deployment
Waiting for rollout to finish: 1 out of 3 new replicas have been updated...
```

- 同时可以看到有两个旧的 rs（一个是最先创建的，已经被缩容到 0 了，另一个是后面创建的，需要等最新版本 READY 一个 pod 后，才能删掉一个 Pod）

```sh
$ kubectl get rs
NAME                          DESIRED   CURRENT   READY   AGE
nginx-deployment-1564180365   3         3         3       25s
nginx-deployment-2035384211   0         0         0       36s
nginx-deployment-3066724191   1         1         0       6s
```

- 查看所创建的 Pod，你会注意到新 ReplicaSet 所创建的 1 个 Pod 卡顿在镜像拉取循环中。

```sh
$ kubectl get pods
NAME                                READY     STATUS             RESTARTS   AGE
nginx-deployment-1564180365-70iae   1/1       Running            0          25s
nginx-deployment-1564180365-jbqqo   1/1       Running            0          25s
nginx-deployment-1564180365-hysrc   1/1       Running            0          25s
nginx-deployment-3066724191-08mng   0/1       ImagePullBackOff   0          6s
```
- 获取 Deployment 信息：

```sh
$ kubectl describe deployment
Name:           nginx-deployment
Namespace:      default
CreationTimestamp:  Tue, 15 Mar 2016 14:48:04 -0700
Labels:         app=nginx
Selector:       app=nginx
Replicas:       3 desired | 1 updated | 4 total | 3 available | 1 unavailable
StrategyType:       RollingUpdate
MinReadySeconds:    0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  app=nginx
  Containers:
   nginx:
    Image:        nginx:1.161
    Port:         80/TCP
    Host Port:    0/TCP
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    ReplicaSetUpdated
OldReplicaSets:     nginx-deployment-1564180365 (3/3 replicas created)
NewReplicaSet:      nginx-deployment-3066724191 (1/1 replicas created)
Events:
  FirstSeen LastSeen    Count   From                    SubobjectPath   Type        Reason              Message
  --------- --------    -----   ----                    -------------   --------    ------              -------
  1m        1m          1       {deployment-controller }                Normal      ScalingReplicaSet   Scaled up replica set nginx-deployment-2035384211 to 3
  22s       22s         1       {deployment-controller }                Normal      ScalingReplicaSet   Scaled up replica set nginx-deployment-1564180365 to 1
  22s       22s         1       {deployment-controller }                Normal      ScalingReplicaSet   Scaled down replica set nginx-deployment-2035384211 to 2
  22s       22s         1       {deployment-controller }                Normal      ScalingReplicaSet   Scaled up replica set nginx-deployment-1564180365 to 2
  21s       21s         1       {deployment-controller }                Normal      ScalingReplicaSet   Scaled down replica set nginx-deployment-2035384211 to 1
  21s       21s         1       {deployment-controller }                Normal      ScalingReplicaSet   Scaled up replica set nginx-deployment-1564180365 to 3
  13s       13s         1       {deployment-controller }                Normal      ScalingReplicaSet   Scaled down replica set nginx-deployment-2035384211 to 0
  13s       13s         1       {deployment-controller }                Normal      ScalingReplicaSet   Scaled up replica set nginx-deployment-3066724191 to 1
```

要解决此问题，要么回滚，要么发新版本。

### 检查 Deployment 上线历史

检查历史修订版本：

```sh
$ kubectl rollout history deployment/nginx-deployment
deployments "nginx-deployment"
REVISION    CHANGE-CAUSE
1           kubectl apply --filename=https://k8s.io/examples/controllers/nginx-deployment.yaml
2           kubectl set image deployment/nginx-deployment nginx=nginx:1.16.1
3           kubectl set image deployment/nginx-deployment nginx=nginx:1.161
```

CHANGE-CAUSE 的内容是从 Deployment 的 kubernetes.io/change-cause 注解复制过来的。这个 CHANGE-CAUSE 复制动作发生在修订版本创建时。

你也可以在 Deployment 更新后，通过以下方式设置 CHANGE-CAUSE 消息：

```sh
$ kubectl annotate deployment/nginx-deployment kubernetes.io/change-cause="image updated to 1.16.1"
```

要查看修订历史的详细信息，运行：

```sh
$ kubectl rollout history deployment/nginx-deployment --revision=2

deployments "nginx-deployment" revision 2
  Labels:       app=nginx
          pod-template-hash=1159050644
  Annotations:  kubernetes.io/change-cause=kubectl set image deployment/nginx-deployment nginx=nginx:1.16.1
  Containers:
   nginx:
    Image:      nginx:1.16.1
    Port:       80/TCP
     QoS Tier:
        cpu:      BestEffort
        memory:   BestEffort
    Environment Variables:      <none>
  No volumes.
```

### 回滚到之前的修订版本

1. 假定现在你已决定撤消当前上线并回滚到以前的修订版本：

```sh
$ kubectl rollout undo deployment/nginx-deployment

deployment.apps/nginx-deployment rolled back

# 你也可以通过使用 --to-revision 来回滚到特定修订版本：
# kubectl rollout undo deployment/nginx-deployment --to-revision=2
```

1. 检查回滚是否成功以及 Deployment 是否正在运行:

```sh
$ kubectl get deployment nginx-deployment
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3/3     3            3           30m

$ kubectl describe deployment nginx-deployment
Name:                   nginx-deployment
Namespace:              default
CreationTimestamp:      Sun, 02 Sep 2018 18:17:55 -0500
Labels:                 app=nginx
Annotations:            deployment.kubernetes.io/revision=4
                        kubernetes.io/change-cause=kubectl set image deployment/nginx-deployment nginx=nginx:1.16.1
Selector:               app=nginx
Replicas:               3 desired | 3 updated | 3 total | 3 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  app=nginx
  Containers:
   nginx:
    Image:        nginx:1.16.1
    Port:         80/TCP
    Host Port:    0/TCP
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    NewReplicaSetAvailable
OldReplicaSets:  <none>
NewReplicaSet:   nginx-deployment-c4747d96c (3/3 replicas created)
Events:
  Type    Reason              Age   From                   Message
  ----    ------              ----  ----                   -------
  Normal  ScalingReplicaSet   12m   deployment-controller  Scaled up replica set nginx-deployment-75675f5897 to 3
  Normal  ScalingReplicaSet   11m   deployment-controller  Scaled up replica set nginx-deployment-c4747d96c to 1
  Normal  ScalingReplicaSet   11m   deployment-controller  Scaled down replica set nginx-deployment-75675f5897 to 2
  Normal  ScalingReplicaSet   11m   deployment-controller  Scaled up replica set nginx-deployment-c4747d96c to 2
  Normal  ScalingReplicaSet   11m   deployment-controller  Scaled down replica set nginx-deployment-75675f5897 to 1
  Normal  ScalingReplicaSet   11m   deployment-controller  Scaled up replica set nginx-deployment-c4747d96c to 3
  Normal  ScalingReplicaSet   11m   deployment-controller  Scaled down replica set nginx-deployment-75675f5897 to 0
  Normal  ScalingReplicaSet   11m   deployment-controller  Scaled up replica set nginx-deployment-595696685f to 1
  Normal  DeploymentRollback  15s   deployment-controller  Rolled back deployment "nginx-deployment" to revision 2
  Normal  ScalingReplicaSet   15s   deployment-controller  Scaled down replica set nginx-deployment-595696685f to 0
```

回滚本质上是会增加一个修订版本，可以看到此时 `revision=4`。

## Deployment 扩容

扩容命令：

```sh
$ kubectl scale deployment/nginx-deployment --replicas=10
deployment.apps/nginx-deployment scaled
```

除了手动扩缩容外，还可以启用了 Pod 的自动缩放，你可以为 Deployment 设置自动缩放器。自动扩缩容可以基于现有 Pod 的 CPU 利用率选择要运行的 Pod 个数下限和上限：

```sh
# Pod 最小 10 个，当 CPU 达到 80% 开始自动扩容，最多可以创建 15 个 Pod
$ kubectl autoscale deployment/nginx-deployment --min=10 --max=15 --cpu-percent=80

deployment.apps/nginx-deployment scaled
```

### 比例缩放

RollingUpdate 的 Deployment 支持同时运行应用程序的多个版本。

当自动扩缩容对上线进程（仍在进行中或暂停）中的 RollingUpdate Deployment 进行扩容时，Deployment 控制器会平衡现有的活跃状态的 ReplicaSets（含 Pod 的 ReplicaSets）中的额外副本，以降低风险。这称为`比例缩放（Proportional Scaling）`。

例如，你正在运行一个 10 个副本的 Deployment，其 maxSurge=3，maxUnavailable=2：

```sh
# 确定有 10 个副本
$ kubectl get deploy
NAME                 DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment     10        10        10           10          50s

# 更新 Deployment 使用新镜像，碰巧该镜像无法从集群内部解析。
$ kubectl set image deployment/nginx-deployment nginx=nginx:sometag
deployment.apps/nginx-deployment image updated

# 镜像更新使用 ReplicaSet nginx-deployment-1989198191 启动新的上线过程
# 但由于上面提到的 maxUnavailable 要求，该进程被阻塞了
$ kubectl get rs
NAME                          DESIRED   CURRENT   READY     AGE
nginx-deployment-1989198191   5         5         0         9s
nginx-deployment-618515232    8         8         8         1m
```

现在，出现了新的 Deployment 扩缩请求，自动缩放器将 Deployment 副本增加到 15。

Deployment 控制器需要决定在何处添加 5 个新副本：

- 如果**未使用比例缩放**，所有 5 个副本都将**添加到新的 ReplicaSet 中**。
- 使用比例缩放时，可以将额外的副本分布到所有 ReplicaSet。
  - 较大比例的副本会被添加到拥有最多副本的 ReplicaSet
  - 较低比例的副本会进入到 副本较少的 ReplicaSet。
  - 所有剩下的副本都会添加到副本最多的 ReplicaSet。
  - 具有零副本的 ReplicaSets 不会被扩容。

```sh
# 3 个副本被添加到旧 ReplicaSet 中，2 个副本被添加到新 ReplicaSet。
$ kubectl get deploy
NAME                 DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment     15        18        7            8           7m

# 上线状态确认了副本是如何被添加到每个 ReplicaSet 的。
$ kubectl get rs
NAME                          DESIRED   CURRENT   READY     AGE
nginx-deployment-1989198191   7         7         0         7m
nginx-deployment-618515232    11        11        11        7m
```

## 暂停、恢复 Deployment 的上线过程

## Deployment 状态

Deployment 的生命周期中会有许多状态：

- Progressing（进行中）
- Complete（已完成）
- Failed（失败）

### Progressing

执行下面的任务期间，Kubernetes 标记 Deployment 为进行中（Progressing）：

- Deployment 创建新的 ReplicaSet
- Deployment 正在为其最新的 ReplicaSet 扩容
- Deployment 正在为其旧有的 ReplicaSet(s) 缩容
- 新的 Pod 已经就绪或者可用（就绪至少持续了 MinReadySeconds 秒）。

当进入 “Progressing” 状态时，Deployment 控制器会向 Deployment 的 `.status.conditions` 中添加包含下面属性的状况条目：

```txt
type: Progressing
status: "True"
reason: NewReplicaSetCreated | reason: FoundNewReplicaSet | reason: ReplicaSetUpdated
```

你可以使用 `kubectl rollout status` 监视 Deployment 的进度。

### 完成的 Deployment

当 Deployment 具有以下特征时，Kubernetes 将其标记为完成（Complete）：

- 与 Deployment 关联的所有副本都已更新到指定的最新版本，这意味着之前请求的所有更新都已完成。
- 与 Deployment 关联的所有副本都可用。
- Deployment 没有还在运行的旧副本。

当进入“Complete”状态时，Deployment 控制器会向 Deployment 的 `.status.conditions` 中添加包含下面属性的状况条目：

```txt
type: Progressing
status: "True"
reason: NewReplicaSetAvailable
```

这一 Progressing 状况的状态值会持续为 "True"，直至新的上线动作被触发。 即使副本的可用状态发生变化（进而影响 Available 状况），Progressing 状况的值也不会变化。

你可以使用 `kubectl rollout status` 检查 Deployment 是否已完成。如果上线成功完成，kubectl rollout status 返回退出代码 0。

### 失败的 Deployment

你的 Deployment 可能会在尝试部署其最新的 ReplicaSet 受挫，一直处于未完成状态。

造成此情况一些可能因素如下：

- 配额（Quota）不足
- 就绪探测（Readiness Probe）失败
- 镜像拉取错误
- 权限不足
- 限制范围（Limit Ranges）问题
- 应用程序运行时的配置错误

检测此状况的一种方法是在 Deployment 规约中指定截止时间参数：`.spec.progressDeadlineSeconds`。

progressDeadlineSeconds 给出的是一个秒数值，Deployment 控制器在（通过 Deployment 状态）标示 Deployment 进展停滞之前，需要等待所给的时长。

以下 kubectl 命令设置规约中的 progressDeadlineSeconds，从而告知控制器 在 10 分钟后报告 Deployment 没有进展：

```sh
$ kubectl patch deployment/nginx-deployment -p '{"spec":{"progressDeadlineSeconds":600}}'
```

超过截止时间后，Deployment 控制器将添加具有以下属性的 Deployment 状况到 Deployment 的 .status.conditions 中：

```txt
type: Progressing
status: "False"
reason: ProgressDeadlineExceeded
```

这一状况也可能会比较早地失败，因而其状态值被设置为 "False"， 其原因为 ReplicaSetCreateError。一旦 Deployment 上线完成，就不再考虑其期限。


Deployment 可能会出现瞬时性的错误，可能因为设置的超时时间过短， 也可能因为其他可认为是临时性的问题。

例如，假定所遇到的问题是配额不足。 如果描述 Deployment，你将会注意到以下部分：

```sh
$ kubectl describe deployment nginx-deployment
<...>
Conditions:
  Type            Status  Reason
  ----            ------  ------
  Available       True    MinimumReplicasAvailable
  Progressing     True    ReplicaSetUpdated
  ReplicaFailure  True    FailedCreate
<...>
```

如果运行 `kubectl get deployment nginx-deployment -o yaml`，Deployment 状态输出 将类似于这样：

```sh
status:
  availableReplicas: 2
  conditions:
  - lastTransitionTime: 2016-10-04T12:25:39Z
    lastUpdateTime: 2016-10-04T12:25:39Z
    message: Replica set "nginx-deployment-4262182780" is progressing.
    reason: ReplicaSetUpdated
    status: "True"
    type: Progressing
  - lastTransitionTime: 2016-10-04T12:25:42Z
    lastUpdateTime: 2016-10-04T12:25:42Z
    message: Deployment has minimum availability.
    reason: MinimumReplicasAvailable
    status: "True"
    type: Available
  - lastTransitionTime: 2016-10-04T12:25:39Z
    lastUpdateTime: 2016-10-04T12:25:39Z
    message: 'Error creating: pods "nginx-deployment-4262182780-" is forbidden: exceeded quota:
      object-counts, requested: pods=1, used: pods=3, limited: pods=2'
    reason: FailedCreate
    status: "True"
    type: ReplicaFailure
  observedGeneration: 3
  replicas: 2
  unavailableReplicas: 2
```

最终，一旦超过 Deployment 进度限期，Kubernetes 将更新状态和进度状况的原因：

```sh
Conditions:
  Type            Status  Reason
  ----            ------  ------
  Available       True    MinimumReplicasAvailable
  Progressing     False   ProgressDeadlineExceeded
  ReplicaFailure  True    FailedCreate
```

### 对失败 Deployment 的操作

可应用于已完成的 Deployment 的所有操作也适用于失败的 Deployment。 你可以对其执行扩缩容、回滚到以前的修订版本等操作，或者在需要对 Deployment 的 Pod 模板应用多项调整时，将 Deployment 暂停。

## 清理策略

Deployment 更新后，旧的 rs 会继续保留方便回滚，那么如何对其进行清理呢？

你可以在 Deployment 中设置 `.spec.revisionHistoryLimit` 字段以指定保留此 Deployment 的多少个旧有 ReplicaSet。其余的 ReplicaSet 将在后台被垃圾回收。

`默认情况下，此值为 10`。

**注意：**

- 当被回收后，该版本就没有办法回滚了。
- 将此字段设置为 0 将导致 Deployment 的所有历史记录被清空，因此 Deployment 将无法回滚。

## 金丝雀部署

如果要使用 Deployment 向用户子集或服务器子集上线版本， 则可以遵循[资源管理](https://kubernetes.io/zh-cn/docs/concepts/cluster-administration/manage-deployment/#canary-deployments)所描述的金丝雀模式， 创建多个 Deployment，每个版本一个。

## 编写 Deployment Spec