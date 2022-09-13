# 字段选择器

“字段选择器（Field selectors）”允许你根据一个或多个资源字段的值，筛选 Kubernetes 资源。 下面是一些使用字段选择器查询的例子：

```txt
metadata.name=my-service
metadata.namespace!=default
status.phase=Pending
```

下面这个 kubectl 命令将筛选出 status.phase 字段值为 Running 的所有 Pod：

```sh
kubectl get pods --field-selector status.phase=Running
```

**注意：**

- 字段选择器本质上是资源 “过滤器（Filters）”，即从所有的对象中过滤（Label 才是真正的查找，会创建索引）。
- 默认情况下，字段选择器是未被使用的（即没有指定 --field-selector 参数），这意味着指定类型的所有资源都会被筛选出来。这使得以下的两个 kubectl 查询是等价的：

```sh
kubectl get pods
kubectl get pods --field-selector ""
```

## 支持的字段

不同的 Kubernetes 资源类型支持不同的字段选择器。

所有资源类型都支持 **metadata.name** 和 **metadata.namespace** 字段。

使用不被支持的字段选择器会产生错误。例如：

```sh
$ kubectl get ingress --field-selector foo.bar=baz
Error from server (BadRequest): Unable to find "ingresses" that match label selector "", field selector
```

## 支持的操作符

你可在字段选择器中使用 =、== 和 != （= 和 == 的意义是相同的）操作符。

例如，下面这个 kubectl 命令将筛选所有不属于 default 命名空间的 Kubernetes 服务：

```sh
kubectl get services  --all-namespaces --field-selector metadata.namespace!=default
```

## 链式选择器

同标签和其他选择器一样，字段选择器可以通过使用逗号分隔的列表组成一个选择链。

下面这个 kubectl 命令将筛选同时满足下面两个条件的 Pods：

- status.phase 字段不等于 Running
- spec.restartPolicy 字段等于 Always：

```sh
kubectl get pods --field-selector=status.phase!=Running,spec.restartPolicy=Always
```

## 多种资源类型

你能够跨多种资源类型来使用字段选择器。 下面这个 kubectl 命令将筛选出所有不在 default 命名空间中的 StatefulSet 和 Service：

```sh
kubectl get statefulsets,services --all-namespaces --field-selector metadata.namespace!=default
```
