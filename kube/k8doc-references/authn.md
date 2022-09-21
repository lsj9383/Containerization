# 认证（Authenticating）

[TOC]

## Kubernetes 中的用户

所有 Kubernetes 集群都有两类用户：

- 由 Kubernetes 管理的服务账号
- 普通用户（normal users）

假设一个独立于集群的服务通过以下方式管理普通用户：

- 负责分发私钥的管理员
- 类似 Keystone 或者 Google Accounts 这类用户数据库
- 包含用户名和密码列表的文件

鉴于此，Kubernetes 并不包含用来代表普通用户账号的对象。普通用户的信息无法通过 API 调用添加到集群中。

尽管无法通过 API 调用来添加普通用户，Kubernetes 仍然认为能够提供由集群的证书机构签名的合法证书的用户是通过身份认证的用户。

基于这样的原因：

- Kubernetes 使用证书中的 'subject' 的通用名称（Common Name）字段 （例如，"/CN=bob"）来确定用户名。
- 接下来，基于角色访问控制（RBAC）子系统会确定用户是否有权针对某资源执行特定的操作。

与此不同，服务账号是 Kubernetes API 所管理的用户。它们被绑定到特定的名字空间，或者由 API 服务器自动创建，或者通过 API 调用创建。服务账号与一组以 Secret 保存的凭据相关，**这些凭据会被挂载到 Pod 中**，从而允许集群内的进程（Pod）访问 Kubernetes API。

API 请求要么与某普通用户相关联，要么与某服务账号相关联， 要么被视作匿名请求：

- 这意味着集群内外的每个进程（键入 kubectl 的人类用户、kubelet 等等）在向 API 服务器发起请求时都必须通过身份认证
- 否则会被视作匿名用户
