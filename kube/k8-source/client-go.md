# 交互式编程 SDK：client-go

[TOC]

## 概述

Kubernetes 系统使用 client-go 作为 Go 语言的官方编程式交互客户端库，提供对 Kubernetes API Server 服务的交互访问。

开发者常使用 client-go 基于 Kubernetes 做二次开发，所以 client-go 是开发者应熟练掌握的必会技能。

## client-go 目录结构

## Client 客户端对象

客户端对象 | 描述 | 备注
-|-|-
RESTClient | 最基础的客户端。RESTClient 对 HTTP Request 进行了封装，实现了 RESTful 风格的 API。| 其他 Client 都基于 RESTClient 实现。
ClientSet | 封装了对 Resource 和 Version 的管理方法。每一个 Resource 可以理解为一个客户端，而 ClientSet 则是多个客户端的集合，每一个 Resource 和 Version 都以函数的方式暴露给开发者。只能访问内置资源，不能访问 CRD。 | 通过 client-gen 代码生成器自动生成的。
DynamicClient | 和 ClientSet 类似，但是可以处理用户自定义资源（CRD）。
DiscoveryClient | 用于发现 kube-apiserver 所支持的资源组、资源版本、资源信息。

RESTClient、ClientSet、DynamicClient、DiscoveryClient 都可以通过 kubeconfig 配置信息连接到指定的 Kubernetes API Server。

### kubeconfig 配置管理

kubeconfig 用于管理访问 kube-apiserver 的配置信息，同时也支持访问多 kube-apiserver 的配置管理，可以在不同的环境下管理不同的 kube-apiserver 集群配置，不同的业务线也可以拥有不同的集群。

kubeconfig 配置信息通常包含3个部分:

- clusters：定义 Kubernetes 集群信息，例如 kube-apiserver 的服务地址及集群的证书信息等。
- users：定义 Kubernetes 集群用户身份验证的客户端凭据，例如 client-certificate、client-key、token 及 username/password 等。
- contexts：定义 Kubernetes 集群用户信息和命名空间等，用于将请求发送到指定的集群。

在默认的情况下，kubeconfig 存放在 `$HOME/.kube/config` 路径下：

```sh
$ cat ~/.kube/config
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CRUdJTiBD......tLS0tLQo=
    server: https://169.254.128.13:60002
  name: local
contexts:
- context:
    cluster: local
    user: admin
  name: master
current-context: master
kind: Config
preferences: {}
users:
- name: admin
  user:
    client-certificate-data: LS0tLS1CRUdJTiB......S0tLS0tCg==
    client-key-data: LS0tLS1CRU......FIEtFWS0tLS0tCg=
```

## Informer 机制

在 Kubernetes 系统中，组件之间通过 HTTP 协议进行通信，在不依赖任何中间件的情况下需要保证消息的实时性、可靠性、顺序性等，这都依赖于 client-go 的 Informer 机制。


