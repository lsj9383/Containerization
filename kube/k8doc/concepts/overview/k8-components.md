# Kubernetes 组件

[TOC]

## 概览

k8s 集群是由一组被称作`节点（node）`的机器组成，这些节点上会运行由 k8s 所管理的容器化应用。 且每个集群至少有一个工作节点。

工作节点会托管所谓的 `Pods`，而 Pod 就是作为应用负载的组件。

`控制平面(control plane)` 管理集群中的工作节点和 Pods，为集群提供故障转移和高可用性。这些控制平面一般跨多主机运行，而集群也会跨多个节点运行。

**注意：**

- 节点，即 k8s 中的工作机器（Worker Machine）
- 控制平面，是容器编排层，暴露公开的 API 和接口，以定义、部署和管理容器的生命周期。

这是一个完整且正常工作的 Kubernetes 集群所需的各种组件：

![](assets/components-of-kubernetes.svg)

## 控制平面组件（Control Plane Components）

控制平面组件会为集群做出全局决策，比如：

- 资源的调度
- 检测和响应集群事件（如当不满足部署的 replicas 字段时，要启动新的 pod）

控制平面组件可以在集群中的任何节点上运行。然而，为了简单起见，设置脚本通常会在同一个计算机上启动所有控制平面组件， 并且不会在此计算机上运行用户容器。

### kube-apiserver

kube-apiserver，是 Kubernetes 控制平面的组件，该组件负责公开了 Kubernetes API，负责处理接受请求的工作。

简单的说来：kube-apiserver 是 Kubernetes 控制平面的前端（这里前端指的是提供一系列接口，并非一系列 UI）。

### etcd

etcd 是兼顾一致性与高可用性的键值数据库，可以作为保存 Kubernetes 所有集群数据的后台数据库。

### kube-scheduler

kube-scheduler 负责：监视新创建的但未分配运行节点的 Pods， 并选择合适的节点来让 Pod 在上面运行。

调度决策考虑的因素包括：

- Pods 的资源需求
- 软硬件及策略约束
- 亲和性及反亲和性规范
- 数据位置
- 工作负载间的干扰
- Pods 生命周期的截止日期

### kube-controller-manager

kube-controller-manager 负责运行控制器进程。

这里存在多种功能的控制器：

- 节点控制器（Node Controller）：负责在节点出现故障时进行通知和响应
- 任务控制器（Job Controller）：监测代表一次性任务的 Job 对象，然后创建 Pods 来运行这些任务直至完成
- 端点控制器（Endpoints Controller）：填充端点（Endpoints）对象（即加入 Service 与 Pod）
- 服务帐户和令牌控制器（Service Account & Token Controllers）：为新的命名空间创建默认帐户和 API 访问令牌

从逻辑上讲，每个控制器都是一个单独的进程，但是为了降低复杂性，所以在物理部署上，它们都被编译到同一个可执行文件，并在同一个进程中运行。

### cloud-controller-manager

一个 Kubernetes 控制平面组件，嵌入了特定于云平台的控制逻辑。

## Node 组件

Node 组件会在每个节点上运行，负责维护运行的 Pod 并提供 Kubernetes 运行环境。

### kubelet

kubelet 会在集群中每个节点（node）上运行。

kubelet 接收一组通过各类机制提供给它的 PodSpecs， 确保这些 PodSpecs 中描述的容器处于运行状态且健康。

**注意：**

- kubelet 不会管理不是由 Kubernetes 创建的容器。

### kube-proxy

kube-proxy 是集群中每个节点（node）所上运行的网络代理，实现 Kubernetes 服务（Service） 概念的一部分。

### 容器运行时（Container Runtime）

Container runtime 是负责运行容器的软件。

Kubernetes 支持许多容器运行环境，例如：

- Docker
- containerd
- CRI-O
- 符合 Kubernetes CRI (容器运行环境接口) 的其他任何实现

## 插件（Addons）

插件使用 Kubernetes 资源（DaemonSet、 Deployment 等）实现集群特性。

因为这些插件提供集群级别的功能，所以插件中命名空间域的资源属于 kube-system 命名空间。

### DNS

尽管其他插件都并非严格意义上的必需组件，但几乎所有 Kubernetes 集群都应该有集群 DNS。

集群 DNS 是一个 DNS 服务器，它为 Kubernetes 服务提供 DNS 记录。

Kubernetes 启动的容器自动将此 DNS 服务器包含在其 DNS 搜索列表中。

### Web 界面（仪表盘，Dashboard）

Dashboard 是 Kubernetes 集群的通用的、基于 Web 的用户界面。

它使用户可以管理集群中运行的应用程序以及集群本身，并进行故障排除。

### 容器资源监控

容器资源监控，将关于容器的一些常见的时间序列度量值保存到一个集中的数据库中， 并提供浏览这些数据的界面。

### 集群层面日志

集群层面日志机制负责将容器的日志数据保存到一个集中的日志存储中， 这种集中日志存储提供搜索和浏览接口。
