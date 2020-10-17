# Docker 开发最佳实践

## 如何保持尽量小的镜像

小的镜像可以更快的从网络中获取到，并更快的加载到内存中。

- 使用一个合适的基础镜像。
- 使用多阶段构建，将构建环境镜像和运行环境镜像独立开。
- 尽量减少镜像的层数。
- 如果有很多共同特定的镜像，将起抽象出一个 base image，并让所有镜像基于该 base image 进行构建。
- 可以将 Production 镜像作为基础镜像，来构造调试用的镜像。
- 构建镜像时，使用有效的标签进行标记，可以包括版本信息、环境信息（prod test）、稳定性等。绝对不能依赖于自动创建的 `latest` 标签。

## 在何处以及如何持久化应用数据

- 避免将应用数据存储在容器的可读写层，这不但会丢失数据，也会增加容器的大小，效率更是不如 volumes 和 mounts。
- 存储数据应该使用 volumes。
- 在开发时可以使用 mounts，但是在 Production 环境使用 volumes 代替。 <!-- TODO(lu) 不知道为什么 -->
- 对于 Production 使用 secret 存储敏感数据，使用 config 存储非敏感数据。

## 使用 CI/CD 进行测试和开发

## Development 和 Production 的不同

## 参考文献

- [Docker development best practices](https://docs.docker.com/develop/dev-best-practices/)
