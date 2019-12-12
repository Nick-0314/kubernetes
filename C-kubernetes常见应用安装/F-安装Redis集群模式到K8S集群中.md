# 安装K8S集群模式到K8S集群中

本节演示安装Redis4.0.8集群模式到Kubernetes集群中，本小节安装采用的是NFS（阿里云可以采用NAS）作为持久化存储，当然也可以使用上节创建的GFS提供的动态存储，只需修改redis-cluster-ss.yaml的storageClassName即可，同时无需再创建redis-cluster-pv.yaml文件。在生产环境中，对Redis集群实现持久化部署并不是必须的，可以采用hostPath模式讲宿主机的本地目录挂载至Redis存储目录，再加上Pod互斥，不让Redis实例部署在同一个宿主机上，之后再利用节点亲和力，尽量将Redis实例部署在原有的宿主机上，此种方式和直接在宿主机上部署Redis并无太大区别，并且实现了Redis的自动容灾功能。当然，在实际使用中，也可以不对Redis进行持久化部署，因为生产环境一般采用Cluster模式部署，同时宕机的可能性较小。

## 1 各文件介绍

1 redis-cluster-configmap.yaml

使用ConfigMap配置Redis的配置文件，请按需修改

```yaml
kind: ConfigMap
apiVersion: v1
metadata:
  name: redis-cluster-config
  namespace: public-service
  labels:
    addonmanager.kubernetes.io/mode: Reconcile
data:
    redis-cluster.conf: |
      # 节点端口
      port 6379
      # #  开启集群模式
      cluster-enabled yes
      # #  节点超时时间，单位毫秒
      cluster-node-timeout 15000
      # #  集群内部配置文件
      cluster-config-file "nodes.conf"
```

2 redis-cluster-pv.yaml文件

定义Redis的持久化文件，请按需修改

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-redis-cluster-1
  namespace: public-service
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: "redis-cluster-storage-class"
  nfs:
    # real share directory
    path: /k8s/redis-cluster/1
    # nfs real ip
    server: 192.168.10.10

---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-redis-cluster-2
  namespace: public-service
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: "redis-cluster-storage-class"
  nfs:
    # real share directory
    path: /k8s/redis-cluster/2
    # nfs real ip
    server: 192.168.10.10

---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-redis-cluster-3
  namespace: public-service
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: "redis-cluster-storage-class"
  nfs:
    # real share directory
    path: /k8s/redis-cluster/3
    # nfs real ip
    server: 192.168.10.10

---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-redis-cluster-4
  namespace: public-service
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: "redis-cluster-storage-class"
  nfs:
    # real share directory
    path: /k8s/redis-cluster/4
    # nfs real ip
    server: 192.168.10.10

---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-redis-cluster-5
  namespace: public-service
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: "redis-cluster-storage-class"
  nfs:
    # real share directory
    path: /k8s/redis-cluster/5
    # nfs real ip
    server: 192.168.10.10

---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-redis-cluster-6
  namespace: public-service
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: "redis-cluster-storage-class"
  nfs:
    # real share directory
    path: /k8s/redis-cluster/6
    # nfs real ip
    server: 192.168.10.10
```

此文件为Redis集群6个实例所用的PV文件，动态存储无需创建此文件，其中path是NFS共享目录，Server为NFS服务器的地址，按需修改

3 redis-cluster-rbac.yaml

此文件定义的是一些权限，无须修改

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: redis-cluster
  namespace: public-service
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: redis-cluster
  namespace: public-service
rules:
  - apiGroups:
      - ""
    resources:
      - endpoints
    verbs:
      - get
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: redis-cluster
  namespace: public-service
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: redis-cluster
subjects:
- kind: ServiceAccount
  name: redis-cluster
  namespace: public-service
```

4 redis-cluster-service.yaml文件

此文件用于定义Redis的service，用于集群节点的通信

```yaml
kind: Service
apiVersion: v1
metadata:
  labels:
    app: redis-cluster-ss
  name: redis-cluster-ss
  namespace: public-service
spec:
  clusterIP: None
  ports:
  - name: redis
    port: 6379
    targetPort: 6379
  selector:
    app: redis-cluster-ss
```

如果开发或运维需要连接到该集群可以使用NodePort，程序连接直接使用Service的地址（name）即可

5 redis-cluster-ss.yaml文件

本例Redis的安装采用Statefulset模式，redis-cluster-ss.yaml文件定义如下

```yaml
kind: StatefulSet
apiVersion: apps/v1beta1
metadata:
  labels:
    app: redis-cluster-ss
  name: redis-cluster-ss
  namespace: public-service
spec:
  replicas: 6
  selector:
    matchLabels:
      app: redis-cluster-ss
  serviceName: redis-cluster-ss
  template:
    metadata:
      labels:
        app: redis-cluster-ss
    spec:
      containers:
      - args:
        - -c
        - cp /mnt/redis-cluster.conf /data ; redis-server /data/redis-cluster.conf
        command:
        - sh
        image: dotbalo/redis-trib:4.0.10
        imagePullPolicy: IfNotPresent
        name: redis-cluster
        ports:
        - containerPort: 6379
          name: masterport
          protocol: TCP
        volumeMounts:
        - mountPath: /mnt/
          name: config-volume
          readOnly: false
        - mountPath: /data/
          name: redis-cluster-storage
          readOnly: false
      serviceAccountName: redis-cluster
      terminationGracePeriodSeconds: 30
      volumes:
      - configMap:
          items:
          - key: redis-cluster.conf
            path: redis-cluster.conf
          name: redis-cluster-config 
        name: config-volume
  volumeClaimTemplates:
  - metadata:
      name: redis-cluster-storage
    spec:
      accessModes:
      - ReadWriteOnce
      storageClassName: "redis-cluster-storage-class"
      resources:
        requests:
          storage: 1Gi
```

此文件用于创建Redis实例，不一定非要使用StatefulSet，采用Deployment部署也可以。本例创建6个实例，用于集群中节点的3主3从。

## 2 创建Redis命名空间

可以将公共服务都放置在同一个Namespace下，比如public-service中，然后按需修改

创建namespace：

```shell
kubectl create namespace public-service
```

如果需要部署到其他Namespace，需要更改当前目录中所有的文件的namespace

```shell
sed -i "s#public-service#<YOUR_NAMESPACE>#g" *
```

## 3创建Redis集群PV

动态PV无需此步骤，本节使用的是NFS作为PV，在实际使用中，Redis的数据不一定需要做持久化，按需配置即可

配置nfs服务器

```shell
mkdir -p /k8s/redis-cluster
mkdir /k8s/redis-cluster/{1..6}
```

```shell
echo "/k8s/redis-cluster/1 *(rw,sync,no_subtree_check,no_root_squash)" >> /etc/exports
echo "/k8s/redis-cluster/2 *(rw,sync,no_subtree_check,no_root_squash)" >> /etc/exports
echo "/k8s/redis-cluster/3 *(rw,sync,no_subtree_check,no_root_squash)" >> /etc/exports
echo "/k8s/redis-cluster/4 *(rw,sync,no_subtree_check,no_root_squash)" >> /etc/exports
echo "/k8s/redis-cluster/5 *(rw,sync,no_subtree_check,no_root_squash)" >> /etc/exports
echo "/k8s/redis-cluster/6 *(rw,sync,no_subtree_check,no_root_squash)" >> /etc/exports
```

```shell
systemctl restart nfs
```

## 4 创建集群

```shell
kubectl apply -f .
```

## 5 创建集群

redis集群模式和Redis哨兵模式有所不同，等待节点全部启动后，开始创建slot

```
v=""
```

```shell
for i in `kubectl get po -n public-service -o wide | awk  '{print $6}' | grep -v IP`; do v="$v $i:6379";done
kubectl exec -ti redis-cluster-ss-5 -n public-service -- redis-trib.rb create --replicas 1 $v
```

![image-20191127231526140](image/F-安装Redis集群模式到K8S集群中/image-20191127231526140.png)

## 

# Redis 5.0部署



ConfigMap 用于挂载到pod内

```yaml
kind: ConfigMap
apiVersion: v1
metadata:
  name: redis-cluster-config
  namespace: default
  labels:
    addonmanager.kubernetes.io/mode: Reconcile
data:
    redis.conf: |
      port 6379
      cluster-enabled yes
      cluster-config-file nodes.conf
      cluster-node-timeout 5000
      appendonly yes
      daemonize no
      protected-mode no
      pidfile  /var/run/redis_6379.pid
      bind 0.0.0.0
      save ""
      save 900 1
      save 300 10
      save 60 10000
      dbfilename dump.rdb
      dir /usr/local/redis/data/
      stop-writes-on-bgsave-error yes
      rdbcompression yes
      rdb-save-incremental-fsync yes
      appendonly yes
      appendfilename "appendonly.aof"
      dir /usr/local/redis/data/
      appendfsync everysec
      no-appendfsync-on-rewrite no
      auto-aof-rewrite-percentage 100
      auto-aof-rewrite-min-size 64mb
      aof-rewrite-incremental-fsync yes
      aof-load-truncated yes
      aof-use-rdb-preamble yes
```

配置nfs服务器，创建目录及子目录

```shell
mkdir -p /k8s/redis-cluster
mkdir /k8s/redis-cluster/{1..6}
```

配置nfs服务

```shell
echo "/k8s/redis-cluster/1 *(rw,sync,no_subtree_check,no_root_squash)" >> /etc/exports
echo "/k8s/redis-cluster/2 *(rw,sync,no_subtree_check,no_root_squash)" >> /etc/exports
echo "/k8s/redis-cluster/3 *(rw,sync,no_subtree_check,no_root_squash)" >> /etc/exports
echo "/k8s/redis-cluster/4 *(rw,sync,no_subtree_check,no_root_squash)" >> /etc/exports
echo "/k8s/redis-cluster/5 *(rw,sync,no_subtree_check,no_root_squash)" >> /etc/exports
echo "/k8s/redis-cluster/6 *(rw,sync,no_subtree_check,no_root_squash)" >> /etc/exports
```

```shell
systemctl restart nfs
```

配置nfs的持久化卷，请根据实际环境修改

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-redis-cluster-1
  namespace: default
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: "redis-cluster-storage-class"
  nfs:
    path: /k8s/redis-cluster/1
    server: 192.168.10.10

---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-redis-cluster-2
  namespace: default
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: "redis-cluster-storage-class"
  nfs:
    path: /k8s/redis-cluster/2
    server: 192.168.10.10

---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-redis-cluster-3
  namespace: default
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: "redis-cluster-storage-class"
  nfs:
    path: /k8s/redis-cluster/3
    server: 192.168.10.10

---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-redis-cluster-4
  namespace: default
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: "redis-cluster-storage-class"
  nfs:
    path: /k8s/redis-cluster/4
    server: 192.168.10.10

---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-redis-cluster-5
  namespace: default
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: "redis-cluster-storage-class"
  nfs:
    path: /k8s/redis-cluster/5
    server: 192.168.10.10

---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-redis-cluster-6
  namespace: default
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: "redis-cluster-storage-class"
  nfs:
    path: /k8s/redis-cluster/6
    server: 192.168.10.10
```

应用文件后的输出信息

```shell
NAME                 CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS                  REASON   AGE
pv-redis-cluster-1   1Gi        RWO            Recycle          Available           redis-cluster-storage-class            7s
pv-redis-cluster-2   1Gi        RWO            Recycle          Available           redis-cluster-storage-class            7s
pv-redis-cluster-3   1Gi        RWO            Recycle          Available           redis-cluster-storage-class            7s
pv-redis-cluster-4   1Gi        RWO            Recycle          Available           redis-cluster-storage-class            7s
pv-redis-cluster-5   1Gi        RWO            Recycle          Available           redis-cluster-storage-class            7s
pv-redis-cluster-6   1Gi        RWO            Recycle          Available           redis-cluster-storage-class            7s
```

### 创建service

```yaml
kind: Service
apiVersion: v1
metadata:
  labels:
    app: redis-cluster-ss
  name: redis-cluster-ss
  namespace: default
spec:
  clusterIP: None
  ports:
  - name: redis
    port: 6379
    targetPort: 6379
  selector:
    app: redis-cluster-ss
```

应用此文件

### 创建service account

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: redis-cluster
  namespace: default
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: redis-cluster
  namespace: default
rules:
  - apiGroups:
      - ""
    resources:
      - endpoints
    verbs:
      - get
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: redis-cluster
  namespace: default
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: redis-cluster
subjects:
- kind: ServiceAccount
  name: redis-cluster
  namespace: public-service
```

应用此文件





### 创建StatStatefulSet

```yaml
kind: StatefulSet
apiVersion: apps/v1
metadata:
  labels:
    app: redis-cluster-ss
  name: redis-cluster-ss
  namespace: default
spec:
  replicas: 6
  selector:
    matchLabels:
      app: redis-cluster-ss
  serviceName: redis-cluster-ss
  template:
    metadata:
      labels:
        app: redis-cluster-ss
    spec:
      containers:
      - name: redis_ss
        image: mytting/chang:redis
        imagePullPolicy: IfNotPresent
        name: redis-cluster
        ports:
        - containerPort: 6379
          name: masterport
          protocol: TCP
        volumeMounts:
        - mountPath: /usr/local/redis/redis.conf
          subPath: redis.conf
          name: config-volume
          readOnly: true
        - mountPath: /usr/local/redis/data/
          name: redis-cluster-storage
          readOnly: false
      serviceAccountName: redis-cluster
      terminationGracePeriodSeconds: 30
      volumes:
      - configMap:
          items:
          - key: redis.conf
            path: redis.conf
          name: redis-cluster-config
        name: config-volume

  volumeClaimTemplates:
  - metadata:
      name: redis-cluster-storage
    spec:
      accessModes:
      - ReadWriteOnce
      storageClassName: "redis-cluster-storage-class"
      resources:
        requests:
          storage: 1Gi
```

应用此文件

pod状态

```
NAME                 READY   STATUS    RESTARTS   AGE   IP               NODE     NOMINATED NODE   READINESS GATES
redis-cluster-ss-0   1/1     Running   0          55s   10.244.219.71    master   <none>           <none>
redis-cluster-ss-1   1/1     Running   0          53s   10.244.104.5     node2    <none>           <none>
redis-cluster-ss-2   1/1     Running   0          46s   10.244.166.137   node1    <none>           <none>
redis-cluster-ss-3   1/1     Running   0          38s   10.244.104.6     node2    <none>           <none>
redis-cluster-ss-4   1/1     Running   0          35s   10.244.219.72    master   <none>           <none>
redis-cluster-ss-5   1/1     Running   0          31s   10.244.166.138   node1    <none>           <none>
```

### 创建集群

注意 第一条命令执行一遍即可，切勿执行多次

```
for i in `kubectl get po  -o wide  | grep redis | awk  '{print $6}' | grep -v IP`; do v="$v $i:6379";done
kubectl exec -ti redis-cluster-ss-5  -- /usr/local/redis/bin/redis-cli  --cluster create --cluster-replicas 1 $v
```

输入yes后创建成功

```shell
>>> Nodes configuration updated
>>> Assign a different config epoch to each node
>>> Sending CLUSTER MEET messages to join the cluster
Waiting for the cluster to join
...
>>> Performing Cluster Check (using node 10.244.219.72:6379)
M: 44d4837ab6fa5aee6a6a418646ea332789bce6e0 10.244.219.72:6379
   slots:[0-5460] (5461 slots) master
   1 additional replica(s)
S: e2db11f747a9067afe959ef1ed6435d8a4f70904 10.244.166.138:6379
   slots: (0 slots) slave
   replicates 8771540d08fad6fb757d00218db5e6957190c9fd
S: 4eb69a55def826e708614a9c20363000660f387f 10.244.219.73:6379
   slots: (0 slots) slave
   replicates 44d4837ab6fa5aee6a6a418646ea332789bce6e0
M: 8771540d08fad6fb757d00218db5e6957190c9fd 10.244.104.5:6379
   slots:[5461-10922] (5462 slots) master
   1 additional replica(s)
S: 3462dafc012875d98faef8d1eec651a67c2fcd41 10.244.104.6:6379
   slots: (0 slots) slave
   replicates 817d8171bd4a10dddc10a020a117c0cab8eafb1d
M: 817d8171bd4a10dddc10a020a117c0cab8eafb1d 10.244.166.137:6379
   slots:[10923-16383] (5461 slots) master
   1 additional replica(s)
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
```

