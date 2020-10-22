# Docker Reference

## Usage

通过 `docker build` 命令，可以从 Dockerfile 和上下文中构建镜像。上下文的 PATH 可以时本地目录，也可以时一个 Git 仓库 URL。

在镜像构建时，会递归地将目录中的所有子目录文件、Git 的所有子模块，都传递给 Docker 作为上下文构建镜像。

`docker build` 触发构建时在 Docker CLI 上进行的，并将上下文打包传递给 Docker Daemon 进行构建。可以通过 `.dockerignore` 指定忽略的文件。

构建时，可以通过 `-t` 选项指定镜像的名称（包括注册服务器、命名空间、镜像名、镜像标签），`-f` 选项指定 Dockerfile 路径。

```sh
docker build -f /path/to/a/Dockerfile .

docker build -t shykes/myapp .

$ docker build -t shykes/myapp:1.0.2 -t shykes/myapp:latest .
```

Docker 将尽可能重用中间映像（缓存），以显着加速 Docker 构建过程。

## BuildKit

## Format

## Syntax

## Escape

## 环境变量替换

## `.dockerignore`

## FROM

```dockerfile
FROM [--platform=<platform>] <image> [AS <name>]
```

```dockerfile
FROM [--platform=<platform>] <image>[:<tag>] [AS <name>]
```

```dockerfile
FROM [--platform=<platform>] <image>[@<digest>] [AS <name>]
```

FROM 指令初始化一个新的构建阶段，并为后续的指令指定一个基础镜像。一个有效的 Dockerfile 必须以 FROM 命令开头（ARG 除外）。

- FROM 可以在 Dockerfile 中出现多次，用于构建多个阶段，或构建一个另一个镜像的依赖。
- AS 指定了构建阶段名称的名字。
- `--platform` 指定用于构建镜像的平台（如 linux/amd64, linux/arm64, windows/amd64），默认情况下使用和构建请求的平台。

FROM 指令前可以通过 ARG 指令声明变量，并在 FROM 指令中使用这些变量：

```dockerfile
ARG  CODE_VERSION=latest
FROM base:${CODE_VERSION}
CMD  /code/run-app
```

FROM 前面的 ARG 位于构建阶段外，因此构建中是不能使用的，若需要在构建后使用，则需要在 FROM 后再声明一次该 ARG 且不带默认值。

```dockerfile
ARG VERSION=latest
FROM busybox:$VERSION
ARG VERSION
RUN echo $VERSION > image_version
```

## RUN

RUN 命令有两种形式

- `RUN <command>` (shell)
- `RUN ["executable", "param1", "param2"]` (exec)

## CMD

## LABEL

## EXPOSE

```sh
EXPOSE <port> [<port>/<protocol>...]
```

EXPOSE 指令声明容器会使用的网络端口，可以指定时 TCP 还是 UDP 端口，若没指定 则默认使用 TCP。

EXPOSE 只是一个声明，更类似一个文档，在容器运行时，需要指定和 EXPOSE 端口之间的映射。如果 docker run 时没有通过 -p 参数进行端口映射，则会生成一个随机的对外端口。

默认情况下，EXPOSE 假定为 TCP。您还可以指定UDP：

```sh
EXPOSE 80/udp
```

也可以同时在 TCP 和 UDP 上：

```sh
EXPOSE 80/tcp
EXPOSE 80/udp
```

## ENV

## ADD

## COPY

## ENTRYPOINT

## VOLUME

```sh
VOLUME ["/data"]
```

通常会将容器写磁盘的目录声明为 VOLUME：

- 一方面是避免操作 Writable 层。
- 另一方面是为了告诉使用者该目录有文件输出，需要进行挂载。
- 再一方面是如果没有显式挂载，则会挂载一个匿名卷，且容器结束声明周期时移除（若没有其他容器再引用该匿名卷）。

## USER

```sh
USER <user>[:<group>]
```

或者

```sh
USER <UID>[:<GID>]
```

当未指定 USER 时，将会以 root:root 的用户和组运行指令和容器。

对于不存在的用户和组，需要在使用 USER 指令前新建：

```dockerfile
RUN groupadd -r usergroup && useradd --no-log-init -r -g usergroup -m user00
```

在 USER 后，RUN 命令将以对应身份运行，但是 COPY 指令仍然是 root 身份。

因为通常需要 root 权限进行镜像的构建，有一种常用模式：

- 创建进程运行的用户和组。
- 用 root 身份 build。
- 启动进程时使用 gosu 修改用户身份为对应的用户（通过 gosu 启动的进程，虽然用户被修改了，但是实际权限和 root 等价）。

除上述方式外，就是自己建立个目录，再修改权限、所属。

## WORKDIR

## ARG

## STOPSIGNAL

```sh
STOPSIGNAL signal
```

STOPSIGNAL 指令设置在容器停止时，将被发送个容器 1 号进程的停止信号。signal 可以使用数字，也可以使用信号名称。

## HEALTHCHECK

## SHELL

## 实验性的 Features

## Dockerfile 示例
