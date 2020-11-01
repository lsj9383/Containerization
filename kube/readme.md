# Kubernetes

## 概述

## 安装

```sh
yum makecache fast
yum install -y kubelet
yum install -y kubectl
yum install -y kubeadm

kubeadm init --service-cidr=10.96.0.0/12 --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=9.134.9.104

kubeadm join 9.134.9.104:6443 --token 51gfwp.2kmsf2fgfz57rr63 \
    --discovery-token-ca-cert-hash sha256:0f8fca4f9e5d214303f3f6616c4fdc206c083abf759a68229d4cd2760182e3fc 
```

### 单机使用 kube

单机使用 kube 意味着 master 节点可以调度 Pods，但是默认情况下 master 节点有污点，不能进行 Pods 调度，因此需要去除污点：

```sh
kubectl taint nodes --all node-role.kubernetes.io/master-
```

### 速记

资源列表

```sh
kubectl get ${resource} -n ${namespace} -l ${label}
kubectl get pods
kubectl get pods -n kube-system
kubectl get pods -l app=hello

# 以 yaml 方式显示资源
kubectl get pods --output=yaml

# 获取资源时，显示资源的 label
kubectl get pods --show-labels
```

资源详情

```sh
kubectl describe ${resource} ${resource-name} -n ${namespace}
kubectl describe pod hello-pod
```
