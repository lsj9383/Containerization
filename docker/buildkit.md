# BuildKit

Docker 版本在 18.09 进行了镜像构建方面的增强，通过集成 BuildKit 用户可以感受到性能、存储管理以及安全性方面的改进。

- 旧版构建引擎使用的 Dockerfile 可以完全被 BuildKit 兼容。
- 可以通过 `--secret` 传递机密信息。

BuildKit 仅适用于 Linux 容器。

## 启动 BuildKit

开启 BuildKit 的方式非常简单，一般有以下三种方式：

- 设置环境变量 `export DOCKER_BUILDKIT=1`。
- 在运行 `docker build` 时指定环境变量 `DOCKER_BUILDKIT=1 docker build .`
- 在 Docker 配置文件（`/etc/docker/daemon.json`）中启动：

  ```json
  { "features": { "buildkit": true } }
  ```

## 启用新语法

需要通过覆盖 frontends 的方式启用 BuildKit 的新 Dockerfile 语法：

```dockerfile
# syntax = <frontend image>, e.g. # syntax = docker/dockerfile:1.0-experimental
```

## Docker build 机密信息

`docker run` 时可以传如新的 `--secret` 参数指定机密文件，在构建时会将机密文件传递到镜像构建中，最终的容器中不会存在该机密。

`--secret` 需要指定两个信息：

- `id`，机密的标识。
- `src`，机密文件。

通过命令进行显示使用方式：

```sh
docker run -t image-name --secret id=mysecret, src=mysecret.txt .
```

上述 `mysecret.txt` 保存了机密信息。

Dockerfile 在 RUN 指令中可以通过 `--mount` 命令引入机密文件：

```Dockerfile
# syntax = docker/dockerfile:1.0-experimental
FROM alpine

# shows secret from default secret location:
RUN --mount=type=secret,id=mysecret cat /run/secrets/mysecret

# shows secret from custom secret location:
RUN --mount=type=secret,id=mysecret,dst=/foobar cat /foobar
```

如果 `--mount` 没有指定 dst，则机密文件为  `/run/secrets/{id}`

**注意：**

- 该语法必须要开启 BuildKit
- 该语法必须要要覆盖 frontends: `# syntax = docker/dockerfile:1.0-experimental`

## 在构建时使用 SSH 访问私有数据
