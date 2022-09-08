# Kubenertes 对象

<!-- TOC -->

- [Kubenertes 对象](#kubenertes-对象)
    - [Namepsace](#namepsace)
        - [设置 Namespace 偏好](#设置-namespace-偏好)
        - [Namespace 和 DNS](#namespace-和-dns)
        - [在命名空间外的对象](#在命名空间外的对象)
    - [Labels 和 Selectors](#labels-和-selectors)
        - [Selectors](#selectors)
        - [API](#api)

<!-- /TOC -->

## Namepsace

一个初始化 kube 集群包含以下几个命名空间：

- default
- kube-system Kubernetes
- kube-public
- kube-node-lease

### 设置 Namespace 偏好

可以设置后续 kubectl 命令所处的命名空间。

```sh
kubectl config set-context --current --namespace=<名字空间名称>

# 验证
kubectl config view | grep namespace:
```

### Namespace 和 DNS

当创建一个服务时，Kube 会自动创建一个响应的 DNS 条目，且格式为：q

```sh
<服务名称>.<命名空间名称>.svc.cluster.local
```

### 在命名空间外的对象

有些 kube 资源对象并不会属于命名空间，可以通过以下命令查看：

```sh
# In a namespace
kubectl api-resources --namespaced=true

# Not in a namespace
kubectl api-resources --namespaced=false
```

## Labels 和 Selectors

Label 是对象上的 kv 对，例如 Pods。Label 的目的是用于选择对象的子集，可以在对象创建时添加，也可以随后进行修改和添加。每个对象都可以有一组 kv 标签，每个 k 在一个对象中是唯一的。

在 kube 中，Label 都在对象的元数据（metadata）中进行声明：

```yaml
metadata:
  labels:
    key1: value1
    key2: value2
```

服务部署和批处理时，资源往往时有交集的，这可以通过 Label 的方式解决，例如通过 Label 划分多环境、发布版本等。例如：

- "release" : "stable", "release" : "canary"
- "environment" : "dev", "environment" : "qa", "environment" : "production"
- "tier" : "frontend", "tier" : "backend", "tier" : "cache"
- "partition" : "customerA", "partition" : "customerB"
- "track" : "daily", "track" : "weekly"

### Selectors

不同于对象的名称和 UIDs，Label 不支持唯一性，因为 Label 的目的就是为了筛选出一组对象。筛选的方式则是通过 Selectors，目前支持两种 Selectors：`equality-based` 和 `set-based`。多个要求之间可以使用逻辑与进行连接，逻辑与在 selector 使用逗号 `,`。

**Equality-based**

可以通过 `=`、`==`、`!=` 进行筛选，前两者意义一致，都代表筛选出 key 和 value 相同的对象，后者筛选出 key 和 value 不相同的对象。

```sh
environment = production
tier != frontend
```

**Set-based**

可以通过 `in`、`not in`、`exists` 进行筛选。

```sh
# 筛选 key 为 environment 值为 production 和 qa 的对象
environment in (production, qa)

# 筛选 key 为 tier，值不为 frontend 和 backend 的对象
tier notin (frontend, backend)

# 存在 key 为 partition 的对象
partition

# 不存在 key 为 partition 的对象
!partition
```

### API

在 kube 的 API（ LIST 和 WATCH） 中，可以通过 selector 过滤出满足期望 label 的对象，例如：

```sh
kubectl get pods -l environment=production,tier=frontend

kubectl get pods -l 'environment in (production),tier in (frontend)'
```
