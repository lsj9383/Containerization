# CGroup V2

在 Linux 上，控制组（一组具有可选资源隔离、审计和限制的 Linux 进程，即 cgroup 机制的进程）约束分配给进程的资源。

kubelet 和底层容器运行时都需要对接 cgroup 来强制执行为 Pod 和容器管理资源，这包括：

- 为容器化工作负载配置 CPU/内存请求和限制。

Linux 中有两个 cgroup 版本：

- cgroup v1
- cgroup v2

其中，cgroup v2 是新一代的 cgroup API。

## 什么是 cgroup v2？

特性状态： Kubernetes v1.25 [stable]

cgroup v2 是 Linux cgroup API 的下一个版本。cgroup v2 提供了一个具有增强资源管理能力的统一控制系统。

cgroup v2 对 cgroup v1 进行了多项改进，例如：

- API 中单个统一的层次结构设计
- 更安全的子树委派给容器
- 更新的功能特性， 例如压力阻塞信息（Pressure Stall Information，PSI）
- 跨多个资源的增强资源分配管理和隔离
  - 统一核算不同类型的内存分配（网络内存、内核内存等）
  - 考虑非即时资源变化，例如页面缓存回写

一些 Kubernetes 特性专门使用 cgroup v2 来增强资源管理和隔离。 例如，MemoryQoS 特性改进了内存 QoS 并依赖于 cgroup v2 原语。

## 使用 cgroup v2

使用 cgroup v2 的推荐方法是使用一个**默认启用 cgroup v2 的 Linux 发行版**。

要检查你的发行版是否使用 cgroup v2，请参阅识别 Linux 节点上的 cgroup 版本。

### 要求

cgroup v2 具有以下要求：

- 操作系统发行版启用 cgroup v2
- Linux 内核为 5.8 或更高版本
- 容器运行时支持 cgroup v2。例如：
  - containerd v1.4 和更高版本
  - cri-o v1.20 和更高版本
- kubelet 和容器运行时被配置为使用 systemd cgroup 驱动

### Linux 发行版 cgroup v2 支持

有关使用 cgroup v2 的 Linux 发行版的列表， 请参阅 cgroup v2 文档。

- Container-Optimized OS（从 M97 开始）
- Ubuntu（从 21.10 开始，推荐 22.04+）
- Debian GNU/Linux（从 Debian 11 Bullseye 开始）
- Fedora（从 31 开始）
- Arch Linux（从 2021 年 4 月开始）
- RHEL 和类似 RHEL 的发行版（从 9 开始）

要检查你的发行版是否使用 cgroup v2， 请参阅你的发行版文档或遵循识别 Linux 节点上的 cgroup 版本中的指示说明。

### 迁移到 cgroup v2

要迁移到 cgroup v2，需确保满足要求，然后升级到一个默认启用 cgroup v2 的内核版本。

kubelet 能够自动检测操作系统是否运行在 cgroup v2 上并相应调整其操作，无需额外配置。

切换到 cgroup v2 时，用户体验应没有任何明显差异，除非用户直接在节点上或从容器内访问 cgroup 文件系统。

cgroup 版本取决于正在使用的 Linux 发行版和操作系统上配置的默认 cgroup 版本。

要检查你的发行版使用的是哪个 cgroup 版本，请在该节点上运行命令：

```sh
$ stat -fc %T /sys/fs/cgroup/
```

- 对于 cgroup v2，输出为 cgroup2fs。
- 对于 cgroup v1，输出为 tmpfs。