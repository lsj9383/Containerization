# Kubectl References

[TOC]

## 概览

什么是 Kubectl？

> Kubernetes 提供 kubectl 是使用 Kubernetes API Server 与 Kubernetes 集群的控制面进行通信的命令行工具。

Kubectl 连接 Kubernetes API Server 前需要先获取配置，以决定使用什么身份、凭证、以及连接哪个 API Server。

kubectl 的配置文件指定方式 ：

- 未指定，使用默认值：`$HOME/.kube/config`。
- 环境变量指定，`KUBECONFIG`
- 命令行指定，`--kubeconfig`

## 语法

Kubectl 不同命令的语法是不同的，但是对于大部分使用者而言，语法是类似的：

```sh
kubectl [command] [TYPE] [NAME] [flags]
```

参数 | 描述 | 示例
-|-|-
command | 指定对资源进行何种操作。 | create、get、describe、delete
TYPE | 指定资源类型。资源类型不区分大小写，可以指定单数、复数或缩写形式。| `kubectl get pod`、`kubectl get pods`、`kubectl get po`
NAME | 指定资源的名称。名称区分大小写。如果省略名称，则显示所有资源的详细信息。| kubectl get pods curl
flags | 指定可选的参数。| 例如 --server 参数指定 Kubernetes API 服务器的地址和端口。


**注意：**

- 可以支持对多个资源进行操作：
  - 要对所有类型相同的资源进行分组，请执行以下操作：TYPE1 name1 name2 name<#>。例子：`kubectl get pod example-pod1 example-pod2`
  - 分别指定多个资源类型：TYPE1/name1 TYPE1/name2 TYPE2/name3 TYPE<#>/name<#>。例子：`kubectl get pod/example-pod1 replicationcontroller/example-rc1`
  - 用一个或多个文件指定资源：-f file1 -f file2 -f file<#>。
- 从命令行指定的参数会覆盖默认值和任何相应的环境变量。

## 集群内身份验证和命名空间覆盖

kubectl 可以在集群内被运行，那什么是集群内运行呢？

> kubectl 命令首先确定它是否在 Pod 中运行，从而被视为在集群中运行。

当同时满足以下条件，Kubectl 就被认为处于 Pod 中：

- 检查环境变量：KUBERNETES_SERVICE_HOST
- 检查环境变量：KUBERNETES_SERVICE_PORT
- 检查服务账户令牌文件：`/var/run/secrets/kubernetes.io/serviceaccount/token`

如果三个条件都被满足，则假定在集群内进行身份验证。

kubectl 此时如果需要使用命名空间，则是用的 serviceaccount 令牌文件中的命名空间。

## 操作

操作 | 语法 | 描述
-|-|-
alpha | `kubectl alpha SUBCOMMAND [flags]` | 列出与 alpha 特性对应的可用命令，这些特性在 Kubernetes 集群中默认情况下是不启用的。
annotate | `kubectl annotate (-f FILENAME | TYPE NAME | TYPE/NAME) KEY_1=VAL_1 ... KEY_N=VAL_N [--overwrite] [--all] [--resource-version=version] [flags]` | 添加或更新一个或多个资源的注解。
api-resources | `kubectl api-resources [flags]` | 列出可用的 API 资源。
api-versions | `kubectl api-versions [flags]` | 列出可用的 API 版本。
apply | `kubectl apply -f FILENAME [flags]` | 从文件或 stdin 对资源应用配置更改。
attach | `kubectl attach POD -c CONTAINER [-i] [-t] [flags]` | 挂接到正在运行的容器，查看输出流或与容器（stdin）交互。
auth | `kubectl auth [flags] [options]` | 检查授权。
cluster-info | `kubectl cluster-info [flags]` | 显示有关集群中主服务器和服务的端口信息。
completion | `kubectl completion SHELL [options]` | 为指定的 Shell（Bash 或 Zsh）输出 Shell 补齐代码。
logs | `kubectl logs POD [-c CONTAINER] [--follow] [flags]` | 打印 Pod 中容器的日志。
top | `kubectl top [flags] [options]` | 显示资源（CPU、内存、存储）的使用情况。
plugin | `kubectl plugin [flags] [options]` | 提供用于与插件交互的实用程序。
