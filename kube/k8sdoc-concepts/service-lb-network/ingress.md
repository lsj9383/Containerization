# Ingress

Ingress 是对集群中服务的外部访问进行管理的 API 对象，典型的访问方式是 HTTP。

Ingress 可以提供负载均衡、SSL 终结和基于名称的虚拟托管。

## 术语

为了表达更加清晰，本指南定义了以下术语：

- 节点（Node）: Kubernetes 集群中的一台工作机器，是集群的一部分。
- 集群（Cluster）: 一组运行由 Kubernetes 管理的容器化应用程序的节点。 在此示例和在大多数常见的 Kubernetes 部署环境中，集群中的节点都不在公共网络中。
- 边缘路由器（Edge Router）: 在集群中强制执行防火墙策略的路由器。可以是由云提供商管理的网关，也可以是物理硬件。
- 集群网络（Cluster Network）: 一组逻辑的或物理的连接，根据 Kubernetes 网络模型在集群内实现通信。
- 服务（Service）：Kubernetes 服务（Service）， 使用标签选择器（selectors）辨认一组 Pod。 除非另有说明，否则假定服务只具有在集群网络中可路由的虚拟 IP。

## Ingress 是什么？

Ingress 公开从集群外部到集群内服务的 HTTP 和 HTTPS 路由。流量路由通过 Ingress 资源上定义的规则控制。

下面是一个将所有流量都发送到同一 Service 的简单 Ingress 示例：

![](assets/ingress.svg)

Ingress 可为 Service 提供外部可访问的 URL、负载均衡流量、终止 SSL/TLS，以及基于名称的虚拟托管(虚拟主机？)。

Ingress Controller 通常负责通过负载均衡器来实现 Ingress，尽管它也可以配置边缘路由器或其他前端来帮助处理流量。

**注意：**

- 因为 Ingress 只支持 HTTP 和 HTTPS 所以 Ingress 不允许开通其他端口，只会对外暴露 80 和 443。
- 如果需要将 HTTP 和 HTTPS 以外的服务公开到 Internet 时，通常使用 Service.Type=NodePort 或 Service.Type=LoadBalancer 类型的 Service。

## 环境准备

你必须拥有一个 [Ingress Controller](https://kubernetes.io/zh-cn/docs/concepts/services-networking/ingress-controllers/) 才能满足 Ingress 的要求。仅创建 Ingress 资源本身没有任何效果。

你可能需要部署 Ingress 控制器，例如 [ingress-nginx](https://kubernetes.github.io/ingress-nginx/deploy/)。 你可以从许多 Ingress 控制器 中进行选择。

理想情况下，所有 Ingress 控制器都应符合参考规范。但实际上，不同的 Ingress 控制器操作略有不同。

一个简单的 ingress-nginx 的安装：

```sh
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.3.1/deploy/static/provider/cloud/deploy.yaml

# 检查是否正确
$ kubectl get pods --namespace=ingress-nginx

kubectl create deployment demo --image=httpd --port=80
kubectl expose deployment demo

kubectl create ingress demo-localhost --class=nginx \
  --rule="demo.localdev.me/*=demo:80"

kubectl port-forward --namespace=ingress-nginx service/ingress-nginx-controller 8080:80
# 您应该会看到一个 HTML 页面告诉您 “It works！”。
```

## Ingress 资源

一个最小的 Ingress 资源示例：

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: minimal-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx-example
  rules:
  - http:
      paths:
      - path: /testpath
        pathType: Prefix
        backend:
          service:
            name: test
            port:
              number: 80
```

Ingress 需要指定:

- apiVersion
- kind
- metadata
- spec

Ingress 对象的命名必须是合法的 DNS 子域名名称。

Ingress 经常使用注解（annotations）来配置一些选项，具体取决于 Ingress 控制器，例如重写目标注解。不同的 Ingress 控制器支持不同的注解。 查看你所选的 Ingress 控制器的文档，以了解其支持哪些注解。

Ingress Spec 提供了配置负载均衡器或者代理服务器所需的所有信息。最重要的是，其中包含与所有传入请求匹配的规则列表。

**注意：**

- Ingress 资源仅支持用于转发 HTTP(S) 流量的规则。
- 如果 ingressClassName 被省略，那么你应该定义一个默认 Ingress 类。
- 有一些 Ingress 控制器不需要定义默认的 IngressClass。比如：Ingress-NGINX 控制器可以通过参数 `--watch-ingress-without-class` 来配置。 不过仍然推荐按下文所示来设置默认的 IngressClass。

### Ingress 规则

每个 HTTP 规则都包含以下信息：

- 可选的 host。在此示例中，未指定 host，因此该规则适用于通过指定 IP 地址的所有入站 HTTP 通信。 如果提供了 host（例如 foo.bar.com），则 rules 适用于该 host。
- 路径列表 paths（例如，/testpath）, 每个路径都有一个由 serviceName 和 servicePort 定义的关联后端。在负载均衡器将流量定向到引用的服务之前，主机和路径都必须匹配传入请求的内容。
- backend（后端）是 Service 文档中所述的服务和端口名称的组合。 与规则的 host 和 path 匹配的对 Ingress 的 HTTP（和 HTTPS ）请求将发送到列出的 backend。


通常在 Ingress Controller 中会配置 defaultBackend（默认后端），以服务于无法与规约中 path 匹配的所有请求。

### 默认后端

没有设置规则的 Ingress 将所有流量发送到同一个默认后端，而 `.spec.defaultBackend` 则是在这种情况下处理请求的那个默认后端。

defaultBackend 通常是 Ingress 控制器的配置选项，而非在 Ingress 资源中指定。如果未设置任何的 .spec.rules，那么必须指定 .spec.defaultBackend。 如果未设置 defaultBackend，那么如何处理所有与规则不匹配的流量将交由 Ingress 控制器决定（请参考你的 Ingress 控制器的文档以了解它是如何处理那些流量的）。

如果没有 hosts 或 paths 与 Ingress 对象中的 HTTP 请求匹配，则流量将被路由到默认后端。

### 资源后端

Resource Backend 是一个引用，指向同一命名空间中的另一个 Kubernetes 资源。

Resource Backend 与 Service 后端是互斥的，在二者均被设置时会无法通过合法性检查。

Resource Backend 的一种常见用法是将所有入站数据导向带有**静态资源**的对象存储后端。

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-resource-backend
spec:
  defaultBackend:
    resource:
      apiGroup: k8s.example.com
      kind: StorageBucket
      name: static-assets
  rules:
    - http:
        paths:
          - path: /icons
            pathType: ImplementationSpecific
            backend:
              resource:
                apiGroup: k8s.example.com
                kind: StorageBucket
                name: icon-assets
```

## Ingress Class

Ingress 可以由不同的 Controller 实现，同时这些 Controller 使用不同的配置。

每个 Ingress 应当指定一个 Class，也就是一个对 IngressClass 资源的引用。IngressClass 资源包含额外的配置，其中包括应当实现该 Ingress 的控制器名称。

```yaml
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: external-lb
spec:
  controller: example.com/ingress-controller
  parameters:
    apiGroup: k8s.example.com
    kind: IngressParameters
    name: external-lb
```

IngressClass 中的 `.spec.parameters` 字段可用于引用其他资源以提供额外的相关配置。

参数（parameters）的具体类型取决于你在 `.spec.controller` 字段中指定的 Ingress 控制器（不同的 Ingress Controller 有不同的 parameters）。

### IngressClass 的作用域

IngressClass 的作用域取决于你的 Ingress 控制器，你可能可以使用：

- 集群范围
- namespace 范围

#### 集群作用域

IngressClass parameters 的默认作用域是集群作用域。

如果你设置了 `.spec.parameters` 字段，且未设置 `.spec.parameters.scope` 字段，或是将 `.spec.parameters.scope` 字段设为了 Cluster，那么该 IngressClass 所指代的即是一个集群作用域的资源。 参数的 kind（和 apiGroup 一起）指向一个集群作用域的 API（可能是一个定制资源（Custom Resource）），而它的 name 则为此 API 确定了一个具体的集群作用域的资源。

示例：

```yaml
---
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: external-lb-1
spec:
  controller: example.com/ingress-controller
  parameters:
    # 此 IngressClass 的配置定义在一个名为 “external-config-1” 的
    # ClusterIngressParameter（API 组为 k8s.example.net）资源中。
    # 这项定义告诉 Kubernetes 去寻找一个集群作用域的参数资源。
    scope: Cluster
    apiGroup: k8s.example.net
    kind: ClusterIngressParameter
    name: external-config-1
```

### 默认 Ingress Class

你可以将一个特定的 IngressClass 标记为集群默认 IngressClass。

将一个 IngressClass 资源的 `ingressclass.kubernetes.io/is-default-class` 注解设置为 true 将确保新的未指定 IngressClassName 字段的 Ingress 能够分配为这个默认的 IngressClass。

**注意：**

- 如果集群中有多个 IngressClass 被标记为默认，准入控制器将阻止创建新的未指定 ingressClassName 的 Ingress 对象。
- 解决这个问题只需确保集群中最多只能有一个 IngressClass 被标记为默认。

有一些 Ingress 控制器不需要定义默认的 IngressClass。比如：Ingress-NGINX 控制器可以通过参数 `--watch-ingress-without-class` 来配置。

不过仍然推荐 设置默认的 IngressClass：

```yaml
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  labels:
    app.kubernetes.io/component: controller
  name: nginx-example
  annotations:
    ingressclass.kubernetes.io/is-default-class: "true"
spec:
  controller: k8s.io/ingress-nginx
```

## Ingress 类型（Type）

Ingress 有三种使用方式：

Type | Description
-|-
Ingress 路由给单个 Service | Ingress 仅为一个 Service 转发请求，直接配置默认后端即可。
简单扇出 | Ingress 根据请求 URI 路由给不同的 Service。
虚拟托管 | Ingress 根据请求的 Host 路由给不同的 Service

**注意：**

- 简单扇出和虚拟托管可以结合使用的。

### 由单个 Service 来完成的 Ingress

这是指的一个 Ingress 就单纯将请求转发给一个 Service，这类无需制定任何路由规则，直接搞个默认后端即可。

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: test-ingress
spec:
  defaultBackend:
    service:
      name: test
      port:
        number: 80
```

**注意：**

- 当然，对于这类暴露单个 Service 的，仅靠 Service 本身也是可以的，例如 NodePort 和 LoadBalancer。

### 简单扇出

一个扇出（fanout）配置根据请求的 HTTP URI 将流量路由到多个 Service。 

![](assets/ingressfanout.svg)

将需要一个如下所示的 Ingress：

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: simple-fanout-example
spec:
  rules:
  - host: foo.bar.com
    http:
      paths:
      - path: /foo
        pathType: Prefix
        backend:
          service:
            name: service1
            port:
              number: 4200
      - path: /bar
        pathType: Prefix
        backend:
          service:
            name: service2
            port:
              number: 8080
```

当你使用 kubectl apply -f 创建 Ingress 时：

```sh
$ kubectl describe ingress simple-fanout-example
Name:             simple-fanout-example
Namespace:        default
Address:          178.91.123.132
Default backend:  default-http-backend:80 (10.8.2.3:8080)
Rules:
  Host         Path  Backends
  ----         ----  --------
  foo.bar.com
               /foo   service1:4200 (10.8.0.90:4200)
               /bar   service2:8080 (10.8.0.91:8080)
Annotations:
  nginx.ingress.kubernetes.io/rewrite-target:  /
Events:
  Type     Reason  Age                From                     Message
  ----     ------  ----               ----                     -------
  Normal   ADD     22s                loadbalancer-controller  default/test
```

### 基于名称的虚拟托管

![](assets/ingressnamebased.svg)


```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: name-virtual-host-ingress
spec:
  rules:
  - host: foo.bar.com
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: service1
            port:
              number: 80
  - host: bar.foo.com
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: service2
            port:
              number: 80
```

## TLS

你可以通过设定包含 TLS 私钥和证书的 Secret 来保护 Ingress。

Ingress **只支持单个 TLS 端口 443**，并假定 TLS 连接终止于 Ingress 节点（与 Service 及其 Pod 之间的流量都以明文传输）。

如果 Ingress 中的 TLS 配置部分指定了不同的主机，那么它们将根据通过 SNI TLS 扩展指定的主机名（如果 Ingress 控制器支持 SNI）在同一端口上进行复用。 TLS Secret 的数据中必须包含用于 TLS 的以键名 tls.crt 保存的证书和以键名 tls.key 保存的私钥。 例如：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: testsecret-tls
  namespace: default
data:
  tls.crt: base64 编码的证书
  tls.key: base64 编码的私钥
type: kubernetes.io/tls
```

在 Ingress 中引用此 Secret 将会告诉 Ingress 控制器使用 TLS 加密从客户端到负载均衡器的通道：

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-example-ingress
spec:
  tls:
  - hosts:
      - https-example.foo.com
    secretName: testsecret-tls
  rules:
  - host: https-example.foo.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: service1
            port:
              number: 80
```
