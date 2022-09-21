# Pod 安全性标准

Pod 安全性标准定义了三种不同的 策略（Policy），以广泛覆盖安全应用场景。

这些策略是叠加式的（Cumulative），安全级别从高度宽松至高度受限。本指南概述了每个策略的要求。

Profile | 针对群体 | 描述
-|-|-
Privileged | 特权较高、受信任的用户 | 不受限制的策略，提供最大可能范围的权限许可。此策略允许已知的特权提升。
Baseline | 应用运维人员和非关键性应用的开发人员 | 限制性最弱的策略，禁止已知的特权提升。允许使用默认的（规定最少）Pod 配置。
Restricted | 运维人员和安全性很重要的应用的开发人员，以及不太被信任的用户 | 限制性非常强的策略，遵循当前的保护 Pod 的最佳实践。

## Profile 细节

### Privileged

Privileged 策略是**有意开放**，且完全无限制的策略。

此类策略通常针对由**特权较高、受信任的用户**所管理的系统级或基础设施级负载。

Privileged 策略定义中限制较少：

- 默认允许的（Allow-by-default）实施机制可以缺省设置为 Privileged。

### Baseline

Baseline 策略的目标是便于常见的容器化应用采用，同时禁止已知的特权提升。此策略针对的是**应用运维人员和非关键性应用的开发人员**。

下面列举的控制应该被实施或禁止：

**说明：**

- 在下述表格中，通配符（\*）意味着一个数组中的所有元素。 例如 `spec.containers[*].securityContext` 表示所定义的所有容器的安全性上下文对象。
- 如果 Pod 所列出的任一容器不能满足要求，整个 Pod 将无法通过校验。

#### Control：HostProcess

Windows Pod 提供了运行 HostProcess 容器的能力，这使得容器对 Windows 节点的特权访问成为可能。

**Baseline 策略中禁止容器对宿主的特权访问**。

限制的字段：

```txt
spec.securityContext.windowsOptions.hostProcess
spec.containers[*].securityContext.windowsOptions.hostProcess
spec.initContainers[*].securityContext.windowsOptions.hostProcess
spec.ephemeralContainers[*].securityContext.windowsOptions.hostProcess
```

准许的取值：

- 未定义、nil
- false

#### Control：宿主名字空间

必须禁止共享宿主上的 namespace。

限制的字段：

```txt
spec.hostNetwork
spec.hostPID
spec.hostIPC
```

准许的取值：

- 未定义、nil
- false

#### Control：特权容器

特权 Pod 会使大多数安全性机制失效，必须被禁止。

限制的字段：

```txt
spec.containers[*].securityContext.privileged
spec.initContainers[*].securityContext.privileged
spec.ephemeralContainers[*].securityContext.privileged
```

准许的取值：

- 未定义、nil
- false

#### Control：Capabilities

必须禁止添加超出以下所列功能的其他功能。

限制的字段：

```txt
spec.containers[*].securityContext.capabilities.add
spec.initContainers[*].securityContext.capabilities.add
spec.ephemeralContainers[*].securityContext.capabilities.add
```

准许的取值：

- 未定义、nil
- AUDIT_WRITE
- CHOWN
- DAC_OVERRIDE
- FOWNER
- FSETID
- KILL
- MKNOD
- NET_BIND_SERVICE
- SETFCAP
- SETGID
- SETPCAP
- SETUID
- SYS_CHROOT

#### Control：HostPath 卷

必须禁止 HostPath 卷。

限制的字段：

```txt
spec.volumes[*].hostPath
```

准许的取值：

- 未定义、nil

#### Control：宿主端口

应该禁止使用宿主端口，或者至少限制只能使用某确定列表中的端口。

限制的字段：

```txt
spec.containers[*].ports[*].hostPort
spec.initContainers[*].ports[*].hostPort
spec.ephemeralContainers[*].ports[*].hostPort
```

准许的取值：

- 未定义、nil
- 已知列表
- 0

#### Control：AppArmor	

在受支持的主机上，默认使用 runtime/default AppArmor 配置。Baseline 策略应避免覆盖或者禁用默认策略，以及限制覆盖一些配置集合的权限。

限制的字段：

```txt
metadata.annotations["container.apparmor.security.beta.kubernetes.io/*"]
```

准许的取值

- 未定义、nil
- `runtime/default`
- `localhost/*`

#### Control：SELinux

设置 SELinux 类型的操作是被限制的，设置自定义的 SELinux 用户或角色选项是被禁止的。

限制的字段：

```txt
spec.securityContext.seLinuxOptions.type
spec.containers[*].securityContext.seLinuxOptions.type
spec.initContainers[*].securityContext.seLinuxOptions.type
spec.ephemeralContainers[*].securityContext.seLinuxOptions.type
```

准许的取值：

- 未定义、""
- container_t
- container_init_t
- container_kvm_t

限制的字段：

```txt
spec.securityContext.seLinuxOptions.user
spec.containers[*].securityContext.seLinuxOptions.user
spec.initContainers[*].securityContext.seLinuxOptions.user
spec.ephemeralContainers[*].securityContext.seLinuxOptions.user
spec.securityContext.seLinuxOptions.role
spec.containers[*].securityContext.seLinuxOptions.role
spec.initContainers[*].securityContext.seLinuxOptions.role
spec.ephemeralContainers[*].securityContext.seLinuxOptions.role
```

准许的取值：

- 未定义、""

#### Control：/proc挂载类型

要求使用默认的 /proc 掩码以减小攻击面。

限制的字段：

```txt
spec.containers[*].securityContext.procMount
spec.initContainers[*].securityContext.procMount
spec.ephemeralContainers[*].securityContext.procMount
```

准许的取值：

- 未定义、nil
- Default

#### Control：Seccomp

Seccomp 配置必须不能显式设置为 Unconfined。

限制的字段：

```txt
spec.securityContext.seccompProfile.type
spec.containers[*].securityContext.seccompProfile.type
spec.initContainers[*].securityContext.seccompProfile.type
spec.ephemeralContainers[*].securityContext.seccompProfile.type
```

准许的取值：

- 未定义、nil
- RuntimeDefault
- Localhost

#### Control：Sysctls

Sysctls 可以禁用安全机制或影响宿主上所有容器，因此除了若干“安全”的子集之外，应该被禁止。

如果某 sysctl 是受容器或 Pod 的 namespace 限制，且与节点上其他 Pod 或进程相隔离，可认为是安全的。

限制的字段：

```txt
spec.securityContext.sysctls[*].name
```

准许的取值：

- 未定义、nil
- kernel.shm_rmid_forced
- net.ipv4.ip_local_port_range
- net.ipv4.ip_unprivileged_port_start
- net.ipv4.tcp_syncookies
- net.ipv4.ping_group_range

### Restricted

Restricted 策略旨在实施**当前保护 Pod 的最佳实践**，尽管这样作可能会**牺牲一些兼容性**。

该类策略主要针对运维人员和安全性很重要的应用的开发人员，以及不太被信任的用户。下面列举的控制需要被实施（禁止）：

#### 继承

Restricted 策略继承了 Baseline 策略的所有要求。

#### Control：卷类型

除了限制 HostPath 卷之外，此类策略还限制可以通过 PersistentVolumes 定义的非核心卷类型。

限制的字段：

```txt
spec.volumes[*]
```

准许的取值（spec.volumes[*] 列表中的每个条目必须将下面字段之一设置为非空值）：

```txt
spec.volumes[*].configMap
spec.volumes[*].csi
spec.volumes[*].downwardAPI
spec.volumes[*].emptyDir
spec.volumes[*].ephemeral
spec.volumes[*].persistentVolumeClaim
spec.volumes[*].projected
spec.volumes[*].secret
```

#### Control：特权提升

禁止（通过 SetUID 或 SetGID 文件模式）获得特权提升。

限制的字段：

```txt
spec.containers[*].securityContext.allowPrivilegeEscalation
spec.initContainers[*].securityContext.allowPrivilegeEscalation
spec.ephemeralContainers[*].securityContext.allowPrivilegeEscalation
```

允许的取值：

- false

#### Control：限制以非 root 账号运行（Running as Non-root）

容器必须以非 root 账号运行。

限制的字段：

```txt
spec.securityContext.runAsNonRoot
spec.containers[*].securityContext.runAsNonRoot
spec.initContainers[*].securityContext.runAsNonRoot
spec.ephemeralContainers[*].securityContext.runAsNonRoot
```

准许的取值：

- true

如果 Pod 级别 spec.securityContext.runAsNonRoot 设置为 true，则允许容器组的安全上下文字段设置为 未定义/nil。

#### Control：非 root 用户

容器不可以将 runAsUser 设置为 0

限制的字段：

```txt
spec.securityContext.runAsUser
spec.containers[*].securityContext.runAsUser
spec.initContainers[*].securityContext.runAsUser
spec.ephemeralContainers[*].securityContext.runAsUser
```

准许的取值

- 所有的非零值
- undefined/null

#### Control：Seccomp (v1.19+)

Seccomp Profile 必须被显式设置成一个允许的值。禁止使用 Unconfined Profile 或者指定不存在的 Profile。

限制的字段：

```txt
spec.securityContext.seccompProfile.type
spec.containers[*].securityContext.seccompProfile.type
spec.initContainers[*].securityContext.seccompProfile.type
spec.ephemeralContainers[*].securityContext.seccompProfile.type
```

准许的取值：

- RuntimeDefault
- Localhost

#### Control：Capabilities (v1.22+)

容器必须弃用 ALL 权能，并且只允许添加 NET_BIND_SERVICE 权能。

限制的字段：

```txt
spec.containers[*].securityContext.capabilities.drop
spec.initContainers[*].securityContext.capabilities.drop
spec.ephemeralContainers[*].securityContext.capabilities.drop
```

准许的取值：

- 包括 ALL 在内的任意权能列表。

限制的字段：

```txt
spec.containers[*].securityContext.capabilities.add
spec.initContainers[*].securityContext.capabilities.add
spec.ephemeralContainers[*].securityContext.capabilities.add
```

准许的取值：

- 未定义、nil
- NET_BIND_SERVICE

## 策略实例化

将策略定义从策略实例中解耦出来有助于形成跨集群的策略理解和语言陈述，以免绑定到特定的下层实施机制。

[Pod 安全性准入控制器](https://kubernetes.io/zh-cn/docs/concepts/security/pod-security-admission/)

- [Privileged namespace](https://raw.githubusercontent.com/kubernetes/website/main/content/zh-cn/examples/security/podsecurity-privileged.yaml)
- [Baseline namespace](https://raw.githubusercontent.com/kubernetes/website/main/content/zh-cn/examples/security/podsecurity-baseline.yaml)
- [Restricted namespace](https://raw.githubusercontent.com/kubernetes/website/main/content/zh-cn/examples/security/podsecurity-restricted.yaml)

下面是针对 `Pod 安全性准入控制器` 示例：

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

[PodSecurityPolicy（已弃用）](https://kubernetes.io/zh-cn/docs/concepts/security/pod-security-policy/)

- [Privileged](https://raw.githubusercontent.com/kubernetes/website/main/content/zh-cn/examples/policy/privileged-psp.yaml)
- [Baseline](https://raw.githubusercontent.com/kubernetes/website/main/content/zh-cn/examples/policy/baseline-psp.yaml)
- [Restricted](https://raw.githubusercontent.com/kubernetes/website/main/content/zh-cn/examples/policy/restricted-psp.yaml)

下面是针对 PodSecurityPolicy 的示例：

```yaml
# privileged
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: privileged
  annotations:
    seccomp.security.alpha.kubernetes.io/allowedProfileNames: '*'
spec:
  privileged: true
  allowPrivilegeEscalation: true
  allowedCapabilities:
  - '*'
  volumes:
  - '*'
  hostNetwork: true
  hostPorts:
  - min: 0
    max: 65535
  hostIPC: true
  hostPID: true
  runAsUser:
    rule: 'RunAsAny'
  seLinux:
    rule: 'RunAsAny'
  supplementalGroups:
    rule: 'RunAsAny'
  fsGroup:
    rule: 'RunAsAny'

# restricted
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: restricted
  annotations:
    # docker/default 标识 seccomp 的配置文件，但它与 Docker 运行时没有特别关联
    seccomp.security.alpha.kubernetes.io/allowedProfileNames: 'docker/default,runtime/default'
    apparmor.security.beta.kubernetes.io/allowedProfileNames: 'runtime/default'
    apparmor.security.beta.kubernetes.io/defaultProfileName:  'runtime/default'
spec:
  privileged: false
  # 防止权限升级到 root
  allowPrivilegeEscalation: false
  requiredDropCapabilities:
    - ALL
  # 允许的核心卷类型.
  volumes:
    - 'configMap'
    - 'emptyDir'
    - 'projected'
    - 'secret'
    - 'downwardAPI'
    # 假设集群管理员设置的临时 CSI 驱动程序和持久卷可以安全使用
    - 'csi'
    - 'persistentVolumeClaim'
    - 'ephemeral'
  hostNetwork: false
  hostIPC: false
  hostPID: false
  runAsUser:
    # 要求容器在没有 root 权限的情况下运行
    rule: 'MustRunAsNonRoot'
  seLinux:
    # 此策略假定节点使用 AppArmor 而不是 SELinux
    rule: 'RunAsAny'
  supplementalGroups:
    rule: 'MustRunAs'
    ranges:
      # 禁止添加 root 组
      - min: 1
        max: 65535
  fsGroup:
    rule: 'MustRunAs'
    ranges:
      # 禁止添加 root 组
      - min: 1
        max: 65535
  readOnlyRootFilesystem: false

```

### 替代方案

在 Kubernetes 生态系统中还在开发一些其他的替代方案，例如：

- Kubewarden
- Kyverno
- OPA Gatekeeper

## 常见问题

### 为什么不存在介于 Privileged 和 Baseline 之间的策略类型？

这里定义的三种策略框架有一个明晰的线性递进关系，从最安全（Restricted）到最不安全，并且覆盖了很大范围的工作负载。

Privileged 要求超出 Baseline 策略者通常是特定于应用的需求，所以我们没有在这个范围内提供标准框架。这并不意味着在这样的情形下仍然只能使用 Privileged 框架， 只是说处于这个范围的策略需要因地制宜地定义。

SIG Auth 可能会在将来考虑这个范围的框架，前提是有对其他框架的需求。

### Security Profile 与 Security Context 的区别是什么？

Security Context 在运行时配置 Pod 和容器。Security Context是在 Pod Spec 中容器的一部分的，所代表的是传递给容器运行时的参数。

Security profiles 是 Control Panel 用来对 Security Context 参数实施某种设置的机制（这些安全策略就是用来限制 Security Context 的，同时也不仅仅可以限制 Security Context）。

### 我应该为我的 Windows Pod 实施哪种框架？

Kubernetes 中的 Windows 负载与标准的基于 Linux 的负载相比有一些局限性和区别。 尤其是 Pod SecurityContext 字段对 Windows 不起作用。 因此，目前没有对应的标准 Pod 安全性框架。

如果你为一个 Windows Pod 应用了 Restricted 策略，可能会 对该 Pod 的运行时产生影响。 Restricted 策略需要强制执行 Linux 特有的限制（如 seccomp Profile，并且禁止特权提升）。 如果 kubelet 和/或其容器运行时忽略了 Linux 特有的值，那么应该不影响 Windows Pod 正常工作。 然而，对于使用 Windows 容器的 Pod 来说，缺乏强制执行意味着相比于 Restricted 策略，没有任何额外的限制。

你应该只在 Privileged 策略下使用 HostProcess 标志来创建 HostProcess Pod。 在 Baseline 和 Restricted 策略下，创建 Windows HostProcess Pod 是被禁止的， 因此任何 HostProcess Pod 都应该被认为是有特权的。

### 沙箱（Sandboxed）Pod 怎么处理？

现在还没有 API 标准来控制 Pod 是否被视作沙箱化 Pod。沙箱 Pod 可以通过其是否使用沙箱化运行时（如 gVisor 或 Kata Container）来辨别， 不过目前还没有关于什么是沙箱化运行时的标准定义。

沙箱化负载所需要的保护可能彼此各不相同。例如，当负载与下层内核直接隔离开来时， 限制特权化操作的许可就不那么重要。这使得那些需要更多许可权限的负载仍能被有效隔离。

此外，沙箱化负载的保护高度依赖于沙箱化的实现方法。 因此，现在还没有针对所有沙箱化负载的建议配置。

本页面中的条目引用了第三方产品或项目，这些产品（项目）提供了 Kubernetes 所需的功能。Kubernetes 项目的开发人员不对这些第三方产品（项目）负责。

在提交更改建议，向本页添加新的第三方链接之前，你应该先阅读内容指南。
