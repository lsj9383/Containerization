# 容器

## 概览

### 容器镜像

容器镜像是一个随时可以运行的软件包， 包含运行应用程序所需的一切：代码和它需要的所有运行时、应用程序和系统库，以及一些基本设置的默认值。

根据设计，容器是不可变的：你不能更改已经运行的容器的代码。 如果有一个容器化的应用程序需要修改，则需要构建包含更改的新镜像，然后再基于新构建的镜像重新运行容器。

### 容器运行时

Kubernetes 支持多个容器运行环境：`docker`、`containerd`、`CRI-O` 以及任何实现了 Kubernetes CRI 接口的环境。

## 镜像

### 镜像名称

如果没有指定仓库的名称，会默认使用 Docker Hub 仓库里面的镜像。

如果没有指定标签，会默认使用 latest。

### 更新镜像

默认的镜像拉取策略是 `IfNotPresent`。镜像如果已经在节点中存在，则不会去拉取镜像。如果希望强制重新拉取镜像，可以使用以下方式之一：

- 设置容器的 imagePullPolicy 为 Always。
- 省略 imagePullPolicy，并使用 :latest 作为要使用的镜像的标签。
- 省略 imagePullPolicy 和要使用的镜像标签。
- 启用 AlwaysPullImages 准入控制器（Admission Controller）。

### 配置 Node 访问私有 Registry

Docker 将私有仓库的密钥保存在 $HOME/.dockercfg 或 $HOME/.docker/config.json 文件中。

可以先通过 docker login 生成机密信息，再分发给各个节点。

### 提前拉取镜像

默认情况下，kubelet 会尝试从指定的仓库拉取每个镜像。 如果容器属性 imagePullPolicy 设置为 IfNotPresent 或者 Never， 则会优先使用（对应 IfNotPresent）或者一定使用（对应 Never）本地镜像。

这一方案可以用来提前载入指定的镜像以提高速度，或者作为向私有仓库执行身份认证的一种替代方案。

### 创建 Registry 密钥

```sh
kubectl create secret docker-registry <名称> \
  --docker-server=DOCKER_REGISTRY_SERVER \
  --docker-username=DOCKER_USER \
  --docker-password=DOCKER_PASSWORD \
  --docker-email=DOCKER_EMAIL
```

在创建完 Registry 密钥后，就可以在 yaml 文件中指定使用的密钥：

```sh
cat <<EOF > pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: foo
  namespace: awesomeapps
spec:
  containers:
    - name: foo
      image: janedoe/awesomeapp:v1
  imagePullSecrets:
    - name: myregistrykey
EOF
```

## 容器环境

kube 的容器环境给容器提供了几个重要的资源：

- 一个文件系统，包含了镜像和多个 VOLUME。
- 容器自身的信息。
- 集群中其他对象的信息。
