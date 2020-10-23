# Docker Reference

<!-- TOC -->

- [Docker Reference](#docker-reference)
    - [Usage](#usage)
    - [Format](#format)
    - [Syntax](#syntax)
    - [Escape](#escape)
    - [环境变量替换](#环境变量替换)
    - [`.dockerignore`](#dockerignore)
    - [FROM](#from)
    - [RUN](#run)
    - [CMD](#cmd)
    - [ENTRYPOINT](#entrypoint)
    - [LABEL](#label)
    - [EXPOSE](#expose)
    - [ENV](#env)
    - [ADD](#add)
    - [COPY](#copy)
    - [VOLUME](#volume)
    - [USER](#user)
    - [WORKDIR](#workdir)
    - [ARG](#arg)
    - [STOPSIGNAL](#stopsignal)
    - [HEALTHCHECK](#healthcheck)
    - [SHELL](#shell)
    - [实验性的 Features](#实验性的-features)
    - [Dockerfile 示例](#dockerfile-示例)

<!-- /TOC -->

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

CMD 的主要目的是执行容器进程提供默认命令和参数。CMD 指令有三种形式：

```dockerfile
# exec form, this is the preferred form
CMD ["executable","param1","param2"]

# as default parameters to ENTRYPOINT
CMD ["param1","param2"]

# shell form
CMD command param1 param2
```

每个 Dockerfile 应该只有一个 CMD，如果存在多个 CMD，则最后一个 CMD 有效。`docker run` 可以覆盖 CMD 的默认参数。

CMD 提供的默认值可以包含一个可执行文件，也可以不包含（此时 ENTRYPOINT 提供可执行文件）。

shell 模式和 exec 模式最大的区别在于 exec 模式不是 shell 命令后，无法进行变量替换。

**注意：**

- 因为 CMD 是容器启动执行的命令，而 ARG 参数只是镜像构建使用，因此无法将 ARG 作为 CMD 的一部分（ENV 可以，因为环境变量在容器启动时存在）。
- 如果又想使用 exec form，又想使用 shell 变量替换，则可以使用 `CMD ["/bin/sh", "-c", "echo $HOME"]`。

## ENTRYPOINT

该命令有两种形式:

```dockerfile
# exec form
ENTRYPOINT ["executable", "param1", "param2"]

# shell form
ENTRYPOINT command param1 param2
```

## LABEL

```dockerfile
LABEL <key>=<value> <key>=<value> <key>=<value> ...
```

LABEL 指定可以给镜像添加元数据，且提供 keyvalue 对。可以在一行中给出多个 LABEL，也可以通过多个 LABEL 给出，两者的效果是一致的（在 Docker 1.10 前，多个 LABEL 会增加镜像大小）。

如果需要查看镜像 LABEL，可以使用 `docker inspect $image`。

## EXPOSE

```sh
EXPOSE <port> [<port>/<protocol>...]
```

EXPOSE 指令声明容器会使用的网络端口，可以指定时 TCP 还是 UDP 端口，若没指定 则默认使用 TCP。

EXPOSE 只是一个声明，更类似一个文档，在容器运行时，需要指定和 EXPOSE 端口之间的映射。如果 docker run 时没有通过 -p 参数进行端口映射，则会生成一个随机的对外端口。

默认情况下，EXPOSE 假定为 TCP。您还可以指定UDP：

```dockerfile
EXPOSE 80/udp
```

也可以同时在 TCP 和 UDP 上：

```dockerfile
EXPOSE 80/tcp
EXPOSE 80/udp
```

## ENV

```sh
ENV <key>=<value> ...
```

ENV 指令设置环境变量 `key=value`，并在随后的构建指令中可以使用并进行内联替换。

ENV 语法支持一次设置多个环境变量，但其实和多个 ENV 设置多个环境变量的方式没有任何区别，采用多个 ENV 的方式可读性更好。

```dockerfile
# Method 1
ENV K1=V1 K2=V2

# Method 2
ENV K1=V1
ENV K2=V2
```

ENV 不但支持 `ENV K=V` 也支持 `ENV K V`，省略了 `=`，但是并不推荐使用省略等号的语法，因为可能会导致意义混淆，可读性差，例如：

```dockerfile
ENV ONE TWO= THREE=world
```

通过 ENV 设置环境变量，在镜像启动时也会存在（可以通过 `docker run --env k=v` 进行覆盖）。若仅需要在构建镜像时使用变量，请使用 `ARG`。

## ADD

该命令有两种形式：

```dockerfile
ADD [--chown=<user>:<group>] <src>... <dest>
ADD [--chown=<user>:<group>] ["<src>",... "<dest>"]
```

如果路径中包含空格，推荐第二种形式。

**注意：**

- chown 功能尽在 Linux 的构建机上支持，Windows 上不支持。

ADD 命令中的 `src` 指定文件、目录以及远程 URL，并将其复制到 `dest` 中。

`dest` 路径是绝对路径，或者是相对于 WORKDIR 的相对路径。

`--chown` 选项，可以指定文件复制后的所有权，若没有指定则以 USER 指令的用户作为所有权。如果 chown 的用户不存在，则添加会失败。

```dockerfile
ADD --chown=55:mygroup files* /somedir/
ADD --chown=bin files* /somedir/
ADD --chown=1 files* /somedir/
ADD --chown=10:11 files* /somedir/
```

如果 src 是远程 URL，且需要认证，则只能使用 RUN 和 wget / curl 的方式获取文件，因为 ADD 指令不支持认证相关配置。

如果 src 是 URL，则 ADD 到容器中的文件权限为 600。如果 src 是本地文件或者目录，则其权限和本地完全一致（在 USER 对应的用户下）。

如果 src 采用了工人压缩方式（gzip / bzip / xy / identity） 则将其解压为目录。

## COPY

该指令有两种形式：

```dockerfile
COPY [--chown=<user>:<group>] <src>... <dest>
COPY [--chown=<user>:<group>] ["<src>",... "<dest>"]
```

src 只能是上下文中的目录/文件，且不能是远程资源，不能自动解压，其他特性和 ADD 基本一致。

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

在 USER 后，命令将以对应身份运行。因为通常需要 root 权限进行镜像的构建，有一种常用模式：

- 创建进程运行的用户和组。
- 用 root 身份 build。
- 启动进程时使用 gosu 修改用户身份为对应的用户（通过 gosu 启动的进程，虽然用户被修改了，但是实际权限和 root 等价）。

除上述方式外，就是自己建立个目录，再修改权限和所属。

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
