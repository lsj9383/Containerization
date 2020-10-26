# Pods

## 概述

一个 Pod 是一组容器的集合，是 kube 中创建、管理和调度的最小单元。一个 Pod 中的容器共享存储、网络以及如何运行容器的声明。

Pod 除了应用容器，也包括 Pod 启动期间运行的 Init 容器。

就 Docker 概念的术语而言，Pod 类似于共享命名空间和文件系统卷的一组 Docker 容器。

### 使用 Pods

通常而言，即便是单 Pods 应用，也不需要我们直接创建 Pods。我们通过工作负载资源（Workload Resource）创建 Pods，例如：Deployment 或者 Job。如果 Pods 需要记录状态，考虑使用 StatefutSet。

Pods 在 kube 集群中通常有两种用法：

- Pods 运行单个容器。每个 Pods 中一个容器是 kube 中最常见到的用法， 此时 Pods 是容器的包装，kube 通过管理 Pods 来管理容器。
- Pods 运行多个容器。Pods 封装了应用程序，应用程序由紧密联系且需要共享资源的容器组成。

无论是哪种方式，都希望 Pods 中运行了一个应用实例（可能是单容器，也可能是多容器），可以通过部署多个 Pods 以水平的扩展资源。在 kube 中，这种方式被称为 *副本（replication）*。通常使用 Workload Resource 或 Controller 来创建和管理一组 Pods 的副本。

### Pods 管理多容器

Pods 可以支持多个容器之间协同工作的服务，Pods 中的容器会位于同一个节点上，并在同一位置进行调度。这些容器可以共享资源和依赖关系，彼此通信，并协调何时以及如何终止它们。

某些 Pods 具有 init container，可以在 Pods 的应用容器前运行。

### Working with Pods

当创建 Pod 后，该 Pod 会调度在集群上的某个节点，并且节点会一直保留该 Pod，直到 Pod 结束执行，Pod 对象被删除。如果节点缺少资源，或节点发生故障，Pod 将会被驱逐。

**注意：**

- Pods 中的容器 restart 和 Pods 的 restart 不应该混淆，Pods 是运行环境，一个 Pods 将会一直存在，直到被删除。

**Pods and controllers**

可以使用 Workload Resource 进行 Pods 的创建和管理，Resource Controller 会处理 Pods 故障时的复制和恢复。例如，如果某个节点发生故障，则控制器会注意到该节点上的 Pod 已停止工作，并创建了一个替换Pod。 调度程序将替换Pod放置在健康的节点上。

**Pod templates**

Workload Resource 的控制器会从 Pod Template 中创建 Pods，并且 Pod Template 是 Workload Resource 期望达到状态的一部分。

这是一个 Job 的 template：

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: hello
spec:
  template:
    # This is the pod template
    spec:
      containers:
      - name: hello
        image: busybox
        command: ['sh', '-c', 'echo "Hello, Kubernetes!" && sleep 3600']
      restartPolicy: OnFailure
    # The pod template ends here
```

修改一个 pod template 都不会对已经存在的 Pod 有影响，kube 会根据新的 template 创建一个新的 Pod。例如 Deployment 更新 Pods 的 template，会由 Deployment 生成新的 Pod，再删除老的 Pod，以达到新 Pods 代替老 Pods 的目的。

### Static Pods

Static Pods 由指定节点的 kubelet 直接控制，API Server 无法观察到 Static Pods，因此也不能被大多数的 Controller 进行控制。对于 Static Pods 而言，kubelet 直接监控着它们。

这是一个 Static Pods：

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
```

## Pod 的生命周期

Pod 的生命周期是被定义好的，起始于 Pending，当有至少一个容器正常启动时，则进入 Running 状态，最后根据 Pod 的容器是否存在失败，进入 Succeeded 或者 Failed 状态。

需要注意，Pod 在生命周期中只会被调度一次，一旦被调度到节点上，Pod 会在该节点一直执行，直到 Pod 停止、或别人为停止、甚至被系统驱逐。因此 Pod 是不会重启的，遇到节点异常，可能导致 Pod 被驱逐，启动一个新的 Pod。

### Pod 阶段

Pod 的 status 字段是一个 PodStatus 对象，其中包含表示当前阶段的 phase 字段。

phase 仅包含以下值：

- Pending，Pod 已被 kube 接受，但容器没有创建和运行。此阶段包括 Pod 调度时间以及从网络下载镜像等时间。
- Running，Pod 已经调度并绑定到节点上，并且容器都创建完成，且至少一个容器在运行、启动或重启状态。
- Succeeded，所有容器都成功终止，并且不会再重启。
- Failed，所有容器都终止，并且至少有一个容器是失败退出的（即退出码不为 0）。
- Unknown，因为某些原因无法获取到 Pod 的状态，表示 Pod 与主机通信失败。

**注意：**

- 一定要注意，上面的 status 是 `kubectl describe pods <pod-name>` 中返回的 Status 字段，而不是 `kubectl get pods` 时返回的 STATUS 列。

### 容器状态

容器状态可以通过 `kubectl describe pods <pod-name>` 拿到 Pods 中所有容器的状态（State 字段拿到当前状态，Last State 字段拿到上一次的状态）。

```sh
$ kubectl describe pods hello
...
Containers:
  hello:
    Container ID:  docker://03d7c6a98f0344eb98b4af4a4ed910697a5aa7d6716db0a9321775d8c40846ee
    Image:         busybox
    Image ID:      docker-pullable://busybox@sha256:a9286defaba7b3a519d585ba0e37d0b2cbee74ebfe590960b0b1d6a5e97d1e1d
    Port:          <none>
    Host Port:     <none>
    Command:
      sh
      -c
      echo "Hello World!" && date && sleep 10
    State:          Waiting
      Reason:       CrashLoopBackOff
    Last State:     Terminated
      Reason:       Completed
      Exit Code:    0
      Started:      Mon, 26 Oct 2020 23:23:21 +0800
      Finished:     Mon, 26 Oct 2020 23:23:31 +0800
    Ready:          False
    Restart Count:  4
    Environment:    <none>
    Mounts:
...
```

- Waiting，如果容器不处于 Running 或者 Terminated 状态之一，则处于 Waiting。Reason 字段会描述处于 Waiting 的原因，例如拉取镜像、回退重启等。
- Running，容器正在执行的状态。
- Terminated，容器正常和失败退出都会进入 Terminated 状态，并且 Reason 字段会告诉进入该状态的原因。例如 Completed，或者是 Error（Exit Code 不为 0）。

### 容器重启策略

Pod 的 spec 中包含一个 restartPolicy 字段，其可能取值包括 Always、OnFailure 和 Never。

- Always（默认值），容器退出后将始终重启。
- OnFailure，容器以非 0 状态码退出时，容器将重启。
- Never，容器退出后将不再重启。

容器重启策略会影响 Pod 的状态（如果会重启，Pod 状态为 Running，如果不会重启，Pod 状态为 Succeeded 或 Failed）。

restartPolicy 适用于 Pod 中的所有容器，当 Pod 退出后，需要重启，会按指数回退方式计算重启延迟（10s、20s、40s），延迟最长 5min。容器正常运行十分钟后，会重置重启研制。

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

### Pod 状况

可以通过 `kubectl describe pods <pod-name>` 拿到 Pod 的状况，即响应的 Conditions 字段：

```sh
$ kubectl describe pods hello
...
Conditions:
  Type              Status
  Initialized       True
  Ready             False
  ContainersReady   False
  PodScheduled      True
...
```

- PodScheduled：Pod 已经被调度到某节点；
- ContainersReady：Pod 中所有容器都已就绪；
- Initialized：所有的 Init 容器 都已成功启动；
- Ready：Pod 可以为请求提供服务，并且应该被添加到对应服务的负载均衡池中。

### 容器探针

探针提供了 kubelet 对容器进行的定期诊断的能力，为了进行诊断，kubelet 会调用容器提供的 handler，目前有三种类型的 handler：

- ExecAction，在容器中执行指定命令，如果命令退出时返回码为 0 则认为 Success，否则 Failure。
- TCPSocketAction，对容器的 IP 地址上的指定端口执行 TCP 检查。如果端口打开，则诊断被认为是成功的。
- HTTPGetAction，对容器的 IP 地址上指定端口和路径执行 HTTP Get 请求。如果响应的状态码大于等于 200 且小于 400，则诊断被认为是成功的。

每次探测都将获得以下三种结果之一：

- Success（成功）：容器通过了诊断。
- Failure（失败）：容器未通过诊断。
- Unknown（未知）：诊断失败，因此不会采取任何行动。

针对运行的容器，kube 提供三种探针，这些探针可以使用上面提到的 handler 进行探测。

- livenessProbe，探测容器中的应用是否处于运行，如果探测失败，kube 会立即杀掉容器，并按照重启策略对容器重启。
  - 如果你希望容器在探测失败时被杀死并重新启动，那么请指定一个存活态探针，并指定restartPolicy 为 "Always" 或 "OnFailure"。
- readinessProbe，指示容器是否准备好为请求提供服务。如果就绪态探测失败， 端点控制器将从与 Pod 匹配的所有服务的端点列表中删除该 Pod 的 IP 地址。 初始延迟之前的就绪态的状态值默认为 Failure。
  - 如果要仅在探测成功时才开始向 Pod 发送请求流量，请指定就绪态探针。
  - 如果你的容器需要加载大规模的数据、配置文件或者在启动期间执行迁移操作，可以添加一个就绪态探针。
- startupProbe，探测容器中的应用是否成功启动，如果提供了启动探针，则所有其他探针都会被禁用，直到此探针成功为止。如果启动探测失败，kubelet 将杀死容器，而容器依其 重启策略进行重启。
  - 对于所包含的容器需要较长时间才能启动就绪的 Pod 而言，启动探针是有用的。

### Pod Hook

- PostStart：这个钩子在容器创建后立即执行。但是，并不能保证钩子将在容器 ENTRYPOINT 之前运行，因为没有参数传递给处理程序。
  - 主要用于资源部署、环境准备等。
  - 需要注意的是如果钩子花费太长时间以至于不能运行或者挂起， 容器将不能达到 running 状态。
- PreStop：这个钩子在容器终止之前立即被调用。它是阻塞的，它必须在删除容器的调用发出之前完成。
  - 主要用于优雅关闭应用程序、通知其他系统等。
  - 如果钩子在执行期间挂起， Pod 阶段将停留在 running。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hook-test
spec:
  containers:
  - name: hook-test
    image: busybox
    command: ['sh', '-c', 'echo "Hello World!" && date && sleep 10']
    lifecycle:
      postStart:
        exec:
          command: ["/bin/sh", "-c", 'echo "****** postStart ******"']
      preStop:
        exec:
          command: ["/bin/sh", "-c", 'echo "====== preStop ======"']
```

### Pod 终止

容器运行时会向 Pod 中的每个容器的主进程发送 TERM 信号，要求以优雅的方式关闭进程和容器，若超出了时间限制后仍未关闭的，则会发送 KILL 信号强制关闭。

若 Pod 中的容器配置了 preStop，会在发送 TERM 信号前先进行 preStop 回调。

### 失效 Pod 的回收

对于已失败的 Pod 而言，对应的 API 对象仍然会保留在集群的 API 服务器上，直到用户或者控制器进程显式地将其删除。

kube 会在 Pod 个数超出所配置的阈值（根据 kube-controller-manager 的 terminated-pod-gc-threshold 设置），删除已终止（Succeeded 或 Failed）的 Pod。这一行为会避免随着时间演进不断创建和终止 Pod 而引起的资源泄露问题。

## Init 容器

Init 容器比较特殊，会在 Pod 的应用容器启动前启动，主要是运行一些工具。

Pod 可以有一个或多个 Init 容器，Init 容器与普通应用容器很类似，除了这两点：

- 总是运行到完成。
- Init 容器是串行执行的，只有当一个完成后才会运行下一个。

当所有的 Init 容器运行完成时， Kubernetes 才会为 Pod 初始化应用容器并像平常一样运行。

如果 Pod 的 Init 容器失败，则 Kubernetes 会重启 Init 容器，直到 Init 容器成功为止。如果 Pod 对应的 restartPolicy 值为 Never，并且 Init 容器返回失败，则 kube 视为整个 Pod 失败。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: init-test
  labels:
    app: myapp
spec:
  containers:
  - name: init-test
    image: busybox
    command: ['sh', '-c', 'echo "Hello World!" && date && sleep 10']
  initContainers:
  - name: init-1
    image: busybox
    command: ['sh', '-c', 'echo "======= 1 ========" && date && sleep 10']
  - name: init-2
    image: busybox
    command: ['sh', '-c', 'echo "======= 2 ========" && date && sleep 10']
```

如果需要查看 init container 的输出，可以使用如下命令：

```sh
kubectl logs <pod-name> -c <container-name>

kubectl logs init-test -c init-1
kubectl logs init-test -c init-2
```

### 具体行为

在 Pod 启动过程中，每个 Init 容器在网络和数据卷初始化之后会按顺序启动。

每个 Init 容器成功退出后才会启动下一个 Init 容器。 如果它们因为容器运行时的原因无法启动，或以错误状态退出，它会根据 Pod 的 restartPolicy 策略进行重试。 

**注意：**

- 如果 Pod 的 restartPolicy 设置为 "Always"，Init 容器失败时会使用 restartPolicy 的 "OnFailure" 策略。

对 Init 容器规约的修改仅限于容器的 image 字段。 更改 Init 容器的 image 字段，等同于重启该 Pod。

因为 Init 容器可能会被重启、重试或者重新执行，所以 Init 容器的代码应该是幂等的。 特别地，基于 emptyDirs 写文件的代码，应该对输出文件可能已经存在做好准备。
