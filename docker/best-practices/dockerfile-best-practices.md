# Dockerfile 最佳实践

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
