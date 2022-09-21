# Pod 安全性准入

Kubernetes Pod 安全性标准（Security Standards）为 Pod 定义不同的隔离级别。这些标准能够让你以一种清晰、一致的方式定义如何限制 Pod 行为。

Kubernetes 提供了一个内置的 Pod Security 准入控制器来执行 Pod 安全标准（Pod Security Standard）。 创建 Pod 时在 namespace 中应用这些 Pod 安全限制。

## Pod 安全性级别

Pod 安全性准入插件对 Pod 的安全性上下文有一定的要求，并且依据 Pod 安全性标准所定义的三个级别 （privileged、baseline 和 restricted）对其他字段也有要求。

## 为 namespace 设置 Pod 安全性准入控制标签

一旦特性被启用或者安装了 Webhook，你可以配置 namespace 以定义每个名字空间中 Pod 安全性准入控制模式。

Kubernetes 定义了一组标签，你可以设置这些标签来定义某个 namespace 上要使用的预定义的 Pod 安全性标准级别。你所选择的标签定义了检测到潜在违例时，Control Panel 要采取什么样的动作。

Control Panel 在检查到违例时，具体动作参考下表：

模式 | 描述
-|-
enforce | 策略违例会导致 Pod 被拒绝
audit | 策略违例会触发审计日志中记录新事件时添加审计注解，但是 Pod 仍是被接受的。
warn | 策略违例会触发用户可见的警告信息，但是 Pod 仍是被接受的。

namespace 可以配置任何一种或者所有模式，或者甚至为不同的模式设置不同的级别。

对于每种模式，决定所使用策略的标签有两个：

```sh
# 模式的级别标签用来标示对应模式所应用的策略级别
#
# MODE 必须是 `enforce`、`audit` 或 `warn` 之一
# LEVEL 必须是 `privileged`、baseline` 或 `restricted` 之一
pod-security.kubernetes.io/<MODE>: <LEVEL>

# 可选：针对每个模式版本的版本标签可以将策略锁定到
# 给定 Kubernetes 小版本号所附带的版本（例如 v1.25）
#
# MODE 必须是 `enforce`、`audit` 或 `warn` 之一
# VERSION 必须是一个合法的 Kubernetes 小版本号或者 `latest`
pod-security.kubernetes.io/<MODE>-version: <VERSION>
```

下面是一个配置示例：

```yaml
# Privileged
apiVersion: v1
kind: Namespace
metadata:
  name: my-privileged-namespace
  labels:
    pod-security.kubernetes.io/enforce: privileged
    pod-security.kubernetes.io/enforce-version: latest

# Baseline
apiVersion: v1
kind: Namespace
metadata:
  name: my-baseline-namespace
  labels:
    pod-security.kubernetes.io/enforce: baseline
    pod-security.kubernetes.io/enforce-version: latest
    pod-security.kubernetes.io/warn: baseline
    pod-security.kubernetes.io/warn-version: latest

# Restricted
apiVersion: v1
kind: Namespace
metadata:
  name: my-restricted-namespace
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: latest
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/warn-version: latest
```

## 负载资源和 Pod 模板

Pod 通常是通过创建 Deployment 或 Job 这类工作负载对象 来间接创建的。工作负载对象为工作负载资源定义一个 Pod 模板和一个对应的负责基于该模板来创建 Pod 的控制器。

在 Kubernetes 中：

- 为了尽早地捕获安全策略的违例状况，audit 和 warn 模式都应用到负载资源。
- 不过，enforce 模式并不应用到工作负载资源，仅应用到所生成的 Pod 对象上（估计是因为只有这样才保障创建的 Pod 符合要求）。

## 豁免

你可以为 Pod 安全性的实施设置 `豁免（Exemptions）` 规则， 从而允许创建一些本来会被与给定名字空间相关的策略所禁止的 Pod。豁免规则可以在准入控制器配置中配置。

豁免规则可以显式枚举，满足豁免标准的请求会被准入控制器忽略（所有 enforce、audit 和 warn 行为都会被略过）。

豁免的维度包括：

- Username：来自用户名已被豁免的、已认证的（或伪装的）的用户的请求会被忽略。
- RuntimeClassName： 指定了已豁免的运行时类名称的 Pod 和负载资源会被忽略。
- Namespace：位于被豁免的 namespace 中的 Pod 和负载资源会被忽略。

**注意：**

- 大多数 Pod 是作为对工作负载资源的响应，由控制器所创建的，这意味着为某最终用户提供豁免时，只会当该用户直接创建 Pod 时对其实施安全策略的豁免。用户创建工作负载资源时不会被豁免。
- 控制器服务账号（例如：system:serviceaccount:kube-system:replicaset-controller） 通常不应该被豁免，因为豁免这类服务账号隐含着对所有能够创建对应工作负载资源的用户豁免。

策略检查时会对以下 Pod 字段的更新操作予以豁免，这意味着如果 Pod 更新请求仅改变这些字段时，即使 Pod 违反了当前的策略级别，请求也不会被拒绝。
