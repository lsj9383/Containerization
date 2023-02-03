# Pod 的生命周期

Pod 遵循一个预定义的生命周期(Pod 阶段):

- 起始于 Pending 阶段
- 如果至少中有一个主要容器正常启动，则进入 Running
- 之后取决于 Pod 中是否有容器以`失败状态`结束而进入 Succeeded 或者 Failed 阶段。

**注意:**

- Pod 除了由阶段外, 还有状态.
- 阶段是 Pod 的简单宏观概述.

在 Pod 运行期间，kubelet 能够重启容器以处理一些失效场景。 在 Pod 内部，Kubernetes 跟踪不同容器的状态 并确定使 Pod 重新变得健康所需要采取的动作。

在 Kubernetes API 中，Pod 包含规约部分和实际状态部分。 Pod 对象的状态包含了一组 Pod 状况（Conditions）。 如果应用需要的话，你也可以向其中注入自定义的就绪性信息。

Pod 在其生命周期中**只会被调度一次**。 一旦 Pod 被调度（分派）到某个节点，**Pod 会一直在该节点运行，直到 Pod 停止或者被终止**。

## Pod 生命期(Pod Lifetime)

和一个个独立的应用容器一样，Pod 也被认为是相对临时性（而不是长期存在）的实体:

- Pod 会被创建、赋予一个唯一的 ID（UID）
- 并被调度到节点
- 在终止（根据重启策略）或删除之前一直运行在该节点

如果一个 Node 死亡，调度到该 Node 的 Pod 会在超时后被调度删除。

Kubernetes 使用一种高级抽象来管理这些相对而言可随时丢弃的 Pod 实例，称作**控制器**。

如果某物声称其生命期与某 Pod 相同(例如存储卷), 这就意味着该对象在此 Pod （UID 亦相同）存在期间也一直存在。如果 Pod 因为任何原因被删除，甚至某完全相同的替代 Pod 被创建时，这个相关的对象（例如这里的卷）也会被删除并重建。

## Pod 阶段

Pod 的 **status** 字段是一个 PodStatus 对象，其中包含一个 **phase** 字段。

Pod 的阶段（Phase）是 Pod 在其生命周期中所处位置的简单宏观概述。**该阶段并不是对容器或 Pod 状态的综合汇总，也不是为了成为完整的状态机。**

Pod 阶段的数量和含义是严格定义的。 除了本文档中列举的内容外，不应该再假定 Pod 有其他的 phase 值。

下面是 phase 可能的值：

取值 | 描述
-|-
Pending（悬决） | Pod 已被 Kubernetes 系统接受，但有一个或者多个容器尚未创建亦未运行。此阶段包括等待 Pod 被调度的时间和通过网络下载镜像的时间。
Running（运行中） | Pod 已经绑定到了某个节点，Pod 中所有的容器都已被创建。至少有一个容器仍在运行，或者正处于启动或重启状态。
Succeeded（成功） | Pod 中的所有容器都已成功终止，并且不会再重启。
Failed（失败） | Pod 中的所有容器都已终止，并且至少有一个容器是因为失败终止。也就是说，容器以非 0 状态退出或者被系统终止。
Unknown（未知） | 因为某些原因无法取得 Pod 的状态。这种情况通常是因为与 Pod 所在主机通信失败。

**注意:**

- 当一个 Pod 被删除时，执行一些 kubectl 命令会展示这个 Pod 的状态为 Terminating（终止）。
- 这个 Terminating 状态并不是 Pod 阶段之一。 Pod 被赋予一个可以体面终止的期限，默认为 30 秒。 你可以使用 --force 参数来强制终止 Pod。

如果某节点死掉或者与集群中其他节点失联，Kubernetes 会实施一种策略，将失去的节点上运行的所有 Pod 的 phase 设置为 Failed。

```sh
$ kubectl describe pod nginx-deployment-66b6c48dd5-bzphg
...
Status:       Running
...
```

上面的 Status 是指的 Pod 的 phase.

```sh

$ kubectl get pods
NAME                                READY   STATUS      RESTARTS   AGE
easyngx                             1/1     Running     0          110m
hello--1-6pcq7                      0/1     Completed   0          103s
kubernetes-proxy-6564b76594-k7k4l   1/1     Running     0          4d13h
kubernetes-proxy-6564b76594-mbf7d   1/1     Running     0          4d13h
nginx-deployment-66b6c48dd5-bzphg   1/1     Running     0          4d12h
nginx-deployment-66b6c48dd5-f74zh   1/1     Running     0          4d12h
ngx-deployment-5765c75759-4v2zd     1/1     Running     0          4d14h
ngx-deployment-5765c75759-mj7xp     1/1     Running     0          4d14h
```

这里的 STATUS 并不是指的 phase.

## 容器状态

Kubernetes 会跟踪 Pod 中每个容器的状态，就像它跟踪 Pod 总体上的阶段一样。

一旦调度器将 Pod 分派给某个节点，kubelet 就通过 Container Runtime(如 Docker) 开始为 Pod 创建容器。

容器的状态有三种：

- Waiting（等待）
- Running（运行中）
- Terminated（已终止）。

要检查 Pod 中容器的状态，你可以使用 kubectl describe pod <pod 名称>。 其输出中包含 Pod 中每个容器的状态。

```sh
$ kubectl describe pod nginx-deployment-66b6c48dd5-bzphg
...
Containers:
  nginx:
    Container ID:   docker://6e3c89b09bc878134289b0bdecbff611b13214c13d6c1db75f7b8256dca5a0dd
    Image:          nginx:1.14.2
    Image ID:       docker-pullable://nginx@sha256:f7988fb6c02e0ce69257d9bd9cf37ae20a60f1df7563c3a2a6abe24160306b8d
    Port:           80/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Fri, 09 Sep 2022 14:41:57 +0800
...
```

### Waiting

如果容器并不处在 Running 或 Terminated 状态之一，它就处在 Waiting 状态。

处于 Waiting 状态的容器仍在运行它完成启动所需要的操作：例如，从某个容器镜像仓库拉取容器镜像，或者向容器应用 Secret 数据等等。

当你使用 kubectl 来查询包含 Waiting 状态的容器的 Pod 时，你也会看到一个 **Reason** 字段，其中给出了容器处于等待状态的原因。

### Running

Running 状态表明容器正在执行状态并且没有问题发生。

如果配置了 postStart 回调，那么该回调已经执行且已完成(postStart 未执行完成, 是 Waiting 状态)。

如果你使用 kubectl 来查询包含 Running 状态的容器的 Pod 时，你也会看到关于容器进入 Running 状态的信息。

### Terminated

处于 Terminated 状态的容器已经开始执行后, 正常结束或者因为某些原因失败。

如果你使用 kubectl 来查询包含 Terminated 状态的容器的 Pod 时，你会看到:

- 容器进入此状态的原因
- 退出代码
- 容器执行期间的起止时间。

如果容器配置了 preStop 回调，则该回调会在容器进入 Terminated 状态之前执行。

```sh
$ kubectl describe pods hello--1-6pcq7
...
Containers:
  hello:
    Container ID:  docker://becd03bff385e3d1120a6f1b2a74996bdd54afc3b09b7f9e9d34971422ad8bbf
    Image:         busybox:1.28
    Image ID:      docker-pullable://busybox@sha256:141c253bc4c3fd0a201d32dc1f493bcf3fff003b6df416dea4f41046e0f37d47
    Port:          <none>
    Host Port:     <none>
    Command:
      sh
      -c
      echo "Hello, Kubernetes!" && sleep 10
    State:          Terminated
      Reason:       Completed
      Exit Code:    0
      Started:      Wed, 14 Sep 2022 03:07:44 +0800
      Finished:     Wed, 14 Sep 2022 03:07:54 +0800
...
```

## 容器重启策略

Pod 的 spec 中包含一个 **restartPolicy** 字段，其可能取值包括:

- Always（默认值），容器退出后将始终重启。
- OnFailure，容器以非 0 状态码退出时，容器将重启。
- Never，容器退出后将不再重启。

**注意:**

- 这里是容器重启策略, 不是 Pod 重启策略. Pod 是不能重启的.
- 存活探针, 启动探针等场景会重启. 正常退出时, 若使用 Always 策略是会重启的.

容器重启策略会影响 Pod 的状态（如果会重启，Pod 状态为 Running，如果不会重启，Pod 状态为 Succeeded 或 Failed）。

restartPolicy 适用于 Pod 中的所有容器，当 Pod 退出后，需要重启，会按指数回退方式计算重启延迟（10s、20s、40s）。

容器等待重启这段时间内：

- Pod 的 Status 为 Running。
- 容器的 State 为 Waiting，且 Reason 为 CrashLoopBackOff。

重启策略配置示意：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hello
spec:
  containers:
    - name: hello
      image: busybox
      command: ['sh', '-c', 'echo "Hello World!" && date && sleep 10']
  restartPolicy: OnFailure
```

**注意:**

- 上面的 yaml 配置的 pod, 会启动容器并运行 10s, 然后成功结束, 此时会触发重启.
- 在多次重启后, 可以看到 Pod 的 Status 为 CrashLoopBackOff, 这代表虽然容器了, 但是容器又运行结束了, 即便是成功结束. 因为重启会指数回退, 所以重启次数越多, 处在 CrashLoopBackOff 的时间就越久.

```sh
[root@VM-1-2-centos ~]# kubectl get pods
NAME                                READY   STATUS             RESTARTS        AGE
easyngx                             1/1     Running            0               23h
hello                               0/1     CrashLoopBackOff   8 (4m10s ago)   21m
hello--1-6pcq7                      0/1     Completed          0               21h
kubernetes-proxy-6564b76594-k7k4l   1/1     Running            0               5d11h
```

## Pod 状况

Pod 有一个 PodStatus 对象，其中包含一个 **PodConditions** 数组。Pod 可能通过, 也可能未通过其中的一些 Conditions 测试。

Kubelet 管理以下 PodCondition：

- PodScheduled：Pod 已经被调度到某节点；
- PodHasNetwork：Pod 沙箱被成功创建并且配置了网络（Alpha 特性，必须被显式启用）；
- ContainersReady：Pod 中所有容器都已就绪；
- Initialized：所有的 **Init 容器** 都已成功完成；
- Ready：Pod 可以为请求提供服务，并且应该被添加到对应服务的负载均衡池中。

字段名称 | 描述
-|-
type | Pod 状况的名称
status | 表明该状况是否适用，可能的取值有 "True", "False" 或 "Unknown"
lastProbeTime | 上次探测 Pod 状况时的时间戳
lastTransitionTime | Pod 上次从一种状态转换到另一种状态时的时间戳
reason | 机器可读的、驼峰编码（UpperCamelCase）的文字，表述上次状况变化的原因
message | 人类可读的消息，给出上次状态转换的详细信息

```sh
$ kubectl describe pod nginx-deployment-66b6c48dd5-bzphg
...
Conditions:
  Type              Status
  Initialized       True 
  Ready             True 
  ContainersReady   True 
  PodScheduled      True 
...
```

### Pod 就绪态(Pod readiness)

您的应用程序可以向 PodStatus: Pod readiness 注入额外的反馈或信号 。

要使用这一特性，可以设置 Pod spec 中的 readinessGates 列表，为 kubelet 提供一组额外的状况供其评估 Pod 就绪态时使用。

就绪态门控基于 Pod 的 status.conditions 字段的当前值来做决定。 如果 Kubernetes 无法在 status.conditions 字段中找到某状况，则该状况的 状态值默认为 "False"。

这里是一个例子：

```yaml
kind: Pod
...
spec:
  readinessGates:
    - conditionType: "www.example.com/feature-1"
status:
  conditions:
    - type: Ready                              # 内置的 Pod 状况
      status: "False"
      lastProbeTime: null
      lastTransitionTime: 2018-01-01T00:00:00Z
    - type: "www.example.com/feature-1"        # 额外的 Pod 状况
      status: "False"
      lastProbeTime: null
      lastTransitionTime: 2018-01-01T00:00:00Z
  containerStatuses:
    - containerID: docker://abcd...
      ready: true
...
```

### Status for Pod readiness

### Pod 网络就绪

## 容器探针

probe 是由 kubelet 对**容器**执行的定期诊断。

要执行诊断，kubelet 既可以在**容器内**执行代码，也可以发出一个网络请求。

**注意:**

- 这是容器探针, 是对容器的探测.

### 检查机制

机制 | 描述
-|-
exec | **在容器内**执行指定命令。如果命令退出时返回码为 0 则认为诊断成功。
grpc | 使用 gRPC 执行一个远程过程调用。 目标应该实现 gRPC 健康检查。 如果响应的状态是 "SERVING"，则认为诊断成功。<br>gRPC 探针是一个 alpha 特性，只有在你启用了 "GRPCContainerProbe" 特性门控时才能使用。
httpGet | 对容器的 IP 地址上指定端口和路径执行 HTTP GET 请求。如果响应的状态码大于等于 200 且小于 400，则诊断被认为是成功的。
tcpSocket | 对容器的 IP 地址上的指定端口执行 TCP 检查。如果端口打开，则诊断被认为是成功的。 如果远程系统（容器）在打开连接后立即将其关闭，这算作是健康的。

### 探测结果

每次探测都将获得以下三种结果之一：

探测结果 | 描述
-|-
Success（成功） | 容器通过了诊断。
Failure（失败） | 容器未通过诊断。
Unknown（未知） | 诊断失败，因此不会采取任何行动。

### 探测类型

针对运行中的容器，kubelet 可以选择是否执行以下三种探针，以及如何针对探测结果作出反应：

探针类型 | 名称 | 描述
-|-|-
livenessProbe | 存活探针 | 指示容器是否正在运行。如果存活态探测失败，则 kubelet 会杀死容器，并且容器将根据其重启策略决定未来。如果容器不提供存活探针，则默认状态为 Success。
readinessProbe | 就绪探针 | 指示容器是否准备好为请求提供服务。如果就绪态探测失败，端点控制器将从与 Pod 匹配的所有服务的端点列表中删除该 Pod 的 IP 地址。 初始延迟之前的就绪态的状态值默认为 Failure。 如果容器不提供就绪态探针，则默认状态为 Success。
startupProbe | 启动探针 | 指示容器中的应用是否已经启动。如果提供了启动探针，**则所有其他探针都会被禁用，直到此探针成功为止**。如果启动探测失败，kubelet 将杀死容器，而容器依其重启策略进行重启。如果容器没有提供启动探测，则默认状态为 Success。

探针配置示例:

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: liveness
  name: liveness-exec
spec:
  containers:
  - name: liveness
    image: k8s.gcr.io/busybox
    args:
    - /bin/sh
    - -c
    - touch /tmp/healthy; sleep 30; rm -f /tmp/healthy; sleep 600
    livenessProbe:
      exec:
        command:
        - cat
        - /tmp/healthy
      initialDelaySeconds: 5
      periodSeconds: 5
```


### 何时该使用存活态探针?

如果容器中的进程能够在遇到问题或不健康的情况下自行崩溃，则不一定需要存活态探针, kubelet 将根据 Pod 的 restartPolicy 自动执行修复操作。

如果你希望容器在探测失败时被杀死并重新启动，那么请指定一个存活态探针， 并指定 restartPolicy 为 "Always" 或 "OnFailure"。

### 何时该使用就绪态探针? 

- 仅在探测成功时才开始向 Pod 发送请求流量，请指定就绪态探针。
- 如果你希望容器能够自行进入维护状态，也可以指定一个就绪态探针。
- 如果你的应用程序对后端服务有严格的依赖性，你可以同时实现存活态和就绪态探针。当应用程序本身是健康的，存活态探针检测通过后，就绪态探针会额外检查每个所需的后端服务是否可用。 这可以帮助你避免将流量导向只能返回错误信息的 Pod。
- 如果你的容器需要在启动期间加载大型数据、配置文件或执行迁移，你可以使用启动探针。然而，如果你想区分已经失败的应用和仍在处理其启动数据的应用，你可能更倾向于使用就绪探针。

### 何时该使用启动探针？

对于所包含的容器需要较长时间才能启动就绪的 Pod 而言，启动探针是有用的。 你不再需要配置一个较长的存活态探测时间间隔，只需要设置另一个独立的配置选定， 对启动期间的容器执行探测，从而允许使用远远超出存活态时间间隔所允许的时长。

如果你的容器启动时间通常超出 initialDelaySeconds + failureThreshold × periodSeconds 总值，你应该设置一个启动探测，对存活态探针所使用的同一端点执行检查。 periodSeconds 的默认值是 10 秒。你应该将其 failureThreshold 设置得足够高， 以便容器有充足的时间完成启动，并且避免更改存活态探针所使用的默认值。 这一设置有助于减少死锁状况的发生。

## Pod 的终止

由于 Pod 所代表的是在集群中节点上运行的进程，当不再需要这些进程时允许其优雅地终止是很重要的。一般不应武断地使用 KILL 信号终止它们，否则这将导致这些进程没有机会完成清理操作。

设计的目标是令你能够请求删除进程，并且知道进程何时被终止，同时也能够确保删除操作终将完成。当你请求删除某个 Pod 时，集群会记录并跟踪 Pod 的优雅终止周期，而不是直接强制地杀死 Pod。在存在强制关闭设施的前提下，kubelet 会尝试优雅地终止 Pod。

通常情况下，Container Runtime 会发送一个 **TERM 信号**到**每个容器中的主进程**。很多 Container Runtime 时都能够注意到容器镜像中 STOPSIGNAL 的值，并发送该信号而不是 TERM。

一旦超出了优雅终止限期，容器运行时会向所有剩余进程发送 KILL 信号，之后 Pod 就会被从 API 服务器上移除。如果 kubelet 或者 Container Runtime 的管理服务在等待进程终止期间被重启，集群会从头开始重试，赋予 Pod 完整的体面终止限期。

下面是一个例子：

1. 你使用 kubectl 工具手动删除某个特定的 Pod，而该 Pod 的优雅终止限期是默认值（30 秒）。
1. API 服务器中的 Pod 对象被更新，记录优雅终止限期在内 Pod 的最终死期，超出所计算时间点则认为 Pod 已死（dead）。
   - 如果你使用 kubectl describe 来查验你正在删除的 Pod，该 Pod 会显示为 "Terminating" （正在终止）。
   - 在 Pod 运行所在的节点上：kubelet 一旦看到 Pod 被标记为正在终止（已经设置了体面终止限期），kubelet 即开始本地的 Pod 关闭过程。
   - 如果 Pod 中的容器之一定义了 preStop 回调， kubelet 开始在容器内运行该回调逻辑。如果超出体面终止限期时，preStop 回调逻辑仍在运行，kubelet 会请求给予该 Pod 的宽限期一次性增加 2 秒钟。
   - kubelet 接下来触发容器运行时发送 TERM 信号给每个容器中的进程 1。
   - 说明：
     - 如果 preStop 回调所需要的时间长于默认的体面终止限期，你必须修改 terminationGracePeriodSeconds 属性值来使其正常工作。
     - Pod 中的容器会在不同时刻收到 TERM 信号，接收顺序也是不确定的。 如果关闭的顺序很重要，可以考虑使用 preStop 回调逻辑来协调。
1. 与此同时，kubelet 启动优雅关闭逻辑，控制面会将 Pod 从对应的端点列表中移除，过滤条件是 Pod 被对应的服务以某选择算符选定。 ReplicaSets和其他工作负载资源不再将关闭进程中的 Pod 视为合法的、能够提供服务的副本。关闭动作很慢的 Pod 也无法继续处理请求数据，因为负载均衡器（例如服务代理）已经在终止宽限期开始的时候将其从端点列表中移除。
1. 超出终止宽限期限时，kubelet 会触发强制关闭过程。容器运行时会向 Pod 中所有容器内 仍在运行的进程发送 SIGKILL 信号。 kubelet 也会清理隐藏的 pause 容器，如果容器运行时使用了这种容器的话。
1. kubelet 触发强制从 API 服务器上删除 Pod 对象的逻辑，并将体面终止限期设置为 0 （这意味着马上删除）。
1. API 服务器删除 Pod 的 API 对象，从任何客户端都无法再看到该对象。

### 强制终止 Pod

默认情况下，所有的删除操作都会附有 30 秒钟的优雅关闭宽限期限。 kubectl delete 命令支持 `--grace-period=<seconds>` 选项，允许你重载默认值， 设定自己希望的期限值。

将宽限期限强制设置为 0 意味着立即从 API 服务器强制删除 Pod。 如果 Pod 仍然运行于某节点上，强制删除操作会触发 kubelet 立即执行清理操作。

**注意：** 

- 对于某些工作负载及其 Pod 而言，强制删除很可能会带来某种破坏。
- 你必须在设置 --grace-period=0 的同时额外设置 --force 参数才能发起强制删除请求。


执行强制删除操作时，API 服务器不再等待来自 kubelet 的、关于 Pod 已经在原来运行的节点上终止执行的确认消息。 API 服务器直接删除 Pod 对象，这样新的与之同名的 Pod 即可以被创建。 在节点侧，被设置为立即终止的 Pod 仍然会在被强行杀死之前获得一点点的宽限时间。

### 已终止 Pod 的垃圾收集

对于已失败的 Pod 而言，对应的 API 对象仍然会保留在集群的 API 服务器上，直到用户或者控制器进程显式地将其删除。

控制面组件会在 Pod 个数超出所配置的阈值 （根据 kube-controller-manager 的 terminated-pod-gc-threshold 设置）时删除已终止的 Pod（阶段值为 Succeeded 或 Failed）。 这一行为会避免随着时间演进不断创建和终止 Pod 而引起的资源泄露问题。
