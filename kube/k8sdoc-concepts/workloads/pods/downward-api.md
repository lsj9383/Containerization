# Downward API

有时，容器拥有关于自身的信息而不过度耦合到 Kubernetes 是很有用的。

Downward API 允许容器在不使用 Kubernetes 客户端或 API 服务器的情况下使用有关自己或集群的信息。

例如, 在 Kubernetes 中，有两种方法可以将 Pod 和容器字段暴露给正在运行的容器：

- 作为环境变量
- 作为卷中的文件downwardAPI

这两种暴露 Pod 和容器字段的方式统称为 Downward API。

## 可用字段

只有部分 Kubernetes API 字段可以通过 Downward API 使用。本节列出了您可以使哪些字段可用。


- 你可以使用 fieldRef 传递来自可用的 Pod 级字段的信息。
- 你可以使用 resourceFieldRef 传递来自可用的 Container 级字段的信息。

### fieldRef

对于大多数 Pod 级别的字段，你可以通过 fieldRef 将它们作为**环境变量**或使用 **Downward API 卷**提供给容器。

通过这两种机制可用的字段有：

字段 | 环境变量机制 | Downward API 卷机制 | 描述
-|-|-|-
metadata.name | √ | √ | Pod 的名称
metadata.namespace | √ | √ | Pod 的命名空间
metadata.uid | √ | √ | Pod 的唯一 ID
metadata.annotations['<KEY>'] | √ | √ | Pod 的注解 <KEY> 对应的 Value 值（例如：metadata.annotations['myannotation']）
metadata.labels['<KEY>'] | √ | √ | Pod 的标签 <KEY> 对应的 Value 值（例如：metadata.labels['mylabel']）
spec.serviceAccountName | √ | √ | Pod 的服务账号名称
spec.nodeName | √ | √ | Pod 运行时所处的节点名称
status.hostIP | √ | √ | Pod 所在节点的主 IP 地址
status.podIP | √ | √ | Pod 的主 IP 地址（通常是其 IPv4 地址）
metadata.labels | × | √ | 所有 pod 的标签，格式为 label-key="escaped-label-value" 每行一个标签
metadata.annotations | × | √ | 所有 pod 的注释，格式为 annotation-key="escaped-annotation-value" 每行一个注释

这是一个例子:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: dapi-envars-fieldref
spec:
  containers:
    - name: test-container
      image: k8s.gcr.io/busybox
      command: [ "sh", "-c"]
      args:
      - while true; do
          echo -en '\n';
          printenv MY_NODE_NAME MY_POD_NAME MY_POD_NAMESPACE;
          printenv MY_POD_IP MY_POD_SERVICE_ACCOUNT;
          sleep 10;
        done;
      env:
        - name: MY_NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        - name: MY_POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: MY_POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: MY_POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        - name: MY_POD_SERVICE_ACCOUNT
          valueFrom:
            fieldRef:
              fieldPath: spec.serviceAccountName
  restartPolicy: Never
```

### resourceFieldRef

这些容器级字段允许您提供有关 CPU 和内存等资源的请求和限制的信息。

字段 | 描述
-|-
resource: limits.cpu | 容器的 CPU 限制
resource: requests.cpu | 容器的 CPU 请求
resource: limits.memory | 容器的内存限制
resource: requests.memory | 容器的内存请求
resource: limits.hugepages-* | 容器的大页面限制（前提是启用了DownwardAPIHugePages 功能门）
resource: requests.hugepages-* | 容器的大页面请求（前提是启用了DownwardAPIHugePages 功能门）
resource: limits.ephemeral-storage | 容器的临时存储限制
resource: requests.ephemeral-storage | 容器的临时存储请求

这是一个例子:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: dapi-envars-resourcefieldref
spec:
  containers:
    - name: test-container
      image: k8s.gcr.io/busybox:1.24
      command: [ "sh", "-c"]
      args:
      - while true; do
          echo -en '\n';
          printenv MY_CPU_REQUEST MY_CPU_LIMIT;
          printenv MY_MEM_REQUEST MY_MEM_LIMIT;
          sleep 10;
        done;
      resources:
        requests:
          memory: "32Mi"
          cpu: "125m"
        limits:
          memory: "64Mi"
          cpu: "250m"
      env:
        - name: MY_CPU_REQUEST
          valueFrom:
            resourceFieldRef:
              containerName: test-container
              resource: requests.cpu
        - name: MY_CPU_LIMIT
          valueFrom:
            resourceFieldRef:
              containerName: test-container
              resource: limits.cpu
        - name: MY_MEM_REQUEST
          valueFrom:
            resourceFieldRef:
              containerName: test-container
              resource: requests.memory
        - name: MY_MEM_LIMIT
          valueFrom:
            resourceFieldRef:
              containerName: test-container
              resource: limits.memory
  restartPolicy: Never
```

#### 资源限制的后备信息

如果没有为容器指定 CPU 和内存限制，并且您使用向下 API 尝试公开该信息，则 kubelet 默认公开基于**节点可分配计算的 CPU 和内存的最大可分配值**。
