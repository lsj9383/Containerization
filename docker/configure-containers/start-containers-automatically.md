# 容器自动启动

Docker 提供了重启策略来控制容器何时启动，例如可以在容器退出时或 Docker 启动时。启动策略也明确了相关容器的正确启动顺序。

Docker 推荐使用容器自动启动策略，避免人为的启动容器。

## 使用重启策略

在 `docker run` 时，可以使用 `--restart` 标识重启策略，取值如下：

Flag | 说明
-|-
no | 不进行容器重启。(the default)
on-failure | 容器以非 0 的 exit code 退出时，会进行重启（非 0 退出码认为时失败）。
always | 如果停止则总是重启容器。如果是手动停止容器，尽在 Docker daemon 重启时才重启。
unless-stopped | 与 always 相似，但是手动停止容器后，即便 Docker daemon 重启也不会重启。

```sh
# 启动容器时 指定重启策略
docker run -d --restart unless-stopped redis

# 如果容器已经启动 通过 docker update 修改重启策略
docker update --restart unless-stopped redis
```

## 重启策略细节

- 重启策略尽在容器成功启动后生效。容器成功启动意味着容器已经运行了 10 秒。这是为了避免根本无法启动的重启进入重启循环。
- 如果手动停止一个容器，则重启策略将会被忽略。这也是为了避免进入重启循环。

## 使用 Process Manager

如果重启策略不满足需求，可以使用 upstart, systemd, or supervisor 等工具控制重启。
