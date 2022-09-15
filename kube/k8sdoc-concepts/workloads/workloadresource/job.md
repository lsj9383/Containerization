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
