# Concepts

概念部分帮助你了解 k8s 系统的各个部分，以及用于表示集群的相关抽象概念（例如 Pods、Deployment 等），并帮助你更深入地理解 k8s 是如何工作的。

章节 | 描述
-|-
[概述](overview/readme.md) | k8s 是一个可移植、可扩展的开源平台，用于管理容器化的工作负载和服务，方便进行声明式配置和自动化。<br>k8s 拥有一个庞大且快速增长的生态系统，其服务、支持和工具的使用范围广泛。
[Kubernetes 架构](arch/readme.md) | k8s 背后的架构概念。
[容器](container/readme.md) | 打包应用及其运行依赖环境的技术，例如 Docker 等。
Kubernetes 中的 Windows | -
[工作负载](workloads/readme.md) | 理解 Pods，这是 k8s 中可部署的最小对象，同时 k8s 还有辅助 Pods 运行的高层抽象对象（例如 Deployment）。
[服务、负载均衡和联网](service-lb-network/readme.md) | k8s 网络背后的概念和资源。
[存储](storage/readme.md) | 为集群中的 Pods 提供长期和临时存储的方法。
[配置](config/readme.md) | k8s 为配置 Pods 提供的资源。
安全 | 确保云原生工作负载安全的一组概念。
[策略](policies/readme.md) | 可配置的、可应用到一组资源的策略。
[调度、抢占和驱逐](scheduling-preemption-and-eviction/readme.md) | 在 k8s 中，调度 (scheduling) 指的是确保 Pod 匹配到合适的节点， 以便 kubelet 能够运行它们。<br>抢占 (Preemption) 指的是终止低优先级的 Pod 以便高优先级的 Pod 可以调度运行的过程。<br>驱逐 (Eviction) 是在资源匮乏的节点上，主动让一个或多个 Pod 失效的过程。
集群管理 | 关于创建和管理 k8s 集群的底层细节。
扩展 Kubernetes | 改变你的 k8s 集群的行为的若干方法。
