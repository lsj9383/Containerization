# 投射卷

[TOC]

## 概览

一个 projected volume 可以将若干现有的不同卷源，映射到同一个目录之上。

目前，以下类型的卷源可以被投射：

- secret
- downwardAPI
- configMap
- serviceAccountToken

所有的卷源都要求处于 Pod 所在的同一个名字空间内。

## 示例

下面展示几种不同场景示例。

### 带有 Secret、DownwardAPI 和 ConfigMap 的配置示例

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: volume-test
spec:
  containers:
  - name: container-test
    image: busybox:1.28
    volumeMounts:
    - name: all-in-one
      mountPath: "/projected-volume"
      readOnly: true
  volumes:
  - name: all-in-one
    projected:
      sources:
      - secret:
          name: mysecret
          items:
            - key: username
              path: my-group/my-username
      - downwardAPI:
          items:
            - path: "labels"
              fieldRef:
                fieldPath: metadata.labels
            - path: "cpu_limit"
              resourceFieldRef:
                containerName: container-test
                resource: limits.cpu
      - configMap:
          name: myconfigmap
          items:
            - key: config
              path: my-group/my-config
```

### 带有非默认权限模式设置的 Secret 的配置示例

投射卷也可以配置访问权限。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: volume-test
spec:
  containers:
  - name: container-test
    image: busybox:1.28
    volumeMounts:
    - name: all-in-one
      mountPath: "/projected-volume"
      readOnly: true
  volumes:
  - name: all-in-one
    projected:
      sources:
      - secret:
          name: mysecret
          items:
            - key: username
              path: my-group/my-username
      - secret:
          name: mysecret2
          items:
            - key: password
              path: my-group/my-password
              mode: 511
```

每个被投射的卷源都列举在规约中的 sources 下面。参数几乎相同，只有两个例外：

- 对于 Secret，secretName 字段被改为 name 以便于 ConfigMap 的命名一致；
- defaultMode 只能在投射层级设置，不能在卷源层级设置。不过，正如上面所展示的，你可以显式地为每个投射单独设置 mode 属性。

### serviceAccountToken 投射卷

当 TokenRequestProjection 特性被启用时，你可以将当前**服务账号的令牌**注入到 Pod 中特定路径下。例如：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: sa-token-test
spec:
  containers:
  - name: container-test
    image: busybox:1.28
    volumeMounts:
    - name: token-vol
      mountPath: "/service-account"
      readOnly: true
  serviceAccountName: default
  volumes:
  - name: token-vol
    projected:
      sources:
      - serviceAccountToken:
          audience: api
          expirationSeconds: 3600
          path: token
```

示例 Pod 中包含一个投射卷，其中包含注入的服务账号令牌。

此 Pod 中的容器可以使用该令牌访问 Kubernetes API 服务器，使用 Pod 的 ServiceAccount 进行身份验证。

audience 字段包含令牌所针对的受众：

- 收到令牌的主体必须使用令牌受众中所指定的某个标识符来标识自身，否则应该拒绝该令牌。
- 此字段是可选的，默认值为 API 服务器的标识。

字段 expirationSeconds 是服务账号令牌预期的生命期长度：

- 默认值为 1 小时，必须至少为 10 分钟（600 秒）。
- 管理员也可以通过设置 API 服务器的命令行参数 --service-account-max-token-expiration 来为其设置最大值上限。

path 字段给出与投射卷挂载点之间的相对路径。

**注意：**

- 以 subPath 形式使用投射卷源的容器无法收到对应卷源的更新。

## 与 SecurityContext 间的关系

请参考 [与 SecurityContext 间的关系](https://kubernetes.io/zh-cn/docs/concepts/storage/projected-volumes/#securitycontext-interactions)。
