# 配置
<!-- TOC -->

- [配置](#配置)
    - [概览](#概览)
    - [Manifest 最佳实践](#manifest-最佳实践)
        - [容器镜像](#容器镜像)
        - [使用 kubectl](#使用-kubectl)
    - [ConfigMap](#configmap)
    - [Secret](#secret)
        - [内置 Secret](#内置-secret)
        - [自定义 Secret](#自定义-secret)

<!-- /TOC -->

## 概览

## Manifest 最佳实践

- 定义配置时，指定最新的稳定 API 版本。
- 使用 YAML 文件而不是 JSON 编写配置文件。
- 除非必要，否则不指定默认值：简单的最小配置会降低错误的可能性。
- 将相关的对象写在一个 YAML 文件中，通常单个文件比多个文件更容易管理（HELM 除外，HELM 本身就是一个包）。

### 容器镜像

imagePullPolicy 和镜像标签会影响 kubelet 何时尝试拉取指定的镜像。

- imagePullPolicy: IfNotPresent：仅当镜像在本地不存在时才被拉取。
- imagePullPolicy: Always：每次启动 Pod 的时候都会拉取镜像。
- imagePullPolicy 省略时，镜像标签为 :latest 或不存在，使用 Always 值。
- imagePullPolicy 省略时，指定镜像标签并且不是 :latest，使用 IfNotPresent 值。
- imagePullPolicy: Never：假设镜像已经存在本地，不会尝试拉取镜像。

### 使用 kubectl

- 使用 `kubectl apply -f <directory>`。 它在 `<directory>` 中的所有 .yaml、.yml 和 .json 文件中查找 Kubernetes 配置，并将其传递给 apply。
- 使用标签选择器进行 get 和 delete 操作，而不是特定的对象名称。
- 请参阅标签选择器和有效使用标签部分。
- 使用 `kubectl run` 和 `kubectl expose` 来快速创建单容器部署和服务。

## ConfigMap

ConfigMap 是一种 API 对象，用来将非机密性的数据保存到健值对中。使用时可以用作环境变量、命令行参数或者存储卷中的配置文件。

ConfigMap 通过 `.data` 字段来进行配置数据的存储。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: game-demo
data:
  # 类属性键；每一个键都映射到一个简单的值
  player_initial_lives: "3"
  ui_properties_file_name: "user-interface.properties"
  #
  # 类文件键
  game.properties: |
    enemy.types=aliens,monsters
    player.maximum-lives=5
  user-interface.properties: |
    color.good=purple
    color.bad=yellow
    allow.textmode=true
```

**注意：**

- 要在 Pods 中使用 Config，要求它们都在同一个 Namespace 中。
- kube 不会去区分单行的属性值，或者是多行的文件值（例如上述的 `player_initial_lives` 和 `game.properties`）。

设置后可以通过 `kubectl get configmap` 和 `kubectl describe configmap` 来显示相关信息：

```sh
# 获取 ConfigMap 对象列表
$ kubectl get configmap
NAME                              DATA   AGE
game-demo                         4      5s

# 获取配置详情
$ kubectl describe configmap game-demo
Name:         game-demo
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
player_initial_lives:
----
3
ui_properties_file_name:
----
user-interface.properties
user-interface.properties:
----
color.good=purple
color.bad=yellow
allow.textmode=true

game.properties:
----
enemy.types=aliens,monsters
player.maximum-lives=5

Events:  <none>
```

Pods 有四种方法获取 ConfigMap 中的值：

- 容器 entrypoint 的命令行参数。
- 容器的环境变量。
- 在只读卷里面添加一个文件，让应用来读取。
- 编写代码在 Pod 中运行，使用 Kubernetes API 来读取 ConfigMap。

下面描述了环境变量和卷的方式：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: configmap-demo-pod
spec:
  containers:
    - name: demo
      image: busybox
      command: ['sh', '-c', 'echo "Hello World!" && date && sleep 3600']
      env:
        # 定义环境变量
        - name: PLAYER_INITIAL_LIVES # 请注意这里和 ConfigMap 中的键名是不一样的
          valueFrom:
            configMapKeyRef:
              name: game-demo           # 这个值来自 ConfigMap
              key: player_initial_lives # 需要取值的键
        - name: UI_PROPERTIES_FILE_NAME
          valueFrom:
            configMapKeyRef:
              name: game-demo
              key: ui_properties_file_name
        - name: GAME_PROPERTIES
          valueFrom:
            configMapKeyRef:
              name: game-demo
              key: game.properties
      volumeMounts:
      - name: config
        mountPath: "/config"
        readOnly: true
  volumes:
    # 您可以在 Pod 级别设置卷，然后将其挂载到 Pod 内的容器中
    - name: config
      configMap:
        # 提供你想要挂载的 ConfigMap 的名字
        name: game-demo
```

- 对于挂载的访问配置文件的方式：会把 game-demo 中所有的 key 作为文件名挂载到 `/config/` 目录下。
  - `/config/player_initial_lives`
  - `/config/ui_properties_file_name`
  - `/config/game.properties`
  - `/config/user-interface.properties`
- 对于环境变量的方式：会把指定 key 的值写入指定的环境变量中。
- 对于文件 key 的多行值，不建议使用环境变量。虽然环境变量也会存在，也能获取到，但是不再有换行符号（测试后使用的是空格代替换行符）。

```sh
# 进入 pods
kubectl exec -it configmap-demo-pod /bin/sh
```

## Secret

Secret 对象类型用来保存敏感信息，例如密码、OAuth 令牌和 SSH 密钥。 将这些信息放在 secret 中比放在 Pod 的定义或者 容器镜像 中来说更加安全和灵活。

### 内置 Secret

在 kube 集群建立的时候，就自动生成了用于访问 kube API 的 Secret，并自动修改你的 Pod 以使用此类型的 Secret。该 Secret 为 ServiceAccount 服务。

```sh
$ kubectl get secret
NAME                  TYPE                                  DATA   AGE
default-token-f2hcv   kubernetes.io/service-account-token   3      9d
```

### 自定义 Secret

有两种方式创建 Secret，一个是通过 `kubectl create secret`，另一个是写 Secret 的 yaml 文件。

**kubectl create secret**

```sh
# 创建本例中要使用的文件
$ echo -n 'admin' > ./username.txt
$ echo -n '1f2d1e2e67df' > ./password.txt

$ kubectl create secret generic db-user-pass \
--from-file=username=./username.txt \
--from-file=./password.txt

secret "db-user-pass" created
```

**注意：**

Secret 的每个 key 都拥有一个名字，默认情况下为文件名，可以通过 `--from-file=<key>=<filepath>` 进行指定。

**manifest**

Secret 和 ConfigMap 不同，`.data` 中的 value 值需要是 base64 编码。

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
data:
  username: YWRtaW4=
  password: MWYyZDFlMmU2N2Rm
```

使用 base64 编码可能并不方便，因为每次都得对值做一次编码。可以使用 `.stringData` 代替 `.data`，该字段写入值时使用 bae64，并在生成 secret 时自行对值进行 base64 处理。

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
data:
  password: MWYyZDFlMmU2N2Rm
stringData:
  username: administrator
```

普通的 `kubectl describe secret` 是无法看到对应信息的，但可以通过 `kubectl get secret -o yaml` 查到：

```sh
$ kubectl get secret mysecret -o yaml
apiVersion: v1
data:
  password: MWYyZDFlMmU2N2Rm
  username: YWRtaW5pc3RyYXRvcg==
...
```

可以通过挂载和环境的变量方式在 Pods 中使用 Secret。

对于挂载的方式：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-test-pod
spec:
  containers:
  - name: mypod
    image: busybox
    command: ['sh', '-c', 'echo "Hello World!" && date && sleep 3600']
    volumeMounts:
    - name: foo
      mountPath: "/etc/foo"
      readOnly: true
  volumes:
  - name: foo
    secret:
      secretName: mysecret
```
