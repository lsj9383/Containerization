# Docker CLI

<!-- TOC -->

- [Docker CLI](#docker-cli)
    - [Docker Info](#docker-info)
    - [Docker Run](#docker-run)
    - [Docker Stop](#docker-stop)
    - [Docker Kill](#docker-kill)
    - [Docker Exec](#docker-exec)
    - [Docker Search](#docker-search)
    - [Docker Container](#docker-container)
    - [Docker Inspect](#docker-inspect)
    - [Docker PS](#docker-ps)
    - [Docker Top](#docker-top)
    - [Docker Build](#docker-build)
    - [Docker RM](#docker-rm)
    - [Docker RMI](#docker-rmi)
    - [Docker Tag](#docker-tag)
    - [Docker Push](#docker-push)
    - [Docker Volume](#docker-volume)
    - [Docker Logs](#docker-logs)
    - [Docker Login](#docker-login)
    - [Docker logout](#docker-logout)
    - [常用命令](#常用命令)

<!-- /TOC -->

## Docker Info

显示 Docker 系统信息。

```sh
docker info [OPTIONS]
```

显示的信息包括：内核版本、容器数、镜像数等。可以指定显示格式。

OPTIONS | Default | Description
-|-|-
--format , -f | - | 使用给定的 Go 模板格式化输出。例如 `'{{json .}}'`

```sh
# 默认格式输出
docker info

# JSON 格式输出
docker info --format '{{json .}}'

# JSON 格式输出，并筛选出其中的 RegistryConfig 字段。
docker info --format '{{json .RegistryConfig}}'|python -m json.tool
```

## Docker Run

启动一个新的容器，并运行命令。

```sh
docker run [OPTIONS] IMAGE [COMMAND] [ARG...]
```

## Docker Stop

停止一个或多个运行中的容器。

```sh
docker stop [OPTIONS] CONTAINER [CONTAINER...]
```

容器内的主进程（pid == 1）将会收到 SIGTERM，若进程未结束会等待一段时间后发送 SIGKILL 强杀进程并退出。

OPTIONS | Default | Description
-|-|-
`--time , -t` | 10 ｜ 多少秒后对容器中的进程进行 kill 9（SIGKILL） 强杀。

## Docker Kill

Kill 一个或多个容器，Kill 可以理解为发送信号，也可以理解为杀死。类似 Linux 的 kill 命令。

```sh
docker kill [OPTIONS] CONTAINER [CONTAINER...]
```

`docker kill` 可以给一个或多个容器。容器内的主进程（pid == 1）将会收到 `SIGKILL` 信号（默认），或者通过 `--signal` 选项指定的信号。

OPTIONS | Default | Description
-|-|-
`--signal , -s` | `KILL` ｜ 发送给容器的信号。可以使用名称，也可以使用信号值。

```sh
docker kill --signal=SIGHUP my_container
docker kill --signal=HUP my_container
docker kill --signal=1 my_container
```

## Docker Exec

## Docker Search

在 *Docker Hub* 中搜索镜像。

**注意：**

- 仅支持在 Docker Hub 搜索镜像。

```sh
docker search [OPTIONS] TERM
```

OPTIONS | Default | Description
-|-|-
--filter , -f | - | 根据提供的条件过滤输出。
--format | - | 使用Go模板进行漂亮的打印搜索。
--limit | 25 | 最多搜索结果数。
--no-trunc | - | 不要截断输出。

```sh
docker search --filter=stars=3 --no-trunc busybox
```

## Docker Container

## Docker Inspect

## Docker PS

容器的 STATUS 包括：

- created，创建了容器但是并未启动。
  - `docker create` 的方式可以创建容器但不进行启动。
  - 容器创建后但是没有启动成功，例如无法 CMD 和 ENTRYPOINT 启动容器失败。（如果是 CMD shell 的方式，因为会先启动 shell 所以容器可以正常启动，即便失败，不会为 created 状态）。
- restarting，容器正在进行重启的状态。
- running，容器正常。
- removing，容器正在删除的状态。
- paused，容器暂停。
- exited，容器运行完成后退出的状态。
- dead，容器被删除，但是仅部分删除成功。

docker ps 可以通过 `--filter status=running` 的方式过滤出期望的状态。

## Docker Top

## Docker Build

## Docker RM

删除一个或多个容器。默认情况下不能删除正在运行的容器。

```sh
docker rm [OPTIONS] CONTAINER [CONTAINER...]
```

OPTIONS | Default | Description
-|-|-
--force , -f | 强制删除正在运行的容器（使用SIGKILL）。
--link , -l | 删除指定的链接。
--volumes , -v | 删除与容器关联的匿名卷。

```sh
# 删除所有停止的容器

docker rm $(docker ps -a -q)
```

## Docker RMI

删除一个或多个镜像。

```sh
docker rmi [OPTIONS] IMAGE [IMAGE...]
```

从主机删除镜像，如果镜像具有多个标签，则仅删除标签。若标签啥镜像的唯一标签，则镜像和标签一起删除。除非使用 `-f` 否则无法删除正在运行容器的镜像。

OPTIONS | Default | Description
-|-|-
`--force , -f` | - | 强制删除镜像，否则正在运行容器的镜像无法删除。
`--no-prune` | - | 不擅长没有标签的父对象。

**注意：**

- 删除镜像时，如果没有指定标签，则默认标签为 latest。

## Docker Tag

## Docker Push

## Docker Volume

## Docker Logs

## Docker Login

登录 Registry 服务器：

```sh
docker login [OPTIONS] [SERVER]
```

默认情况下 `docker login` 会用交互的方式来进行登录，但也可以通过 STDIN 提供登录密钥（使用STDIN可以防止密码出现在Shell的历史记录或日志文件中）：

```sh
cat ~/my_password.txt | docker login --username foo --password-stdin
```

docker login 登录成功后，会在 `$HOME/.docker/config.json` 生成对应服务的登录密钥信息，后续到对应 registry 的请求将会直接使用该密钥。

docker logout 退出后，会清理 `$HOME/.docker/config.json` 对应 registry 的密钥信息。

## Docker logout

## 常用命令
