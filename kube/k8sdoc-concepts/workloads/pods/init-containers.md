# Init 容器

Init 容器是一种特殊容器，在 Pod 内的应用容器启动之前运行。Init 容器可以包括一些应用镜像中不存在的实用工具和安装脚本。

你可以在 Pod 的规约中与用来描述应用容器的 containers 数组平行的位置指定 Init 容器。

## 理解 Init 容器

每个 Pod 中可以包含多个容器， 应用运行在这些容器里面，同时 **Pod 也可以有一个或多个先于应用容器启动的 Init 容器**。

Init 容器与普通的容器非常像，除了如下两点：

- 它们总是运行到完成。
- 每个都必须在下一个启动之前成功完成。


**如果 Pod 的 Init 容器失败，kubelet 会不断地重启该 Init 容器直到该容器成功为止**，但如果 Pod 对应的 restartPolicy 值为 "Never"，并且 Pod 的 Init 容器失败， 则 Kubernetes 会将整个 Pod 状态设置为失败。

为 Pod 设置 Init 容器需要在 Pod Spec 中添加 initContainers 字段，该字段以 Container 类型对象数组的形式组织，和应用的 containers 数组同级相邻。

Init 容器的状态在 status.initContainerStatuses 字段中以容器状态数组的格式返回 （类似 status.containerStatuses 字段）。

### 与普通容器的不同之处

Init 容器支持应用容器的全部字段和特性，包括资源限制、数据卷和安全设置。 然而，Init 容器对资源请求和限制的处理稍有不同。

同时 Init 容器不支持 lifecycle、livenessProbe、readinessProbe 和 startupProbe，因为它们必须在 Pod 就绪之前运行完成。

如果为一个 Pod 指定了多个 Init 容器，这些容器会按顺序逐个运行。每个 Init 容器必须运行成功，下一个才能够运行。当所有的 Init 容器运行完成时， Kubernetes 才会为 Pod 初始化应用容器并像平常一样运行。

## 使用 Init 容器

因为 Init 容器具有与应用容器分离的单独镜像，其启动相关代码具有如下优势：

- Init 容器可以包含一些安装过程中应用容器中不存在的实用工具或个性化代码。 例如，没有必要仅为了在安装过程中使用类似 sed、awk、python 或 dig 这样的工具而去 FROM 一个镜像来生成一个新的镜像。
- Init 容器可以安全地运行这些工具，避免这些工具导致应用镜像的安全性降低。
- 应用镜像的创建者和部署者可以各自独立工作，而没有必要联合构建一个单独的应用镜像。
- Init 容器能以不同于 Pod 内应用容器的文件系统视图运行。因此，Init 容器可以访问 应用容器不能访问的 Secret 的权限。

由于 Init 容器必须在应用容器启动之前运行完成，因此 Init 容器提供了一种机制来阻塞或延迟应用容器的启动，直到满足了一组先决条件。 一旦前置条件满足，Pod 内的所有的应用容器会并行启动。

### 使用 Init 容器的情况

下面的例子定义了一个具有 2 个 Init 容器的简单 Pod。

第一个等待 myservice 启动， 第二个等待 mydb 启动。 一旦这两个 Init 容器都启动完成，Pod 将启动 spec 中的应用容器。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app.kubernetes.io/name: MyApp
spec:
  containers:
  - name: myapp-container
    image: busybox:1.28
    command: ['sh', '-c', 'echo The app is running! && sleep 3600']
  initContainers:
  - name: init-myservice
    image: busybox:1.28
    command: ['sh', '-c', "until nslookup myservice.$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace).svc.cluster.local; do echo waiting for myservice; sleep 2; done"]
  - name: init-mydb
    image: busybox:1.28
    command: ['sh', '-c', "until nslookup mydb.$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace).svc.cluster.local; do echo waiting for mydb; sleep 2; done"]
```

你通过运行下面的命令启动 Pod：

```sh
$ kubectl apply -f myapp.yaml
pod/myapp-pod created
```

使用下面的命令检查其状态：

```sh
$ kubectl get -f myapp.yaml
NAME        READY     STATUS     RESTARTS   AGE
myapp-pod   0/1       Init:0/2   0          6m

$ kubectl describe -f myapp.yaml
Name:          myapp-pod
Namespace:     default
[...]
Labels:        app.kubernetes.io/name=MyApp
Status:        Pending
[...]
Init Containers:
  init-myservice:
[...]
    State:         Running
[...]
  init-mydb:
[...]
    State:         Waiting
      Reason:      PodInitializing
    Ready:         False
[...]
Containers:
  myapp-container:
[...]
    State:         Waiting
      Reason:      PodInitializing
    Ready:         False
[...]
Events:
  FirstSeen    LastSeen    Count    From                      SubObjectPath                           Type          Reason        Message
  ---------    --------    -----    ----                      -------------                           --------      ------        -------
  16s          16s         1        {default-scheduler }                                              Normal        Scheduled     Successfully assigned myapp-pod to 172.17.4.201
  16s          16s         1        {kubelet 172.17.4.201}    spec.initContainers{init-myservice}     Normal        Pulling       pulling image "busybox"
  13s          13s         1        {kubelet 172.17.4.201}    spec.initContainers{init-myservice}     Normal        Pulled        Successfully pulled image "busybox"
  13s          13s         1        {kubelet 172.17.4.201}    spec.initContainers{init-myservice}     Normal        Created       Created container with docker id 5ced34a04634; Security:[seccomp=unconfined]
  13s          13s         1        {kubelet 172.17.4.201}    spec.initContainers{init-myservice}     Normal        Started       Started container with docker id 5ced34a04634
```

如需查看 Pod 内 Init 容器的日志(这里的日志其实就是容器的标准错误输出)，请执行：

```sh
kubectl logs myapp-pod -c init-myservice # 查看第一个 Init 容器
kubectl logs myapp-pod -c init-mydb      # 查看第二个 Init 容器
```

在这一刻，Init 容器将会等待至发现名称为 mydb 和 myservice 的 Service。

如下为创建这些 Service 的配置文件：

```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: myservice
spec:
  ports:
  - protocol: TCP
    port: 80
    targetPort: 9376
---
apiVersion: v1
kind: Service
metadata:
  name: mydb
spec:
  ports:
  - protocol: TCP
    port: 80
    targetPort: 9377
```

创建 mydb 和 myservice 服务的命令：

```yaml
$ kubectl apply -f services.yaml
service/myservice created
service/mydb created
```

这样你将能看到这些 Init 容器执行完毕，随后 my-app 的 Pod 进入 Running 状态：

```sh
$ kubectl get -f myapp.yaml
NAME        READY     STATUS    RESTARTS   AGE
myapp-pod   1/1       Running   0          9m
```

这个简单例子应该能为你创建自己的 Init 容器提供一些启发。 接下来节提供了更详细例子的链接。

## 具体行为

在 Pod 启动过程中，每个 Init 容器会在**网络和数据卷初始化之后**按顺序启动。

kubelet 运行依据 Init 容器在 Pod Spec 中的出现顺序依次运行之:

- 每个 Init 容器成功退出后才会启动下一个 Init 容器。
- 如果某容器因为容器运行时的原因无法启动，或以错误状态退出，kubelet 会根据 Pod 的 restartPolicy 策略进行重试。
- 如果 Pod 的 restartPolicy 设置为 "Always"，Init 容器失败时会使用 restartPolicy 的 "OnFailure" 策略。
如果 Pod 重启，所有 Init 容器必须重新执行。

在所有的 Init 容器没有成功之前:

- 正在初始化中的 Pod 处于 Pending 状态，但会将状况 Initializing 设置为 false。
- Pod 将不会变成 Ready 状态。Init 容器的端口将不会在 Service 中进行聚集。

对 Init 容器规约的修改仅限于容器的 image 字段。**更改 Init 容器的 image 字段，等同于重启该 Pod。**

因为 Init 容器可能会被重启、重试或者重新执行，所以 Init 容器的执行应该是幂等的。特别地，基于 emptyDirs 写文件的代码，应该对输出文件可能已经存在做好准备。

Init 容器具有应用容器的所有字段。然而 Kubernetes 禁止使用 readinessProbe， 因为 Init 容器不能定义不同于完成态（Completion）的就绪态（Readiness）。 Kubernetes 会在校验时强制执行此检查。

在 Pod 上使用 activeDeadlineSeconds 和在容器上使用 livenessProbe 可以避免 Init 容器一直重复失败:

- activeDeadlineSeconds 时间包含了 Init 容器启动的时间。
- 建议仅在团队将其应用程序部署为 Job 时才使用 activeDeadlineSeconds， 因为 activeDeadlineSeconds 在 Init 容器结束后仍有效果。
- 如果你设置了 activeDeadlineSeconds，已经在正常运行的 Pod 会被杀死。

在 Pod 中的每个应用容器和 Init 容器的名称必须唯一, 与任何其它容器共享同一个名称，会在校验时抛出错误。

### 资源

在给定的 Init 容器执行顺序下，资源使用适用于如下规则：

- 所有 Init 容器上定义的任何特定资源的 limit 或 request 的最大值，作为 Pod 有效初始 request/limit。 如果任何资源没有指定资源限制，这被视为最高限制。
- Pod 对资源的 **有效 limit/request** 是如下两者中的较大者：
  - 所有应用容器对某个资源的 limit/request 之和
  - 对某个资源的有效初始 limit/request
- 基于有效 limit/request 完成调度，这意味着 Init 容器能够为初始化过程预留资源， 这些资源在 Pod 生命周期过程中并没有被使用。
- Pod 的 **有效 QoS 层**，与 Init 容器和应用容器的一样。

### Pod 重启的原因

Pod 重启会导致 Init 容器重新执行，主要有如下几个原因：

- Pod 的基础设施容器 (如 pause 容器) 被重启。这种情况不多见，必须由具备 root 权限访问节点的人员来完成。
- 当 restartPolicy 设置为 Always，Pod 中所有容器会终止而强制重启。由于垃圾收集机制的原因，Init 容器的完成记录将会丢失。
