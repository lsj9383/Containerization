# Kubernetes API

## 子章节

- [Kubernetes API 概念](kube-api-concetps.md)
- [服务端应用（Server Side Apply）]()
- [Kubernetes 弃用策略]()
- [已弃用 API 的迁移指南]()
- [Kubernetes API 健康端点]()

## 概述

REST API 是 Kubernetes 的基础，组件之间的所有操作和通信以及外部用户命令都是 API 服务器处理的 REST API 调用。

Kubernetes 平台中的所有内容都被视为 API 对象，请参考 [API](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.26/)。

## TKE 的 API Server 访问办法

```sh
# 先进入有 kubelet-kubeconfig 的目录
$ cd /etc/kubernetes

# 提取配置文件中的客户端证书
$ cat  ./kubelet-kubeconfig |grep client-certificate-data | awk -F ' ' '{print $2}' |base64 -d > client-cert.pem

# 提取客户端私钥
$ cat  ./kubelet-kubeconfig |grep client-key-data | awk -F ' ' '{print $2}' |base64 -d > client-key.pem

# 提取 Server
$ APISERVER=`cat  ./kubelet-kubeconfig |grep server | awk -F ' ' '{print $2}'`
$ echo $APISERVER
https://10.0.1.15

# 测试请求（因为 APISERVER 是 https 的，但是没有权威证书，这里用 -k 忽略）
$ curl --cert client-cert.pem --key client-key.pem -k $APISERVER/api/
```

## API 版本控制

API 版本和 Kubernetes 版本是间接相关的，关于 Kubernetes 版本可以参考 [Kubernetes Release Versioning](https://github.com/kubernetes/sig-release/blob/master/release-engineering/versioning.md)。

不同的 API 版本代表着不同的稳定性和支持级别，主要有这几个级别：

Alpha：

- 版本名称包含 alpha（例如：v1alpha1）。
- 内置的 Alpha API 版本默认被禁用且必须在 kube-apiserver 配置中显式启用才能使用。
- 软件**可能会有 Bug**。启用某个特性可能会暴露出 Bug。
- 对某个 Alpha API 特性的支持可能会随时**被删除**，恕不另行通知。
- API 可能在以后的软件版本中以**不兼容**的方式更改，恕不另行通知。
- 由于缺陷风险增加和缺乏长期支持，建议该软件**仅用于短期测试集群**。

Beta：

- 版本名称包含 beta（例如：v2beta3）。
- 内置的 Beta API 版本默认被禁用且必须在 kube-apiserver 配置中显式启用才能使用 （例外情况是 Kubernetes 1.22 之前引入的 Beta 版本的 API，这些 API 默认被启用）。
- 内置 Beta API 版本从引入到弃用的最长生命周期为 9 个月或 3 个次要版本（以较长者为准）， 从弃用到移除的最长生命周期为 9 个月或 3 个次要版本（以较长者为准）。
- 软件被很好的测试过。启用某个特性被认为是**安全的**。
- 尽管一些特性会发生**细节上的变化**，但它们将会被**长期支持**。
- 在随后的 Beta 版或 Stable 版中，对象的模式和（或）语义可能以不兼容的方式改变。 当这种情况发生时，将提供迁移说明。 适配后续的 Beta 或 Stable API 版本可能需要编辑或重新创建 API 对象，这可能并不简单。 对于依赖此功能的应用程序，**可能需要停机迁移**。
- 该版本的软件**不建议**生产使用。 后续发布版本可能会有不兼容的变动。 一旦 Beta API 版本被弃用且不再提供服务， 则使用 Beta API 版本的用户需要转为使用后续的 Beta 或 Stable API 版本。

Stable：

- 版本名称如 vX，其中 X 为整数。
- 特性的 Stable 版本会出现在后续很多版本的发布软件中。 Stable API 版本仍然适用于 Kubernetes 主要版本范围内的所有后续发布， 并且 Kubernetes 的主要版本当前没有移除 Stable API 的修订计划。

## API 组（API Groups）

[API Groups](https://github.com/kubernetes/design-proposals-archive/blob/main/api-machinery/api-group.md) 使得 Kubernetes API 更容易被扩展。API Group 在 REST 路径和序列化对象的 apiVersion 中被指定。

这里有几个 Kubernetes 中的 API Groups：

- 核心组（The Core Group），也被称为 legacy（传统），使用 REST Path：`/api/v1`。核心组不会在 apiVersion 字段中出现，例如核心组的 apiVersion 会是：`v1`。
- 命名组（The Named Group），使用 REST Path：`/apis/$GROUP_NAME/$VERSION`。命名组会在 apiVersion 字段中出现：`$GROUP_NAME/$VERSION`，；例如：`batch/v1`。

### 启动或禁用 API Groups

资源和 API Groups 在默认情况下被启用的。

你可以通过在 API Server 上设置 --runtime-config 参数来启用或禁用它们：

- --runtime-config 参数接受逗号分隔的 `<key>=true/false` 对，来描述 API 服务器的运行时关于 key 的配置是否开启。
- 如果省略了 = 部分，那么视其指定为 `=true`。 

配置示例：

- 禁用 batch/v1，对应参数设置 --runtime-config=batch/v1=false
- 启用 batch/v2alpha1，对应参数设置 --runtime-config=batch/v2alpha1
- 要启用特定版本的 API，如 storage.k8s.io/v1beta1/csistoragecapacities，可以设置 --runtime-config=storage.k8s.io/v1beta1/csistoragecapacities

**注意：**

- 启用或禁用组或资源时， 你需要重启 API 服务器和控制器管理器来使 --runtime-config 生效。
