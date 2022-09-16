# CronJob

CronJob 创建基于时隔重复调度的 Jobs。

一个 CronJob 对象就像 crontab (cron table) 文件中的一行。它用 Cron 格式进行编写，并周期性地在给定的调度时间执行 Job。

## CronJob

CronJob 用于执行周期性的动作，例如备份、报告生成等。这些任务中的每一个都应该配置为周期性重复的（例如：每天/每周/每月一次）； 你可以定义任务开始执行的时间间隔。

下面的 CronJob 示例清单会在每分钟打印出当前时间和问候消息：

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: hello
spec:
  schedule: "* * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: hello
            image: busybox:1.28
            imagePullPolicy: IfNotPresent
            command:
            - /bin/sh
            - -c
            - date; echo Hello from the Kubernetes cluster
          restartPolicy: OnFailure
```

Cron 时间表语法：

```sh
# ┌───────────── 分钟 (0 - 59)
# │ ┌───────────── 小时 (0 - 23)
# │ │ ┌───────────── 月的某天 (1 - 31)
# │ │ │ ┌───────────── 月份 (1 - 12)
# │ │ │ │ ┌───────────── 周的某天 (0 - 6)（周日到周一；在某些系统上，7 也是星期日）
# │ │ │ │ │                          或者是 sun，mon，tue，web，thu，fri，sat
# │ │ │ │ │
# │ │ │ │ │
# * * * * *
```

## CronJob 限制

CronJob 根据其计划编排，在每次该执行任务的时候大约会创建一个 Job。我们之所以说 "大约"，是因为在某些情况下，可能会创建两个 Job，或者不会创建任何 Job。我们试图使这些情况尽量少发生，但不能完全杜绝。因此 Job 应该是 幂等的。

如果 startingDeadlineSeconds 设置为很大的数值或未设置（默认），并且 concurrencyPolicy 设置为 Allow，则作业将始终至少运行一次。

**注意：**

- 如果 startingDeadlineSeconds 的设置值低于 10 秒钟，CronJob 可能无法被调度。这是因为 CronJob 控制器每 10 秒钟执行一次检查。


对于每个 CronJob，CronJob 控制器（Controller）检查从上一次调度的时间点到现在所错过了调度次数。如果错过的调度次数超过 100 次， 那么它就不会启动这个任务，并记录这个错误:

```txt
Cannot determine if job needs to be started. Too many missed start time (> 100). Set or decrease .spec.startingDeadlineSeconds or check clock skew.
```

需要注意的是，如果 startingDeadlineSeconds 字段非空，则控制器会统计从 startingDeadlineSeconds 设置的值到现在而不是从上一个计划时间到现在错过了多少次 Job。 例如，如果 startingDeadlineSeconds 是 200，则控制器会统计在过去 200 秒中错过了多少次 Job。

如果未能在调度时间内创建 CronJob，则计为错过。 例如，如果 concurrencyPolicy 被设置为 Forbid，并且当前有一个调度仍在运行的情况下， 试图调度的 CronJob 将被计算为错过。

CronJob 仅负责创建与其调度时间相匹配的 Job，而 Job 又负责管理其代表的 Pod。
