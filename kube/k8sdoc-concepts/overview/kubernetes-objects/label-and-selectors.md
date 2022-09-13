# 标签和选择器

标签（Labels） 是附加到 Kubernetes 对象（比如 Pod）上的键值对。

Labels 旨在用于指定对用户有意义且相关的对象的标识属性，但不核心系统并不直接理解这些标签的语义。

标签的主要作用是：

- 标签可以用于组织和选择对象的子集。

标签的特性：

- 标签可以在创建时附加到对象，随后可以随时添加和修改。
- 每个对象都可以定义一组 kv 标签，每个 key 对于给定对象必须是唯一的。

例如：

```json
"metadata": {
  "labels": {
    "key1" : "value1",
    "key2" : "value2"
  }
}
```

标签方便高效的查询和监控 Kubernetes 的对象。对于不需要用于高效查询的 kv 对，应该使用注解（annotations）来进行记录。

## 动机

标签使用户能够以松散耦合的方式将他们自己的组织结构映射到系统对象，而无需客户端存储这些映射。

服务部署和批处理流水线通常是多维对象（例如，多个分区或部署、多个发行序列、多个层，每层多个微服务）。

对于这些 Kubernetes 对象的管理通常需要多个维度的交叉操作，这打破了严格的层次表示的封装（即不适合树形结构）。

示例标签：

Labels | Values
-|-
"release" | "stable", "canary"
"environment" | "dev", "qa", "production"
"tier" | "frontend", "backend", "cache"
"partition" | "customerA", "customerB"
"track" | "daily", "weekly"

有一些[常用标签](https://kubernetes.io/zh-cn/docs/concepts/overview/working-with-objects/common-labels/)的例子；

当然，你也可以任意制定自己的约定。

请记住：

- 标签的 Key 对于给定对象必须是唯一的。

## 语法和字符集

标签是键值对。

有效的`标签键(Key)`有两个段：前缀（可选）和名称（必需），它们之间用斜杠（/）分隔。

- 前缀是可选的。如果指定，前缀必须是 DNS 子域。具体字符集请参考 [语法和字符集](https://kubernetes.io/zh-cn/docs/concepts/overview/working-with-objects/labels/#syntax-and-character-set)。一般用以表示该 Key 是哪个组件使用或者标记上的。如果省略前缀，则假定标签键对用户是私有的（即其他组件不会使用或标记的）。这只是一种约定。
- 名称段是必需的。具体字符集请参考 [语法和字符集](https://kubernetes.io/zh-cn/docs/concepts/overview/working-with-objects/labels/#syntax-and-character-set)。

向用户的 Kubernetes 对象添加标签的自动系统组件（例如 kube-scheduler、kube-controller-manager、 kube-apiserver、kubectl 或其他第三方自动化工具）必须指定前缀。此外：

- `kubernetes.io/` 和 `k8s.io/` 前缀是为 Kubernetes 核心组件保留的。其他组件不要使用。

kubernetes.io/ 和 k8s.io/ 前缀是为 Kubernetes 核心组件保留的。

有效标签值：

必须为 63 个字符或更少（可以为空）
除非标签值为空，必须以字母数字字符（[a-z0-9A-Z]）开头和结尾
包含破折号（-）、下划线（_）、点（.）和字母或数字

例如，这是一个有 environment: production 和 app: nginx 标签的 Pod 配置文件：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: label-demo
  labels:
    environment: production
    app: nginx
spec:
  containers:
  - name: nginx
    image: nginx:1.14.2
    ports:
    - containerPort: 80
```

## 标签选择器

与名称和 UID 不同， 标签不支持唯一性。通常，我们希望许多对象携带相同的标签。

通过 Labels Selector，可以识别一组对象（Selectors 是 Kubernetes 中的核心分组原语）

API 目前支持两种类型的 Selectors：

- 基于等值的 Selector
- 基于集合的 Selector

Selector 可以由逗号分隔的多个需求组成。在多个需求的情况下，必须满足所有要求，因此逗号分隔符充当逻辑与（&&）运算符。

**注意：**

- 对于基于等值的和基于集合的条件而言，不存在逻辑或（||）操作符。 你要确保你的过滤语句按合适的方式组织。
- 对于某些对象（例如 ReplicaSet），在进行 Selector 选择时，不能选择相同命名空间下的相同标签，否则不同参数的 ReplicaSet 对选择的 Pods 进行控制时会出现矛盾。

当 Selector 为空，或者未指定时，其语义取决于上下文。API 支持 Selector 是，应该将其合法性和含义用文档记录下来。

### 基于等值的 Selector

`基于等值`或`基于不等值`的 Selector 允许按标签键和值进行过滤。

匹配对象必须满足所有指定的标签约束，尽管它们也可能具有其他标签。

可接受的运算符有 =、== 和 != 三种。 前两个表示相等（并且是同义词），而后者表示不相等。例如：

```txt
environment = production
environment == production    # 和上面一个完全等价
tier != frontend
```

其中：

- 前者选择 Kubernetes 对象的标签需要满足：其键名等于 environment，值等于 production。
- 后者选择 Kubernetes 对象的标签需要满足：其键名等于 tier，值不同于 frontend，以及所有不带 tier 标签的资源（不带 tier 的对象，可以看作带了 tier ，但值为空）。

可以使用逗号运算符来过滤 production 环境中的非 frontend 层资源：

- `environment=production,tier!=frontend`

基于等值的标签要求的一种使用场景是 Pod 要指定节点的选择标准（Pod 运行在具有什么标签的节点上）。

例如，下面的示例 Pod 选择带有标签 "accelerator=nvidia-tesla-p100" 的节点来运行：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: cuda-test
spec:
  containers:
    - name: cuda-test
      image: "registry.k8s.io/cuda-vector-add:v0.1"
      resources:
        limits:
          nvidia.com/gpu: 1
  nodeSelector:
    accelerator: nvidia-tesla-p100
```

### 基于集合的 Selector

基于集合的标签需求允许你通过一组值来过滤键。

集合支持三种操作符：

- in
- notin
- exists

例如：

```txt
environment in (production, qa)
tier notin (frontend, backend)
partition
!partition
```

解释：

- 第一个示例选择了所有键等于 environment 并且值等于 production 或者 qa 的资源。
- 第二个示例选择了所有键等于 tier 并且值不等于 frontend 或者 backend 的资源，以及所有没有 tier 键标签的资源。
- 第三个示例选择了所有包含了有 partition 标签的资源；没有校验它的值。
- 第四个示例选择了所有没有 partition 标签的资源；没有校验它的值。

类似地，逗号分隔符充当`与`运算符。

**注意：**

- `基于集合的 Selector`是`基于相等的 Selector`的一般形式，因为 `environment=production` 等同于 `environment in (production)`。此外，`!=` 和 `notin` 也是类似的。
- `基于集合的 Selector`可以与`基于相等的 Selector`混合使用。例如：`partition in (customerA, customerB),environment!=qa`。

## API

这里 API 指的是对 Kubernetes 可以接受到的请求。下文中会提到 API 对象，API 对象本质上就是指的 Kubernetes 对象。

### LIST 和 WATCH 过滤

LIST 和 WATCH 操作可以使用查询参数 (Query) 指定标签 Selector 过滤一组对象。

可以有以下两种形式：

- 基于等值的需求：`?labelSelector=environment%3Dproduction,tier%3Dfrontend`
- 基于集合的需求：`?labelSelector=environment+in+%28production%2Cqa%29%2Ctier+in+%28frontend%29`


两种标签选择算符都可以通过 REST 客户端用于对资源进行 list 或者 watch。

例如，使用 kubectl 对满足指定标签的 Pods 进行 list 操作：

```sh
# 基于等值的标签选择器
kubectl get pods -l environment=production,tier=frontend

# 基于集合的标签选择器
kubectl get pods -l 'environment in (production),tier in (frontend)'
```

正如刚才提到的，基于集合的需求更具有表达力：

```sh
# 它们可以实现值的或操作
kubectl get pods -l 'environment in (production, qa)'

# 或者通过 exists 运算符限制不匹配
kubectl get pods -l 'environment,environment notin (frontend)'
```

### 在 API 对象中设置引用

一些 Kubernetes 对象，例如 `services` 和 `replicationcontrollers`， 也使用了标签选择算符去指定了其他资源的集合，例如 pods。

例如：

- 一个 Service 指向的哪一组 Pods 是由标签选择算符定义的。
- 一个 ReplicationController 应该管理哪些 Pods，也是由标签选择算符定义的。

Json 格式：

```json
"selector": {
    "component" : "redis",
}
```

Yaml 格式：

```yaml
selector:
    component: redis
```

但需要**注意：**

- 对于 Service 和 ReplicationControllers 这两个对象，只支持`基于等值的 Selector`。


比较新的资源，例如 Job、 Deployment、 ReplicaSet 和 DaemonSet， 也支持`基于集合的 Selector`：

```yaml
selector:
  matchLabels:
    component: redis
  matchExpressions:
    - {key: tier, operator: In, values: [cache]}
    - {key: environment, operator: NotIn, values: [dev]}
```