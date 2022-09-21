# Kubernetes API 访问控制

[TOC]

## 概览

用户访问 Kubernetes API 的方式：

- kubectl
- 客户端库
- 构造 REST 请求来访问 Kubernetes API

人类用户和 Kubernetes 服务账户都可以访问 API 时被鉴权。

当请求到达 API 时，它会经历多个阶段，如下图所示：

![](assets/access-control-overview.svg)

## 传输安全

默认情况下，Kubernetes API 服务器在第一个非 localhost 网络接口的 6443 端口上进行监听，受 TLS 保护。

但是，在一个典型的 Kubernetes 生产集群中，API 使用的是 443 端口。

该 TLS 端口可以通过 `--secure-port` 进行变更，监听 IP 地址可以通过 `--bind-address` 标志进行变更。

API 服务器出示证书。该证书可以使用私有证书颁发机构（CA）签名，也可以基于链接到公认的 CA 的公钥基础架构签名。API 服务器证书和相应的私钥可以通过使用 `--tls-cert-file` 和 `--tls-private-key-file` 标志进行设置。

如果你的集群使用私有证书颁发机构，你需要在客户端的 ~/.kube/config 文件中提供该 CA 证书的副本，以便你可以信任该连接并确认该连接没有被拦截。

你的客户端可以在此阶段出示 TLS 客户端证书。

## 认证（Authentication）

如上图步骤 1 所示，建立 TLS 后， HTTP 请求将进入认证（Authentication）步骤，即识别出用户是谁。

集群创建脚本或者集群管理员配置 API 服务器，使之运行一个或多个身份认证组件。身份认证组件在[认证](https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/authentication/)中有更详细的描述。

认证步骤的输入整个 HTTP 请求，但是，通常认证组件只检查：

- 头部
- 客户端证书

认证模块包含：

- 客户端证书
- 密码
- 普通令牌
- 引导令牌
- JSON Web 令牌（JWT，用于服务账户）。

可以指定多个认证模块，在这种情况下，**服务器依次尝试每个验证模块，直到其中一个成功**（也就是说，只要有一个认证模块通过，则认为通过）。

如果请求认证不通过，服务器将以 HTTP 状态码 401 拒绝该请求。反之，**该用户被认证为特定的 username**，并且该用户名可用于后续步骤以在其决策中使用。

## 鉴权（Authorization）

如上图的步骤 2 所示，将请求验证为来自特定的用户后，请求必须被鉴权，即判断用户是否有权限。

请求必须包含：

- 请求者的用户名
- 请求的行为（例如增删改查）
- 受该操作影响的对象

如果现有策略声明用户有权完成请求的操作，那么该请求被鉴权通过。

例如，如果 Bob 有以下策略，那么他只能在 projectCaribou namespace 中读取 Pod。

```json
{
    "apiVersion": "abac.authorization.kubernetes.io/v1beta1",
    "kind": "Policy",
    "spec": {
        "user": "bob",
        "namespace": "projectCaribou",
        "resource": "pods",
        "readonly": true
    }
}
```

如果 Bob 执行以下请求，那么请求会被鉴权，因为允许他读取 projectCaribou namespace 中的对象。

```json
{
  "apiVersion": "authorization.k8s.io/v1beta1",
  "kind": "SubjectAccessReview",
  "spec": {
    "resourceAttributes": {
      "namespace": "projectCaribou",
      "verb": "get",
      "group": "unicorn.example.org",
      "resource": "pods"
    }
  }
}
```

**注意：**

- 如果 Bob 在 projectCaribou namespace 中请求写（create 或 update）对象，其鉴权请求将被拒绝。
- 如果 Bob 在诸如 projectFish 这类其它 namespace 中请求读取（get）对象，其鉴权也会被拒绝。

Kubernetes 支持多种鉴权模块，例如：

- ABAC 模式
- RBAC 模式
- Webhook 模式

管理员创建集群时，他们配置应在 API 服务器中使用的鉴权模块。如果配置了多个鉴权模块，则 Kubernetes 会检查每个模块，**任意一个模块鉴权该请求，请求即可继续**；如果所有模块拒绝了该请求，请求将会被拒绝（HTTP 状态码 403）。

## 准入控制

准入控制模块是可以**修改**或**拒绝请求**的模块。也就是准入控制也可以做一些鉴权模块的工作，除此外，准入控制模块还可以访问正在创建或修改的对象的内容。

准入控制器对创建、修改、删除或连接的对象进行操作，也就是说，准入控制器**不会对仅读取对象的请求起作用**。有多个准入控制器被配置时，服务器将依次调用它们。

这一操作如上图的步骤 3 所示。

与身份认证和鉴权模块不同，如果任何准入控制器模块拒绝某请求，则该请求将立即被拒绝（所有准入控制都允许的请求，才能到下一步）。

除了拒绝对象之外，准入控制器还可以为字段设置复杂的默认值。

可用的准入控制模块在准入控制器中进行了描述。

请求通过所有准入控制器后，将使用检验例程检查对应的 API 对象，然后将其写入对象存储（如步骤 4 所示）。

## 审计

Kubernetes 审计提供了一套与安全相关的、按时间顺序排列的记录，其中记录了集群中的操作序列。

集群对用户、使用 Kubernetes API 的应用程序以及控制平面本身产生的活动进行[审计]()。
