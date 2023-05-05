# Kubernetes 源码解析

[TOC]

## 概述

Kubernetes 系统用于管理分布式节点集群中的微服务或容器化应用程序，并且其提供了零停机时间部署、自动回滚、缩放和容器的自愈（其中包括自动配置、自动重启、自动复制的高弹性基础设施，以及容器的自动缩放等）等功能。

![](assets/1.png)

Kubernetes 系统架构遵循客户端/服务端架构，系统架构分为 Master 和 Node 两部分，Master 作为服务端，Node 作为客户端。

Master 服务端（主控节点）主要负责管理和控制整个 Kubernetes 集群，对集群做出全局性决策，相当于整个集群的“大脑”。集群所执行的所有控制命令都由 Master 服务端接收并处理：

- kube-apiserver 组件：集群的 HTTP REST API 接口，是集群控制的入口。
- kube-controller-manager 组件：集群中所有资源对象的自动化控制中心。
- kube-scheduler 组件：集群中Pod资源对象的调度服务。

Node 客户端（工作节点）是 Kubernetes 集群中的工作节点，Node 节点上的工作由 Master 服务端进行分配，比如当某个 Node 节点宕机时，Master 节点会将其上面的工作转移到其他 Node 节点上。包含的组件：

- kubelet 组件：负责管理节点上容器的创建、删除、启停等任务，与 Master 节点进行通信。
- kube-proxy 组件：负责 Kubernetes 服务的通信及负载均衡服务。
- container 组件：负责容器的基础管理服务，接收 kubelet 组件的指令。


**注意：**

- Kubernetes 系统具有多个 Master 服务端，可以实现高可用。
- 在默认的情况下，一个 Master 服务端即可完成所有工作。

## 各个组件功能


