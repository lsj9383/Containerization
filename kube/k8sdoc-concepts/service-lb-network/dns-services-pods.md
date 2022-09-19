# Pod 与 Service 的 DNS

Kubernetes 为 Service 和 Pod 创建 DNS 记录。 你可以使用一致的 DNS 名称而非 IP 地址访问 Service。

Kubernetes DNS 除了在集群上调度 DNS Pod 和 Service，还配置 kubelet 以告知各个容器使用 DNS Service 的 IP 来解析 DNS 名称。

集群中定义的每个服务（包括 DNS 服务器本身）都被分配了一个 DNS 名称。默认情况下，客户端 Pod 的 DNS 搜索列表包括 Pod 自己的命名空间和集群的默认域。

## Service 的名字空间

DNS 查询可能因为执行查询的 Pod 所在的 namespace 而返回不同的结果。

**注意：**

- 不指定名字空间的 DNS 查询会被限制在 Pod 所在的名字空间内。
- 要访问其他名字空间中的 Service，需要在 DNS 查询中指定名字空间。

例如：

- namespace test 中存在一个 Pod
- namespace prod 中存在一个服务 data。

Pod 查询 data 时没有返回结果，因为使用的是 Pod 的名字空间 test。

Pod 查询 data.prod 时则会返回预期的结果，因为查询中指定了名字空间。

Pod 使用的 DNS 查询可以看 Pod 中的 `/etc/resolv.conf`。kubelet 会为每个 Pod 生成此文件：

```sh
$ kubectl exec -ti <pod-name> -- cat /etc/resolv.conf
nameserver 172.16.254.137
search default.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

## DNS 记录

对于这些对象都能获得 DNS 记录：

- Services
- Pods

### Services

对于 Services 的 A 记录：

类型 | 域名格式 | 域名解析结果
-|-|-
无头服务 | `<srv-name>.<namespace>.svc.cluster-<cluster-domain>` | 服务 Endpoints 列表（即所有的 Pod IP）。<br>客户端要能够使用这组 IP，或者使用标准的轮转策略从这组 IP 中进行选择。
其他服务 | `<srv-name>.<namespace>.svc.cluster-<cluster-domain>` | 服务的 Cluster IP（即服务 VIP）。

**注意：**

- 可见所有服务的域名格式都是一致的。

### Pods

一般而言，Pod 会对应如下 DNS 名字解析：

`<pod-ip-address>.<namespace>.pod.cluster-<cluster-domain>`

例如，对于一个位于 `default` 名字空间，IP 地址为 `172.17.0.3` 的 Pod，如果集群的域名为 `cluster.local`，则 Pod 会对应 DNS 名称：

```txt
172-17-0-3.default.pod.cluster.local.
```

通过 Service 暴露出来的所有 Pod 都会有如下 DNS 解析名称可用：

```txt
<pod-ip-address>.<service-name>.<namespace>.svc.cluster-<cluster-domain>
```

#### Pod 的 hostname 和 subdomain 字段

当前，创建 Pod 时其主机名取自 Pod 的 `metadata.name` 值。

Pod 规约中包含一个可选的 hostname 字段，可以用来指定 Pod 的主机名。 当这个字段被设置时，它将优先于 Pod 的名字成为该 Pod 的主机名。 

举个例子，给定一个 hostname 设置为 "my-host" 的 Pod， 该 Pod 的主机名将被设置为 "my-host"。

Pod 规约还有一个可选的 subdomain 字段，可以用来指定 Pod 的子域名。

举个例子，某 Pod 的 hostname 设置为 “foo”，subdomain 设置为 “bar”， 在名字空间 “my-namespace” 中对应的完全限定域名（FQDN）为：

```txt
foo.bar.my-namespace.svc.cluster-domain.example
```

示例：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: default-subdomain
spec:
  selector:
    name: busybox
  clusterIP: None
  ports:
  - name: foo # 实际上不需要指定端口号
    port: 1234
    targetPort: 1234
---
apiVersion: v1
kind: Pod
metadata:
  name: busybox1
  labels:
    name: busybox
spec:
  hostname: busybox-1
  subdomain: default-subdomain
  containers:
  - image: busybox:1.28
    command:
      - sleep
      - "3600"
    name: busybox
---
apiVersion: v1
kind: Pod
metadata:
  name: busybox2
  labels:
    name: busybox
spec:
  hostname: busybox-2
  subdomain: default-subdomain
  containers:
  - image: busybox:1.28
    command:
      - sleep
      - "3600"
    name: busybox
```
