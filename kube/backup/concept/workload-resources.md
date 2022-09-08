# 工作负载资源
<!-- TOC -->

- [工作负载资源](#工作负载资源)
    - [ReplicaSet](#replicaset)
    - [Deployments](#deployments)
        - [更新](#更新)
        - [回滚](#回滚)
        - [Deployment Spec](#deployment-spec)
    - [StatefulSets](#statefulsets)
    - [DaemonSet](#daemonset)
    - [Jobs](#jobs)
        - [Job 并行执行](#job-并行执行)
        - [Pod 回退失效策略](#pod-回退失效策略)
        - [Job 终止与清理](#job-终止与清理)
    - [CronJob](#cronjob)
    - [TTL 控制器](#ttl-控制器)
    - [垃圾收集](#垃圾收集)

<!-- /TOC -->


## ReplicaSet

ReplicaSet 可以维护一组任何时候都处于运行状态的 Pods 副本的集合。主要目的是保证 Pods 副本完全相同，且数量恒定。

ReplicaSet 需要提供三个信息：

- 识别需要进行维护的 Pods，这通过 Selector Label 来完成。
- Pods 的副本数量。
- Pods 的模板。

ReplicaSet 通过 Pods 的 `metadata.ownerReferences` 属性来获知自己拥有哪些 Pods，以及对应的状态。

通常，我们并不直接使用 ReplicaSet，而是使用 Deployment，并且 Deployment 会自动创建 ReplicaSet 并管理 ReplicaSet。

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-rs-test
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx-rs-pod
  template:
    metadata:
      labels:
        app: nginx-rs-pod
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```

**注意：**

- Pods 的名字由 rs 控制器自己生成。
- `.spec.template.spec.restartPolicy` 重启策略仅支持 Always，这也是默认的重启策略。
- rs 不会给 pods 新增 label，只能通过 `metadata.ownerReferences` 来判断 pods 属于哪个 pods。
- `.spec.replicas` 如果没有设置，则默认为 1。

更新 `.spec.replicas` 字段，重新进行 `kubectl apply` 可以进行扩缩容。RS 支持 HPA （自动扩缩容）

```sh
$ kubectl get rs
NAME                          DESIRED   CURRENT   READY   AGE
nginx-deployment-66b6c48dd5   3         3         3       100s
```

字段 | 描述
-|-
NAME | 列出名字空间中 ReplicaSet 的名称；
DESIRED | 显示应用的期望副本个数，即在创建 Deployment 时所定义的值。 此为期望状态；
CURRENT | 显示当前运行状态中的副本个数；
READY | 显示应用中有多少副本可以为用户提供服务；
AGE | 显示应用已经运行的时间长度。

## Deployments

Deployments 为 Pods 和 RS 提供了声明式更新的能力，并且支持控制更新速率、支持记录版本和回滚，支持 HPA 等。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```

**注意：**

- Deployment 会自动生成 RS，RS 会自动生成 Pods，其中 RS 的名称由 Deployment 控制，Pods 的名称由 RS 控制。
- RS 的名称格式为 `[Deployment名称]-[随机字符串]`, Pods 的名称格式为 `[RS名称]-[随机字符串]`

### 更新

当对 yaml 文件进行修改后，可以通过下面的方式进行更新：

- 修改 yaml 文件后重新 apply
- 使用 set 的方式

  ```sh
  kubectl set image deployment/nginx-deployment nginx=nginx:1.16.1 --record
  ```

- 直接编辑 yaml 文件

  ```sh
  # 运行命令后会显示 manifest 编辑后保存直接生效
  kubectl edit deployment.v1.apps/nginx-deployment
  ```

当更新完成后，会启动一个新的 rs，并将旧的 rs 的副本数量为逐步变更为 0，最后旧 rs 的 pods 将被清除。

更新时，旧的 rs 并不会被清除，这是为了方便快速回滚，回滚的本质方式就是将当期 rs 的副本数变更为 0，且对应版本的 rs 副本数量恢复。

可以在 Deployment 的 Event 中查看到更新事件：

```sh
$ kubectl describe deployment nginx-deployment
...
Events:
  Type    Reason             Age    From                   Message
  ----    ------             ----   ----                   -------
  Normal  ScalingReplicaSet  10m    deployment-controller  Scaled up replica set nginx-deployment-66b6c48dd5 to 3
  Normal  ScalingReplicaSet  2m22s  deployment-controller  Scaled up replica set nginx-deployment-559d658b74 to 1
  Normal  ScalingReplicaSet  2m19s  deployment-controller  Scaled down replica set nginx-deployment-66b6c48dd5 to 2
  Normal  ScalingReplicaSet  2m19s  deployment-controller  Scaled up replica set nginx-deployment-559d658b74 to 2
  Normal  ScalingReplicaSet  2m17s  deployment-controller  Scaled down replica set nginx-deployment-66b6c48dd5 to 1
  Normal  ScalingReplicaSet  2m17s  deployment-controller  Scaled up replica set nginx-deployment-559d658b74 to 3
  Normal  ScalingReplicaSet  2m15s  deployment-controller  Scaled down replica set nginx-deployment-66b6c48dd5 to 0
```

很明显：

- `nginx-deployment-66b6c48dd5` 是旧的 RS，当 Deployment 启动的时候，直接启动该 RS，并设置副本数为 3。
- `nginx-deployment-559d658b74` 是新的 RS，当 Deployment 更新后，启动该 RS，并且设置副本数为 1。
- 新的 RS 副本数量逐步增大，旧的 RS 副本数逐步减小。
- 这个变更速率是可以控制的。

### 回滚

可以检查 Deployment 的更新历史：

```sh
$ kubectl rollout history deployment.v1.apps/nginx-deployment
deployment.apps/nginx-deployment 
REVISION  CHANGE-CAUSE
2         <none>
3         kubectl set image deployment/nginx-deployment nginx=nginx:1.16.2 --record=true
```

`CHANGE-CAUSE` 的内容是从 Deployment 的注解 `kubernetes.io/change-cause` 复制过来的。

当在变更时使用 `--record=true`，就会将命令记录到 Deployment 的注解 `kubernetes.io/change-cause` 中，以便 `CHANGE-CAUSE` 中显示。

正是这个原因，如果直接编辑 yaml 文件，因为只会记录 `kubectl apply` 命令，无法记录详细的变更原因。当然，也可以显示设置当前版本的 `CHANGE-CAUSE`（本质上就是设置注解）

```sh
kubectl annotate deployment.v1.apps/nginx-deployment kubernetes.io/change-cause="test"
```

可以通过 `kubectl rollout history` 命令查看 deployment 某个版本的详细信息。

```sh
$ kubectl rollout history deployment.v1.apps/nginx-deployment --revision=5
deployment.apps/nginx-deployment with revision #5
Pod Template:
  Labels:       app=nginx
        pod-template-hash=66b6c48dd5
  Annotations:  kubernetes.io/change-cause: kubectl set image deployment/nginx-deployment nginx=nginx:1.16.2 --record=true
  Containers:
   nginx:
    Image:      nginx:1.14.2
    Port:       80/TCP
    Host Port:  0/TCP
    Environment:        <none>
    Mounts:     <none>
  Volumes:      <none>
```

在确定回退的版本后，就可以通过 `kubectl rollout undo` 命令进行回退了。

```sh
# kubectl rollout undo deployment.v1.apps/nginx-deployment 不指定具体版本，则回退至上一个版本
$ kubectl rollout undo deployment.v1.apps/nginx-deployment --to-revision=2
deployment.apps/nginx-deployment rolled back
```

Deployment 中 `.spec.revisionHistoryLimit` 字段以指定保留此 Deployment 的多少个旧有 RS。其余的 ReplicaSet 将在后台被垃圾回收。 默认情况下，此值为 10。

**注意：**

- 由于是回退，所以 rs 不会新增，而是直接用之前的 rs。
- 当前版本号会递增，回退到的版本号会消失（因为当前版本号即对应回退的版本号）。

### Deployment Spec

- `.spec.strategy`，Deployment 更新 Pods 的策略描述。
  - `.spec.strategy.type==Recreate`，创建新 Pods 之前，所有现有的 Pods 会被杀死。
  - `.spec.strategy.type==RollingUpdate`，采取 滚动更新的方式更新 Pods。这也是默认配置。
  - `.spec.strategy.rollingUpdate.maxUnavailable`，最大不可用，指定更新过程之不可用 Pods 的数量。该值可以是绝对数字，也可以是百分比。默认值 25%。
  - `.spec.strategy.rollingUpdate.maxSurge`，指定可以创建的超出 期望 Pod 个数的 Pod 数量。此字段的默认值为 25%。
- `.spec.minReadySeconds` 最短就绪时间，用于指定新创建的 Pod 在没有任意容器崩溃情况下的最小就绪时间， 只有超出这个时间 Pod 才被视为可用。默认值为 0。

**注意：**

- 如果 `maxUnavailable=0`，则势必要求通过新建超出预期副本数量的 Pods 来进行滚动更新，所以 `maxUnavailable=0` 时，`maxSurge` 一定不能为 0。
- 如果有多个控制器（例如 Deployment）的选择算符发生重叠，则控制器之间会因冲突而无法正常工作。但是 kube 不会禁止你这样做。


## StatefulSets

## DaemonSet

## Jobs

Job 会创建一个或多个相同的 Pods，并跟踪这些 Pods 的完成情况，当 Pods 完成的个数达到数阈值，Job 结束。

Job 结束并不会自动清理掉 Job 和 Pods（基本上所有的控制器都不会自动清理掉）。如果手动删除掉 Job，会自动清理掉 Pods。

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: test-job
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: init-test
        image: busybox
        command: ['sh', '-c', 'echo "Hello World!" && date && sleep 10']
```

**注意：**

- Pods 中的容器默认重启策略为 `Always`，但是 Job 中的 Pods 容器必须能够结束（否则 Pods 无法完成，Job 也无法完成），必须显示设置
- Pods 的名称由 Jobs 自动生成，并且自动给生成标签 `job-name=test-job`

要获取某个 Job 的所有 Pods，可以依赖 Pod 的 Label：

```sh
label="job-name=test-job"
pods=$(kubectl get pods --selector=${label} --output=jsonpath='{.items[*].metadata.name}')
echo $pods
```

ReplicaSet 管理的是那些不希望被终止的 Pod （例如，Web 服务器）， Job 管理的是那些希望被终止的 Pod（例如，批处理作业）。

正如在 Pod 生命期 中讨论的， Job 仅适合于 restartPolicy 设置为 OnFailure 或 Never 的 Pod。 注意：如果 restartPolicy 未设置，其默认值是 Always。

### Job 并行执行

- 非并行 Job
  - 通常只启动一个 Pod，除非该 Pod 失败。
  - 当 Pod 成功终止时，立即视 Job 为完成状态。
- 具有确切完成数的 Job
  - `.spec.completions` 字段设置为非 0 的正数值。
  - Job 用来代表整个任务，当对应于 1 和 `.spec.completions` 之间的每个整数都存在一个成功的 Pod 时，Job 被视为完成。
- 带工作队列的 Job
  - Pods 并行运行个数是通过 `.spec.parallelism` 进行配置的。该参数默认值为 1。
  - 当没有指定 `.spec.completions` 时，默认值为 `.spec.parallelism`。

Job 运行时会自动调整 Pods 的个数和 spec 的一致，例如 `.spec.parallelism` 为 1，并且重启策略为 Never，则容器失败，导致 Pods 失败，会自动启动一个新的 Pods 重新运行容器。


### Pod 回退失效策略

在 `OnFailure` 的重启策略下，失败的容器会不断的重启，但是由于某些配置错误等原因，不希望容器一直重启。

`.spec.backoffLimit` 可以配置回退次数，若没有配置，默认回退次数为 6。

kube 官方建议尽量使用 Never 的重启重启策略，如果容器失败，会自动启动新的 Pods 并重新运行容器。当然，重启 Pods 的次数这也是受到了 `.spec.backoffLimit` 限制的。

测试后发现，重启策略为 OnFailure 时，Pods 中的容器不断重启，当达到 `.spec.backoffLimit` 时，会自动把 Pods 删除。

### Job 终止与清理

Job 完成时不会再创建新的 Pod，不过已有的 Pod 也不会被删除。

保留这些 Pod 使得你可以查看已完成的 Pod 的日志输出，以便检查错误、警告 或者其它诊断性输出。

Job 完成时 Job 对象也一样被保留下来，这样你就可以查看它的状态。 

- 终止：`.spec.activeDeadlineSeconds` 配置 Job 的最大执行时间，当达到该时间，会自动停止掉 Job 中的所有 Pods。
- 清理：`.spec.ttlSecondsAfterFinished` 配置 Job 完成（失败或成功）后多长时间将 Job 进行自动删除，避免 Job 的累积。

## CronJob

CronJob 用于提供周期性、反复运行的 Job。每次到执行时间，CronJob 都会启动一个新的 Job 来运行。

```yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: test-cron-job
spec:
  schedule: "* * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: Never
          containers:
          - name: test
            image: busybox
            command: ['sh', '-c', 'echo "Hello World!" && date && sleep 10']
```

**注意：**

- 每次执行任务时大约会创建一个 Job，之所以是 “大约”，是因为某些情况下可能创建两个 Job，或者根本不会创建 Job（？？为什么，猜想是集群压力导致有时候不能调度任务），因此要求 Job 是幂等。
- 每次 CronJob 执行的时候都会检查之前错过的调度的次数（不是失败，只是错过），如果错过了超过 100 次，则不会启动这个任务，并记录错误：
  ```text
  Cannot determine if job needs to be started. Too many missed start time (> 100). Set or decrease .spec.startingDeadlineSeconds or check clock skew.
  ```
- 每次调度生成的 Job、Pods 都可以通过 `kubectl get` 命令查看以及对应的输出，但是只能看到最近的 3 个 Jobs 和 Pods。

## TTL 控制器

TTL 可以限制对象结束后的生命周期，避免结束的对象积累，目前 TTL 只能用于 Job。

Job 的 `.spec.ttlSecondsAfterFinished` 配置设置 Job 在完成后进行清理的时间。

## 垃圾收集

垃圾收集用于处理曾经拥有 Owner 的，但是现在没有 Owner 的对象（有点类似子进程的父进程死后的处理）。

某些 Kubernetes 对象有对应的 Owner，例如，一个 ReplicaSet 是一组 Pod 的 Owner，并由 `metadata.ownerReferences` 参数指出（可以通过 `kubectl get pods --output=yaml` 获得所有 pods 的详细信息）。

当删除一个对象时，可以指定对象对应的附属是否也进行删除，自动删除的行为被称为 `Cascading Deletion`，若没有自动删除附属，则这些附属会成为孤立对象（Orphaned）。通过 `kubectl delete` 的 `--cascade==false` 选项，可以进行非级联删除。

对于，级联删除又分了两种：

- Foreground（前台级联删除），会先给根对象标记为 `deletion in progress` 状态，并对附属对象进行处理后，才会删除根对象：
  - 处于 `deletion in progress` 时，根对象仍然可以通过 REST API 可见。
  - 根对象的 `metadata.finalizers` 字段包含值 foregroundDeletion。
  - 垃圾收集器在删除了所有有阻塞能力的附属（对象的 ownerReference.blockOwnerDeletion=true） 之后，删除属主对象。
- Background（后台级联删除），Kube 会立即删除属主对象，之后垃圾收集器 会在后台删除其附属对象。

通过指定删除策略来删除对象：

- kubectl
  - kubectl 的级联删除
  
    ```sh
    # 默认 --cascade 就是 true，但是不太确认是前台还是后台的级联删除
    
    kubectl delete deployment DEPLOYMENT_NAME --cascade=true
    ```
  
  - kubectl 的非级联删除
  
    ```sh
    kubectl delete deployment DEPLOYMENT_NAME --cascade=false
    ```
- curl
  - 后台级联删除
  
    ```sh
    kubectl proxy --port=8080
    curl -X DELETE localhost:8080/apis/apps/v1/namespaces/default/replicasets/my-repset \
      -d '{"kind":"DeleteOptions","apiVersion":"v1","propagationPolicy":"Background"}' \
      -H "Content-Type: application/json"
    ```
  
  - 前台级联删除
  
    ```sh
    kubectl proxy --port=8080
    curl -X DELETE localhost:8080/apis/apps/v1/namespaces/default/replicasets/my-repset \
      -d '{"kind":"DeleteOptions","apiVersion":"v1","propagationPolicy":"Foreground"}' \
      -H "Content-Type: application/json"
    ```
  
  - 非级联删除
  
    ```sh
    kubectl proxy --port=8080
    curl -X DELETE localhost:8080/apis/apps/v1/namespaces/default/replicasets/my-repset \
      -d '{"kind":"DeleteOptions","apiVersion":"v1","propagationPolicy":"Orphan"}' \
      -H "Content-Type: application/json"
    ```
