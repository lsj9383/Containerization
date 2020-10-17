# 容器日志

通过命令 `docker log ${container-name}` 可以获取容器日志，容器日志本质上就是容器 STDOUT 和 STDERR 的输出。

在一些场景下，`docker log` 并不能输出有用的信息：

- 如果你使用 `logging driver` 将日志输出到文件、其他主机、数据库等，STDOUT 和 STDERR 可能不存在有效输出。
- 容器是 Web Server 或者 Database 等无交互进程，这些应用可能将日志输出到文件而不是 STDOUT 和 STDERR。
