# Kubernetes

## 概述

[Kubernetes 文档](https://kubernetes.io/zh-cn/docs/home/)

### 速记

进入容器：

```sh
# docker 方案
docker exec -i 69d1 bash
```

资源列表

```sh
kubectl get ${resource} -n ${namespace} -l ${label}
kubectl get pods
kubectl get pods -n kube-system     # 指定命名空间
kubectl get pods -l app=hello       # 根据标签筛选
kubectl get pods  -o wide           # 输出更多 pods 信息

# 以 yaml 方式显示资源
kubectl get ${resource} ${resource-name} --output=yaml
kubectl get ${resource} ${resource-name} -o=yaml

# 显示所有的 Pods 资源的 yaml
kubectl get pods --output=yaml

# 获取资源时，显示资源的 label
kubectl get pods --show-labels
```

资源详情

```sh
kubectl describe ${resource} ${resource-name} -n ${namespace}
kubectl describe pod hello-pod
```

删除资源

```sh
# 指令式
kubectl delete ${resource} ${resource-name} -n ${namespace}
kubectl delete pods hello-pod

# 声明式
kubectl delete -f hello.yaml
```

kubectl 中常见资源类型缩写：

- endpoints ---> ep
- service -----> svc
- namespace ---> ns
- replicaset --> rs
