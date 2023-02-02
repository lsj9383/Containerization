# Kubernetes 概览

Kubernetes 是一个可移植、可扩展的开源平台，用于管理容器化的工作负载和服务，可促进声明式配置和自动化。

Kubernetes 拥有一个庞大且快速增长的生态，其服务、支持和工具的使用范围相当广泛。

Kubernetes 和 k8s 命名的由来：

> The name Kubernetes originates from Greek, meaning helmsman or pilot. K8s as an abbreviation results from counting the eight letters between the "K" and the "s". Google open-sourced the Kubernetes project in 2014. Kubernetes combines over 15 years of Google's experience running production workloads at scale with best-of-breed ideas and practices from the community.

## 时光回溯

![image](https://user-images.githubusercontent.com/13912637/216247632-564900ca-4d45-488f-a071-f14cbd8eb944.png)

## 为什么需要 k8s

Kubernetes 为你提供：

能力 | 描述
-|-
服务发现和负载均衡 | k8s 使用 DNS 或 IP 地址暴露容器，并可以自动进行负载均衡。
存储编排 | k8s 允许你自动挂载你选择的存储系统，例如本地存储、公共云提供商等。
自动部署和回滚 | 用户可以告诉 k8s 部署容器所需要的状态，它可以以受控的速率将实际状态更改为期望状态。
自动完成装箱计算 | 用户告诉 k8s 每个容器需要多少 CPU 和内存 (RAM)。 Kubernetes 可以将这些容器按实际情况调度到你的节点上，以最佳方式利用你的资源。
自我修复 | k8s 将重新启动失败的容器、替换容器、杀死不响应用户定义的运行状况检查的容器。
密钥与配置管理 | k8s 允许你存储和管理敏感信息，例如密码、OAuth 令牌和 ssh 密钥。

## k8s 不是什么
