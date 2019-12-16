# 安装RabbitMQ集群到K8S集群中

本例安装仍采用NFS作为后端存储，同样也可以采用GFS作为动态存储，更改方式和上一节类似，本次安装在public-service命名空间内。和Redis集群类似，持久化部署也不是必须的，同样也可以采用hostpath进行挂载等

### 安装配置

### 1 创建命名空间

```shell
kubectl create namespace public-service
```

### 2 配置NFS

创建目录

```
mkdir  -p /k8s/rmq-cluster/rabbitmq-cluster-1
mkdir  -p /k8s/rmq-cluster/rabbitmq-cluster-2
mkdir  -p /k8s/rmq-cluster/rabbitmq-cluster-3
```



```
echo "/k8s/rmq-cluster/rabbitmq-cluster-1/ *(rw,sync,no_subtree_check,no_root_squash)" >> /etc/exports
echo "/k8s/rmq-cluster/rabbitmq-cluster-2/ *(rw,sync,no_subtree_check,no_root_squash)" >> /etc/exports
echo "/k8s/rmq-cluster/rabbitmq-cluster-3/ *(rw,sync,no_subtree_check,no_root_squash)" >> /etc/exports
```

```
systemctl restart nfs
systemctl enable nfs
```



###  3 配置各YAML文件

1 rabbitmq-configmap.yaml

同样将RabbitMQ的配置文件放置到ConfigMap中，定义ConfigMap的文件rabbitmq-configmap.yaml如下（需要开启的插件请按需修改）：

```yaml
kind: ConfigMap
apiVersion: v1
metadata:
  name: rmq-cluster-config
  namespace: public-service
  labels:
    addonmanager.kubernetes.io/mode: Reconcile
data:
    enabled_plugins: |
      [rabbitmq_management,rabbitmq_peer_discovery_k8s].
    rabbitmq.conf: |
      loopback_users.guest = false
      cluster_formation.peer_discovery_backend = rabbit_peer_discovery_k8s
      cluster_formation.k8s.host = kubernetes.default.svc.cluster.local
      cluster_formation.k8s.address_type = hostname
      cluster_formation.k8s.hostname_suffix = .rmq-cluster.public-service.svc.cluster.local
      cluster_formation.node_cleanup.interval = 10
      cluster_formation.node_cleanup.only_log_warning = true
      cluster_partition_handling = autoheal
      queue_master_locator=min-masters
```

2 rabbitmq-pv.yaml

定义RabbitMQ的持久化文件，即rabbitmq-pv.yaml(请按需修改)

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-rmq-1
spec:
  capacity:
    storage: 4Gi
  accessModes:
    - ReadWriteMany
  volumeMode: Filesystem
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: "rmq-storage-class"
  nfs:
    path: /k8s/rmq-cluster/rabbitmq-cluster-1
    server: 192.168.10.10
    
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-rmq-2
spec:
  capacity:
    storage: 4Gi
  accessModes:
    - ReadWriteMany
  volumeMode: Filesystem
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: "rmq-storage-class"
  nfs:
    path: /k8s/rmq-cluster/rabbitmq-cluster-2
    server: 192.168.10.10

---

apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-rmq-3
spec:
  capacity:
    storage: 4Gi
  accessModes:
    - ReadWriteMany
  volumeMode: Filesystem
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: "rmq-storage-class"
  nfs:
    path: /k8s/rmq-cluster/rabbitmq-cluster-3
    server: 192.168.10.10
```

本例RabbitMQ持久化所用的也是静态PV，动态PV无需创建此文件，此处使用的是NFS作为后端存储，请按需修改

3 rbac授权文件

rabbitmq-rbac.yaml



```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: rmq-cluster
  namespace: public-service
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: rmq-cluster
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
  name: rmq-cluster
  namespace: public-service
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: rmq-cluster
subjects:
- kind: ServiceAccount
  name: rmq-cluster
  namespace: public-service
```



4 secre文件

rabbitmq-secret.yaml

```yaml
kind: Secret
apiVersion: v1
metadata:
  name: rmq-cluster-secret
  namespace: public-service
stringData:
  cookie: ERLANG_COOKIE
  password: RABBITMQ_PASS
  url: amqp://RABBITMQ_USER:RABBITMQ_PASS@rmq-cluster-balancer
  username: RABBITMQ_USER
type: Opaque
```

4 service.yaml

rabbitmq-service-cluster.yaml

```yaml
kind: Service
apiVersion: v1
metadata:
  labels:
    app: rmq-cluster
  name: rmq-cluster
  namespace: public-service
spec:
  ports:
  - name: amqp
    port: 5672
    targetPort: 5672
  selector:
    app: rmq-cluster
```



rabbitmq-service-lb.yaml

```yaml
kind: Service
apiVersion: v1
metadata:
  labels:
    app: rmq-cluster
    type: LoadBalancer
  name: rmq-cluster-balancer
  namespace: public-service
spec:
  ports:
  - name: http
    port: 15672
    protocol: TCP
    targetPort: 15672
  - name: amqp
    port: 5672
    protocol: TCP
    targetPort: 5672
  selector:
    app: rmq-cluster
  type: NodePort
```



 rabbitnq-cluster-ss.yaml

本例安装同样采用StatefulSet的形式，定义StatefulSet文件rabbitmq-cluster-ss.yaml如下

```yaml
kind: StatefulSet
apiVersion: apps/v1
metadata:
  labels:
    app: rmq-cluster
  name: rmq-cluster
  namespace: public-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: rmq-cluster
  serviceName: rmq-cluster
  template:
    metadata:
      labels:
        app: rmq-cluster
    spec:
      containers:
      - args:
        - -c
        - cp -v /etc/rabbitmq/rabbitmq.conf ${RABBITMQ_CONFIG_FILE}; exec docker-entrypoint.sh
          rabbitmq-server
        command:
        - sh
        env:
        - name: RABBITMQ_DEFAULT_USER
          valueFrom:
            secretKeyRef:
              key: username
              name: rmq-cluster-secret
        - name: RABBITMQ_DEFAULT_PASS
          valueFrom:
            secretKeyRef:
              key: password
              name: rmq-cluster-secret
        - name: RABBITMQ_ERLANG_COOKIE
          valueFrom:
            secretKeyRef:
              key: cookie
              name: rmq-cluster-secret
        - name: K8S_SERVICE_NAME
          value: rmq-cluster
        - name: POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: RABBITMQ_USE_LONGNAME
          value: "true"
        - name: RABBITMQ_NODENAME
          value: rabbit@$(POD_NAME).rmq-cluster.$(POD_NAMESPACE).svc.cluster.local
        - name: RABBITMQ_CONFIG_FILE
          value: /var/lib/rabbitmq/rabbitmq.conf
        image: rabbitmq:3.8.2-management
        imagePullPolicy: IfNotPresent
        livenessProbe:
          exec:
            command:
            - rabbitmqctl
            - status
          initialDelaySeconds: 30
          timeoutSeconds: 10
        name: rabbitmq
        ports:
        - containerPort: 15672
          name: http
          protocol: TCP
        - containerPort: 5672
          name: amqp
          protocol: TCP
        readinessProbe:
          exec:
            command:
            - rabbitmqctl
            - status
          initialDelaySeconds: 10
          timeoutSeconds: 10
        volumeMounts:
        - mountPath: /etc/rabbitmq
          name: config-volume
          readOnly: false
        - mountPath: /var/lib/rabbitmq
          name: rabbitmq-storage
          readOnly: false
      serviceAccountName: rmq-cluster
      terminationGracePeriodSeconds: 30
      volumes:
      - configMap:
          items:
          - key: rabbitmq.conf
            path: rabbitmq.conf
          - key: enabled_plugins
            path: enabled_plugins
          name: rmq-cluster-config
        name: config-volume
  volumeClaimTemplates:
  - metadata:
      name: rabbitmq-storage
    spec:
      accessModes:
      - ReadWriteMany
      storageClassName: "rmq-storage-class"
      resources:
        requests:
          storage: 4Gi
```

验证文件

```
[root@master rabbitmq]# ls
rabbitmq-cluster-ss.yaml  rabbitmq-pv.yaml    rabbitmq-secret.yaml           rabbitmq-service-lb.yaml
rabbitmq-configmap.yaml   rabbitmq-rbac.yaml  rabbitmq-service-cluster.yaml
```

创建资源

```
kubectl apply -f .
```

