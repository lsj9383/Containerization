# Kubernetes 对象管理

kubectl 命令行工具支持多种不同的方式来**创建**和**管理** Kubernetes 对象。

## 对象管理技术

管理技术 | 作用于的对象 | 推荐的环境 | Supported writers | 学习曲线
-|-|-|-|-
指令式名类（Imperative commands） | 活跃对象 | 开发环境 | 1+ | 低
指令式对象配置（Imperative object configuration）| 单个文件 | 生产环境 | 1 | 中
声明式对象配置（Declarative object configuration）| 文件目录 | 生产环境 | 1+ | 高

## 指令式命令

使用指令式命令时，用户可以在集群中的活动对象上进行操作：用户将操作传给 kubectl 命令作为参数或标志。

这是开始或者在集群中运行一次性任务的推荐方法。

**注意：**

- 因为这个技术直接在活跃对象上操作，所以它不提供以前对象配置的历史记录。

例子：

```sh
# 通过创建 Deployment 对象来运行 nginx 容器的实例

kubectl create deployment nginx --image nginx
```

与对象配置相比的优点：

- 命令简单，易学且易于记忆。
- 命令仅需一步即可对集群进行更改。

与对象配置相比的缺点：

- 命令不与变更审查流程集成。
- 命令不提供与更改关联的审核跟踪。
- 除了实时内容外，命令不提供记录源。
- 命令不提供用于创建新对象的模板。

## 指令式对象配置

在指令式对象配置中，kubectl 命令需要指定：

- 执行什么操作（例如 create、apply、replace 等，现在一般主流用 apply 代替 create 和 replace）
- 可选的某些标志参数
- 至少一个文件名（指定的文件必须包含 YAML 或 JSON 格式的对象的完整定义）

例子：

```sh
# 创建配置文件中定义的对象
kubectl create -f nginx.yaml

# 删除 nginx.yaml 和 redis.yaml 两个配置文件中定义的对象
kubectl delete -f nginx.yaml -f redis.yaml

# 更新配置文件中定义的对象
kubectl replace -f nginx.yaml
```

## 声明式对象配置

使用声明式对象配置时，用户对本地存储的对象配置文件进行操作，但是用户没有定义要对该文件执行的操作。

kubectl 会**自动检测**每个文件的创建、更新和删除操作。

这使得配置可以在目录上工作，根据目录中配置文件对不同的对象执行不同的操作。

例子：

```sh
kubectl diff -f configs/
kubectl apply -f configs/
```


递归处理目录：

```sh
kubectl diff -R -f configs/
kubectl apply -R -f configs/
```
