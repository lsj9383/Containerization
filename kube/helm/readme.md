# Helm

## 安装

## 内置对象速记

- Release：这个对象描述了 release 本身。
- Values：从values.yaml文件和用户提供的文件传入模板的值。默认情况下，Values 是空的。
- Chart：Chart.yaml文件的内容。
- Capabilities：这提供了关于 Kubernetes 集群支持的功能的信息。

## 命令速记

创建本地默认的 Chart

```sh
$ helm create hello-helm
Creating hello-helm
```

列出 helm 的仓库

```sh
$ helm repo list
NAME    URL
stable  https://kubernetes-charts.storage.googleapis.com/
```

从仓库中搜索对应的 chart

```sh
$ helm search repo
NAME                                    CHART VERSION   APP VERSION             DESCRIPTION
stable/acs-engine-autoscaler            2.2.2           2.1.1                   DEPRECATED Scales worker nodes within agent pools
stable/aerospike                        0.3.2           v4.5.0.5                A Helm chart for Aerospike in Kubernetes
stable/airflow                          6.5.0           1.10.4                  Airflow is a platform to programmatically autho...
stable/ambassador                       5.3.1           0.86.1                  A Helm chart for Datawire Ambassador
stable/anchore-engine                   1.4.4           0.6.1                   Anchore container analysis and policy evaluatio...
stable/apm-server                       2.1.5           7.0.0                   The server receives data from the Elastic APM a...
stable/ark                              4.2.2           0.10.2                  DEPRECATED A Helm chart for ark
...
```

仅下载但不安装

```sh
$ helm pull stable/redis
```

安装本地 Chart

```sh
$ helm install hello-helm-xx ./hello-helm
NAME: hello-helm-xx
LAST DEPLOYED: Sun Nov  1 11:06:59 2020
NAMESPACE: default
STATUS: deployed
REVISION: 1
NOTES:
1. Get the application URL by running these commands:
  export POD_NAME=$(kubectl get pods --namespace default -l "app.kubernetes.io/name=hello-helm,app.kubernetes.io/instance=hello-helm-xx" -o jsonpath="{.items[0].metadata.name}")
  echo "Visit http://127.0.0.1:8080 to use your application"
  kubectl --namespace default port-forward $POD_NAME 8080:80
```

列出当前 kube 集群中安装并运行的 Chart

```sh
$ helm list
NAME            NAMESPACE       REVISION        UPDATED                                 STATUS          CHART                   APP VERSION
hello-helm-xx   default         1               2020-11-01 10:55:24.095014362 +0800 CST deployed        hello-helm-0.1.0        1.16.0
```

查看 hello-helm-xx 状态

```sh
$ helm status hello-helm-xx
NAME: hello-helm-xx
LAST DEPLOYED: Sun Nov  1 11:06:59 2020
NAMESPACE: default
STATUS: deployed
REVISION: 1
NOTES:
1. Get the application URL by running these commands:
  export POD_NAME=$(kubectl get pods --namespace default -l "app.kubernetes.io/name=hello-helm,app.kubernetes.io/instance=hello-helm-xx" -o jsonpath="{.items[0].metadata.name}")
  echo "Visit http://127.0.0.1:8080 to use your application"
  kubectl --namespace default port-forward $POD_NAME 8080:80
```

删除安装的 Chart（会自动删除相应的 kube 资源）

```sh
$ helm delete hello-helm-xx
release "hello-helm-xx" uninstalled
```

从 Repo 安装 Chart

```sh
# helm install -f config.yaml test-helm-mysql stable/mysql
$ helm install test-helm-mysql stable/mysql
NAME: test-helm-mysql
LAST DEPLOYED: Sun Nov  1 11:08:52 2020
NAMESPACE: default
STATUS: deployed
REVISION: 1
NOTES:
MySQL can be accessed via port 3306 on the following DNS name from within your cluster:
test-helm-mysql.default.svc.cluster.local

To get your root password run:

    MYSQL_ROOT_PASSWORD=$(kubectl get secret --namespace default test-helm-mysql -o jsonpath="{.data.mysql-root-password}" | base64 --decode; echo)

To connect to your database:

1. Run an Ubuntu pod that you can use as a client:

    kubectl run -i --tty ubuntu --image=ubuntu:16.04 --restart=Never -- bash -il

2. Install the mysql client:

    $ apt-get update && apt-get install mysql-client -y

3. Connect using the mysql cli, then provide your password:
    $ mysql -h test-helm-mysql -p

To connect to your database directly from outside the K8s cluster:
    MYSQL_HOST=127.0.0.1
    MYSQL_PORT=3306

    # Execute the following command to route the connection:
    kubectl port-forward svc/test-helm-mysql 3306

    mysql -h ${MYSQL_HOST} -P${MYSQL_PORT} -u root -p${MYSQL_ROOT_PASSWORD}
```

查看 Chart 的 Values 信息

```sh
$ helm inspect values hello-helm
# Default values for hello-helm.
# This is a YAML-formatted file.
# Declare variables to be passed into your templates.

replicaCount: 1

image:
  repository: nginx
  pullPolicy: IfNotPresent

imagePullSecrets: []
nameOverride: ""
fullnameOverride: ""
...
```

查看安装 Chart 的版本信息：

```sh
$ helm history test-helm-mysql
REVISION        UPDATED                         STATUS          CHART           APP VERSION     DESCRIPTION
1               Sun Nov  1 11:23:11 2020        deployed        mysql-1.6.2     5.7.28          Install complete
```

回滚 Chart 到某个版本。

**注意：**

- 回滚时会产生新的版本号，如当前版本号为 3，回滚到版本 1，则当前会到版本号 4，且 Chart 使用版本号 1 的。
- 回滚成功后，`helm history` 中的 DESCRIPTION 显示为 `Rollback to 1`。

```sh
$ helm rollback test-helm-mysql 1
Rollback was a success! Happy Helming!
```

查看以安装 Chart 的实际被渲染出来的资源 manifest 信息。

```sh
# helm get manifest ${chart-name}
$ helm get manifest test-helm-mysql
---
# Source: mysql/templates/secrets.yaml
apiVersion: v1
kind: Secret
metadata:
  name: test-helm-mysql
  namespace: default
  labels:
    app: test-helm-mysql
    chart: "mysql-1.6.2"
...
```

预先生成 Chart 资源的 mainfest，这不会触发部署。

```sh
$ helm install --dry-run --debug . ./mychart
HOOKS:
MANIFEST:
---
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mychart-configmap
data:
  myvalue: "Hello World"
...
```
