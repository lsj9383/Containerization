# 运行时选项和指标

默认情况下，一个运行的容器没有资源限制，该容器可以获得宿主机内核可以调度的所有资源。

为了控制容器对资源的使用， Docker 提供了命令控制内存和 CPU 的方式。

在进行资源限制时，大多数时正整数加单位的形式，单位通常有：b, k, m, g（字节、K 字节、M 字节、G 字节）。

## 统计信息

`docker stats` 可以实时显示容器的运行时指标，包括CPU，内存使用率，内存限制和网络IO指标。

命令后面可以指定容器，若不指定则显示所有容器的的信息。

```sh
$ docker stats redis1 redis2

CONTAINER           CPU %               MEM USAGE / LIMIT     MEM %               NET I/O             BLOCK I/O
redis1              0.07%               796 KB / 64 MB        1.21%               788 B / 648 B       3.568 MB / 512 KB
redis2              0.07%               2.746 MB / 64 MB      4.29%               1.266 KB / 648 B    12.4 MB / 0 B
```

## 内存限制

Option | 说明
-|-
-m or --memory= | 容器可以使用内存最大值。如果设置了该选项，则容器最小值为 4m。在容器内通过 free 看到还是宿主机内存，可以通过 `docker stats` 观察最大内存。
--memory-swap* | The amount of memory this container is allowed to swap to disk. See --memory-swap details.
--memory-swappiness | By default, the host kernel can swap out a percentage of anonymous pages used by a container. You can set --memory-swappiness to a value between 0 and 100, to tune this percentage. See --memory-swappiness details.
--memory-reservation | Allows you to specify a soft limit smaller than --memory which is activated when Docker detects contention or low memory on the host machine. If you use --memory-reservation, it must be set lower than --memory for it to take precedence. Because it is a soft limit, it does not guarantee that the container doesn’t exceed the limit.
--kernel-memory | The maximum amount of kernel memory the container can use. The minimum allowed value is 4m. Because kernel memory cannot be swapped out, a container which is starved of kernel memory may block host machine resources, which can have side effects on the host machine and on other containers. See --kernel-memory details.
--oom-kill-disable | By default, if an out-of-memory (OOM) error occurs, the kernel kills processes in a container. To change this behavior, use the --oom-kill-disable option. Only disable the OOM killer on containers where you have also set the -m/--memory option. If the -m flag is not set, the host can run out of memory and the kernel may need to kill the host system’s processes to free memory.
