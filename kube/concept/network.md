# 网络

## 概述

## Service

我们创建一个 Pods 后，虽然可以直接通过 Pods IP 来对 Pods 中的网络应用发起请求，但是 Pods 是可以随时销毁并创建的，因此 Pods IP 是会变更的，这导致当其他服务依赖 Pods 提供的网络应用时无法确定使用哪个 IP。

为了方便请求方进行调用，需要将 Pods 的路由发现专门抽取出来进行管理，进行管理的就是 Service 对象。

通过下述 yaml，Service 会提供 8888 端口，并将接收到的请求转发给 Pods 的 80 端口。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 8888
      targetPort: 80
```

**注意：**

- 通常 port 是指的自己对外暴露的端口，targetPort 是转发到的端口。这里 targetPort 就是转发给 Pods 的应用端口号。
- `.spec.sepector` 用来选择需要提供网络服务的 Pods。
- 会自动创建 `Endpoint` 对象，这个对象是 Service 接收到请求后进行转发的目标。`Endpoint` 的名称和 Service 名称一致。

创建好 Service 后可以进行查看对外暴露的 IP 和 Port，以及使用的 Service 类型。

```sh
$ kubectl get svc
NAME         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
my-service   ClusterIP   10.108.98.98   <none>        8888/TCP   12m
```

### 服务发现

kube 的 service 支持两种服务发现：环境变量和 DNS。

对于 ClusterIP 的 Service，环境变量和 DNS 都指向该 ClusterIP。

**环境变量**

- {SVCNAME}_SERVICE_HOST
- {SVCNAME}_SERVICE_PORT

```sh
# 当前仅 my_service 一个 service 存在

$ env

...
MY_SERVICE_PORT=tcp://10.108.98.98:8888
MY_SERVICE_SERVICE_PORT=8888
...
```

环境变量主要缺点是，在 service 前启动的 Pods 是没有这些 Service 环境变量的。

**DNS**

 DNS 服务器（例如 CoreDNS）监视 Kubernetes API 中的新服务，并为每个服务创建一组 DNS 记录。 如果在整个群集中都启用了 DNS，则所有 Pod 都应该能够通过其 DNS 名称自动解析服务。

### 服务类型

Service 根据不同的服务暴露方式，存在以下 3 种类型：

- ClusterIP，通过集群内部 IP 暴露服务，仅支持集群内访问。
- NodePort，会暴露一个 Node IP 以及 Node Port，外网可以通过 NodeIP:NodePort 发起请求，并转发至 ClusterIP:ClusterPort。集群内使用 Cluster IP:Cluster Port。
- LoadBalancer
- ExternalName

**ClusterIP**

有两种形式：

- 正常形式，提供一个 ClusterIP 作为 VIP，默认情况下 VIP 为自动分配。DNS 记录解析到的是 VIP。
- headless，指定 ClusterIP 为 None，这种情况下生成 Service 不会有 VIP，但有 DNS。DNS 解析出来的是任意一个 Pods IP。

**NodePort**

Node Port 其实和 Cluster IP 基本一样，也有 ClusterIP 和 ClusterPort，但是暴露了 NodeIP:NodePort 转给 ClusterIP:ClusterPort 的入口。

- NodeIP，集群中的任意节点 IP。
- NodePort 在 manifest 的 `.spec.ports.nodePort` 中进行指定。如果没有指定，则会自动生。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-nodeport-service
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
    - port: 8888
      targetPort: 80
      # 可选字段
      # 默认情况下，为了方便起见，Kubernetes 控制平面会从某个范围内分配一个端口号（默认：30000-32767）
      nodePort: 30007
```

通过 `kubectl get svc` 就能查看到 NodePort、ClusterIP、ClusterPort：

```sh
$ kubectl get svc
NAME                  TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
my-nodeport-service   NodePort    10.103.221.204   <none>        8888:30007/TCP   4m53s
my-service            ClusterIP   10.108.98.98     <none>        8888/TCP         35m
```

- ClusterIP 为 10.103.221.204。
- ClusterPort 为 8888。
- NodeIP 为集群节点任意 IP。
- NodePort 为 30007。

## DNS

## 使用 Service 连接到应用

要通过网络暴露应用，需要先创建 Pod：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  selector:
    matchLabels:
      run: my-nginx
  replicas: 2
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - name: my-nginx
        image: nginx
        ports:
        - containerPort: 80
```

**注意：**

- `.spec.template.spec.ports.containerPort` 只是用来声明 Pods 会使用哪些端口，即便不声明也不影响请求。

通过 `kubectl describe` 和 `kubectl get pods` 都将可以获得 Pod IP。

```sh
$ kubectl get pods -l run=my-nginx -o yaml | grep " podIP:"
    podIP: 10.244.0.230
    podIP: 10.244.0.231

$ kubectl describe pods my-nginx-5b56ccd65f-8hs4w
...
IP:           10.244.0.230
IPs:
  IP:           10.244.0.230
...
```

可以直接给这些 Pods 发起请求：

```sh
# 因为我机器上有代理，会进行限制，所以 curl 时强制不使用代理

$ curl "http://10.244.0.230" --noproxy "*"
$ curl "http://10.244.0.231" --noproxy "*"
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

为了解决 Pods 自动创建、销毁时的 IP 问题，需要通过 Service 进行管理。有两种方式：

- 使用 Service 的 yaml 文件。
- 直接使用 `kubectl expose`，该方法简单方便，但是不够灵活。`kubectl expose deployment/my-nginx`，等价于下述 yaml 文件：

    ```yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: my-nginx
      labels:
        run: my-nginx
    spec:
      ports:
      - port: 80
        protocol: TCP
      selector:
        run: my-nginx
    ```

```sh
$ kubectl get svc
NAME         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
my-nginx     ClusterIP   10.102.218.2   <none>        80/TCP    3s
```

每个 Service 都是通过 Endpoint 对象连接到 Pods 的，且 Endpoint 的名称和 Service 一致：

```sh
$ kubectl get ep
NAME         ENDPOINTS                         AGE
my-nginx     10.244.0.230:80,10.244.0.231:80   56s
```

在集群中的任何节点、任何 Pods，都能通过 ClusterIP 和 ClusterPort 进行访问：

```sh
curl "http://10.102.218.2" --noproxy "*"
```

现在进入集群 Pods 中进行测试：

```sh
# 启动有 curl 命令的 Pods，这是一个静态 Pods
$ kubectl run curl-test --image=radial/busyboxplus:curl -i --tty

# 查看环境变量
$ env
MY_NGINX_SERVICE_PORT=80
MY_NGINX_PORT=tcp://10.102.218.2:80
MY_NGINX_PORT_80_TCP_ADDR=10.102.218.2
MY_NGINX_PORT_80_TCP_PORT=80
MY_NGINX_PORT_80_TCP_PROTO=tcp
...

# 查看 DNS
$ nslookup my-nginx
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      my-nginx
Address 1: 10.102.218.2 my-nginx.default.svc.cluster.local

# curl
curl "http://my-nginx.default.svc.cluster.local"
```

**注意：**

- 只有在 Pods 中才能解析 Service 域名，因为 Pods 的 DNS Resolver 是交给 kube-dns 服务来专门进行管理的。
- 宿主机节点上不能使用 Service 域名来获取 Cluster IP，因为宿主机节点的 DNS Resolver 不是 kube-dns。

对于 Service 的暴露，可以使用 NodePort 和 LoadBalancer。

## Ingress
