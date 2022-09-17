# 服务内部流量策略

**服务内部流量策略** 开启了内部流量限制：

- 只路由内部流量到和发起方处于**相同节点**的服务 Endpoint。
- 这里的**内部流量**指当前**集群中的 Pod 所发起的流量**。

这种机制有助于节省开销，提升效率。

## 使用服务内部流量策略

**服务内部流量策略**需要通过特性门控 `ServiceInternalTrafficPolicy` 设置，特性默认是启用的。

`.spec.internalTrafficPolicy` 的取值：

- Cluster，默认
- Local，开启服务内部流量策略

启用该功能后，你就可以通过将 `Services` 的 `.spec.internalTrafficPolicy` 项设置为 `Local`，来为它指定一个内部专用的流量策略。

此设置就相当于告诉 kube-proxy 对于集群内部流量只能使用本地的服务端口。

**注意：**

- kube-proxy 将会假定 pod 只想与 pod 所在节点上的服务端点通信，如果没有本地端点，则丢弃流量。

示例：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app.kubernetes.io/name: MyApp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
  internalTrafficPolicy: Local
```

## 工作原理

kube-proxy 基于 spec.internalTrafficPolicy 的设置来过滤路由的目标服务端点。

当它的值设为 Local 时，只选择节点本地的服务端点。
当它的值设为 Cluster 或缺省时，则选择所有的服务端点。

启用特性门控 ServiceInternalTrafficPolicy 后， spec.internalTrafficPolicy 的值默认设为 Cluster。

## 限制

在一个Service上，当 `externalTrafficPolicy` 已设置为 Local时，`internalTrafficPolicy` 无法使用。

换句话说：在一个集群的不同 Service 上可以同时使用这两个特性，但在一个 Service 上不行。

## 一个简单的测试

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx-internal-traffic
spec:
  selector:
    matchLabels:
      run: my-nginx-internal-traffic
  replicas: 1
  template:
    metadata:
      labels:
        run: my-nginx-internal-traffic
    spec:
      containers:
      - name: my-nginx
        image: nginx
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: my-nginx-internal-traffic
  labels:
    run: my-nginx-internal-traffic
spec:
  ports:
    - port: 80
      protocol: TCP
  selector:
    run: my-nginx-internal-traffic
  internalTrafficPolicy: Local
```

在启动了该 service 和 pods 后：

```sh
$ kubectl get svc
my-nginx-internal-traffic   ClusterIP      172.16.253.97    <none>        80/TCP          2m36s

# 在节点 1 上发起 curl 请求（失败）：
$ curl "http://172.16.253.97"
curl: (7) Failed to connect to 172.16.253.97 port 80: Connection refused

# 在节点 2 上发起 curl 请求（成功）：
$ curl "http://172.16.253.97"
!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
...
```