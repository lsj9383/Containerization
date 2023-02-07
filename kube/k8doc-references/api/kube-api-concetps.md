# Kubernetes API 概念

[TOC]

## 概述

Kubernetes API 通过 HTTP 提供的基于资源 (RESTful) 的编程接口。通过使用 HTTP 标准动词（POST、PUT、PATCH、DELETE、GET）对资源进行检索、创建、更新和删除。

对于某些资源，Kubernetes 支持子资源，允许细粒度授权（例如 Pod 详细信息和日志检索的单独视图），并且可以以不同的表示形式接受和服务这些资源，以方便或提高效率。

Kubernetes 通过 `watches` 来支持高效的资源变更通知，Kubernetes 还提供了一致的列表操作，以便 API 客户端可以有效地缓存、跟踪和同步资源的状态。

## Kubernetes API 术语

Kubernetes 通常利用常见的 RESTful 术语来描述 API 概念：

- 资源类型（Resource Type），是 URL 中使用的名称（pod、namespace、service）
- 所有资源类型都有一个具体的表示（它们的对象模式），称为*类别（Kind）*
- 资源实例的列表称为*集合（Collection）*
- 资源类型的单个实例称为*资源（Resource）*，通常也代表一个*对象（Object）*。
- 对于某些资源类型，API 包含一个或多个*子资源（Sub-Resource）*，这些子资源表示为资源下方的 URI 路径。

**注意：**

- 大多数 Kubernetes API 资源类型都是对象：它们代表集群上某个概念的具体实例，例如 pod 或命名空间。
- 少数 API 资源类型是**虚拟的**，因为它们通常表示是操作，而非对象本身。例如：
  - 权限检查（subjectaccessreviews 资源）
  - 逐出子 Pod 的资源（[API-initiated Eviction](https://kubernetes.io/docs/concepts/scheduling-eviction/api-eviction/)）。

### 对象名字

你可以通过 API 创建的所有对象都有一个唯一的名字，以允许幂等创建和检索。

**注意：**

- 如果虚拟资源类型不可检索或不依赖幂等性，则它们可能没有唯一名称。

在命名空间内，同一时刻只能有一个给定类别的对象具有给定名称。但是，如果你删除该对象，你可以创建一个具有相同名称的新对象。有些对象没有命名空间（例如：节点），因此它们的名称在整个集群中必须是唯一的。

### API 动词

几乎所有对象资源类型都支持标准 HTTP 动词：GET、POST、PUT、PATCH 和 DELETE。Kubernetes 也使用自己的动词，这些动词通常写成小写，以区别于 HTTP 动词。

Kubernetes 使用 list 来描述返回资源集合， 以区别于通常称为 get 的单个资源检索。

如果你发送带有 ?watch 查询参数的 HTTP GET 请求， Kubernetes 将其称为 watch 而不是 get。

对于 PUT 请求，Kubernetes 在内部根据现有对象的状态将它们分类为 create 或 update。

**注意：**

- update 不同于 patch；patch 的 HTTP 动词是 PATCH。

## 资源 URI

资源的作用域只有两种：

- 集群作用域：`/apis/GROUP/VERSION/*`
- 命名空间作用域的：`/apis/GROUP/VERSION/namespaces/NAMESPACE/*`

名字空间作用域的资源类型会在其名字空间被删除时也被删除， 并且对该资源类型的访问是由定义在命名空间域中的授权检查来控制的。

**注意：**

- 核心资源使用 /api 而不是 /apis，并且不包含 GROUP 路径段。

例如:

- /api/v1/namespaces
- /api/v1/pods
- /api/v1/namespaces/my-namespace/pods
- /apis/apps/v1/deployments
- /apis/apps/v1/namespaces/my-namespace/deployments
- /apis/apps/v1/namespaces/my-namespace/deployments/my-deployment

你还可以访问资源集合（例如：列出所有 Node）。以下路径用于检索集合和资源：

集群作用域的资源：

- GET /apis/GROUP/VERSION/RESOURCETYPE - 返回指定资源类型的资源的集合
- GET /apis/GROUP/VERSION/RESOURCETYPE/NAME - 返回指定资源类型下名称为 NAME 的资源

名字空间作用域的资源：

- GET /apis/GROUP/VERSION/RESOURCETYPE - 返回所有名字空间中指定资源类型的全部实例的集合
- GET /apis/GROUP/VERSION/namespaces/NAMESPACE/RESOURCETYPE - 返回名字空间 NAMESPACE 内给定资源类型的全部实例的集合
- GET /apis/GROUP/VERSION/namespaces/NAMESPACE/RESOURCETYPE/NAME - 返回名字空间 NAMESPACE 中给定资源类型的名称为 NAME 的实例

由于名字空间本身是一个集群作用域的资源类型，你可以通过 GET /api/v1/namespaces/ 检视所有名字空间的列表（“集合”），使用 GET /api/v1/namespaces/NAME 查看特定名字空间的详细信息。

- 集群作用域的子资源：GET /apis/GROUP/VERSION/RESOURCETYPE/NAME/SUBRESOURCE
- 名字空间作用域的子资源：GET /apis/GROUP/VERSION/namespaces/NAMESPACE/RESOURCETYPE/NAME/SUBRESOURCE

取决于对象是什么，每个子资源所支持的动词有所不同。跨多个资源来访问其子资源是不可能的，如果需要这一能力，则通常意味着需要一种新的虚拟资源类型了。

## 高效检测变更

Kubernetes API 允许客户端对对象或集合发出初始请求，然后跟踪自该初始请求以来的更改：watch。客户端可以发送 list 或者 get 请求，然后发出后续 watch 请求。

为了使这种更改跟踪成为可能，每个 Kubernetes 对象都有一个 resourceVersion 字段，表示存储在底层持久层中的该资源的版本。在检索资源集合（命名空间或集群范围）时，来自 API Server 的响应包含一个 resourceVersion 值。 客户端可以使用该 resourceVersion 来启动对 API 服务器的 watch。

当你发送 watch 请求时，API Server 响应一个变化流（一个 CHUNKED，保持长连接）。这些更改逐项列出了在你指定为 watch 请求参数的 resourceVersion 之后发生的操作（例如 create、delete 和 update）的结果。 整个 watch 机制允许客户端获取当前状态，然后订阅后续更改，而不会丢失任何事件。

如果客户端 watch 连接断开，则该客户端可以从最后返回的 resourceVersion 开始新的 watch 请求； 客户端还可以执行新的 get/list 请求并重新开始。有关更多详细信息，请参阅资源版本语义。

例如：

列举给定名字空间中的所有 Pod：

```http
GET /api/v1/namespaces/test/pods
---
200 OK
Content-Type: application/json

{
  "kind": "PodList",
  "apiVersion": "v1",
  "metadata": {"resourceVersion":"10245"},
  "items": [...]
}
```

从资源版本 10245 开始，接收影响 test 命名空间中 Pod 的所有 API 操作 （例如 create、delete、apply 或 update）的通知。 每个更改通知都是一个 JSON 文档。 HTTP 响应正文（用作 application/json）由一系列 JSON 文档组成。

```http
GET /api/v1/namespaces/test/pods?watch=1&resourceVersion=10245
---
200 OK
Transfer-Encoding: chunked
Content-Type: application/json

{
  "type": "ADDED",
  "object": {"kind": "Pod", "apiVersion": "v1", "metadata": {"resourceVersion": "10596", ...}, ...}
}
{
  "type": "MODIFIED",
  "object": {"kind": "Pod", "apiVersion": "v1", "metadata": {"resourceVersion": "11020", ...}, ...}
}
...
```

给定的 Kubernetes 服务器只会保留一定的时间内发生的历史变更列表。 使用 etcd3 的集群默认保存过去 5 分钟内发生的变更。 当所请求的 watch 操作因为资源的历史版本不存在而失败， 客户端必须能够处理因此而返回的状态代码 410 Gone，清空其本地的缓存， 重新执行 get 或者 list 操作， 并基于新返回的 resourceVersion 来开始新的 watch 操作。

对于订阅集合，Kubernetes 客户端库通常会为 list -然后- watch 的逻辑提供某种形式的标准工具。 （在 Go 客户端库中，这称为 反射器（Reflector），位于 k8s.io/client-go/tools/cache 包中。）
