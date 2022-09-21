# 为 Pod 配置服务账号

[TOC]

## 概览

Kubernetes 中有两种访问 API 服务器的方式：

- 当你（自然人）访问集群时（例如，使用 kubectl），API 服务器将你的身份验证为特定的用户账号（当前这通常是 admin，除非你的集群管理员已经定制了你的集群配置）。
- Pod 内的容器中的进程也可以与 API 服务器接触。当它们进行身份验证时，它们被验证为特定的 serviceaccount（例如，default）。

## 使用 default serviceaccount 访问 API 服务器

当你创建 Pod 时，如果没有指定 serviceaccount，Pod 会被指定给 namespace 中的 `default` 服务账号。

如果你查看 Pod 的原始 JSON 或 YAML（例如：`kubectl get pods <podname> -o yaml`），你可以看到 `spec.serviceAccountName` 字段已经被自动设置了。

```sh
$ kubectl get pods frontend-t2kvl -o yaml
...
serviceAccount: default
serviceAccountName: default
...
```

你可以使用自动挂载给 Pod 的服务账号凭据访问 API，[访问集群](https://kubernetes.io/zh-cn/docs/tasks/access-application-cluster/access-cluster/)中有相关描述。

serviceaccount 凭证是一个 JWT，默认挂载到：`/var/run/secrets/kubernetes.io/serviceaccount/token`。

```sh
$ kubectl get pods frontend-t2kvl -o yaml
...
serviceAccount: default
serviceAccountName: default
spec:
  containers:
  - image: nginx:1.14.2
    volumeMounts:
    - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      name: kube-api-access-mnnbg
      readOnly: true
volumes:
  - name: kube-api-access-mnnbg
    projected:
      defaultMode: 420
      sources:
      - serviceAccountToken:
          expirationSeconds: 3607
          path: token
      - configMap:
          items:
          - key: ca.crt
            path: ca.crt
          name: kube-root-ca.crt
      - downwardAPI:
          items:
          - fieldRef:
              apiVersion: v1
              fieldPath: metadata.namespace
            path: namespace
...

$ kubectl exec -it frontend-t2kvl -- /bin/bash
$ cat /var/run/secrets/kubernetes.io/serviceaccount/token
eyJhbGciOiJSUzI1NiIsImtpZCI6IlpJT3ZMWlVJWk9zTXFUTy02ZU1fY19ydEQwdHlBSDh5UVVmSmxvdDIyMVkifQ.eyJhdWQiOlsiaHR0cHM6Ly9rdWJlcm5ldGVzLmRlZmF1bHQuc3ZjLmNsdXN0ZXIubG9jYWwiXSwiZXhwIjoxNjk1MzAxNDk4LCJpYXQiOjE2NjM3NjU0OTgsImlzcyI6Imh0dHBzOi8va3ViZXJuZXRlcy5kZWZhdWx0LnN2Yy5jbHVzdGVyLmxvY2FsIiwia3ViZXJuZXRlcy5pbyI6eyJuYW1lc3BhY2UiOiJkZWZhdWx0IiwicG9kIjp7Im5hbWUiOiJmcm9udGVuZC10Mmt2bCIsInVpZCI6IjNmNWMxYWUwLTYyMTktNDJlMC1hMTVkLWYxOWI3YTM0MzcxNiJ9LCJzZXJ2aWNlYWNjb3VudCI6eyJuYW1lIjoiZGVmYXVsdCIsInVpZCI6Ijk3ZjBlNDA0LWZiNzctNDMyOS04NzY5LWNmYjdhMDEwM2VjYSJ9LCJ3YXJuYWZ0ZXIiOjE2NjM3NjkxMDV9LCJuYmYiOjE2NjM3NjU0OTgsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDpkZWZhdWx0OmRlZmF1bHQifQ.IYr4-BJYl5BufdDU8xXZvkhFAINOq615AAIeC_yF-min0aI7Zv4tPrgHZFWiimeuwr6QL7cYrxSNOklF25QIlRA5essz2DtQ73-C5jk92PbIR1iv0SEenCU3f_XT6cG2S5xj2PI3Sc9vjAOXFWZwLa9A9uXc4Z7DrwMCXNV8W5T3qYfln5l_zF8ykzy4Pj9pzIDnWZVJCmW7Ibaati-Dg4TtJKiY6C-bvvfG4H5RPTTFK-_HDDXgH8UQnnradLMW4wg4jwlDAOw-eC9s7oCJOEwVK5jqiIx-vc8ZoGlivFSHqB5KHupxHLZ_b3TUyAyq7NEKa7p3gp0_hpuw-9Vvng
```

上面的 jwt header：

```json
{
  "alg": "RS256",
  "kid": "ZIOvLZUIZOsMqTO-6eM_c_rtD0tyAH8yQUfJlot221Y"
}
```

jwt payload：

```json
{
  "aud": [
    "https://kubernetes.default.svc.cluster.local"
  ],
  "exp": 1695301498,
  "iat": 1663765498,
  "iss": "https://kubernetes.default.svc.cluster.local",
  "kubernetes.io": {
    "namespace": "default",
    "pod": {
      "name": "frontend-t2kvl",
      "uid": "3f5c1ae0-6219-42e0-a15d-f19b7a343716"
    },
    "serviceaccount": {
      "name": "default",
      "uid": "97f0e404-fb77-4329-8769-cfb7a0103eca"
    },
    "warnafter": 1663769105
  },
  "nbf": 1663765498,
  "sub": "system:serviceaccount:default:default"
}
```

**注意：**

- 因为不同 Pod 的 id 不同、name 不同，所以不同 Pod 分配到的 JWT 是不同的。

serviceaccount 的 API 许可取决于你所使用的鉴权策略。

你可以通过在 ServiceAccount 上设置 `automountServiceAccountToken: false` 来实现不给 serviceaccount 自动挂载 API 凭据到 `/var/run/secrets/kubernetes.io/serviceaccount/token` 的目的：

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: build-robot
automountServiceAccountToken: false
...
```
