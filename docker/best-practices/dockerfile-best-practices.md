# Dockerfile 最佳实践

<!-- TOC -->

- [Dockerfile 最佳实践](#dockerfile-最佳实践)
    - [概览](#概览)
    - [创建短暂的容器](#创建短暂的容器)
    - [构建上下文](#构建上下文)
    - [通过 stdin 输入 Dockerfile 的场景](#通过-stdin-输入-dockerfile-的场景)
    - [.dockerignore](#dockerignore)
    - [多阶段构建](#多阶段构建)
    - [仅安装需要的依赖](#仅安装需要的依赖)
    - [解耦应用](#解耦应用)
    - [镜像层数最小化](#镜像层数最小化)
    - [整理多行参数](#整理多行参数)
    - [利用构建缓存](#利用构建缓存)
    - [Docker 指令](#docker-指令)
        - [FROM](#from)
        - [LABEL](#label)
        - [RUN](#run)
        - [CMD](#cmd)
        - [EXPOSE](#expose)
        - [ENV](#env)
        - [ADD & COPY](#add--copy)
        - [ENTRYPOINT](#entrypoint)
        - [VOLUME](#volume)
        - [USER](#user)
        - [WORKDIR](#workdir)
        - [ONBUILD](#onbuild)
        - [STOPSIGNAL](#stopsignal)
        - [VOLUME](#volume-1)
    - [参考文献](#参考文献)

<!-- /TOC -->

## 概览

一个 Docker 镜像由 readonly layers 组成，每一个指令都将会生成一个 readonly layers，相邻的两层之间是增量的关系。

当你运行一个镜像，生成一个容器时，会在顶部生成一个可写层，容器中的所有 IO 变更都是对可写层的操作。

## 创建短暂的容器

通过镜像生成的容器应该是“短暂”的，“短暂”的含义是容器可以随时启动和关闭，并可以随便的重建和替换。

## 构建上下文

当运行 `docker build` 时，当前的工作目录即构建上下文，默认情况下 Dockerfile 应该位于构建上下文中，也可以通过 `-f` 指定 Dockerfile 的位置。

构建上下文中的所有文件都会全部打包发送给 `Docker Daemon` 进行处理。

```sh
docker build -t ${tag} -f ${dockerfile} ${context}
```

## 通过 stdin 输入 Dockerfile 的场景

## .dockerignore

为了减少构建上下文的传输大小，加快镜像的构建时间，可以在 `.dockerignore` 文件中将与构建无关的文件剔除。

## 多阶段构建

多阶段构建可以讲镜像的构建环境和运行环境分离，减少最终镜像的大小。

多阶段中，可以利用缓存将已经构建好的层直接用来使用，而不必每次都进行构建。根据使用的频率，将层进行排序，以最大程度利用缓存：

- 安装构建应用程序所需要的工具。
- 安装应用程序依赖的库。
- 生成应用程序镜像。

```dockerfile
# 第一阶段 构建镜像所需要的环境

FROM build-iamge AS build

RUN install && build

# 第二阶段 将生成的包和依赖复制到运行环境

FROM run-image

COPY --from=build /bin/from /bin/target

ENTRYPOINT ["/bin/target"]
CMD ["--help"]
```

## 仅安装需要的依赖

为了减少复杂性，简化依赖，镜像大小，构建时间，仅安装需要的依赖。绝对不能因为仅仅可能使用，而将不必要的依赖进行安装。

## 解耦应用

一个容器应该只有一个关注点，这类似于面向对象里面的单一职责，这便于进行水平扩容和重复使用。例如，一个 Web 应用可以进行拆分为如下三个功能的容器（当然，这些容器各自都有对应的镜像）：

- Web 应用进程用一个容器。
- Nginx 接入层的容器。
- MySQL 存储层的容器。

每个容器限制为一个进程是一个较好的经验法则，但是并不是强制和一成不变的，例如：

- Nginx 的 master-worker 多进程工作的方式。
- 某些镜像在容器化时会启动一个初始进程，该进程可以辅助生成实际的工作进程。

## 镜像层数最小化

在旧版本的 Docker 中，最重要的是通过减少层来确保性能，因此通过以下的特征来最小化层数：

- 只有 RUN, COPY, ADD 指令会增加层，其他指令仅生成临时层，最终的构建不受影响。
- 使用多阶段构建。

## 整理多行参数

通过将参数进行字符串排序进行传参数，有助于参数列表的更新，并避免重复。

## 利用构建缓存

Dockerfile 中的指令按顺序执行，在构建镜像时，Docker 会检查每条指令对应的缓存镜像是否存在，若存在则直接使在该缓存基础上构建。

若不希望使用缓存，可以在 docker build 时，指定 `--no-cache=true`。

缓存镜像的查询策略：

当某个指令的缓存无效后，后续的缓存均无效，都会进行重新构建。

## Docker 指令

构建一个高效易维护的镜像，通常会涉及如下指令。

### FROM

尽可能的使用在官方提供的镜像基础上进行构建，对于 Linux 可以使用 Alpine 镜像，该镜像大小受到严格控制（5 MB），并且仍然是 Linux 发行版。

### LABEL

可以通过 LABEL 给镜像添加多个标签，以便于组织镜像。

label 中若带空格，必须使用双引号，或者使用转义符。

```dockerfile
LABEL vendor1="ACME Incorporated"
LABEL vendor2=ZENITH\ Incorporated
```

在 Docker 1.10 前，若一个镜像有多个标签，则可以将其写在一个 label 中，每个标签用空格隔开。这样可以避免每个 LABEL 指令都生成一层。

```dockerfile
LABEL vendor=ACME\ Incorporated \
      com.example.is-beta= \
      com.example.is-production="" \
      com.example.version="0.0.1-beta" \
      com.example.release-date="2015-02-12"
```

现在版本的 Docker 因为 LABEL 不会给镜像新增层，所以不需要这样写了。

### RUN

将长并复杂的 RUN 语句进行拆分，通过 `\` 将多条语句分割为多行，让 Dockerfile 更易读、理解和维护。

在写多行命令时，可以使用 `&&` 进行连接，也可以使用 `;` 进行连接，如果使用 `&&` 连接，通常将其放在行首。使用 `;` 连接，则放在行尾。

- Redis Dockerfile

  ```dockerfile
  RUN set -eux; \
      savedAptMark="$(apt-mark showmanual)"; \
      apt-get update; \
      ...
  ```

- Nginx Dockerfile

  ```dockerfile
  RUN set -x \
      && addgroup --system --gid 101 nginx \
      && ...
  ```

**APT-GET**

在 RUN 中最常用的命令可能是 `apt-get` 因为该命令用于安装软件，该命令在 Dockerfile 的使用中存在一些需要避免的陷阱。

- 避免使用 `apt-get upgrade` 和 `dist-upgrade`，该命令用于更新过期的软件。在一些父镜像中，某些软件不应该被更新，有些甚至也没有权限更新。如果发现了软件过期，更好的做法是告知镜像作者。
- `apt-get update`（仅更新本地的软件版本列表，不会主动去更新软件） 和 `apt-get install` 应该在同一个 RUN 中使用，避免拆为多行，因为这可能导致后期添加新的安装软件时，由于镜像缓存的缘故，导致 `apt-get update` 没有执行，进而安装了过期版本的软件。

  ```dockerfile
  FROM ubuntu:18.04
  RUN apt-get update
  RUN apt-get install -y curl
  ```

- 考虑在安装时指定软件版本号。
- 安装完成后，清理 apt 缓存，缩小镜像，`rm -rf /var/lib/apt/lists/*`。如果使用 `Debian` 和 `Ubuntu` 的官方镜像，会自动清理 apt 缓存。

这是一个完整的 apt-get 示例：

```dockerfile
RUN apt-get update && apt-get install -y \
    aufs-tools \
    automake \
    build-essential \
    curl \
    dpkg-sig \
    libcap-dev \
    libsqlite3-dev \
    mercurial \
    reprepro \
    ruby1.9.1 \
    ruby1.9.1-dev \
    s3cmd=1.1.* \
 && rm -rf /var/lib/apt/lists/*
```

**USING PIPES**

有些命令会使用管道，但是如果管道的前一个指令失败，后一个指令仍然会继续执行，例如：

```dockerfile
RUN wget -O - https://some.site | wc -l > /number
```

如果请求 `https://some.site` 失败，并不会影响 `wc -l` 的影响， 最终也会顺利生成镜像， 然而这个镜像是错误的。

为了避免管道前的命令失败，导致错误镜像的生成，可以使用 `set -o` 进行阻止，例如：

```dockerfile
RUN set -o pipefail && wget -O - https://some.site | wc -l > /number
```

虽然官网说用 `set -o`，但是看很多 Dockerfile 都会使用 `set -e` 进行阻止，该选项在命令失败时会直接关闭 shell。

**SET**

看了一些 Dockerfile 后，在 RUN 前往往会执行 set 指令，用于设置 shell 的运行方式，常见的有：

- `RUN set -x`
- `RUN set -ex`
- `RUN set -eux`

这些 set 参数作用：

- `-x`, 执行指令后，会先显示该指令及所下的参数。
- `-e`, 若指令传回值不等于0，则立即退出shell。很明显，通常使用这个时一般都有管道命令，这可以及时停止生成镜像。
- `-u`, 当执行时使用到未定义过的变量，则显示错误信息。

### CMD

CMD 用于运行镜像容器化时，运行其中的进程。`docker run` 后的参数将会覆盖 CMD 命令的内容。

- 对于后台服务场景，CMD 通常总会使用形式 `CMD ["executable", "param1", "param2"…]`。因此对于基于 Apache 服务的镜像，可以使用命令 `CMD ["apache2","-DFOREGROUND"]`。实际上这也是基于服务的镜像推荐的使用方式。
- 在终端场景，CMD 需要给出交互式终端，例如 `CMD ["python"]`，使用这样的形式意味着在 docker run 时必须指定交互外壳 `docker run -it image-name`。
- 在命令参数场景，CMD 中仅仅是命令参数，而命令由 ENTRYPOINT 指出。这样的形式方便通过 `docker run` 传递参数（因为会覆盖 CMD）。

### EXPOSE

EXPOSE 用于指定镜像运行时会监听的端口，例如 Web 服务经常使用 `EXPOSE 80` 和 `EXPOSE 443`。

为了容器外访问，`docker run` 时会指定端口映射。

### ENV

ENV 可以设置镜像构建中和容器运行中的环境变量，通常有以下场景：

- 设置软件运行的路径，简化 CMD 或 ENTRYPOINT 的命令（避免输入全路径，但实际上还是建议使用全路径）。
  - 例如，对于 nginx 镜像，`ENV PATH=/usr/local/nginx/bin:$PATH`，运行命令仅需要 `CMD ["nginx"]`
- ENV 也方便指定相关依赖或者应用本身所需要的环境变量。
- ENV 也通常用于设置版本号信息。

ENV 会给镜像添加一层，即便通过 RUN 的 unset 取消环境变量，但是实际上环境变量仍然存在。

### ADD & COPY

COPY 仅支持上下文中的文件复制到镜像中，而 ADD 还能复制远程文件，以及将 tar 文件复制到镜像中后自动解压。

所以，即便 ADD 和 COPY 很类似，但是我们通常选择 COPY，因为 COPY 的功能更简单，语义更直接。

如果在 Dockerfile 的不同步骤中使用了不同的上下文文件，则可以考虑将 COPY 进行拆分，将稳定的文件排在前方，以便最大的利用构建缓存。但是这有个缺点，因为多个 COPY 会导致镜像层数增多，体积增大。在实际使用过程中应综合衡量 build 效率和镜像大小后作出决定。

参考下述文件，因为依赖文件的变更会触发将所有的包进行检查和安装，因此依赖文件应该单独拎出来进行 COPY。若将 tmp 目录直接 COPY，会导致非依赖文件的变更都触发依赖包的检查和安装。

```dockerfile
COPY requirements.txt /tmp/
RUN pip install --requirement /tmp/requirements.txt
COPY . /tmp/
```

应该尽量避免通过 ADD 到远程获取解压包，因为这些解压包最后都会删掉，导致镜像中包含了 ADD 和 RUN 两层。更好的解决方法是在 RUN 中使用 wget 或 curl 命令获得文件，并在 RUN 层中删除。

*错误：*

```dockerfile
ADD https://example.com/big.tar.xz /usr/src/things/
RUN tar -xJf /usr/src/things/big.tar.xz -C /usr/src/things
RUN make -C /usr/src/things all
```

*正确：*

```dockerfile
RUN mkdir -p /usr/src/things && \
    curl -SL https://example.com/big.tar.xz | tar -xJC /usr/src/things && \
    make -C /usr/src/things all
```

### ENTRYPOINT

该命令的最佳用法是用于设置主进程运行命令，使得该镜像想命令一样运行，并可以通过 CMD 传递参数。如果没有指定 ENTRYPOINT，则是将 CMD 作为进程启动命令。

```dockerfile
ENTRYPOINT ["s3cmd"]
CMD ["--help"]
```

可以直接运行镜像。

```sh
docker run s3cmd

docker run s3cmd ls s3://mybucket
```

实际上，大多数著名的应用 Dockerfile 中，ENTRYPOINT 通常会和辅助脚本一起使用，并且辅助脚本名字统一为 `docker-entrypoint.sh`。

辅助脚本在处理完毕后，最后通常使用 `exec "$@"` 或 `bin-name "$@"`，将启动真正工作进程（`$@` 得到辅助脚本的所有输入参数），并且工作进程的 PID 将为 1。

下面这是 Redis 的 `docker-entrypoint.sh`。

```sh
#!/bin/sh
set -e

# first arg is `-f` or `--some-option`
# or first arg is `something.conf`
if [ "${1#-}" != "$1" ] || [ "${1%.conf}" != "$1" ]; then
  set -- redis-server "$@"
fi

# allow the container to be started with `--user`
if [ "$1" = 'redis-server' -a "$(id -u)" = '0' ]; then
  find . \! -user redis -exec chown redis '{}' +
  exec gosu redis "$0" "$@"
fi

exec "$@"
```

Redis 的 Dockerfile 的最后：

```dockerfile
# ...

COPY docker-entrypoint.sh /usr/local/bin/
ENTRYPOINT ["docker-entrypoint.sh"]

EXPOSE 6379
CMD ["redis-server"]
```

### VOLUME

VOLUME 用于暴露数据存储区，强烈建议使用 VOLUME 而不是直接 mounts，因为使用 VOLUME 方便进行查阅信息，而 mounts 无法查询使用信息。

### USER

USER 命令可以设置工作的用户。如果应用不需要 root 特权，则一定要给创建一个用户，并通过 USER 切换为该用户。

如果每次构建镜像都新生成用户 id 和组 id，会分配新的值，可以考虑分配一个固定的用户 id 和组 id。

为减少层次和复杂性，请避免频繁来回切换USER。

### WORKDIR

为了清晰和可靠，WORKDIR 设置工作目录一定要使用绝对路径。

避免使用  `RUN cd…&& do-something` 这样的命令，因为这难以阅读和排查问题。

### ONBUILD

### STOPSIGNAL

在 `docker stop` 停止容器时，向容器中的进程 PID 为 1 的发送的停止信号，STOPSITNAL 则是指定停止信号是什么。如果十秒后，进程仍未关闭，docker 会发送 SIGKILL 进行强制关闭。

默认使用的是 SIGTERM(15)，通常为了强调，我们在 Dockerfile 中也会写明 `STOPSIGNAL SIGTERM`，因为 SIGTERM 通常用于优雅的关闭进程。

### VOLUME

容器中应避免对存储层进行写操作，如果存在写操作，需要对其目录声明为 VOLUME。

```dockerfile
VOLUME /data
```

VOLUME 声明容器中需要进行挂载的卷，并且在 docker run 时，若没有认为指定和宿主机的挂载映射关系，则会自动在宿主机挂载为匿名卷。

当然，运行时可以覆盖这个匿名挂载设置。

```sh
docker run -d -v mydata:/data xxxx
```

## 参考文献

- [Best practices for writing Dockerfiles](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/#entrypoint)
