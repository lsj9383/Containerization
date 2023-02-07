# Kubernetes

## 概述

[Kubernetes 文档](https://kubernetes.io/zh-cn/docs/home/)

## 速记

对常用的 Kubernetes 运维操作命令进行记录。

## 词汇

kubectl 中常见资源类型缩写：

- service -----> svc
- namespace ---> ns
- replicaset --> rs
- endpoints ---> ep

对于 Kubernetes 中的数值单位主要有（数值单位解释了数值的含义）：

- 十进制单位：
  - m（milli）
  - k（kilo，有意小写）
  - M（mega）
  - G（giga）
  - T（terra）
  - P（peta）
  - E（exa）
- 二进制单位
  - Ki (kibi)
  - Mi (mebi)
  - Gi (gibi)
  - Ti (tebi)
  - Pi (pebi)
  - Ei (exbi)

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

## 集群方面

做一些集群网络相关的测试，构造一个简单的 Pod：

```sh
$ kubectl run curl --image=radial/busyboxplus:curl -i --tty

# Pod 名称是 curl
$ kubectl get pods
NAME                                     READY   STATUS    RESTARTS   AGE
curl                                     1/1     Running   0          7s
```
