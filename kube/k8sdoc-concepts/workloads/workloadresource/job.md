# Job

Job 会创建一个或者多个 Pod，并将继续重试 Pod 的执行，直到指定数量的 Pod 成功终止。

随着 Pod 成功结束，Job 跟踪记录成功完成的 Pod 个数。 当数量达到指定的成功个数阈值时，任务（即 Job）结束。删除 Job 的操作会清除所创建的全部 Pod。挂起 Job 的操作会删除 Job 的所有活跃 Pod，直到 Job 被再次恢复执行。

一种简单的使用场景下，你会创建一个 Job 对象以便以一种可靠的方式运行某 Pod 直到完成。当第一个 Pod 失败或者被删除（比如因为节点硬件失效或者重启）时，Job 对象会**启动一个新的 Pod**。

你也可以使用 Job 以并行的方式运行多个 Pod。

如果你想按某种排期表（Schedule）运行 Job（单个任务或多个并行任务），请参阅 CronJob。

## 运行示例 Job

下面是一个 Job 配置示例。它负责计算 π 到小数点后 2000 位，并将结果打印出来。 此计算大约需要 10 秒钟完成：

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pi
spec:
  template:
    spec:
      containers:
      - name: pi
        image: perl:5.34.0
        command: ["perl",  "-Mbignum=bpi", "-wle", "print bpi(2000)"]
      restartPolicy: Never
  backoffLimit: 4
```

执行 Job：

```sh
$ kubectl apply -f https://kubernetes.io/examples/controllers/job.yaml
job.batch/pi created

# 查看 job 状态
$ kubectl describe jobs/pi
Name:           pi
Namespace:      default
Selector:       controller-uid=c9948307-e56d-4b5d-8302-ae2d7b7da67c
Labels:         controller-uid=c9948307-e56d-4b5d-8302-ae2d7b7da67c
                job-name=pi
Annotations:    kubectl.kubernetes.io/last-applied-configuration:
                  {"apiVersion":"batch/v1","kind":"Job","metadata":{"annotations":{},"name":"pi","namespace":"default"},"spec":{"backoffLimit":4,"template":...
Parallelism:    1
Completions:    1
Start Time:     Mon, 02 Dec 2019 15:20:11 +0200
Completed At:   Mon, 02 Dec 2019 15:21:16 +0200
Duration:       65s
Pods Statuses:  0 Running / 1 Succeeded / 0 Failed
Pod Template:
  Labels:  controller-uid=c9948307-e56d-4b5d-8302-ae2d7b7da67c
           job-name=pi
  Containers:
   pi:
    Image:      perl:5.34.0
    Port:       <none>
    Host Port:  <none>
    Command:
      perl
      -Mbignum=bpi
      -wle
      print bpi(2000)
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Events:
  Type    Reason            Age   From            Message
  ----    ------            ----  ----            -------
  Normal  SuccessfulCreate  14m   job-controller  Created pod: pi-5rwd7

# describe job 不太能看出来 job 的 pod
# 要查看属于某 Job 的全部 Pod，你可以使用类似下面这条命令：
$ pods=$(kubectl get pods --selector=job-name=pi --output=jsonpath='{.items[*].metadata.name}') && echo $pods
pi-5rwd7

# 查看标准输出：
$ kubectl logs $pods
3.1415926535897932384626433832795028....
```

## 编写 Job Spec

与 Kubernetes 中其他资源的配置类似，Job 也需要 apiVersion、kind 和 metadata 字段。 Job 的名字必须是合法的 DNS 子域名。

Job 配置还需要一个 `.spec`。

### Pod 模板

Job 的 `.spec` 中只有 `.spec.template` 是必需的字段。

字段 `.spec.template` 的值是一个 Pod 模板。 其定义规范与 Pod 完全相同，只是其中不再需要 apiVersion 或 kind 字段。

除了作为 Pod 所必需的字段之外，Job 中的 Pod 模板必须设置合适的标签 （参见 Pod 选择算符）和合适的重启策略。

**Job 中 Pod 的 RestartPolicy 只能设置为 Never 或 OnFailure 之一**，默认策略是 Always，所以一定要修改该策略。

### Pod 选择算符

字段 `.spec.selector` 是可选的。在绝大多数场合，你都不需要为其赋值。默认会自动为 Pod 生成 Label，以合适的 Job Label Selector：

```sh
$ kubectl get job pi -o yaml
...
spec:
  selector:
    matchLabels:
      controller-uid: 21c5d1f4-cbde-424d-b7ae-0e65bd149464
  suspend: false
  template:
    metadata:
      creationTimestamp: null
      labels:
        controller-uid: 21c5d1f4-cbde-424d-b7ae-0e65bd149464
        job-name: pi
...
```

### Job 的并行执行

适合以 Job 形式来运行的任务主要有三种：

- 非并行 Job：通常只启动一个 Pod，除非该 Pod 失败。当 Pod 成功终止时，立即视 Job 为完成状态。
- 具有确定完成计数的并行 Job：`.spec.completions` 字段设置为非 0 的正数值。
  - Job 用来代表整个任务，当成功的 Pod 个数达到 `.spec.completions` 时，Job 被视为完成。
  - 当使用 `.spec.completionMode="Indexed"` 时，每个 Pod 都会获得一个不同的 索引值，介于 0 和 .spec.completions-1 之间。
- 带工作队列的并行 Job：
  - 不设置 spec.completions，默认值为 .spec.parallelism。
  - 多个 Pod 之间必须相互协调，或者借助外部服务确定每个 Pod 要处理哪个工作条目。 例如，任一 Pod 都可以从工作队列中取走最多 N 个工作条目。
  - 每个 Pod 都可以独立确定是否其它 Pod 都已完成，进而确定 Job 是否完成。
  - 当 Job 中任何 Pod 成功终止，不再创建新 Pod。
  - 一旦至少 1 个 Pod 成功完成，并且所有 Pod 都已终止，即可宣告 Job 成功完成。
  - 一旦任何 Pod 成功退出，任何其它 Pod 都不应再对此任务执行任何操作或生成任何输出。 所有 Pod 都应启动退出过程。

具体而言：

- 对于非并行的 Job，你可以不设置 spec.completions 和 spec.parallelism。这两个属性都不设置时，均取默认值 1。
- 对于确定完成计数类型的 Job，你应该设置 .spec.completions 为所需要的完成个数。你可以设置 .spec.parallelism，也可以不设置。其默认值为 1。当设置了 `.spec.parallelism` 时，代表了并行可以执行多少个 Pod。
- 对于一个工作队列 Job，你不可以设置 .spec.completions，但要将.spec.parallelism 设置为一个非负整数。

### 控制并行性

并行性请求（.spec.parallelism）可以设置为任何非负整数，有两个特殊的取值：

- 如果未设置，则默认为 1。
- 如果设置为 0，则 Job 相当于启动之后便被暂停，直到此值被增加。

实际并行性（在任意时刻运行状态的 Pod 个数）可能比并行性请求略大或略小， 原因如下：

- 对于确定完成计数 Job，实际上并行执行的 Pod 个数不会超出剩余的完成数。因此，如果 `.spec.parallelism` 值较高，会被忽略。
- 对于工作队列 Job，有任何 Job 成功结束之后，不会有新的 Pod 启动。不过，剩下的 Pod 允许执行完毕。
- 如果 Job 控制器 没有来得及作出响应，或者如果 Job 控制器因为任何原因（例如，缺少 ResourceQuota 或者没有权限）无法创建 Pod。 Pod 个数可能比请求的数目小。
- Job 控制器可能会因为之前同一 Job 中 Pod 失效次数过多而压制新 Pod 的创建。
- 当 Pod 处于优雅终止进程中，需要一定时间才能停止。

### 完成模式

带有确定完成计数的 Job，即 `.spec.completions` 不为 null 的 Job，都可以在其 `.spec.completionMode` 中设置完成模式：

- NonIndexed（默认值）：当成功完成的 Pod 个数达到 .spec.completions 所设值时认为 Job 已经完成。换言之，每个 Job 完成事件都是独立无关且同质的。要注意的是，当 .spec.completions 取值为 null 时，Job 被隐式处理为 NonIndexed。
- Indexed：Job 的 Pod 会获得对应的完成索引，取值为 0 到 .spec.completions-1。 该索引可以通过三种方式获取：
  - Pod 注解 batch.kubernetes.io/job-completion-index。
  - 作为 Pod 主机名的一部分，遵循模式 `$(job-name)-$(index)`。当你同时使用带索引的 Job（Indexed Job）与 服务（Service），Job 中的 Pod 可以通过 DNS 使用确切的主机名互相寻址。
  - 对于容器化的任务，在环境变量 JOB_COMPLETION_INDEX 中。
  - 当每个索引都对应一个完成完成的 Pod 时，Job 被认为是已完成的。关于如何使用这种模式的更多信息，可参阅 用带索引的 Job 执行基于静态任务分配的并行处理。
  - 需要注意的是，对同一索引值可能被启动的 Pod 不止一个，尽管这种情况很少发生。这时，只有一个会被记入完成计数中。

## 处理 Pod 和容器失效

Pod 中的容器可能因为多种不同原因失效，例如：

- 因为其中的进程退出时返回值非零
- 或者容器因为超出内存约束而被杀死等等

如果发生这类事件，并且 `.spec.template.spec.restartPolicy = "OnFailure"`，Pod 则继续留在当前节点，但容器会被重新运行。

因此，你的程序需要能够处理在本地被重启的情况。

如果设置 `.spec.template.spec.restartPolicy = "Never"`，则容器不会重启，反应出来了则是 Pod 失败。

整个 Pod 也可能会失败，且原因各不相同：

- 当 Pod 启动时，节点失效（被升级、被重启、被删除等）
- 容器失败而 `.spec.template.spec.restartPolicy = "Never"`

Job 控制器面对这些 Pod 失败时，会启动一个新的 Pod。这意味着，你的应用需要处理在一个新 Pod 中被重启的情况。尤其是应用需要处理之前运行所产生的临时文件、锁、不完整的输出等问题。

**注意：**

- 即使你将 .spec.parallelism 设置为 1，且将 .spec.completions 设置为 1，并且 .spec.template.spec.restartPolicy 设置为 "Never"，同一程序仍然有可能被启动两次。
- 如果你确实将 .spec.parallelism 和 .spec.completions 都设置为比 1 大的值， 那就有可能同时出现多个 Pod 运行的情况。 为此，你的 Pod 也必须能够处理并发性问题。

### Pod 回退失效策略

对于 Pod 失败或者容器失败，这种情形下，你可能希望 Job 在经历若干次重试之后直接进入失败状态，因为这很可能意味着遇到了配置错误。

为了实现这点，可以将 `.spec.backoffLimit` 设置为视 Job 为失败之前的重试次数。失效回退的限制值默认为 6。

与 Job 相关的失效的 Pod 会被 Job 控制器重建，回退重试时间将会按指数增长 （从 10 秒、20 秒到 40 秒）最多至 6 分钟。

计算重试次数有以下两种方法：

- 计算 .status.phase = "Failed" 的 Pod 数量。
- 当 Pod 的 restartPolicy = "OnFailure" 时，针对 .status.phase 等于 Pending 或 Running 的 Pod。计算其中所有容器的重试次数。

如果两种方式其中一个的值达到 .spec.backoffLimit，则 Job 被判定为失败。

**注意：**

- 如果你的 Job 的 restartPolicy 被设置为 "OnFailure"，就要注意运行该 Job 的 Pod 会在 Job 到达失效回退次数上限时自动被终止。这会使得调试 Job 中可执行文件的工作变得非常棘手。
- 我们建议在调试 Job 时将 restartPolicy 设置为 "Never"，或者使用日志系统来确保失效 Job 的输出不会意外遗失。

这是一个容器重启策略为 Never，并且会始终失败的容器：

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: fail-pod-job-test
spec:
  template:
    spec:
      containers:
      - name: exit1
        image: busybox:1.28
        command: ['sh', '-c', 'echo "Hello, Kubernetes!" && sleep 10 && exit 1']
      restartPolicy: Never
  backoffLimit: 3
```

backoffLimit 为 3 次，即可以重试 3 次。加上第一次，一共会触发四次 Pod 执行：

```sh
$ kubectl get pods
NAME                                READY   STATUS    RESTARTS   AGE
fail-pod-job-test--1-427tx          0/1     Error     0          2m59s
fail-pod-job-test--1-9jt5x          0/1     Error     0          3m20s
fail-pod-job-test--1-dvj8q          0/1     Error     0          3m33s
fail-pod-job-test--1-kmt42          0/1     Error     0          2m39s
```

查看 job 信息：

```sh
$ kubectl get jobs
NAME                COMPLETIONS   DURATION   AGE
fail-pod-job-test   0/1           4m2s       4m2s

$ kubectl describe jobs
Events:
  Type     Reason                Age    From            Message
  ----     ------                ----   ----            -------
  Normal   SuccessfulCreate      4m18s  job-controller  Created pod: fail-pod-job-test--1-dvj8q
  Normal   SuccessfulCreate      4m5s   job-controller  Created pod: fail-pod-job-test--1-9jt5x
  Normal   SuccessfulCreate      3m44s  job-controller  Created pod: fail-pod-job-test--1-427tx
  Normal   SuccessfulCreate      3m24s  job-controller  Created pod: fail-pod-job-test--1-kmt42
  Warning  BackoffLimitExceeded  2m44s  job-controller  Job has reached the specified backoff limit
```

那如果是 Container 失败呢？参考这样一个 Container 固定失败，且会拉起的 job：

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: fail-container-job-test
spec:
  template:
    spec:
      containers:
      - name: exit1
        image: busybox:1.28
        command: ['sh', '-c', 'echo "Hello, Kubernetes!" && sleep 10 && exit 1']
      restartPolicy: OnFailure
  backoffLimit: 3
```

在容器重试一定次数后，job 会变成：

```sh
$ kubectl get jobs
NAME                      COMPLETIONS   DURATION   AGE
fail-container-job-test   0/1           3m8s       3m8s

$ kubectl describe job fail-container-job-test
...
Events:
  Type     Reason                Age    From            Message
  ----     ------                ----   ----            -------
  Normal   SuccessfulCreate      2m28s  job-controller  Created pod: fail-container-job-test--1-n5t26
  Normal   SuccessfulDelete      77s    job-controller  Deleted pod: fail-container-job-test--1-n5t26
  Warning  BackoffLimitExceeded  77s    job-controller  Job has reached the specified backoff limit
```

同时可以看到，对应的 pod 会被删除，这会导致不方便定位问题，因此建议使用 Never，即不进行容器级别的重启，而是进行 Pod 的重启。

## 自动清理完成的 Job

从实验中我们可以看到，Job 完成工作，或者失败，也会继续留在 k8s 系统中，可以通过 kubectl 查询到。

但是，完成的 Job 通常不需要留存在系统中。在系统中一直保留它们会给 API 服务器带来额外的压力。如果 Job 由某种更高级别的控制器来管理，例如 CronJob， 则 Job 可以被 CronJob 基于特定的根据容量裁定的清理策略清理掉。

### 已完成 Job 的 TTL 机制

自动清理已完成 Job （状态为 Complete 或 Failed）的另一种方式是使用由 TTL 控制器所提供的 TTL 机制。 通过设置 Job 的 `.spec.ttlSecondsAfterFinished` 字段，可以让该控制器清理掉已结束的资源。

TTL 控制器清理 Job 时，会级联式地删除 Job 对象。换言之，它会删除所有依赖的对象，包括 Pod 及 Job 本身。

**注意：**

- 当 Job 被删除时，系统会考虑其生命周期保障，例如其 Finalizers。

例如：

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pi-with-ttl
spec:
  ttlSecondsAfterFinished: 100
  template:
    spec:
      containers:
      - name: pi
        image: perl:5.34.0
        command: ["perl",  "-Mbignum=bpi", "-wle", "print bpi(2000)"]
      restartPolicy: Never
```

Job `pi-with-ttl` 在结束 100 秒之后，可以成为被自动删除的对象。

有两个特殊的设置：

- 如果该字段设置为 0，Job 在结束之后立即成为可被自动删除的对象。
- 如果该字段没有设置，Job 不会在结束之后被 TTL 控制器自动清除。

## Job 模式 

Job 对象可以用来支持多个 Pod 的可靠的并发执行。

Job 对象不是设计用来支持相互通信的并行进程的，后者一般在科学计算中应用较多。

Job 的确能够支持对一组相互独立而又有所关联的工作条目的并行处理。这类工作条目可能是：

- 要发送的电子邮件
- 要渲染的视频帧
- 要编解码的文件
- NoSQL 数据库中要扫描的主键范围等等

在一个复杂系统中，可能存在多个不同的工作条目集合。这里我们仅考虑用户希望一起管理的工作条目集合之一：批处理作业。

并行计算的模式有好多种，每种都有自己的强项和弱点。这里要权衡的因素有：

- 每个工作条目对应一个 Job 或者所有工作条目对应同一 Job 对象。后者更适合处理大量工作条目的场景； 前者会给用户带来一些额外的负担，而且需要系统管理大量的 Job 对象。
- 创建与工作条目相等的 Pod 或者令每个 Pod 可以处理多个工作条目。前者通常不需要对现有代码和容器做较大改动；后者则更适合工作条目数量较大的场合，原因同上。
- 有几种技术都会用到工作队列。这意味着需要运行一个队列服务， 并修改现有程序或容器使之能够利用该工作队列。 与之比较，其他方案在修改现有容器化应用以适应需求方面可能更容易一些。

下面是对这些权衡的汇总，第 2 到 4 列对应上面的权衡比较。 模式的名称对应了相关示例和更详细描述的链接。

模式 | 单个 Job 对象 | Pod 数少于工作条目数？ | 直接使用应用无需修改?
-|-|-|-
每工作条目一 Pod 的队列 | ✓ | | 有时
Pod 数量可变的队列 | ✓ | ✓ | 
静态任务分派的带索引的 Job | ✓ | | ✓
Job 模板扩展 | | | ✓

当你使用 .spec.completions 来设置完成数时，Job 控制器所创建的每个 Pod 使用完全相同的 spec。 这意味着任务的所有 Pod 都有相同的命令行，都使用相同的镜像和数据卷， 甚至连环境变量都（几乎）相同。 这些模式是让每个 Pod 执行不同工作的几种不同形式。

下表显示的是每种模式下 .spec.parallelism 和 .spec.completions 所需要的设置。 其中，W 表示的是工作条目的个数。

模式 | .spec.completions | .spec.parallelism
-|-|-
每工作条目一 Pod 的队列 | W | 任意值
Pod 个数可变的队列 | 1 | 任意值
静态任务分派的带索引的 Job | W |
Job 模板扩展 | 1 | 应该为 1

## 高级用法

### 挂起 Job

Job 被创建时，Job 控制器会马上开始执行 Pod 创建操作以满足 Job 的需求，并持续执行此操作直到 Job 完成为止。不过你可能想要暂时挂起 Job 执行，或启动处于挂起状态的 Job，并拥有一个自定义控制器以后再决定什么时候开始。

要挂起一个 Job，你可以更新 `.spec.suspend` 字段为 true， 之后，当你希望恢复其执行时，将其更新为 false。创建一个 `.spec.suspend` 被设置为 true 的 Job 本质上会将其创建为被挂起状态。

当 Job 被从挂起状态恢复执行时，其 `.status.startTime` 字段会被**重置为当前的时间**。 这意味着 .spec.activeDeadlineSeconds 计时器会在 Job 挂起时被停止， 并在 Job 恢复执行时复位。

当你挂起一个 Job 时，所有正在运行且状态不是 Completed 的 Pod 将被终止。Pod 的优雅终止期限会被考虑，不过 Pod 自身也必须在此期限之内处理完信号。处理逻辑可能包括保存进度以便将来恢复，或者取消已经做出的变更等等。Pod 以这种形式终止时，不会被记入 Job 的 completions 计数。

处于被挂起状态的 Job 的定义示例可能是这样子：

```sh
$ kubectl get job myjob -o yaml

apiVersion: batch/v1
kind: Job
metadata:
  name: myjob
spec:
  suspend: true
  parallelism: 1
  completions: 5
  template:
    spec:
      ...

# 挂起一个活跃的 Job：
$ kubectl patch job/myjob --type=strategic --patch '{"spec":{"suspend":true}}'

# 恢复一个挂起的 Job：
$ kubectl patch job/myjob --type=strategic --patch '{"spec":{"suspend":false}}'

# Job 的 status 可以用来确定 Job 是否被挂起，或者曾经被挂起.
# 当 Job 被挂起和恢复执行时，也会生成事件.
$ kubectl get jobs/myjob -o yaml
apiVersion: batch/v1
kind: Job
status:
  conditions:
  - lastProbeTime: "2021-02-05T13:14:33Z"
    lastTransitionTime: "2021-02-05T13:14:33Z"
    status: "True"
    type: Suspended
  startTime: "2021-02-05T13:13:48Z"
...
Events:
  Type    Reason            Age   From            Message
  ----    ------            ----  ----            -------
  Normal  SuccessfulCreate  12m   job-controller  Created pod: myjob-hlrpl
  Normal  SuccessfulDelete  11m   job-controller  Deleted pod: myjob-hlrpl
  Normal  Suspended         11m   job-controller  Job suspended
  Normal  SuccessfulCreate  3s    job-controller  Created pod: myjob-jvb44
  Normal  Resumed           3s    job-controller  Job resumed
```

## 替代方案

替代方案 | 描述
-|-
裸 Pod | 当 Pod 运行所在的节点重启或者失败，Pod 会被终止并且不会被重启。<br>Job 会重新创建新的 Pod 来替代已终止的 Pod。<br>因为这个原因，我们建议你使用 Job 而不是独立的裸 Pod， 即使你的应用仅需要一个 Pod。
Replica Controller | Job 与 rc 是彼此互补的。rc 管理的是那些**不希望被终止**的 Pod （例如，Web 服务器），Job 管理的是那些**希望被终止的** Pod（例如，批处理作业）。<br>正如在 Pod 生命期中讨论的， Job 仅适合于 restartPolicy 设置为 OnFailure 或 Never 的 Pod。 <br>**注意：**如果 restartPolicy 未设置，其默认值是 Always。
单个 Job 启动控制器 Pod | 另一种模式是用唯一的 Job 来创建 Pod，而该 Pod 负责启动其他 Pod，因此扮演了一种后启动 Pod 的控制器的角色。 这种模式的灵活性更高，但是有时候可能会把事情搞得很复杂，很难入门， 并且与 Kubernetes 的集成度很低。<br>这种模式的实例之一是用 Job 来启动一个运行脚本的 Pod，脚本负责启动 Spark 主控制器（参见 Spark 示例），运行 Spark 驱动，之后完成清理工作。
