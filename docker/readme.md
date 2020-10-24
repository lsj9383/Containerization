# Docker

## 安装

## 配置

## 速记

```sh
# 登录 Docker Hub
docker login

# 登录指定的 Docker 注册服务器
docker login authority-name

# 构建 docker
docker build -t image-name -f dockerfile-path --no-cache .

# 前台运行容器，并伪造 TTY，保有输入输出
docker run -it --rm image-name

# 后台运行容器
docker run -d image-name
```

## Next

- [Docker 开发最佳实践]()
- [Dockerfile 最佳实践]()
- [最佳实践总结]()
- [容器配置]()
  - [容器自动启动]()
- [Reference]()
  - [Dockerfile Reference]()
  - [Docker CLI Reference]()
