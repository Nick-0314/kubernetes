# StatefulSet

## StatefulSet(有状态集)常用于部署有状态的且需要有序启动的应用程序

## 1 statefulset的基本概念

Statefulset主要用于管理有状态应用程序的工作负载API对象。比如在生产环境中，可以部署ElasticSearch集群，MongoDB集群或者需要持久化的RabbitMQ集群，Redis集群，kafka集群和Zookeeper集群等。

和Deployment类似，一个Statefulset也同样管理者基于相同容器规范的Pod。不同的是，StatefulSet为每个Pod维护了一个粘性标识。这些Pod是根据相同的规范创建的，但是不可交换，每个Pod都有一个持久的标识符，在重新调度时也会保留，一般格式为StatefulSetName-Number。比如定义一个名字是Redis——Sentinel的StatefulSet，指定创建三个Pod，那么创建出来的Pod名字就为Redis-Sentinel-0，Redis-Sentinel-1，Redis-Sentinel-2。而StatefulSet创建的Pod一般使用Headless Service（无头服务）进行通信，和普通的Service的区别在于Headless Service 没有ClusterIP，它使用的是Endpoint进行互相通信，Headless一般的格式为:

```
statefulSetName-(0..N-1).serviceName.namespace.svc.cluster.local
```

说明

- serviceName 为Headless service的名字
- 0..N-1为Pod所在的序号，从0开始到N-1。
- statefulSetName为StatefulSet的名字
- namespace为服务所在的命名空间
- .cluster.local为Cluster Domain（集群域）

比入，一个Redis主从架构，Slave连接Master主机配置就可以使用不会更改的Master的Headless Service，例如Redis从节点（Slave）配置文件如下：

```
port 6379
slaveofredis-sentinel-master-ss-0.redis-sentinel-master-ss.public-service.svc.cluster.local 6379
tcp-backlog 511
timeout 0
tcp-keepalive 0
.....
```

其中：slaveofredis-sentinel-master-ss-0.redis-sentinel-master-ss.public-service.svc.cluster.local 6379是Redis Master的Headless Service。

## 2 使用StatefulSet

一般StatefulSet用于以下一个或者多个需求的应用程序：

- 需要稳定的独一无二的网络标识符。
- 需要持久化数据
- 需要有序的 优雅的部署和扩展
- 需要有序的 自动滚动更新

如果应用程序不需要任何稳定的标识符或者有序的部署，删除或者扩展，应该使用无状态的控制器部署应用程序，比如Deployment或者ReplicaSet

### 3 StatefulSet的限制

StatefulSet是Kubernetes1.9版本之前的beta资源，在1.5版本之前的任何Kubernetes版本都没有

Pod所用的存储必须由PersistentVolume Provisioner（持久化卷配置器）根据请求配置StrongeClass，或者由管理员预先配置。

为了确保数据安全，删除和缩放StatefulSet不会删除与StatefulSet关联的卷，可以手动选择性的删除PVC和PV（关于PV和PVC可以参考Stronge节）

StatefulSet目前使用Headless Service（无头服务）负责Pod的网络身份和通信，但需要创建此服务。

删除一个StatefulSet时，不保证对Pod的终止，要在StatefulSet中实现对Pod的有序和正常终止，可以在删除之前将StatefulSet的副本缩减为0.

### 4 StatefulSet组件

定义一个简单的statefulset的示例如下：

```
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None
  selector:
    app: nginx
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "nginx"
  replicas: 2
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
        image: nginx
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "nginx-storage-class"
      resources:
        requests:
          storage: 1Gi
```

其中

- kind： Service 定义了一个名字为Nginx的Headless Service，创建的Service格式为nginx-0.nginx.default.svc.cluster.local，其他的类似，因为没有指定Namespace(命名空间)，所以默认部署在default。
- kind:StatefulSet定义了一个名字为web的StatefulSet，replicas表示部署Pod的副本数，本实例为2.
- volumeClaimTemplates表示将提供稳定的存储PV（持久化卷）作持久化。PV可以是手动创建或者自动创建。在上述示例中，每个Pod将配置一个PV，当Pod重新调度到某个节点上时，Pod会重新挂载volumeMounts指定的目录（当前statefulset挂载到/usr/share/nginx/html)，当删除Pod或者StatefulSet时，不会删除PV。

在Stateful中必须设置Pod选择器(.spec.seletor)用来匹配其标签(.spec.template.metadata.labels)。在1.8版本之前，如果为配置该字段（.spec.seletor），将被设置为默认值，在1.8版本之后，如果为指定匹配Pod Selector，则会导致StatefulSet创建错误。

当StatefulSet控制器创建Pod时，它会添加一个标签statefulset.kubernetes.io/pod-name，该标签的值为Pod的名称，用于匹配Service。

### 5 创建StatefulSet

创建StatefulSet之前，需要提前创建StatefulSet持久化所用的PersistentVolumes（持久化卷，以下简称PV,也可以使用emptyDir不对数据进行保留），当然也可以使用动态方式自动创建PV，关于PV会在后面进行详细整理 ，本节只作为演示使用，也可以先阅读后面

本例使用NFS提供静态PV，配置一台NFS服务器（在master主节点上配置 IP:192.168.10.10）

配置的共享目录如下

编辑nfs配置文件

```
echo "/nfs/web/ *(rw,sync,no_subtree_check,no_root_squash)" >> /etc/exports
```

创建目录

```
mkdir -p /nfs/web
mkdir /nfs/web/nginx{0..5}
```

启动nfs

```
exportfs -r
systemctl restart nfs-server
```



Nginx0-5作为StatefulSet Pod的PV的数据存储目录，使用PersistenVolume创建PV，文件如下

这是其中一个片段  把ngix-5 改成0-5 每个目录一个片段 即为整个文件

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-nginx-5
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: "nginx-storage-class"
  nfs:
    # real share directory
    path: /nfs/web/nginx5
    # nfs real ip
    server: 192.168.10.10
```





全部文件



```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-nginx-o
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: "nginx-storage-class"
  nfs:
    path: /nfs/web/nginx0
    server: 192.168.10.10
...
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-nginx-1
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: "nginx-storage-class"
  nfs:
    path: /nfs/web/nginx1
    server: 192.168.10.10
...
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-nginx-2
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: "nginx-storage-class"
  nfs:
    path: /nfs/web/nginx2
    server: 192.168.10.10
...
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-nginx-3
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: "nginx-storage-class"
  nfs:
    path: /nfs/web/nginx3
    server: 192.168.10.10
...
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-nginx-4
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: "nginx-storage-class"
  nfs:
    path: /nfs/web/nginx4
    server: 192.168.10.10
...
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-nginx-5
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: "nginx-storage-class"
  nfs:
    path: /nfs/web/nginx5
    server: 192.168.10.10
...                                                  
```

创建pv

```
kubectl create -f pv.yaml
```

输出信息

```
persistentvolume/pv-nginx-o created
persistentvolume/pv-nginx-1 created
persistentvolume/pv-nginx-2 created
persistentvolume/pv-nginx-3 created
persistentvolume/pv-nginx-4 created
persistentvolume/pv-nginx-5 created
```

查看PV

```
kubectl get pv
```

输出信息

```
NAME         CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS          REASON   AGE
pv-nginx-1   1Gi        RWO            Recycle          Available           nginx-storage-class            5s
pv-nginx-2   1Gi        RWO            Recycle          Available           nginx-storage-class            5s
pv-nginx-3   1Gi        RWO            Recycle          Available           nginx-storage-class            5s
pv-nginx-4   1Gi        RWO            Recycle          Available           nginx-storage-class            5s
pv-nginx-5   1Gi        RWO            Recycle          Available           nginx-storage-class            5s
pv-nginx-o   1Gi        RWO            Recycle          Available           nginx-storage-class            5s
```

创建staefulset

```
kubectl create -f StatefulSet-nginx.yaml
```

输出信息

```
service/nginx created
statefulset.apps/web created
```

查看创建的statefulset

```
kubectl get statefulsets.apps
```

输出信息

```
NAME   READY   AGE
web    2/2     97s
```

查看创建的service

```
kubectl get service
```

输出信息

```
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.250.0.1   <none>        443/TCP   8h
nginx        ClusterIP   None         <none>        80/TCP    2m49s
```

查看PVC和PV，可以看到Statefulset创建的两个Pod的PVC已经和PV绑定成功

```
kubectl get pvc
```

输出信息

```
NAME        STATUS   VOLUME       CAPACITY   ACCESS MODES   STORAGECLASS          AGE
www-web-0   Bound    pv-nginx-o   1Gi        RWO            nginx-storage-class   5m6s
www-web-1   Bound    pv-nginx-2   1Gi        RWO            nginx-storage-class   4m40s
```

```
kubectl get pv
```

输出信息

```
NAME         CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM               STORAGECLASS          REASON   AGE
pv-nginx-1   1Gi        RWO            Recycle          Available                       nginx-storage-class            21m
pv-nginx-2   1Gi        RWO            Recycle          Bound       default/www-web-1   nginx-storage-class            21m
pv-nginx-3   1Gi        RWO            Recycle          Available                       nginx-storage-class            21m
pv-nginx-4   1Gi        RWO            Recycle          Available                       nginx-storage-class            21m
pv-nginx-5   1Gi        RWO            Recycle          Available                       nginx-storage-class            21m
pv-nginx-o   1Gi        RWO            Recycle          Bound       default/www-web-0   nginx-storage-class            21m
```

查看创建的pod

```
kubectl get pod
```

输出信息 可以看到创建的pod的名字是由顺序 有规则的

```
NAME    READY   STATUS    RESTARTS   AGE
web-0   1/1     Running   0          7m13s
web-1   1/1     Running   0          6m59s
```



### 6 部署和扩展保障

pod的部署和扩展规则如下：

- 对于具有N个副本的StatefulSet，将按顺序从0到N-1开始创建Pod
- 当删除Pod时，将按照N-1到0的反顺序终止
- 在缩放Pod之前，必须保证当前的Pod是Running（运行中）或者Ready（就绪）。
- 在终止Pod之前，它所有的继任者必须是完全关闭状态。

StatefulSet的pod.spec.TerminationGracPeriodSeconds不应该指定为0,设置为0对StatefulSet的Pod是极其不安全的做法，优雅的删除StatefulSet的Pod是非常有必要的，而且是安全的，因为它可以确保在Kubelet从APIServer删除之前，让Pod正常关闭

当创建上面的Nginx实例时，Pod将按web-0 web-1 web-2的顺序部署3个Pod。在web-0处于Running或者Ready之前，web-1不会被部署，相同的，web-2在web-1未处于Running和Ready之前也不会被部署。如果在web-1处于Running和Ready状态时，web-0变成Failed（失败）状态，那么web-2将不会被启动，直到web-0恢复未Running和Ready状态。

如果用户将StatefulSet的replicas设置为1，那么web-2将首先被终止，在完全关闭并删除web-2之前，不会删除web-1.如果web-2终止并且完全关闭后，web-0突然失败，那么在web-0未恢复成Running或者Ready时，web-1不会被删除

### 7 StatefulSet扩容和缩容

和Deployment类似，可以通过更新replicas字段扩容/缩容StatefulSet，也可以使用kubect scale 或者kubectl patch 来扩容/缩容一个StatefulSet/

1 扩容

将上面创建的StatefulSet副本增加到5个（扩容之前必须保证有创建完成的静态PV。动态PV和emptyDir）：

```
kubectl scale statefulset web --replicas=5 --record
```

输出信息

```
statefulset.apps/web scaled
```

查看Pod及PVC的状态

```
kubectl get pvc
```

输出信息

```
NAME        STATUS   VOLUME       CAPACITY   ACCESS MODES   STORAGECLASS          AGE
www-web-0   Bound    pv-nginx-o   1Gi        RWO            nginx-storage-class   46m
www-web-1   Bound    pv-nginx-2   1Gi        RWO            nginx-storage-class   45m
www-web-2   Bound    pv-nginx-1   1Gi        RWO            nginx-storage-class   78s
www-web-3   Bound    pv-nginx-3   1Gi        RWO            nginx-storage-class   62s
www-web-4   Bound    pv-nginx-5   1Gi        RWO            nginx-storage-class   47s
```

```
kubectl get pods
```

输出信息

```
NAME    READY   STATUS    RESTARTS   AGE
web-0   1/1     Running   0          46m
web-1   1/1     Running   0          46m
web-2   1/1     Running   0          106s
web-3   1/1     Running   0          90s
web-4   1/1     Running   0          75s
```

也可以使用以下命令动态查看

```
kubectl get pods -w -l app=nginx
```

2 缩容：

在一个终端上动态查看：

```
kubectl get pods -w -l app=nginx
```

输出信息

```
NAME    READY   STATUS    RESTARTS   AGE
web-0   1/1     Running   0          48m
web-1   1/1     Running   0          48m
web-2   1/1     Running   0          4m4s
web-3   1/1     Running   0          3m48s
web-4   1/1     Running   0          3m33s
```

在另一个终端上将副本数改为3

```
kubectl patch statefulsets.apps web -p '{"spec":{"replicas":3}}'
```

输出信息

```
statefulset.apps/web patched
```

此时在另一个终端上可以看到web-4 web-3的Pod正在被删除（或终止）

```
NAME    READY   STATUS    RESTARTS   AGE
web-0   1/1     Running   0          48m
web-1   1/1     Running   0          48m
web-2   1/1     Running   0          4m4s
web-3   1/1     Running   0          3m48s
web-4   1/1     Running   0          3m33s
web-4   1/1     Terminating   0          5m41s
web-4   0/1     Terminating   0          5m43s
web-4   0/1     Terminating   0          5m52s
web-4   0/1     Terminating   0          5m52s
web-3   1/1     Terminating   0          6m7s
web-3   0/1     Terminating   0          6m9s
web-3   0/1     Terminating   0          6m10s
web-3   0/1     Terminating   0          6m10s
```

查看状态 此时PV和PVC不会被删除

```
kubectl get pods
```

输出信息

```
NAME    READY   STATUS    RESTARTS   AGE
web-0   1/1     Running   0          52m
web-1   1/1     Running   0          52m
web-2   1/1     Running   0          8m11s
```

```
kubectl get pvc
```

输出信息

```
NAME        STATUS   VOLUME       CAPACITY   ACCESS MODES   STORAGECLASS          AGE
www-web-0   Bound    pv-nginx-o   1Gi        RWO            nginx-storage-class   53m
www-web-1   Bound    pv-nginx-2   1Gi        RWO            nginx-storage-class   53m
www-web-2   Bound    pv-nginx-1   1Gi        RWO            nginx-storage-class   8m39s
www-web-3   Bound    pv-nginx-3   1Gi        RWO            nginx-storage-class   8m23s
www-web-4   Bound    pv-nginx-5   1Gi        RWO            nginx-storage-class   8m8s
```

```
kubectl get pv
```

输出信息

```
NAME         CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM               STORAGECLASS          REASON   AGE
pv-nginx-1   1Gi        RWO            Recycle          Bound       default/www-web-2   nginx-storage-class            69m
pv-nginx-2   1Gi        RWO            Recycle          Bound       default/www-web-1   nginx-storage-class            69m
pv-nginx-3   1Gi        RWO            Recycle          Bound       default/www-web-3   nginx-storage-class            69m
pv-nginx-4   1Gi        RWO            Recycle          Available                       nginx-storage-class            69m
pv-nginx-5   1Gi        RWO            Recycle          Bound       default/www-web-4   nginx-storage-class            69m
pv-nginx-o   1Gi        RWO            Recycle          Bound       default/www-web-0   nginx-storage-class            69m
```

### 8 更新策略

在kubernetes1.7以上的版本中，StatefulSet的.spec.updateStrategy字段运行配置和禁用容器的自动滚动更新，标签，资源限制以及StatefulSet中Pod的注释等。

1 On Delete策略

On Delete更新策略实现了传统（1.7版本之前）的行为，它也是默认的更新策略。当我们选择这个更新策略并修改StatefulSet的.spec.template的字段时，StatefulSet控制器不会自动更新Pod，我们必须手动删除Pod才能使控制器创建新的Pod

2 RollingUpdate策略

RollingUpdate（滚动更新）更新策略会更新一个StatefulSet中所有的Pod，采用与序号索引相反的顺序进行滚动更新。

比如Patch一个名为web的StatefulSet来执行RollingUpdate更新

```
kubectl patch statefulsets.apps web -p '{"spec":{"updateStrategy":{"type":"RollingUpdate"}}}'
```

输出信息

```
statefulset.apps/web patched (no change)
```

查看更改后的StatefulSet：

```
kubectl get statefulsets.apps web -o yaml | grep -3  updateStrategy:
```

输出信息

```
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
  updateStrategy:
    rollingUpdate:
      partition: 0
    type: RollingUpdate
```

然后改变容器的镜像进行滚动更新

先配置好私有仓库

更新

```
kubectl patch statefulsets.apps web --type='json' -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/image", "value": "192.168.10.66:5000/php"}]'
```

输出信息

```
statefulset.apps/web patched
```

如上所述，StatefulSet里的Pod采用和序号相反的顺序更新（即从2开始向0更新），在更新下一个Pod前，StatefulSet控制器会终止每一个Pod并等待它们变成Running和Ready状态。在当前顺序下的第一个被更新的Pod未变成Running和Ready状态之前，StatefulSet控制器不会更新下一个Pod，但它仍然会重建任何在更新过程中发生故障的Pod，使用它们当前的版本。已经接收到请求的Pod将会被恢复为更新的版本，没有收到请求的Pod则会被恢复为之前的版本。

在更新过程中可以使用kubectl rollout status sts web来查看滚动更新的状态

```
kubectl rollout status sts web
```

输出信息

```
Waiting for 1 pods to be ready...
Waiting for 1 pods to be ready...
Waiting for partitioned roll out to finish: 1 out of 3 new pods have been updated...
Waiting for 1 pods to be ready...
Waiting for 1 pods to be ready...
Waiting for partitioned roll out to finish: 2 out of 3 new pods have been updated...
Waiting for 1 pods to be ready...
Waiting for 1 pods to be ready...
partitioned roll out complete: 3 new pods have been updated...
```

查看Pod

```
kubectl get pods
```

输出信息 根据时间的先后关系可以看出来是从后向前开始更新的

```
NAME    READY   STATUS    RESTARTS   AGE
web-0   1/1     Running   0          39s
web-1   1/1     Running   0          93s
web-2   1/1     Running   0          2m12s
```

查看更新后的镜像

```
for p in 0 1 2; do kubectl get pods web-$p --template '{{range $i, $c := .spec.containers}}{{$c.image}}{{end}}'; echo; done
```

输出信息

```
192.168.10.66:5000/php
192.168.10.66:5000/php
192.168.10.66:5000/php
```

3 分段更新

StatefulSet可以使用RollingUpdate更新策略的partition参数来更新一个StatefulSet。分段更新将会使StatefulSet中其余的所有Pod（序号小于分区）保持当前版本，只更新序号大于等于分区的Pod，利用此特性可以简单实现金丝雀发布（灰度发布）或者分阶段推出新功能等。注：金丝雀发布是指在黑与白之间能够平滑过渡的一种发布方式。

比如我们定义一个分区"partition":3, 可以使用patch直接对StatefulSet进行设置：

```
kubectl patch statefulsets.apps web -p '{"spec":{"updateStrategy":{"type":"RollingUpdate", "rollingUpdate":{"partition":3}}}}'
```

输出信息

```
statefulset.apps/web patched
```

然后再次patch改变容器的镜像：

```
kubectl patch statefulsets.apps web --type='json' -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/image", "value": "nginx"}]'
```

输出信息

```
statefulset.apps/web patched
```

然后删除Pod触发更新

```
kubectl delete pod web-2
```

输出信息

```
pod "web-2" deleted
```

此时，因为Pod web-2的序号小于分区3，所有Pod不会被更新，还是会使用以前的容器恢复Pod

将分区改为2，此时会自动更新web-2（因为之前更改了更新策略），但是不会更新web-0和web-1

```
kubectl patch statefulsets.apps web -p '{"spec":{"updateStrategy":{"type":"RollingUpdate","rollingUpdate":{"partition":2}}}}'
```

输出信息

```
statefulset.apps/web patched
```

按照上述方式，可以实现分阶段更新，类似与灰度/金丝雀发布。查看的最终结果如下

```
for p in 0 1 2; do kubectl get pods web-$p --template '{{range $i, $c := .spec.containers}}{{$c.image}}{{end}}'; echo; done
```

输出镜像信息

```
192.168.10.66:5000/php
192.168.10.66:5000/php
nginx
```

### 9 删除StatefulSet

删除StatefulSet有两种方式，即级联删除和非级联删除。使用非级联方式删除StatefulSet时，StatefulSet的Pod不会被删除，使用级联删除时，StatefulSet和它的Pod都会被删除

1 非级联删除

使用kubectl delete statefulset name 删除StatefulSet时，只需提供--cascade=false参数，就会采用非级联删除，此时删除StatefulSet不会删除它的Pod：

```
kubectl get pods
```

输出信息

```
NAME    READY   STATUS    RESTARTS   AGE
web-0   1/1     Running   0          30m
web-1   1/1     Running   0          31m
web-2   1/1     Running   0          6m55s
```

删除StatefulSet

```
kubectl delete statefulsets.apps web --cascade=false
```

输出信息

```
statefulset.apps "web" deleted
```

查看StatefulSet

```
kubectl get statefulsets.apps
```

输出信息

```
No resources found in default namespace.
```

查看Pod

```
kubectl get pods
```

输出信息 Pod还在

```
NAME    READY   STATUS    RESTARTS   AGE
web-0   1/1     Running   0          32m
web-1   1/1     Running   0          33m
web-2   1/1     Running   0          8m46s
```

由于此时删除了StatefulSet，因此单独删除Pod时

```
kubectl delete pods web-2
```

输出信息

```
pod "web-2" deleted
```

查看Pod

```
NAME    READY   STATUS    RESTARTS   AGE
web-1   1/1     Running   0          34m
web-2   1/1     Running   0          35m
```

当再次创建此StatefulSet时，web-0 会被重新创建，web-1由于已经存在不会被再次创建，因为最初此StatefulSet的replicas是2 所以web-2会被删除 

如下 忽略已经存在的错误

```
kubectl create -f StatefulSet-nginx.yaml
```

```
statefulset.apps/web created
Error from server (AlreadyExists): error when creating "StatefulSet-nginx.yaml": services "nginx" already exists
```

查看Pod

```
kubectl get pods
```

输出信息

```
NAME    READY   STATUS    RESTARTS   AGE
web-0   1/1     Running   0          2m57s
web-1   1/1     Running   0          17s
```

2 级联删除

省略--cascade=false为级联删除

```
kubectl delete statefulsets.apps web
```

输出信息

```
statefulset.apps "web" deleted
```



```
kubectl get pods
```

```
No resources found in default namespace.
```

```
kubectl get statefulsets.apps 
```

```
No resources found in default namespace.
```

当然 也可以使用-f参数直接删除StatefulSet和Service（此文件将Stateful和Service写一起了）

```
kubectl delete -f StatefulSet-nginx.yaml
```

输出信息

```
service "nginx" deleted
statefulset.apps "web" deleted
```

