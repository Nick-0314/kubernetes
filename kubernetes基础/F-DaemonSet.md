# DaemonSet

DaemonSet(守护进程集)和守护进程类似，它在符合匹配条件的节点上均部署一个Pod

## 1 什么是DaemonSet

DaemonSet确保全部（或者某些）节点上运行一个Pod副本。当有新节点加入集群时，也会为它们新增一个Pod。当节点从集群中移除时，这些Pod也会被回收，删除DaemonSet将会删除它创建的所有Pod

使用DaemonSet的一些典型用法：

- 运行集群存储daemon（守护进程），例如在每个节点上运行Glusterd，Ceph等。
- 在每个节点运行日志收集daemon,例如Fluentd，Logstash。
- 在每个节点上运行监控daemon，比如Prometheus Node Exporter ，Collectd，Datadog代理，New Relic代理或Ganglia gmond。

## 2 编写DaemonSet规范

创建一个DaemonSet的内容大致如下，比如创建一个fluentd的DaemonSet

本文中的fluentd为EFK中的F



先下载镜像（所有节点都下载或者推送到私有仓库）

```
docker pull fluent/fluentd-kubernetes-daemonset:v1.7-debian-s3-1
```



编写yaml

```
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-es-v1.7
  namespace: default
  labels:
    K8S-app: fluentd-es
    version: v1.7
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
spec:
  selector:
    matchLabels:
      K8S-app: fluentd-es
      version: v1.7
  template:
    metadata:
      labels:
        K8S-app: fluentd-es
        version: v1.7
        kubernetes.io/cluster-service: "true"
  # this is annotation ensures that fluentd does not get evicted if the node
  # supports critical pod annotation based priority scheme.
  # Note that this does net guarantee admission on the nodes (#40573)
      annotations:
        scheduler.alpha.kubernetes.io/critical-pod: ''
        seccomp.security.alpha.kubernetes.io/pod: 'docker/default'
    spec:
        serviceAccountName: fluentd-es
        containers:
        - name: fluentd-es
          image: 192.168.10.66:5000/fluentd-kubernetes-daemonset:v1.7
          env:
          - name: FLUENTD_ARGS
            value: --no-supervisor -q
          resources:
            limits:
              memory: 500Mi
            requests:
              cpu: 100m
              memory: 200Mi
          volumeMounts:
          - name: varlog
            mountPath: /var/log
          - name: varlibdockercontainers
            mountPath: /var/lib/docker/containers
            readOnly: true
          - name: config-volume
            mountPath: /etc/fluent/config.d
        nodeSelector:
          beta.kubernetes.io/fluentd-ds-ready: "true"
        terminationGracePeriodSeconds: 30
        volumes:
        - name: varlog
          hostPath:
            path: /var/log
        - name: varlibdockercontainers
          hostPath:
            path: /var/lib/docker/containers
        - name: config-volume
          configMap:
            name: fluentd-es-config-v0.1.7
```

1. 必需字段

   和其他所有Kubernetes配置一样，DaemonSet需要apiVersion,kind和metadata字段，同时也需要一个.spec配置段。

2. Pod模板

   .spec唯一需要的字段是.spec.template。 .spec.template是一个Pod模板，它与Pod具有相同的配置方式，但它不具有apiVersion和kind字段

   除了Pod必需的字段外，在DaemonSet中的Pod模板必须指定合理的标签。

   在DaemonSet中的Pod模板必须具有一个restartPolicy,默认为Always。

3. Pod Selector

   .spec.selector字段表示Pod Selector，它与其他资源的.spec.selector的作用相同

   .spec.selector表示一个对象，它有如下两个字段组成：

   - matchLabels,与ReplicationController的.spec.selector的作用相同，用于匹配符合条件的Pod
   - matchExpressions,允许构建更加复杂的Selector，可以通过指定key,value列表以及与key和value列表相关的操作符。

   如果上述两个字段都指定时，结果表示的是AND关系（逻辑与的关系）。

   .spec.selector必须与.spec.template.metadata.lables相匹配。如果没有指定，默认是等价的，如果它们的配置不匹配，则会被API拒绝。

4. 指定节点部署Pod

   如果指定了.spec.template.spec.nodeSelector，DaemonSet Controller将在与Node Selector（节点选择器）匹配的节点上创建Pod，比如部署在磁盘类型为ssd的节点上（需要提前给节点定义标签Label）例如

   ```
       containers:
       - name: nginx
         image: nginx
         imagePullPolicy: IfNotPresent
       nodeSelector:
         disktype: ssd
   ```

   提示 Node Selector 同样适用与其他Controller （控制器）

## 3 创建DaemonSet

在生产环境中，公司业务的应用程序一般无需使用DaemonSet部署，一般情况下只有像Fluentd（日志收集）,Ingress(集群服务入口)，Calico（集群网络组件），Node-Exporter（监控数据采集），等才需要使用DaemonSet部署到每个节点，

本节只演示DaemonSet的使用

比如创建一个nginx的DaemonSet（最基础的DaemonSet）

yaml文件：

```
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: nginx-daemonset
spec:
  selector:
   matchLabels:
     apps: nginxdaemonset
  template:
    metadata:
      labels:
        apps: nginxdaemonset
    spec:
      containers:
      - name: nginx-ds
        image: nginx
```

创建DaemonSet

```
kubectl apply -f nginx-ds.yaml
```

输出信息

```
daemonset.apps/nginx-daemonset created
```

查看创建的DaemonSet

```
kubectl get daemonsets.apps
```

输出信息

```
NAME              DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
nginx-daemonset   3         3         3       3            3           <none>          102s
```

查看创建的Pod

```
kubectl get pods -o wide
```

输出信息 此时在每个节点创建了一个Pod

```
NAME                             READY   STATUS    RESTARTS   AGE   IP     NAME                                 READY   STATUS    RESTARTS   AGE     IP               NODE     NOMINATED NODE   READINESS GATES
nginx-daemonset-5kvc5                1/1     Running   0          2m12s   10.244.104.8     node2    <none>           <none>
nginx-daemonset-bbj45                1/1     Running   0          2m12s   10.244.166.143   node1    <none>           <none>
nginx-daemonset-vz8b9                1/1     Running   0          2m12s   10.244.219.75    master   <none>           <none>

```

由于我的kubernetes集群是使用二进制方法安装的，所以主节点没有污点Taint

在生产环境下 为了保障主节点的调度能力 最好加上污点 不要把pod调度到主节点上

## 4 更新和回滚DaemonSet

如果修改了节点标签（Label），DaemonSet将立刻向新匹配上的节点添加Pod，同时删除不能匹配的节点上的Pod

在kubernetes1.6以后的版本中，可以在DaemonSet上执行滚动更新，未来的Kubernetes版本将支持节点的可控更新

1 DaemonSet滚动更新

DaemonSet的更新策略和StatefulSet类似，也有OnDelete和RollingUpdate两种方式

参考 https://kubernetes.io/docs/tasks/manage-daemon/update-daemon-set/

### 检查DaemonSet `RollingUpdate `更新策略

```
kubectl get daemonsets.apps -n ingress-nginx nginx-ingress-controller  -o go-template='{{.spec.updateStrategy.type}}{{"\n"}}'
```

输出信息

```
RollingUpdate
```



命令式更新

```
kubectl edit ds/<daemonset-name>

kubectl patch ds/<daemonset-name> -p=<strategic-merge-patch>
```



例如 更新镜像格式

```
kubectl set image ds/<daemonset-name> <container-name>=<container-new-image>
```

```
kubectl set image daemonset -n  ingress-nginx nginx-ingress-controller nginx-ingress-controller=nginx
```

输出信息

```
daemonset.apps/nginx-ingress-controller image updated
```

查看滚动更新状态

```
kubectl rollout status daemonset -n  ingress-nginx nginx-ingress-controller
```

输出信息类似与StatefulSet的查看滚动更新状态 会一直占用一个终端 直到更新完成





列出修订版本

```
kubectl rollout history daemonset <daemonset-name>
```

回滚到指定版本　revision

```
kubectl rollout undo daemonset <daemonset-name> --to-revision=<revision>
```

DaemonSet的更新与回滚与Deployment类似,此处不再演示