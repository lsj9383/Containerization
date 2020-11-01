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
