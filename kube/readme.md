# Kubernetes

[TOC]

## 概述

[Kubernetes 文档](https://kubernetes.io/zh-cn/docs/home/)

## 速记

对常用的 Kubernetes 运维操作命令进行记录。

### 词汇

kubectl 中常见资源类型缩写：

- service -----> svc
- namespace ---> ns
- replicaset --> rs
- endpoints ---> ep

### 容器方面

进入容器：

```sh
# docker 方案：docker exec -i <container-id> <command>
docker exec -i 69d1 bash
```

查看容器日志：

```sh
# kubectl logs <pod-name> -c <container-name>
kubectl logs myapp-pod -c init-myservice
```

### 资源方面

资源列表

```sh
# kubectl get ${resource} -n ${namespace} -l ${label}
kubectl get pods
kubectl get pods -n kube-system     # 指定命名空间
kubectl get pods -l app=hello       # 根据标签筛选
kubectl get pods -o wide            # 输出更多 pods 信息
kubectl get pods --output=yaml      # 显示所有的 Pods 资源的 yaml
kubectl get pods --show-labels      # 获取资源时，显示资源的 label
```

资源详情

```sh
# Describe 命令
kubectl describe ${resource} ${resource-name} -n ${namespace}
kubectl describe pod hello-pod

# 以 yaml 方式显示资源
# kubectl get ${resource} ${resource-name} --output=yaml
# kubectl get ${resource} ${resource-name} -o=yaml
kubectl get pod hello-pod -o=yaml
```

删除资源

```sh
# 指令式
# kubectl delete ${resource} ${resource-name} -n ${namespace}
kubectl delete pods hello-pod

# 声明式
kubectl delete -f hello.yaml
```

编辑资源

```sh
# kubectl delete <resource> <resource-name>
kubectl edit pv task-pv-volume
```

### 集群方面

组件方面：

```sh
# 查看客户端 kubectl 版本和 kubernetes 集群（Master）版本
$ kubectl version
Client Version: version.Info{Major:"1", Minor:"22+", GitVersion:"v1.22.5-tke.9", GitCommit:"717871e426fe30284bc7675e1c28c70f49c7750c", GitTreeState:"clean", BuildDate:"2023-01-04T14:20:35Z", GoVersion:"go1.16.14", Compiler:"gc", Platform:"linux/amd64"}
Server Version: version.Info{Major:"1", Minor:"22+", GitVersion:"v1.22.5-tke.9", GitCommit:"717871e426fe30284bc7675e1c28c70f49c7750c", GitTreeState:"clean", BuildDate:"2023-01-04T14:17:29Z", GoVersion:"go1.16.14", Compiler:"gc", Platform:"linux/amd64"}

# 查看集群信息
$ kubectl cluster-info
Kubernetes control plane is running at https://10.0.1.15
CoreDNS is running at https://10.0.1.15/api/v1/namespaces/kube-system/services/kube-dns:dns-tcp/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.

# 查看 kubectl 配置
$ kubectl config view
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: DATA+OMITTED
    server: https://10.0.1.15
  name: cls-0glkf4gu
contexts:
- context:
    cluster: cls-0glkf4gu
    user: "99449107"
  name: cls-0glkf4gu-99449107-context-default
current-context: cls-0glkf4gu-99449107-context-default
kind: Config
preferences: {}
users:
- name: "99449107"
  user:
    client-certificate-data: REDACTED
    client-key-data: REDACTED
```

做一些集群网络相关的测试，构造一个简单的 Pod：

```sh
$ kubectl run curl --image=radial/busyboxplus:curl -i --tty

# Pod 名称是 curl
$ kubectl get pods
NAME                                     READY   STATUS    RESTARTS   AGE
curl                                     1/1     Running   0          7s
```

### 调试相关

```sh
# 列出资源
$ kubectl get

# 显示有关资源的详细信息
$ kubectl describe

# 打印 pod 和其中容器的日志
$ kubectl logs

# 在 pod 中的容器上执行命令
$ kubectl exec

# 直接启动一个 pod
$ kubectl run
```
