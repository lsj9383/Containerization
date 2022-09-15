# DaemonSet

DaemonSet 确保：

- 全部（或者某些）节点上运行一个 Pod 的副本，
- 当有节点加入集群时， 也会为他们新增一个 Pod
- 当有节点从集群移除时，这些 Pod 也会被回收。

删除 DaemonSet 将会删除它创建的所有 Pod。

DaemonSet 的一些典型用法：

- 在每个节点上运行集群守护进程
- 在每个节点上运行日志收集守护进程
- 在每个节点上运行监控守护进程

有两种常见的搞法：

- 一种简单的用法是，一个覆盖所有节点的 DaemonSet 将用于每种类型的守护进程。
- 一个复杂的用法是，为同一种守护进程部署多个 DaemonSet；每个具有不同的标志，并且对不同硬件类型具有不同的内存、CPU 要求。

## 编写 DaemonSet Spec

你可以在 YAML 文件中描述 DaemonSet。 例如，下面的 daemonset.yaml 文件描述了一个运行 fluentd-elasticsearch Docker 镜像的 DaemonSet：

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-elasticsearch
  namespace: kube-system
  labels:
    k8s-app: fluentd-logging
spec:
  selector:
    matchLabels:
      name: fluentd-elasticsearch
  template:
    metadata:
      labels:
        name: fluentd-elasticsearch
    spec:
      tolerations:
      # 这些容忍度设置是为了让该守护进程集在控制平面节点上运行
      # 如果你不希望自己的控制平面节点运行 Pod，可以删除它们
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule
      - key: node-role.kubernetes.io/master
        operator: Exists
        effect: NoSchedule
      containers:
      - name: fluentd-elasticsearch
        image: quay.io/fluentd_elasticsearch/fluentd:v2.5.2
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
      terminationGracePeriodSeconds: 30
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
```

### 必需字段

和所有其他 Kubernetes 配置一样，DaemonSet 需要 apiVersion、kind 和 metadata 字段。

DaemonSet 对象的名称必须是一个合法的 DNS 子域名。DaemonSet 也需要一个 .spec 配置段。

### Pod 模板

`.spec` 中唯一必需的字段是 `.spec.template`，是一个 Pod 模板。

除了 Pod 必需字段外，在 DaemonSet 中的 Pod 模板必须指定合理的标签（查看 Pod 选择算符）。

在 DaemonSet 中的 Pod 模板必须具有一个值为 Always 的 RestartPolicy。 当该值未指定时，默认是 Always。

### Pod 选择算符

`.spec.selector` 字段表示 Pod 选择算符，它与 Job 的 .spec.selector 的作用是相同的。

你必须指定与 .spec.template 的标签匹配的 Pod 选择算符。此外，一旦创建了 DaemonSet，它的 .spec.selector 就不能修改。

> 修改 Pod 选择算符可能导致 Pod 意外悬浮，并且这对用户来说是费解的。

**注意：**

- .spec.selector 必须与 .spec.template.metadata.labels 相匹配。如果配置中这两个字段不匹配，则会被 API 拒绝。

### 仅在某些节点上运行 Pod

如果指定了 `.spec.template.spec.nodeSelector`，DaemonSet 控制器将在能够与 Node 选择算符匹配的节点上创建 Pod。

类似这种情况，可以指定 `.spec.template.spec.affinity`，之后 DaemonSet 控制器将在能够与节点亲和性匹配的节点上创建 Pod。

如果根本就没有指定，则 DaemonSet Controller 将在所有节点上创建 Pod。

## Daemon Pods 是如何被调度的

### 通过默认调度器调度

DaemonSet 确保所有符合条件的**节点**都运行该 Pod 的一个副本。

此外，`node.kubernetes.io/unschedulable:NoSchedule` 会自动添加到 DaemonsSet 的 Pod 中，这会忽略掉 `unschedulable` 节点。

### 污点和容忍度 

尽管 Daemon Pods 遵循污点和容忍度规则，根据相关特性，控制器会自动将以下容忍度添加到 DaemonSet Pod：

容忍度键名 | 效果 | 版本 | 描述
-|-|-|-
node.kubernetes.io/not-ready | NoExecute | 1.13+ | 当出现类似网络断开的情况导致节点问题时，DaemonSet Pod 不会被逐出。
node.kubernetes.io/unreachable | NoExecute | 1.13+ | 当出现类似于网络断开的情况导致节点问题时，DaemonSet Pod 不会被逐出。
node.kubernetes.io/disk-pressure | NoSchedule | 1.8+ | DaemonSet Pod 被默认调度器调度时能够容忍磁盘压力属性。
node.kubernetes.io/memory-pressure | NoSchedule | 1.8+ | DaemonSet Pod 被默认调度器调度时能够容忍内存压力属性。
node.kubernetes.io/unschedulable | NoSchedule | 1.12+ | DaemonSet Pod 能够容忍默认调度器所设置的 unschedulable 属性.
node.kubernetes.io/network-unavailable | NoSchedule | 1.12+ | DaemonSet 在使用宿主网络时，能够容忍默认调度器所设置的 network-unavailable 属性。

## 与 Daemon Pods 通信

与 DaemonSet 中的 Pod 进行通信的几种可能模式如下：

- 推送（Push）：配置 DaemonSet 中的 Pod，将更新发送到另一个服务，例如统计数据库。 这些服务没有客户端。
- NodeIP 和已知端口：DaemonSet 中的 Pod 可以使用 hostPort，从而可以通过节点 IP 访问到 Pod。客户端能通过某种方法获取节点 IP 列表，并且基于此也可以获取到相应的端口。
- DNS：创建具有相同 Pod 选择算符的无头服务，通过使用 endpoints 资源或从 DNS 中检索到多个 A 记录来发现 DaemonSet。
- Service：创建具有相同 Pod 选择算符的服务，并使用该服务随机访问到某个节点上的守护进程（没有办法访问到特定节点）。

## 更新 DaemonSet

如果节点的标签被修改，DaemonSet 将立刻向新匹配上的节点添加 Pod，同时删除不匹配的节点上的 Pod。

你可以修改 DaemonSet 创建的 Pod。不过并非 Pod 的所有字段都可更新。下次当某节点（即使具有相同的名称）被创建时，DaemonSet 控制器还会使用最初的模板。

你可以删除一个 DaemonSet。如果使用 kubectl 并指定 --cascade=orphan 选项，则 Pod 将被保留在节点上（这会变成裸 Pod）。接下来如果创建使用相同选择算符的新 DaemonSet， 新的 DaemonSet 会收养已有的 Pod。 如果有 Pod 需要被替换，DaemonSet 会根据其 updateStrategy 来替换。

你可以对 DaemonSet 执行滚动更新操作。

## DaemonSet 的替代方案

替代方案 | 描述
-|-
init 脚本 | 直接在节点上启动守护进程（例如使用 init、upstartd 或 systemd）的做法当然是可行的。 不过，基于 DaemonSet 来运行这些进程有如下一些好处：<br>1) 像所运行的其他应用一样，DaemonSet 具备为守护进程提供监控和日志管理的能力。<br>2) 为守护进程和应用所使用的配置语言和工具（如 Pod 模板、kubectl）是相同的。<br>3) 在资源受限的容器中运行守护进程能够增加守护进程和应用容器的隔离性。
裸 Pod | 直接创建 Pod并指定其运行在特定的节点上也是可以的。 然而，DaemonSet 能够替换由于任何原因（例如节点失败、例行节点维护、内核升级） 而被删除或终止的 Pod。 
静态 Pod | 通过在一个指定的、受 kubelet 监视的目录下编写文件来创建 Pod 也是可行的。 这类 Pod 被称为静态 Pod。<br>不像 DaemonSet，静态 Pod 不受 kubectl 和其它 Kubernetes API 客户端管理。<br>静态 Pod 不依赖于 API 服务器，这使得它们在启动引导新集群的情况下非常有用。<br>此外，静态 Pod 在将来可能会被废弃。
Deployments | 建议为无状态的服务使用 Deployments。<br>当需要 Pod 副本总是运行在全部或特定主机上，并且当该 DaemonSet 提供了节点级别的功能。<br>例如，网络插件通常包含一个以 DaemonSet 运行的组件。这个 DaemonSet 组件确保它所在的节点的集群网络正常工作。